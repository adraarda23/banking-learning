# Topic 3.6 — Concurrent Collections

## Hedef

Java'nın **thread-safe collection** kütüphanesini derinlemesine öğrenmek. `ConcurrentHashMap`'in iç mekanizmasını (segment-free, CAS-based), `BlockingQueue` ailesinin kullanım pattern'lerini, `CountDownLatch`/`CyclicBarrier`/`Semaphore`/`Phaser`'ın hangi senaryoda ne çözdüğünü banking örnekleriyle kavramak. `synchronizedMap` ve `Collections.synchronizedList` gibi legacy yöntemlerden neden uzaklaştığımızı anlamak.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 3.1-3.5 bitti — JMM, atomic, lock, executor, CompletableFuture biliyorsun
- `HashMap`, `ArrayList` gibi single-threaded collection'lar tanıdık
- "Thread-safe" kavramının ne demek olduğunu sezgisel anlıyorsun

---

## Kavramlar

### 1. Single-threaded collection'lar neden thread-safe değil

`HashMap`, `ArrayList`, `LinkedList` — concurrent yazmaya **dayanıksızdır**. Eşzamanlı modifikasyon sonucu:

- **Lost update:** İki thread put yapar, biri kaybolur
- **Infinite loop (Java 7 ve öncesi `HashMap` resize):** Resize sırasında linked list'in döngülenmesi, get() forever spins
- **`ConcurrentModificationException`:** Iteration sırasında başka thread modify ederse fail-fast

Banking örneği — yanlış kod:

```java
private final Map<UUID, BigDecimal> accountBalances = new HashMap<>();

public void deposit(UUID accountId, BigDecimal amount) {
    BigDecimal current = accountBalances.getOrDefault(accountId, BigDecimal.ZERO);
    accountBalances.put(accountId, current.add(amount));
}
```

İki paralel `deposit(id, 100)`:
1. T1 reads current=0
2. T2 reads current=0
3. T1 puts 100
4. T2 puts 100

Sonuç: 100 (200 olması gerekirken). **Atomicity yok.** 100 TL bankaya hediye edildi.

### 2. Eski çözüm: `Collections.synchronizedMap`

```java
Map<UUID, BigDecimal> sync = Collections.synchronizedMap(new HashMap<>());
```

Her metot `synchronized` ile sarılır. **Çalışır ama kötü:**

- **Tüm map için tek lock** → 100 paralel okuma 100 kez sıraya girer (performans katil)
- **Compound action'lar atomik değil:** `get + put` race condition hâlâ var

```java
// HÂLÂ YANLIŞ
synchronized (sync) {   // explicit external lock gerekli
    BigDecimal current = sync.getOrDefault(id, BigDecimal.ZERO);
    sync.put(id, current.add(amount));
}
```

External synchronization sorumluluğu kullanıcıya — kolayca unutulur.

### 3. `ConcurrentHashMap` — banking standardı

Java 5'te tanıtıldı, Java 8'de yeniden yazıldı (segment-free).

**Iç çalışma (Java 8+):**

- Internal array of bins (default 16, grows)
- Her bin **independent CAS-based lock** kullanır (sadece o bin sırasında lock)
- Read operasyonları **lock-free** (volatile read)
- Write operasyonları **bin-level lock** (synchronized on first node of bin)
- Tree-bin'e dönüşüm (collision threshold 8) — TreeMap gibi davranır

Sonuç:
- 16 ayrı bin'e yazan thread'ler **birbirini engellemez**
- Read'ler **lock'suz**
- Throughput: synchronizedMap'ten 5-10x

**Banking örneği — doğru kod:**

```java
private final ConcurrentHashMap<UUID, BigDecimal> accountBalances = new ConcurrentHashMap<>();

public void deposit(UUID accountId, BigDecimal amount) {
    accountBalances.merge(accountId, amount, BigDecimal::add);
}
```

`merge` **atomic** — get + compute + put tek operasyon. Race condition yok.

### 4. `ConcurrentHashMap` atomic operasyonları

#### `putIfAbsent(K, V)`

```java
BigDecimal previous = accountBalances.putIfAbsent(accountId, BigDecimal.ZERO);
// previous == null → biz koyduk
// previous != null → zaten varmış, biz dokunmadık
```

