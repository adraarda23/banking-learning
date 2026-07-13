# Topic 3.2 — Synchronization Primitives: synchronized, volatile, Atomics, CAS

## Hedef

Java'nın **temel senkronizasyon araçlarını** banking-grade derinlikte öğrenmek. `synchronized`'ın monitor lock semantiğini, `volatile`'in sınırlarını (tekrar — Topic 3.1'i pekiştirerek), atomic class ailesini (`AtomicInteger`, `AtomicLong`, `AtomicReference`, `AtomicReferenceFieldUpdater`), **CAS** (Compare-And-Swap) primitivesi ve ABA problemini, modern eklenti olan `LongAdder` / `DoubleAdder`'ı, `LockSupport.park/unpark` ile düşük seviye thread durdurma/uyandırmayı, ve modern `VarHandle` API'sini kavramak. Banking'de gerçek bir **balance update race condition**'ını adım adım reproduce edip her primitivle çözmek.

## Süre

Okuma: 2.5 saat • Mini task'ler: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Topic 3.1 (JMM, happens-before, volatile semantics) tamamlandı
- `BigDecimal`, `Money` value object'i (Phase 1)
- Spring Boot service'lerinde basit business logic yazılabiliyor

---

## Kavramlar

### 1. `synchronized` — monitor lock derinlemesine

`synchronized` Java'nın **dilden gelen** kilitleme primitivesi. Her object'in bir **intrinsic monitor (mutex)** vardır. `synchronized` block'a giren thread, o object'in monitor'unu **acquire** eder; bloktan çıkarken **release** eder.

#### Kullanım formları

```java
class AccountSafe {
    private long balance;
    
    // 1. Instance method'a sync: monitor = `this`
    public synchronized void deposit(long amount) {
        balance += amount;
    }
    
    // 2. Static method'a sync: monitor = Class object
    public static synchronized void resetGlobalCounter() { /* ... */ }
    
    // 3. Block sync: explicit monitor
    private final Object lock = new Object();
    public void slowDeposit(long amount) {
        synchronized (lock) {
            balance += amount;
        }
    }
}
```

**Hangisini seç?**

- **`synchronized` instance method**: junior için en basit, ama monitor `this` olduğu için **dışarıdan** `synchronized (account) { ... }` ile aynı monitor üzerinde yarış oluşturulabilir. Disiplin gerekir.
- **`synchronized` static method**: monitor `Class` object'i. Aynı sınıfın tüm static method çağrıları tek monitor'ı paylaşır. Genellikle istenmez.
- **Block + private final Object lock**: **tercih edilen biçim**. Dışarıdan çakışmaz, scope dar tutulur, performans için kritik.

Banking pratiği: kritik kaynak başına **kendi private lock object'i** kullan. `this`'i lock olarak kullanma — dış kod yanlışlıkla aynı monitor'da bekler.

#### Reentrant özelliği

`synchronized` **reentrant**'tır: aynı thread, sahip olduğu monitor'ı tekrar acquire edebilir, deadlock olmaz.

```java
class Reentrant {
    public synchronized void outer() {
        inner();  // aynı monitor reacquire — OK
    }
    public synchronized void inner() {
        // ...
    }
}
```

İçeride bir sayaç tutulur (lock count). Her çıkış sayacı azaltır. Sıfıra inince gerçekten release.

#### Memory semantics (Topic 3.1 tekrar)

`synchronized` block'a giriş → **acquire fence**: tüm sonraki okumalar bu noktadan sonra olur, monitor'un release tarafından yazılmış her şey görünür.

`synchronized` block'tan çıkış → **release fence**: tüm önceki yazılar bu noktadan önce flush edilir, sonraki acquire eden görür.

Yani `synchronized` **hem mutual exclusion hem visibility** sağlar. `volatile` sadece visibility. `synchronized` daha güçlü ama daha pahalı.

#### Biased locking (HISTORICAL — JDK 15+ kaldırıldı)

JDK 8-14 arası: aynı thread monitor'ı **tekrar tekrar** alıyorsa, monitor "biased to that thread" işaretlenir; sonraki acquire'lar **çok ucuz** olur (CAS bile değil).

JDK 15'te `-XX:UseBiasedLocking` **default kapalı**, JDK 18'de **kaldırıldı**. Sebep: modern multi-thread iş yüklerinde maintenance maliyeti faydadan büyük. Modern uygulamalar contention'a karşı `ReentrantLock`, atomic, veya lock-free yaklaşımları kullanır.

Mülakat sorusu: "Biased locking nedir, neden kaldırıldı?" — JDK 15'te disabled, 18'de kaldırıldı; modern workload artık tek-thread'li mutex pattern'ini kullanmıyor.

#### Lightweight + heavyweight locks

JVM monitor implementation'ında üç seviye var:

- **Biased lock** (kaldırıldı)
- **Lightweight lock**: contention yokken CAS ile object header'a thread ID yaz
- **Heavyweight lock**: contention varsa OS-level mutex (futex / Mach kernel mutex), thread'ler **park**'a alınır

JIT bunları transparently yönetir. Sen sadece `synchronized` yaz.

#### `synchronized` ne zaman seçilir?

- **Kısa kritik bölge** (microsaniyeler), low contention
- Kod basitliği önemli
- Reentrant ihtiyaç var (zaten default)
- Reader-writer ayrımı **yok** (her erişim mutual exclusive)

