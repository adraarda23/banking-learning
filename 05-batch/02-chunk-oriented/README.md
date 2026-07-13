# Topic 5.2 — Chunk-Oriented Processing

## Hedef

Spring Batch'in en yaygın step tipi — **chunk-oriented step**'i derinlemesine öğrenmek. `ItemReader`, `ItemProcessor`, `ItemWriter` üçlüsünün lifecycle'ını, chunk size'ın memory/performance trade-off'unu, transaction boundary'sinin chunk düzeyinde olmasını ve banking örnekleriyle pratik uygulamayı kavramak. JpaPagingItemReader, JdbcPagingItemReader, FlatFileItemReader gibi reader tiplerini banking job'larında kullanmak.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 45 dk • Toplam: ~5 saat

## Önbilgi

- Topic 5.1 (Batch Architecture) bitti
- `Job`, `Step`, `JobRepository`, `JobLauncher` kavramları aşinasın
- Banking domain (account, transaction, journal) Phase 1'den biliniyor

---

## Kavramlar

### 1. Chunk-oriented vs Tasklet

Spring Batch'te iki step tipi var:

**Chunk-oriented step:** Reader → Processor → Writer üçlüsü, chunk-by-chunk işleme.

```
1. Reader: 1000 satırlık chunk oku
2. Processor: her satırı transform et / filtre et
3. Writer: 1000 satırı toplu yaz
4. Commit transaction
5. Sonraki chunk'a geç
```

**Tasklet:** Tek bir method execute eder. Reader/Processor/Writer yok. Basit işler için (örn. file delete, DB cleanup).

```java
@Bean
public Step cleanupStep() {
    return new StepBuilder("cleanupStep", jobRepository)
        .tasklet((contribution, chunkContext) -> {
            jdbcTemplate.update("DELETE FROM temp_data WHERE created_at < ?", 
                Instant.now().minus(30, ChronoUnit.DAYS));
            return RepeatStatus.FINISHED;
        }, transactionManager)
        .build();
}
```

**Banking pratiği:** Çoğu EOD job chunk-oriented (büyük data set işleme). Setup/cleanup için tasklet.

### 2. Chunk-oriented step yapısı

```java
@Bean
public Step interestAccrualStep() {
    return new StepBuilder("interestAccrualStep", jobRepository)
        .<Account, InterestPosting>chunk(1000, transactionManager)
        .reader(activeAccountReader())
        .processor(interestProcessor())
        .writer(interestPostingWriter())
        .faultTolerant()
            .skip(InvalidAccountException.class)
            .skipLimit(100)
            .retry(DeadlockLoserDataAccessException.class)
            .retryLimit(3)
        .build();
}
```

**`<Account, InterestPosting>`:** Reader Account üretir, Processor InterestPosting üretir, Writer InterestPosting yazar.

**`chunk(1000, transactionManager)`:** 1000 item bir chunk'tır. Her chunk **kendi transaction'ında**.

### 3. ItemReader — veri okuma

#### `ItemReader` interface

```java
public interface ItemReader<T> {
    T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;
    // null dönerse "veri kalmadı" anlamında
}
```

Her chunk için reader N kez `read()` çağrılır (N = chunk size). `null` dönerse step biter.

#### `JpaPagingItemReader` — JPA ile sayfa sayfa okuma

```java
@Bean
public JpaPagingItemReader<Account> activeAccountReader() {
    return new JpaPagingItemReaderBuilder<Account>()
        .name("activeAccountReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT a FROM Account a WHERE a.status = 'ACTIVE' ORDER BY a.id")
        .pageSize(1000)
        .build();
}
```

**`pageSize`:** Her DB roundtrip'inde 1000 satır. **Chunk size ile aynı yapılır** genelde.

**`ORDER BY`:** **Mutlaka deterministic sıra**. Yoksa pagination bozulur, satır kaçırır/tekrarlar.

#### `JdbcPagingItemReader` — daha hızlı, JPA yok

