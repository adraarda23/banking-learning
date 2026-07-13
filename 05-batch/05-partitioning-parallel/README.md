# Topic 5.5 — Partitioning & Parallel Execution

## Hedef

Spring Batch büyük dataset paralel işleme stratejileri banking-grade derinlikte: local partitioning, remote partitioning, multi-threaded step, parallel step, banking 1M müşteri 10 partition paralel faiz hesabı, throughput optimization, ShedLock cluster coordination, monitoring.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 5.1-5.4 bitti
- Phase 3 (concurrency) — thread pool
- Postgres partition queries

---

## Kavramlar

### 1. 4 parallel strategy

| Strategy | Anlam | Banking use |
|---|---|---|
| **Single-threaded chunk** | Default | Small datasets (< 10k) |
| **Multi-threaded step** | Multiple threads, single reader | Medium throughput (10k-100k) |
| **Parallel steps** | Independent steps concurrent | Independent operations |
| **Partitioning (master-slave)** | Divide data → parallel workers | Large datasets (100k-100M+) |

### 2. Multi-threaded step — simple parallelism

```java
@Bean
public Step multiThreadedStep(JobRepository repo, PlatformTransactionManager tm) {
    return new StepBuilder("processTransactions", repo)
        .<Transaction, Posting>chunk(100, tm)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .taskExecutor(taskExecutor())   // ← multi-threaded
        .throttleLimit(8)
        .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(8);
    exec.setMaxPoolSize(16);
    exec.setQueueCapacity(50);
    exec.setThreadNamePrefix("batch-worker-");
    exec.initialize();
    return exec;
}
```

**Critical constraints:**
- Reader **thread-safe** (most JpaPagingReader = NOT thread-safe!)
- Writer thread-safe (JdbcBatchItemWriter OK)
- Processor stateless

**SynchronizedItemStreamReader** wrap for non-thread-safe reader:
```java
SynchronizedItemStreamReader<Transaction> safeReader = new SynchronizedItemStreamReader<>();
safeReader.setDelegate(originalReader);
```

Banking trade-off: easy speedup, but read becomes bottleneck.

### 3. Local partitioning — divide and conquer

```
Master Step (orchestrator)
  ↓ partition
[Worker 1: range 1-10000]   ─ executes Worker Step
[Worker 2: range 10001-20000]
[Worker 3: range 20001-30000]
...
[Worker 10: range 90001-100000]
  ↓
Master Step: aggregate results
```

Each worker has **own reader/writer** → no synchronization needed.

```java
@Bean
public Step masterStep(JobRepository repo, Step workerStep, PartitionHandler handler) {
    return new StepBuilder("masterStep", repo)
        .partitioner("worker", partitioner())
        .step(workerStep)
        .partitionHandler(handler)
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        long min = customerRepo.minId();
        long max = customerRepo.maxId();
        long range = (max - min) / gridSize;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (range * i));
            ctx.putLong("maxId", min + (range * (i + 1)) - 1);
            ctx.putInt("partitionIndex", i);
            partitions.put("partition-" + i, ctx);
        }
        // Last partition adjusts for remainder
        partitions.get("partition-" + (gridSize - 1))
            .putLong("maxId", max);
        return partitions;
    };
}

@Bean
public PartitionHandler partitionHandler(Step workerStep, TaskExecutor taskExecutor) {
    TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
    handler.setStep(workerStep);
    handler.setGridSize(10);
    handler.setTaskExecutor(taskExecutor);
    return handler;
}

@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tm) {
    return new StepBuilder("workerStep", repo)
        .<Customer, Accrual>chunk(500, tm)
        .reader(partitionReader(null, null))   // injected per partition
        .processor(processor())
        .writer(writer())
        .build();
}

@Bean
@StepScope
public JdbcCursorItemReader<Customer> partitionReader(
    @Value("#{stepExecutionContext['minId']}") Long minId,
    @Value("#{stepExecutionContext['maxId']}") Long maxId
) {
    JdbcCursorItemReader<Customer> reader = new JdbcCursorItemReader<>();
    reader.setDataSource(ds);
    reader.setSql("SELECT * FROM customer WHERE id BETWEEN ? AND ? ORDER BY id");
    reader.setPreparedStatementSetter(ps -> {
        ps.setLong(1, minId);
        ps.setLong(2, maxId);
    });
    reader.setRowMapper(new CustomerRowMapper());
    return reader;
}
```