**Ne zaman değil:**

- Yüksek contention (multi-thread çekişiyor) — `ReentrantLock` veya atomic daha hızlı
- Lock timeout veya interrupt gerek — `synchronized` tryLock veya lockInterruptibly desteklemez
- Reader > writer çok fazla — `ReentrantReadWriteLock`
- Counter, gauge gibi tek bir field artırma — atomic
- Virtual thread (Topic 3.7) içinde + JDBC çağrısı — **pinning** problemi (`synchronized` virtual thread'i carrier'a pinler)

---

### 2. `volatile` — tekrar ama derin

Topic 3.1'de gördük. Burada **pratik tuzaklar** üzerine duralım.

#### Atomicity'nin olmaması

```java
volatile long counter;
counter++;  // ❌ read, increment, write — üç adım
```

İki thread aynı anda counter=5 okur, ikisi de 6 yazar. **Bir artış kaybedilir.**

`volatile` sadece **yazma → görünürlük** ve **yazma sonrası okuma → en güncel** garantisi. Read-modify-write atomic değil.

#### Volatile array vs array of volatile

```java
volatile int[] arr;          // ✓ array referansı volatile, içerikler değil
int[] arr2;                  // array reference normal
// arr2[i] = x; volatile değil
```

Array elemanlarının her birini volatile yapmak için **`AtomicIntegerArray`** veya **`VarHandle`** kullan.

#### Volatile + compound check

```java
volatile boolean enabled;

if (enabled) {                     // okuma
    enabled = false;               // yazma — başka thread arada başka bir şey yapmış olabilir
    doWork();
}
```

`if-then-set` atomic değil. CAS gerekir.

#### Pratik kullanım örnekleri

- **Flag (boolean)**: shutdown signal, configuration reload trigger
- **Configuration reference** tek-yazan (sahip thread güncellet) → `volatile Configuration config`
- **Sequence number** — TEK thread artırıyorsa, ÇOK thread okuyorsa
- **State machine state** — değişim atomic ise (single field)

---

### 3. Atomic class ailesi — `java.util.concurrent.atomic`

Bu paket **lock-free** primitive'lerin sınıflarını içerir. Hepsi `Unsafe.compareAndSwap*` (veya yeni JDK'larda `VarHandle.compareAndSet`) üzerine inşa.

#### `AtomicInteger` ve `AtomicLong`

```java
var counter = new AtomicLong(0);

counter.incrementAndGet();              // ++counter
counter.getAndIncrement();              // counter++
counter.addAndGet(5);                   // counter += 5
counter.compareAndSet(10, 20);          // if (counter == 10) counter = 20;
counter.updateAndGet(v -> v * 2);       // counter = f(counter) atomically
counter.accumulateAndGet(5, Long::sum); // counter = sum(counter, 5)
```

**Banking örneği — atomic transfer counter:**

```java
public class TransferMetrics {
    private final AtomicLong successCount = new AtomicLong();
    private final AtomicLong failureCount = new AtomicLong();
    private final AtomicLong totalAmount = new AtomicLong();
    
    public void recordSuccess(long amountKurus) {
        successCount.incrementAndGet();
        totalAmount.addAndGet(amountKurus);
    }
    
    public void recordFailure() {
        failureCount.incrementAndGet();
    }
    
    public Snapshot snapshot() {
        return new Snapshot(successCount.get(), failureCount.get(), totalAmount.get());
    }
    
    public record Snapshot(long success, long failure, long total) {}
}
```

#### `AtomicReference<T>`

Referansa atomic CAS yapar. Banking için **immutable state holder** olarak çok kullanışlı:

```java
public class AccountStateHolder {
    private final AtomicReference<AccountState> ref;
    
    public AccountStateHolder(AccountState initial) {
        this.ref = new AtomicReference<>(initial);
    }
    
    public boolean tryDebit(long amount) {
        while (true) {
            var current = ref.get();
            if (current.balance() < amount) return false;
            var next = current.withBalance(current.balance() - amount);
            if (ref.compareAndSet(current, next)) {
                return true;
            }
            // CAS fail → retry
        }
    }
    
    public record AccountState(long balance, long version) {
        AccountState withBalance(long newBalance) {
            return new AccountState(newBalance, version + 1);
        }
    }
}
```

Bu **lock-free** bir debit. `synchronized` yok. Concurrent transfer'lar CAS ile yarışır; başarısız olan retry yapar.

**Pratik notlar:**
- Retry loop'u **bounded** yapmak iyi olabilir (örn. max 1000 retry, sonra fall-back to lock)
- `updateAndGet(Function<T,T>)` aynı şeyi içinde retry loop ile yapar; daha kısa kod
- Lambda **side-effect free** olmalı — CAS fail'de tekrar çağrılır

#### `AtomicIntegerArray`, `AtomicLongArray`, `AtomicReferenceArray`

```java
var slots = new AtomicLongArray(16);
slots.incrementAndGet(5);     // index 5
slots.compareAndSet(0, expected, newValue);
```

Banking'de örnek: ledger account balance'ları için **fixed-size cache**:

```java
public class AccountBalanceCache {
    private final AtomicLongArray balances;
    private final int size;
    
    public AccountBalanceCache(int size) {
        this.size = size;
        this.balances = new AtomicLongArray(size);
    }
    
    public boolean tryDebit(int accountIdx, long amount) {
        return balances.getAndUpdate(accountIdx, b -> 
            b >= amount ? b - amount : b
        ) >= amount;
        // Dikkat: getAndUpdate döndüğü ESKI değer; check ona göre.
    }
}
```

