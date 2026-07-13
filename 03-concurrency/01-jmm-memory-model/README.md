# Topic 3.1 — Java Memory Model (JMM) & Happens-Before

## Hedef

Java Memory Model'i (JMM) banking-grade seviyede anlamak. "Bir thread'in yazdığı değeri başka bir thread ne zaman ve hangi koşullarda görür?" sorusuna kesin cevap verebilmek. `happens-before` ilişkisinin **yedi kuralını** ezbersiz, mantıksal olarak çıkarabilmek. `volatile`'in **ne sağladığını** ve özellikle **ne sağlamadığını** ayırt etmek. Cache coherency, instruction reordering ve memory barrier'ları concrete olarak görmek. Banking domain'inde gerçek bir visibility bug'ını reproduce edip çözebilmek.

## Süre

Okuma: 3 saat • Mini task'ler: 3 saat • Test: 1 saat • Toplam: ~7 saat

Bu Phase 3'ün en yoğun teorik topic'idir. Acele etme. Anlamadığın paragrafı bir kez daha oku, ardından koda dök.

## Önbilgi

- Phase 1 ve Phase 2 tamamlandı
- Java threading'in temel sözdizimi (`Thread`, `Runnable`, `start`, `join`) tanıdık
- "Race condition" terimi en azından duyulmuş (detay burada)
- Donanım perspektifi: CPU, L1/L2/L3 cache, RAM hiyerarşisi hakkında genel fikir

---

## Kavramlar

### 1. Memory Model neden var — donanım gerçekliği

Modern bir x86_64 CPU'nun **bir core'unda** çalışan kodun, **başka bir core**'da nasıl görüleceği konusunda **belirsizlik vardır**. Bunun sebebi donanımın "saf bir RAM" modeli kullanmamasıdır:

```
┌────────────────────────────────────────────────┐
│  CPU Core 0                  CPU Core 1        │
│  ┌─────────┐                 ┌─────────┐       │
│  │  Reg    │                 │  Reg    │       │  ← register'lar
│  ├─────────┤                 ├─────────┤       │
│  │  Store  │                 │  Store  │       │  ← store buffer (yazıların biriktiği yer)
│  │  Buffer │                 │  Buffer │       │
│  ├─────────┤                 ├─────────┤       │
│  │  L1 D$  │                 │  L1 D$  │       │  ← L1 data cache (per-core)
│  ├─────────┤                 ├─────────┤       │
│  │  L2     │                 │  L2     │       │  ← L2 cache (genelde per-core)
│  └────┬────┘                 └────┬────┘       │
│       │   ┌──────────────────┐    │            │
│       └───┤      L3 Cache    ├────┘            │  ← L3 shared
│           └─────────┬────────┘                 │
│                     │                          │
│              ┌──────┴──────┐                   │
│              │     RAM     │                   │
│              └─────────────┘                   │
└────────────────────────────────────────────────┘
```

Bir thread bir field'a yazdığında:
1. Önce **register'a** yazar.
2. Sonra **store buffer**'a (asenkron, henüz cache'e bile yazılmamış).
3. Store buffer drain edildiğinde **L1 cache**'e yazar.
4. Cache coherency protokolü (MESI) ile **diğer core'ların L1'i invalidate** edilir.
5. Eventually RAM'e yazılır.

**Sorun:** Diğer thread aynı anda kendi register/cache'inden okuyorsa, **eski değeri görür**. Bu **görünmezlik (invisibility)** veya **stale read** problemidir.

Banking örneği: Müşteri "transferim hangi durumda?" diye sorgular, başka bir thread az önce status'u `COMPLETED` yapmış ama bu bilgi henüz sorgulayan thread'in cache'ine ulaşmamıştır. Sorgulayan thread `PENDING` görür, müşteri ekrana "İşleminiz beklemede" yazar. Müşteri panik yapar, çağrı merkezi arar.

JMM, dilin **garantilerini formalize eder**: hangi koşullarda Thread B, Thread A'nın yazdığı değeri **görmek zorundadır**.

---

### 2. Memory Model'in iki yüzü — visibility + ordering

JMM iki ana problemi çözer:

**a) Visibility (görünürlük):** Thread A'nın yazdığı değer Thread B tarafından ne zaman görülebilir?

```java
class BalanceHolder {
    private long balance = 0;
    
    public void set(long v) { balance = v; }       // Thread A
    public long get() { return balance; }          // Thread B
}
```

Hiçbir senkronizasyon yoksa, Thread A'nın `set(100)` çağrısı sonrası Thread B'nin `get()` çağrısı **0 döndürebilir** — sonsuza kadar bile. Bu spec'tedir; donanım hatası değil.