**Banking — interest accrual 1M customer:**
- Single thread: ~10 hours
- 10 partitions parallel: ~1 hour
- 20 partitions parallel: ~30 minutes (DB I/O cap)

### 4. Range partitioning — banking strategies

**Strategy 1: ID range** (above)
- Simple, even distribution
- Good for uniform tables

**Strategy 2: Customer segment**
```java
partitioner = gridSize -> Map.of(
    "premium", contextFor("premium"),
    "retail", contextFor("retail"),
    "corporate", contextFor("corporate"));
```
Domain-driven, naturally aligned.

**Strategy 3: Geographic / Branch**
```java
partitioner = gridSize -> branches.stream()
    .collect(toMap(b -> "branch-" + b.getCode(), b -> contextFor(b)));
```

**Strategy 4: Date range**
```java
// 30-day backfill — 1 partition per day
for (int day = 0; day < 30; day++) {
    ctx.putString("processingDate", today.minusDays(day).toString());
    partitions.put("day-" + day, ctx);
}
```

### 5. Remote partitioning — distributed

Workers in different JVMs / pods. Message broker (Kafka, RabbitMQ).

```
Master (Pod 1):
  ↓ Send partition messages to Kafka
[Kafka topic: batch-partitions]
  ↑ Consume + execute
Worker (Pod 2, 3, 4, ...)
  ↓ Reply with result
[Kafka topic: batch-results]
  ↑
Master
```

```java
@Bean
public MessageChannelPartitionHandler partitionHandler(
    MessagingTemplate messagingTemplate
) {
    MessageChannelPartitionHandler handler = new MessageChannelPartitionHandler();
    handler.setMessagingOperations(messagingTemplate);
    handler.setStepName("workerStep");
    handler.setGridSize(20);   // 20 worker pods
    handler.setPollInterval(10_000);
    return handler;
}
```

Spring Integration / Spring Cloud Stream for messaging.

Banking için: 1M+ records, multi-pod K8s cluster — remote partitioning powerful.

### 6. ShedLock — cluster coordination

K8s multi-pod batch:
- Master starts on pod 1
- Worker steps distributed
- BUT: only ONE master should start (avoid duplicate)

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
</dependency>
```

```java
@Component
public class BankingJobScheduler {
    
    @Scheduled(cron = "0 0 23 * * *")
    @SchedulerLock(name = "eodMasterJob", lockAtMostFor = "PT6H", lockAtLeastFor = "PT30M")
    public void runEodMasterJob() throws Exception {
        jobLauncher.run(eodMasterJob, BankingJobParameters.forEodOnDate(LocalDate.now()));
    }
}
```

`lockAtMostFor=6h` → other pods skip if first pod still running.

### 7. Banking — partitioning örnekleri

#### Örnek 1: 1M customer interest accrual

```java
@Bean
public Job interestAccrualJob(JobRepository repo) {
    return new JobBuilder("interestAccrualJob", repo)
        .start(masterStep())
        .build();
}

@Bean
public Step masterStep(JobRepository repo) {
    return new StepBuilder("masterStep", repo)
        .partitioner("worker", customerPartitioner())
        .step(workerStep())
        .gridSize(10)
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner customerPartitioner() {
    return gridSize -> {
        long min = customerRepo.minId();
        long max = customerRepo.maxId();
        long total = max - min + 1;
        long perPartition = (total + gridSize - 1) / gridSize;
        
        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (i * perPartition));
            ctx.putLong("maxId", Math.min(min + ((i + 1) * perPartition) - 1, max));
            partitions.put("partition-" + i, ctx);
        }
        return partitions;
    };
}
```

#### Örnek 2: Multi-branch reconciliation

```java
@Bean
public Partitioner branchPartitioner(BranchRepository branchRepo) {
    return gridSize -> {
        List<String> branches = branchRepo.findAllActiveBranchCodes();
        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (String branchCode : branches) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putString("branchCode", branchCode);
            partitions.put("branch-" + branchCode, ctx);
        }
        return partitions;
    };
}
```

Each worker reconciles one branch's daily transactions.

#### Örnek 3: 30-day backfill

```java
@Bean
public Partitioner backfillPartitioner() {
    return gridSize -> {
        LocalDate start = LocalDate.now().minusDays(30);
        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < 30; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putString("date", start.plusDays(i).toString());
            partitions.put("day-" + i, ctx);
        }
        return partitions;
    };
}
```

### 8. Throughput tuning

```
Single-threaded:
  100 records/sec
  → 1M records = 10000 sec = 2.78 hours

