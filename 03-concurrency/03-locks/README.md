# Topic 3.3 — Lock Family: ReentrantLock, ReadWriteLock, StampedLock, Condition, Deadlock

## Hedef

`java.util.concurrent.locks` paketinin **explicit lock API'sini** banking-grade kavramak. `ReentrantLock` ile `synchronized` arasındaki 5+ farkı tek tek anlamak. `tryLock`, `lockInterruptibly`, fairness, `Condition` variable'ları, `ReentrantReadWriteLock` (banking config cache pattern'i), `StampedLock` (optimistic read), lock striping, ve **deadlock**'un 4 koşulunu kavramak. Banking'de gerçek bir **iki-yönlü transfer deadlock**'unu reproduce edip, `jstack` ile analiz edip, **lock ordering** veya `tryLock` ile çözmek.

## Süre

Okuma: 3 saat • Mini task'ler: 3.5 saat • Test: 1 saat • Toplam: ~7.5 saat

## Önbilgi

- Topic 3.1 (JMM) ve Topic 3.2 (synchronized, atomic, volatile) tamamlandı
- `Thread`, `Runnable`, `ExecutorService` ile basit concurrent kod yazılabiliyor
- Banking transfer kavramı (debit + credit) Phase 1'den
- `jstack` komut satırı aracının varlığı duyulmuş (Topic 3.10'da detay)

---

## Kavramlar

### 1. Neden `synchronized`'a ek olarak `Lock` API?

`synchronized` Java 1.0'dan beri var. Java 5 ile `java.util.concurrent.locks.Lock` interface'i geldi. Sebep: `synchronized` **bazı senaryolarda yetersiz**:

- `synchronized` bloku **kesin** acquire eder; bekleyen bir limit veya timeout yok.
- `synchronized` **interrupt edilemez**; thread `interrupt()` edildiyse, mevcut monitor'ı bekleyen thread bir türlü çıkamaz.
- `synchronized` block yapısı **scope sınırlıdır**; method'un başında acquire, sonunda release. Ortada bir koşulda erken release yapamazsın.
- `synchronized` **fair değildir**; bekleyen thread'lerden hangisinin gireceği belirsiz; bazı thread'ler aç kalabilir (starvation).
- `synchronized` reader-writer ayrımı **yapamaz**; her erişim mutual-exclusive.

`Lock` API bunları çözer:
- `tryLock()` ve `tryLock(timeout)` — beklemeden dene veya sınırlı bekle
- `lockInterruptibly()` — bekliyorken interrupt'a duyarlı
- `lock()`/`unlock()` kontrolü kodun her yerinde
- Fair seçeneği (true/false)
- `ReadWriteLock` ile reader-writer ayrımı

---

### 2. `ReentrantLock` — `synchronized`'ın güçlü kuzeni

```java
import java.util.concurrent.locks.ReentrantLock;

public class AccountWithLock {
    private final ReentrantLock lock = new ReentrantLock();
    private long balance;
    
    public void deposit(long amount) {
        lock.lock();
        try {
            balance += amount;
        } finally {
            lock.unlock();   // ← ASLA finally dışına çıkarma!
        }
    }
}
```

**Kritik kural:** `lock.unlock()` her zaman `finally` block'unda. İstisna olursa lock asla release edilmez → diğer thread'ler sonsuza bekler. Banking'de **production outage**.

#### `tryLock` — beklemeden dene

```java
public boolean tryDeposit(long amount) {
    if (lock.tryLock()) {
        try {
            balance += amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false;  // başka thread tutuyor, beklemeden dön
}
```

`tryLock()` immediately false döner eğer başka thread tutuyorsa. Banking pratiği: "Rate limiter, hesaba aynı anda max 1 işlem" senaryosunda.

#### `tryLock(timeout)` — sınırlı bekle

```java
public boolean depositOrTimeout(long amount, long timeoutMs) throws InterruptedException {
    if (lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
        try {
            balance += amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false;
}
```

Banking'de zaman duyarlı işlemler — transfer max 500ms'de tamamlanmazsa fail. Deadlock'tan çıkış yolu (sonraki bölüm).

#### `lockInterruptibly` — bekleyen thread interrupt'a duyarlı

```java
public void deposit(long amount) throws InterruptedException {
    lock.lockInterruptibly();
    try {
        balance += amount;
    } finally {
        lock.unlock();
    }
}
```

Eğer thread interrupt edilirse, bekliyorken `InterruptedException` fırlatır. Graceful shutdown'da kritik.

#### `synchronized` vs `ReentrantLock` — 5+ fark

| Özellik | `synchronized` | `ReentrantLock` |
|---|---|---|
| Sözdizimi | Dil seviyesi (keyword) | Kütüphane (class) |
| Acquire/release | Implicit (scope) | Explicit (lock/unlock) |
| Timeout / tryLock | YOK | VAR |
| Interrupt edilebilir bekleme | YOK | VAR (lockInterruptibly) |
| Fairness | Bias (yaklaşık FIFO yok) | Optional fair (constructor parameter) |
| Multiple condition variables | YOK (1 monitor.wait/notify) | VAR (newCondition()) |
| Lock acquisition by different scope | YOK (block sonu release) | VAR (method A acquire, method B release) |
| Virtual thread pinning | EVET (sorun) | YOK |
| Reentrant | EVET | EVET |
| Memory semantics | Acquire/release | Acquire/release |
| Performance (low contention) | Çok yakın | Çok yakın |
| Performance (high contention) | Çok yakın (modern JVM) | Genellikle hafif üstün |

**Mülakat:** "Ne zaman `synchronized`, ne zaman `ReentrantLock`?"

- Basit, kısa kritik bölge, fair gerekmiyor → `synchronized`
- Timeout, interrupt, fair, multiple condition gerek → `ReentrantLock`
- Virtual thread + uzun blocking → `ReentrantLock` (pinning yok)

---

### 3. Fairness — adil sıralama

`ReentrantLock` constructor:

```java
ReentrantLock fair = new ReentrantLock(true);     // FIFO
ReentrantLock unfair = new ReentrantLock(false);  // default, throughput-optimized
```

**Fair lock:** Bekleyen thread'leri sıraya alır, en uzun bekleyen ilk geçer. Adil ama performans cezası.

**Unfair lock:** Lock release olduğunda, yeni acquire isteyen thread "kuyruğa girmeden" girebilir (barging). Throughput yüksek ama bazı thread'ler aç kalabilir.

**Banking pratiği:** Fair lock genellikle gerek değil. Lock holding süresi kısa olduğunda starvation pratikte yok. Ama bir thread "her zaman geri çağrılıyor sürekli, hiç giremiyor" şikayeti varsa fair'e geç.

**Performans karşılaştırması:**
- Unfair: lock acquire ~30 ns
- Fair: lock acquire ~150 ns

---

### 4. `Condition` variable — wait/notify'ın gelişmişi

`Object.wait()` / `Object.notify()`'ın `Lock` API karşılığı.

```java
import java.util.concurrent.locks.Condition;

public class BlockingAccount {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition fundsAvailable = lock.newCondition();
    private long balance = 0;
    
    public void deposit(long amount) {
        lock.lock();
        try {
            balance += amount;
            fundsAvailable.signalAll();  // bekleyen withdraw'ları uyandır
        } finally {
            lock.unlock();
        }
    }
    
    public void withdrawBlocking(long amount) throws InterruptedException {
        lock.lock();
        try {
            while (balance < amount) {
                fundsAvailable.await();  // ← await lock'u release eder, bekler, dönünce reacquire
            }
            balance -= amount;
        } finally {
            lock.unlock();
        }
    }
}
```

**Kritik kurallar:**

- `await()` ve `signal()` **sahip lock'la** çağrılmalı (yoksa `IllegalMonitorStateException`)
- `await()` lock'u **release eder**, başka thread giriş yapabilir; uyanınca **reacquire** eder
- **`while (condition)`** kullan, `if` değil — **spurious wakeup**'lar (sebepsiz uyanma) mümkün

**Multiple condition** — `synchronized` yapamaz:

```java
public class BoundedAccountQueue {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Deque<Transfer> queue = new ArrayDeque<>();
    private final int capacity;
    
    public void offer(Transfer t) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) notFull.await();
            queue.offer(t);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    
    public Transfer take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await();
            var t = queue.poll();
            notFull.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }
}
```

Producer "doluyken" bekler, consumer "boşken" bekler. **Aynı lock**, iki farklı condition. `synchronized` ile bu kadar temiz yazılamaz.

#### `signal` vs `signalAll`

- `signal()` — bir bekleyen thread uyanır (hangisi belirsiz)
- `signalAll()` — tüm bekleyenler uyanır

**Pratik:** Eğer uyandırılan thread duruma uymuyorsa (`while` döngüsünde başka koşula girer ve tekrar bekler), tek bir thread bile aç kalabilir. Güvenli yol: `signalAll()`. Performans için `signal()`, ama emin ol ki kondisyon kesinlikle karşılanmış.

---

### 5. `ReentrantReadWriteLock` — okuyucu-yazıcı ayrımı

Çok okuma + nadiren yazma senaryosu için. Okuyucular **eş zamanlı** girebilir; yazıcı **tek başına**.

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class FxRateCache {
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();
    
    private Map<CurrencyPair, BigDecimal> rates = Map.of();
    
    public BigDecimal getRate(CurrencyPair pair) {
        readLock.lock();
        try {
            return rates.get(pair);
        } finally {
            readLock.unlock();
        }
    }
    
    public void reloadRates(Map<CurrencyPair, BigDecimal> newRates) {
        writeLock.lock();
        try {
            this.rates = Map.copyOf(newRates);  // immutable copy
        } finally {
            writeLock.unlock();
        }
    }
}
```

**Banking örneği — FX rate cache:** Her saniyede binlerce rate sorgusu (read), her 30 saniyede bir kez batch reload (write). RWLock ile reader'lar paralel çalışır.

**Karakteristikler:**

- Read lock **çoklu thread alabilir**
- Write lock **tek thread alabilir**, hiçbir read lock yokken
- Read lock varken write **bekler**
- Write lock varken read **bekler**
- Fairness opsiyonel (constructor)
- **Reader downgrade**: write lock'tan read lock'a geçiş güvenli ve klasik pattern
- **Reader upgrade YASAK**: read lock'tan write lock'a geçiş → deadlock (bekler kendisi için bırakmasını bekler)

**Downgrade pattern:**

```java
public BigDecimal getOrCompute(CurrencyPair pair) {
    readLock.lock();
    try {
        var cached = rates.get(pair);
        if (cached != null) return cached;
    } finally {
        readLock.unlock();
    }
    
    writeLock.lock();
    try {
        // double-check (başka thread arada eklemiş olabilir)
        var cached = rates.get(pair);
        if (cached != null) return cached;
        
        var fresh = computeRate(pair);
        rates.put(pair, fresh);
        
        // downgrade: write → read (atomically by acquiring read before releasing write)
        readLock.lock();
        return fresh;
    } finally {
        writeLock.unlock();
        // not: writeLock release sırasında read hâlâ tutulu, no race
        readLock.unlock();
    }
}
```

**Performans tuzağı:** Eğer read lock acquire/release süresi, kritik bölgenin işinden uzunsa, RWLock **avantaj sağlamaz**. Çok kısa kritik bölge için **`ConcurrentHashMap` veya `AtomicReference<Map>`** daha hızlı (allocation-free read).

**Mülakat sorusu:** "RWLock'u ne zaman kullanmazsın?" — Read kısa, write nadirse `ConcurrentHashMap`. Yazma sık ise `synchronized`. RWLock optimum spot **orta-uzun read**'lerde, **nadir write**'larda.

---

### 6. `StampedLock` — optimistic reading

Java 8'de geldi. RWLock'tan **daha hızlı** olabilir; optimistic read pattern'i.

```java
import java.util.concurrent.locks.StampedLock;

public class StampedFxCache {
    private final StampedLock lock = new StampedLock();
    private double rate;       // long bazlı double, sadece örnek için
    
    public double readOptimistic() {
        long stamp = lock.tryOptimisticRead();    // stamp 0 olabilir
        double local = rate;                       // unsafe read
        if (!lock.validate(stamp)) {               // optimistic fail
            stamp = lock.readLock();               // fall-back to read lock
            try {
                local = rate;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return local;
    }
    
    public void write(double newRate) {
        long stamp = lock.writeLock();
        try {
            rate = newRate;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

**Optimistic read** lock acquire **etmez**, sadece bir stamp alır. Okuma yapılır. Sonra `validate(stamp)` çağrılır:
- Stamp geçerliyse (write yapılmamışsa) → okuma kabul, **çok hızlı**
- Geçersizse → klasik read lock'a düş

**Avantajlar:**

- Read'ler **lock-free** olabilir (no atomic write, no contention)
- Çok yüksek read throughput

**Dezavantajlar:**

- **Reentrant DEĞİL** (tekrar alma destekli değil)
- Optimistic read'in `validate`'inden önce yapılan iş **tutarsız** olabilir — read sonucu güvenmeden başka iş yapma
- **Condition variable yok**
- Daha karmaşık API (stamp passing)
- `tryOptimisticRead()` 0 dönerse, **stamp yok** demek; write tutulu olabilir
- Karşı tarafta thread interrupt edilemez (`StampedLock.readLockInterruptibly` var ama default değil)

**Banking pratiği:** Çok yüksek read throughput'lu **immutable snapshot** (FX rate, config) için optimistic read uygun. Ama çoğu zaman `ConcurrentHashMap` veya `AtomicReference<ImmutableSnapshot>` daha basit ve **yeterince hızlı**.

---

### 7. Lock striping — contention dağıtma

Tek bir lock, milyon hesap için yetersiz. **Account başına lock** ise memory bombası. **Striping**: N adet lock, hesap ID'sini hash'leyerek bir lock seç.

```java
public class StripedAccountLocks {
    private static final int STRIPE_COUNT = 64;
    private final ReentrantLock[] stripes = new ReentrantLock[STRIPE_COUNT];
    
    public StripedAccountLocks() {
        for (int i = 0; i < STRIPE_COUNT; i++) {
            stripes[i] = new ReentrantLock();
        }
    }
    
    public Lock lockFor(long accountId) {
        int idx = (int) (Long.hashCode(accountId) & (STRIPE_COUNT - 1));  // power of 2
        return stripes[idx];
    }
}
```

Kullanım:

```java
var lock = stripeLocks.lockFor(account.getId());
lock.lock();
try {
    // hesap işlemi
} finally {
    lock.unlock();
}
```

**Trade-off:**
- Az stripe → contention (false positive: aynı stripe'ı 2 farklı hesap paylaşır)
- Çok stripe → memory + cache footprint

64 veya 128 stripe banking workload için yeterli. Google Guava'nın `Striped<Lock>` sınıfı production-ready bir implementation.

**Banking örneği:** ConcurrentHashMap'in Java 7 implementasyonu segment denen 16 stripe kullanıyordu (Java 8'den sonra lock-free, ama fikir aynı).

---

### 8. Deadlock — 4 koşul, klasik banking örneği

**Coffman koşulları** — deadlock'un dört zorunlu koşulu:

1. **Mutual Exclusion**: Bir kaynak (lock) tek thread tarafından tutulabilir.
2. **Hold and Wait**: Thread bir lock tutarken başka lock için bekler.
3. **No Preemption**: Lock zorla alınamaz (thread kendisi release etmeli).
4. **Circular Wait**: Bekleme bir döngü oluşturur (T1 → L1, L2 bekler; T2 → L2, L1 bekler).

Dördünden biri kırılırsa deadlock olamaz.

#### Banking deadlock — eş zamanlı A→B + B→A transfer

```java
public class DeadlockyBank {
    
    public void transfer(Account from, Account to, long amount) {
        synchronized (from) {            // L1
            synchronized (to) {           // L2
                if (from.getBalance() >= amount) {
                    from.debit(amount);
                    to.credit(amount);
                }
            }
        }
    }
}

// Senaryo:
// Thread 1: transfer(A, B, 100) → from.synchronized(A) ✓, to.synchronized(B) bekler
// Thread 2: transfer(B, A, 50)  → from.synchronized(B) ✓, to.synchronized(A) bekler
// DEADLOCK: T1 B bekler, T2 A bekler, ikisi de bırakmaz.
```

#### Reproduce — kasıtlı kod

```java
@Test
@Timeout(value = 10, unit = TimeUnit.SECONDS)
void demonstrateDeadlock() throws Exception {
    var bank = new DeadlockyBank();
    var a = new Account(1, 1000);
    var b = new Account(2, 1000);
    
    var t1 = new Thread(() -> {
        for (int i = 0; i < 1000; i++) {
            bank.transfer(a, b, 1);
        }
    }, "transfer-A-to-B");
    
    var t2 = new Thread(() -> {
        for (int i = 0; i < 1000; i++) {
            bank.transfer(b, a, 1);
        }
    }, "transfer-B-to-A");
    
    t1.start();
    t2.start();
    
    t1.join(5000);
    t2.join(5000);
    
    if (t1.isAlive() || t2.isAlive()) {
        System.out.println("DEADLOCK detected — running jstack now");
        // jstack <pid> ile incele
    }
}
```

Çoğunlukla saniyeler içinde deadlock olur. Hiçbir transfer ilerlemez.

#### `jstack` ile deadlock teşhisi (detay Topic 3.10)

```
$ jstack <pid>

"transfer-B-to-A" waiting to lock monitor 0x000000076b3c1f80 (object 0x000000076e2c3a48, a Account),
  which is held by "transfer-A-to-B"
"transfer-A-to-B" waiting to lock monitor 0x000000076b3c1fa0 (object 0x000000076e2c3b00, a Account),
  which is held by "transfer-B-to-A"

Java stack information for the threads listed above:
"transfer-B-to-A":
  at DeadlockyBank.transfer(DeadlockyBank.java:8)
"transfer-A-to-B":
  at DeadlockyBank.transfer(DeadlockyBank.java:8)

Found 1 deadlock.
```

`jstack` cycle'ı tespit eder. **Banking'de production deadlock**'unda ilk adım jstack.

---

### 9. Deadlock fix — lock ordering

**Çözüm:** Lock'ları **deterministik bir sırayla** acquire et. Account ID'sine göre.

```java
public class OrderedBank {
    
    public void transfer(Account from, Account to, long amount) {
        Account first, second;
        if (from.getId() < to.getId()) {
            first = from; second = to;
        } else if (from.getId() > to.getId()) {
            first = to; second = from;
        } else {
            throw new IllegalArgumentException("Same account");
        }
        
        synchronized (first) {
            synchronized (second) {
                if (from.getBalance() >= amount) {
                    from.debit(amount);
                    to.credit(amount);
                }
            }
        }
    }
}
```

T1 ve T2 her zaman önce **küçük ID**'yi acquire ederler. Circular wait kırıldı.

**Pratik notlar:**

- ID monoton ve **unique** olmalı. UUID ise lexicographic compare.
- ID eşit olamaz (aynı account → kendinden kendine transfer; business rule).
- Multi-account complex senaryolarda (3+ account) aynı kural geçerli; tümünü ID'ye göre sırala.

---

### 10. Deadlock fix — `tryLock` + timeout

Lock ordering bilinmiyorsa veya başka lock kaynakları varsa (dış sistem mutex'i):

```java
public boolean tryTransfer(Account from, Account to, long amount, 
                           long timeoutMs) throws InterruptedException {
    long deadline = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(timeoutMs);
    
    while (System.nanoTime() < deadline) {
        if (from.getLock().tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.getLock().tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        if (from.getBalance() >= amount) {
                            from.debit(amount);
                            to.credit(amount);
                        }
                        return true;
                    } finally {
                        to.getLock().unlock();
                    }
                }
                // toLock failed → unlock from and retry
            } finally {
                from.getLock().unlock();
            }
        }
        // small backoff
        TimeUnit.MILLISECONDS.sleep(ThreadLocalRandom.current().nextInt(10));
    }
    return false;  // timeout
}
```

**Yaklaşım:** İlk lock acquire edildi, ikincisi bekledi → release et, retry. Hold-and-wait kırıldı.

**Trade-off:** Daha karmaşık, retry overhead, fakat herhangi bir lock-order convention'ına bağlı değil.

**Pratik karar:** Lock ordering > tryLock pattern. ID-based ordering şık ve hızlı.

---

### 11. Diğer concurrency tuzakları

#### Livelock

Thread'ler aktif çalışıyor ama **ilerleme yok**. Klasik örnek:

```java
public void politeTransfer(Account from, Account to, long amount) {
    while (true) {
        if (from.tryLock()) {
            try {
                if (to.tryLock()) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return;
                    } finally {
                        to.unlock();
                    }
                }
            } finally {
                from.unlock();
            }
        }
        // ikisi de "ben kibarca bekleyeyim" diye sürekli yield ediyor
    }
}
```

İki thread sürekli try → fail → release → retry yapar; hiçbir ilerleme yok. Çözüm: **randomized backoff** (uyku süresini farklılaştır).

#### Starvation

Bir thread'in lock'a hiç giremiyor olması. Unfair lock + sürekli akan thread'ler → bekleyenler aç. Fair lock veya queue-based scheduling ile çözülür.

#### Priority inversion

Düşük öncelikli thread bir lock tutarken, yüksek öncelikli thread bekler. Üçüncü orta-öncelikli thread düşüğü preempt eder → yüksek bekliyor düşüğün lock'unu bırakmasını, düşük orta yüzünden çalışamıyor.

Java'da OS-level scheduling ile fazla kontrolün yok. Real-time JVM'ler dışında nadiren problem.

#### Lock convoy

Çok thread aynı lock'u sırayla almak istiyor. Lock release sonrası tüm bekleyenler aynı anda uyanır, çoğu retry'da rejected olur, CPU israfı.

Çözüm: lock contention'ı azalt (striping, lock-free, vs.).

---

### 12. Lock observability — banking pratiği

Production'da lock contention takip etmek için:

#### `lock.getQueueLength()` — kaç thread bekliyor

```java
ReentrantLock lock = new ReentrantLock();
// ...
log.info("Lock queue length: {}", lock.getQueueLength());
```

#### `lock.hasQueuedThreads()` — kuyrukta thread var mı

```java
if (lock.hasQueuedThreads()) {
    log.warn("Lock contention detected");
}
```

#### JFR (Java Flight Recorder) `jdk.JavaMonitorEnter` event

Otomatik tüm `synchronized` enter event'lerini yakalar. `jcmd JFR.start` ile başlat (Topic 3.10).

#### `async-profiler` `--lock` mode

Lock contention flame graph üretir (Topic 3.10).

#### Banking metric

Mikrometre veya Prometheus'a custom metric:

```java
public class InstrumentedLock {
    private final ReentrantLock lock;
    private final Timer lockTime;
    
    public void lock() {
        var sample = Timer.start();
        lock.lock();
        sample.stop(lockTime);
    }
}
```

p99 lock wait > 100ms → alert.

---

### 13. Banking pratik karar matrisi

| Senaryo | Lock seçimi |
|---|---|
| Basit balance update | `synchronized` veya `AtomicLong` |
| Multi-step transfer (debit + credit) | `synchronized` (account başına, ID-ordered) veya `ReentrantLock` |
| Transfer timeout gerekli | `ReentrantLock` + `tryLock(timeout)` |
| Per-account lock with millions of accounts | Striped lock veya optimistic versioning (`@Version`) |
| FX rate cache (read-heavy) | `ConcurrentHashMap` (genelde yeterli) veya `ReadWriteLock` |
| Configuration reload (rare write) | `volatile reference` veya `AtomicReference<ImmutableSnapshot>` |
| Producer-consumer queue | `BlockingQueue` (Topic 3.6) — kendin yazma |
| Wait-notify with multiple conditions | `ReentrantLock` + multiple `Condition` |
| Virtual thread + JDBC | `ReentrantLock` (synchronized pinning) |
| Highest throughput read, rare write | `StampedLock` optimistic read veya immutable AtomicReference |

---

### 14. Anti-pattern'ler

**Anti-pattern 1: `unlock` `finally`'de değil**

```java
lock.lock();
balance += amount;
if (somethingWrong) return;     // ❌ unlock atlanır
lock.unlock();
```

Çözüm: `try-finally`.

**Anti-pattern 2: `lock` sonrası exception, no unlock**

```java
lock.lock();
business();    // throws exception
lock.unlock(); // hiç çağrılmaz
```

Çözüm: `try-finally`.

**Anti-pattern 3: Reader-to-writer upgrade**

```java
readLock.lock();
try {
    if (needWrite) {
        writeLock.lock();   // ❌ DEADLOCK
    }
}
```

`ReentrantReadWriteLock` upgrade desteklemez. Önce read release, sonra write acquire, sonra durumu yeniden kontrol et (double-check).

**Anti-pattern 4: synchronized in virtual thread + JDBC**

```java
public synchronized void process() {
    jdbc.query(...);   // ❌ virtual thread pinning
}
```

Çözüm: `ReentrantLock` (Topic 3.7).

**Anti-pattern 5: Lock object as `this` exposure**

```java
class Account {
    public synchronized void deposit() { ... }
}

// Dış kod:
synchronized (account) {
    // ❌ iç lock'la çakışır
}
```

Çözüm: private final lock object.

**Anti-pattern 6: `synchronized` over external library object**

```java
synchronized (httpClient) {   // ❌ httpClient'ın iç implementation'ı sync kullanabilir
    httpClient.send(req);
}
```

Çözüm: Senin own lock object'in.

**Anti-pattern 7: Holding lock during IO**

```java
synchronized (this) {
    sendHttpRequest();    // ❌ IO sırasında lock tutuluyor; throughput çöker
}
```

Çözüm: IO'yu lock'tan **çıkar**. State değişikliklerini lock altında, IO'yu lock dışında yap.

---

## Önemli olabilecek araştırma kaynakları

- "Java Concurrency in Practice" Goetz, Chapter 13 (Explicit Locks)
- Doug Lea — "Concurrent Programming in Java" (kitap)
- AbstractQueuedSynchronizer (AQS) JavaDoc + source code
- Google Guava `Striped<Lock>` source
- "Effective Concurrency" series Herb Sutter (article)
- "StampedLock idioms" Heinz Kabutz article
- Coffman conditions Wikipedia
- "Deadlock prevention" textbook chapters
- jstack manual (Oracle docs)
- Aleksey Shipilev — "JMM workshop" lock visibility detayları

---

## Mini task'ler

### Task 3.3.1 — `ReentrantLock` ile temel account (30 dk)

`concurrency-playground/locks/AccountWithLock.java`:

- `private final ReentrantLock lock = new ReentrantLock();`
- `deposit(long amount)`, `withdraw(long amount)` (lock + finally + unlock)
- `tryDeposit(long amount, long timeoutMs)` — timeout
- `getBalance()` — read için de lock? (atomic long yerine lock — pedagojik)

Race test: AtomicAccount benzeri, 100 thread × 1000 deposit. Final balance kontrol.

### Task 3.3.2 — `synchronized` vs `ReentrantLock` 5 fark testi (30 dk)

`SyncVsLock.md` defter dosyası oluştur. 5 farkı **kendi cümlenle** yaz:

1. Acquire/release sözdizimi
2. Timeout (tryLock)
3. Interruptibility
4. Fairness
5. Multiple condition variables

Her birinin pratik banking örneğini ver.

### Task 3.3.3 — `Condition` ile blocking withdraw (45 dk)

`BlockingAccount.java`:

- `deposit(long amount)` — `notFull` veya benzer condition signal
- `withdraw(long amount)` — `balance >= amount` olana kadar `await`
- Test: 10 thread withdraw, 1 thread tek seferde büyük deposit → tüm withdraw'lar uyanır

### Task 3.3.4 — `ReadWriteLock` ile FX cache (45 dk)

`FxRateCache.java`:

- Map<CurrencyPair, BigDecimal> rates
- `getRate(pair)` — read lock
- `reloadAll(newRates)` — write lock
- Test: 100 reader thread, 1 writer thread. Reader throughput'u ölç.
- Karşılaştır: `ConcurrentHashMap` ile aynı senaryo. Hangi daha hızlı?

### Task 3.3.5 — `StampedLock` optimistic read (30 dk)

`StampedFxCache.java`:

- `getRate()` optimistic read + fallback
- `setRate()` write
- 100 thread read benchmark: stamped vs read-write-lock

### Task 3.3.6 — Lock striping (45 dk)

`StripedAccountLocks.java`:

- 64 stripe
- `lockFor(accountId)` — hash & masking
- Test: 1000 hesap, 100 thread, random transfer'lar. Tek lock vs stripe karşılaştır (throughput).

### Task 3.3.7 — Deadlock reproduction (45 dk)

`DeadlockyBank.java`:

- `transfer(Account, Account, long)` — synchronized(from) then synchronized(to)
- Demo main: T1 transfer A→B, T2 transfer B→A, ikisi 1000 kez
- Saniyeler içinde deadlock olmalı
- `jstack <pid>` çıktısını dosyaya kaydet: `deadlock-jstack.txt`
- Çıktıyı **defterine kopyala**, "Found 1 deadlock" satırını highlight et

### Task 3.3.8 — Deadlock fix lock ordering (30 dk)

`OrderedBank.java`:

- Aynı transfer, ID-ordered locks
- Aynı testi koş, deadlock yok, 1000 transfer tamamlanır

### Task 3.3.9 — Deadlock fix tryLock pattern (30 dk)

`TryLockBank.java`:

- Account'ta `ReentrantLock`
- `transfer` tryLock with timeout pattern
- Retry on second lock fail
- Aynı testi koş, deadlock yok

### Task 3.3.10 — Livelock demonstration (defter notu, 15 dk)

`LivelockDemo.java`:

- Yukarıdaki "politeTransfer" benzeri kod
- Backoff olmadan iki thread sürekli yield → ilerleme yok
- Çözüm: randomized backoff ekle (örn. `Thread.sleep(ThreadLocalRandom.current().nextInt(50))`)
- Defterine "livelock != deadlock — neden?" yaz

---

## Test yazma rehberi

### Test 3.3.1 — `AccountWithLock` thread-safety

```java
@Test
void accountWithLockIsThreadSafe() throws Exception {
    var account = new AccountWithLock();
    int threads = 100;
    int perThread = 1000;
    
    var pool = Executors.newFixedThreadPool(threads);
    var latch = new CountDownLatch(threads);
    for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
            for (int j = 0; j < perThread; j++) account.deposit(1);
            latch.countDown();
        });
    }
    latch.await();
    pool.shutdown();
    
    assertThat(account.getBalance()).isEqualTo((long) threads * perThread);
}
```

### Test 3.3.2 — `tryLock` timeout

```java
@Test
void tryDepositReturnsFalseWhenLockHeld() throws Exception {
    var account = new AccountWithLock();
    var t1Acquired = new CountDownLatch(1);
    var t1Release = new CountDownLatch(1);
    
    Thread t1 = new Thread(() -> {
        account.lock();
        try {
            t1Acquired.countDown();
            t1Release.await();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            account.unlock();
        }
    });
    t1.start();
    t1Acquired.await();
    
    boolean result = account.tryDeposit(100, 100);
    assertThat(result).isFalse();
    
    t1Release.countDown();
    t1.join();
}
```

### Test 3.3.3 — `Condition.await` ile withdraw blocking

```java
@Test
void withdrawBlocksUntilDeposit() throws Exception {
    var account = new BlockingAccount();
    var withdrawn = new CountDownLatch(1);
    
    Thread t = new Thread(() -> {
        try {
            account.withdrawBlocking(100);
            withdrawn.countDown();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
    t.start();
    
    // önce withdraw yapamamalı
    assertThat(withdrawn.await(200, TimeUnit.MILLISECONDS)).isFalse();
    
    account.deposit(150);
    
    // şimdi withdraw yapmalı
    assertThat(withdrawn.await(1, TimeUnit.SECONDS)).isTrue();
}
```

### Test 3.3.4 — `ReadWriteLock` ile concurrent reader'lar

```java
@Test
void readersDoNotBlockEachOther() throws Exception {
    var cache = new FxRateCache();
    cache.reloadRates(Map.of(USD_TRY, new BigDecimal("33.50")));
    
    var pool = Executors.newFixedThreadPool(50);
    var startedReaders = new CountDownLatch(50);
    var done = new CountDownLatch(50);
    
    for (int i = 0; i < 50; i++) {
        pool.submit(() -> {
            startedReaders.countDown();
            cache.getRate(USD_TRY);
            done.countDown();
        });
    }
    
    assertThat(startedReaders.await(2, TimeUnit.SECONDS)).isTrue();
    assertThat(done.await(2, TimeUnit.SECONDS)).isTrue();
    pool.shutdown();
}
```

### Test 3.3.5 — Deadlock detection (kasıtlı, timeout ile)

```java
@Test
@Timeout(value = 10, unit = TimeUnit.SECONDS)
void deadlockIsReproducible() throws Exception {
    var bank = new DeadlockyBank();
    var a = new Account(1, 10000);
    var b = new Account(2, 10000);
    
    var t1 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) bank.transfer(a, b, 1);
    }, "T1");
    var t2 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) bank.transfer(b, a, 1);
    }, "T2");
    t1.start();
    t2.start();
    
    t1.join(5000);
    t2.join(5000);
    
    // Beklenti: deadlock — en azından biri hâlâ alive
    assertThat(t1.isAlive() || t2.isAlive())
        .as("Deadlock beklenmiyordu mı?")
        .isTrue();
    
    t1.interrupt();
    t2.interrupt();
}
```

### Test 3.3.6 — `OrderedBank` no deadlock

```java
@Test
@Timeout(value = 10, unit = TimeUnit.SECONDS)
void orderedBankCompletesWithoutDeadlock() throws Exception {
    var bank = new OrderedBank();
    var a = new Account(1, 10000);
    var b = new Account(2, 10000);
    
    var t1 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) bank.transfer(a, b, 1);
    });
    var t2 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) bank.transfer(b, a, 1);
    });
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    
    // Sum invariant: a + b = 20000
    assertThat(a.getBalance() + b.getBalance()).isEqualTo(20000);
}
```

---

## Claude-verify prompt

```
Aşağıdaki Java kodum Lock API'lerini öğrenmeye yönelik. Lütfen şu kriterlere 
göre değerlendir ve EKSİKLERİ söyle, kod yazma:

1. AccountWithLock:
   - `lock.lock()` ve `unlock()` try-finally ile sarmalanmış mı?
   - tryLock ve tryLock(timeout) örneği var mı?
   - `getBalance()` thread-safe mi (ya lock ya volatile ya atomic)?

2. synchronized vs ReentrantLock farkları:
   - 5+ farkı listeli olarak yazılmış mı?
   - Her farkın pratik banking örneği var mı?

3. BlockingAccount:
   - ReentrantLock + Condition kullanılmış mı?
   - `await()` WHILE loop içinde mi (spurious wakeup'a karşı)?
   - `signal` vs `signalAll` arasında bilinçli seçim yapılmış mı?
   - Lock sahip değilken await/signal çağırma anti-pattern'i farkında mı?

4. FxRateCache (RWLock):
   - ReadLock ve WriteLock ayrı kullanılmış mı?
   - Reader downgrade pattern denenmiş mi?
   - Reader-to-writer upgrade anti-pattern'i farkında mı?
   - ConcurrentHashMap ile karşılaştırma yapılmış mı?

5. StampedLock:
   - tryOptimisticRead + validate pattern var mı?
   - Fallback to readLock var mı?
   - Reentrant DEĞİL olduğu not edilmiş mi?
   - Condition variable olmadığı bilinmiyor mu (yok hata mı?)?

6. Lock striping:
   - 64 veya 128 stripe array yazılmış mı?
   - Hash + masking ile stripe seçimi (power of 2) yapılmış mı?
   - Performance karşılaştırma testi yapılmış mı (single vs striped)?

7. Deadlock:
   - DeadlockyBank reproduce edilebilir mi (test timeout ile)?
   - jstack çıktısı kaydedilmiş mi (deadlock-jstack.txt)?
   - "Found 1 deadlock" satırı not edilmiş mi?
   - 4 Coffman koşulu yazılmış mı?
   - Hangi koşul kırılarak deadlock çözüldüğü açıklanmış mı?

8. Deadlock fix:
   - OrderedBank ID-ordered locking ile yazılmış mı?
   - TryLockBank tryLock + timeout + retry pattern ile yazılmış mı?
   - Aynı senaryo iki fix ile çalışmış mı (deadlock yok)?

9. Livelock:
   - LivelockDemo örneği var mı?
   - Randomized backoff ile fix yapılmış mı?
   - Livelock != deadlock farkı defterine yazılmış mı?

10. Anti-pattern kontrolü:
    - Unlock try-finally dışında kullanılmış mı?
    - synchronized this leak var mı?
    - Lock altında IO yapan kod var mı?
    - Reader-to-writer upgrade denenmiş mi?

Her madde için PASS / FAIL / EKSIK. Kod yazma, sadece eksiklikleri söyle.
```

---

## Tamamlama kriterleri

- [ ] `ReentrantLock.lock()/unlock()` try-finally pattern'ini her zaman uyguluyorum
- [ ] `tryLock(timeout)` ile bekleme süresi sınırlı transfer yazabiliyorum
- [ ] `lockInterruptibly` ile interrupt-safe lock yazabiliyorum
- [ ] `synchronized` vs `ReentrantLock` 5+ farkı ezberimde
- [ ] `Condition` variable'ı multi-condition senaryoda kullanabiliyorum
- [ ] `await` her zaman `while` döngüsünde (spurious wakeup'a karşı)
- [ ] `ReentrantReadWriteLock` ile FX cache yazdım, reader downgrade pattern'i biliyorum
- [ ] `StampedLock` optimistic read pattern'ini örnekle açıklayabiliyorum
- [ ] Lock striping ile 64 stripe ile per-account lock implement ettim
- [ ] Coffman 4 koşulunu ezberimde
- [ ] Deadlock reproduce ettim, `jstack` çıktısını okudum, "Found 1 deadlock" gördüm
- [ ] Lock ordering ile fix yaptım, aynı senaryo deadlock-free çalıştı
- [ ] tryLock + timeout + retry ile fix yaptım
- [ ] Livelock'u deadlock'tan ayırt edebiliyorum

Hepsi onaylı → Topic 3.4'e geç → [04-executor-framework/](../04-executor-framework/index.md)

---

## Defter notları

1. "ReentrantLock'un synchronized'e göre 5 avantajı: ____, ____, ____, ____, ____."
2. "Lock'u finally bloğunda release etmemenin sonucu: ____. Production banking'de bu ____ demek."
3. "Fair lock vs unfair lock — performans/adalet trade-off: ____."
4. "`Condition.await` her zaman `while` içinde çünkü: ____."
5. "Multiple condition variable kullanmanın senaryosu (banking): ____."
6. "ReadWriteLock'u ConcurrentHashMap'ten tercih ettiğim senaryo: ____."
7. "Reader-to-writer upgrade neden yasak: ____. Downgrade neden serbest: ____."
8. "StampedLock'un Reentrant lock'a göre avantajı: ____. Dezavantajı: ____."
9. "Coffman 4 koşulu: ____, ____, ____, ____. Bunlardan birini kırmak deadlock'u önler."
10. "Banking transfer'de lock ordering: hangi field'a göre sırala, neden: ____."
