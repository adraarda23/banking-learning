# Topic 7.7 — Distributed Locks & Transactions

## Hedef

Microservice ortamında **distributed lock** ve **distributed transaction** çözümlerini banking-grade seviyede uygulamak. Redis Redlock, ZooKeeper, ShedLock, pg_advisory_lock pattern'leri. 2PC neden production banking'de kötü, alternatif olarak Saga (Phase 6'da gördük) ve TCC. Idempotency cross-service.

## Süre

Okuma: 2 saat • Mini task: 1.5 saat • Test: 30 dk • Toplam: ~4 saat

## Önbilgi

- Topic 7.1-7.6 bitti (microservice architecture)
- Phase 4 Topic 6 (DB concurrency) — SELECT FOR UPDATE, DBMS_LOCK
- Phase 5 Topic 6 (Scheduling) — ShedLock
- Phase 6 Topic 7 (Saga pattern)

---

## Kavramlar

### 1. Distributed lock — kullanım senaryoları

Microservice'lerde **mutex** ihtiyacı:

1. **Singleton job:** EOD reconciliation 3 instance'tan **tek**inde çalışsın
2. **Exclusive resource:** Bir kart için aynı anda **tek transaction**
3. **Race condition prevention:** Cross-service operation'da
4. **Cache stampede:** Çok thread aynı değeri compute etmeye çalışır → sadece 1
5. **Distributed singleton:** Service initialization'ı bir instance yapsın

**Banking örnek:**

```
3 instance running EOD reconciliation job
Cron 23:55'te tetiklenir → 3 instance aynı anda başlar
   ↓ distributed lock olmadan
3 paralel run → duplicate işlem, mismatch report 3 kez yazılır → felaket
```

### 2. Lock semantic — distributed sistem zorluğu

**Single-machine lock:**
- Mutex, ReentrantLock — JVM içinde
- Thread crash → JVM ownership tracking
- **Reliable**

**Distributed lock:**
- Network'le iletişim
- Process crash → lock release nasıl?
- Network partition → split-brain
- Clock drift → TTL belirsiz
- **Hard problems**

Banking için pragmatik: **DB-level lock primary, distributed lock advisory**.

### 3. Redis SETNX with TTL — basic distributed lock

```java
@Service
public class RedisDistributedLock {
    
    private final StringRedisTemplate redis;
    
    public Optional<String> tryLock(String key, Duration ttl) {
        String token = UUID.randomUUID().toString();
        Boolean acquired = redis.opsForValue().setIfAbsent(
            "lock:" + key, token, ttl
        );
        return Boolean.TRUE.equals(acquired) ? Optional.of(token) : Optional.empty();
    }
    
    public boolean unlock(String key, String token) {
        // Lua script — atomic check-and-delete
        String script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
        """;
        Long result = redis.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList("lock:" + key),
            token
        );
        return result != null && result == 1L;
    }
}
```

**Kritik detaylar:**

1. **SETNX + TTL atomic:** `setIfAbsent(key, value, ttl)` — both in one operation. Yoksa: SETNX → app crash → lock sonsuza kalır.

2. **Token-based ownership:** UUID token. Unlock sadece **owner** yapabilir. Lua script atomic check-and-delete.

3. **TTL — fail-safe:** Lock holder crash etse bile TTL sonrası release.

#### Banking kullanımı

```java
@Service
public class CardTransactionService {
    
    private final RedisDistributedLock lock;
    
    public void processCardTransaction(UUID cardId, BigDecimal amount) {
        Optional<String> token = lock.tryLock("card:" + cardId, Duration.ofSeconds(30));
        
        if (token.isEmpty()) {
            throw new ResourceBusyException(
                "Card transaction already in progress for: " + cardId);
        }
        
        try {
            // Exclusive operation
            cardService.processTransaction(cardId, amount);
        } finally {
            lock.unlock("card:" + cardId, token.get());
        }
    }
}
```

Aynı kartla 2 paralel transaction → ikincisi `ResourceBusyException` → retry.

### 4. Redlock — Redis cluster için

**Single Redis fail edebilir.** Redlock — multiple Redis instance'a quorum-based locking.

```java
RedissonClient redisson = Redisson.create(config);
RLock lock = redisson.getLock("card:" + cardId);

if (lock.tryLock(0, 30, TimeUnit.SECONDS)) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

**Martin Kleppmann critique:** Redlock asynchronous network'te 100% güvenli değil:
- Clock drift problem
- GC pause sırasında token expired olabilir
- Process pause sonrası dual ownership

**Pragmatik karar (banking):**
- **Non-critical** (cache invalidation, rate limit) → Redis SETNX yeterli
- **Critical** (financial transaction) → DB-level lock (SELECT FOR UPDATE, optimistic locking) tercih

Banking pratiği: Redis lock **advisory** amaçlı, gerçek tutarlılık DB-level'da.

### 5. ZooKeeper — ephemeral nodes

ZooKeeper consensus protocol (ZAB). Strong consistency.

```java
CuratorFramework client = CuratorFrameworkFactory.newClient(...);
client.start();

InterProcessMutex lock = new InterProcessMutex(client, "/banking/locks/eod-job");

if (lock.acquire(10, TimeUnit.SECONDS)) {
    try {
        eodReconciliationJob.run();
    } finally {
        lock.release();
    }
}
```

**Ephemeral node:** Client connection koparsa node otomatik silinir → lock otomatik release.

**Avantajlar:**
- Strong consistency
- Connection-based release (TTL guesswork yok)
- Multi-data-center support

**Dezavantajlar:**
- ZooKeeper cluster operate etmek (operational overhead)
- Latency Redis'e göre yüksek

**Banking pratiği:** ZooKeeper banking'de **yaygın değil** (eski Kafka için kullanılırdı). Modern: Kafka KRaft + Redis veya DB lock.

### 6. ShedLock — Phase 5 revisit

`@SchedulerLock` ile scheduled job mutex. JDBC-backed:

```java
@Bean
public LockProvider lockProvider(DataSource dataSource) {
    return new JdbcTemplateLockProvider(
        JdbcTemplateLockProvider.Configuration.builder()
            .withJdbcTemplate(new JdbcTemplate(dataSource))
            .usingDbTime()
            .build()
    );
}

@Scheduled(cron = "0 55 23 * * *")
@SchedulerLock(name = "eodReconciliation", lockAtMostFor = "2h", lockAtLeastFor = "5m")
public void runEodReconciliation() { ... }
```

DB tablosu (shedlock):

```sql
CREATE TABLE shedlock (
    name VARCHAR(64) PRIMARY KEY,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL
);
```

Banking pratiği: Scheduled job tek instance garantisi → ShedLock. Phase 5'te detay.

### 7. pg_advisory_lock — PostgreSQL native

PostgreSQL session-level distributed lock:

```sql
-- Session 1
SELECT pg_advisory_lock(12345);   -- lock number
-- ... work ...
SELECT pg_advisory_unlock(12345);

-- Session 2
SELECT pg_advisory_lock(12345);   -- blocks until session 1 unlocks
```

```java
@Service
public class PostgresLockService {
    
    private final JdbcTemplate jdbc;
    
    public boolean tryLock(long lockId) {
        return Boolean.TRUE.equals(
            jdbc.queryForObject("SELECT pg_try_advisory_lock(?)", Boolean.class, lockId)
        );
    }
    
    public void unlock(long lockId) {
        jdbc.queryForObject("SELECT pg_advisory_unlock(?)", Boolean.class, lockId);
    }
    
    public <T> T executeWithLock(long lockId, Supplier<T> action) {
        if (!tryLock(lockId)) {
            throw new IllegalStateException("Could not acquire lock: " + lockId);
        }
        try {
            return action.get();
        } finally {
            unlock(lockId);
        }
    }
}
```

**Avantajlar:**
- No external service (Redis, ZK)
- Session ile bağlı — connection drop = auto release
- ACID guarantees (PostgreSQL)

**Dezavantajlar:**
- Single DB instance dependency
- Long lock = connection pool consume

Banking pratiği: Banking app zaten PostgreSQL kullanıyor → `pg_advisory_lock` extra dependency yok, simple.

### 8. Distributed transaction problemi

Cross-service ACID transaction:

```
account-service.debit(A) ✓
fraud-service.score(req) ✓
account-service.credit(B) ✗ FAIL
```

`@Transactional` tek DB'de geçerli. Cross-DB nasıl?

**Yöntemler:**
1. **2PC** — XA, distributed transaction. Banking için yasak (Topic 6.7).
2. **Saga** — sequence of local tx + compensation (Topic 6.7).
3. **TCC** — Try-Confirm-Cancel.
4. **Idempotent operations + outbox** — Phase 6 pattern.

### 9. TCC (Try-Confirm-Cancel) — alternative pattern

3 phase:
1. **Try:** Reserve resource (hold)
2. **Confirm:** Reserved → finalize
3. **Cancel:** Reserved → release

Banking örnek — kart authorization:

```
Authorization:
  Try: Card hold X TL (reserved balance increases, available decreases)
  ...customer signs...
Capture (success path):
  Confirm: Held → actual debit (balance reduce, reserved release)
Void (cancel path):
  Cancel: Held release (no debit, balance restored)
```

#### TCC implementation

```java
// Account service
@Service
public class AccountService {
    
    @Transactional
    public String tryHoldBalance(UUID accountId, Money amount, UUID sagaId) {
        Account account = accountRepo.findByIdAndLock(accountId).orElseThrow();
        
        if (account.getAvailableBalance().isLessThan(amount)) {
            throw new InsufficientFundsException();
        }
        
        Hold hold = new Hold(
            UUID.randomUUID(),
            accountId,
            amount,
            sagaId,
            HoldStatus.PENDING,
            Instant.now().plus(15, ChronoUnit.MINUTES)   // expire after 15 min
        );
        holdRepo.save(hold);
        
        account.setReservedBalance(account.getReservedBalance().add(amount));
        accountRepo.save(account);
        
        return hold.getId().toString();
    }
    
    @Transactional
    public void confirmHold(UUID holdId) {
        Hold hold = holdRepo.findById(holdId).orElseThrow();
        if (hold.getStatus() != HoldStatus.PENDING) {
            return;   // idempotent
        }
        
        Account account = accountRepo.findByIdAndLock(hold.getAccountId()).orElseThrow();
        account.setBalance(account.getBalance().subtract(hold.getAmount()));
        account.setReservedBalance(account.getReservedBalance().subtract(hold.getAmount()));
        accountRepo.save(account);
        
        hold.setStatus(HoldStatus.CONFIRMED);
        holdRepo.save(hold);
    }
    
    @Transactional
    public void cancelHold(UUID holdId) {
        Hold hold = holdRepo.findById(holdId).orElseThrow();
        if (hold.getStatus() != HoldStatus.PENDING) {
            return;   // idempotent
        }
        
        Account account = accountRepo.findByIdAndLock(hold.getAccountId()).orElseThrow();
        account.setReservedBalance(account.getReservedBalance().subtract(hold.getAmount()));
        accountRepo.save(account);
        
        hold.setStatus(HoldStatus.CANCELLED);
        holdRepo.save(hold);
    }
    
    @Scheduled(fixedDelay = 60000)
    @SchedulerLock(name = "expireHolds")
    public void expireOldHolds() {
        List<Hold> expired = holdRepo.findByStatusAndExpiresAtBefore(
            HoldStatus.PENDING, Instant.now());
        
        for (Hold hold : expired) {
            cancelHold(hold.getId());
            log.warn("Expired hold cancelled: {}", hold.getId());
        }
    }
}
```

#### Saga + TCC kombine

```java
@Service
public class CrossBankTransferSaga {
    
    public void execute(TransferRequest req) {
        UUID sagaId = UUID.randomUUID();
        
        // Phase 1: Try (reserve resources)
        String holdAId = bankA.tryHoldBalance(req.fromAccountId(), req.amount(), sagaId);
        String holdBId = bankB.tryReserveIncoming(req.toAccountId(), req.amount(), sagaId);
        
        try {
            // Phase 2: Confirm (finalize)
            bankA.confirmHold(holdAId);
            bankB.confirmIncoming(holdBId);
        } catch (Exception e) {
            // Phase 3: Cancel (compensate)
            try { bankA.cancelHold(holdAId); } catch (Exception ignored) {}
            try { bankB.cancelIncoming(holdBId); } catch (Exception ignored) {}
            throw e;
        }
    }
}
```

**Banking adoption:** Booking/reservation pattern (otel, uçak, kart auth) → TCC. Cross-bank transfer → Saga.

### 10. Idempotency cross-service

Cross-service request 2 kez gelebilir (network retry, async re-delivery). Backend dedup.

#### Idempotency-Key pattern

```java
@PostMapping("/v1/transfers")
public Mono<ResponseEntity<Transfer>> transfer(
    @RequestHeader("Idempotency-Key") UUID idempotencyKey,
    @Valid @RequestBody TransferRequest req,
    @AuthenticationPrincipal Jwt jwt
) {
    return idempotencyService.checkAndExecute(
        idempotencyKey,
        () -> transferService.execute(req, UUID.fromString(jwt.getSubject()))
    ).map(transfer -> ResponseEntity.status(201).body(transfer));
}

@Service
public class IdempotencyService {
    
    private final IdempotencyKeyRepository repo;
    
    public <T> Mono<T> checkAndExecute(UUID idempotencyKey, Supplier<Mono<T>> action) {
        return Mono.fromCallable(() -> repo.findByKey(idempotencyKey))
            .flatMap(existing -> {
                if (existing.isPresent()) {
                    // Already processed — return stored response
                    IdempotencyRecord record = existing.get();
                    if (record.getStatus() == IdempotencyStatus.COMPLETED) {
                        return Mono.just(deserializeResponse(record.getResponse()));
                    } else {
                        // In progress
                        return Mono.error(new IdempotencyInProgressException());
                    }
                }
                
                // New request — record + execute
                repo.save(new IdempotencyRecord(idempotencyKey, IdempotencyStatus.IN_PROGRESS));
                
                return action.get()
                    .doOnSuccess(result -> {
                        IdempotencyRecord record = repo.findByKey(idempotencyKey).orElseThrow();
                        record.setStatus(IdempotencyStatus.COMPLETED);
                        record.setResponse(serializeResponse(result));
                        repo.save(record);
                    })
                    .doOnError(e -> {
                        IdempotencyRecord record = repo.findByKey(idempotencyKey).orElseThrow();
                        record.setStatus(IdempotencyStatus.FAILED);
                        record.setError(e.getMessage());
                        repo.save(record);
                    });
            });
    }
}
```

**Banking standard:** POST /v1/transfers — `Idempotency-Key` zorunlu (Phase 1, Topic 1.5).

### 11. Banking örnek — full distributed transaction

Cross-bank transfer (TR Bank A → German Bank B):

```java
@Service
public class CrossBankTransferOrchestrator {
    
    public CrossBankTransfer execute(CrossBankTransferRequest req, UUID userId) {
        UUID sagaId = UUID.randomUUID();
        
        // Idempotency check (cross-service)
        if (sagaRepo.existsByIdempotencyKey(req.getIdempotencyKey())) {
            return sagaRepo.findByIdempotencyKey(req.getIdempotencyKey()).orElseThrow();
        }
        
        // Save initial state
        SagaState saga = sagaRepo.save(SagaState.init(sagaId, req));
        
        try {
            // Phase 1: Try
            String holdAId = bankA.tryHoldBalance(req.fromAccountId(), req.amount(), sagaId);
            saga.recordTry("BANK_A_HOLD", holdAId);
            
            FxRate fxRate = fxService.getCurrentRate(req.fromCurrency(), req.toCurrency());
            Money convertedAmount = fxRate.convert(req.amount());
            saga.recordTry("FX_RATE", fxRate.getId());
            
            String holdBId = bankB.tryReserveIncoming(req.toAccountId(), convertedAmount, sagaId);
            saga.recordTry("BANK_B_RESERVE", holdBId);
            
            // Phase 2: Confirm
            bankA.confirmHold(holdAId);
            bankB.confirmIncoming(holdBId);
            saga.markCompleted();
            
            return new CrossBankTransfer(sagaId, "COMPLETED", convertedAmount);
            
        } catch (Exception e) {
            log.error("Saga failed, compensating: {}", sagaId, e);
            
            // Phase 3: Cancel (compensation)
            saga.getTryRecords().forEach(record -> {
                try {
                    switch (record.getType()) {
                        case "BANK_A_HOLD": bankA.cancelHold(record.getResourceId()); break;
                        case "BANK_B_RESERVE": bankB.cancelIncoming(record.getResourceId()); break;
                    }
                } catch (Exception cancelException) {
                    log.error("Compensation failed for {}", record, cancelException);
                    alerts.notifyOps("Compensation failure: " + sagaId);
                }
            });
            
            saga.markCompensated();
            throw new CrossBankTransferFailedException(e.getMessage());
        } finally {
            sagaRepo.save(saga);
        }
    }
}
```

### 12. Anti-pattern'ler

**Anti-pattern 1: 2PC banking için**

Topic 6.7'de detaylandırıldı. Banking için **YASAK**.

**Anti-pattern 2: Distributed lock'a aşırı güven**

```java
lock.tryLock("transfer:" + transferId, Duration.ofMinutes(5));
// ... 6 dakika sonra hala iş bitti mi?
```

Lock TTL aşıldı, lock release oldu, başka client aldı. Original client devam etmiş ama lock yok → state inconsistency.

**Çözüm:** Critical operation **DB-level lock + idempotency**, distributed lock advisory.

**Anti-pattern 3: Compensation idempotent değil**

```java
public void cancelHold(UUID holdId) {
    Hold hold = holdRepo.findById(holdId).orElseThrow();
    accountService.refundBalance(hold.getAccountId(), hold.getAmount());
    // ❌ no status check → 2 kez çağrılırsa double refund
}
```

**Fix:** Status check + early return.

**Anti-pattern 4: TCC Cancel implementation yok**

Try succeed, Confirm fail, Cancel — implementation eksik → reserved sonsuza kalır.

**Çözüm:** Cancel her zaman implement edilmiş + scheduled expiry job.

**Anti-pattern 5: Idempotency key olmadan POST retry**

```java
@Retry(name = "x")
public Mono<Transfer> postTransfer(req) {
    return webClient.post().uri("/transfers").bodyValue(req).retrieve()...
}
```

Retry → 2x transfer creation. **Idempotency-Key zorunlu.**

**Anti-pattern 6: Lock granularity yanlış**

Çok geniş lock (e.g. `lock.acquire("account-service")`) → tüm service serial. Throughput katil.

Çok dar lock (her field) → race condition kaçar.

**Doğrusu:** Resource-level lock (`lock:card:{cardId}`, `lock:account:{accountId}`).

**Anti-pattern 7: Network partition split-brain**

3-node Redis cluster, network split → 2 partition. Her partition kendi lock'larını verir → dual ownership.

**Banking pragmatik:** Critical operation DB-level lock (PostgreSQL'in ACID guarantee'leri). Distributed lock advisory.

---

## Önemli olabilecek araştırma kaynakları

- Martin Kleppmann's Redlock critique blog post
- "Designing Data-Intensive Applications" (Kleppmann) — Chapter 8 (Distributed Systems)
- Redisson documentation
- Apache Curator ZooKeeper recipes
- ShedLock GitHub
- PostgreSQL advisory locks documentation
- TCC pattern (Pat Helland — Life Beyond Distributed Transactions)

---

## Mini task'ler

### Task 7.7.1 — Redis distributed lock with Lua release (45 dk)

`RedisDistributedLock` class implement et. Token-based ownership, Lua atomic release.

Test:
- 2 paralel thread aynı key için tryLock → birisi success, diğeri fail
- Lock holder unlock → diğer thread alabilir
- TTL aşımı → otomatik release
- Wrong token unlock → reject

### Task 7.7.2 — pg_advisory_lock pattern (30 dk)

PostgreSQL native lock. `PostgresLockService`. Test: cross-session lock.

### Task 7.7.3 — TCC Hold pattern banking (60 dk)

`Hold` entity, `HoldStatus` enum. `AccountService.tryHoldBalance / confirmHold / cancelHold`.

Test:
- tryHold → available balance decreases, reserved increases
- confirmHold → balance reduces, reserved decreases
- cancelHold → reserved decreases, balance restored
- Expired hold auto-cancel

### Task 7.7.4 — Cross-bank Saga + TCC (60 dk)

Phase 6 Topic 6.7'deki saga'ya TCC pattern uygula. Mock external bank services.

Happy path + compensation test.

### Task 7.7.5 — Idempotency middleware (45 dk)

`IdempotencyService` class. `IdempotencyKey` tablo. Cross-service idempotency.

Test:
- Same Idempotency-Key 2 kez → same response (no duplicate)
- Concurrent same key → second blocks until first completes

### Task 7.7.6 — Lock granularity test (30 dk)

`lock:card:{cardId}` vs `lock:card-service`. 1000 paralel card transaction:
- Wide lock: throughput tek-thread'lik
- Narrow lock: paralel throughput

---

## Test yazma rehberi

```java
@SpringBootTest
@Testcontainers
class DistributedLockTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Autowired RedisDistributedLock lock;
    
    @Test
    void onlyOneClientShouldAcquireLock() throws Exception {
        String key = "test-lock";
        
        Optional<String> token1 = lock.tryLock(key, Duration.ofSeconds(10));
        Optional<String> token2 = lock.tryLock(key, Duration.ofSeconds(10));
        
        assertThat(token1).isPresent();
        assertThat(token2).isEmpty();
        
        lock.unlock(key, token1.get());
        
        Optional<String> token3 = lock.tryLock(key, Duration.ofSeconds(10));
        assertThat(token3).isPresent();
    }
    
    @Test
    void wrongTokenUnlockShouldFail() {
        String key = "test-lock";
        String token1 = lock.tryLock(key, Duration.ofSeconds(10)).orElseThrow();
        
        boolean released = lock.unlock(key, "wrong-token");
        assertThat(released).isFalse();
        
        // Original token still owns lock
        Optional<String> token2 = lock.tryLock(key, Duration.ofSeconds(10));
        assertThat(token2).isEmpty();
    }
    
    @Test
    void ttlExpiryShouldReleaseLock() throws Exception {
        String key = "test-lock";
        lock.tryLock(key, Duration.ofSeconds(1));
        
        Thread.sleep(1500);
        
        Optional<String> token2 = lock.tryLock(key, Duration.ofSeconds(10));
        assertThat(token2).isPresent();
    }
    
    @Test
    void concurrentLockAcquisitionShouldSerialize() throws Exception {
        String key = "test-lock";
        int threadCount = 100;
        AtomicInteger successCount = new AtomicInteger();
        
        ExecutorService exec = Executors.newFixedThreadPool(threadCount);
        CountDownLatch done = new CountDownLatch(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            exec.submit(() -> {
                Optional<String> token = lock.tryLock(key, Duration.ofMillis(100));
                if (token.isPresent()) {
                    successCount.incrementAndGet();
                    try { Thread.sleep(10); } catch (InterruptedException e) {}
                    lock.unlock(key, token.get());
                }
                done.countDown();
            });
        }
        
        done.await();
        exec.shutdown();
        
        // Some succeed, others fail. Not all 100 simultaneously.
        assertThat(successCount.get()).isLessThan(threadCount);
    }
}

