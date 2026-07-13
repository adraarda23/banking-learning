# Phase 3 Mini-Project — Concurrency Hardening

## Hedef

Phase 3'te öğrendiğin 11 topic'i `core-banking` üstüne **uygula**: rate limiter, virtual thread benchmark, deadlock reproduction + fix, memory leak hunting, pinning detection, parallel FX fetch.

Sonunda elinde: concurrent yükler altında çalışan, profiler ile incelenmiş, deadlock-free, pool-safe bir banking servisi.

## Süre

4-6 gün.

## Önbilgi

Phase 3'ün 11 topic'i bitti, kavramları biliyorsun.

---

## Project görevleri

### Görev 1 — Per-user Rate Limiter (1 gün)

`core-banking`'in `/v1/transfers` endpoint'inde **kullanıcı bazlı rate limit** uygula.

**Spec:**
- Saniyede 10 transfer max
- Token bucket pattern
- `Semaphore` + `ScheduledExecutorService` refill
- `ConcurrentHashMap<UUID, TokenBucket>` per-user state
- Limit aşılırsa HTTP 429 + RFC 7807

```java
public class TokenBucket {
    private final Semaphore tokens;
    
    public TokenBucket(int maxTokens) {
        this.tokens = new Semaphore(maxTokens);
    }
    
    public boolean tryAcquire() {
        return tokens.tryAcquire();
    }
    
    public void refill(int count) {
        int available = tokens.availablePermits();
        int toAdd = Math.min(count, MAX_TOKENS - available);
        tokens.release(toAdd);
    }
}

@Component
public class RateLimiter {
    private final ConcurrentHashMap<UUID, TokenBucket> buckets = new ConcurrentHashMap<>();
    private final ScheduledExecutorService refiller = Executors.newSingleThreadScheduledExecutor();
    
    public RateLimiter() {
        refiller.scheduleAtFixedRate(this::refillAll, 1, 1, TimeUnit.SECONDS);
    }
    
    public boolean allow(UUID userId) {
        return buckets.computeIfAbsent(userId, k -> new TokenBucket(10)).tryAcquire();
    }
    
    private void refillAll() {
        buckets.values().forEach(b -> b.refill(10));
    }
}
```

Controller'a interceptor ekle:
```java
if (!rateLimiter.allow(currentUserId)) {
    throw new RateLimitExceededException();
}
```

Gatling ile test: 100 req/sec, kullanıcı 1 → response code dağılımı (10 başarılı, 90 → 429 bekleniyor sn başına).

### Görev 2 — Virtual thread enable + benchmark (1 gün)

`application.yml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

JMH benchmark `TransferEndpointBenchmark`:
- 100/1000/10000 concurrency
- Platform thread vs virtual thread
- Average latency + throughput

**Defterine tablo yaz:**

| Concurrency | Platform p99 (ms) | Virtual p99 (ms) | Throughput Platform | Throughput Virtual |
|---|---|---|---|---|
| 100 | ? | ? | ? | ? |
| 1000 | ? | ? | ? | ? |
| 10000 | ? | ? | ? | ? |

### Görev 3 — Deadlock reproduction + lock ordering fix (1 gün)

**Reproduction:**

```java
@Test
void shouldReproduceDeadlock() throws InterruptedException {
    UUID accountA = createAccount("TRY", "1000.00");
    UUID accountB = createAccount("TRY", "1000.00");
    
    int iterations = 100;
    CountDownLatch latch = new CountDownLatch(iterations * 2);
    
    ExecutorService exec = Executors.newFixedThreadPool(20);
    for (int i = 0; i < iterations; i++) {
        exec.submit(() -> { try { transferService.transferWithDeadlockBug(accountA, accountB, Money.of("1", "TRY")); } catch (Exception e) {} latch.countDown(); });
        exec.submit(() -> { try { transferService.transferWithDeadlockBug(accountB, accountA, Money.of("1", "TRY")); } catch (Exception e) {} latch.countDown(); });
    }
    
    boolean finished = latch.await(30, TimeUnit.SECONDS);
    if (!finished) {
        // jstack ile deadlock yakala
        ManagementFactory.getThreadMXBean().findDeadlockedThreads();
    }
}
```

`transferWithDeadlockBug` yazılışı: iki account'u sırayla lock'la, fakat hangisini önce lock'ladığını parametre sırasından alır → A→B + B→A deadlock yaratır.

**Fix:**

```java
public void transferWithFix(UUID from, UUID to, Money amount) {
    // Lock ordering — küçük ID önce
    UUID first = from.compareTo(to) < 0 ? from : to;
    UUID second = from.compareTo(to) < 0 ? to : from;
    
    Account firstAcc = accountRepo.lockById(first);
    Account secondAcc = accountRepo.lockById(second);
    
    // ... transfer
}
```

`jstack` ile deadlock'u CANLI yakala (test 30 saniyede bitmiyorsa thread dump al).

### Görev 4 — Memory leak hunting (1 gün)

`core-banking`'e bilinçli leaky cache ekle:

```java
@Component
public class UnboundedTransactionCache {
    private final Map<UUID, TransactionDetails> store = new HashMap<>();   // bounded değil
    