**Tek atomic adım.** Eski:

```java
// ❌ Race condition var
if (!map.containsKey(id)) map.put(id, value);
```

#### `compute(K, BiFunction<K, V, V>)`

```java
accountBalances.compute(accountId, (k, v) -> 
    (v == null ? BigDecimal.ZERO : v).add(amount)
);
```

Get + compute + put **atomik**. v null gelirse yok demek.

#### `computeIfAbsent(K, Function<K, V>)`

```java
List<Transaction> txList = transactionsByAccount
    .computeIfAbsent(accountId, k -> new ArrayList<>());
txList.add(transaction);
```

Yok ise compute et + koy + dön. **Atomik.** Banking pattern: per-account collection'lar.

**Tuzak:** Compute lambda'sı **kısa olmalı**. İçinde uzun iş yapma (DB call, sleep) — bin lock tutulur, başka thread'ler bekler.

#### `computeIfPresent(K, BiFunction)`

```java
accountBalances.computeIfPresent(accountId, (k, v) -> v.add(amount));
// Yoksa hiçbir şey yapma
```

#### `merge(K, V, BiFunction)`

```java
accountBalances.merge(accountId, amount, BigDecimal::add);
// Yoksa amount koy, varsa add ile birleştir
```

**Banking örneği — günlük transaction sayacı:**

```java
private final ConcurrentHashMap<LocalDate, AtomicLong> dailyTxCount = new ConcurrentHashMap<>();

public void recordTransaction(Instant occurredAt) {
    LocalDate date = LocalDate.ofInstant(occurredAt, ZoneId.systemDefault());
    dailyTxCount.computeIfAbsent(date, k -> new AtomicLong()).incrementAndGet();
}
```

`computeIfAbsent` ile `AtomicLong` yarat (atomik), sonra `incrementAndGet` (atomik). İki ayrı atomik adım, **aralarında race var mı?** Hayır — `incrementAndGet` zaten thread-safe, hangi thread çağırırsa çağırsın doğru artırır.

### 5. `ConcurrentHashMap.computeIfAbsent` atomicity vs `putIfAbsent`

İki yakın görünür ama farklı:

```java
// A — putIfAbsent
map.putIfAbsent(key, new ExpensiveObject());   // ← ExpensiveObject HER ZAMAN yaratılır
```

`new ExpensiveObject()` argüman olarak değerlendirilir. Atomicity sadece put için. Eğer key zaten varsa, yarattığın obje **çöpe gider**.

```java
// B — computeIfAbsent
map.computeIfAbsent(key, k -> new ExpensiveObject());   // ← yalnızca yoksa yaratılır
```

Lambda **sadece gerektiğinde** çağrılır. Lazy.

**Banking:** Yüksek volume bir map'te (her transaction için per-customer state) `computeIfAbsent` ile **gereksiz yaratım önlenir**.

### 6. `ConcurrentSkipListMap` — sıralı concurrent

`TreeMap`'in thread-safe versiyonu. Internal: lock-free skip list.

**Kullanım:** Sıralı erişim gerekir + concurrent modification gerekir.

```java
// Order book pattern
ConcurrentSkipListMap<BigDecimal, Order> bidOrders = new ConcurrentSkipListMap<>();
bidOrders.put(price, order);
Map.Entry<BigDecimal, Order> highestBid = bidOrders.lastEntry();   // O(log n)
```

Banking: order matching engine, time-series cache. Çoğu banking app'inde **gerek yok** — `ConcurrentHashMap` yeter.

### 7. `CopyOnWriteArrayList` — read-heavy / write-rare

Yazma yapıldığında **tüm internal array kopyalanır**. Okuma lock'suz.

**Kullanım:** Read >>> write (örn. 1000:1).

```java
// Banking: subscriber list (rarely changes, queried per event)
CopyOnWriteArrayList<EventSubscriber> subscribers = new CopyOnWriteArrayList<>();

public void onEvent(Event e) {
    for (EventSubscriber sub : subscribers) {   // lock-free iteration
        sub.notify(e);
    }
}
```