@SpringBootTest
class TccPatternTest {
    
    @Autowired AccountService accountService;
    @Autowired AccountRepository accountRepo;
    
    @Test
    void tryHoldShouldReserveBalance() {
        UUID accountId = createAccount("1000.00", "TRY");
        Money amount = Money.of("100.00", "TRY");
        
        String holdId = accountService.tryHoldBalance(accountId, amount, UUID.randomUUID());
        
        Account account = accountRepo.findById(accountId).orElseThrow();
        assertThat(account.getAvailableBalance()).isEqualTo(Money.of("900.00", "TRY"));
        assertThat(account.getReservedBalance()).isEqualTo(Money.of("100.00", "TRY"));
        assertThat(account.getBalance()).isEqualTo(Money.of("1000.00", "TRY"));   // unchanged
    }
    
    @Test
    void confirmHoldShouldFinalize() {
        UUID accountId = createAccount("1000.00", "TRY");
        String holdId = accountService.tryHoldBalance(accountId, Money.of("100.00", "TRY"), UUID.randomUUID());
        
        accountService.confirmHold(UUID.fromString(holdId));
        
        Account account = accountRepo.findById(accountId).orElseThrow();
        assertThat(account.getBalance()).isEqualTo(Money.of("900.00", "TRY"));
        assertThat(account.getReservedBalance()).isEqualTo(Money.zero("TRY"));
    }
    