```java
@Bean
public JdbcPagingItemReader<Account> jdbcAccountReader() {
    return new JdbcPagingItemReaderBuilder<Account>()
        .name("jdbcAccountReader")
        .dataSource(dataSource)
        .selectClause("SELECT id, owner_id, balance_amount, currency")
        .fromClause("FROM accounts")
        .whereClause("WHERE status = 'ACTIVE'")
        .sortKeys(Map.of("id", Order.ASCENDING))
        .pageSize(1000)
        .rowMapper((rs, n) -> new Account(
            UUID.fromString(rs.getString("id")),
            UUID.fromString(rs.getString("owner_id")),
            rs.getBigDecimal("balance_amount"),
            rs.getString("currency")
        ))
        .build();
}
```

**JPA overhead'i yok** — direct JDBC. 1M+ satır işlerken **2-3x hızlı**.

#### `FlatFileItemReader` — CSV dosya okuma

```java
@Bean
public FlatFileItemReader<TransferRequest> csvReader() {
    return new FlatFileItemReaderBuilder<TransferRequest>()
        .name("csvReader")
        .resource(new FileSystemResource("/data/transfers.csv"))
        .delimited()
        .delimiter(",")
        .names("fromAccountId", "toAccountId", "amount", "currency")
        .targetType(TransferRequest.class)
        .linesToSkip(1)   // header
        .build();
}
```

Banking örneği: Toplu havale dosyası (BANKA-ARASI-TOPLU-HAVALE-FORMAT.csv).

#### `MultiResourceItemReader` — birden fazla dosya

```java
@Bean
public MultiResourceItemReader<TransferRequest> multiCsvReader() {
    return new MultiResourceItemReaderBuilder<TransferRequest>()
        .name("multiCsvReader")
        .resources(new PathMatchingResourcePatternResolver().getResources("file:/data/transfers-*.csv"))
        .delegate(csvReader())
        .build();
}
```

`/data/transfers-001.csv`, `/data/transfers-002.csv` ... hepsini birleştir.

#### `StaxEventItemReader` — XML

ISO 20022 mesajları XML formatında. Spring Batch'in StAX-based reader'ı:

```java
@Bean
public StaxEventItemReader<Payment> iso20022Reader() {
    return new StaxEventItemReaderBuilder<Payment>()
        .name("iso20022Reader")
        .resource(new FileSystemResource("/data/pacs.008.xml"))
        .addFragmentRootElements("CdtTrfTxInf")
        .unmarshaller(paymentUnmarshaller())
        .build();
}
```

### 4. ItemProcessor — transform + filter

```java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}
```

Reader → Processor → Writer akışında **transform** veya **filter**.

```java
@Bean
public ItemProcessor<Account, InterestPosting> interestProcessor() {
    return account -> {
        if (account.getBalance().compareTo(BigDecimal.ZERO) <= 0) {
            return null;   // filter — bu item Writer'a gitmez
        }
        
        BigDecimal interest = account.getBalance()
            .multiply(new BigDecimal("0.085"))
            .divide(new BigDecimal("365"), 4, RoundingMode.HALF_EVEN);
        
        return new InterestPosting(account.getId(), interest, LocalDate.now());
    };
}
```

**`null` dönerse:** Item Writer'a gönderilmez (filter). Tüm chunk yok demek değil — sadece bu satır.

#### `CompositeItemProcessor` — pipeline

Birden fazla processor zinciri:

```java
@Bean
public CompositeItemProcessor<Account, InterestPosting> composite() {
    return new CompositeItemProcessorBuilder<Account, InterestPosting>()
        .delegates(validationProcessor(), interestCalculator(), auditTagger())
        .build();
}
```

Önce validate, sonra hesapla, sonra audit tag ekle.

### 5. ItemWriter — yazma

```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> chunk) throws Exception;
}
```

**`Chunk<T>`** — bir liste (chunk size kadar item). Writer **toplu** yazar (batch insert).

#### `JdbcBatchItemWriter` — JDBC batch insert