(Production'da bu kadar primitiv değil ama atomic array fikrini gösteriyor.)

#### `AtomicReferenceFieldUpdater` ve `AtomicIntegerFieldUpdater`

`AtomicReference<T>` her field için ekstra wrapper allocation maliyeti. Eğer **var olan** bir field'a atomic CAS yapmak istiyorsan, **FieldUpdater** kullan:

```java
class Account {
    private volatile long balance;  // ← volatile zorunlu
    
    private static final AtomicLongFieldUpdater<Account> BALANCE_UPDATER =
        AtomicLongFieldUpdater.newUpdater(Account.class, "balance");
    
    public boolean tryDebit(long amount) {
        while (true) {
            long current = balance;
            if (current < amount) return false;
            if (BALANCE_UPDATER.compareAndSet(this, current, current - amount)) {
                return true;
            }
        }
    }
}
```

**Avantaj:** Bir tane static `FieldUpdater`, milyon Account için tek. Memory tasarrufu.

**Dezavantaj:** Reflection kullanır, hatalı field ismi runtime fail. Modern alternatif: **`VarHandle`** (madde 9).

#### `AtomicBoolean`

CAS'lı boolean. Singleton initialization, "tek-sefer-çalıştır" kontrolünde kullanışlı:

```java
class OnceInitializer {
    private final AtomicBoolean initialized = new AtomicBoolean(false);
    
    public void initialize() {
        if (initialized.compareAndSet(false, true)) {
            // sadece BIR kez çalışacak
            doExpensiveInit();
        }
    }
}
```

---

### 4. CAS (Compare-And-Swap) — donanım seviyesinde

CAS, modern CPU'ların sağladığı **atomik talimat**:

> "Eğer bellek konumundaki değer X ise, Y yap. Aksi halde yapma. Tek bir atomik adımda."

x86: `CMPXCHG` (ve `LOCK CMPXCHG` multi-core için).
ARM: `LDREX/STREX` (Load-Exclusive/Store-Exclusive) çiftleri.

Java'da `Atomic*.compareAndSet(expected, new)` direkt bu enstrüksiyona iner.

#### CAS semantiği

```
boolean cas(memory, expected, newValue) {
    atomic {
        if (memory == expected) {
            memory = newValue;
            return true;
        } else {
            return false;
        }
    }
}
```

#### CAS-loop pattern

```java
long current;
do {
    current = atom.get();
    long next = compute(current);
} while (!atom.compareAndSet(current, next));
```

Bu pattern lock-free algoritmaların temelidir. CAS başarısız ise retry. Başarısız olan **diğer bir thread başarılı oldu** demek; yani **progress garantili** (lock-free).

#### CAS pros/cons

**Pros:**
- Lock yok → blocking yok → priority inversion, deadlock yok
- Yüksek contention'da bile bazı thread'ler ilerler (lock-free progress)
- Çok hızlı (donanım enstrüksiyon)

**Cons:**
- High contention'da çok retry → CPU spinning, faydasız iş
- ABA problemi (sonraki madde)
- Read-modify-write tek bir field'la sınırlı (kompleks transaction zor)
- Karmaşık kod (lock-free veri yapıları korkunç zor)

---

### 5. ABA problemi ve `AtomicStampedReference`

CAS'ın ünlü tuzağı: bir değişken A→B→A olabilir, CAS bunu fark etmez (sadece **mevcut değer == beklenen** kontrol eder).

**Klasik örnek — lock-free stack:**

```java
class LockFreeStack<T> {
    private final AtomicReference<Node<T>> head = new AtomicReference<>();
    
    public void push(T value) {
        var newNode = new Node<>(value);
        Node<T> oldHead;
        do {
            oldHead = head.get();
            newNode.next = oldHead;
        } while (!head.compareAndSet(oldHead, newNode));
    }
    
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) return null;
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead, newHead));
        return oldHead.value;
    }
}
```

**ABA senaryo:**
1. Thread T1: `pop()` çağırır. `head = A`, `newHead = B` hesaplar.
2. T1 preempted (OS scheduler).
3. T2: `pop()` → A çıkar (head=B).
4. T2: `pop()` → B çıkar (head=C).
5. T2: `push(A)` → head=A (recycled).
6. T1 uyanır, `head.compareAndSet(A, B)` → **başarılı!** Ama B artık valid değil (zaten pop edilmiş).
7. head=B (kullanılmaz değer). Stack korupte.

**Çözüm:** `AtomicStampedReference` — değer + versiyon:

```java
var ref = new AtomicStampedReference<Node<T>>(initialHead, 0);

int[] stampHolder = new int[1];
var current = ref.get(stampHolder);
int currentStamp = stampHolder[0];

boolean success = ref.compareAndSet(current, newHead, currentStamp, currentStamp + 1);
```

Versiyon her güncellemede artar, ABA artık yakalanır.

#### Banking ABA senaryosu

```
Account.balance:
  T1 reads 1000 (planning to subtract 500 → 500)
  T2 reads 1000, withdraws 500 → balance 500
  T2 deposits 500 → balance 1000
  T1 CAS(1000, 500) → success
  Sonuç: T1 ve T2 birlikte 500 + 500 = 1000 çekti, ama balance hâlâ 500
```

Bu kritik bir hata. Çözüm:
- **Versiyon**: `AtomicStampedReference<Long>` veya record içinde `version` field
- **Lock**: `synchronized` veya `ReentrantLock` ile read+write tek transaction
- **JPA optimistic lock**: `@Version` (Phase 2'de gördük, distributed çözüm)

#### `AtomicMarkableReference` — alternative

ABA'nın özel hali — bir field'ı `(value, boolean marked)` olarak tut. Versiyon yerine "deleted/active" flag. CAS hem değeri hem flag'i kontrol eder. Linked list'lerden node logical delete için kullanılır.

---

### 6. `LongAdder`, `DoubleAdder` — high-contention counter

`AtomicLong.incrementAndGet()` yüksek contention'da yavaş çünkü tüm thread'ler **aynı cache line**'a yarışır. CAS retry'lar artar.

**`LongAdder`** çözümü: değeri **birden çok cell**'e böler (striped sum). Her thread kendi cell'ine yazar. `sum()` çağrısında tüm cell'leri toplar.

```java
var counter = new LongAdder();
counter.increment();
counter.add(5);
counter.sum();      // tüm cell'lerin toplamı
counter.reset();    // sıfırla (test/metric reset)
```

**Banking örneği:** her saniyede 50k TPS'lik bir sistem için **istatistik counter**:

```java
public class HighThroughputMetrics {
    private final LongAdder transferCount = new LongAdder();
    private final LongAdder failureCount = new LongAdder();
    private final DoubleAdder totalAmount = new DoubleAdder();  // dikkat: double money değil!
    // burada total long olabilir; sadece example
    
    public void recordTransfer(long amount) {
        transferCount.increment();
        totalAmount.add(amount);
    }
    
    public Snapshot snapshot() {
        return new Snapshot(
            transferCount.sum(), 
            failureCount.sum(), 
            totalAmount.sum()
        );
    }
}
```

**Trade-off:**
- `AtomicLong`: kesin "şu an" değer alma kolay, low contention'da daha hızlı
- `LongAdder`: high contention'da çok daha hızlı, **`sum()`** kesin "anlık" snapshot vermeyebilir (tutarlı ama point-in-time aproximate)

**Karar:** Banking metrik counter (TPS, error count) → `LongAdder`. Sayaç check + action (rate limiter) → `AtomicLong` veya `Semaphore`.

#### `LongAccumulator` ve `DoubleAccumulator`

Generalize edilmiş Adder. Custom combine function alır.

```java
var maxLatency = new LongAccumulator(Long::max, 0);
maxLatency.accumulate(50);
maxLatency.accumulate(120);
maxLatency.accumulate(30);
maxLatency.get();  // 120
```

Banking örneği: en uzun transfer süresi (latency monitoring).

---

### 7. `LockSupport.park` ve `unpark`

**Düşük seviye** thread durdurma/uyandırma primitivesi. `ReentrantLock`, `Semaphore`, `BlockingQueue` gibi yüksek seviye sınıfların **altında** bu yatar.

```java
LockSupport.park();         // mevcut thread'i parka al
LockSupport.unpark(thread); // belirli thread'i uyandır
LockSupport.parkNanos(100_000_000);  // 100ms timeout
```

#### `park`/`unpark` vs `wait`/`notify` farkları

| Özellik | park/unpark | wait/notify |
|---|---|---|
| Monitor gereksinimi | YOK | Monitor sahibi olmak gerek |
| Permit semantics | ✓ (unpark önce gelirse hatırlanır) | ✗ (notify, wait'i bekleyen yoksa kaybolur) |
| Spesifik thread'i uyandırma | ✓ (unpark(t)) | ✗ (notify random; notifyAll hepsi) |
| Interrupt'a duyarlı | ✓ (park çıkar, status set) | ✓ (InterruptedException) |

Permit modeli: her thread'in bir **permit**'i vardır (0 veya 1). `unpark` permit'i 1 yapar. `park` permit varsa hemen alır, yoksa bekler.

**Tipik kullanım — custom lock:**

```java
class SimpleLock {
    private final AtomicReference<Thread> owner = new AtomicReference<>();
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<>();
    
    public void lock() {
        while (!owner.compareAndSet(null, Thread.currentThread())) {
            waiters.add(Thread.currentThread());
            // double-check after enqueueing
            if (!owner.compareAndSet(null, Thread.currentThread())) {
                LockSupport.park(this);
            }
            waiters.remove(Thread.currentThread());
        }
    }
    
    public void unlock() {
        owner.set(null);
        var next = waiters.peek();
        if (next != null) LockSupport.unpark(next);
    }
}
```

Bu **production-grade değil** (spurious wakeup, cancellation handling eksik). `ReentrantLock`'un Doug Lea implementation'ı çok daha sofistike. Ama park/unpark'ın motivasyonunu gösterir.

#### Banking'de doğrudan kullanım nadiren olur

Sen muhtemelen `LockSupport`'u doğrudan çağırmazsın. **Concurrent kütüphane içinde** görürsün. Mülakat sorusu: "ReentrantLock altında nasıl çalışır?" → AQS (AbstractQueuedSynchronizer) + park/unpark.

---

### 8. `VarHandle` — modern `Unsafe` replacement

Java 9'da geldi. `sun.misc.Unsafe` yerine **standart, güvenli** memory access API.

```java
class Account {
    private long balance;
    
    private static final VarHandle BALANCE_VH;
    static {
        try {
            BALANCE_VH = MethodHandles.lookup()
                .findVarHandle(Account.class, "balance", long.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
    
    public boolean tryDebit(long amount) {
        while (true) {
            long current = (long) BALANCE_VH.getAcquire(this);  // volatile-like read
            if (current < amount) return false;
            if (BALANCE_VH.compareAndSet(this, current, current - amount)) {
                return true;
            }
        }
    }
}
```

#### Access modes

VarHandle çeşitli "ne kadar güçlü senkronizasyon" seçenekleri sunar:

| Mode | Açıklama |
|---|---|
| `get`, `set` | Plain — JMM garantisi yok |
| `getOpaque`, `setOpaque` | Aynı thread'de bitness garantisi (rarely needed) |
| `getAcquire`, `setRelease` | C++ acquire/release semantics — bazı barrier'lar |
| `getVolatile`, `setVolatile` | Full volatile semantics |
| `compareAndSet`, `compareAndExchange` | CAS varyantları |
| `getAndAdd`, `getAndSet`, `getAndBitwiseOr` | atomik RMW |

Çoğu banking kodunda `getVolatile/setVolatile` ve `compareAndSet` yeterli. Acquire/release pattern'i performance-critical concurrent veri yapılarında.

#### `VarHandle` vs `Atomic*FieldUpdater`

VarHandle:
- Tip güvenli (cast var ama compile-time check yok — runtime'da exact match)
- Daha hızlı (JIT için optimize edilir)
- Array element'lerine de erişebilir
- `static` field'lara da erişebilir

FieldUpdater:
- Reflection-based, eski API
- Java 8'de kullanılırdı
- Modern kodda **VarHandle tercih**

---

### 9. Banking race condition reproduction — adım adım

Senaryo: `Account.balance` field'ı, eş zamanlı 100 thread'in `deposit(10)` çağırması.

#### Versiyon 1 — naif, race kaybı

```java
public class RaceyAccount {
    private long balance = 0;
    
    public void deposit(long amount) {
        balance += amount;  // ❌ race
    }
    
    public long getBalance() {
        return balance;
    }
}
```

Test:

```java
@Test
void shouldShowRaceCondition() throws Exception {
    var account = new RaceyAccount();
    int threads = 100;
    int perThread = 1000;
    
    var pool = Executors.newFixedThreadPool(threads);
    var latch = new CountDownLatch(threads);
    for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
            for (int j = 0; j < perThread; j++) {
                account.deposit(1);
            }
            latch.countDown();
        });
    }
    latch.await();
    pool.shutdown();
    
    long expected = (long) threads * perThread;
    long actual = account.getBalance();
    System.out.println("Expected: " + expected + ", Actual: " + actual);
    // Actual genellikle expected'tan ÇOK az (race kayıpları)
}
```

Çıktı (örnek): `Expected: 100000, Actual: 87432`. ~12% kaybı.

#### Versiyon 2 — synchronized

```java
public synchronized void deposit(long amount) {
    balance += amount;
}
```

Test: balance == 100000. ✓

Performans: lock contention nedeniyle yavaş, ama doğru.

#### Versiyon 3 — AtomicLong

```java
private final AtomicLong balance = new AtomicLong();

public void deposit(long amount) {
    balance.addAndGet(amount);
}

public long getBalance() {
    return balance.get();
}
```

Test: 100000. ✓

Performans: synchronized'tan daha hızlı (CAS-based, lock-free).

#### Versiyon 4 — LongAdder (high throughput)

```java
private final LongAdder balance = new LongAdder();

public void deposit(long amount) {
    balance.add(amount);
}

public long getBalance() {
    return balance.sum();
}
```

Test: 100000. ✓

Performans: çok yüksek contention'da en hızlı. **Ama** withdraw / check-before-write için uygun değil (sum atomic single snapshot vermez).

#### Versiyon 5 — withdraw + check (compound op)

Burada `balance >= amount` kontrolü + subtract atomic olmalı. `LongAdder` **yetersiz**. `AtomicLong` ile CAS-loop:

```java
private final AtomicLong balance = new AtomicLong();

public boolean tryWithdraw(long amount) {
    while (true) {
        long current = balance.get();
        if (current < amount) return false;
        if (balance.compareAndSet(current, current - amount)) {
            return true;
        }
    }
}
```

Veya kısa hali:

```java
public boolean tryWithdraw(long amount) {
    long previous = balance.getAndUpdate(b -> b >= amount ? b - amount : b);
    return previous >= amount;
}
```

**Test:** 100 thread aynı anda `tryWithdraw(10)` çağırsın. Başlangıç balance 500.

```java
@Test
void noOverdraftAllowed() throws Exception {
    var account = new AtomicAccount(500);
    int threads = 100;
    var successes = new AtomicInteger();
    var pool = Executors.newFixedThreadPool(threads);
    var latch = new CountDownLatch(threads);
    
    for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
            if (account.tryWithdraw(10)) successes.incrementAndGet();
            latch.countDown();
        });
    }
    latch.await();
    pool.shutdown();
    
    assertThat(successes.get()).isEqualTo(50);  // 500 / 10
    assertThat(account.getBalance()).isZero();
}
```

50 başarı, 50 başarısızlık. Hiç overdraft yok.

---

### 10. Anti-pattern'ler

**Anti-pattern 1: "synchronized + this leak"**

```java
class Account {
    public synchronized void deposit(long a) { ... }
}

// Dış kod:
synchronized (account) {     // ❌ aynı monitor!
    // yanlışlıkla iç metotları bloke ediyor
}
```

Çözüm: private final lock object kullan.

**Anti-pattern 2: "Atomic ama atomic değil"**

```java
private final AtomicInteger count = new AtomicInteger();
private final AtomicLong total = new AtomicLong();

public void record(long amount) {
    count.incrementAndGet();      // atomik
    total.addAndGet(amount);      // atomik
    // ama (count, total) PAIR atomik değil
}

public Snapshot snapshot() {
    return new Snapshot(count.get(), total.get());
    // count okundu, sonra total okundu — arada record yapılmış olabilir
}
```

Eğer snapshot **kesinlikle** consistent olmalıysa: `synchronized` veya `AtomicReference<record>`.

**Anti-pattern 3: "Volatile + read-modify-write"**

```java
volatile int counter;
counter++;     // ❌ race
```

**Anti-pattern 4: "CAS retry'da side-effect"**

```java
ref.updateAndGet(state -> {
    log.info("Updating state");  // ❌ retry'da çoklu log!
    sendNotification();           // ❌ retry'da çoklu mail!
    return state.withBalance(...);
});
```

CAS lambda **pure function** olmalı. Yan etkileri retry loop sonrası yap.

**Anti-pattern 5: "LongAdder ile check-and-act"**

```java
private final LongAdder balance = new LongAdder();

public boolean tryWithdraw(long amount) {
    if (balance.sum() >= amount) {       // ❌ sum() snapshot atomic değil
        balance.add(-amount);
        return true;
    }
    return false;
}
```

LongAdder counter için. Check-and-act için AtomicLong CAS.

**Anti-pattern 6: "Biased locking için optimize etme"**

Modern JDK'larda biased locking yok. "Biased locking için fast path" varsayımı yapma. Genel concurrent veri yapıları + atomic kullan.

**Anti-pattern 7: "Atomic field tek başına yeterli sanma"**

```java
class Account {
    private final AtomicLong balance = new AtomicLong();
    private LocalDateTime lastTxn;    // ❌ atomic değil + ilişkili!
    
    public void deposit(long amount) {
        balance.addAndGet(amount);
        lastTxn = LocalDateTime.now();
    }
}
```

balance ve lastTxn iki bağımsız field; tutarsız okunabilir. State'i bir `record` içine al, `AtomicReference<State>` kullan.

---

### 11. Banking pratik karar matrisi

| Senaryo | Primitif |
|---|---|
| Tek bir sayaç (TPS metrics) | `LongAdder` |
| Balance read-modify-write | `AtomicLong` + CAS / `synchronized` |
| Konfigürasyon flag (hot reload) | `volatile boolean` |
| Account state (balance + version + lastUpdated) | `AtomicReference<record>` |
| Singleton initialization (tek-sefer) | `AtomicBoolean.compareAndSet` |
| Shutdown signal | `volatile boolean` |
| Stack/Queue lock-free | `ConcurrentLinkedQueue` (yapma kendin) |
| Çok-thread yazar, çok-thread okur (compound state) | `synchronized` blok veya `ReentrantLock` (Topic 3.3) |
| Tek-thread yazar, çok-thread okur (görünürlük) | `volatile` |
| Field-level CAS, allocation kaçınma | `VarHandle` |

---

## Önemli olabilecek araştırma kaynakları

- "Java Concurrency in Practice" Brian Goetz, Chapter 15 (atomics)
- "The Art of Multiprocessor Programming" Maurice Herlihy, Nir Shavit (kitap, lock-free algorithms)
- Doug Lea j.u.concurrent kaynak kodu (`AtomicLong`, `LongAdder` source)
- `VarHandle` JEP 193
- Aleksey Shipilev — "Atomic*::lazySet is a Special Tool" (article)
- "Compare-and-Swap" Wikipedia
- Maurice Herlihy — "Wait-free synchronization" (1991 paper, hazır okursan derin)
- ABA problem Wikipedia
- "Lock-free programming considerations" Microsoft article (Bizans karmaşıklığını gösterir)
- jcstress test örnekleri

---

## Mini task'ler

### Task 3.2.1 — Race condition reproduction (30 dk)

`concurrency-playground/synch/RaceyAccount.java` — yukarıdaki naif versiyonu yaz. `synchronized` versiyonu da yan dosyada (`SyncAccount.java`). 100 thread × 1000 deposit testi koş. Race kaybını gör. Defterine "expected vs actual" yaz.

### Task 3.2.2 — Atomic ailesi karşılaştırma (45 dk)

Aynı 100 thread × 1000 deposit senaryosunu 4 farklı sınıfla yaz:

- `RaceyAccount` (long)
- `SyncAccount` (synchronized)
- `AtomicAccount` (`AtomicLong`)
- `AdderAccount` (`LongAdder`)

Her birinin elapsed time'ını ölç (`System.nanoTime()`). Defterine tabloyu yaz:

```
Variant      | Final balance | Elapsed (ms)
-------------|---------------|-------------
Racey        | 87432         | 120
Synchronized | 100000        | 850
AtomicLong   | 100000        | 220
LongAdder    | 100000        | 180
```

Yorumla.

### Task 3.2.3 — `tryWithdraw` CAS-loop (45 dk)

`AtomicAccount.tryWithdraw(long amount)` — yukarıdaki CAS-loop versiyonu. Başlangıç balance 500, 100 thread her biri `tryWithdraw(10)` denesin. Beklenen: 50 başarı, 50 başarısızlık, balance 0.

Test'i yaz (JUnit 5 + AssertJ).

### Task 3.2.4 — `AccountStateHolder` (60 dk)

`record AccountState(long balance, long version, Instant lastUpdated)`.

`AccountStateHolder` class'ı, `AtomicReference<AccountState>` ile state tutar:

- `tryDebit(long amount)` — CAS-loop, lock-free
- `credit(long amount)` — updateAndGet
- `snapshot()` — anlık state döner
- Her başarılı update'te `version` artar

Test:
- Concurrent debit (50 thread × 100 transfer)
- Toplam balance kaybı yok
- Version monotonic artıyor
- Hiçbir negative balance'a düşmüyor (initial 50000, her debit 10 → 50 başarısız debit)

### Task 3.2.5 — `AtomicStampedReference` ile ABA tuzağı (45 dk)

`abrasion/AbaDemo.java`:

1. `AtomicReference<Integer>` ile basit CAS senaryosu
2. ABA tuzağını **manuel** üret (T1 oku → uyu → T2 değiştir-geri al → T1 CAS) — sleep ile yapay senaryo
3. Bu kodun naif AtomicReference ile **başarılı** olduğunu (yanlışlıkla) göster
4. `AtomicStampedReference` ile aynı senaryo: T1 CAS başarısız

Defterine "ABA neden tehlikeli, stamp neden gerekli" yaz.

### Task 3.2.6 — `LongAdder` vs `AtomicLong` benchmark (30 dk)

`metrics/CounterBenchmark.java`:

- 200 thread, her biri 10000 increment
- Bir kez `AtomicLong`, bir kez `LongAdder`
- Elapsed time karşılaştır
- Beklenen: yüksek contention'da `LongAdder` 2-5x hızlı

(JMH gerçek benchmark için Topic 3.11'de. Burada **sanity check**.)

### Task 3.2.7 — `VarHandle` ile field CAS (30 dk)

`varhandle/VarHandleAccount.java`:

```java
public class VarHandleAccount {
    private volatile long balance;
    private static final VarHandle BALANCE_VH;
    static {
        try {
            BALANCE_VH = MethodHandles.lookup()
                .findVarHandle(VarHandleAccount.class, "balance", long.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
    
    public boolean tryDebit(long amount) {
        while (true) {
            long current = (long) BALANCE_VH.getVolatile(this);
            if (current < amount) return false;
            if (BALANCE_VH.compareAndSet(this, current, current - amount)) {
                return true;
            }
        }
    }
}
```

Test'ini yaz. AtomicAccount ile aynı behavior beklenir.

---

## Test yazma rehberi

### Test 3.2.1 — Race condition gösteren test (eğitici, kasten flaky)

```java
@Test
void naiveAccountLosesUpdates() throws Exception {
    var account = new RaceyAccount();
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
    
    // Bu test KASTEN race gösteriyor — flaky olabilir
    assertThat(account.getBalance()).isLessThan((long) threads * perThread);
}
```

### Test 3.2.2 — AtomicAccount kesin doğru

```java
@Test
void atomicAccountIsCorrect() throws Exception {
    var account = new AtomicAccount(0);
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

### Test 3.2.3 — `tryWithdraw` no-overdraft invariant

```java
@Test
void noOverdraftEvenUnderContention() throws Exception {
    var account = new AtomicAccount(500);
    int threads = 100;
    var successes = new AtomicInteger();
    
    var pool = Executors.newFixedThreadPool(threads);
    var latch = new CountDownLatch(threads);
    for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
            if (account.tryWithdraw(10)) successes.incrementAndGet();
            latch.countDown();
        });
    }
    latch.await();
    pool.shutdown();
    
    assertThat(successes.get()).isEqualTo(50);
    assertThat(account.getBalance()).isZero();
}
```

### Test 3.2.4 — `AccountStateHolder` version monotonicity

```java
@Test
void versionMonotonicUnderConcurrentUpdates() throws Exception {
    var holder = new AccountStateHolder(new AccountState(10000, 0, Instant.now()));
    int writers = 10;
    int readers = 10;
    var stop = new AtomicBoolean(false);
    var versionViolations = new AtomicInteger();
    
    var pool = Executors.newFixedThreadPool(writers + readers);
    for (int i = 0; i < writers; i++) {
        pool.submit(() -> {
            while (!stop.get()) holder.credit(1);
        });
    }
    for (int i = 0; i < readers; i++) {
        pool.submit(() -> {
            long lastVersion = 0;
            while (!stop.get()) {
                var snap = holder.snapshot();
                if (snap.version() < lastVersion) versionViolations.incrementAndGet();
                lastVersion = snap.version();
            }
        });
    }
    
    Thread.sleep(2000);
    stop.set(true);
    pool.shutdown();
    pool.awaitTermination(5, TimeUnit.SECONDS);
    
    assertThat(versionViolations.get()).isZero();
}
```

### Test 3.2.5 — `AtomicStampedReference` ABA fix

```java
@Test
void atomicStampedRefDetectsAba() {
    var ref = new AtomicStampedReference<Integer>(100, 0);
    int[] stamp = new int[1];
    
    Integer initial = ref.get(stamp);
    int initialStamp = stamp[0];
    
    // başka thread A → B → A yaptı (simulated)
    ref.compareAndSet(initial, 200, initialStamp, initialStamp + 1);
    ref.compareAndSet(200, 100, initialStamp + 1, initialStamp + 2);
    
    // T1 hâlâ initial gözlemiyle CAS denesin
    boolean success = ref.compareAndSet(100, 999, initialStamp, initialStamp + 1);
    assertThat(success).isFalse();  // stamp uyumsuz, ABA tespit edildi
}
```

---

## Claude-verify prompt

```
Aşağıdaki kod synchronization primitives'i öğrenmek için yazıldı. Lütfen şu 
kriterlere göre değerlendir ve EKSİKLERİ söyle, kod yazma:

1. RaceyAccount + SyncAccount:
   - Race condition gösteren naif versiyon var mı?
   - synchronized'lı versiyon test ile karşılaştırılmış mı?
   - synchronized block veya method tercihi (this leak var mı) doğru mu?

2. AtomicAccount:
   - AtomicLong kullanılmış mı?
   - tryWithdraw için CAS-loop veya getAndUpdate kullanılmış mı?
   - Negative balance'a düşmüyor mu (CAS koşulu doğru mu)?

3. AccountStateHolder:
   - AtomicReference<record> kullanılmış mı?
   - State record immutable mi (record + final field'lar)?
   - updateAndGet veya compareAndSet ile state transition var mı?
   - Lambda pure function mı (side-effect free)?
   - Version monotonic artıyor mu (test ile)?

4. ABA problem:
   - AbaDemo'da naif AtomicReference'ın ABA'ya açık olduğu gösterilmiş mi?
   - AtomicStampedReference ile fix yapılmış mı?
   - Defter notuna "neden ABA tehlikeli" yazılmış mı?

5. LongAdder:
   - Counter senaryosu için LongAdder kullanılmış mı?
   - LongAdder yerine AtomicLong'un yetersiz kaldığı durum açıklanmış mı?
   - check-and-act için LongAdder kullanma anti-pattern'i farkında mı?

6. VarHandle:
   - Field VarHandle ile bağlanmış mı?
   - getVolatile / compareAndSet access modes kullanılmış mı?
   - Field volatile zorunluluğu (VarHandle ile birlikte) anlaşılmış mı?

7. Anti-pattern kontrolü:
   - volatile counter increment YAPMIŞ MI? (yapmamalı)
   - synchronized + this leak var mı?
   - CAS retry lambda'sında side-effect var mı?
   - İki bağımsız atomic field'ı tutarlı snapshot için kullanmaya çalışılmış mı?

8. Banking perspektifi:
   - Balance race condition reproduce edilmiş mi?
   - Tüm 4 versiyon (Racey/Sync/Atomic/Adder) karşılaştırılmış mı?
   - no-overdraft invariant test edilmiş mi?

Her madde için PASS / FAIL / EKSIK. Kod yazma, sadece eksiklikleri söyle.
```

---

## Tamamlama kriterleri

- [ ] `synchronized` instance method / static method / block farkını anlatabiliyorum
- [ ] Reentrant özelliği örnekle açıklayabiliyorum
- [ ] `volatile`'in 3 garantisi ve 1 NON-garantisi (atomicity yok) ezberimde
- [ ] `AtomicLong` ile race-free counter yazdım
- [ ] CAS-loop pattern'ini açıklayabiliyorum ve yazabiliyorum
- [ ] ABA problemini örnekle anlatabiliyorum
- [ ] `AtomicStampedReference` ile ABA fix gösterdim
- [ ] `LongAdder` ile `AtomicLong` arasında benchmark çalıştırdım
- [ ] `AtomicReference<record>` pattern'i ile state holder yazdım
- [ ] `VarHandle` ile field CAS yaptım
- [ ] Biased locking'in JDK 15+ kaldırıldığını biliyorum, sebebini açıklayabiliyorum
- [ ] `LockSupport.park/unpark`'ın permit semantik özelliğini biliyorum
- [ ] Banking karar matrisini ezberledim (hangi senaryoda hangi primitif)

Hepsi onaylı → Topic 3.3'e geç → [03-locks/](../03-locks/index.md)

---

## Defter notları

1. "synchronized'ın memory semantics: enter = ____, exit = ____."
2. "`volatile` 3 garantisi: ____, ____, ____. Bir non-garantisi: ____."
3. "CAS donanım enstrüksiyonu: ____. Pseudo-code'u: ____."
4. "ABA problemi nedir, neden tehlikeli: ____. Çözüm: ____."
5. "`LongAdder` neden `AtomicLong`'tan high contention'da hızlıdır: ____."
6. "`updateAndGet` lambda'sında side-effect olmamasının sebebi: ____."
7. "Biased locking JDK ____ kaldırıldı. Sebep: ____."
8. "`VarHandle` neden `Unsafe`'ten daha iyi: ____, ____, ____."
9. "`AtomicReference<State>` pattern'inin tutarlı snapshot avantajı: ____."
10. "Banking'de balance update için 100 thread eşzamanlı çağırırsam, `synchronized`/`AtomicLong`/`LongAdder` karşılaştırması (final değer + hız): ____."