**Anti-pattern:** Sık yazılan collection'da kullanma. Her yazımda full copy = O(n) yazma maliyeti.

### 8. `BlockingQueue` ailesi — producer/consumer

**Banking örneği:** Transfer request'leri queue'ya at, worker thread'ler işlesin.

```java
BlockingQueue<TransferRequest> queue = new LinkedBlockingQueue<>(1000);

// Producer
queue.put(request);   // blocks if queue full

// Consumer
TransferRequest req = queue.take();   // blocks if empty
```

#### Aile üyeleri

**`ArrayBlockingQueue`:** Sabit kapasite, array-backed, FIFO, fair option.

```java
new ArrayBlockingQueue<>(1000);   // 1000 capacity
new ArrayBlockingQueue<>(1000, true);   // fair (FIFO strict)
```

Kullanım: Bounded queue ihtiyacı, kapasite tahmin edilebilir.

**`LinkedBlockingQueue`:** Linked node-based, optionally bounded.

```java
new LinkedBlockingQueue<>();          // UNBOUNDED — OOM riski
new LinkedBlockingQueue<>(10000);     // bounded
```

**Tuzak:** Unbounded default — eğer consumer yavaşsa queue sınırsız büyür, OOM. **Banking'de her zaman bounded.**

**`SynchronousQueue`:** 0 kapasite. Her `put` bir `take` bekler (direct handoff).

```java
new SynchronousQueue<>();
```

Kullanım: Producer-consumer arasında back-pressure (consumer hazır değilse producer bekler).

**`PriorityBlockingQueue`:** Unbounded, priority-ordered (`Comparable` veya `Comparator`).

```java
PriorityBlockingQueue<FraudCheck> queue = new PriorityBlockingQueue<>(
    100, Comparator.comparingInt(FraudCheck::priority).reversed()
);
```

Banking: Fraud check queue — yüksek risk skorlu transaction'lar önce.

**`DelayQueue`:** Element belirli süre sonra alınabilir hale gelir.

```java
public class DelayedRetry implements Delayed {
    private final long executeAt;
    
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }
    
    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.executeAt, ((DelayedRetry) o).executeAt);
    }
}

DelayQueue<DelayedRetry> retryQueue = new DelayQueue<>();
retryQueue.put(new DelayedRetry(System.currentTimeMillis() + 5000));   // 5 sn sonra al
DelayedRetry next = retryQueue.take();   // blocks until ready
```

Banking: Failed transaction retry (5 sn → 30 sn → 5 dk exponential backoff).

**`TransferQueue` (`LinkedTransferQueue`):** `BlockingQueue` + `transfer(element)` method'u (consumer'ı **bekler**).

```java
LinkedTransferQueue<Notification> queue = new LinkedTransferQueue<>();
queue.transfer(notification);   // bir consumer alana kadar block
```

#### Banking pattern — bounded queue + thread pool

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5, 10,
    60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000),   // bounded — backpressure
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()   // queue dolu → caller işler
);
```

`CallerRunsPolicy`: Queue dolu olduğunda task'ı submit eden thread çalıştırır. **Doğal backpressure** — producer yavaşlar, sistem stabil kalır.

### 9. `CountDownLatch` — one-shot çakıl

Bir thread'in **n event** beklemesi gerekiyorsa.

```java
CountDownLatch latch = new CountDownLatch(3);

// 3 farklı task
new Thread(() -> { doWork1(); latch.countDown(); }).start();
new Thread(() -> { doWork2(); latch.countDown(); }).start();
new Thread(() -> { doWork3(); latch.countDown(); }).start();

latch.await();   // blocks until count = 0
System.out.println("All done");
```

**Banking:** EOD reconciliation — 3 farklı sistem rapor üretir, hepsi bittiğinde mutabakat başlar.

**Sınır:** Count **sıfıra düşünce** yeniden kullanılamaz. One-shot.

### 10. `CyclicBarrier` — yeniden kullanılabilir senkronizasyon

n thread aynı noktaya geldiğinde devam etsin.

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Tüm faz bitti"));

// 3 worker
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        for (int phase = 0; phase < 5; phase++) {
            doPhase(phase);
            barrier.await();   // diğer 2'yi bekle
        }
    }).start();
}
```