**b) Ordering (sıralama):** İki thread'in **gözlediği** olay sırası ne kadar tutarlı?

```java
class FlagAndValue {
    int value = 0;
    boolean ready = false;
    
    void writer() {
        value = 42;       // (1)
        ready = true;     // (2)
    }
    
    void reader() {
        if (ready) {                              // (3)
            System.out.println(value);            // (4)
        }
    }
}
```

Naif beklenti: `reader` `ready=true` görürse `value=42` görür. **JMM bu garantiyi vermez** (yeterli senkronizasyon yoksa). Çünkü:
- **Compiler reordering**: derleyici (1) ve (2)'yi swap'layabilir
- **CPU reordering**: out-of-order execution
- **Cache reordering**: store buffer'da farklı sırada flush

`reader` `ready=true` görüp `value=0` görebilir. Bu öğrenmesi en zor concurrency tuzaklarından biri.

---

### 3. Happens-Before — JMM'in çekirdek kavramı

**Happens-before** iki memory operasyonu arasındaki **ilişkidir**. "A happens-before B" şu anlama gelir:

> A operasyonunun **etkileri** (yazdıkları) B operasyonuna **görünür**, ve A, B'den **önce gerçekleşmiş gibi** muamele edilir.

Bu salt zamansal "önce-sonra" değil — **görünürlük + sıra garantisi**.

**Happens-before yoksa, görme garantisi yoktur.**

#### 3.1 Program order rule

Aynı thread içindeki ifadeler, **program sırasına göre** happens-before ilişkisindedir.

```java
void method() {
    int a = 1;   // A
    int b = 2;   // B
    int c = a + b;  // C
}
```

A happens-before B, B happens-before C, A happens-before C (geçişli). Bu **tek thread için** her zaman geçerli. Compiler reorder etse bile, **gözlenebilir sonuç değişmez** (single-thread for the win).

**Ama bu kural sadece kendi thread'i için garanti.** Başka thread bunu gözlüyorsa, ek senkronizasyon olmadan reorder'lar görünür.

#### 3.2 Monitor lock rule (synchronized)

`synchronized` block'unun **unlock**'u (kapanışı), **aynı monitor'a** sonradan giren thread'in **lock**'una (girişine) happens-before'dur.

```java
class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;        // (Thread A burada)
    }
    
    public synchronized int read() {
        return count;   // (Thread B burada)
    }
}
```

Thread A `increment()` bitirdiğinde monitor'ı release eder. Thread B `read()` ile aynı monitor'ı acquire eder. **Release happens-before acquire** → Thread A'nın yazdığı her şey Thread B'ye görünür. `count` doğru okunur.

#### 3.3 Volatile rule

Bir `volatile` field'a **yazma**, aynı field'ı **sonradan okuyan** her thread'e happens-before'dur.

```java
class FlagAndValue {
    int value = 0;
    volatile boolean ready = false;
    
    void writer() {
        value = 42;        // (1)
        ready = true;      // (2) volatile write
    }
    
    void reader() {
        if (ready) {       // (3) volatile read
            assert value == 42;  // ✓ artık garanti
        }
    }
}
```

Volatile write `(2)` happens-before volatile read `(3)`. Volatile write'ın **öncesi** her şey (program order ile) `(2)`'ye happens-before. Reader `(3)`'ten sonra her şey `(2)`'yi takip eder. Yani `value = 42` `(1)`, reader'a görünür.

Bu **piggyback** etkisi: volatile field, kendisinden önceki **tüm normal yazıları** taşır.

#### 3.4 Thread start rule

`thread.start()` çağrısı, başlatılan thread'in **ilk aksiyonuna** happens-before'dur.

```java
int shared = 0;

Thread t = new Thread(() -> {
    System.out.println(shared);  // garantili 42
});

shared = 42;
t.start();  // start happens-before thread'in run() çağrısı
```

#### 3.5 Thread termination rule (join)

Bir thread'in `run()` metodunun bitişi, `thread.join()` döndüğüne happens-before'dur.

```java
final int[] result = new int[1];
Thread t = new Thread(() -> {
    result[0] = compute();
});
t.start();
t.join();
System.out.println(result[0]);  // garantili compute()'in sonucu
```

#### 3.6 Interruption rule

`thread.interrupt()` çağrısı, `thread`'in interrupt'ı **fark ettiği** noktaya happens-before'dur (örn. `Thread.interrupted()` true dönmesi, `InterruptedException` fırlatması).

#### 3.7 Finalizer rule (kullanılmaz — deprecated)

