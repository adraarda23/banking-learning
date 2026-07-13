# Topic 5.4 — Listeners & Flow Control

## Hedef

Spring Batch event listener mekanizması ve flow control banking-grade derinlikte: `JobExecutionListener`, `StepExecutionListener`, `ChunkListener`, `ItemReadListener`, `ItemProcessListener`, `ItemWriteListener`, conditional flow (`on(...).to(...)`), parallel flow (`split`), nested job (`JobStep`), decision pattern, banking EOD master orchestration.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 5.1-5.3 bitti
- Spring event listener pattern

---

## Kavramlar

### 1. Listener hierarchy

```
JobExecutionListener
  beforeJob / afterJob
  ├── StepExecutionListener
  │   beforeStep / afterStep
  │   ├── ChunkListener
  │   │   beforeChunk / afterChunk / afterChunkError
  │   │   ├── ItemReadListener
  │   │   │   beforeRead / afterRead / onReadError
  │   │   ├── ItemProcessListener
  │   │   │   beforeProcess / afterProcess / onProcessError
  │   │   └── ItemWriteListener
  │   │       beforeWrite / afterWrite / onWriteError
  │   └── SkipListener / RetryListener
```

Banking için: **JobExecutionListener** (audit + notification), **StepExecutionListener** (metric + duration), **ChunkListener** (incremental progress).

### 2. JobExecutionListener — banking audit

```java
@Component
public class BankingJobExecutionListener implements JobExecutionListener {
    
    private final AuditService auditService;
    private final SlackNotifier slackNotifier;
    private final MeterRegistry meterRegistry;
    
    @Override
    public void beforeJob(JobExecution exec) {
        String jobName = exec.getJobInstance().getJobName();
        log.info("Starting job: {} (id={}, params={})", 
            jobName, exec.getId(), exec.getJobParameters());
        
        auditService.log(AuditEvent.builder()
            .action("JOB_STARTED")
            .resourceType("batch_job")
            .resourceId(jobName)
            .details(Map.of(
                "executionId", exec.getId(),
                "params", exec.getJobParameters().getParameters()))
            .build());
    }
    
    @Override
    public void afterJob(JobExecution exec) {
        String jobName = exec.getJobInstance().getJobName();
        Duration duration = Duration.between(exec.getStartTime(), exec.getEndTime());
        
        long readCount = exec.getStepExecutions().stream()
            .mapToLong(StepExecution::getReadCount).sum();
        long writeCount = exec.getStepExecutions().stream()
            .mapToLong(StepExecution::getWriteCount).sum();
        long skipCount = exec.getStepExecutions().stream()
            .mapToLong(StepExecution::getSkipCount).sum();
        
        log.info("Job {} {}: duration={}, read={}, write={}, skip={}",
            jobName, exec.getStatus(), duration, readCount, writeCount, skipCount);
        
        auditService.log(AuditEvent.builder()
            .action("JOB_FINISHED")
            .resourceType("batch_job")
            .resourceId(jobName)
            .details(Map.of(
                "status", exec.getStatus().toString(),
                "duration_ms", duration.toMillis(),
                "read_count", readCount,
                "write_count", writeCount,
                "skip_count", skipCount))
            .build());
        
        // Metrics
        meterRegistry.timer("banking.batch.duration", "job", jobName, 
            "status", exec.getStatus().toString())
            .record(duration);
        meterRegistry.counter("banking.batch.records_processed", "job", jobName)
            .increment(writeCount);
        
        // Alert on failure
        if (exec.getStatus() == BatchStatus.FAILED) {
            slackNotifier.alert(Severity.HIGH, 
                "Batch job FAILED: " + jobName,
                "Execution " + exec.getId() + " — " + exec.getAllFailureExceptions());
        }
        
        // Alert on excessive skips
        if (skipCount > 1000) {
            slackNotifier.alert(Severity.MEDIUM,
                "High skip count: " + jobName,
                skipCount + " records skipped in execution " + exec.getId());
        }
    }
}
```

### 3. StepExecutionListener — per-step audit

