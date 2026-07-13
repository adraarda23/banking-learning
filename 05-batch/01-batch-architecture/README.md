# Topic 5.1 — Spring Batch Mimarisi: Job, Step, JobRepository

## Hedef

Spring Batch'in **temel yapı taşlarını** anlamak: `Job`, `Step`, `JobInstance`, `JobExecution`, `StepExecution`, `JobRepository`, `JobLauncher`, `JobOperator`, `JobParameters`. Bir TR bankasındaki "EOD job'u" kavramının arkasında yatan teknik makinayı görmek.

Bu topic'te **kod yazma odağı zayıf** — kavramları ezberlersin, sonraki topic'lerde uygulayacaksın. Ama buradaki bilgi olmadan diğer topic'ler havada kalır.

## Süre

Okuma: 2 saat • Mini task: 1.5 saat • Test: 30 dk • Toplam: ~4 saat

## Önbilgi

- Faz 1-4 tamamlandı (`core-banking` çalışıyor, PostgreSQL bağlı, JPA biliyorsun)
- "Cron job" kavramına aşinasın (Linux'ta `cron -e` görmüşsen yeter)
- Online API endpoint'i ile batch job'un farkını sezgisel biliyorsun

---

## Kavramlar

### 1. Spring Batch nedir, ne değildir?

**Spring Batch** = "büyük hacimli, tekrar eden, deterministik veri işleme job'ları için" framework. 2008'de Accenture + SpringSource ortak çıkardı. **Kaynak:** Finansal kurumlarda EOD/EOM/EOY job'ları için tasarlandı. Bu yüzden DNA'sı banking'e uygun.

**Spring Batch'i seç:**

- Periyodik (gece/ay sonu/yıl sonu) toplu işleme
- Çok büyük veri (DB tablosu, CSV/Excel dosya, XML, JSON)
- Resilient: hata olursa kaldığı yerden devam
- Auditable: ne zaman çalıştı, ne kadar sürdü, ne sonuç verdi
- Long-running: dakikalar / saatler süren işler
- Transaction discipline'i kritik

**Spring Batch'i seçme:**

- Real-time stream processing (Kafka Streams / Flink)
- Saniye altı low-latency API
- Tek seferlik küçük script ("100 müşteriye email at" — `for` loop yazıver)
- Reactive event-driven workflow (Camel / temporal.io)
- Cron + bash script ile çözülebilen 5 dakikalık iş

**Banking örnek karşılaştırması:**

| İş | Spring Batch? | Sebep |
|----|---------------|-------|
| Müşterinin EFT'sini gerçek zamanlı yapma | Hayır | Online, sub-second; REST + sync TX |
| 10M müşteri için günlük faiz tahakkuku | Evet | Büyük, periyodik, restartable lazım |
| Para çekiminde anti-fraud kontrolü | Hayır | Real-time, request-scoped |
| Gün sonu MASAK STR raporu üretme | Evet | EOD, dosya output, restart önemli |
| Müşteriye SMS atma | Hayır | Async messaging (Kafka/RabbitMQ) yeterli |
| Aylık ekstre PDF üretimi | Evet | Toplu, idempotent, restartable |

### 2. Domain modeli: Job, Step, JobInstance, JobExecution

Spring Batch'in **5 ana kavramı** var — bunları kafanda kesin ayırmadan ilerleme.

#### 2.1 Job

**Tanım:** Bir batch işin **tasarımı / template'i**. "Günsonu mutabakat işi" gibi bir konsepttir.

```java
@Bean
public Job eodReconciliationJob(JobRepository jobRepository, Step reconcileStep) {
    return new JobBuilder("eodReconciliationJob", jobRepository)
        .start(reconcileStep)
        .build();
}
```

`Job` bir **bean**, uygulama başlangıcında tanımlanır. Bir tane Job tanımı — sonsuz kez **çalıştırılır**.

**Önemli:** Job objesinin kendisinde state yoktur. Bir `Job`'un her "çalıştırılışı" yeni bir `JobExecution` üretir. Job'un kendisi reusable bir blueprint.

#### 2.2 JobInstance

**Tanım:** Bir Job'un belirli **JobParameters seti ile mantıksal çalıştırılışı**.

İlginç kısım: `JobInstance` **JobParameters'a göre tanımlanır**. Aynı Job + aynı JobParameters → **aynı** JobInstance (DB'de aynı row). Aynı Job + farklı JobParameters → **farklı** JobInstance.

Banking örneği:

```
Job: "eodReconciliationJob"

JobInstance 1: eodReconciliationJob + {businessDate=2025-05-10}
JobInstance 2: eodReconciliationJob + {businessDate=2025-05-11}  ← farklı parametre, farklı instance
JobInstance 3: eodReconciliationJob + {businessDate=2025-05-12}
```

**Niye önemli?** Spring Batch "aynı JobInstance'ı **başarıyla** bir kez çalıştırırsın" der. 2025-05-10 mutabakatını başarıyla çalıştırdıysan, tekrar çalıştıramazsın (default davranış). Bu, **idempotency garantisi** sağlar — "Ya iki kez çalıştırırsam?" derdini ortadan kaldırır.

Tekrar çalıştırmaya çalışırsan: `JobInstanceAlreadyCompleteException`.

#### 2.3 JobExecution

**Tanım:** Bir JobInstance'ı çalıştırma **denemesi**. Bir JobInstance birden fazla JobExecution'a sahip olabilir — ilki fail oldu, ikincide restart edip başarılı bitirdiyseniz.