    @Test
    void cancelHoldShouldReleaseReservation() {
        UUID accountId = createAccount("1000.00", "TRY");
        String holdId = accountService.tryHoldBalance(accountId, Money.of("100.00", "TRY"), UUID.randomUUID());
        
        accountService.cancelHold(UUID.fromString(holdId));
        
        Account account = accountRepo.findById(accountId).orElseThrow();
        assertThat(account.getBalance()).isEqualTo(Money.of("1000.00", "TRY"));
        assertThat(account.getReservedBalance()).isEqualTo(Money.zero("TRY"));
    }
    
    @Test
    void duplicateConfirmShouldBeIdempotent() {
        UUID accountId = createAccount("1000.00", "TRY");
        String holdId = accountService.tryHoldBalance(accountId, Money.of("100.00", "TRY"), UUID.randomUUID());
        
        accountService.confirmHold(UUID.fromString(holdId));
        accountService.confirmHold(UUID.fromString(holdId));   // duplicate
        
        Account account = accountRepo.findById(accountId).orElseThrow();
        // Balance reduced once, not twice
        assertThat(account.getBalance()).isEqualTo(Money.of("900.00", "TRY"));
    }
}
```

---

## Claude-verify prompt

```
Distributed locks & transactions implementation'ımı banking-grade kriterlere göre 
değerlendir:

1. Redis distributed lock:
   - SETNX + TTL atomic mı (setIfAbsent)?
   - Token-based ownership (UUID)?
   - Lua script atomic release?
   - Critical operation için DB-level lock tercih edilmiş mi?

2. ZooKeeper / pg_advisory_lock:
   - Banking için pragmatik seçim?
   - Connection-based release (auto release on crash)?

3. ShedLock:
   - Scheduled job multi-instance mutex var mı?
   - lockAtMostFor job süresinin 2-3x'i?

4. TCC pattern:
   - Try / Confirm / Cancel hepsi implement edilmiş mi?
   - Cancel implementation eksik DEĞİL?
   - Expired hold auto-cancel scheduler var mı?
   - confirmHold / cancelHold idempotent (status check)?

5. Idempotency cross-service:
   - Idempotency-Key header zorunlu POST/PUT'te?
   - IdempotencyService check-and-execute pattern?
   - Concurrent same-key handling (in-progress blocking)?

6. Saga + TCC kombine:
   - Cross-bank transfer 3-phase (try → confirm → cancel)?
   - Compensation idempotent?

7. Anti-pattern:
   - 2PC kullanılmış mı? (YASAK)
   - Distributed lock'a aşırı güven (critical op'lar için)?
   - Compensation idempotent değil?
   - TCC Cancel implementation eksik?
   - Idempotency-Key olmadan POST retry?