    public void put(UUID id, TransactionDetails details) {
        store.put(id, details);
    }
}
```

Bir endpoint ekle: her çağrıda 1000 entry cache'le.

JVM flag:

```bash
-Xmx256m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=./oom.hprof
```

Yük: 100 request → OOM tetiklenir.

**Görevin:**
1. `oom.hprof` MAT ile analiz et
2. Leak Suspects Report — "UnboundedTransactionCache" işaret edilmeli
3. Dominator Tree — retained heap görüntüsü al
4. Path to GC Roots ile leak source bul
5. **Düzelt:** Caffeine ile bounded cache (max 10000 entry, TTL 1 dakika)

**Defterine** screenshot + yorum yaz.

### Görev 5 — Pinning detection + fix (1 gün)

Kasten pinning yarat:

```java
public class PinningTransferService {
    private final Object monitor = new Object();
    
    public void transfer(UUID from, UUID to, BigDecimal amount) {
        synchronized (monitor) {
            // JDBC call — synchronized içinde blocking
            Account fromAcc = jdbcTemplate.queryForObject(...);
            Account toAcc = jdbcTemplate.queryForObject(...);
            // ...
            Thread.sleep(50);   // simulating I/O
        }
    }
}
```

Spring config:

```bash
-Djdk.tracePinnedThreads=full
-XX:+UnlockDiagnosticVMOptions
-XX:StartFlightRecording=duration=60s,filename=pinning.jfr
```

100 paralel virtual thread'le çağır. Log'da pinning warning gör.

```bash
jfr print --events jdk.VirtualThreadPinned pinning.jfr
```

**Fix:** `synchronized` → `ReentrantLock`. Benchmark öncesi/sonrası throughput karşılaştır.

### Görev 6 — Parallel FX fetcher (StructuredTaskScope) (45 dk)

`core-banking`'e FX rate aggregator ekle. 3 mock provider:
- `TcmbProvider` — 1 sn gecikme
- `EcbProvider` — 200 ms gecikme
- `FixerProvider` — 500 ms gecikme

```java
@Service
public class FxRateAggregator {
    
