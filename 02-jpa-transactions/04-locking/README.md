# Topic 2.4 — Locking: Optimistic & Pessimistic

## Hedef

Concurrency'nin banking domain'inde yarattığı problemleri (lost update, double withdraw, deadlock) **gerçek reprodüksiyon kodlarıyla** görmek. Optimistic locking (`@Version`) ve pessimistic locking (`LockModeType.PESSIMISTIC_WRITE`) arasında karar verebilmek. Deadlock'u Java kodunda **kasten üretip** `jstack` ile gözlemleyip, **lock ordering** ile çözmek. PostgreSQL `SERIALIZABLE` serialization failure pattern'ini ve retry strategy'lerini (exponential backoff, `@Retryable`) banking-grade yazabilmek. `SELECT FOR UPDATE NOWAIT`, `SKIP LOCKED` gibi SQL primitiflerini bilinçli kullanmak.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1.5 saat • Toplam: ~6.5 saat

## Önbilgi

- Topic 2.3 (Transactions) bitti — propagation, isolation level, `@Transactional` mekaniği biliyorsun
- ACID'in "I"sının (isolation) neden tam çözüm olmadığını biliyorsun (lost update phenomenon REPEATABLE_READ'te bile mümkün, isolation level'a göre)
- `OptimisticLockException`, `PessimisticLockException` isimlerini duydun ama ne zaman fırlatıldığını net görmedin

---

## Kavramlar

### 1. Concurrency'nin banking'deki yüzü — lost update senaryosu

Bir hesap, bakiyesi 1000 TL. İki user (veya iki paralel API call) aynı anda gelir:

- **T1:** "Hesaptan 800 çek" — SELECT balance (1000) → balance - 800 = 200 → UPDATE
- **T2:** "Hesaptan 500 çek" — SELECT balance (1000) → balance - 500 = 500 → UPDATE

Eğer iki transaction da READ_COMMITTED isolation'da çalışıyorsa ve her biri `SELECT` → uygulama katmanında matematik → `UPDATE` şeklinde işliyorsa, ikisi de **kendi snapshot'ında 1000** görür. Son commit hangisi olursa, onun yazdığı değer (500 veya 200) DB'de kalır. **Ötekinin update'i kaybolur** — bu "lost update" problemi.

Sonuç: hesaptan toplam 1300 TL çekildi (kaynak yetersiz olduğu halde her iki işlem de "başarılı" dedi), bakiye 500 veya 200 kaldı. **Banka 800 TL veya 500 TL açık.**

**Bu lost update isolation level ile çözülmüyor mu?**

- READ_COMMITTED: Hayır (iki SELECT de 1000 görür, lock yok).
- REPEATABLE_READ (MySQL InnoDB): Hayır — MySQL `SELECT` salt okuyucu, lock almaz.
- REPEATABLE_READ (PostgreSQL): "Could not serialize access" hatası bazı durumlarda, ama UPDATE-only pattern'de garantili değil.
- SERIALIZABLE: Evet, ama performans cezası ağır ve hâlâ serialization failure retry'ı gerektirir.

**Pratik çözüm:** Locking — ya **optimistic** (`@Version`), ya **pessimistic** (`SELECT FOR UPDATE`). Banking'de bu seçim **iş kuralları + tahmini contention seviyesine** göre yapılır.

---

### 2. Optimistic Locking — `@Version`

**Felsefe:** "Çatışma az olur — biri olduğunda kullanıcıyı reddederim ve retry istersem isterim." Yani **iyimser**.

**Mekanik:**

1. Entity'ye `@Version` ile bir integer (genelde `long` veya `int`) kolon eklenir.
2. Her UPDATE'te Hibernate bu kolonu WHERE'e koyar ve değerini bir artırır:
   ```sql
   UPDATE accounts 
   SET balance_amount = ?, version = ? + 1 
   WHERE id = ? AND version = ?
   ```
