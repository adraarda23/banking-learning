# Topic 5.3 — Skip, Retry, Restart

## Hedef

Spring Batch'in **hata yönetimi** mekanizmalarını banking-grade derinlikte öğrenmek: `SkipPolicy` (transient veya data quality hataları atla), `RetryPolicy` (network/DB transient hataları), **restartability** (kesilen job kaldığı yerden devam), ExecutionContext + StepExecutionContext state, idempotency, banking EOD job pattern. Bir hata tüm job'u patlatmamalı, restart kayıpsız olmalı.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 5.1, 5.2 bitti (architecture + chunk-oriented)
- Spring transaction kavramı (Phase 3)
- Exception hierarchy

---

## Kavramlar

### 1. Failure modes — banking batch

EOD job 1M record işliyor. Hata türleri:

| Type | Anlam | Strategy |
|---|---|---|
| **Data quality** | Single record bozuk (eksik field, parse hatası) | **Skip** |
| **Transient infra** | DB connection hiccup, network timeout | **Retry** |
| **Business rule violation** | Limit exceeded, sanctions hit | **Skip + log** |
| **Programming bug** | NullPointerException | **Fail fast** |
| **System crash** | JVM/pod restart | **Restart** |
| **Poison pill** | Always-failing record | **Skip after N retries** |

Banking batch design: **graceful degradation** + audit + recovery.

### 2. Skip policy — single record drop

```java
@Bean
public Step processTransactions(JobRepository jobRepo, PlatformTransactionManager tm,
                                ItemReader<Transaction> reader,
                                ItemProcessor<Transaction, Posting> processor,
                                ItemWriter<Posting> writer) {
    return new StepBuilder("processTransactions", jobRepo)
        .<Transaction, Posting>chunk(100, tm)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .skipPolicy(bankingSkipPolicy())
        .skipLimit(100)
        .skip(InvalidIbanException.class)
        .skip(ParseException.class)
        .skip(BusinessValidationException.class)
        .noSkip(NullPointerException.class)
        .listener(skipListener())
        .build();
}

@Bean
public SkipPolicy bankingSkipPolicy() {
    return new SkipPolicy() {
        @Override
        public boolean shouldSkip(Throwable t, long skipCount) {
            if (t instanceof InvalidIbanException || t instanceof ParseException) {
                if (skipCount > 100) {
                    log.error("Too many skips ({}); likely upstream issue", skipCount);
                    return false;
                }
                return true;
            }
            if (t instanceof BusinessValidationException) {
                return skipCount < 50;
            }
            return false;   // Fail fast on unknown
        }
    };
}

@Bean
public SkipListener<Transaction, Posting> skipListener() {
    return new SkipListenerSupport<>() {
        @Override
        public void onSkipInRead(Throwable t) {
            log.warn("Skipped record during read", t);
            skipMetrics.recordReadSkip();
        }
        
        @Override
        public void onSkipInProcess(Transaction tx, Throwable t) {
            log.warn("Skipped tx={} during process", tx.getId(), t);
            skipAuditRepo.save(SkipAudit.builder()
                .transactionId(tx.getId())
                .phase("PROCESS")
                .reason(t.getMessage())
                .occurredAt(Instant.now())
                .build());
        }
        
        @Override
        public void onSkipInWrite(Posting posting, Throwable t) {
            log.warn("Skipped posting during write", t);
            skipAuditRepo.save(SkipAudit.builder()
                .postingId(posting.getId())
                .phase("WRITE")
                .reason(t.getMessage())
                .build());
        }
    };
}
```

**Banking — audit critical:**
- Skipped record asla unutulmamalı
- Skip audit table (compliance + investigation)
- Daily skip count → metric + alert if > threshold

### 3. Retry — transient failures

```java
@Bean
public Step processWithRetry(JobRepository repo, PlatformTransactionManager tm) {
    return new StepBuilder("processWithRetry", repo)
        .<Transaction, Posting>chunk(100, tm)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .retryPolicy(bankingRetryPolicy())
        .retryLimit(3)
        .retry(DeadlockLoserDataAccessException.class)
        .retry(CannotAcquireLockException.class)
        .retry(TransientDataAccessException.class)
        .retry(JedisConnectionException.class)
        .backOffPolicy(exponentialBackoff())
        .listener(retryListener())
        .build();
}

@Bean
public RetryPolicy bankingRetryPolicy() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return policy;
}

@Bean
public BackOffPolicy exponentialBackoff() {
    ExponentialBackOffPolicy policy = new ExponentialBackOffPolicy();
    policy.setInitialInterval(500);          // 500 ms
    policy.setMaxInterval(10_000);           // max 10 sec
    policy.setMultiplier(2.0);
    return policy;
}
```