    public FxRate getRate(Currency from, Currency to) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<FxRate>()) {
            scope.fork(() -> tcmbProvider.getRate(from, to));
            scope.fork(() -> ecbProvider.getRate(from, to));
            scope.fork(() -> fixerProvider.getRate(from, to));
            
            scope.join();
            return scope.result();   // ilk başarılı (yaklaşık 200 ms)
        }
    }
}
```

`--enable-preview` flag aktif.

**Test:** 10 paralel çağrı, her birinin ~200 ms civarı sonuçlandığını ölç (en hızlı provider yarışı kazanır).

---

## Acceptance criteria

- [ ] Per-user rate limiter çalışıyor, 429 HTTP response veriyor limit aşıldığında
- [ ] Spring Boot virtual thread enabled, log'da `isVirtual=true`
- [ ] JMH platform vs virtual benchmark sonuçları **defterimde tablo**
- [ ] Deadlock canlı reproduce edildi (test 30s timeout)
- [ ] jstack ile "Found Java-level deadlock" satırı görüldü
- [ ] Lock ordering fix uygulandı, deadlock kayboldu
- [ ] Bilinçli memory leak yaratıldı, MAT ile analiz edildi, dominator tree screenshot
- [ ] Caffeine ile bounded cache fix
- [ ] Pinning JFR event'i ile yakalandı (`jdk.VirtualThreadPinned`)
- [ ] `synchronized` → `ReentrantLock` fix uygulandı
- [ ] FX aggregator StructuredTaskScope.ShutdownOnSuccess ile
- [ ] FX latency test: 10 çağrı, hepsi ~200ms

---

## Kasten kırma görevleri

Phase 3'ün ruhu: **bir şeyi kasten bozup düzeltmek**. Her görevde:

1. Deadlock üret → jstack yakala → fix
2. Memory leak yarat → MAT ile bul → bounded cache fix
3. Pinning oluştur → JFR ile gör → ReentrantLock fix

Her birinde **stack trace okumayı** öğreniyorsun.

---

## Claude-verify prompt

```
Phase 3 mini-project'imi banking-grade kriterlere göre değerlendir:

1. Rate limiter:
   - Token bucket pattern doğru implement edilmiş mi?
   - Per-user ConcurrentHashMap kullanımı thread-safe mi?
   - Refill scheduled task çalışıyor mu?
   - Rate limit aşımında 429 + ProblemDetail RFC 7807 mu?

2. Virtual thread:
   - `spring.threads.virtual.enabled: true` mu?
   - JMH benchmark sonuçları gerçekçi mi (10k concurrency'de virtual avantajı?)?
   - Pinning yok mu (synchronized + JDBC kombinasyonu)?

3. Deadlock:
   - Reproduction canlı oldu mu (test 30s timeout)?
   - jstack'te "Found Java-level deadlock" line görüldü mü?
   - Lock ordering ile fix uygulandı mı (sorted account IDs)?

4. Memory leak:
   - OOM heap dump aldın mı?
   - MAT analizi yapıldı mı (Leak Suspects, Dominator Tree)?
   - Path to GC Roots ile root cause bulundu mu?
   - Caffeine bounded cache ile fix uygulandı mı?

5. Pinning:
   - `jdk.tracePinnedThreads=full` ile uyarılar görüldü mü?
   - JFR `jdk.VirtualThreadPinned` event'i yakalandı mı?
   - `synchronized` → `ReentrantLock` fix uygulandı mı?
   - Fix öncesi/sonrası throughput karşılaştırması yapıldı mı?

6. StructuredTaskScope:
   - ShutdownOnSuccess doğru pattern mi?
   - `--enable-preview` flag aktif mi?
   - 3 provider paralel sorgu doğru kurulmuş mı?
   - Latency en hızlı provider'a göre mi (200ms civarı)?

7. Genel:
   - Tüm test'ler `mvn test` ile geçiyor mu?
   - JaCoCo coverage Phase 2'deki seviyenin altında değil mi (75%+)?
   - Git commit'leri anlamlı mı?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Defter notları

1. "Token bucket rate limiter'ın `Semaphore` ile implementation kararı: ____."
2. "Virtual thread enable etmenin core-banking üzerindeki ölçülen etkisi: ____."
3. "Deadlock üretip jstack'le yakalama deneyimi - öğrendiklerim: ____."
4. "MAT ile memory leak hunting - dominator tree + path to GC roots: ____."
5. "Pinning JFR event'ini canlı yakalama - benim için en şaşırtıcı kısmı: ____."
6. "StructuredTaskScope.ShutdownOnSuccess pattern'inin banking kullanımı: ____."
7. "Phase 3'te en zorlandığım kavram: ____."
8. "Phase 3 sonrasında bir senior dev'e en rahat anlatabileceğim konu: ____."