3. Etkilenen satır sayısı 0 ise (çünkü başka biri version'u değiştirdi), Hibernate `OptimisticLockException` fırlatır.

**Banking örneği:**

```java
@Entity
@Table(name = "accounts")
public class AccountJpaEntity extends AuditableEntity {
    
    @Id
    private UUID id;
    
    @Column(name = "owner_id", nullable = false)
    private UUID ownerId;
    
    @Column(name = "currency", nullable = false, length = 3)
    private String currency;
    
    @Column(name = "balance_amount", nullable = false, precision = 19, scale = 4)
    private BigDecimal balanceAmount;
    
    @Version
    @Column(name = "version", nullable = false)
    private long version;
    
    // getters/setters (no setter for version — Hibernate yönetir)
}
```

Migration:

```sql
ALTER TABLE accounts ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
```

**Reprodüksiyon — lost update artık olmuyor:**

```java
@Test
@Transactional(propagation = Propagation.NOT_SUPPORTED)
void optimisticLockingShouldPreventLostUpdate() throws Exception {
    UUID accountId = createAccountWithBalance("1000.00");
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch startLatch = new CountDownLatch(1);
    AtomicReference<Exception> ex1 = new AtomicReference<>();
    AtomicReference<Exception> ex2 = new AtomicReference<>();
    
    Runnable withdraw800 = () -> transactionTemplate.executeWithoutResult(tx -> {
        try {
            startLatch.await();
            AccountJpaEntity a = repo.findById(accountId).orElseThrow();
            Thread.sleep(100);   // simulate processing delay
            a.setBalanceAmount(a.getBalanceAmount().subtract(new BigDecimal("800.00")));
            repo.saveAndFlush(a);
        } catch (Exception e) { ex1.set(e); }
    });
    
    Runnable withdraw500 = () -> transactionTemplate.executeWithoutResult(tx -> {
        try {
            startLatch.await();
            AccountJpaEntity a = repo.findById(accountId).orElseThrow();
            Thread.sleep(100);
            a.setBalanceAmount(a.getBalanceAmount().subtract(new BigDecimal("500.00")));
            repo.saveAndFlush(a);
        } catch (Exception e) { ex2.set(e); }
    });
    
    executor.submit(withdraw800);
    executor.submit(withdraw500);
    startLatch.countDown();
    executor.shutdown();
    executor.awaitTermination(5, TimeUnit.SECONDS);
    
    // Birinin OptimisticLockException alması garantili
    assertThat(
        ex1.get() instanceof OptimisticLockingFailureException
        || ex2.get() instanceof OptimisticLockingFailureException
    ).isTrue();
    
    BigDecimal finalBalance = repo.findById(accountId).orElseThrow().getBalanceAmount();
    // Sadece bir update kaldı: ya 200 ya 500
    assertThat(finalBalance).isIn(new BigDecimal("200.00"), new BigDecimal("500.00"));
}
```

**Hibernate'in fırlattığı exception:** `StaleObjectStateException` → Spring `OptimisticLockingFailureException` ile sarmalar (DataAccessException hierarchy).

**Avantajlar:**

- DB lock yok → throughput yüksek
- Read-heavy workload'da ideal (zaten çatışma az)
- Deadlock olmaz (lock yok)

**Dezavantajlar:**

- Çatışma anında **işlem reddedilir** → kullanıcı retry yapmalı
- Çatışma yoğunsa retry storm → ters performans

**Banking'de ne zaman:**

- Hesap profili güncelleme (telefon, adres) — çatışma çok düşük
- Hesap detayları edit ekranı (kullanıcı 5 dk düzeltir, save'ler)
- Microservice'te dağıtık veri — pessimistic lock kurumsal olarak imkânsız

---

### 3. Optimistic Lock + Retry pattern

Çatışma olduğunda otomatik retry uygula. **İki strateji:**

#### Strateji A — Spring Retry (`@Retryable`)

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

`@EnableRetry` ekle, sonra:

```java
@Service
public class WithdrawService {
    
    @Retryable(
        retryFor = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 50, multiplier = 2.0, maxDelay = 500)
    )
    @Transactional
    public void withdraw(AccountId accountId, Money amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.withdraw(amount);   // domain rule
        accountRepository.save(account);
    }
    
    @Recover
    public void recover(OptimisticLockingFailureException ex, AccountId accountId, Money amount) {
        // 3 deneme de başarısız oldu — kullanıcıya 409 Conflict dön
        throw new ConcurrentModificationException(
            "Hesap üzerinde eşzamanlı işlem var, lütfen tekrar deneyin"
        );
    }
}
```

**Uyarı:** `@Retryable` AOP proxy ile çalışır — **self-invocation tuzağı geçerli** (Topic 2.3'teki gibi). Aynı sınıf içinden çağrılırsa retry aktif değil.

**Dikkat 2:** `@Retryable` `@Transactional`'lı method'u retry ederken, **TX retry başında yeniden açılmalı** — yani method'a giriyor → TX açılıyor → fail → TX rollback → retry → yeni TX. Bunun için method seviyesinde `@Transactional` çok önemli. Eğer outer TX zaten açıksa retry işe yaramaz (aynı TX içinde version stale kalır).

#### Strateji B — Manuel retry loop

Kütüphane istemiyorsan veya finer control gerekiyorsa:

```java
@Service
public class WithdrawService {
    
    private static final int MAX_ATTEMPTS = 3;
    private static final long BASE_DELAY_MS = 50;
    
    @Autowired
    private WithdrawService self;   // self-injection (Topic 2.3)
    
    public void withdraw(AccountId id, Money amount) {
        OptimisticLockingFailureException lastEx = null;
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            try {
                self.doWithdraw(id, amount);
                return;   // success
            } catch (OptimisticLockingFailureException ex) {
                lastEx = ex;
                long delay = BASE_DELAY_MS * (long) Math.pow(2, attempt - 1);
                long jitter = ThreadLocalRandom.current().nextLong(0, delay / 2);
                try {
                    Thread.sleep(delay + jitter);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
        throw new ConcurrentModificationException(
            "Hesap güncellenemiyor — 3 deneme başarısız",
            lastEx
        );
    }
    
    @Transactional
    public void doWithdraw(AccountId id, Money amount) {
        Account account = accountRepository.findById(id).orElseThrow();
        account.withdraw(amount);
        accountRepository.save(account);
    }
}
```

**Exponential backoff + jitter:** Birden fazla worker aynı anda retry yaparsa "thundering herd" oluşur — hepsi aynı delay'de gelir. Random jitter bunu önler.

**Banking pratiği:** 3 denemeden sonra fail → 409 Conflict, kullanıcıya "tekrar deneyin" mesajı. Retry sayısı **idempotency** ile birleşince güvenli — aynı işlem iki kere uygulanmaz.

---

### 4. `@Version` data type seçimi ve tuzaklar

**Type tercihleri:**

- `long` / `Long` — en yaygın, overflow problemi yok
- `int` / `Integer` — ~2.1 milyar update sonrası overflow (banking için yetersiz)
- `Instant` / `Timestamp` — JPA destekler ama saat senkronizasyon problemi olabilir
- `Short` — anti-pattern

**Tuzak 1: Detached entity'i save**

```java
AccountJpaEntity entity = repo.findById(id).orElseThrow();  // version = 5
em.detach(entity);
// ... başka request entity'i update etti, DB'de version = 6
entity.setBalanceAmount(...);
repo.save(entity);   // → OptimisticLockingFailureException (DB version 6, entity version 5)
```

**Tuzak 2: Manuel version değiştirme**

```java
entity.setVersion(0);   // ❌ HİÇBİR ZAMAN
```

Bu Hibernate'i karıştırır. `@Version` field'a Java tarafından dokunulmaz.

**Tuzak 3: Version olmayan entity'i optimistic test etme**

```java
@Lock(LockModeType.OPTIMISTIC)   // ama @Version yok
Optional<AccountJpaEntity> findWithLock(UUID id);   
// Hibernate hiçbir şey yapmaz — version olmadan lock yok
```

---

### 5. Pessimistic Locking — `SELECT FOR UPDATE`

**Felsefe:** "Çatışma sık olur — okurken kilitlerim, kimse aynı zamanda alamaz." Yani **kötümser**.

**Mekanik:** DB seviyesinde **row-level lock**. `SELECT ... FOR UPDATE` çalıştırırsın, satır kilitlenir. Aynı satırı başka transaction `FOR UPDATE` ile veya UPDATE ile almak isterse **bekler** (veya timeout / hata, davranışa göre).

**LockModeType seçenekleri (JPA standardı):**

| LockMode | SQL eşdeğeri | Anlam |
|---|---|---|
| `NONE` | (yok) | Lock yok |
| `OPTIMISTIC` | (yok) | `@Version` check at flush |
| `OPTIMISTIC_FORCE_INCREMENT` | UPDATE version | Read-only erişimde bile version artır |
| `PESSIMISTIC_READ` | `SELECT FOR SHARE` (PostgreSQL) | Shared lock — başkaları okuyabilir, yazamaz |
| `PESSIMISTIC_WRITE` | `SELECT FOR UPDATE` | Exclusive lock — başkaları okuyamaz da, yazamaz da (DB'ye göre) |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT FOR UPDATE` + version++ | Lock + optimistic version |

**Banking'in en yaygın seçimi: `PESSIMISTIC_WRITE`.**

#### `@Lock` annotation — Spring Data JPA

```java
interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM AccountJpaEntity a WHERE a.id = :id")
    Optional<AccountJpaEntity> findByIdForUpdate(@Param("id") UUID id);
}
```

Üretilen SQL (PostgreSQL):

```sql
SELECT a.id, a.balance_amount, a.version, ... 
FROM accounts a 
WHERE a.id = ? 
FOR UPDATE;
```

#### Banking transfer örneği — pessimistic ile

```java
@Service
public class TransferService {
    
    @Transactional
    public void transfer(AccountId fromId, AccountId toId, Money amount) {
        // İki hesabı da kilitleyerek al
        AccountJpaEntity from = jpaRepo.findByIdForUpdate(fromId.value())
            .orElseThrow(() -> new AccountNotFoundException(fromId));
        AccountJpaEntity to = jpaRepo.findByIdForUpdate(toId.value())
            .orElseThrow(() -> new AccountNotFoundException(toId));
        
        if (from.getBalanceAmount().compareTo(amount.value()) < 0) {
            throw new InsufficientFundsException(fromId);
        }
        
        from.setBalanceAmount(from.getBalanceAmount().subtract(amount.value()));
        to.setBalanceAmount(to.getBalanceAmount().add(amount.value()));
        
        jpaRepo.save(from);
        jpaRepo.save(to);
        
        // journal_entry + 2 journal_line oluştur ...
    }
}
```

Şimdi paralel olarak iki transfer aynı `from` hesabı üzerinden gelirse, **biri kilitler, diğeri bekler**. İlki commit ettiğinde lock bırakılır, ikincisi yeni veriyle (eski transfer sonrası bakiye) çalışır. Lost update **imkânsız**.

**Trade-off:** Throughput düşer (bekleyen var). Banking'de para transferinde bu OK — doğruluk hızdan önemli.

---

### 6. Lock timeout — `NOWAIT`, `WAIT n`, `SKIP LOCKED`

`FOR UPDATE`'ı süresiz beklemek tehlikeli — bir transaction lock'ı tutarken diğerleri pool'u tüketebilir. Lock timeout şart.

#### `NOWAIT` — beklemeden hata

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "javax.persistence.lock.timeout", value = "0")  // 0 ms = NOWAIT
})
@Query("SELECT a FROM AccountJpaEntity a WHERE a.id = :id")
Optional<AccountJpaEntity> findByIdForUpdateNoWait(@Param("id") UUID id);
```

PostgreSQL SQL:

```sql
SELECT ... FROM accounts WHERE id = ? FOR UPDATE NOWAIT;
```

Eğer satır kilitliyse: anında `PessimisticLockException` fırlatır. Bekleme yok.

**Banking use case:** Hızlı API çağrısı — beklemektense reddetip kullanıcıyı tekrar denemeye yönlendirmek.

#### `WAIT n` (Oracle terminolojisi) / `lock_timeout` (PostgreSQL)

```java
@QueryHints({
    @QueryHint(name = "javax.persistence.lock.timeout", value = "3000")  // 3 saniye bekle
})
```

PostgreSQL'de bu hint genelde `SET LOCAL lock_timeout` ile uygulanır. 3 saniyede lock alınamazsa exception.

**Banking pratiği:** API endpoint'lerinde 2-5 saniye timeout. Batch job'larda 30+ saniye OK.

#### `SKIP LOCKED` — queue pattern

```java
@Query(value = """
    SELECT * FROM pending_transactions 
    WHERE status = 'PENDING' 
    ORDER BY created_at 
    LIMIT :limit 
    FOR UPDATE SKIP LOCKED
""", nativeQuery = true)
List<PendingTransactionEntity> claimNextBatch(@Param("limit") int limit);
```

Birden fazla worker aynı tabloyu çekiyor. **Kilit gördüğü satırı atlar**, başkasını alır. Her worker farklı batch alır, çatışma yok.

**Banking use case:** Pending transfer queue, asenkron fee hesaplama, retry kuyruğu. Multi-worker'lı job processing'in altın standardı.

**Tuzak:** Sadece PostgreSQL 9.5+, Oracle, SQL Server 2016+ destekler. MySQL InnoDB 8.0.1+.

---

### 7. Deadlock — banking'in klasik tuzağı

İki transaction farklı sırada aynı kaynakları kilitlerse → deadlock.

**Senaryo: A → B ve B → A transfer aynı anda**

```
T1: SELECT account A FOR UPDATE   (lock A)
T2: SELECT account B FOR UPDATE   (lock B)
T1: SELECT account B FOR UPDATE   → waits for T2
T2: SELECT account A FOR UPDATE   → waits for T1
                                     ↑
                                  DEADLOCK
```

DB deadlock'ı tespit eder (PostgreSQL ~1 saniye sonra) ve **birini öldürür**: `40P01: deadlock detected`.

#### Reprodüksiyon kodu

```java
@SpringBootTest
@Testcontainers
class DeadlockReproductionTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired TransferService transferService;
    @Autowired AccountRepository accountRepository;
    
    UUID accountA, accountB;
    
    @BeforeEach
    void setup() {
        accountA = createAccountWithBalance("10000.00");
        accountB = createAccountWithBalance("10000.00");
    }
    
    @Test
    void shouldReproduceDeadlock() throws Exception {
        int iterations = 100;
        ExecutorService executor = Executors.newFixedThreadPool(20);
        AtomicInteger successes = new AtomicInteger();
        AtomicInteger deadlocks = new AtomicInteger();
        AtomicInteger otherErrors = new AtomicInteger();
        
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < iterations; i++) {
            // A → B
            futures.add(executor.submit(() -> {
                try {
                    transferService.transfer(
                        new AccountId(accountA),
                        new AccountId(accountB),
                        Money.of("10.00", "TRY")
                    );
                    successes.incrementAndGet();
                } catch (CannotAcquireLockException e) {
                    deadlocks.incrementAndGet();
                } catch (Exception e) {
                    otherErrors.incrementAndGet();
                }
            }));
            // B → A
            futures.add(executor.submit(() -> {
                try {
                    transferService.transfer(
                        new AccountId(accountB),
                        new AccountId(accountA),
                        Money.of("10.00", "TRY")
                    );
                    successes.incrementAndGet();
                } catch (CannotAcquireLockException e) {
                    deadlocks.incrementAndGet();
                } catch (Exception e) {
                    otherErrors.incrementAndGet();
                }
            }));
        }
        
        for (Future<?> f : futures) f.get(30, TimeUnit.SECONDS);
        executor.shutdown();
        
        System.out.println("Success: " + successes.get());
        System.out.println("Deadlock: " + deadlocks.get());
        System.out.println("Other: " + otherErrors.get());
        
        // Sıralama yokken deadlock kaçınılmaz
        assertThat(deadlocks.get()).isGreaterThan(0);
    }
}
```

Bu test'i çalıştırdığında PostgreSQL log'da:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 567; blocked by process 5678.
        Process 5678 waits for ShareLock on transaction 568; blocked by process 1234.
HINT: See server log for query details.
```