```java
@Bean
public JdbcBatchItemWriter<InterestPosting> interestWriter() {
    return new JdbcBatchItemWriterBuilder<InterestPosting>()
        .dataSource(dataSource)
        .sql("INSERT INTO interest_postings(account_id, amount, business_date) VALUES (:accountId, :amount, :businessDate)")
        .beanMapped()
        .build();
}
```

JDBC `addBatch` + `executeBatch` ile **tek roundtrip**, çok satır INSERT. **1000-10x hızlı.**

#### `JpaItemWriter` — JPA entity yazma

```java
@Bean
public JpaItemWriter<InterestPostingEntity> jpaWriter() {
    return new JpaItemWriterBuilder<InterestPostingEntity>()
        .entityManagerFactory(entityManagerFactory)
        .build();
}
```

**Tuzak:** JPA batching için `hibernate.jdbc.batch_size` config gerekli.

#### `FlatFileItemWriter` — CSV yaz

```java
@Bean
public FlatFileItemWriter<ReconciliationReport> reportWriter() {
    return new FlatFileItemWriterBuilder<ReconciliationReport>()
        .name("reportWriter")
        .resource(new FileSystemResource("/data/reconciliation-report.csv"))
        .delimited()
        .delimiter(",")
        .names("accountId", "expectedBalance", "actualBalance", "difference")
        .build();
}
```

#### `CompositeItemWriter` — birden fazla writer

Aynı item'ı hem DB'ye yaz hem CSV'ye:

```java
@Bean
public CompositeItemWriter<InterestPosting> composite() {
    return new CompositeItemWriterBuilder<InterestPosting>()
        .delegates(jdbcWriter(), csvWriter())
        .build();
}
```

### 6. Chunk size — performance trade-off

**Küçük chunk (örn. 10):**
- Memory az kullanır
- Sık transaction commit → DB stres
- Restart granular (kayıp az)
- Yavaş

**Büyük chunk (örn. 10000):**
- Memory çok kullanır (potential OOM)
- Az transaction → throughput yüksek
- Restart koarse (chunk içinde kayıp daha fazla)
- Hızlı

**Banking sweet spot:** 100-1000 genelde optimal. Mass insert için 1000-5000. Memory-bound iş için 50-200.

```java
.<Account, InterestPosting>chunk(1000, transactionManager)
```

### 7. Transaction boundary — chunk düzeyinde

```
Read chunk (1000) → Process all → Write all → COMMIT
```

Bir chunk'ın **read + process + write hepsi tek transaction**. İçinden bir hata atarsa **tüm chunk rollback**.

**Banking impact:**
- 999 başarılı + 1 fail → tüm 1000 satır rollback
- Skip policy ile bu davranışı değiştirebilirsin (sonraki topic 5.3)

### 8. Reader/Processor/Writer lifecycle

```
Step start
├── Reader.open() (varsa ItemStream)
├── Loop:
│   ├── For 1..chunkSize: Reader.read()
│   ├── For each item: Processor.process()
│   ├── Writer.write(chunk)
│   └── COMMIT
├── (No more items)
└── Reader.close()
```

**`ItemStream` interface** (opsiyonel) — open/update/close lifecycle. Reader'lar restart için durumlarını `ExecutionContext`'e yazar.

### 9. `ExecutionContext` ile restartability

```java
public class CustomReader implements ItemReader<Account>, ItemStream {
    
    private int currentIndex;
    
    @Override
    public void open(ExecutionContext executionContext) {
        currentIndex = executionContext.getInt("currentIndex", 0);
    }
    
    @Override
    public Account read() {
        // process and increment currentIndex
        currentIndex++;
        return next;
    }
    
    @Override
    public void update(ExecutionContext executionContext) {
        executionContext.putInt("currentIndex", currentIndex);
    }
    
    @Override
    public void close() { }
}
```

Job crash sonrası restart → `currentIndex` ExecutionContext'ten yüklenir → kaldığı yerden devam.

Default reader'lar (JpaPaging, JdbcPaging, FlatFile) bunu **otomatik** yapar.

### 10. Banking job örneği — Interest Accrual