Multi-threaded step (8 threads):
  600 records/sec (synchronized reader = bottleneck)
  → 1M = 1667 sec = 28 min

10-partition local:
  900 records/sec
  → 1M = 1111 sec = 18 min

20-partition remote (10 pods):
  4000 records/sec
  → 1M = 250 sec = 4 min
```

**Bottlenecks:**
- DB connection pool (HikariCP size)
- DB IOPS (storage)
- Network (worker ↔ DB)
- CPU (processor-bound)

### 9. Partitioning monitoring

```java
@Bean
public StepExecutionListener partitionListener() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution exec) {
            log.info("Partition {} finished: {} records in {}",
                exec.getStepName(), exec.getWriteCount(),
                Duration.between(exec.getStartTime(), exec.getEndTime()));
            
            meterRegistry.timer("banking.batch.partition.duration",
                "step", exec.getStepName())
                .record(Duration.between(exec.getStartTime(), exec.getEndTime()));
            
            return exec.getExitStatus();
        }
    };
}
```

Grafana dashboard:
- Per-partition duration
- Throughput per partition
- Skip / retry counts

Spot **stragglers** (one partition slow) → repartition strategy adjust.

### 10. Banking — partitioning anti-pattern'leri

**Anti-pattern 1: Unbalanced partitions**

```java
// Partition 1: 100k records
// Partition 2: 5 records
// Partition 3: 1M records   ← straggler
```

ID range yanılır if hash distribution uneven. Domain-aware partition.

**Anti-pattern 2: Non-thread-safe reader without sync**

```java
.taskExecutor(taskExecutor)
.reader(jpaPagingItemReader())   // NOT thread-safe!
```

SynchronizedItemStreamReader wrap or use partitioning.

**Anti-pattern 3: Shared mutable state in processor**

```java
public class StatefulProcessor implements ItemProcessor<...> {
    private int counter = 0;   // ❌ race condition
}
```

Stateless processor.

**Anti-pattern 4: Partition writer contention**

10 partitions all writing same table same row → lock contention. Distribute by primary key range matching.

**Anti-pattern 5: gridSize too high**

`gridSize=1000` → DB connection pool exhausted. Banking realistic: 5-50.

**Anti-pattern 6: Master step heavy work**

Master should only partition + dispatch. Heavy work in workers.

**Anti-pattern 7: No ShedLock**

K8s multi-pod scheduler → duplicate job triggers.

**Anti-pattern 8: Worker timeout misconfig**

Slow partition → master times out → false failure. `pollInterval` + `timeout` tune.

**Anti-pattern 9: No retry on remote**

Network blip → partition fails. Retry policy for remote partition messages.

**Anti-pattern 10: Distributed without monitoring**

Partition stuck → no observability. Each worker step metric + heartbeat.

---

## Önemli olabilecek araştırma kaynakları

- Spring Batch reference — partitioning
- "Pro Spring Batch" — Michael Minella
- Spring Cloud Task / Stream for remote
- ShedLock GitHub

---

## Mini task'ler

### Task 5.5.1 — Multi-threaded step (45 dk)

8-thread chunk. SynchronizedItemStreamReader. Throughput compare to single.

### Task 5.5.2 — Local partitioning ID range (60 dk)

10 partition customer interest accrual. ID range divide.

### Task 5.5.3 — Domain partitioning (45 dk)

Branch-based partition. Each worker = one branch.

### Task 5.5.4 — Date backfill partitioning (45 dk)

30-day backfill, 1 partition per day.

### Task 5.5.5 — Remote partitioning Kafka (90 dk)

MessageChannelPartitionHandler. 2 worker pods.

### Task 5.5.6 — ShedLock cluster coordination (45 dk)

@SchedulerLock annotation. 2 instance test single-execution.

### Task 5.5.7 — Throughput benchmark (60 dk)

1, 2, 4, 8, 16 partition compare. Bottleneck identify (DB / CPU / I/O).

### Task 5.5.8 — Straggler detection (45 dk)

Skewed partition. Monitoring metric reveals slow partition.

---

## Test yazma rehberi

```java
@SpringBatchTest
class PartitioningTest {
    