Spring tarafında `CannotAcquireLockException` (DataAccessException) yakalanır.

#### `jstack` ile deadlock analizi (uygulama tarafında)

Bazen deadlock DB seviyesinde değil, **Java thread seviyesinde** olur (lock objesi, monitor). Bunu `jstack` ile görürsün:

```bash
jps                                  # PID'i bul
jstack -l <pid> > thread-dump.txt    # full dump
```

Çıktıda:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007fbe9c008f78 (object 0x00000007abc...,
  a com.mavibank.banking.Account),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007fbe9c008e38 (object 0x00000007def...,
  a com.mavibank.banking.Account),
  which is held by "Thread-1"
```

DB deadlock için `pg_stat_activity` ve `pg_locks` view'larından okursun. Banking üretim ortamında bunu monitoring'e bağlamak şart.

---

### 8. Deadlock fix — Lock Ordering

**Çözüm:** Tüm transaction'lar aynı sırada kilitlesin. `account.id` sıralı al, küçük olanı önce.

```java
@Service
public class TransferService {
    
    @Transactional
    public void transfer(AccountId fromId, AccountId toId, Money amount) {
        if (fromId.equals(toId)) {
            throw new SelfTransferException();
        }
        
        // Lock ordering: küçük ID önce
        AccountId firstLockId, secondLockId;
        if (fromId.value().compareTo(toId.value()) < 0) {
            firstLockId = fromId;
            secondLockId = toId;
        } else {
            firstLockId = toId;
            secondLockId = fromId;
        }
        
        AccountJpaEntity first = jpaRepo.findByIdForUpdate(firstLockId.value()).orElseThrow();
        AccountJpaEntity second = jpaRepo.findByIdForUpdate(secondLockId.value()).orElseThrow();
        
        // Hangisi from, hangisi to olduğunu ayırt et
        AccountJpaEntity from = firstLockId.equals(fromId) ? first : second;
        AccountJpaEntity to = firstLockId.equals(fromId) ? second : first;
        
        if (from.getBalanceAmount().compareTo(amount.value()) < 0) {
            throw new InsufficientFundsException(fromId);
        }
        
        from.setBalanceAmount(from.getBalanceAmount().subtract(amount.value()));
        to.setBalanceAmount(to.getBalanceAmount().add(amount.value()));
        
        jpaRepo.save(from);
        jpaRepo.save(to);
    }
}
```

Şimdi A→B ve B→A senaryosunda: ikisi de önce min(A,B), sonra max(A,B) kilitler. **Sıra aynı → deadlock yok.** İkinci transaction birinci bittikten sonra çalışır.

**Çoklu hesap senaryosu** (örnek: 5 hesabı kilitleyen bir batch): hepsini ID'ye göre `sort` et, sırayla kilitle.

```java
List<AccountId> sortedIds = accountIds.stream()
    .sorted(Comparator.comparing(AccountId::value))
    .toList();