Retry attempt: 500ms → 1s → 2s → 4s ... cap 10s.

**Skip + Retry composition:**

```
1. Read item
2. Process item → exception
3. Retry up to N times
4. Still fails → check Skip policy
5. Skippable → skip + audit
6. Not skippable → propagate (Step fails)
```

### 4. Restartability — Spring Batch killer feature

```java
// JobLauncher
JobExecution execution = jobLauncher.run(eodJob, jobParameters);

// If terminated mid-flight (kill -9, pod evict):
// State stored in JobRepository (DB):
//  - BATCH_JOB_INSTANCE
//  - BATCH_JOB_EXECUTION (status=FAILED or STOPPED)
//  - BATCH_STEP_EXECUTION (read_count, write_count, ...)
//  - BATCH_EXECUTION_CONTEXT (chunk position, last processed ID)

// Restart same job (same parameters):
JobExecution restart = jobLauncher.run(eodJob, sameJobParameters);
// Spring Batch resumes from last committed chunk
```

**Key:**
- Same `JobParameters` (date + run_id semantically equivalent) → restart same instance
- New parameters → new instance

```java
JobParameters params = new JobParametersBuilder()
    .addString("eodDate", "2024-05-12")        // Identifying
    .addLong("runId", System.currentTimeMillis())   // Non-identifying (for re-run)
    .toJobParameters();
```

`runId` non-identifying — multiple executions of same instance.

### 5. ExecutionContext — checkpoint state

```java
@Bean
@StepScope
public ItemReader<Transaction> partitionedReader(
    @Value("#{stepExecutionContext['minId']}") Long minId,
    @Value("#{stepExecutionContext['maxId']}") Long maxId
) {
    JdbcCursorItemReader<Transaction> reader = new JdbcCursorItemReader<>();
    reader.setDataSource(ds);
    reader.setSql("SELECT * FROM transaction WHERE id BETWEEN ? AND ? ORDER BY id");
    reader.setPreparedStatementSetter(ps -> {
        ps.setLong(1, minId);
        ps.setLong(2, maxId);
    });
    reader.setRowMapper(new TransactionRowMapper());
    reader.setSaveState(true);    // ← KEY: persist position to ExecutionContext
    return reader;
}
```

`saveState(true)` → Spring Batch persists last read position after each chunk commit.

Restart:
1. Read state: last position = 12500
2. Skip first 12500 records (or position cursor)
3. Continue chunk processing

**ItemStream interface** — custom reader/writer that supports save/restore:

```java
public class BankingReader implements ItemStreamReader<Transaction> {
    
    private long position = 0;
    
    @Override
    public void open(ExecutionContext ctx) {
        if (ctx.containsKey("reader.position")) {
            this.position = ctx.getLong("reader.position");
        }
    }
    
    @Override
    public void update(ExecutionContext ctx) {
        ctx.putLong("reader.position", this.position);
    }
    
    @Override
    public Transaction read() {
        position++;
        return fetchNext();
    }
    
    @Override
    public void close() {}
}
```

### 6. Idempotency — restart-safe writer

Restart sırasında **same record written twice** olabilir. Writer idempotent olmalı.

```java
@Bean
public ItemWriter<Posting> idempotentWriter() {
    return chunk -> {
        for (Posting p : chunk) {
            // Use UPSERT veya idempotent INSERT
            jdbcTemplate.update("""
                INSERT INTO ledger_entry (id, account_id, debit, credit, currency, posted_at)
                VALUES (?, ?, ?, ?, ?, ?)
                ON CONFLICT (id) DO NOTHING
                """,
                p.getId(), p.getAccountId(), p.getDebit(), p.getCredit(), p.getCurrency(), p.getPostedAt());
        }
    };
}
```