```
JobInstance: eodReconciliationJob + {businessDate=2025-05-10}
    JobExecution 1: 23:30 başladı → 23:42 FAILED (DB connection drop)
    JobExecution 2: 23:45 restart edildi → 23:55 COMPLETED ✓
```

`JobExecution`'da bulunan bilgiler:

- `id` (DB'de primary key)
- `jobInstance` (hangi instance'a ait)
- `status` (BatchStatus: STARTING, STARTED, COMPLETED, FAILED, STOPPED, ABANDONED)
- `exitStatus` (ExitStatus: COMPLETED, FAILED, NOOP, custom)
- `startTime`, `endTime`, `createTime`
- `executionContext` (key-value store, restart için)
- `failureExceptions` (eğer fail olduysa)
- `stepExecutions` (alt-step execution'ları)

**BatchStatus vs ExitStatus farkı:**

- `BatchStatus`: enum, framework'ün gözünden state. STARTED, COMPLETED, FAILED, vb.
- `ExitStatus`: framework dışı tüketim için string. "COMPLETED", "FAILED", veya custom: "COMPLETED_WITH_SKIPS", "DATA_MISSING".

Conditional flow'da (Topic 4) `ExitStatus` kullanılır — "step bittiyse ama exit status NOT_FOUND ise farklı yola git" gibi.

#### 2.4 Step

**Tanım:** Job'un içindeki **bağımsız faz**. Bir Job N tane Step içerir; sırayla veya conditional/paralel çalışır.

```java
@Bean
public Job dailyEodJob(JobRepository jobRepository,
                       Step reconcileStep,
                       Step accrueInterestStep,
                       Step generateReportStep) {
    return new JobBuilder("dailyEodJob", jobRepository)
        .start(reconcileStep)
        .next(accrueInterestStep)
        .next(generateReportStep)
        .build();
}
```

Step'in iki ana türü:

1. **Tasklet step**: Tek bir method (`execute`) çağrılır. Bir kez çalışır, bir TX içinde bir şey yapar. (Topic 2'de detay)
2. **Chunk-oriented step**: Reader → Processor → Writer döngüsü. Bigdata için. (Topic 2 ana konu)

Banking örneği:

```
Job: monthlyInterestAccrualJob
  Step 1 (tasklet): "lock all accounts" → bir SQL update
  Step 2 (chunk): "read customers, calculate interest, write postings" → milyonlar
  Step 3 (chunk): "read interest postings, send notifications" → milyonlar
  Step 4 (tasklet): "unlock all accounts" → bir SQL update
```

#### 2.5 StepExecution

**Tanım:** Bir Step'in bir JobExecution içindeki çalıştırılışı.

`StepExecution`'da:

- `id`
- `jobExecution` (parent)
- `stepName`
- `status` (BatchStatus)
- `exitStatus` (ExitStatus)
- `readCount`, `writeCount`, `commitCount`, `rollbackCount`, `skipCount` (chunk metrics)
- `filterCount`, `readSkipCount`, `processSkipCount`, `writeSkipCount`
- `startTime`, `endTime`
- `executionContext` (step-level, restart için)

Bu sayaçlar **DB'de persist edilir**. `BATCH_STEP_EXECUTION` tablosunu sorgulayarak "dün gece reconciliation step'i kaç satır okudu, kaç satır yazdı, kaç skip etti" sorusuna cevap alırsın. Banking audit'inde kritik.

---

### 3. JobRepository — state'in tutulduğu yer

`JobRepository`, **batch state'inin persistence katmanı**. Spring Batch çalışırken her şey buraya yazılır:

- Job'un başladığı an
- Step'in commit ettiği her chunk
- Skip edilen item'lar
- ExecutionContext (restart için checkpoint)
- Final status

**Default:** Spring Batch ilk başladığında, configure edilmiş `DataSource`'ta `BATCH_*` tablolarını arar. Bulamazsa schema migration ile oluşturulması gerekir.

**Banking projende** PostgreSQL kullanıyorsun. Spring Batch'in PostgreSQL schema'sı:

```sql
-- Sadece tabloların adları, full schema sonra Flyway ile.
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS
BATCH_JOB_EXECUTION_CONTEXT
BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
BATCH_JOB_SEQ
BATCH_JOB_EXECUTION_SEQ
BATCH_STEP_EXECUTION_SEQ
```

Spring Boot Batch starter ile gelen schema dosyası: `org/springframework/batch/core/schema-postgresql.sql` (jar'ın içinden çıkarılır). Production'da bunu **Flyway migration olarak** kendi `db/migration/V5__add_batch_tables.sql` dosyana koyarsın — Spring Batch'in `spring.batch.jdbc.initialize-schema=always` ile otomatik kurmasına bırakmazsın (banking'de tüm DDL kontrolü Flyway'de olmalı).

**Spring Boot config (önemli):**

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never   # production'da NEVER. Flyway kursun.
    job:
      enabled: false             # app start ettiğinde job'ları auto-run ETME
                                 # senin JobLauncher / @Scheduled tetiklesin
```

**Neden `spring.batch.job.enabled=false`?** Default'ta Spring Boot başladığında classpath'teki tüm `Job` bean'lerini otomatik çalıştırmaya kalkar. Banking'de **istenmeyen** bir davranıştır: app her restart'ta job tetiklemek istemezsin; scheduling ayrı bir konu (Topic 6).

#### 3.1 BATCH_JOB_INSTANCE tablosu

```
JOB_INSTANCE_ID | VERSION | JOB_NAME              | JOB_KEY (hash of params)
----------------+---------+-----------------------+------------------------
1               | 0       | eodReconciliationJob  | a4f9c2...
2               | 0       | eodReconciliationJob  | b5g0d3...
3               | 0       | interestAccrualJob    | c6h1e4...
```

`JOB_KEY`, JobParameters'ın **identifying** olanlarından hash üretilir. Bu sayede framework "aynı Job + aynı params = aynı JobInstance" eşliğini sağlar.

#### 3.2 BATCH_JOB_EXECUTION tablosu

```
JOB_EXECUTION_ID | JOB_INSTANCE_ID | CREATE_TIME | START_TIME | END_TIME | STATUS    | EXIT_CODE
-----------------+-----------------+-------------+------------+----------+-----------+-----------
1                | 1               | 23:30       | 23:30      | 23:42    | FAILED    | FAILED
2                | 1               | 23:45       | 23:45      | 23:55    | COMPLETED | COMPLETED
3                | 2               | 23:30       | 23:30      | 23:50    | COMPLETED | COMPLETED
```

Aynı `JOB_INSTANCE_ID` (1) için iki execution var → restart oldu.

#### 3.3 BATCH_STEP_EXECUTION tablosu

```
STEP_EXECUTION_ID | JOB_EXECUTION_ID | STEP_NAME      | STATUS    | READ_COUNT | WRITE_COUNT | COMMIT_COUNT | ROLLBACK_COUNT
------------------+------------------+----------------+-----------+------------+-------------+--------------+----------------
1                 | 1                | reconcileStep  | FAILED    | 45000      | 44000       | 44           | 1
2                 | 2                | reconcileStep  | COMPLETED | 100000     | 100000      | 100          | 0
```

Banking gözlem: `JobExecution 1`'de step 45K okudu, 44K yazdı (1K skip edildi muhtemelen), commit 44, rollback 1 → bir chunk rollback ettikten sonra patladı. `JobExecution 2`'de restart ile 100K tamamlandı.

#### 3.4 ExecutionContext (BATCH_JOB_EXECUTION_CONTEXT, BATCH_STEP_EXECUTION_CONTEXT)

**Key-value store** — restart için. İçinde reader'ın "kaçıncı satırdayım", writer'ın "kaçıncı kayda kadar yazdım" gibi state tutulur. **Topic 3'te ayrıntı.**

```
"reader.position": 45000
"reader.currentResource": "transactions_2025_05_10.csv"
```

Restart anında framework bu context'i okur, reader'a "45000'den devam" der.

---

### 4. JobParameters: kimlik ve identifying flag

`JobParameters` = Job'a runtime'da geçirilen key-value parametreler.

```java
JobParameters params = new JobParametersBuilder()
    .addString("businessDate", "2025-05-10")
    .addString("reportFormat", "CSV", false)   // false = non-identifying
    .toJobParameters();

jobLauncher.run(eodReconciliationJob, params);
```

**Identifying vs non-identifying:**

- `addString("businessDate", "2025-05-10")` → default identifying=TRUE
- `addString("reportFormat", "CSV", false)` → identifying=FALSE

JobInstance'ın `JOB_KEY` hash'ine **sadece identifying olanlar** girer.

Sonuç:

```
Run 1: businessDate=2025-05-10, reportFormat=CSV    → JobInstance 1
Run 2: businessDate=2025-05-10, reportFormat=XML    → JobInstance 1 (aynı!)
Run 3: businessDate=2025-05-11, reportFormat=CSV    → JobInstance 2
```

Identifying olan `businessDate` farklıysa farklı instance. `reportFormat` farklılığı yeni instance üretmez.

**Banking pratiği:** `businessDate` her zaman identifying. `traceId`, `triggeredBy` gibi audit alanları non-identifying. Bu sayede aynı gün için yanlışlıkla iki kez tetiklenmez.

**Tipler:** `addString`, `addLong`, `addDouble`, `addDate`, `addLocalDate` (Spring Batch 5+). Banking'de tarihler için her zaman `addLocalDate`.

**Spring Batch 5 değişikliği:** Spring Boot 3 / Spring Batch 5'te `JobParameters` artık `Object` map'i tutuyor (önceden sadece String/Long/Double/Date). Type-safe access için `JobParametersBuilder`'da convertable converter'lar var.

**Önemli not — restart davranışı:**

- Aynı Job + aynı params → eğer son execution COMPLETED ise → `JobInstanceAlreadyCompleteException`
- Aynı Job + aynı params → eğer son execution FAILED ise → **restart edilir** (kaldığı yerden)
- Aynı Job + farklı params → yeni JobInstance, yeni execution

Bu **deterministik** davranış banking'in güveni için kritik. "EOD job'unu iki kez çalıştırdım, balance iki kez güncellendi" senaryosu oluşamaz.

**Spring Batch 5 değişikliği ek:** `RunIdIncrementer` ve `JobParametersIncrementer` artık `incrementer` ile zincirleniyor. Cron job'larda gün bilgisi yoksa `RunIdIncrementer` her run için artan `run.id` parametresi ekleyerek "her tetikleme yeni instance" sağlıyor.

---

### 5. JobLauncher vs JobOperator

**JobLauncher:** Programatik tetikleme. `Job` + `JobParameters` alır, çalıştırır.

```java
@Service
class EodTrigger {
    private final JobLauncher jobLauncher;
    private final Job eodReconciliationJob;

    @Scheduled(cron = "0 55 23 * * *")
    public void runEod() throws Exception {
        var params = new JobParametersBuilder()
            .addLocalDate("businessDate", LocalDate.now())
            .toJobParameters();
        jobLauncher.run(eodReconciliationJob, params);
    }
}
```

**Synchronous vs Async:**

Default `SimpleJobLauncher` **synchronous** — `run()` çağrısı job bitene kadar bloklar. Banking'de problem:

- `@Scheduled` thread'ini saatlerce bloklar
- Yan job tetiklemesi gecikir

Çözüm: `TaskExecutor` set et → asenkron olur, job ayrı thread'de çalışır, `run()` hemen `JobExecution`'ı döner (henüz STARTED state'inde).

```java
@Bean
public JobLauncher asyncJobLauncher(JobRepository jobRepository) {
    var launcher = new TaskExecutorJobLauncher();
    launcher.setJobRepository(jobRepository);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor("batch-"));
    try {
        launcher.afterPropertiesSet();
    } catch (Exception e) {
        throw new IllegalStateException(e);
    }
    return launcher;
}
```

**Spring Batch 5 değişikliği:** `SimpleJobLauncher` deprecate oldu, `TaskExecutorJobLauncher` kullanılıyor.

**JobOperator:** Daha yüksek seviye, **operasyonel** API. Job'u name ile başlatma, restart, stop, abandone:

```java
@Service
class EodOperatorService {
    private final JobOperator jobOperator;

    public Long start(String jobName, String params) throws Exception {
        return jobOperator.start(jobName, params);  // "businessDate=2025-05-10"
    }

    public Long restart(Long executionId) throws Exception {
        return jobOperator.restart(executionId);
    }

    public boolean stop(Long executionId) throws Exception {
        return jobOperator.stop(executionId);
    }

    public Set<Long> findRunningExecutions(String jobName) {
        return jobOperator.getRunningExecutions(jobName);
    }
}
```

`JobOperator`, "operasyon ekibi job'ları yönetmek için" tasarlandı — REST endpoint'in arkasında kullanılır:

```java
@PostMapping("/admin/batch/jobs/{name}/restart/{execId}")
public ResponseEntity<Long> restart(@PathVariable String name, @PathVariable Long execId) {
    return ResponseEntity.ok(jobOperator.restart(execId));
}
```

Banking'de operatör paneli (Operations Center) bu API'leri kullanır. Faz 8'de security ile koruyacaksın.

---

### 6. Job tetiklemenin yolları

Bir Job nasıl başlar?

1. **CommandLineJobRunner** (deprecated Spring Batch 5'te, ama klasik): `java -jar app.jar --spring.batch.job.name=eodReconciliationJob businessDate=2025-05-10`. Cron + jar.
2. **@Scheduled** + `JobLauncher.run()`: aynı uygulama içinden cron-style. (Topic 6)
3. **REST endpoint** + `JobLauncher`/`JobOperator`: operatör tetikler. **Banking'de yaygın.**
4. **Message-driven** (Kafka consumer + JobLauncher): event'le tetikleme. **Topic 6 (messaging) ile birleşir.**
5. **Quartz scheduler**: enterprise scheduling. Cluster-aware. (Topic 6)
6. **Airflow / Kubernetes CronJob / external orchestrator**: en yaygın modern setup — Spring Batch sadece "executor", scheduling external.

**TR bankalarında en yaygın setup:**

- Control-M (BMC) veya Autosys veya AppWorx — enterprise scheduler
- O scheduler `curl` ile REST endpoint çağırır (`POST /admin/batch/eod/start?businessDate=...`)
- Spring Batch app'i job'u çalıştırır, status'ü DB'den ya da response'tan polling ile kontrol eder

Bu, scheduling ile execution'ı **decouple eder** — Spring Batch sadece job runner; orchestration external. Modern bulut yapılarında Kubernetes CronJob aynı rolü oynar.

---

### 7. Sequential, Conditional ve Parallel flow (giriş seviyesi)

`Job`'da step'ler **sırayla**, **conditional**, veya **paralel** akabilir. Detayı Topic 4'te, burada özet:

**Sequential:**

```java
new JobBuilder("eodJob", repo)
    .start(reconcileStep)
    .next(accrueInterestStep)
    .next(generateReportStep)
    .build();
```

**Conditional (decider/exit status):**

```java
new JobBuilder("eodJob", repo)
    .start(reconcileStep)
        .on("COMPLETED").to(accrueInterestStep)
        .from(reconcileStep).on("FAILED").to(alertStep)
    .end()
    .build();
```

**Parallel (split):**

```java
new JobBuilder("eodJob", repo)
    .start(prepareStep)
    .split(new SimpleAsyncTaskExecutor())
        .add(fraudReportFlow, masakReportFlow, bddkReportFlow)
    .next(finalizeStep)
    .build();
```

Bu konseptler Topic 4'ün ana konusu.

---

### 8. Banking örneği — eodReconciliationJob mimari taslağı

Senin `core-banking` projende kuracağın "EOD Reconciliation" job'u:

```
Job: eodReconciliationJob
JobParameters: businessDate (identifying), traceId (non-identifying), triggeredBy (non-identifying)

Step 1 (tasklet): "snapshotAccountBalancesStep"
  - Tüm hesapların bakiyelerini bir snapshot tablosuna yaz
  - Kısa, hızlı step
  - SELECT id, balance FROM accounts → INSERT INTO eod_snapshot

Step 2 (chunk): "recalculateBalancesFromLedgerStep"
  - journal_entries + journal_lines'tan her hesap için balance recompute et
  - 10M+ satır olabilir → chunk-oriented + paging reader
  - chunk size 1000
  - INSERT INTO recomputed_balances

Step 3 (chunk): "findMismatchesStep"
  - eod_snapshot vs recomputed_balances → SELECT JOIN WHERE diff != 0
  - mismatch bulunanları → INSERT INTO reconciliation_report

Step 4 (tasklet): "cleanupTemporaryTablesStep"
  - TRUNCATE eod_snapshot, recomputed_balances (sadece son JobExecution sonrası)
  - Yalnızca status=COMPLETED ise (decider ile conditional)

Listener:
  - JobExecutionListener: başladı/bitti audit log
  - SkipListener: skip edilen item'ları dlq_journal_lines tablosuna yaz
  - StepExecutionListener: her step'in metrics'ini Prometheus'a push et
```

Bu taslakta her topic'ten parça var. Mini-project'te bunu **gerçekten** yazacaksın.

---

### 9. JobRepository transaction izolasyonu (banking için kritik)

JobRepository, **batch state'i** ile **iş verisi**ni **aynı DataSource'ta** tutar. Bu önemli bir karar:

**Same DataSource (default):** Spring Batch state'i + business data aynı PostgreSQL. Avantaj: chunk commit, hem business data'yı hem state'i tek TX'te yazar — exactly-once semantik.

**Separate DataSource:** Banking'de bazen `BATCH_*` tablolarını ayrı bir "ops" DB'ye koyarlar. Sebep: prod DB'de schema cleanliness. Ama dezavantaj: chunk commit artık atomik değil (XA TX olmazsa).

**Banking pratiği:** Aynı DB. Schema separation (schema=batch) ile.

**Isolation level:** JobRepository default `SERIALIZABLE` çalışır — bu, "iki concurrent JobLauncher'ın aynı JobInstance'ı yaratmaya çalışmasını" engeller. PostgreSQL'de bu yüksek izolasyon, deadlock riski. `READ_COMMITTED`'a düşürmek mümkün ama race condition (duplicate JobInstance) riski doğar.

```java
@Bean
public JobRepository jobRepository(DataSource ds, PlatformTransactionManager tm) throws Exception {
    var factory = new JobRepositoryFactoryBean();
    factory.setDataSource(ds);
    factory.setTransactionManager(tm);
    factory.setIsolationLevelForCreate("ISOLATION_SERIALIZABLE");
    factory.setTablePrefix("BATCH_");
    factory.afterPropertiesSet();
    return factory.getObject();
}
```

Bu config'i Spring Boot otomatik yapar; sadece `@EnableBatchProcessing` annotation ile (Spring Batch 5'te genelde gereksiz — Boot otomatik enable eder).

---

### 10. Anti-pattern'ler — ne yapma

**Anti-pattern 1: Job'u her başlangıçta çalıştırmak**

```java
// ❌ Spring Boot startup'ta her başlangıçta otomatik çalıştırır
spring.batch.job.enabled=true
```

App restart'ında "günsonu mutabakat" 14:23'te çalışır. **Felaket.**

Doğru: `spring.batch.job.enabled=false` ve scheduling'i ayrıca ele al.

**Anti-pattern 2: JobParameters'a timestamp koyup identifying yapmak**

```java
// ❌ Her run yeni JobInstance üretir — restart kaybolur
new JobParametersBuilder()
    .addLong("timestamp", System.currentTimeMillis())
    .toJobParameters();
```

Sonuç: aynı businessDate için 5 farklı JobInstance, restart imkânsız. Doğrusu: `timestamp`'i **non-identifying** veya hiç koyma.

**Anti-pattern 3: Business logic'i Job orchestration'a karıştırmak**

```java
// ❌ Step içinde "if today is weekend, skip" logic'i
```

Doğrusu: business calendar check'i **scheduler / decider** seviyesinde. Job çalıştığında "iş yapacaktır" varsayımı.

**Anti-pattern 4: Non-restartable kod yazmak**

```java
// ❌ Processor'da random UUID üreterek output yazıyorsun
class FraudReportProcessor implements ItemProcessor<...> {
    public ... process(...) {
        // random uuid her execution'da farklı → idempotent değil
        report.setReportId(UUID.randomUUID());
    }
}
```

Restart'ta aynı report farklı ID ile iki kez yazılır. Doğrusu: deterministik ID (`businessDate + customerId` hash).

**Anti-pattern 5: JobRepository tablolarını manuel update etmek**

Bir job FAILED kaldı, "ben elle COMPLETED'e çevireyim" → ekosistem bozulur, restart davranışı kırılır. Doğrusu: `JobOperator.abandon()` veya `setStatus`/`setExitStatus` ile programatik.

---

### 11. Spring Batch'in Spring Boot 3 + Java 21 ile durumu

Sürüm tablosu:

| Spring Boot | Spring Batch | Notlar |
|-------------|--------------|--------|
| 2.x | 4.x | LTS değil, Java 8/11 |
| 3.0-3.2 | 5.0-5.1 | Java 17+, EE9 (jakarta.*) |
| 3.3+ | 5.1+ | Java 21 destekli, virtual threads opsiyon |

**Spring Batch 5 önemli değişiklikleri:**

- `jakarta.*` namespace (Spring 6)
- `@EnableBatchProcessing` artık opsiyonel (Boot otomatik enable eder)
- `JobBuilderFactory` / `StepBuilderFactory` deprecate → `JobBuilder` / `StepBuilder` direkt
- `JobRepository` artık `JobBuilder` constructor'ına geçirilir
- `JobParameters` artık any-type (önceden 4 tip)
- `SimpleJobLauncher` deprecate → `TaskExecutorJobLauncher`
- Virtual thread executor desteği (Java 21)

**Senin projende:** Spring Boot 3.3.x kullandığın için Spring Batch 5.1+'te olacaksın. Bu README'de gösterilen tüm API'ler Spring Batch 5 syntax'ı.

---

## Önemli olabilecek araştırma kaynakları (keyword)

- "Spring Batch reference documentation" (docs.spring.io/spring-batch/docs/current/reference/html)
- "Spring Batch 5 migration guide" (4 → 5 geçiş notları)
- "Spring Batch JobRepository schema postgresql" (DDL referansı)
- "JobInstance vs JobExecution" Stack Overflow tartışmaları
- "JobParameters identifying flag" — restart davranışı için
- "Spring Batch BatchStatus ExitStatus difference"
- Michael Minella "The Definitive Guide to Spring Batch" (kitap, 3rd edition)
- Spring IO YouTube — Mahmoud Ben Hassine konferans konuşmaları (Spring Batch lead)
- Baeldung "Spring Batch Tutorial" serisi (giriş için iyi)
- Spring Batch GitHub — `spring-projects/spring-batch` issues & releases

---

## Mini task'ler

Bu topic'te kod yazma az; çoğunluk kavramları sindirmek ve **JobRepository'i incelemek**.

### Task 5.1.1 — Spring Batch starter ve schema kurulumu (40 dk)

`core-banking/pom.xml`'e ekle:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

`src/main/resources/application.yml`'a ekle:

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never
    job:
      enabled: false
```

Flyway migration ekle: `src/main/resources/db/migration/V100__add_spring_batch_tables.sql`.

Spring Batch'in `schema-postgresql.sql` dosyasını jar'dan çıkar (IDE'de jar içinde `org/springframework/batch/core/schema-postgresql.sql` ara) → kendi migration dosyana kopyala. Yapıştır + commit et.

Banking discipline: **Production'da batch schema'sı senin kontrolündedir.** Spring Boot'a "kursana" demek profesyonel değil.

`mvn flyway:info` ile migration'ı doğrula. `mvn spring-boot:run` ile uygulamayı çalıştır, `psql` ile bağlan, `\dt batch_*` veya `\dt BATCH_*` ile tabloları gör. (PostgreSQL case-folding'i — küçük tabloya çevirir varsayılan.)

### Task 5.1.2 — İlk Job'unu yaz: "helloBatchJob" (45 dk)

`core-banking/banking/batch/config/HelloBatchJobConfig.java`:

```java
@Configuration
public class HelloBatchJobConfig {

    @Bean
    public Step helloStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
        return new StepBuilder("helloStep", jobRepository)
            .tasklet((contribution, chunkContext) -> {
                var businessDate = chunkContext.getStepContext()
                    .getJobParameters()
                    .get("businessDate");
                System.out.println("Hello batch, businessDate=" + businessDate);
                return RepeatStatus.FINISHED;
            }, txManager)
            .build();
    }

    @Bean
    public Job helloBatchJob(JobRepository jobRepository, Step helloStep) {
        return new JobBuilder("helloBatchJob", jobRepository)
            .start(helloStep)
            .build();
    }
}
```

Tetiklemek için bir `@RestController`:

```java
@RestController
@RequestMapping("/admin/batch")
class BatchAdminController {
    private final JobLauncher jobLauncher;
    private final Job helloBatchJob;

    @PostMapping("/hello")
    public String trigger(@RequestParam String date) throws Exception {
        var params = new JobParametersBuilder()
            .addString("businessDate", date)
            .toJobParameters();
        var execution = jobLauncher.run(helloBatchJob, params);
        return "JobExecutionId=" + execution.getId() + " status=" + execution.getStatus();
    }
}
```

`curl -X POST 'http://localhost:8080/admin/batch/hello?date=2025-05-10'`

Çıktı: console'da "Hello batch, businessDate=2025-05-10". Response: JobExecutionId ve status.

### Task 5.1.3 — JobRepository tablolarını manuel incele (30 dk)

`psql`'a bağlan:

```sql
SELECT * FROM batch_job_instance;
SELECT * FROM batch_job_execution;
SELECT * FROM batch_step_execution;
SELECT * FROM batch_job_execution_params;
```

Şunları gör:

- `batch_job_instance.JOB_KEY` hash'i
- `batch_job_execution.STATUS=COMPLETED`, `EXIT_CODE=COMPLETED`
- `batch_step_execution.READ_COUNT`, `WRITE_COUNT` (tasklet'te 0)
- `batch_job_execution_params`'ta businessDate parametresi

### Task 5.1.4 — JobInstanceAlreadyComplete'i tetikle (20 dk)

Aynı `curl`'ü ikinci kez çalıştır.

```
curl -X POST 'http://localhost:8080/admin/batch/hello?date=2025-05-10'
```

Hata bekleniyor: `JobInstanceAlreadyCompleteException`.

**Niye?** `businessDate=2025-05-10` identifying, son execution COMPLETED, tekrar çalıştırılamaz.

`date=2025-05-11` ile çalıştır → yeni instance üretilir, çalışır.

### Task 5.1.5 — Non-identifying parameter ekle (20 dk)

Controller'ı düzenle:

```java
var params = new JobParametersBuilder()
    .addString("businessDate", date)
    .addString("triggeredBy", "manual-curl", false)  // non-identifying
    .toJobParameters();
```

Aynı tarihte iki run dene: hâlâ ikinci run hata verir (`businessDate` identifying olduğu için).

`addString("triggeredBy", "manual-curl-1", false)` ve `addString("triggeredBy", "manual-curl-2", false)` ile iki run dene → ikisi de aynı `JobInstance`'a düşer; ikincisi yine hata (instance complete).

Sonra `RunIdIncrementer` ekle:

```java
@Bean
public Job helloBatchJob(JobRepository jobRepository, Step helloStep) {
    return new JobBuilder("helloBatchJob", jobRepository)
        .incrementer(new RunIdIncrementer())
        .start(helloStep)
        .build();
}
```

Şimdi `JobLauncher` tetiklenirken otomatik `run.id` artar — her tetikleme yeni instance.

**Sonuç:** "Aynı parametrelerle bir kez çalış" davranışını kontrolün altında tut.

### Task 5.1.6 — JobOperator ile job listeleme endpoint'i (30 dk)

```java
@GetMapping("/jobs/{name}/executions")
public List<String> listExecutions(@PathVariable String name) throws Exception {
    return jobOperator.getJobInstances(name, 0, 50).stream()
        .flatMap(instanceId -> {
            try {
                return jobOperator.getExecutions(instanceId).stream();
            } catch (NoSuchJobInstanceException e) {
                return Stream.empty();
            }
        })
        .map(execId -> "ExecutionId=" + execId)
        .toList();
}
```

`curl http://localhost:8080/admin/batch/jobs/helloBatchJob/executions` → tüm execution ID listesi.

Bu endpoint'in Faz 8'de **role-based** olarak korunması lazım (admin only). Şimdilik açık bırak.

### Task 5.1.7 — `@Scheduled` ile tetikleme (Topic 6 ön-tat) (20 dk)

```java
@Component
class EodScheduler {
    private final JobLauncher jobLauncher;
    private final Job helloBatchJob;

    @Scheduled(cron = "0 */1 * * * *")  // her dakika (test için)
    public void run() throws Exception {
        var params = new JobParametersBuilder()
            .addString("businessDate", LocalDate.now().toString())
            .addLong("run.id", System.currentTimeMillis())   // bypass JobInstance unique
            .toJobParameters();
        jobLauncher.run(helloBatchJob, params);
    }
}
```

`@EnableScheduling` ana class'a ekle. Uygulamayı çalıştır, console'da her dakika "Hello batch" gör.

**Önemli:** Bu sadece deneme. Topic 6'da bunu **ShedLock** ile sarmalayacaksın (multi-instance'ta tek tetikleme).

### Task 5.1.8 — ADR yaz: "Spring Batch seçimi" (15 dk)

`core-banking/docs/adr/0010-use-spring-batch-for-eod.md`:

```
# ADR-010: Spring Batch for EOD jobs

## Context
core-banking projesinde günsonu, ay sonu batch işleri lazım.
Adaylar: Spring Batch, custom Java code, AWS Step Functions, Airflow.

## Decision
Spring Batch 5 kullanılacak. JobRepository PostgreSQL'de aynı schema.
spring.batch.job.enabled=false; tetikleme @Scheduled/REST endpoint.

## Consequences
+ Restart semantiği framework verdiği için
+ Audit (BATCH_* tabloları) bedava
+ Skip/Retry mekanizmaları hazır
- Cron scheduler ayrı bir bileşen olacak (Quartz / external)
- Schema migration Flyway'de manuel yönetilir
```

---

## Test yazma rehberi

Bu topic'te asıl test = **smoke test**. Job'un çalıştığını, JobRepository'nin state yazdığını gör. Üst düzey unit test'ler Topic 2'de gelecek.

### Test 5.1.1 — Job çalıştırma integration test (45 dk)

`src/test/java/.../HelloBatchJobIntegrationTest.java`:

```java
@SpringBootTest
@SpringBatchTest
@Testcontainers
class HelloBatchJobIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;

    @Test
    void helloBatchJob_shouldCompleteSuccessfully() throws Exception {
        var params = new JobParametersBuilder()
            .addString("businessDate", "2025-05-10")
            .addLong("run.id", System.currentTimeMillis())
            .toJobParameters();

        var execution = jobLauncherTestUtils.launchJob(params);

        assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        assertThat(execution.getExitStatus().getExitCode()).isEqualTo("COMPLETED");
    }

    @Test
    void helloBatchJob_shouldFailOnDuplicateInstance() throws Exception {
        var params = new JobParametersBuilder()
            .addString("businessDate", "2025-05-11")
            .toJobParameters();

        jobLauncherTestUtils.launchJob(params);  // ilk run

        assertThatThrownBy(() -> jobLauncherTestUtils.launchJob(params))
            .isInstanceOf(JobInstanceAlreadyCompleteException.class);
    }
}
```

`@SpringBatchTest` annotation `JobLauncherTestUtils` ve `JobRepositoryTestUtils` bean'lerini sağlar.

### Test 5.1.2 — JobRepository state assert (30 dk)

```java
@Autowired
private JobRepository jobRepository;

@Test
void afterRun_jobRepositoryShouldHaveExecutionRecord() throws Exception {
    var params = new JobParametersBuilder()
        .addString("businessDate", "2025-05-12")
        .toJobParameters();

    var execution = jobLauncherTestUtils.launchJob(params);

    var found = jobRepository.getLastJobExecution("helloBatchJob", params);
    assertThat(found).isNotNull();
    assertThat(found.getId()).isEqualTo(execution.getId());
    assertThat(found.getStatus()).isEqualTo(BatchStatus.COMPLETED);
    assertThat(found.getStepExecutions()).hasSize(1);
}
```

### Test 5.1.3 — Parameters identifying flag davranışı (30 dk)

```java
@Test
void nonIdentifyingParam_shouldNotCreateNewInstance() throws Exception {
    var params1 = new JobParametersBuilder()
        .addString("businessDate", "2025-05-13")
        .addString("traceId", "trace-a", false)
        .toJobParameters();
    var params2 = new JobParametersBuilder()
        .addString("businessDate", "2025-05-13")
        .addString("traceId", "trace-b", false)
        .toJobParameters();

    jobLauncherTestUtils.launchJob(params1);

    assertThatThrownBy(() -> jobLauncherTestUtils.launchJob(params2))
        .isInstanceOf(JobInstanceAlreadyCompleteException.class);
}
```

### Test 5.1.4 — JobOperator restart davranışı (Topic 3 ön-tat, 30 dk)

İlerideki topic'lerde tam restart test ettireceksin. Şimdilik:

```java
@Autowired
private JobOperator jobOperator;

@Test
void jobOperator_canQueryExecutions() throws Exception {
    var params = new JobParametersBuilder()
        .addString("businessDate", "2025-05-14")
        .toJobParameters();
    jobLauncherTestUtils.launchJob(params);

    var executions = jobOperator.getExecutions(
        jobOperator.getJobInstances("helloBatchJob", 0, 10).get(0)
    );
    assertThat(executions).hasSize(1);
}
```

---

## Claude-verify prompt

```
Aşağıdaki Spring Batch konfigürasyonum ve task'lerimi Spring Batch 5 + Spring Boot 3
+ banking EOD job context'inde DEĞERLENDİR ve EKSİKLERİ söyle, KOD YAZMA:

1. pom.xml'de spring-boot-starter-batch var mı?

2. application.yml:
   - spring.batch.jdbc.initialize-schema: never mı?
   - spring.batch.job.enabled: false mı?
   Yoksa banking discipline kaybı.

3. Spring Batch schema:
   - Flyway migration olarak mı eklendi (V100__add_spring_batch_tables.sql)?
   - Yoksa spring boot'a otomatik kurdurma fonksiyonu mu açık?

4. HelloBatchJobConfig:
   - JobBuilder (Spring Batch 5) mu, JobBuilderFactory (Spring Batch 4 deprecated) mı?
   - StepBuilder ve tasklet düzgün yazılmış mı?
   - PlatformTransactionManager tasklet'e geçirilmiş mi?

5. JobParameters kullanımı:
   - businessDate identifying mi?
   - traceId / triggeredBy non-identifying mi (3. arg false)?
   - RunIdIncrementer test edildi mi?

6. JobLauncher:
   - SimpleJobLauncher mı (deprecated) yoksa TaskExecutorJobLauncher mı?
   - Synchronous mu (default) — banking için bunu bilinçli mi seçtin?

7. Test'ler:
   - @SpringBatchTest annotation kullanıldı mı?
   - JobLauncherTestUtils ile mi launch ediliyor?
   - JobInstanceAlreadyComplete test edildi mi?
   - Testcontainers ile gerçek PostgreSQL mü, H2 mi (banking'de H2 kaçınılması gereken)?

8. ADR:
   - 0010-use-spring-batch-for-eod.md yazıldı mı?
   - Context, Decision, Consequences var mı?

9. Banking discipline:
   - Job'u app startup'ta otomatik tetikleyen kod var mı (olmamalı)?
   - JobOperator endpoint'i güvenliksiz açık mı (Faz 8'e not düştün mü)?

Her madde için PASS / FAIL / EKSİK olarak işaretle. KOD YAZMA, sadece neyin eksik
olduğunu söyle.
```

---

## Tamamlama kriterleri

- [ ] `spring-boot-starter-batch` dependency eklendi
- [ ] `spring.batch.job.enabled=false`, `spring.batch.jdbc.initialize-schema=never` ayarlı
- [ ] Spring Batch schema Flyway migration olarak eklendi
- [ ] `helloBatchJob` çalıştı, console'da log gördüm
- [ ] `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` tablolarını psql ile inceledim
- [ ] `JobInstanceAlreadyCompleteException`'ı kasten tetikledim
- [ ] Identifying vs non-identifying parameter farkını test ile gözledim
- [ ] `RunIdIncrementer` ile her run yeni instance olabildiğini gördüm
- [ ] `@SpringBatchTest` ile integration test yazdım (en az 3 test)
- [ ] ADR-010 yazıldı

Hepsi yeşil ise → Topic 2'ye geç → [02-chunk-oriented/](../02-chunk-oriented/README.md)

---

## Defter notları (kendi cümlelerinle yaz)

1. "JobInstance ile JobExecution arasındaki fark: ____. Banking'de bunun önemi ____."
2. "JobRepository tabloları PostgreSQL'de neyi tutar? ____. Restart için en kritik olan tablo ____."
3. "Identifying vs non-identifying JobParameter farkı ____. Banking'de businessDate her zaman ____ çünkü ____."
4. "JobLauncher synchronous vs async farkı, banking @Scheduled context'inde önemi ____."
5. "Spring Batch'i ne zaman seçmem? ____."
6. "Spring Batch 5 + Spring Boot 3'te major API değişiklikleri: ____."