```java
@Configuration
public class InterestAccrualJobConfig {
    
    @Bean
    public Job interestAccrualJob(JobRepository jobRepository, Step interestStep) {
        return new JobBuilder("interestAccrualJob", jobRepository)
            .start(interestStep)
            .incrementer(new RunIdIncrementer())
            .build();
    }
    
    @Bean
    public Step interestStep(JobRepository jobRepository, 
                             PlatformTransactionManager transactionManager,
                             ItemReader<Account> reader,
                             ItemProcessor<Account, InterestPosting> processor,
                             ItemWriter<InterestPosting> writer) {
        return new StepBuilder("interestStep", jobRepository)
            .<Account, InterestPosting>chunk(1000, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
    
    @Bean
    @StepScope
    public JpaPagingItemReader<Account> reader(
            @Value("#{jobParameters['businessDate']}") LocalDate businessDate,
            EntityManagerFactory emf) {
        return new JpaPagingItemReaderBuilder<Account>()
            .name("activeAccountReader")
            .entityManagerFactory(emf)
            .queryString("SELECT a FROM Account a WHERE a.status = 'ACTIVE' AND a.openedAt <= :businessDate")
            .parameterValues(Map.of("businessDate", businessDate))
            .pageSize(1000)
            .build();
    }
    
    @Bean
    public ItemProcessor<Account, InterestPosting> processor() {
        return account -> {
            BigDecimal interest = compoundInterest(account.getBalance(), 8.5, 1);
            return new InterestPosting(account.getId(), interest, LocalDate.now());
        };
    }
    
    @Bean
    public JdbcBatchItemWriter<InterestPosting> writer(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<InterestPosting>()
            .dataSource(dataSource)
            .sql("INSERT INTO interest_postings(account_id, amount, business_date, created_at) " +
                 "VALUES (:accountId, :amount, :businessDate, NOW())")
            .beanMapped()
            .build();
    }
}
```

### 11. `@StepScope` — runtime parameter binding

```java
@Bean
@StepScope
public JpaPagingItemReader<Account> reader(
        @Value("#{jobParameters['businessDate']}") LocalDate businessDate, ...) {
```

**`@StepScope`** lazy initialization — step çalışırken `jobParameters` bind edilir. Yoksa Spring bean container startup'ta yaratmaya çalışır, parameter olmadan fail.

**`@JobScope`** benzer ama job düzeyinde. Step'ler arası paylaşılan parameter için.

### 12. Banking pattern — EOD Reconciliation

```java
@Bean
public Step reconciliationStep(JobRepository jobRepository, ...) {
    return new StepBuilder("reconciliationStep", jobRepository)
        .<Account, ReconciliationMismatch>chunk(500, txManager)
        .reader(allActiveAccountsReader())
        .processor(reconciliationProcessor())   // null return → mismatch yok, skip
        .writer(mismatchWriter())
        .build();
}

@Bean
public ItemProcessor<Account, ReconciliationMismatch> reconciliationProcessor() {
    return account -> {
        BigDecimal calculatedBalance = journalLineRepo.sumBalanceByAccount(account.getId());
        if (account.getBalance().compareTo(calculatedBalance) == 0) {
            return null;   // mismatch yok, sonraki satır
        }
        return new ReconciliationMismatch(
            account.getId(),
            account.getBalance(),
            calculatedBalance,
            account.getBalance().subtract(calculatedBalance)
        );
    };
}
```

Sadece mismatch'leri Writer'a yollar. Eşleşenler skip.

### 13. Banking anti-pattern'leri

**Anti-pattern 1: Reader içinde DB call per-item**

```java
return account -> {
    Owner owner = ownerRepo.findById(account.getOwnerId()).orElseThrow();   // ❌ N+1
    // ...
};
```

100k account × 1 DB call = 100k extra query. **Çözüm:** Reader query'sinde JOIN ile owner'ı çek (`JOIN FETCH`).

**Anti-pattern 2: Chunk size sabit, memory'e bakmadan**

Chunk size 10000, item büyük → OOM. Memory profile'a göre seç.