```java
@Component
public class BankingStepListener implements StepExecutionListener {
    
    @Override
    public void beforeStep(StepExecution stepExec) {
        String stepName = stepExec.getStepName();
        log.info("Starting step: {}", stepName);
        
        // Promote ExecutionContext key from job to step
        Map<String, Object> jobCtx = stepExec.getJobExecution().getExecutionContext().toMap();
        if (jobCtx.containsKey("eodDate")) {
            stepExec.getExecutionContext().put("eodDate", jobCtx.get("eodDate"));
        }
    }
    
    @Override
    public ExitStatus afterStep(StepExecution stepExec) {
        Duration duration = Duration.between(stepExec.getStartTime(), stepExec.getEndTime());
        
        log.info("Step {} status={} duration={} read={} write={} skip={}",
            stepExec.getStepName(), stepExec.getStatus(), duration,
            stepExec.getReadCount(), stepExec.getWriteCount(), stepExec.getSkipCount());
        
        // Conditional ExitStatus based on banking rules
        if (stepExec.getSkipCount() > 100) {
            return new ExitStatus("DEGRADED", 
                "Too many skips: " + stepExec.getSkipCount());
        }
        
        if (stepExec.getReadCount() == 0) {
            return new ExitStatus("NO_DATA", "No input records");
        }
        
        return stepExec.getExitStatus();
    }
}
```

### 4. ChunkListener — incremental progress

```java
@Component
public class ChunkProgressListener implements ChunkListener {
    
    private final ProgressRepository progressRepo;
    
    @Override
    public void beforeChunk(ChunkContext ctx) {}
    
    @Override
    public void afterChunk(ChunkContext ctx) {
        StepExecution stepExec = ctx.getStepContext().getStepExecution();
        String stepName = stepExec.getStepName();
        long writeCount = stepExec.getWriteCount();
        long commitCount = stepExec.getCommitCount();
        
        // Persist progress for UI / monitoring
        progressRepo.update(stepExec.getId(), writeCount, commitCount);
        
        // Periodic metric
        if (commitCount % 100 == 0) {
            log.info("{}: {} records processed ({} chunks)", stepName, writeCount, commitCount);
        }
    }
    
    @Override
    public void afterChunkError(ChunkContext ctx) {
        StepExecution stepExec = ctx.getStepContext().getStepExecution();
        log.warn("Chunk error in {}: rollback at commit {}",
            stepExec.getStepName(), stepExec.getCommitCount());
    }
}
```

### 5. Conditional flow — banking decision logic

```java
@Bean
public Job conditionalFlowJob(JobRepository repo, 
                              Step validateStep, 
                              Step processStep,
                              Step skipReportStep,
                              Step failureCleanupStep) {
    return new JobBuilder("conditionalFlowJob", repo)
        .start(validateStep)
            .on("NO_DATA").end()                       // no work to do
            .on("COMPLETED").to(processStep)
            .from(processStep)
                .on("COMPLETED").to(skipReportStep)
                .on("DEGRADED").to(skipReportStep)
                .on("FAILED").to(failureCleanupStep)
            .from(skipReportStep)
                .on("*").end()
            .from(failureCleanupStep)
                .on("*").fail()
        .end()
        .build();
}
```

**Banking örnek — EOD flow:**

```java
@Bean
public Job eodMasterJob(JobRepository repo,
                       Step preflightCheck,
                       Step interestAccrual,
                       Step generateStatements,
                       Step reconciliation,
                       Step alertOnBreaks,
                       Step bypassAlertStep) {
    return new JobBuilder("eodMasterJob", repo)
        .start(preflightCheck)
            .on("FAILED").fail()           // Stop if precondition not met
            .on("COMPLETED").to(interestAccrual)
            
        .from(interestAccrual)
            .on("*").to(generateStatements)
            
        .from(generateStatements)
            .on("*").to(reconciliation)
            
        .from(reconciliation)
            .on("BREAKS_FOUND").to(alertOnBreaks)
            .on("COMPLETED").to(bypassAlertStep)
            
        .from(alertOnBreaks)
            .on("*").end()
            
        .from(bypassAlertStep)
            .on("*").end()
        .end()
        .build();
}
```

### 6. Parallel flow — split