Object'in constructor'ının bitişi, finalizer'ının başlangıcına happens-before'dur. Modern Java'da `finalize()` deprecated; **bu kuralı pratikte göz ardı et**.

#### 3.8 Transitivity

A happens-before B, B happens-before C ise A happens-before C.

```
Thread A: balance = 100  →  volatile flag = true
                                                     \
                                                      happens-before transitive
                                                     /
Thread B:                volatile flag == true → read balance == 100
```

---

### 4. `volatile` — visibility + ordering ama atomicity DEĞİL

`volatile` keyword'ü bir field'a koyduğunda:

**a) Görünürlük garantisi:** Yazılan değer **anında** tüm thread'lere görünür (cache flush + invalidation).

**b) Ordering garantisi:** Volatile write'tan önceki yazılar volatile write'a; volatile read'den sonraki okumalar volatile read'a göre **reorder edilmez**.

**c) Atomicity garantisi YOK:** `volatile int counter; counter++;` atomic **değildir**. `counter++` üç adımdır: read, increment, write. Aralarında başka thread iş yapabilir.

```java
class WrongCounter {
    private volatile int count = 0;
    
    public void increment() {
        count++;   // ❌ race condition!
    }
}
```

İki thread aynı anda `count = 5` okur, ikisi de `6` yazar. Bir artış kaybolur.

**Doğru kullanım volatile için:** Tek bir thread yazar, çok thread okur. Veya **bayrak**.

```java
class GracefulShutdown {
    private volatile boolean shutdown = false;
    
    // controller thread
    public void requestShutdown() {
        shutdown = true;
    }
    
    // worker thread
    public void run() {
        while (!shutdown) {
            processNextTask();
        }
    }
}
```

`shutdown` `volatile` olmasaydı, worker thread'in `while (!shutdown)` döngüsü kendi cache'inde `false` okumaya devam eder, **sonsuza kadar dönerdi**. Volatile olduğu için controller'ın `true` ataması worker'a görünür, döngü sonlanır.

**Banking örneği — yanlış kullanım:**

```java
class AccountBalanceHolder {
    private volatile long balance = 0;
    
    public void deposit(long amount) {
        balance += amount;  // ❌ ATOMIC DEĞİL — race condition
    }
}
```

Bu kodu 1000 thread eş zamanlı çağırdığında balance **kaybolan transfer'ler** içerir. `volatile` yetmez — `AtomicLong` veya `synchronized` veya `Lock` gerek.

**Volatile'in tek-tek tipler için davranışı:**