**Anti-pattern 3: `ORDER BY` olmayan reader**

```java
.queryString("SELECT a FROM Account a WHERE status = 'ACTIVE'")   // ❌ ORDER BY yok
```

PagingItemReader sayfa kaçırır veya tekrar okur. **Mutlaka deterministic ORDER BY.**

**Anti-pattern 4: Writer'da JPA save loop**

```java
public void write(Chunk<? extends Account> chunk) {
    for (Account a : chunk) {
        repo.save(a);   // ❌ batch yok
    }
}
```

`JdbcBatchItemWriter` veya `JpaItemWriter` ile batch et.

**Anti-pattern 5: Tasklet'te uzun iş**

Tasklet'in tek transaction'ı var. Uzun iş = uzun transaction = lock contention. **Chunk-oriented kullan**.

---

## Önemli olabilecek araştırma kaynakları

- Spring Batch reference documentation (chunk-oriented section)
- "Spring Batch in Action" (Arnaud Cogoluègnes)
- Spring Batch samples GitHub
- Vlad Mihalcea — JPA batching with Spring Batch
- Banking ETL pattern blog posts

---

## Mini task'ler

### Task 5.2.1 — JpaPagingItemReader basic (30 dk)

`InterestAccrualJob` yarat — ACTIVE account'ları oku, log'a yaz (dummy writer).

```java
@Bean
public ItemWriter<Account> dummyWriter() {
    return chunk -> chunk.forEach(a -> log.info("Processing: {}", a.getId()));
}
```

10000 account ile test et. 1000'er sayfa.

### Task 5.2.2 — Interest calculation processor (30 dk)

`Account → InterestPosting` transformation. Balance × 8.5% / 365 → günlük faiz.

### Task 5.2.3 — JdbcBatchItemWriter (30 dk)

`interest_postings` tablosuna batch insert. JPA Writer ile karşılaştır — SQL log'da kaç INSERT?

### Task 5.2.4 — CSV reader → DB writer (45 dk)

`/data/transfers.csv` (1000 satır mock data). `TransferRequest` deserialize → DB'ye yaz.

### Task 5.2.5 — Composite processor (30 dk)

ValidationProcessor + InterestProcessor + AuditTagProcessor zinciri. Bir invalid account skip ediliyor mu?

### Task 5.2.6 — Chunk size benchmark (30 dk)

Aynı job'u chunk size = 10, 100, 1000, 10000 ile çalıştır. Sürede ölç, memory ölç. **Defterine** tablo.

### Task 5.2.7 — Reconciliation mismatch processor (45 dk)

EOD reconciliation step. `null` return ile mismatch'siz hesapları skip. Mismatch'leri writer'a göndər.

---

## Test yazma rehberi

### Test 5.2.1 — Job parameters ile integration test

```java
@SpringBatchTest
@SpringBootTest
@Testcontainers
class InterestAccrualJobTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @Autowired JobLauncherTestUtils jobLauncher;
    @Autowired AccountRepository accountRepo;
    
    @Test
    void shouldAccrueInterestForActiveAccounts() throws Exception {
        // Setup
        createAccount("1000", "ACTIVE");
        createAccount("2000", "ACTIVE");
        createAccount("3000", "CLOSED");   // skip
        
        JobParameters params = new JobParametersBuilder()
            .addLocalDate("businessDate", LocalDate.now())
            .toJobParameters();
        
        JobExecution exec = jobLauncher.launchJob(params);
        
        assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        assertThat(exec.getStepExecutions().iterator().next().getReadCount()).isEqualTo(2);   // ACTIVE'ler
        assertThat(exec.getStepExecutions().iterator().next().getWriteCount()).isEqualTo(2);
    }
}
```

### Test 5.2.2 — Processor unit test