Banking: deterministic ID (`hash(transactionId + accountSide)`) idempotency key.

### 7. Banking — restart senaryoları

#### Senaryo 1: Pod evict mid-job

```
T0: Job starts, processing 1M transactions
T1 (chunk 5000 of 10000): Pod evicted (node maintenance)
  - State persisted: chunk 5000 committed
  - JobExecution status: STOPPED

T2: Pod restarts, restart-trigger detects STOPPED job
  - Resume from chunk 5001
  - Same JobInstance, new JobExecution
  - Completion: full 10000 chunks done
```

#### Senaryo 2: DB transient failure

```
Chunk 500 processing:
  Read 100 items ✓
  Process 100 items ✓
  Write — DB connection lost!
  
Retry 1 (500ms backoff): Write — still failing
Retry 2 (1s backoff): Write succeeds
Continue to chunk 501
```

#### Senaryo 3: Bad data record

```
Chunk 100, record 56:
  Read → ParseException (corrupt CSV)
  Skip policy: ParseException skippable
  Audit log: skipped record details
  Continue with other 99 records
  
At job end:
  Skip count: 1
  Status: COMPLETED (with skips)
```

#### Senaryo 4: Job stuck — manual abort

```
Job running 6 hours, no progress
Ops investigates, decides to abort:
  
jobOperator.stop(executionId)
  → ChunkContext.getStepContext().setTerminateOnly()
  → After current chunk commit, step stops
  → Status: STOPPED (NOT FAILED)
  
Next morning: restart
  → Resume from last commit
```

### 8. Job parameters strategy

```java
public class BankingJobParameters {
    
    public static JobParameters forEodOnDate(LocalDate eodDate) {
        return new JobParametersBuilder()
            // Identifying — defines instance
            .addString("eodDate", eodDate.toString(), true)
            .addString("environment", System.getenv("ENV"), true)
            
            // Non-identifying — varies per run
            .addLong("runId", System.currentTimeMillis(), false)
            .addString("triggeredBy", "scheduler", false)
            
            .toJobParameters();
    }
}
```

**Banking pratiği:**
- `eodDate` identifying → same date = same instance (restart-able)
- `runId` non-identifying → multiple executions allowed
- Two runs same `eodDate` only if previous FAILED/STOPPED

### 9. Stop + restart workflow

```java
@Component
public class BankingJobOperator {
    
    private final JobOperator jobOperator;
    private final JobExplorer jobExplorer;
    
    public void stopRunning(String jobName) {
        Set<Long> runningExecutions = jobExplorer.findRunningJobExecutions(jobName)
            .stream().map(JobExecution::getId).collect(toSet());
        
        for (Long id : runningExecutions) {
            try {
                jobOperator.stop(id);   // Graceful stop
                log.info("Stop requested for execution {}", id);
            } catch (Exception e) {
                log.error("Failed to stop execution {}", id, e);
            }
        }
    }
    
    public Long restart(Long previousExecutionId) throws Exception {
        return jobOperator.restart(previousExecutionId);
    }
    
    @Scheduled(cron = "0 0 5 * * *")   // 05:00 daily
    public void resumeStalledJobs() {
        // Restart any FAILED/STOPPED job from yesterday
        LocalDate yesterday = LocalDate.now().minusDays(1);
        Set<JobExecution> failed = jobExplorer.findJobExecutionsByStatus(
            "eodJob", BatchStatus.FAILED);
        
        for (JobExecution exec : failed) {
            LocalDate execDate = LocalDate.parse(
                exec.getJobParameters().getString("eodDate"));
            if (execDate.equals(yesterday)) {
                log.warn("Resuming failed job execution {}", exec.getId());
                jobOperator.restart(exec.getId());
            }
        }
    }
}
```

### 10. Skip + Retry composition order

```java
.faultTolerant()
.retryLimit(3)
.retry(TransientDataAccessException.class)
.skipLimit(100)
.skip(InvalidIbanException.class)
.skip(ParseException.class)
```

Spring Batch order:
1. Item read → exception
2. Is exception retryable? → retry (within limit)
3. After retries exhausted → is exception skippable? → skip + audit
4. Not skippable → step FAILED

### 11. Banking — restart anti-pattern'leri