List<AccountJpaEntity> locked = new ArrayList<>();
for (AccountId id : sortedIds) {
    locked.add(jpaRepo.findByIdForUpdate(id.value()).orElseThrow());
}
```

---

### 9. PostgreSQL SERIALIZABLE — serialization failure ve retry

PostgreSQL'in SERIALIZABLE isolation'ı **SSI** (Serializable Snapshot Isolation) algoritmasını kullanır. Lock almaz, ama transaction'lar bir "uygun sıra" oluşturamazsa **commit'i reddeder**:

```
ERROR: could not serialize access due to read/write dependencies among transactions
SQLSTATE: 40001
```

Spring tarafında `CannotSerializeTransactionException`. **Retry edilmeli.**

```java
@Service
public class SerializableTransferService {
    
    private static final int MAX_ATTEMPTS = 5;
    
    @Autowired
    private SerializableTransferService self;
    
    public void transfer(TransferRequest req) {
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            try {
                self.doTransfer(req);
                return;
            } catch (CannotSerializeTransactionException ex) {
                if (attempt == MAX_ATTEMPTS) {
                    throw new TransferRetryExhaustedException(req.idempotencyKey(), ex);
                }
                sleepWithBackoff(attempt);
            }
        }
    }
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void doTransfer(TransferRequest req) {
        // ... transfer logic, lock-free
    }
    
    private void sleepWithBackoff(int attempt) {
        long base = 20L * (1L << (attempt - 1));   // 20, 40, 80, 160, 320 ms
        long jitter = ThreadLocalRandom.current().nextLong(0, base);
        try {
            Thread.sleep(base + jitter);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

**SERIALIZABLE'in pessimistic lock'a göre avantajı:**

- Lock yok → deadlock yok (DB tarafında)
- Read-heavy workload'da daha iyi
- DB tek algoritmayı yönetir, uygulama lock ordering ile uğraşmaz

**Dezavantajı:**

- Retry oranı yüksek olabilir (contention'a göre)
- PostgreSQL-specific algoritma — Oracle/MySQL'de SERIALIZABLE farklı (lock-based, deadlock olabilir)

**Banking pratiği:** PostgreSQL'de SERIALIZABLE + idempotency-key + retry kombinasyonu çok güçlü pattern. Birçok TR fintech bu pattern'i kullanır.

---

### 10. Pessimistic vs Optimistic — karar matrisi

| Kriter | Optimistic | Pessimistic |
|---|---|---|
| Çatışma oranı | Düşük (< %10) | Yüksek (> %10) |
| TX süresi | Uzun (kullanıcı düşünür, edit eder) | Kısa (atomik DB işlemi) |
| Cluster içi mi? | Tek node — pessimistic OK | Distributed → optimistic (veya distributed lock) |
| Deadlock toleransı | Sıfır (lock yok) | Var (lock ordering şart) |
| Retry stratejisi | Mutlaka | Opsiyonel |
| Throughput | Yüksek | Düşük |
| Banking örnekleri | Profile update, hesap detay edit | Para transferi, bakiye kontrolü |

**Banking pratiği:**

- **Money movement** (transfer, deposit, withdraw): Pessimistic + lock ordering
- **Account configuration** (limit değişimi, isim): Optimistic + retry
- **Reporting** (read-only): Lock yok, `readOnly = true`, snapshot isolation
- **Distributed transaction** (microservice): Saga / Outbox + optimistic local

---

### 11. Lock granularity — row vs table vs custom

**Row-level lock:** `SELECT FOR UPDATE` — sadece o satırlar kilitlenir. **Yaygın olan.**

**Table-level lock:** `LOCK TABLE accounts IN EXCLUSIVE MODE;` — tüm tablo kilitli. Sadece migration veya DDL benzeri operasyon için. Banking'de operasyonel kullanımı **yok**.

**Advisory lock** (PostgreSQL özelliği): Custom semantic lock. `pg_advisory_xact_lock(account_id_hash)`. Bir transaction içinde alınır, commit/rollback'te bırakılır. Uygulama-seviyesi mutex gibi.

```sql
-- Aynı transaction'da
SELECT pg_advisory_xact_lock(hashtext('transfer-' || :idempotencyKey));
-- ... transfer işlemi ...
```

Banking'de **idempotency lock** için faydalı: aynı idempotency-key için iki istek geldiğinde, advisory lock ikincinin beklemesini sağlar.

---

### 12. `LockModeType.OPTIMISTIC_FORCE_INCREMENT` — kullanım senaryosu

Bir transaction sadece bir aggregate'i **okuyor** ama version'unu artırmak istiyor (parent-child invariant'ı korumak için).

**Banking örneği:** `Account` (parent) ve `JournalLine` (child). Yeni bir journal line eklediğinde, Account aggregate root'unun version'u artmalı — başka bir transaction Account'u eski snapshot'ta görüp inconsistent değişiklik yapmasın.

```java
@Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
@Query("SELECT a FROM AccountJpaEntity a WHERE a.id = :id")
Optional<AccountJpaEntity> findAndIncrementVersion(@Param("id") UUID id);
```

Account hiç değişmese bile, flush zamanında `UPDATE accounts SET version = version + 1 WHERE id = ? AND version = ?` çıkar.

DDD aggregate invariant pattern'in JPA implementasyonu.

---

### 13. Banking-grade retry — exponential backoff + jitter + circuit breaker

Tam production-grade retry pattern:

```java
@Service
public class ResilientTransferService {
    
    private static final int MAX_ATTEMPTS = 5;
    private static final long BASE_DELAY_MS = 50;
    private static final long MAX_DELAY_MS = 2000;
    
    @Autowired
    private ResilientTransferService self;
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public void transfer(TransferRequest req) {
        Timer.Sample sample = Timer.start(meterRegistry);
        Exception lastEx = null;
        
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            try {
                self.doTransfer(req);
                meterRegistry.counter("transfer.success", 
                    "attempts", String.valueOf(attempt)).increment();
                sample.stop(meterRegistry.timer("transfer.duration"));
                return;
            } catch (OptimisticLockingFailureException 
                   | CannotSerializeTransactionException 
                   | CannotAcquireLockException ex) {
                lastEx = ex;
                meterRegistry.counter("transfer.retry", 
                    "reason", ex.getClass().getSimpleName()).increment();
                
                if (attempt == MAX_ATTEMPTS) break;
                sleepWithBackoff(attempt);
            } catch (InsufficientFundsException | AccountNotFoundException ex) {
                // Business exception — retry etme
                throw ex;
            }
        }
        
        meterRegistry.counter("transfer.failure", "reason", "retry_exhausted").increment();
        throw new TransferRetryExhaustedException(req.idempotencyKey(), lastEx);
    }
    
    @Transactional
    public void doTransfer(TransferRequest req) {
        // ... lock ordering + transfer logic
    }
    
    private void sleepWithBackoff(int attempt) {
        long exponential = BASE_DELAY_MS * (1L << (attempt - 1));
        long capped = Math.min(exponential, MAX_DELAY_MS);
        long jitter = ThreadLocalRandom.current().nextLong(0, capped);
        long total = capped + jitter;
        
        try {
            Thread.sleep(total);
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(ie);
        }
    }
}
```

**Kritik noktalar:**

1. **Sadece concurrency exception'larını retry et.** Business exception (insufficient funds) retry edilmez — sonsuza kadar aynı hata.
2. **Jitter şart.** Aynı anda 100 retry, hepsi aynı delay sonrası → ikinci dalga deadlock.
3. **Max delay cap.** Exponential 2^10 = 1024, 1024 * 50ms = 51 saniye. Banking'de bu çok uzun. Cap koy.
4. **Idempotency-key**: Retry sırasında DB'de aynı kayıt iki kere oluşmasın. Bu pattern Phase 5'te detaylı.
5. **Metrics**: Retry sayısı, hangi exception, hangi attempt success — Prometheus'a yaz. SLO için kritik.

---

### 14. Anti-pattern'ler

**Anti-pattern 1: `@Version` olmadan optimistic test etmek**

`@Lock(LockModeType.OPTIMISTIC)` ama entity'de `@Version` yok → hiçbir şey yapmaz. Sessiz fail.

**Anti-pattern 2: Lock'lu okumanın sonucunu cache'lemek**

```java
@Cacheable("accounts")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Account findById(...);   // ❌
```

Cache hit'te lock alınmaz, garantisiz davranış.

**Anti-pattern 3: Long-running pessimistic lock**

```java
@Transactional
public void process() {
    AccountJpaEntity a = repo.findByIdForUpdate(id).orElseThrow();
    externalService.callApi(...);   // 10 saniye sürer
    // Lock 10 saniye boyunca açık → pool'u tüketir
}
```

Kural: Pessimistic lock'lu TX **mümkün olduğunca kısa**. External call YOK.

**Anti-pattern 4: Lock ordering'i unutmak**

A→B ve B→A senaryosunda her zaman lock ordering. **Test et.**

**Anti-pattern 5: Retry'da business exception yutmak**

```java
for (...) {
    try { ... } 
    catch (Exception e) { ... }   // ❌ InsufficientFunds da retry edilir
}
```

Sadece concurrency exception'larını yakala.

**Anti-pattern 6: Retry sonsuza kadar**

```java
while (true) { try { ... } catch (...) { ... } }
```

Always max attempt + cap + fail-fast yol. Sonsuza kadar retry = canlı kilitlenme.

**Anti-pattern 7: `findById` sonra `findByIdForUpdate`**

```java
Optional<Account> a = repo.findById(id);   // SELECT (lock yok)
// ...
Optional<Account> locked = repo.findByIdForUpdate(id);   // 2. SELECT, lock
```

Race condition: ikisi arasında başka biri güncelleyebilir. **İlk SELECT'i `FOR UPDATE` ile yap.**

**Anti-pattern 8: Distributed lock için DB tabanlı çözümü hafife almak**

Mikroservislerde "ben DB'de bir lock tablosu yaparım" → idempotency için OK, ama gerçek dağıtık lock için Redis Redlock, Zookeeper, etcd gibi araçlar daha sağlam. Banking'de Phase 7'de detay.

---

## Önemli olabilecek araştırma kaynakları

- Hibernate ORM User Guide — Locking chapter
- PostgreSQL docs — Explicit Locking, Serialization Failure
- Vlad Mihalcea "High-Performance Java Persistence" — Concurrency Control
- "Designing Data-Intensive Applications" Martin Kleppmann — Chapter 7 (Transactions)
- Spring Retry official docs
- "Concurrency Control in Distributed Systems" — Vogels (Werner) blog
- PostgreSQL "Serializable Snapshot Isolation" — Kevin Grittner paper
- `pg_locks`, `pg_stat_activity` views

---

## Mini task'ler

### Task 2.4.1 — `@Version` ekle (30 dk)

`AccountJpaEntity` ve `JournalEntryJpaEntity`'e `@Version` ekle. Migration V5 yaz:

```sql
ALTER TABLE accounts ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
ALTER TABLE journal_entries ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
```

Mevcut testlerin geçtiğini doğrula. Sonra optimistic conflict üreten bir test yaz (yukarıdaki örnek).

### Task 2.4.2 — Manuel retry loop ile `WithdrawService` (45 dk)

`WithdrawService.withdraw(AccountId, Money)` method'u:

- Maksimum 3 deneme
- Exponential backoff (50ms, 100ms, 200ms) + jitter
- Sadece `OptimisticLockingFailureException` retry edilsin
- 3 başarısız sonra `ConcurrentModificationException` fırlat (mapping: 409 Conflict)

`@Retryable` kullanma — manuel loop ile öğren.

### Task 2.4.3 — `@Retryable` ile aynı kalibrasyon (30 dk)

Aynı `WithdrawService`'in `@Retryable` versiyonunu yaz. Self-injection veya separate bean ile self-invocation problem'inden kaçın.

İki implementasyonu test et:
- Aynı concurrency davranışı mı?
- Hangisi daha temiz?
- Defter notu: Hangisini production'da seçersin, neden?

### Task 2.4.4 — `PESSIMISTIC_WRITE` ile `TransferService` (1 saat)

`TransferService.transfer(...)`'i pessimistic locking ile yeniden yaz:

- `findByIdForUpdate` method'u JPA repository'de `@Lock` ile
- Lock ordering: account ID'ye göre küçükten büyüğe
- Lock timeout: 3 saniye (`@QueryHints` ile)
- Transfer logic: bakiye kontrolü, debit, credit, journal_entry, 2 journal_line

Test: paralel A→B + B→A 100 iterasyon — **0 deadlock** olmalı.

### Task 2.4.5 — Deadlock reprodüksiyon ve fix karşılaştırması (1 saat)

İki versiyon kod yaz:

- **A:** Lock ordering YOK (her zaman `from` önce, `to` sonra)
- **B:** Lock ordering VAR (sorted by ID)

Aynı concurrency test'i (A↔B 100 iterasyon, 20 thread) ile ikisini çalıştır:

- A'da kaç deadlock?
- B'de kaç deadlock?
- A'da success / B'de success oranı?

`jstack` çıktısı al — A versiyonu çalışırken `jps` ile PID bul, `jstack -l <pid>` ile thread dump. Çıktıyı `docs/deadlock-analysis.md`'a yapıştır.

### Task 2.4.6 — PostgreSQL SERIALIZABLE alternatifi (45 dk)

`SerializableTransferService` yaz:
- `@Transactional(isolation = Isolation.SERIALIZABLE)`
- Lock yok (`FOR UPDATE` kullanma)
- 5 retry exponential backoff
- `CannotSerializeTransactionException` yakala

Aynı concurrency test'i ile çalıştır. Performans karşılaştırması:
- Pessimistic version: ms/transfer ortalama
- Serializable version: ms/transfer ortalama
- Hangisinde retry sayısı yüksek?

Defter notu: Hangi yaklaşım hangi senaryoda?

### Task 2.4.7 — `SKIP LOCKED` ile job queue (45 dk)

`pending_transactions` tablosu (yeni migration):

```sql
CREATE TABLE pending_transactions (
    id UUID PRIMARY KEY,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    claimed_at TIMESTAMP WITH TIME ZONE,
    claimed_by VARCHAR(100)
);

CREATE INDEX idx_pending_status ON pending_transactions(status, created_at);
```

Repository method:

```java
@Query(value = """
    SELECT * FROM pending_transactions 
    WHERE status = 'PENDING' 
    ORDER BY created_at 
    LIMIT :limit 
    FOR UPDATE SKIP LOCKED
""", nativeQuery = true)
List<PendingTransactionEntity> claimBatch(@Param("limit") int limit);
```

Service:

```java
@Scheduled(fixedDelay = 1000)
@Transactional
public void processNextBatch() {
    List<PendingTransactionEntity> batch = repo.claimBatch(10);
    for (var pending : batch) {
        pending.setStatus("PROCESSING");
        pending.setClaimedAt(Instant.now());
        pending.setClaimedBy(workerId);
        // ... process
    }
}
```

İki uygulama instance'ı aç (farklı port), 50 pending kayıt insert et, log'larda ne gördüğünü gözle. **Aynı kayıt iki instance'ta görünmemeli**.

### Task 2.4.8 — Banking-grade `ResilientTransferService` (1 saat)

Yukarıdaki tam örneği uyarla:
- 5 attempt
- Base delay 50ms, max delay 2 saniye
- Jitter
- Concurrency exception'larını yakala, business'i yakalama
- Micrometer metrics (`transfer.retry`, `transfer.success`, `transfer.failure`)

`/actuator/metrics/transfer.retry` endpoint'inde retry sayısını gör.

---

## Test yazma rehberi

### Test 2.4.1 — Optimistic lock conflict

```java
@SpringBootTest
@Testcontainers
class OptimisticLockTest {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired AccountJpaRepository repo;
    @Autowired TransactionTemplate tx;
    
    @Test
    void parallelUpdatesShouldFailOne() throws Exception {
        UUID id = createAccount("1000.00");
        CountDownLatch latch = new CountDownLatch(1);
        AtomicReference<Exception> ex1 = new AtomicReference<>(), ex2 = new AtomicReference<>();
        
        Runnable r1 = () -> tx.executeWithoutResult(s -> {
            try {
                AccountJpaEntity a = repo.findById(id).orElseThrow();
                latch.await();
                a.setBalanceAmount(a.getBalanceAmount().subtract(new BigDecimal("100")));
                repo.saveAndFlush(a);
            } catch (Exception e) { ex1.set(e); }
        });
        
        Runnable r2 = () -> tx.executeWithoutResult(s -> {
            try {
                AccountJpaEntity a = repo.findById(id).orElseThrow();
                latch.await();
                a.setBalanceAmount(a.getBalanceAmount().subtract(new BigDecimal("200")));
                repo.saveAndFlush(a);
            } catch (Exception e) { ex2.set(e); }
        });
        
        Thread t1 = new Thread(r1), t2 = new Thread(r2);
        t1.start(); t2.start();
        Thread.sleep(100);
        latch.countDown();
        t1.join(); t2.join();
        
        boolean oneFailed = 
            ex1.get() instanceof OptimisticLockingFailureException
            || ex2.get() instanceof OptimisticLockingFailureException;
        assertThat(oneFailed).isTrue();
    }
}
```

### Test 2.4.2 — Retry success on second attempt

```java
@Test
void retryShouldSucceedAfterTransientConflict() {
    UUID id = createAccount("1000.00");
    
    // İlk SELECT sonrası başka bir thread version'u artırsın
    doAnswer(invocation -> {
        UUID accountId = invocation.getArgument(0);
        Optional<AccountJpaEntity> result = repo.findById(accountId);
        // İlk denemede stale entity dön — gerçek kullanım simülasyonu yok ama
        // unit test düzeyinde mock'la veya integration test'te race koş
        return result;
    }).when(/*...*/);
    
    // Daha temiz: integration test'te paralel iki withdraw, biri retry'da geçecek
    // ... (yukarıdaki gibi)
}
```

### Test 2.4.3 — Deadlock zero with lock ordering

```java
@Test
void lockOrderingShouldEliminateDeadlock() throws Exception {
    UUID a = createAccount("100000.00");
    UUID b = createAccount("100000.00");
    
    ExecutorService exec = Executors.newFixedThreadPool(20);
    AtomicInteger deadlocks = new AtomicInteger();
    int iterations = 100;
    CountDownLatch done = new CountDownLatch(iterations * 2);
    
    for (int i = 0; i < iterations; i++) {
        exec.submit(() -> {
            try {
                transferService.transfer(new AccountId(a), new AccountId(b), Money.of("1", "TRY"));
            } catch (CannotAcquireLockException e) {
                deadlocks.incrementAndGet();
            } finally { done.countDown(); }
        });
        exec.submit(() -> {
            try {
                transferService.transfer(new AccountId(b), new AccountId(a), Money.of("1", "TRY"));
            } catch (CannotAcquireLockException e) {
                deadlocks.incrementAndGet();
            } finally { done.countDown(); }
        });
    }
    
    done.await(30, TimeUnit.SECONDS);
    exec.shutdown();
    
    assertThat(deadlocks.get()).isZero();
}
```

### Test 2.4.4 — `SKIP LOCKED` queue test

İki transaction simultaneously aynı `claimBatch(5)` çağrısı yapsın, sonuçların **kesişimi boş** olmalı. PostgreSQL gerçek behavior'u TestContainers ile test.

---

## Claude-verify prompt

```
Banking domain'inde yazdığım concurrency / locking kodumu aşağıdaki kriterlere göre 
değerlendir. PASS / FAIL / EKSIK işaretle, KOD YAZMA, sadece neyin yanlış olduğunu söyle:

1. Optimistic locking:
   - `@Version` field'ı tüm domain entity'lerde var mı (Account, JournalEntry)?
   - `@Version` data type long mu (int değil)?
   - Field'a Java tarafından setter var mı? (Olmamalı)
   - DB migration ile version kolonu eklenmiş mi? Default 0?
   - `OptimisticLockingFailureException` test ile reprodüksiyon edildi mi?

2. Optimistic retry:
   - Sadece concurrency exception'ları (OptimisticLocking, CannotSerialize, CannotAcquireLock) 
     retry ediliyor mu?
   - Business exception (InsufficientFunds) retry ediliyor mu? (Olmamalı)
   - Max attempts + max delay cap var mı?
   - Jitter ekli mi (thundering herd önleme)?
   - Manuel veya @Retryable — self-invocation problemi çözülmüş mü?

3. Pessimistic locking:
   - `@Lock(LockModeType.PESSIMISTIC_WRITE)` ile `findByIdForUpdate` method'u var mı?
   - Method `@Query` ile yazılmış mı (derived query yetmez)?
   - Lock timeout (`javax.persistence.lock.timeout` query hint) var mı?
   - TransferService bunu kullanıyor mu?

4. Lock ordering:
   - Birden fazla hesap kilitlenirken ID sırasıyla mı?
   - Comparator hatasız mı (compareTo)?
   - 100 iterasyon A↔B test'i 0 deadlock üretiyor mu?

5. Deadlock analizi:
   - `docs/deadlock-analysis.md` jstack çıktısı + analiz ile yazılı mı?
   - Lock ordering öncesi vs sonrası deadlock sayıları karşılaştırılmış mı?

6. PostgreSQL SERIALIZABLE alternatifi:
   - SerializableTransferService implementasyonu var mı?
   - CannotSerializeTransactionException yakalanıp retry mı?
   - Performans karşılaştırması (pessimistic vs serializable) defter notunda mı?

7. SKIP LOCKED:
   - pending_transactions tablosu + claimBatch native query mevcut mu?
   - İki instance'lı test ile aynı kayıt iki yerde görülmüyor mu?

8. Retry pattern:
   - Exponential backoff doğru mu (2^n)?
   - Max delay cap ile sonsuz büyüme önlenmiş mi?
   - Jitter random aralıkta mı?
   - Metric (transfer.retry, transfer.success) Micrometer'a yazılıyor mu?

9. Anti-pattern:
   - `@Version` olmadan @Lock(OPTIMISTIC) var mı? (Olmamalı)
   - `findById` sonra `findByIdForUpdate` race condition var mı?
   - Pessimistic TX içinde external HTTP call var mı? (Olmamalı)
   - Retry sonsuza kadar mı? Cap var mı?
   - Cache'lenen entity üzerinde lock kullanılmış mı?

10. Banking-grade kalite:
    - Idempotency-key concurrency korumasıyla birleştirilmiş mi?
    - Money movement (transfer/withdraw/deposit) pessimistic + lock ordering mi?
    - Read-only reporting `readOnly = true` mu?
    - Retry exhausted sonrası exception 409 Conflict'e map ediliyor mu?

Her madde için PASS / FAIL / EKSIK ve kısa gerekçe. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] `@Version` tüm money-movement entity'lerinde aktif, DB'de NOT NULL DEFAULT 0
- [ ] Optimistic conflict reprodüksiyon test'i geçiyor
- [ ] Manuel retry loop ve `@Retryable` versiyonlarını her ikisini de yazdım
- [ ] `PESSIMISTIC_WRITE` ile `findByIdForUpdate` repository method'u var
- [ ] Lock timeout query hint ile 3 saniye limit
- [ ] Lock ordering ile `TransferService` 100 iterasyon A↔B test'inde 0 deadlock
- [ ] Lock ordering YOK versiyonunda deadlock üretildi, jstack analizi `docs/`'a kaydedildi
- [ ] PostgreSQL SERIALIZABLE alternatifi yazıldı, performans karşılaştırıldı
- [ ] `SKIP LOCKED` ile job queue pattern çalıştı, multi-instance test geçti
- [ ] Exponential backoff + jitter + max delay cap doğru
- [ ] Retry metrics Micrometer'a yazılıyor (`/actuator/metrics/transfer.retry` gözüktü)
- [ ] Anti-pattern listesi rahat — özellikle "lock ordering unutmak", "business exception retry"

Hepsi onaylı → Topic 2.5'e geç → [05-n-plus-one/](../05-n-plus-one/index.md)

---

## Defter notları

1. "Optimistic locking'in temeli `@Version` kolonu + UPDATE WHERE version = ?. Çatışma olursa ____ exception fırlatılır."
2. "Pessimistic locking'in SQL karşılığı ____. PostgreSQL'de `FOR UPDATE` ile `FOR SHARE` farkı ____."
3. "`NOWAIT` ile `lock_timeout` farkı ____. Banking'de hangi senaryoda hangisi: ____."
4. "`SKIP LOCKED`'ın asıl kullanım senaryosu ____ (queue pattern). Hangi DB versiyonlarında destekli: ____."
5. "Deadlock üretmek için minimum şart: ____. Lock ordering ile çözüm prensibi: ____."
6. "Exponential backoff formülü ____. Jitter neden eklenir: ____."
7. "PostgreSQL SERIALIZABLE algoritması (SSI) lock yerine ____ kullanır. Pessimistic'e göre avantajı: ____."
8. "Optimistic vs Pessimistic karar matrisi 3 boyutta: ____, ____, ____."
9. "`OPTIMISTIC_FORCE_INCREMENT` ne zaman kullanılır (DDD aggregate pattern): ____."
10. "Banking-grade retry pattern'in 5 zorunlu unsuru: ____, ____, ____, ____, ____."