8. Lock granularity:
   - Resource-level (card, account) lock?
   - Wide lock (service-level) anti-pattern yok mu?

9. Network partition:
   - Split-brain durumunda davranış düşünülmüş mü?
   - Banking critical operations DB-level (ACID)?

10. Banking-specific:
    - Card authorization TCC pattern?
    - Cross-bank Saga + TCC kombine?
    - Hold expiry job?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] RedisDistributedLock + Lua atomic release implementation
- [ ] pg_advisory_lock pattern denedim
- [ ] TCC pattern AccountService.tryHold/confirm/cancel
- [ ] Cross-bank Saga + TCC kombine
- [ ] IdempotencyService middleware
- [ ] Expired hold auto-cancel scheduler
- [ ] Lock granularity test (wide vs narrow)
- [ ] 4+ integration test

---

## Defter notları (10 madde)

1. "Distributed lock 5 kullanım senaryosu (singleton, exclusive, race, cache stampede, init): ____."
2. "Redis SETNX + Lua atomic release detayı: ____."
3. "Redlock controversy + Kleppmann critique + banking pragmatik karar: ____."
4. "pg_advisory_lock vs Redis vs ZooKeeper tercih: ____."
5. "TCC Try-Confirm-Cancel banking örneği (kart authorization): ____."
6. "TCC Cancel implementation eksik anti-pattern: ____."
7. "Idempotency cross-service Idempotency-Key pattern: ____."
8. "Saga + TCC kombine cross-bank transfer: ____."
9. "Lock granularity (resource-level vs service-wide) trade-off: ____."
10. "Network partition split-brain banking critical op için DB-level guarantee: ____."