```java
@Bean
public Job parallelFlowJob(JobRepository repo,
                           Step interestAccrual,
                           Step feeAccrual,
                           Step penaltyAccrual,
                           Step consolidateStep) {
    
    Flow accrualFlow1 = new FlowBuilder<Flow>("interestFlow")
        .start(interestAccrual).build();
    Flow accrualFlow2 = new FlowBuilder<Flow>("feeFlow")
        .start(feeAccrual).build();
    Flow accrualFlow3 = new FlowBuilder<Flow>("penaltyFlow")
        .start(penaltyAccrual).build();
    
    return new JobBuilder("accrualJob", repo)
        .start(new FlowBuilder<Flow>("parallelAccruals")
            .split(taskExecutor())   // Run in parallel
            .add(accrualFlow1, accrualFlow2, accrualFlow3)
            .build())
        .next(consolidateStep)
        .end()
        .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(3);
    executor.setMaxPoolSize(5);
    executor.setQueueCapacity(10);
    executor.setThreadNamePrefix("batch-");
    executor.initialize();
    return executor;
}
```

Banking: independent accruals (interest + fee + penalty) parallel run.

### 7. JobStep — nested job

```java
@Bean
public Step nestedJobStep(JobRepository repo, Job innerJob, JobLauncher launcher) {
    JobStep step = new JobStep();
    step.setJobLauncher(launcher);
    step.setJob(innerJob);
    step.setJobRepository(repo);
    step.setJobParametersExtractor(jobParametersExtractor());
    return step;
}

@Bean
public JobParametersExtractor jobParametersExtractor() {
    DefaultJobParametersExtractor extractor = new DefaultJobParametersExtractor();
    extractor.setKeys(new String[]{"eodDate", "batchSize"});
    return extractor;
}
```

Modular composition — sub-job as a step in main job. Banking master job → reuse sub-jobs.

### 8. Decision pattern — JobExecutionDecider

```java
public class BankingDecider implements JobExecutionDecider {
    
    private final TransactionVolumeRepository volumeRepo;
    
    @Override
    public FlowExecutionStatus decide(JobExecution jobExec, StepExecution stepExec) {
        LocalDate eodDate = LocalDate.parse(
            jobExec.getJobParameters().getString("eodDate"));
        
        long txCount = volumeRepo.countByDate(eodDate);
        
        if (txCount == 0) return new FlowExecutionStatus("NO_DATA");
        if (txCount > 1_000_000) return new FlowExecutionStatus("HIGH_VOLUME");
        return new FlowExecutionStatus("NORMAL");
    }
}

@Bean
public Job dynamicJob(JobRepository repo, Step normal, Step highVolumePartitioned) {
    BankingDecider decider = new BankingDecider();
    
    return new JobBuilder("dynamicJob", repo)
        .start(preflightStep)
            .next(decider)
                .on("NO_DATA").end()
                .on("NORMAL").to(normal)
                .on("HIGH_VOLUME").to(highVolumePartitioned)
        .from(normal).on("*").end()
        .from(highVolumePartitioned).on("*").end()
        .end()
        .build();
}
```

Banking — dynamic strategy: small day → single-threaded, big day → partitioned.

### 9. Banking EOD orchestration full example

```java
@Bean
public Job bankingEodMasterJob(JobRepository repo, JobLauncher launcher,
                                Step preflightStep,
                                Step interestAccrualJobStep,
                                Step bsmvKkdfCalcJobStep,
                                Step statementGenJobStep,
                                Step masakReportJobStep,
                                Step cardReconJobStep,
                                Step eftReconJobStep,
                                Step swiftReconJobStep,
                                Step regulatoryReportStep,
                                Step finalAuditStep,
                                Step incidentNotifyStep) {
    
    Flow reconParallel = new FlowBuilder<Flow>("reconParallel")
        .split(taskExecutor())
        .add(
            new FlowBuilder<Flow>("cardRecon").start(cardReconJobStep).build(),
            new FlowBuilder<Flow>("eftRecon").start(eftReconJobStep).build(),
            new FlowBuilder<Flow>("swiftRecon").start(swiftReconJobStep).build()
        )
        .build();
    
    return new JobBuilder("bankingEodMasterJob", repo)
        .listener(new BankingJobExecutionListener(...))
        
        .start(preflightStep)
            .on("FAILED").to(incidentNotifyStep)
            .on("COMPLETED").to(interestAccrualJobStep)
            
        .from(interestAccrualJobStep)
            .on("*").to(bsmvKkdfCalcJobStep)
            
        .from(bsmvKkdfCalcJobStep)
            .on("*").to(statementGenJobStep)
            
        .from(statementGenJobStep)
            .on("*").to(masakReportJobStep)
            
        .from(masakReportJobStep)
            .on("*").to(reconParallel)
            
        .next(reconParallel)
            .on("*").to(regulatoryReportStep)
            
        .from(regulatoryReportStep)
            .on("*").to(finalAuditStep)
            
        .from(finalAuditStep)
            .on("*").end()
            
        .from(incidentNotifyStep)
            .on("*").fail()
        .end()
        .build();
}
```