**Anti-pattern 1: Non-deterministic IDs**

```java
post.setId(UUID.randomUUID());   // ❌
```

Restart writes same logical record with new UUID → duplicate.

**Fix:** `hash(transactionId + side)` deterministic.

**Anti-pattern 2: saveState(false)**

Reader doesn't save state → restart re-reads from beginning.

**Anti-pattern 3: External side effects in processor**

```java
emailService.sendNotification(tx);   // ❌
```

Restart resends emails to same customer. Side effect → externalize to outbox + dedupe.

**Anti-pattern 4: Writer not idempotent**

```sql
INSERT INTO ledger ...
```

Restart → duplicate. UPSERT or ON CONFLICT DO NOTHING.

**Anti-pattern 5: Skip without audit**

Skipped records lost. Banking için audit critical.

**Anti-pattern 6: Skip limit too high**

`skipLimit(1_000_000)` → all bad data skipped silently. Reasonable threshold.

**Anti-pattern 7: Retry on non-transient**

```java
.retry(NullPointerException.class)
```

NPE = bug. Retry pointless.

**Anti-pattern 8: No backoff policy**

```java
.retry(...)   // tight loop retry
```

DB overwhelmed. Exponential backoff.

**Anti-pattern 9: identifying parameter wrong**

```java
.addLong("runId", System.currentTimeMillis(), true)   // ❌ identifying
```

Every run = new instance. Restart impossible.

**Anti-pattern 10: Restart after data fix without re-running step**

Sometimes restart not appropriate; full re-run safer (banking idempotency-safe).

---

## Önemli olabilecek araştırma kaynakları

- Spring Batch reference — Skip + Retry + Restart
- "Pro Spring Batch" — Michael Minella
- Spring Retry library
- Banking EOD operations patterns

---

## Mini task'ler

### Task 5.3.1 — SkipPolicy + listener (60 dk)

Bad CSV data 5%. Skip ParseException. SkipListener audit table.

### Task 5.3.2 — RetryPolicy exponential backoff (45 dk)

Mock transient DB failure. 3 retry, 500ms initial backoff.

### Task 5.3.3 — Restartable reader (60 dk)

Job 10k record processing. Kill after 5k. Restart resumes from 5001.

### Task 5.3.4 — Idempotent writer (45 dk)

Deterministic ID. Restart twice. No duplicates.

### Task 5.3.5 — Banking EOD job senaryo (90 dk)

50k transactions. Inject:
- 1% parse error → skip
- 0.1% DB transient → retry → success
- Mid-job pod evict → restart succeeds

### Task 5.3.6 — Job stop + manual restart (45 dk)

`jobOperator.stop()` + verify graceful. `restart()` continue.

### Task 5.3.7 — Skip audit + alert (45 dk)

Skipped record audit table. Daily skip count > threshold → alert.

### Task 5.3.8 — Identifying vs non-identifying params (30 dk)

Same `eodDate` two runs (one FAILED). Restart-able. New `eodDate` different instance.

---

## Test yazma rehberi

```java
@SpringBatchTest
@SpringBootTest
class SkipRetryRestartTest {
    
    @Autowired JobLauncherTestUtils launcher;
    @Autowired JobRepository jobRepo;
    
    @Test
    void shouldSkipParseError() throws Exception {
        seedDataWithParseErrors(100, 5);   // 100 records, 5 bad
        
        JobExecution exec = launcher.launchJob();
        
        StepExecution step = exec.getStepExecutions().iterator().next();
        assertThat(step.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        assertThat(step.getReadCount()).isEqualTo(100);
        assertThat(step.getProcessSkipCount()).isEqualTo(5);
        assertThat(step.getWriteCount()).isEqualTo(95);
    }
    
    @Test
    void shouldRetryThenSucceedOnTransientFailure() throws Exception {
        AtomicInteger callCount = new AtomicInteger();
        when(externalService.call(any())).thenAnswer(inv -> {
            int count = callCount.incrementAndGet();
            if (count < 3) throw new TransientDataAccessException("transient");
            return success();
        });
        
        JobExecution exec = launcher.launchJob();
        
        assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        verify(externalService, times(3)).call(any());
    }
    
    @Test
    void shouldRestartFromLastCommittedChunk() throws Exception {
        seedData(10_000);
        
        JobExecution failed = launcher.launchJob();
        // Force fail mid-job
        killAtChunk(50);
        
        assertThat(failed.getStatus()).isEqualTo(BatchStatus.FAILED);
        long itemsProcessed = failed.getStepExecutions().iterator().next().getWriteCount();
        
        // Restart
        JobExecution restart = launcher.launchJob();
        
        long totalProcessed = restart.getStepExecutions().iterator().next().getWriteCount();
        assertThat(totalProcessed).isEqualTo(10_000 - itemsProcessed);
        
        // No duplicates
        long dbCount = jdbcTemplate.queryForObject("SELECT COUNT(*) FROM ledger_entry", Long.class);
        assertThat(dbCount).isEqualTo(10_000);
    }
    
    @Test
    void shouldNotRestartCompletedInstance() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("eodDate", "2024-05-12", true)
            .toJobParameters();
        
        JobExecution first = launcher.launchJob(params);
        assertThat(first.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        
        assertThatThrownBy(() -> launcher.launchJob(params))
            .isInstanceOf(JobInstanceAlreadyCompleteException.class);
    }
}
```