    @Test
    void partitionedJobSplitsWorkEvenly() {
        seedCustomers(10_000);
        
        JobExecution exec = launcher.launchJob();
        
        assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        
        // 10 partitions = 10 worker step executions
        List<StepExecution> workerExecs = exec.getStepExecutions().stream()
            .filter(s -> s.getStepName().startsWith("worker:"))
            .toList();
        assertThat(workerExecs).hasSize(10);
        
        // Sum read counts = total
        long totalRead = workerExecs.stream().mapToLong(StepExecution::getReadCount).sum();
        assertThat(totalRead).isEqualTo(10_000);
    }
    
    @Test
    void parallelFasterThanSerial() {
        seedCustomers(50_000);
        
        long serialStart = System.currentTimeMillis();
        launcher.launchJob(BatchStatus.serialJob);
        long serialTime = System.currentTimeMillis() - serialStart;
        
        long parallelStart = System.currentTimeMillis();
        launcher.launchJob(BatchStatus.partitionedJob);
        long parallelTime = System.currentTimeMillis() - parallelStart;
        
        assertThat(parallelTime).isLessThan(serialTime / 2);
    }
    
    @Test
    void shedLockPreventsDuplicateExecution() throws Exception {
        AtomicInteger actualRuns = new AtomicInteger();
        
        ExecutorService pool = Executors.newFixedThreadPool(2);
        pool.submit(() -> {
            scheduler.runEodMasterJob();
            actualRuns.incrementAndGet();
        });
        pool.submit(() -> {
            scheduler.runEodMasterJob();   // Should be skipped
            actualRuns.incrementAndGet();
        });
        pool.shutdown();
        pool.awaitTermination(60, TimeUnit.SECONDS);
        
        assertThat(actualRuns.get()).isEqualTo(1);
    }
}
```

---

## Claude-verify prompt

```
Partitioning + parallel setup'ımı banking-grade kriterlere göre değerlendir:

1. Strategy selection:
   - Dataset size + partitioning rationale?
   - Multi-threaded vs partitioning trade-off?
   - Local vs remote?

2. Local partitioning:
   - Partitioner generates ExecutionContext per partition?
   - @StepScope reader with parameter injection?
   - PartitionHandler gridSize realistic?

3. Partition strategies banking:
   - ID range?
   - Customer segment?
   - Branch?
   - Date range backfill?

4. Reader thread safety:
   - Reader stateless?
   - SynchronizedItemStreamReader wrap if needed?
   - Per-partition reader instance?

5. Cluster coordination:
   - ShedLock @SchedulerLock?
   - lockAtMostFor reasonable?
   - K8s multi-pod tested?

6. Throughput:
   - Per-partition metric?
   - Bottleneck identified (DB / CPU / I/O)?
   - Connection pool size adequate?

7. Monitoring:
   - Per-partition duration metric?
   - Straggler detection?
   - Grafana dashboard?

8. Remote (if applicable):
   - Spring Cloud Stream / Kafka?
   - Message channel partition handler?
   - Reply timeout config?

9. Resilience:
   - Partition retry on transient?
   - Failed partition resume?
   - Master timeout handling?

10. Anti-pattern:
    - Unbalanced partition YOK?
    - Non-thread-safe reader without sync YOK?
    - Shared state in processor YOK?
    - gridSize 1000+ YOK?
    - Heavy work in master YOK?
    - No ShedLock cluster YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Multi-threaded step + SynchronizedItemStreamReader
- [ ] Local partitioning ID range
- [ ] Domain partitioning (branch / segment)
- [ ] Date backfill partitioning
- [ ] Remote partitioning Kafka
- [ ] ShedLock cluster coordination
- [ ] Throughput benchmark gridSize sweep
- [ ] Per-partition monitoring
- [ ] Straggler detection
- [ ] 6+ integration test

---

## Defter notları (10 madde)

1. "4 parallel strategy (single / multi-threaded / parallel steps / partitioning) banking selection: ____."
2. "Multi-threaded step + SynchronizedItemStreamReader thread safety: ____."
3. "Local partitioning master-slave + ExecutionContext per partition: ____."
4. "Partition strategy (ID range / customer segment / branch / date) banking domain: ____."
5. "Remote partitioning Kafka MessageChannelPartitionHandler K8s multi-pod: ____."
6. "ShedLock @SchedulerLock cluster duplicate-prevention: ____."
7. "Throughput single → partitioned 10x speedup banking 1M customer accrual: ____."
8. "Bottleneck identification (DB pool + IOPS + CPU + network): ____."
9. "Straggler detection per-partition metric + repartition adjust: ____."
10. "Anti-pattern (unbalanced + non-thread-safe reader + shared state + no ShedLock): ____."