Banking realistic EOD orchestration — accrual + reporting + parallel recon + final audit.

### 10. Banking — listener anti-pattern'leri

**Anti-pattern 1: Heavy work in listener**

```java
@Override
public void afterChunk(ChunkContext ctx) {
    bigDataAggregation();   // ❌ blocks chunk
}
```

Listener fast. Heavy work → async.

**Anti-pattern 2: Mutating state in listener**

```java
@Override
public void afterStep(StepExecution exec) {
    exec.getJobExecution().getExecutionContext().put("counter", increment);   // ❌
}
```

Listener side-effect free. Use ExecutionContext properly via promotion listener.

**Anti-pattern 3: Exception in listener swallowed**

```java
@Override
public void afterJob(JobExecution exec) {
    try {
        externalApi.notify();
    } catch (Exception e) {
        // silently ignore   ❌
    }
}
```

Banking için audit + log + metric ihmal edilemez. Async with retry.

**Anti-pattern 4: Conditional flow .on("FAILED").fail() forget**

Missing failure path → job continues unintentionally.

**Anti-pattern 5: Split without TaskExecutor**

```java
.split(new SyncTaskExecutor())   // ❌ not parallel
```

ThreadPoolTaskExecutor.

**Anti-pattern 6: Nested job without ParametersExtractor**

Sub-job missing context. DefaultJobParametersExtractor.

**Anti-pattern 7: Decider with side effects**

```java
public FlowExecutionStatus decide(...) {
    sendEmail();   // ❌
    return ...;
}
```

Pure decision. Side effects → step.

**Anti-pattern 8: Per-record listener for high volume**

```java
@Override
public void afterRead(Transaction tx) {
    log.info("Read tx {}", tx.getId());   // ❌ 1M log lines
}
```

ChunkListener daha pratik.

**Anti-pattern 9: ExitStatus mismatch**

`afterStep` returns custom but flow check different code:

```java
return new ExitStatus("DEGRADED", ...);   // Listener
.on("WARNING").to(...)                    // Flow — mismatch
```

**Anti-pattern 10: JobExecutionListener async without ack**

Notification queued, never sent. Banking audit retry mechanism.

---

## Önemli olabilecek araştırma kaynakları

- Spring Batch reference — listeners
- Spring Batch flow control
- "Pro Spring Batch" — Michael Minella

---

## Mini task'ler

### Task 5.4.1 — JobExecutionListener audit (45 dk)

beforeJob + afterJob → audit table + metric. Slack notify on FAILED.

### Task 5.4.2 — ChunkListener progress (30 dk)

Per 100 chunk → progress DB. UI poll endpoint.

### Task 5.4.3 — StepExecutionListener conditional ExitStatus (30 dk)

skipCount > 100 → "DEGRADED". 0 records → "NO_DATA".

### Task 5.4.4 — Conditional flow (60 dk)

3-step job. Step 1 → COMPLETED/FAILED/NO_DATA branching.

### Task 5.4.5 — Parallel split — accrual flow (60 dk)

Interest + fee + penalty parallel. ThreadPoolTaskExecutor.

### Task 5.4.6 — JobStep nested (45 dk)

Outer master job. Inner = independent reconciliation job. ParametersExtractor.

### Task 5.4.7 — JobExecutionDecider banking (45 dk)

Day's tx volume → choose normal vs partitioned execution.

### Task 5.4.8 — EOD master orchestration (90 dk)

Multi-step banking EOD job (yukarıdaki). Listener + conditional + parallel.

---

## Test yazma rehberi