```java
@Test
void shouldCalculateDailyInterest() {
    InterestProcessor processor = new InterestProcessor(new BigDecimal("8.5"));
    
    Account account = new Account(UUID.randomUUID(), UUID.randomUUID(), 
        new BigDecimal("10000.00"), "TRY");
    
    InterestPosting result = processor.process(account);
    
    // 10000 * 0.085 / 365 = 2.3287... ≈ 2.3288 (HALF_EVEN)
    assertThat(result.getAmount()).isEqualByComparingTo("2.3288");
}

@Test
void shouldFilterZeroBalanceAccounts() {
    InterestProcessor processor = new InterestProcessor(new BigDecimal("8.5"));
    
    Account zeroBalance = new Account(..., BigDecimal.ZERO, "TRY");
    
    assertThat(processor.process(zeroBalance)).isNull();
}
```

### Test 5.2.3 — Reader pagination

```java
@Test
void shouldReadAllActiveAccountsInPages() throws Exception {
    createAccounts(5000, "ACTIVE");
    createAccounts(1000, "CLOSED");
    
    JpaPagingItemReader<Account> reader = ... ; // builder
    reader.open(new ExecutionContext());
    
    int count = 0;
    Account a;
    while ((a = reader.read()) != null) {
        count++;
    }
    reader.close();
    
    assertThat(count).isEqualTo(5000);
}
```

---

## Claude-verify prompt

```
Spring Batch chunk-oriented kodumu banking-grade kriterlere göre değerlendir:

1. Reader:
   - JpaPagingItemReader veya JdbcPagingItemReader kullanılmış mı (cursor değil)?
   - pageSize chunk size ile uyumlu mu?
   - ORDER BY deterministic mi (UUID veya unique ID)?
   - @StepScope ile job parameter binding doğru mu?

2. Processor:
   - Reader'dan ayrı, sadece transform/filter yapıyor mu?
   - null return ile filter pattern'i uygulanmış mı?
   - DB call yapıyor mu? (Olmamalı — pre-load et)

3. Writer:
   - JdbcBatchItemWriter veya JpaItemWriter ile batch insert mi?
   - Tek tek save() loop'u var mı? (Olmamalı)
   - CompositeWriter ile multi-destination var mı (gerekirse)?

4. Chunk size:
   - Memory ile uyumlu seçilmiş mi (1000 banking sweet spot)?
   - Chunk transaction boundary'i bilinçli mi?

5. Banking specific:
   - Interest accrual job mantıksal olarak doğru mu (compound interest)?
   - EOD reconciliation null return ile mismatch'siz skip mu?
   - Bulk insert için JdbcBatchItemWriter mı?

6. Anti-pattern:
   - Reader içinde per-item DB call var mı? (N+1)
   - ORDER BY olmayan pagination var mı?
   - Tasklet uzun iş yapıyor mu?

7. ExecutionContext:
   - ItemStream interface'i custom reader'da uygulanmış mı?
   - Default reader'ların restart desteği biliniyor mu?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] JpaPagingItemReader ve JdbcPagingItemReader farkını biliyorum
- [ ] ItemProcessor null return ile filter pattern'i uyguladım
- [ ] JdbcBatchItemWriter ile batch insert
- [ ] Composite processor + writer pattern'i denedim
- [ ] Chunk size farklı değerlerle benchmark yaptım
- [ ] @StepScope ile job parameter binding
- [ ] Banking interest accrual + reconciliation job çalışıyor
- [ ] Reader'da DB call (N+1) yapmıyorum
- [ ] ORDER BY deterministic pagination

---

## Defter notları

1. "Chunk-oriented vs tasklet ne zaman hangisi: ____."
2. "ItemReader, ItemProcessor, ItemWriter sorumlulukları: ____."
3. "Chunk transaction boundary'sinin restart için anlamı: ____."
4. "JpaPagingItemReader vs JdbcPagingItemReader performans farkı: ____."
5. "Chunk size küçük/büyük trade-off (memory vs throughput): ____."
6. "ItemProcessor null return'ün anlamı: ____."
7. "JdbcBatchItemWriter ile JpaItemWriter batching farkı: ____."
8. "@StepScope ve @JobScope farkı: ____."
9. "Reader'da N+1 problem ve çözümü: ____."
10. "ExecutionContext ile restartability: ____."