---

## Claude-verify prompt

```
Skip+Retry+Restart setup'ımı banking-grade kriterlere göre değerlendir:

1. Skip:
   - SkipPolicy specific exception types?
   - skipLimit reasonable (banking: 100-1000)?
   - SkipListener audit table?
   - Daily skip threshold alert?

2. Retry:
   - retry only TransientDataAccessException etc?
   - retryLimit 3-5?
   - Exponential backoff (initial 500ms, multiplier 2)?
   - No retry on NPE / non-transient?

3. Restart:
   - identifying eodDate parameter?
   - runId non-identifying for retry?
   - Reader saveState(true)?
   - ExecutionContext checkpoint?

4. Idempotency:
   - Deterministic ID (no random)?
   - UPSERT / ON CONFLICT DO NOTHING?
   - No external side effects in processor (or outbox + dedupe)?

5. Banking EOD:
   - Skip audit table + retention?
   - Daily skip metric?
   - Pod evict recovery automated?
   - Failed job re-run scheduler (05:00 morning)?

6. Listeners:
   - onSkipInRead/Process/Write captured?
   - RetryListener attempt logged?
   - JobExecutionListener metrics?

7. Composition:
   - Retry first, then skip order?
   - skipPolicy + retryPolicy both configured?

8. Operations:
   - JobOperator.stop() graceful?
   - JobOperator.restart() resume?
   - jobExplorer.findRunningJobExecutions?

9. Audit + compliance:
   - Skipped record investigation possible?
   - Restart audit trail?
   - Banking BDDK regulatory?

10. Anti-pattern:
    - Non-deterministic ID YOK?
    - saveState(false) YOK?
    - Side effects in processor YOK?
    - Non-idempotent writer YOK?
    - Skip without audit YOK?
    - Retry on bug YOK?
    - No backoff YOK?
    - Wrong identifying params YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Skip policy + audit table
- [ ] Retry policy + exponential backoff
- [ ] Restartable reader + saveState
- [ ] Idempotent writer (UPSERT)
- [ ] ExecutionContext checkpoint
- [ ] Banking EOD restart senaryo
- [ ] Daily skip threshold alert
- [ ] Stop + restart workflow
- [ ] Identifying vs non-identifying params
- [ ] 6+ integration test

---

## Defter notları (10 madde)

1. "Banking batch failure modes (data quality / transient / business / bug / crash): ____."
2. "SkipPolicy specific exception + audit table + threshold alert: ____."
3. "RetryPolicy + exponential backoff + transient-only banking pattern: ____."
4. "Restartability ExecutionContext + saveState + checkpoint: ____."
5. "Identifying (eodDate) vs non-identifying (runId) JobParameters: ____."
6. "Idempotent writer UPSERT / ON CONFLICT DO NOTHING banking: ____."
7. "Composition order: Retry first → Skip → propagate: ____."
8. "JobOperator stop/restart workflow + 05:00 morning recovery: ____."
9. "External side effect (email) externalize via outbox + dedupe: ____."
10. "Anti-pattern (random ID, no saveState, no audit, retry NPE): ____."