Her `await()`'da 3 thread toplanır, **barrier action** çalışır (varsa), sonra devam. **Yeniden kullanılabilir.**

**Banking:** Multi-step batch — 3 worker her adımda senkronize ilerler.

### 11. `Semaphore` — kaynak sayısı sınırı

n izin var, alana ver, alamayan beklesin.

```java
Semaphore connectionLimit = new Semaphore(5);   // 5 izin

public void callExternalApi() throws InterruptedException {
    connectionLimit.acquire();   // izin al (blocks if 0)
    try {
        // external API call
    } finally {
        connectionLimit.release();   // izni iade et
    }
}
```

**Banking örneği — rate limiter:**

```java
public class TokenBucketRateLimiter {
    private final Semaphore tokens;
    private final ScheduledExecutorService refiller;
    
    public TokenBucketRateLimiter(int maxTokens, int refillPerSecond) {
        this.tokens = new Semaphore(maxTokens);
        this.refiller = Executors.newSingleThreadScheduledExecutor();
        refiller.scheduleAtFixedRate(() -> {
            int available = tokens.availablePermits();
            int toAdd = Math.min(refillPerSecond, maxTokens - available);
            tokens.release(toAdd);
        }, 1, 1, TimeUnit.SECONDS);
    }
    
    public boolean tryAcquire() {
        return tokens.tryAcquire();   // hemen dön — yoksa false
    }
}
```