- `boolean`, `int`, `long`, `double`, reference: volatile ile **read ve write atomic**
- `long` ve `double` non-volatile **olmayabilir atomic** (32-bit JVM'de iki 32-bit yarımda yazılır) — bu nadiren karşılaşılır ama spec'te yer alır
- `volatile long` ve `volatile double` her zaman atomic read/write

---

### 5. Memory barrier'lar — donanım perspektifi

`happens-before` JVM seviyesinde soyutlanmıştır. Altında **memory barrier (fence)** denen CPU enstrüksiyonları yatar.

JVM 4 tip barrier kullanır (JSR-133 cookbook):

| Barrier | İşlevi |
|---|---|
| `LoadLoad` | Sonraki load'lar önceki load'lardan **sonra** yapılır |
| `StoreStore` | Sonraki store'lar önceki store'lardan **sonra** yapılır |
| `LoadStore` | Sonraki store'lar önceki load'lardan **sonra** yapılır |
| `StoreLoad` | Sonraki load'lar önceki store'lardan **sonra** yapılır (en pahalı) |

**Volatile write için:** Compiler `StoreStore` (öncesi) + `StoreLoad` (sonrası) barrier'ları emit eder.

**Volatile read için:** `LoadLoad` + `LoadStore` (sonrası) barrier'lar emit edilir.

**`synchronized` enter için:** Lock acquire = `Acquire fence` (LoadLoad + LoadStore benzeri).

**`synchronized` exit için:** `Release fence` (StoreStore + LoadStore benzeri).

x86_64 mimarisinde bu barrier'ların çoğu **zaten implicit** (TSO — Total Store Ordering); sadece `StoreLoad` explicit (`MFENCE` veya `lock; addl $0, (%rsp)` enstrüksiyonu). ARM ve POWER mimarileri daha gevşek; barrier'lar daha pahalı.

Bu yüzden **bir Java programı x86'da çalışırken bug göstermezken**, ARM (örn. Apple Silicon production servers) veya AWS Graviton'da bug **patlayabilir**. JMM ile yazarsan platform-bağımsız doğru kod yazarsın.

---

### 6. Cache coherency — MESI protokolü kısaca

Modern multi-core CPU'lar cache satırlarının (cache line, tipik 64 byte) state'ini takip eder. **MESI** dört state tanımlar:

- **M**odified: Bu core üzerinde değiştirilmiş, henüz RAM'e yazılmamış. Sadece bu core'da.
- **E**xclusive: Sadece bu core'un cache'inde, değişmemiş.
- **S**hared: Birden fazla core'un cache'inde, değişmemiş.
- **I**nvalid: Geçersiz, kullanılamaz.

Core 0 bir cache line'a yazdığında, MESI protokolü diğer core'lardaki kopyaları **I (Invalid)** durumuna getirir. Diğer core okuduğunda cache miss alır, Core 0'dan günceli ister.

**False sharing problemi:** İki **farklı** değişken aynı cache line'da yer alıyorsa, biri değiştiğinde diğerinin core'unda da invalidation tetiklenir. Performance kıyısı yer.

```java
class FalseSharing {
    long a;       // cache line 1, offset 0
    long b;       // cache line 1, offset 8
}
```

Thread A `a`'ya, Thread B `b`'ye yazıyorsa, aynı cache line üzerinde yarış çıkar.

**Çözüm:** `@Contended` annotation (JDK internal; production'da kasıtlı padding) veya manuel padding.

```java
@jdk.internal.vm.annotation.Contended
class IsolatedField {
    long value;
}
```

Banking örneği: `LongAdder` (Phase 3 Topic 2'de) `Striped64` ile false sharing'i önlemek için padding yapar.

---

### 7. Instruction reordering — derleyici ve CPU iş birliği

Java spec'i derleyici ve CPU'ya **single-thread davranışını koruduğu sürece** reorder etme **izni** verir. "As if serial" ilkesi.

Örnek:

```java
int a = readField1();
int b = readField2();
int c = a + b;
```

Derleyici `readField2()`'yi `readField1()`'den **önce** çağırabilir. Tek thread için sonuç aynı.

Multi-thread'de:

```java
class WriteOrdering {
    int x = 0;
    int y = 0;
    
    void writer() {       // Thread A
        x = 1;
        y = 2;
    }
    
    void reader() {       // Thread B
        int b = y;        // gözlem
        int a = x;
        if (b == 2 && a == 0) {
            // tartışmalı: writer'ın bir kısmı görüldü?
        }
    }
}
```

Reader **`b == 2` görüp `a == 0` görebilir mi?** Evet, yeterli senkronizasyon olmadan. Çünkü writer'ın `x = 1` ve `y = 2` atamaları reorder edilebilir; veya reader'ın `b = y` ve `a = x` okumaları reorder edilebilir.

Bu hata banking'de gerçekleşirse: Bir thread `account.balance = newBalance; account.lastUpdated = now;` yazar. Başka thread `if (account.lastUpdated > X) check(account.balance);` yapar — eski balance okuyabilir.

**Çözüm:** Volatile, synchronized, veya lock. Reorder'ları kısıtlar.

---

### 8. Final field semantics — safe publication

`final` field için JMM özel garanti verir:

> Bir object'in constructor'ı bittiğinde, `final` field'ların değerleri **herhangi bir thread için** (object'in referansını gördüğü andan itibaren) **görünür**dür.

```java
class ImmutableAccount {
    private final long id;
    private final long openingBalance;
    
    public ImmutableAccount(long id, long openingBalance) {
        this.id = id;
        this.openingBalance = openingBalance;
    }
}
```

Bir thread bu object'i yarattıktan sonra referansını başka thread'e geçirirse (örn. via volatile field, BlockingQueue, ConcurrentHashMap), o thread `id` ve `openingBalance`'ı **doğru** okur.

**Ama:** Constructor içinde `this` referansını dışarı sızdırırsan (örn. `someStaticMap.put(id, this)` constructor içinde), bu garanti **bozulur**. Yarı-construct edilmiş object başka thread'e açılır.

**Banking pratiği:** `Money`, `AccountId`, `Transfer` gibi value object'lerini `record` yap. `record`'lar tüm field'ları implicit `final`. JMM safe publication garantisi otomatik.

---

### 9. Double-checked locking ve volatile

Klasik DCL pattern'i:

```java
class FxRateService {
    private static FxRateService instance;
    
    public static FxRateService getInstance() {
        if (instance == null) {                      // 1st check
            synchronized (FxRateService.class) {
                if (instance == null) {              // 2nd check
                    instance = new FxRateService();  // ❌ broken!
                }
            }
        }
        return instance;
    }
}
```

**Sorun:** `instance = new FxRateService()` üç adımlıdır:
1. Memory allocate
2. Constructor çalıştır
3. `instance`'a referans ata

Compiler 1, 3, 2 sıralayabilir. Başka thread `instance != null` görüp **yarı-construct edilmiş** object'i kullanır.

**Düzeltme:** `volatile`.

```java
private static volatile FxRateService instance;
```

Volatile, `instance` write'ından önceki tüm yazıları (constructor'ı tamamlayan tüm field assignment'ları) görünür yapar.

**Banking notu:** Modern Java'da singleton için ya `enum` (en güvenli) ya da Spring bean (managed by container) kullan. DCL yine bilinmesi gereken klasik tuzak.

---

### 10. Banking visibility bug — concrete reproduction

Şu kod **deterministically bug üretir**:

```java
package com.mavibank.banking.bug;

public class BalanceCheckBug {
    
    private long balance = 1000;       // ❌ volatile yok
    private boolean overdraftEnabled = false;  // ❌ volatile yok
    
    public void enableOverdraft() {
        balance = -500;
        overdraftEnabled = true;
    }
    
    public boolean canWithdraw(long amount) {
        if (overdraftEnabled) {
            return true;
        }
        return balance >= amount;
    }
    
    public static void main(String[] args) throws InterruptedException {
        var holder = new BalanceCheckBug();
        
        Thread reader = new Thread(() -> {
            int iterations = 0;
            while (true) {
                iterations++;
                if (holder.canWithdraw(2000)) {
                    System.out.println("Iteration " + iterations + 
                        ": canWithdraw=true, balance=" + holder.balance + 
                        ", overdraft=" + holder.overdraftEnabled);
                    return;
                }
            }
        });
        reader.start();
        
        Thread.sleep(500);  // reader döngüye girsin
        holder.enableOverdraft();
        
        reader.join(5000);
        if (reader.isAlive()) {
            System.out.println("Reader hâlâ döngüde — visibility bug!");
            reader.interrupt();
        }
    }
}
```

x86'da bu çoğu zaman çalışır (TSO sayesinde). ARM mimarilerinde (Apple Silicon, AWS Graviton) reader **sonsuza kadar takılabilir**.

JIT optimization sonrası daha kötü: HotSpot, `while (!shutdown)` döngüsünde `shutdown`'ın değişmeyeceğini varsayıp **hoist** edebilir (loop dışına alır). Sonra döngü `while (true)` olur, hiçbir zaman çıkmaz.

**Düzeltme:**

```java
private volatile long balance = 1000;
private volatile boolean overdraftEnabled = false;
```

Veya hepsini bir `synchronized` block içine al.

Daha temiz banking yaklaşımı (`record`-based immutable state, `AtomicReference`):

```java
public class BalancePolicy {
    private final AtomicReference<State> state = 
        new AtomicReference<>(new State(1000, false));
    
    private record State(long balance, boolean overdraftEnabled) {}
    
    public void enableOverdraft() {
        state.set(new State(-500, true));
    }
    
    public boolean canWithdraw(long amount) {
        var s = state.get();
        return s.overdraftEnabled || s.balance >= amount;
    }
}
```

`AtomicReference` volatile-gibi davranır + immutable State sayesinde **tutarlı snapshot** alırsın (balance ve overdraftEnabled aynı anda görülür).

---

### 11. Anti-pattern'ler

**Anti-pattern 1: "İki volatile field bağımlı tutmak"**

```java
volatile long balance;
volatile long lastUpdated;
```

Bu **iki bağımsız volatile**'dır. Reader, balance'ın güncelini görürken lastUpdated'ı eski görebilir. **Tutarlılık garantisi yok.**

Çözüm: ikisini bir `record State(long balance, long lastUpdated)` içinde paketle, `AtomicReference<State>` ile değiştir.

**Anti-pattern 2: "Volatile counter ile increment"**

```java
volatile int count;
// ...
count++;  // ❌
```

Read-modify-write atomic değil. `AtomicInteger` veya `synchronized` kullan.

**Anti-pattern 3: "Constructor'da `this` sızdırma"**

```java
class Account {
    public Account(EventBus bus) {
        bus.register(this);  // ❌ object yarım, ama referans dışarıda
        this.balance = 0;    // race
    }
}
```

Doğrusu: factory method ile **fully constructed** halde register et.

```java
public static Account create(EventBus bus) {
    var a = new Account();
    bus.register(a);
    return a;
}
```

**Anti-pattern 4: "synchronized'i sadece yazan tarafa koymak"**

```java
class WrongBalance {
    private long balance;
    
    public synchronized void deposit(long a) { balance += a; }
    public long getBalance() { return balance; }   // ❌ visibility garantisi yok
}
```

Reader synchronized değil → balance eski görülebilir. Read tarafına da `synchronized` veya field'a `volatile` koy.

**Anti-pattern 5: "Lazy initialization without proper publication"**

Yukarıda 9. maddedeki DCL kötü pattern'i. `volatile` veya `enum singleton`.

---

### 12. JMM ve modern Java (records, sealed, virtual threads)

**`record` ve JMM:** Record'lar all-final by design. Safe publication otomatik. Banking için ideal: `Money`, `Transfer`, `JournalEntry` hep record olmalı.

**Sealed classes:** Concurrency garantisi getirmiyor; sadece type system. JMM açısından regular class gibi.

**Virtual threads (Topic 3.7):** JMM aynı kalır. Virtual thread'ler de happens-before kurallarına uyar. Volatile, synchronized, atomic — hepsi virtual thread'de aynı semantiğe sahip.

---

## Önemli olabilecek araştırma kaynakları (kuralın gereği: senin için keyword, kendi bul)

- "JSR-133 Java Memory Model and Thread Specification"
- "JSR-133 Cookbook for Compiler Writers" Doug Lea
- "Java Concurrency in Practice" Brian Goetz (kitap, özellikle Chapter 3 ve 16)
- "Java Memory Model Pragmatics" Aleksey Shipilev (article)
- Shipilev — "Close encounters of the Java Memory Model kind" (talk)
- "The Java Language Specification" Chapter 17 (resmi spec)
- jcstress (Java Concurrency Stress Tests) — OpenJDK projesi
- "Memory Barriers: a Hardware View for Software Hackers" Paul McKenney
- "What every systems programmer should know about concurrency" Matt Kline
- Aleksey Shipilev blog (shipilev.net) — JMM, volatile, fence örnekleri

---

## Mini task'ler

Bunları sırayla yap. `~/projects/core-banking/` altında `concurrency-playground` adıyla yan modül oluştur (Maven submodule değil, sadece ayrı bir package). Bu klasör atılabilir kod için.

### Task 3.1.1 — Visibility bug reproduction (30 dk)

`concurrency-playground/jmm/VisibilityBug.java` oluştur:

- Bir `boolean ready` field'ı (volatile **YOK**)
- Worker thread `while (!ready) {}` döngüsünde
- Main thread 1 saniye uyuyup `ready = true` yapar
- Worker thread 5 saniye içinde döngüden çıkamazsa "BUG REPRODUCED" yazdır

```java
package com.mavibank.banking.playground.jmm;

public class VisibilityBug {
    private static boolean ready = false;
    private static int value = 0;
    
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int spins = 0;
            while (!ready) {
                spins++;
                if (spins == Integer.MAX_VALUE) {
                    spins = 0;  // overflow protection
                }
            }
            System.out.println("Worker saw ready=true, value=" + value);
        });
        worker.start();
        
        Thread.sleep(1000);
        value = 42;
        ready = true;
        
        worker.join(5000);
        if (worker.isAlive()) {
            System.out.println("BUG REPRODUCED: worker stuck despite ready=true");
            worker.interrupt();
        }
    }
}
```

Çalıştır:
```bash
java -Xcomp com.mavibank.banking.playground.jmm.VisibilityBug
```

`-Xcomp` JIT'i full force çalıştırır, hoisting'i hızlandırır. Bug deterministically tetiklenir.

### Task 3.1.2 — Volatile ile fix (10 dk)

`VisibilityBugFixed.java` — `ready`'i `volatile boolean` yap. Tekrar çalıştır. Worker hemen çıkmalı. Sonucu **defterine yaz**.

### Task 3.1.3 — Bağımlı field tutarsızlığı (45 dk)

`InconsistentSnapshot.java`:

- İki volatile field: `balance` (long), `lastUpdated` (long)
- Writer thread: `balance = X; lastUpdated = now;` döngüde
- Reader thread: snapshot alır, `(balance, lastUpdated)` print eder
- Belirli iterasyon sonra (1 milyon) durdur
- Beklenen: bazen reader, balance yeni ama lastUpdated eski (veya tersi) yakalar

Ardından `AtomicReference<State>` ile düzelt. İki versiyonu da çalıştır, output'u defterine kopyala.

### Task 3.1.4 — DCL singleton (30 dk)

`DclSingleton.java`:

1. `volatile` **olmadan** klasik double-checked locking
2. Singleton class'ın constructor'ında 100 ms sleep + `loaded = true` field set
3. 50 thread aynı anda `getInstance().isLoaded()` çağırır
4. Yarı-construct gözlemleyebilir misin? (modern JVM'de zor, ama spec'e göre mümkün)
5. `volatile` ekleyerek düzelt

Bu task'ta bug'ı gözlemlemek zor olabilir; spec'e güven. Defterine "neden bu hata teorik olarak mümkün?" yaz.

### Task 3.1.5 — Banking `AccountSnapshot` (45 dk)

`account/AccountSnapshot.java`:

```java
public record AccountSnapshot(
    long balance,
    long version,
    Instant lastUpdated
) {}
```

`AccountSnapshotHolder` class'ı:

```java
public class AccountSnapshotHolder {
    private final AtomicReference<AccountSnapshot> current;
    
    public AccountSnapshotHolder(AccountSnapshot initial) {
        this.current = new AtomicReference<>(initial);
    }
    
    public AccountSnapshot read() { return current.get(); }
    
    public void update(long newBalance) {
        current.updateAndGet(s -> new AccountSnapshot(
            newBalance,
            s.version() + 1,
            Instant.now()
        ));
    }
}
```

10 writer + 10 reader thread başlat, 5 saniye çalıştır. Reader'ların gördüğü `version` ve `balance` her zaman **aynı snapshot'tan** gelmeli (tutarlı).

### Task 3.1.6 — Happens-before akış diyagramı (defter notu, 15 dk)

Şu senaryonun **happens-before zincirini** defterine çiz:

> Thread A: `account.balance = 100; transfer.completed = true;` (volatile)
> Thread B: `if (transfer.completed) read(account.balance);`

Zincir: `A: balance=100` -program order→ `A: completed=true` -volatile rule→ `B: read completed` -program order→ `B: read balance`. Sonuç: A'nın balance yazısı B'ye görünür.

---

## Test yazma rehberi

JMM bug'larını **deterministically test etmek zor**. Stress test yaklaşımı gerekir.

### Test 3.1.1 — AccountSnapshotHolder consistency test

```java
@Test
void readersShouldAlwaysSeeConsistentSnapshot() throws Exception {
    var holder = new AccountSnapshotHolder(
        new AccountSnapshot(0, 0, Instant.now())
    );
    int writerCount = 4;
    int readerCount = 4;
    int durationSec = 3;
    
    var stop = new AtomicBoolean(false);
    var violations = new AtomicInteger(0);
    var executor = Executors.newFixedThreadPool(writerCount + readerCount);
    
    for (int i = 0; i < writerCount; i++) {
        executor.submit(() -> {
            long counter = 0;
            while (!stop.get()) {
                holder.update(counter++);
            }
        });
    }
    
    for (int i = 0; i < readerCount; i++) {
        executor.submit(() -> {
            while (!stop.get()) {
                var snap1 = holder.read();
                var snap2 = holder.read();
                // tek snapshot tutarlı, ama iki ardışık okumada version monoton artmalı
                if (snap2.version() < snap1.version()) {
                    violations.incrementAndGet();
                }
            }
        });
    }
    
    Thread.sleep(durationSec * 1000);
    stop.set(true);
    executor.shutdown();
    assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    assertThat(violations.get()).isZero();
}
```

### Test 3.1.2 — VisibilityBug deterministically tetiklenmeli mi?

Test yazmak zor — JIT, OS scheduling, hardware bağımlı. **jcstress** kullanmadan deterministically test edemezsin. Bu yüzden:

```java
@Test
@Timeout(value = 10, unit = TimeUnit.SECONDS)
void volatileFlagShouldBeVisible() throws Exception {
    var flag = new VolatileFlag();
    var done = new CountDownLatch(1);
    
    Thread worker = new Thread(() -> {
        while (!flag.isReady()) {
            // spin
        }
        done.countDown();
    });
    worker.start();
    
    Thread.sleep(50);
    flag.setReady();
    
    assertThat(done.await(5, TimeUnit.SECONDS)).isTrue();
}
```

Volatile olmadan bu test bazen **fail** edebilir; bunu kasten yaz ve bul.

### Test 3.1.3 — `record` immutability ve safe publication

```java
@Test
void recordIsSafelyPublished() throws Exception {
    var publisher = new AtomicReference<Money>();
    var reader = new Thread(() -> {
        Money m;
        while ((m = publisher.get()) == null) {
            // spin
        }
        assertThat(m.amount()).isEqualByComparingTo("100.00");
        assertThat(m.currency().getCurrencyCode()).isEqualTo("TRY");
    });
    reader.start();
    
    Thread.sleep(10);
    publisher.set(Money.of("100.00", "TRY"));
    reader.join();
}
```

Record + AtomicReference yeterli. Volatile field veya synchronized'a ihtiyaç yok.

---

## Claude-verify prompt

```
Aşağıda Java Memory Model'i kavramaya yönelik kodum var. Lütfen şu kriterlere 
göre değerlendir ve EKSİKLERİ söyle, kod yazma:

1. VisibilityBug reproduction:
   - `ready` field'ı volatile DEĞİL mi (kasten)?
   - Worker thread'in döngüsü hoisting'e açık mı?
   - `-Xcomp` veya yeterli iterasyonla JIT'in optimize edebileceği yapı kuruldu mu?
   - Bug reproduce eden ve volatile ile fix eden iki ayrı dosya var mı?

2. InconsistentSnapshot:
   - İki bağımsız volatile field kullanılmış mı (kasten yanlış)?
   - Reader'ın iki field'ı atomic olmayan biçimde okuduğu test edilmiş mi?
   - AtomicReference<Record> ile düzeltme yapılmış mı?
   - Record kullanılmış mı (final field semantics)?

3. AccountSnapshotHolder:
   - AtomicReference + immutable record kullanılmış mı?
   - updateAndGet veya compareAndSet ile state transition yapılmış mı?
   - Reader-writer stres testi yazılmış mı?
   - Test'te violation counter (version monotonicity) kontrolü var mı?

4. Happens-before anlayışı (kavram testi — defterine yazılmış mı kontrol):
   - Volatile write happens-before volatile read kuralı açıklanabilir mi?
   - synchronized release/acquire happens-before açıklanabilir mi?
   - Thread.start happens-before run() kuralı var mı?
   - Transitivity örneği verilebilir mi?

5. Anti-pattern kontrolü:
   - volatile counter increment ile race condition gösterilmiş mi?
   - synchronized sadece yazana koyma anti-pattern'i fark edilmiş mi?
   - Constructor'da this leak örneği görülmüş mü?

6. Banking perspektifi:
   - Balance check + balance write race senaryosu reproduce edilmiş mi?
   - Volatile vs AtomicReference tradeoff'u anlaşılmış mı?

Her madde için PASS / FAIL / EKSIK işaretle. Kod yazma, sadece eksiklikleri söyle.
```

---

## Tamamlama kriterleri (kendine sor)

- [ ] Happens-before'un yedi kuralını sırayla sayabiliyorum
- [ ] `volatile`'in **ne sağladığını** ve **ne sağlamadığını** ayrı ayrı söyleyebiliyorum
- [ ] `count++`'ın volatile ile neden hâlâ race olduğunu açıklayabiliyorum
- [ ] Memory barrier'ların 4 tipini (LoadLoad/StoreStore/LoadStore/StoreLoad) tanıyorum
- [ ] Final field semantics ve safe publication kavramını açıklayabiliyorum
- [ ] DCL pattern'inde volatile'in neden zorunlu olduğunu biliyorum
- [ ] VisibilityBug.java'mı çalıştırdım, en az bir kez bug'ı gözlemledim
- [ ] AccountSnapshotHolder'ımı yazdım, stress test'i geçiyor
- [ ] Cache coherency MESI protokolünü en azından bir cümleyle anlatabiliyorum
- [ ] Compiler/CPU reordering'in pratik etkisini örnekle anlatabiliyorum

Hepsi onaylı → Topic 3.2'ye geç → [02-synchronization-primitives/](../02-synchronization-primitives/README.md)

---

## Defter notları (kendi cümlelerinle doldur)

1. "Java Memory Model'in çözdüğü iki ana problem ____ ve ____. Birincisi ____, ikincisi ____."
2. "Happens-before ilişkisinin pratik anlamı şu: ____. Bu olmadan ____ olur."
3. "Volatile write'ın yaptıkları: ____. Yapmadıkları: ____."
4. "synchronized release happens-before synchronized acquire — bu bana ____ garantisi verir."
5. "DCL pattern'inde volatile olmadan ____ problemi olur çünkü ____."
6. "Memory barrier'ların 4 tipinden en pahalısı ____ çünkü ____."
7. "Cache coherency MESI'nin temel fikri ____. False sharing ____ demek."
8. "Final field semantics bana ____ garantisi verir, bu yüzden record kullanmak ____."
9. "İki bağımsız volatile field tutmanın problemi ____. Çözüm: ____."
10. "Banking'de visibility bug'ının iş etkisi: ____ ve ____."