```java
@SpringBatchTest
class FlowControlTest {
    
    @Test
    void conditionalFlowSkipsOnNoData() {
        // Setup empty input
        seedTransactions(0);
        
        JobExecution exec = launcher.launchJob();
        
        assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        // Only preflight step ran
        assertThat(exec.getStepExecutions()).hasSize(1);
        assertThat(exec.getStepExecutions().iterator().next().getExitStatus())
            .isEqualTo(ExitStatus.NOOP);
    }
    
    @Test
    void parallelSplitRunsConcurrently() {
        Instant start = Instant.now();
        JobExecution exec = launcher.launchJob();
        Duration totalDuration = Duration.between(start, Instant.now());
        
        // If serial each step takes 2 sec, parallel should be < 4 sec for 3 steps
        assertThat(totalDuration).isLessThan(Duration.ofSeconds(4));
    }
    
    @Test
    void listenerCapturesJobMetrics() {
        launcher.launchJob();
        
        Timer timer = meterRegistry.find("banking.batch.duration")
            .tag("job", "eodJob")
            .timer();
        assertThat(timer.count()).isEqualTo(1);
    }
    
    @Test
    void deciderRoutesByVolume() {
        seedTransactions(2_000_000);   // High volume
        
        JobExecution exec = launcher.launchJob();
        
        assertThat(exec.getStepExecutions().stream()
            .map(StepExecution::getStepName))
            .contains("highVolumePartitioned");
    }
}
```

---

## Claude-verify prompt

```
Listener + Flow control banking-grade kriterlere göre değerlendir:

1. Listeners:
   - JobExecutionListener audit + metric + alert?
   - StepExecutionListener duration + conditional ExitStatus?
   - ChunkListener progress?
   - SkipListener / RetryListener audit?

2. Banking listener content:
   - Audit log per job start/end?
   - Metrics (Topic 9.2)?
   - Slack notify on FAILED?
   - Skip count threshold alert?

3. Conditional flow:
   - on("FAILED").fail() explicit?
   - on("COMPLETED").to(...) standard?
   - Custom ExitStatus matched in flow?
   - End vs fail correct usage?

4. Parallel:
   - split() with ThreadPoolTaskExecutor (not Sync)?
   - Independent flows?
   - Aggregation step after?

5. JobStep:
   - JobParametersExtractor for context propagation?
   - Modular composition?

6. Decider:
   - Pure decision (no side effects)?
   - Repository-based decision banking?

7. Banking EOD:
   - Master job orchestration?
   - Recon parallel?
   - Notification step on failure?
   - Audit + regulatory step?

8. Performance:
   - Listener fast (no heavy work)?
   - Per-chunk progress not per-record?

9. Audit + compliance:
   - Job start/finish audit?
   - Step metric?
   - Failure incident workflow?

10. Anti-pattern:
    - Heavy work in listener YOK?
    - Listener side effects (mutation) YOK?
    - Missing FAILED branch YOK?
    - Per-record listener YOK?
    - ExitStatus mismatch YOK?
    - Sync TaskExecutor in split YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] JobExecutionListener audit + metric + Slack
- [ ] StepExecutionListener conditional ExitStatus
- [ ] ChunkListener progress persist
- [ ] Conditional flow 3+ branch
- [ ] Parallel split (TaskExecutor)
- [ ] JobStep nested + ParametersExtractor
- [ ] Decider banking volume-based
- [ ] EOD master orchestration
- [ ] 6+ integration test

---

## Defter notları (10 madde)

1. "Listener hierarchy (Job/Step/Chunk/Item) banking levels: ____."
2. "JobExecutionListener audit + metric + Slack notify failure: ____."
3. "StepExecutionListener custom ExitStatus banking conditional flow: ____."
4. "ChunkListener per-100 progress vs per-record overhead: ____."
5. "Conditional flow `.on(...).to(...)` banking EOD branching: ____."
6. "Parallel split + ThreadPoolTaskExecutor (not Sync) parallel accrual: ____."
7. "JobStep nested + JobParametersExtractor modular EOD composition: ____."
8. "JobExecutionDecider banking volume-based dynamic strategy: ____."
9. "Banking EOD master orchestration (preflight → accrual → recon parallel → regulatory): ____."
10. "Anti-pattern (heavy listener work, missing FAILED branch, sync split, per-record listener): ____."