Banking: External API rate limit (TCMB'nin 100 req/sn limiti gibi).

**Fairness:** `new Semaphore(5, true)` FIFO — kim önce bekledi, önce alır. Default false (throughput odaklı).

### 12. `Phaser` — dinamik CountDownLatch

`CountDownLatch` ve `CyclicBarrier`'ın esnek hibridi. Phase'ler arasında thread sayısı **değişebilir**.

```java
Phaser phaser = new Phaser(1);   // 1 main thread registered

for (int i = 0; i < 5; i++) {
    phaser.register();   // her worker register
    new Thread(() -> {
        doWork();
        phaser.arriveAndDeregister();   // bitti, çıktım
    }).start();
}

phaser.arriveAndAwaitAdvance();   // diğerlerinin bitmesini bekle
```

Banking: Dinamik worker pool, runtime'da hesabı değişen senaryolar.

Genelde Phaser **karmaşık** — `CountDownLatch` veya `CyclicBarrier` yeterse onları tercih et.

### 13. `Exchanger` — iki thread arası nesne takası

İki thread aynı `Exchanger`'da `exchange(x)` çağırır → birinin verdiği X, diğerine; diğerinin verdiği Y, birinciye.

```java
Exchanger<Buffer> exchanger = new Exchanger<>();

// Filler thread
Buffer filled = exchanger.exchange(filledBuffer);   // boş bir tane al

// Drainer thread
Buffer empty = exchanger.exchange(emptyBuffer);   // dolu bir tane al
```

Banking: Nadiren gerekir. Double-buffering pattern.

### 14. Banking anti-pattern'leri

**Anti-pattern 1: `HashMap` concurrent kullanımı**

```java
// ❌ Tehlike
private final Map<UUID, Account> cache = new HashMap<>();
```

Hangi thread çağırırsa çağırsın → corruption, infinite loop, NPE.

**Çözüm:** `ConcurrentHashMap` veya local-thread cache (`ThreadLocal`).

**Anti-pattern 2: Iteration sırasında modification**

```java
for (Map.Entry<UUID, Account> entry : accounts.entrySet()) {
    if (someCondition(entry)) {
        accounts.remove(entry.getKey());   // ConcurrentModificationException
    }
}
```

**Çözüm:** `accounts.entrySet().removeIf(...)` veya `ConcurrentHashMap` ile concurrent iteration (weakly consistent).

**Anti-pattern 3: Unbounded queue**

```java
new LinkedBlockingQueue<>()   // OOM ediyor production'da
```

**Çözüm:** Her zaman bounded, capacity belirle.

**Anti-pattern 4: `CountDownLatch` reuse**

```java
CountDownLatch latch = new CountDownLatch(3);
// ... await
// İkinci faz için aynı latch kullanmak istiyorsun
latch.await();   // ❌ HÂLÂ 0, anında dönecek
```

**Çözüm:** Yeni `CountDownLatch` veya `CyclicBarrier` kullan.

**Anti-pattern 5: `Collections.synchronizedXxx` modern banking'de**

```java
Map<UUID, Account> map = Collections.synchronizedMap(new HashMap<>());
```

Çalışır ama tüm map için tek lock → contention. **`ConcurrentHashMap` tercih et.**

**Anti-pattern 6: Heavy operation in `computeIfAbsent` lambda**

```java
accounts.computeIfAbsent(id, k -> repository.findById(k).orElseThrow());   // DB call!
```

Bin lock tutuluyor, DB call yavaş, başka thread'ler bekler. **Çözüm:** Lambda hızlı olsun, DB call dışarıda.

---

## Önemli olabilecek araştırma kaynakları

- "Java Concurrency in Practice" (Brian Goetz) — Chapter 5 (Building Blocks)
- ConcurrentHashMap source code (OpenJDK, doğru iç anlama için)
- "ConcurrentHashMap analysis" blog posts (multi-author)
- BlockingQueue Javadoc
- Vlad Mihalcea blog — concurrency-related posts
- LMAX Disruptor (alternatif yüksek-throughput queue)
- Aeron messaging library (Real Logic) — low-latency

---

## Mini task'ler

### Task 3.6.1 — `ConcurrentHashMap.merge` ile balance tracker (30 dk)

`InMemoryBalanceCache` yaz:

```java
public class InMemoryBalanceCache {
    private final ConcurrentHashMap<UUID, BigDecimal> balances = new ConcurrentHashMap<>();
    
    public void credit(UUID accountId, BigDecimal amount) {
        balances.merge(accountId, amount, BigDecimal::add);
    }
    
    public void debit(UUID accountId, BigDecimal amount) {
        // önce mevcut bakiye kontrol et, sonra eksilt — atomik olmalı
    }
}
```

Debit metodunu `compute` ile yaz. Yetersiz bakiyede `InsufficientFundsException`.

100 thread'le paralel credit/debit testi, son bakiyenin doğru olduğunu doğrula.

### Task 3.6.2 — Bounded transfer queue + worker (45 dk)

Producer/consumer pattern:

```java
public class TransferQueue {
    private final BlockingQueue<TransferRequest> queue = new ArrayBlockingQueue<>(1000);
    private final ExecutorService workers;
    
    public void submit(TransferRequest req) throws InterruptedException {
        queue.put(req);   // blocks if full
    }
    
    public void start(int workerCount) {
        // n worker thread başlat, queue.take() ile işle
    }
}
```

`CallerRunsPolicy` benzeri: queue dolu, ana thread işlesin.

### Task 3.6.3 — Priority fraud queue (45 dk)

`PriorityBlockingQueue<FraudCheck>` ile yüksek-risk transaction'lar önce işlensin.

```java
record FraudCheck(UUID txId, int riskScore) implements Comparable<FraudCheck> {
    @Override
    public int compareTo(FraudCheck o) {
        return Integer.compare(o.riskScore, this.riskScore);   // descending
    }
}
```

10 random FraudCheck koy, take() ile sırayla al — yüksek skor önce gelmeli.

### Task 3.6.4 — Token bucket rate limiter (45 dk)

`Semaphore` + `ScheduledExecutorService` ile yukarıdaki örneği yaz.

Test: 10 token, sn'de 5 refill. 20 paralel `tryAcquire()` çağrısı yap, 10'unun başarılı, 10'unun fail olmasını gözle.

### Task 3.6.5 — `CountDownLatch` ile EOD reconciliation (30 dk)

3 sistem rapor üretir (Account, Transfer, Card). Hepsi bitince `ReconciliationEngine.start()` tetiklensin.

```java
CountDownLatch latch = new CountDownLatch(3);

ExecutorService exec = Executors.newFixedThreadPool(3);
exec.submit(() -> { Account.report(); latch.countDown(); });
exec.submit(() -> { Transfer.report(); latch.countDown(); });
exec.submit(() -> { Card.report(); latch.countDown(); });

latch.await();
ReconciliationEngine.start();
```

### Task 3.6.6 — `DelayQueue` ile retry scheduler (45 dk)

Failed transaction'lar `DelayQueue`'a 5 sn sonra retry için konulsun. Worker thread `take()` ile çekip retry yapsın.

---

## Test yazma rehberi

### Test 3.6.1 — `ConcurrentHashMap` thread safety

```java
@Test
void mergeShouldBeAtomicUnderConcurrency() throws InterruptedException {
    InMemoryBalanceCache cache = new InMemoryBalanceCache();
    UUID accountId = UUID.randomUUID();
    int threadCount = 100;
    int incrementsPerThread = 1000;
    
    ExecutorService exec = Executors.newFixedThreadPool(threadCount);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(threadCount);
    
    for (int i = 0; i < threadCount; i++) {
        exec.submit(() -> {
            try {
                start.await();
                for (int j = 0; j < incrementsPerThread; j++) {
                    cache.credit(accountId, BigDecimal.ONE);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown();
            }
        });
    }
    
    start.countDown();   // herkes aynı anda başlasın
    done.await();
    exec.shutdown();
    
    BigDecimal expected = BigDecimal.valueOf((long) threadCount * incrementsPerThread);
    assertThat(cache.getBalance(accountId)).isEqualByComparingTo(expected);
}
```

### Test 3.6.2 — BlockingQueue back-pressure

```java
@Test
void boundedQueueShouldBlockWhenFull() throws InterruptedException {
    BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(2);
    queue.put(1);
    queue.put(2);
    
    AtomicBoolean blocked = new AtomicBoolean(true);
    Thread producer = new Thread(() -> {
        try {
            queue.put(3);   // blocks
            blocked.set(false);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
    producer.start();
    
    Thread.sleep(100);
    assertThat(blocked).isTrue();   // hâlâ block
    
    queue.take();   // birini al, producer kurtulsun
    producer.join(1000);
    assertThat(blocked).isFalse();
}
```

### Test 3.6.3 — PriorityBlockingQueue order

```java
@Test
void priorityQueueShouldDeliverHighestRiskFirst() throws InterruptedException {
    PriorityBlockingQueue<FraudCheck> queue = new PriorityBlockingQueue<>();
    queue.put(new FraudCheck(UUID.randomUUID(), 3));
    queue.put(new FraudCheck(UUID.randomUUID(), 9));
    queue.put(new FraudCheck(UUID.randomUUID(), 1));
    
    assertThat(queue.take().riskScore()).isEqualTo(9);
    assertThat(queue.take().riskScore()).isEqualTo(3);
    assertThat(queue.take().riskScore()).isEqualTo(1);
}
```

### Test 3.6.4 — CountDownLatch tek-shot

```java
@Test
void latchShouldReleaseWhenCountReachesZero() throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(3);
    AtomicBoolean released = new AtomicBoolean(false);
    
    Thread waiter = new Thread(() -> {
        try {
            latch.await();
            released.set(true);
        } catch (InterruptedException e) { /* */ }
    });
    waiter.start();
    
    latch.countDown();
    Thread.sleep(50);
    assertThat(released).isFalse();
    
    latch.countDown();
    latch.countDown();
    waiter.join(1000);
    assertThat(released).isTrue();
}
```

### Test 3.6.5 — Semaphore concurrent permit

```java
@Test
void semaphoreShouldLimitConcurrentAccess() throws InterruptedException {
    Semaphore sem = new Semaphore(2);
    AtomicInteger maxConcurrent = new AtomicInteger();
    AtomicInteger currentConcurrent = new AtomicInteger();
    
    ExecutorService exec = Executors.newFixedThreadPool(10);
    CountDownLatch done = new CountDownLatch(10);
    
    for (int i = 0; i < 10; i++) {
        exec.submit(() -> {
            try {
                sem.acquire();
                int now = currentConcurrent.incrementAndGet();
                maxConcurrent.updateAndGet(prev -> Math.max(prev, now));
                Thread.sleep(50);
                currentConcurrent.decrementAndGet();
                sem.release();
            } catch (InterruptedException e) { /* */ }
            finally { done.countDown(); }
        });
    }
    
    done.await();
    exec.shutdown();
    
    assertThat(maxConcurrent.get()).isLessThanOrEqualTo(2);
}
```

---

## Claude-verify prompt

```
Aşağıdaki Java concurrent collections kodumu banking-grade kriterlere göre 
değerlendir. Sadece eksikleri ve yanlışları işaretle, kod yazma:

1. ConcurrentHashMap kullanımı:
   - `HashMap` thread-shared yerlerde kullanılmış mı? (Olmamalı)
   - `merge`, `compute`, `computeIfAbsent` ile atomic update yapılmış mı?
   - `get + put` race condition pattern'i hâlâ var mı?
   - `computeIfAbsent` lambda'sında DB call veya I/O var mı? (Olmamalı, bin lock tutuluyor)

2. BlockingQueue:
   - Queue'lar bounded mı (capacity belirli)?
   - `new LinkedBlockingQueue<>()` unbounded kullanılmış mı? (Olmamalı production'da)
   - Producer/consumer pattern doğru mu (put/take BlockingQueue API, queue/poll değil)?
   - `RejectedExecutionHandler` belirli mi (CallerRunsPolicy gibi)?

3. Priority queue:
   - `Comparable` veya `Comparator` doğru sıralıyor mu?
   - Reverse order (yüksek öncelik önce) açıkça belirtilmiş mi?

4. Synchronization aidleri:
   - `CountDownLatch` reuse edilmek istenmiş mi? (Hatalı — one-shot)
   - `CyclicBarrier`'a karşı `CountDownLatch` doğru seçilmiş mi?
   - `Semaphore` rate limiter'da tryAcquire/acquire farkı bilinçli mi?

5. Banking pattern'ler:
   - Balance tracking gibi yüksek-frekans operasyonda ConcurrentHashMap kullanılmış mı?
   - Rate limiter Semaphore ile implement edilmiş mi?
   - Per-account collection için computeIfAbsent pattern'i kullanılmış mı?

6. Anti-pattern:
   - `Collections.synchronizedMap` modern code'da var mı?
   - Iteration sırasında modification var mı?
   - Compound action (check-then-act) atomik değil mi?

7. Test:
   - 100+ thread'le concurrent stress test yapılmış mı?
   - CountDownLatch ile "herkes aynı anda başlasın" pattern'i test'lerde kullanılmış mı?
   - Final assertion expected = thread count × ops per thread mu?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `InMemoryBalanceCache` `ConcurrentHashMap.merge` ile race condition'a karşı dirençli
- [ ] Bounded `BlockingQueue` ile producer/consumer çalışıyor
- [ ] `PriorityBlockingQueue` ile fraud queue
- [ ] Token bucket rate limiter `Semaphore` ile
- [ ] `CountDownLatch` ile EOD coordination
- [ ] `DelayQueue` ile retry scheduler
- [ ] 100 thread'le stress test geçiyor (final balance doğru)
- [ ] `Collections.synchronizedMap` modern code'da kullanmıyorum
- [ ] `computeIfAbsent` lambda'sında DB call yapmıyorum
- [ ] `LinkedBlockingQueue` unbounded production'da kullanmıyorum

---

## Defter notları

1. "`ConcurrentHashMap` Java 8 öncesi vs sonrası iç farkı: ____."
2. "`merge`, `compute`, `computeIfAbsent`'ın atomic guarantee'si: ____."
3. "`putIfAbsent` ile `computeIfAbsent` farkı (lazy yaratım): ____."
4. "`Collections.synchronizedMap`'i neden production'da kullanmıyorum: ____."
5. "`BlockingQueue` ailesinde her birinin en uygun senaryosu: ____."
6. "Bounded vs unbounded queue: banking için doğru karar: ____."
7. "`CountDownLatch` ile `CyclicBarrier` farkı + reuse: ____."
8. "`Semaphore` ile rate limiter pattern (token bucket): ____."
9. "`CopyOnWriteArrayList` ne zaman doğru, ne zaman felaket: ____."
10. "`computeIfAbsent` lambda'sında DB call yapmamamın sebebi: ____."
