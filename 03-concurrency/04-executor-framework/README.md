# Topic 3.4 — Executor Framework: ThreadPoolExecutor, Queues, ForkJoinPool

## Hedef

Java'nın **`Executor` ailesini** banking-grade kavramak. Thread lifecycle, interrupt semantiği, `ExecutorService` API'si, `ThreadPoolExecutor`'ün **7 parametresi** (core/max/keep-alive/queue/threadFactory/rejection/timeUnit), bounded vs unbounded queue tuzakları (özellikle `Executors.newFixedThreadPool`'un gizli `LinkedBlockingQueue` tuzağı), `RejectedExecutionHandler` stratejileri, `ScheduledExecutorService`, `ForkJoinPool` ve common pool tuzakları, custom `ThreadFactory` (jstack için isimlendirme), graceful shutdown semantiği. Banking için: domain başına ayrı pool (transfer pool, notification pool, audit pool) tasarımı.

## Süre

Okuma: 2.5 saat • Mini task'ler: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Topic 3.1, 3.2, 3.3 tamamlandı (JMM, sync, lock)
- Spring Boot service'lerde basit `@Async` veya `Thread` kullanımı duyuldu
- `Runnable`, `Callable`, `Future` interface'leri tanıdık

---

## Kavramlar

### 1. Thread lifecycle ve interrupt

Java thread'in `Thread.State` enum'ı 6 değer alır:

| State | Anlam |
|---|---|
| `NEW` | Thread yaratıldı ama `start()` çağrılmadı |
| `RUNNABLE` | Çalışmaya hazır veya çalışıyor |
| `BLOCKED` | Monitor lock bekliyor (synchronized) |
| `WAITING` | Bir sinyali sınırsız bekliyor (`wait`, `join`, `park`) |
| `TIMED_WAITING` | Sınırlı süre bekliyor (`sleep`, `wait(t)`, `join(t)`, `parkNanos`) |
| `TERMINATED` | `run()` bitti veya exception |

State machine basit ama bazı **incelikler** banking için kritik:

- `RUNNABLE` durumu **çalışıyor anlamına gelmez** — OS scheduler'ın hazır kuyruğunda olabilir. JFR `jdk.ThreadCPULoad` event'i gerçek CPU kullanımını verir.
- `BLOCKED` thread JIT optimizasyonları **kaybeder** — long-block edilen thread'in cache'i invalidate olur, performance penalty.
- `WAITING` ve `TIMED_WAITING` ayrı state'ler ama programatik fark çok azdır — `jstack`'te `parking to wait for ...` veya `sleeping` görürsün.

#### Interrupt semantik

`thread.interrupt()` thread'i **öldürmez**. Sadece **interrupt flag**'ini set eder. Thread bunu **kendi kontrol etmeli**:

- `Thread.interrupted()` — flag'i okur ve **temizler**
- `Thread.currentThread().isInterrupted()` — flag'i okur, **temizlemez**

Blocking metotlar (`Thread.sleep`, `Object.wait`, `BlockingQueue.take`, `Future.get`, `lock.lockInterruptibly`) interrupt flag set ise **`InterruptedException`** fırlatır + flag'i temizler.

**Banking pratiği — interrupt'a uyumlu kod:**

```java
public class InterruptAwareWorker implements Runnable {
    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                var task = takeNextTask();  // BlockingQueue.take throws InterruptedException
                process(task);
            }
        } catch (InterruptedException e) {
            // Re-set the interrupt flag (very important!)
            Thread.currentThread().interrupt();
            log.info("Worker interrupted, shutting down gracefully");
        }
    }
}
```

**Kritik kural — interrupt swallow yasak:**

```java
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    // ❌ flag'i kaybettin — üst seviye fark etmiyor
}
```

Çözüm: ya **rethrow** (`throw new RuntimeException(e)` veya re-throw), ya da **flag'i geri set et**:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // ← anahtar
    return;  // veya gerekli graceful kapanış
}
```

---

### 2. `Executor` ve `ExecutorService` interface'leri

#### Executor — en küçük abstraction

```java
public interface Executor {
    void execute(Runnable command);
}
```

Tek metot. Genelde direkt kullanmıyoruz; ama abstraction olarak güçlü.

#### ExecutorService — yönetim metotları

`Executor`'ı extend eder. Eklediği:

- `submit(Callable<T>)` → `Future<T>` (sonuç + exception alma)
- `submit(Runnable)` → `Future<?>`
- `invokeAll(Collection<Callable>)` — hepsi bitince dön (timeout opsiyonel)
- `invokeAny(Collection<Callable>)` — ilki bitince dön
- `shutdown()` — yeni iş kabul etme, mevcutları tamamla
- `shutdownNow()` — yeni iş kabul etme, mevcutlara interrupt gönder
- `awaitTermination(timeout)` — tüm task'ler bitene kadar bekle (timeout)

#### Future — async sonuç

```java
Future<Money> future = executor.submit(() -> fxService.fetchRate("USD/TRY"));
Money rate = future.get();              // blocking, exception unwrap
Money rateTimeout = future.get(5, TimeUnit.SECONDS);  // timeout
boolean cancelled = future.cancel(true); // mayInterruptIfRunning
```

`Future.get()` **blocking**. `CompletableFuture` (Topic 3.5) non-blocking chain'leri çözer.

---

### 3. `Executors` factory'leri — kullanım ve TUZAKLARI

`java.util.concurrent.Executors` factory class'ı **kullanışlı** ama **production'da tehlikeli** birkaç metot içerir.

#### `newFixedThreadPool(int n)` — sabit thread, UNBOUNDED queue

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
```

Görünüşte: 10 thread ile iş yapar. **GİZLİ TUZAK:** Backing queue `LinkedBlockingQueue` ve **default capacity `Integer.MAX_VALUE`**. Eğer thread'ler iş ile yetişemezse, kuyruk **sınırsız büyür** → **OutOfMemoryError**.

Banking örneği: 100 TPS gelen request, 10 worker thread her saniye 50 işleyebiliyor. Kuyruk **saniyede 50 task** büyür. 1 saatte 180,000 pending task, 1 günde 4.3 milyon. JVM heap dolar, GC döngüye girer, sonra OOM crash. Üretimden tasvir edilen alarm.

#### `newCachedThreadPool()` — UNBOUNDED thread, SYNCHRONOUS handoff

```java
ExecutorService pool = Executors.newCachedThreadPool();
```

Her task için **yeni thread yarat** (eğer 60 saniye idle thread yoksa). Cap **yok**. Burst trafiği → **thread sayısı patlar** → OS thread limit (Linux default ~32k) veya RAM (her thread 1MB stack default) tükenir → JVM crash.

Banking örneği: Black Friday gibi peak günü, 10k concurrent transfer geldi → 10k thread yarat → OOM.

#### `newSingleThreadExecutor()` — tek thread, UNBOUNDED queue

```java
ExecutorService pool = Executors.newSingleThreadExecutor();
```

Aynı tuzak: UNBOUNDED queue. Tek thread yetişemezse kuyruk büyür.

Yararlı yer: **sıralı işleme garantisi** gereken yerler (audit log, single-account sequential update). Ama kuyruğa cap koyman lazım.

#### `newScheduledThreadPool(int n)` — zamanlı, UNBOUNDED priority queue

```java
ScheduledExecutorService pool = Executors.newScheduledThreadPool(4);
pool.scheduleAtFixedRate(this::reloadFxRates, 0, 30, TimeUnit.SECONDS);
```

Periyodik iş için. Internal queue **DelayedWorkQueue**, unbounded ama task delay-based sıralanır.

#### `newWorkStealingPool()` — `ForkJoinPool` arkada

```java
ExecutorService pool = Executors.newWorkStealingPool();  // Runtime.availableProcessors() boyutu
```

`ForkJoinPool` döndürür. Work-stealing algorithm (sonraki bölüm).

#### `newVirtualThreadPerTaskExecutor()` — virtual thread (Java 21+)

```java
ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor();
```

Her task **yeni virtual thread**. Topic 3.7'de detay.

#### Sonuç — `Executors`'i kullanma!

Banking production'da **doğrudan `Executors` factory'leri kullanma** (`newVirtualThreadPerTaskExecutor` hariç). Bunun yerine **`ThreadPoolExecutor` constructor'ını explicit** kullan.

---

### 4. `ThreadPoolExecutor` — explicit constructor

```java
new ThreadPoolExecutor(
    int corePoolSize,                  // min thread
    int maximumPoolSize,                // max thread
    long keepAliveTime,                 // idle thread yaşam süresi (max-core arasındakiler)
    TimeUnit unit,                      // keepAlive birimi
    BlockingQueue<Runnable> workQueue,  // pending task queue
    ThreadFactory threadFactory,        // thread naming, daemon flag
    RejectedExecutionHandler handler    // queue full / shutdown senaryosu
);
```

#### Thread yaratma sırası — kritik!

`execute(task)` çağrıldığında:

1. **`activeThreadCount < corePoolSize`** → yeni thread yarat, task ata
2. Aksi halde, **queue'ya offer et** — başarılı ise task kuyruğa girer
3. Queue full ise, **`activeThreadCount < maximumPoolSize`** → yeni thread yarat, task ata
4. Max'a ulaşıldıysa, **`RejectedExecutionHandler`** çağrılır

**Banking nüansı:** Bu sıra "core'a kadar thread yarat, sonra queue, sonra max'a kadar thread, sonra reject" demek. Eğer queue **unbounded**'sa (Integer.MAX_VALUE), **2. adım hiç fail etmez**, max'a hiç ulaşılmaz. Yani max parameter pratikte **yok**.

Bu yüzden `LinkedBlockingQueue` (unbounded default) ile `corePoolSize = 10, maximumPoolSize = 100` yazsan, **hiç 100 thread'e çıkmaz**, 10'da kalır.

#### Bounded queue çözümü

```java
BlockingQueue<Runnable> queue = new LinkedBlockingQueue<>(500);  // capacity 500
// veya
BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(500);    // sabit boyut array
```

Queue dolarsa max thread'lere kadar yaratır, sonra reject.

#### `corePoolSize` ne kadar olmalı?

**CPU-bound iş:** `Runtime.availableProcessors()` veya `+1`. Bağıntı: işin CPU'da harcadığı zaman / toplam (CPU + IO).

**IO-bound iş:** Daha yüksek; "Little's Law"e göre = (target throughput) × (avg request duration).
- 100 TPS, her request 200ms → 100 × 0.2 = 20 thread minimum
- Practical buffer: 1.5x = 30 thread

**Banking pratiği — domain başına ayrı pool:**

- Transfer pool: 50 thread (yüksek throughput)
- Notification pool: 10 thread (SMS/email — IO-bound, biraz daha)
- Audit pool: 5 thread (DB write, low priority)
- FX pricing pool: 20 thread (cache yüklemek vs.)
- Report generation pool: 5 thread (heavy, low priority)

Domain'i karıştırma — bir alanda overload, diğerini etkilemesin.

#### `keepAliveTime` ve `allowCoreThreadTimeOut`

`keepAliveTime`: core'un üstündeki thread'ler bu süre kadar idle kalırsa termine olur. Default'ta core thread'ler hiç ölmez.

`allowCoreThreadTimeOut(true)` ile core thread'ler de timeout'a tabi olur — düşük yük dönemlerinde memory tasarrufu.

```java
pool.allowCoreThreadTimeOut(true);
```

Banking pratiği: trafik geceleri çok düşüyorsa, gündüze ısınma maliyeti az ise — etkinleştirebilir. Genellikle core fixed bırakılır.

#### `prestartAllCoreThreads`

```java
pool.prestartAllCoreThreads();
```

Lazy creation yerine **hemen** core thread'leri yarat. Cold start latency'sini azaltır.

---

### 5. `BlockingQueue` ailesi — pool'un kalbi

| Sınıf | Özellik |
|---|---|
| `ArrayBlockingQueue` | Sabit boyut, FIFO, bounded |
| `LinkedBlockingQueue` | Linked list, optional bounded (default unbounded) |
| `SynchronousQueue` | Capacity 0, direct hand-off |
| `PriorityBlockingQueue` | Priority order, **unbounded** |
| `DelayQueue` | Element'in delay'i bitince available |
| `LinkedTransferQueue` | Producer-consumer transfer semantic |

#### `SynchronousQueue` ile `CachedThreadPool` mantığı

`SynchronousQueue` capacity 0. `offer()` hemen fail eder eğer karşıda **bekleyen take()** yoksa. `CachedThreadPool` bunu kullanır:

- Task gelir → `queue.offer()` → ya bekleyen idle thread var, ona handoff; ya da fail → max'a kadar yeni thread yarat.

Bounded variant:
```java
new ThreadPoolExecutor(0, 50, 60L, TimeUnit.SECONDS,
    new SynchronousQueue<>(), threadFactory, rejectionHandler);
```

Max 50 thread, queue 0, hemen handoff veya yeni thread. Burst'e iyi cevap.

#### Hangi queue ne için

- **Yüksek throughput + bounded backpressure**: `LinkedBlockingQueue(N)`
- **Düşük latency, burst**: `SynchronousQueue` + yüksek max thread
- **CPU-bound iş**: Bounded queue, thread = core count
- **Priority** (örn. premium müşteri öncelikli transfer): `PriorityBlockingQueue` ama dikkat **unbounded**

---

### 6. `RejectedExecutionHandler` stratejileri

Queue dolarsa veya pool shutdown'sa hangi politika uygulanır?

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable task, ThreadPoolExecutor executor);
}
```

JDK 4 default sağlar:

#### `AbortPolicy` — default

```java
new ThreadPoolExecutor.AbortPolicy()
```

`RejectedExecutionException` fırlatır. Caller fail'i fark eder, kendi error handling yapar.

**Banking:** Default. Caller "transfer kuyruğu dolu, retry'a alındı" gibi user-facing error mesajına çevirebilir.

#### `CallerRunsPolicy` — backpressure

```java
new ThreadPoolExecutor.CallerRunsPolicy()
```

Caller thread **task'i kendi çalıştırır**. Caller bloke olur → upstream'e doğal backpressure yayılır.

**Banking:** Audit log gibi durdurulamayan iş. Caller'ın **ana iş** olarak çalıştığı durumda dikkatli ol — fail fast yerine yavaşlık yayılır.

#### `DiscardPolicy` — sessizce at

```java
new ThreadPoolExecutor.DiscardPolicy()
```

Task hiç çalışmaz, hata da yok. **Banking'de yasak** — para işlemi sessizce kaybedilemez.

#### `DiscardOldestPolicy` — en eski queued atılır, yeni eklenir

```java
new ThreadPoolExecutor.DiscardOldestPolicy()
```

Banking'de **yasak** — eski transfer atılırsa, müşteri "transferim nerede?" diye sorar.

#### Custom handler

```java
RejectedExecutionHandler customHandler = (task, executor) -> {
    metrics.increment("transfer.rejected");
    log.error("Transfer queue full, task rejected. Queue size: {}, pool size: {}", 
        executor.getQueue().size(), executor.getPoolSize());
    
    // Banking: rejected transfer'i bir dead-letter queue'ya yaz, sonra manual review
    deadLetterQueue.offer(task);
    
    throw new RejectedExecutionException("Transfer queue overloaded");
};
```

---

### 7. `ScheduledExecutorService` — zamanlı işler

```java
ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(
    4,
    namedThreadFactory("scheduler"),
    new ThreadPoolExecutor.AbortPolicy()
);

// One-shot delayed task
scheduler.schedule(() -> reloadFxRates(), 30, TimeUnit.SECONDS);

// Fixed rate — her N saniyede başlat (overlap olabilir)
scheduler.scheduleAtFixedRate(this::reloadFxRates, 0, 30, TimeUnit.SECONDS);

// Fixed delay — bir iş bitince N saniye sonra başlat (overlap yok)
scheduler.scheduleWithFixedDelay(this::healthCheck, 0, 10, TimeUnit.SECONDS);
```

#### `scheduleAtFixedRate` vs `scheduleWithFixedDelay`

- **FixedRate**: T, T+30, T+60, T+90... — eğer iş 35 saniye sürdüyse, sıradakine **hemen** başlar (catch-up). İş çok uzunsa, queue birikir.
- **FixedDelay**: T, sonra (iş bitiş + 30), sonra (önceki bitiş + 30)... — hiç overlap yok. Polling/heartbeat için ideal.

**Banking pratiği:**
- FX rate reload (idempotent, sabit aralık tercih) → `FixedRate`
- Health check (eski cevap önemsiz) → `FixedDelay`
- Reconciliation (yarım iş tehlikeli) → `FixedDelay`

#### Exception'a dikkat — silent kill

```java
scheduler.scheduleAtFixedRate(() -> {
    throw new RuntimeException("boom");   // ← ilk hata sonrası ARTIK ÇAĞRILMAZ
}, 0, 10, TimeUnit.SECONDS);
```

`ScheduledExecutorService` exception fırlayınca **periodic task'i susturur**. Bu **silent failure** banking için ölümcül.

**Çözüm — exception swallow wrapper:**

```java
scheduler.scheduleAtFixedRate(() -> {
    try {
        doActualWork();
    } catch (Throwable t) {
        log.error("Scheduled task failed", t);
        metrics.increment("scheduler.failure");
        // do NOT rethrow → next iteration runs
    }
}, 0, 10, TimeUnit.SECONDS);
```

Veya Spring `@Scheduled` kullanıyorsan, `ErrorHandler` configure et.

---

### 8. `ForkJoinPool` ve work-stealing

`ForkJoinPool` özel pool: her thread'in kendi **deque**'i (double-ended queue) vardır. Boş thread, başkasının deque'sinden **steal** eder. CPU-bound recursive task'lerde efficient.

```java
ForkJoinPool pool = new ForkJoinPool(8);

class FibonacciTask extends RecursiveTask<Long> {
    private final long n;
    FibonacciTask(long n) { this.n = n; }
    
    @Override
    protected Long compute() {
        if (n <= 1) return n;
        var f1 = new FibonacciTask(n - 1);
        var f2 = new FibonacciTask(n - 2);
        f1.fork();           // başka thread'e gönder
        return f2.compute() + f1.join();
    }
}

long result = pool.invoke(new FibonacciTask(30));
```

#### Common pool — gizli tuzak

`ForkJoinPool.commonPool()` — JVM çapında shared default pool. **`parallelStream()`** ve `CompletableFuture` default executor olarak bunu kullanır.

```java
List<Long> ids = ...;
ids.parallelStream().forEach(id -> processTransfer(id));
```

Bu kod **common pool**'da çalışır. Eğer `processTransfer` JDBC veya HTTP çağırıyorsa, common pool'un thread'lerini **bloke eder**. JVM çapında diğer parallel stream'ler de bloke olur — **cascading failure**.

**Banking kural:** `parallelStream()` veya `CompletableFuture.xxxAsync()` çağırırken **kendi executor**'unu pass et:

```java
CompletableFuture.supplyAsync(() -> fetchRate(), myCustomExecutor);
```

Topic 3.5'te detay.

#### Common pool boyutu

`Runtime.availableProcessors() - 1`. Yani 8 core'lu makinede 7 thread. Bu **çok az**. Eğer banking workload'unda parallel stream'i yoğun kullanıyorsan, JVM startup'ta:

```bash
-Djava.util.concurrent.ForkJoinPool.common.parallelism=32
```

ile artır. Ama gerçekten **kendi pool'unu** kullanman daha iyi.

---

### 9. Custom `ThreadFactory` — banking için zorunlu

Default `Executors.defaultThreadFactory()` thread'leri `pool-N-thread-M` adlandırır. Production'da **jstack** çıktısında "hangi pool ne yapıyor" anlaşılmaz.

**Custom factory:**

```java
public class NamedThreadFactory implements ThreadFactory {
    private final String namePrefix;
    private final boolean daemon;
    private final AtomicInteger counter = new AtomicInteger(1);
    private final Thread.UncaughtExceptionHandler uncaught;
    
    public NamedThreadFactory(String namePrefix, boolean daemon,
                              Thread.UncaughtExceptionHandler uncaught) {
        this.namePrefix = namePrefix;
        this.daemon = daemon;
        this.uncaught = uncaught;
    }
    
    @Override
    public Thread newThread(Runnable r) {
        var t = new Thread(r, namePrefix + "-" + counter.getAndIncrement());
        t.setDaemon(daemon);
        t.setUncaughtExceptionHandler(uncaught);
        return t;
    }
}
```

Kullanım:

```java
ThreadFactory tf = new NamedThreadFactory(
    "transfer-pool",
    false,  // non-daemon — shutdown'da bekle
    (thread, ex) -> log.error("Uncaught in {}: {}", thread.getName(), ex.getMessage(), ex)
);

var transferPool = new ThreadPoolExecutor(
    50, 100, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500),
    tf,
    new CustomRejectionHandler()
);
```

Şimdi `jstack` çıktısında `"transfer-pool-1"`, `"transfer-pool-2"`... görürsün. **Hemen** "transfer pool nedeniyle bloke" tespitini yapabilirsin.

#### `daemon` vs non-daemon

- **Daemon thread**: JVM bunlar dışında thread kalmadığında **kapanır**, daemon'ları zorla terminate eder.
- **Non-daemon**: JVM bekler. `pool.shutdown()` veya `System.exit()` gerekir.

Banking'de **non-daemon** kullan. Tamamlanmamış transfer kaybedilmesin.

#### `UncaughtExceptionHandler`

Pool içinde fırlatılan **unchecked** exception'lar varsayılan olarak slowly logged olur. Custom handler ile alert sistemine bağla.

---

### 10. Graceful shutdown semantiği

```java
public void shutdownGracefully(ExecutorService pool) {
    pool.shutdown();   // yeni iş kabul etme
    try {
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            log.warn("Pool did not terminate in 30s, forcing shutdown");
            pool.shutdownNow();    // pending iş list döner
            if (!pool.awaitTermination(10, TimeUnit.SECONDS)) {
                log.error("Pool did not terminate even after force");
            }
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

#### `shutdown()` vs `shutdownNow()`

| | `shutdown()` | `shutdownNow()` |
|---|---|---|
| Yeni iş kabul | ❌ | ❌ |
| Mevcut iş | ✓ tamamlatır | ⚠️ interrupt gönderir |
| Queue'daki iş | ✓ çalışır | ❌ atar, list döner |
| Bekleyen iş varsa | ✓ bekler | hemen interrupt |
| Block | ❌ (async) | ❌ (async) |

`shutdown()` graceful. `shutdownNow()` immediate.

#### Banking shutdown stratejisi

Spring Boot'ta `@PreDestroy` veya `DisposableBean`:

```java
@Component
public class TransferPoolManager implements DisposableBean {
    private final ExecutorService transferPool;
    
    @Override
    public void destroy() {
        log.info("Shutting down transfer pool gracefully");
        transferPool.shutdown();
        if (!transferPool.awaitTermination(60, TimeUnit.SECONDS)) {
            log.warn("Transfer pool did not terminate gracefully, forcing");
            var pending = transferPool.shutdownNow();
            log.warn("Pending tasks: {}", pending.size());
        }
    }
}
```

Spring Boot `application.yml`:

```yaml
spring:
  task:
    execution:
      shutdown:
        await-termination: true
        await-termination-period: 60s
```

---

### 11. Banking örneği — domain-specific pool layout

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean(name = "transferExecutor", destroyMethod = "shutdown")
    public ExecutorService transferExecutor() {
        return new ThreadPoolExecutor(
            50,                              // core
            100,                             // max
            60L, TimeUnit.SECONDS,           // keep alive
            new LinkedBlockingQueue<>(500),  // bounded queue
            new NamedThreadFactory("transfer", false, transferUncaughtHandler()),
            new TransferRejectedHandler()    // dead-letter queue
        );
    }
    
    @Bean(name = "notificationExecutor", destroyMethod = "shutdown")
    public ExecutorService notificationExecutor() {
        return new ThreadPoolExecutor(
            10, 30, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new NamedThreadFactory("notify", true, defaultUncaught()),  // daemon — kaybolabilir
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    @Bean(name = "auditExecutor", destroyMethod = "shutdown")
    public ExecutorService auditExecutor() {
        return new ThreadPoolExecutor(
            5, 5, 0L, TimeUnit.MILLISECONDS,  // fixed
            new LinkedBlockingQueue<>(5000),  // büyük queue, audit kaybedilmemeli
            new NamedThreadFactory("audit", false, auditUncaught()),
            new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure
        );
    }
    
    @Bean(name = "scheduledExecutor", destroyMethod = "shutdown")
    public ScheduledExecutorService scheduledExecutor() {
        var s = new ScheduledThreadPoolExecutor(
            4,
            new NamedThreadFactory("scheduler", false, defaultUncaught())
        );
        s.setRemoveOnCancelPolicy(true);
        return s;
    }
}
```

**Tasarım prensipleri:**

- **Domain başına ayrı pool** — transfer overload notification'ı etkilemesin
- **Naming convention** — jstack'te hemen anlaşılır
- **Bounded queue** — OOM koruması
- **Uncaught handler** — log + metric
- **Rejection policy** — domain-aware (transfer için dead letter, audit için CallerRuns)
- **Graceful shutdown** — destroyMethod = "shutdown"

---

### 12. Pool monitoring — banking observability

`ThreadPoolExecutor` introspection metotları:

```java
pool.getActiveCount();        // şu an çalışan thread
pool.getPoolSize();           // toplam thread (idle dahil)
pool.getCorePoolSize();       // core size
pool.getMaximumPoolSize();    // max
pool.getQueue().size();       // pending task
pool.getCompletedTaskCount(); // tamamlanan
pool.getTaskCount();          // toplam görev (queue'dakiler dahil)
```

**Micrometer / Prometheus binding:**

```java
@Bean
public MeterBinder threadPoolMetrics(ThreadPoolExecutor transferPool) {
    return registry -> {
        Gauge.builder("transfer.pool.active", transferPool, ThreadPoolExecutor::getActiveCount)
            .register(registry);
        Gauge.builder("transfer.pool.queue.size", transferPool, p -> p.getQueue().size())
            .register(registry);
        // ...
    };
}
```

**Banking alert eşikleri:**

- `queue.size > capacity * 0.8` → warning
- `queue.size > capacity * 0.95` → critical
- `active == max` sürekli → max yetmiyor, ölçeklendir
- `completed.task / elapsed` düşüyor → contention veya downstream slowdown

---

### 13. Anti-pattern'ler

**Anti-pattern 1: `Executors.newCachedThreadPool` production'da**

OOM riski. Custom `ThreadPoolExecutor`'a geç.

**Anti-pattern 2: `Executors.newFixedThreadPool` production'da**

UNBOUNDED queue. Burst trafik OOM.

**Anti-pattern 3: parallelStream banking'de**

Common pool blocking — başka feature'ları etkiler. Kendi executor'ını kullan.

**Anti-pattern 4: Pool naming yok**

jstack çıktısında `pool-3-thread-7` → debugging cehennemi.

**Anti-pattern 5: Exception silent kill scheduled task'i**

Wrap try-catch, log, devam.

**Anti-pattern 6: Aynı pool'a karışık iş**

Transfer + notification + audit aynı pool'da → biri yavaşladığında hepsi yavaşlar (head-of-line blocking).

**Anti-pattern 7: `Future.get()` timeout'suz**

```java
future.get();   // ❌ sonsuza kadar bekler
```

Her zaman `future.get(timeout, unit)`.

**Anti-pattern 8: Submit edilen task'in exception swallowing**

```java
pool.submit(() -> {
    try {
        doWork();
    } catch (Exception e) {
        // sessizce kaybediliyor
    }
});
```

Çözüm: `future.get()` ile alma, ya da uncaught handler.

**Anti-pattern 9: `shutdown` çağrılmamış**

JVM kapanmaz (non-daemon thread varsa).

**Anti-pattern 10: Thread per request (eski) virtual thread olmadan**

Java 21+ virtual thread var; ama lego'lar kurulu değilse, classical bounded pool yine doğru. Topic 3.7'de detay.

---

## Önemli olabilecek araştırma kaynakları

- "Java Concurrency in Practice" Goetz, Chapter 6 (Task Execution) ve 8 (Applying Thread Pools)
- ThreadPoolExecutor JavaDoc (resmi, çok bilgi)
- "Modern Java in Action" Manning, executor framework chapter
- Brian Goetz — "Java concurrency, in retrospect" talk
- Doug Lea — "Concurrent programming in Java" (kitap)
- "Little's Law" Wikipedia (queueing theory)
- ForkJoinPool JEP and JavaDoc
- Spring Boot AsyncConfigurer, TaskExecutor docs
- Micrometer thread pool metrics docs
- Heinz Kabutz — "Newsletter on executor pitfalls" (article series)

---

## Mini task'ler

### Task 3.4.1 — `Executors` factory tuzakları (defter, 20 dk)

`ExecutorsTraps.md` yaz:
- `newFixedThreadPool`'un gizli unbounded queue tuzağı
- `newCachedThreadPool`'un unbounded thread tuzağı
- Production'da neden custom `ThreadPoolExecutor` constructor kullanılır

### Task 3.4.2 — Bounded pool + named factory (45 dk)

`pool/TransferPool.java`:

- Core 50, max 100, bounded queue (capacity 500)
- `NamedThreadFactory` ile thread'leri `transfer-N` adlandır
- Custom `RejectedExecutionHandler`: log + metric + (mock) dead-letter
- 1000 task submit et, jstack ile thread isimlerini gör (manuel test)

### Task 3.4.3 — Queue full reject reproduction (30 dk)

Yukarıdaki pool'a 5000 task'i hızlıca submit et. Her task'in `Thread.sleep(1000)`'i var. Queue 500 + pool 100 = 600'den fazlası reject olur. Reject sayısını metric olarak ölç. Defterine yaz.

### Task 3.4.4 — `ScheduledExecutorService` + exception silent kill (45 dk)

`scheduler/FxReloader.java`:

- Her 5 saniyede `reloadRates()` çağır
- İlk versiyon: exception swallow yok — kasıtlı `throw new RuntimeException()` ekle 3. iterasyonda
- Çıktıda 3. iterasyondan sonra **artık çağrılmadığını** gör (defterine yaz)
- İkinci versiyon: try-catch wrapper. Tekrar test.

### Task 3.4.5 — `parallelStream` common pool blocking (45 dk)

`forkjoin/CommonPoolBlocking.java`:

- `List<Integer> ids = IntStream.range(0, 100).boxed().toList();`
- `ids.parallelStream().forEach(id -> Thread.sleep(1000))` — common pool dolar
- Aynı anda başka thread `IntStream.range(0, 100).parallel().sum()` — bloke olur
- Custom `ForkJoinPool(16)` ile aynı kodu yaz, izole çalış
- Karşılaştır.

### Task 3.4.6 — Domain-specific pool tasarımı (60 dk)

`config/ThreadPoolConfig.java` Spring config (veya plain factory class):

- `transferExecutor` (core 50, max 100, bounded 500)
- `notificationExecutor` (core 10, max 30, queue 1000, CallerRunsPolicy)
- `auditExecutor` (fixed 5, queue 5000, CallerRunsPolicy)
- `scheduledExecutor` (4 thread)

Her birinin `NamedThreadFactory` + uncaught handler ile.

### Task 3.4.7 — Graceful shutdown (30 dk)

`pool/GracefulShutdownDemo.java`:

- Yukarıdaki pool'lar
- 100 task submit (her biri 500ms iş)
- 200ms sonra `shutdown()` çağır
- `awaitTermination(30s)` — tüm task'ler bitmeli
- `shutdownNow()` kullansaydın kaç task interrupted olurdu? → ayrı versiyonda dene
- Defterine sonucu yaz

### Task 3.4.8 — Pool monitoring (45 dk)

`metrics/PoolMonitor.java`:

- `transferPool.getActiveCount()`, `getQueue().size()`, `getCompletedTaskCount()` her 1 saniyede log
- 1000 task submit, monitoring'i terminal'de izle
- Defterine snapshot grafiğini çiz (queue size over time, active over time)

---

## Test yazma rehberi

### Test 3.4.1 — Bounded queue reject

```java
@Test
void boundedPoolRejectsAfterCapacity() throws Exception {
    int core = 2, max = 4, queue = 5;
    var rejected = new AtomicInteger();
    
    var pool = new ThreadPoolExecutor(
        core, max, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(queue),
        new NamedThreadFactory("test", false, (t, e) -> {}),
        (r, exec) -> rejected.incrementAndGet()
    );
    
    var latch = new CountDownLatch(1);
    // 4 thread busy + 5 queued = 9 task'e kadar OK
    for (int i = 0; i < 20; i++) {
        pool.execute(() -> {
            try { latch.await(); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
    
    // 20 - 9 = 11 reject
    assertThat(rejected.get()).isEqualTo(11);
    
    latch.countDown();
    pool.shutdown();
    assertThat(pool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
}
```

### Test 3.4.2 — `ScheduledExecutor` silent kill ve fix

```java
@Test
void scheduledTaskStopsOnException_butWrapperFixes() throws Exception {
    var scheduler = new ScheduledThreadPoolExecutor(1);
    var counter = new AtomicInteger();
    
    // Wrap to ensure continuation
    scheduler.scheduleAtFixedRate(() -> {
        try {
            counter.incrementAndGet();
            if (counter.get() == 3) throw new RuntimeException("boom");
        } catch (Throwable t) {
            // swallowed → next iteration runs
        }
    }, 0, 50, TimeUnit.MILLISECONDS);
    
    Thread.sleep(500);
    
    // wrapper sayesinde sayaç 5'ten büyük olmalı
    assertThat(counter.get()).isGreaterThan(5);
    scheduler.shutdown();
}

@Test
void scheduledTaskWithoutWrapperStopsAtFirstException() throws Exception {
    var scheduler = new ScheduledThreadPoolExecutor(1);
    var counter = new AtomicInteger();
    
    scheduler.scheduleAtFixedRate(() -> {
        counter.incrementAndGet();
        if (counter.get() == 3) throw new RuntimeException("boom");
    }, 0, 50, TimeUnit.MILLISECONDS);
    
    Thread.sleep(500);
    
    // 3'te dururdu — wrapper yok
    assertThat(counter.get()).isEqualTo(3);
    scheduler.shutdown();
}
```

### Test 3.4.3 — Thread naming jstack uyumlu

```java
@Test
void threadFactoryNamesThreadsCorrectly() throws Exception {
    var factory = new NamedThreadFactory("test-pool", false, (t, e) -> {});
    var pool = Executors.newFixedThreadPool(3, factory);
    var names = new CopyOnWriteArrayList<String>();
    
    for (int i = 0; i < 3; i++) {
        pool.submit(() -> names.add(Thread.currentThread().getName()));
    }
    
    pool.shutdown();
    assertThat(pool.awaitTermination(2, TimeUnit.SECONDS)).isTrue();
    
    assertThat(names).allMatch(n -> n.startsWith("test-pool-"));
}
```

### Test 3.4.4 — Graceful shutdown completes pending tasks

```java
@Test
void shutdownCompletesPendingTasks() throws Exception {
    var pool = Executors.newFixedThreadPool(2);
    var completed = new AtomicInteger();
    
    for (int i = 0; i < 5; i++) {
        pool.submit(() -> {
            try { Thread.sleep(200); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
            completed.incrementAndGet();
        });
    }
    
    pool.shutdown();
    assertThat(pool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    
    assertThat(completed.get()).isEqualTo(5);
}
```

---

## Claude-verify prompt

```
Aşağıdaki Java kodum executor framework'ünü öğrenmeye yönelik. Lütfen değerlendir 
ve EKSİKLERİ söyle, kod yazma:

1. Executors factory tuzakları:
   - `newFixedThreadPool`'un unbounded queue tuzağı not edilmiş mi?
   - `newCachedThreadPool`'un unbounded thread tuzağı not edilmiş mi?
   - Production'da `Executors`'i değil custom `ThreadPoolExecutor` kullanma kuralı uygulanmış mı?

2. ThreadPoolExecutor parametreleri:
   - Core, max, keepAlive, queue, factory, rejection hepsi explicit set edilmiş mi?
   - Bounded queue kullanılmış mı (LinkedBlockingQueue(N) veya ArrayBlockingQueue)?
   - Thread yaratma sırası (core → queue → max → reject) anlaşılmış mı?

3. NamedThreadFactory:
   - Thread isimleri "<pool>-<n>" formatında mı?
   - Daemon flag set edilmiş mi (banking için non-daemon default)?
   - UncaughtExceptionHandler ile log + metric var mı?

4. RejectedExecutionHandler:
   - 4 default strategy'nin (Abort, CallerRuns, Discard, DiscardOldest) farkları bilinmiyor mu?
   - Banking için DiscardPolicy/DiscardOldest yasak — uyulmuş mu?
   - Custom handler dead-letter queue + metric ile yazılmış mı?

5. ScheduledExecutorService:
   - fixedRate vs fixedDelay farkı bilinmiyor mu?
   - Exception silent kill problemi anlaşılmış mı?
   - Wrapper try-catch ile fix var mı?

6. ForkJoinPool:
   - Common pool tuzağı (parallelStream blocking) reproduce edilmiş mi?
   - Kendi ForkJoinPool ile izolasyon yapılmış mı?
   - Common pool boyutu (-D flag) ayarlama bilgisi var mı?

7. Domain-specific pool layout:
   - Transfer / notification / audit / scheduled ayrı pool'lar tasarlanmış mı?
   - Her pool'un parametreleri (core, max, queue, rejection) domain'e uygun mu?
   - Naming convention pool başına farklı mı?

8. Graceful shutdown:
   - shutdown() → awaitTermination → shutdownNow pattern var mı?
   - @PreDestroy veya destroyMethod="shutdown" ile lifecycle bağlı mı?
   - Timeout makul mü (30-60s)?

9. Pool monitoring:
   - getActiveCount, getQueue().size(), getCompletedTaskCount kullanılmış mı?
   - Metric binding (Micrometer veya custom) yapılmış mı?
   - Alert eşikleri (queue > 80%) düşünülmüş mü?

10. Anti-pattern kontrolü:
    - Future.get() timeout'suz çağrılmış mı?
    - Submit edilen task içinde exception swallowed mı?
    - parallelStream banking iş yükünde kullanılmış mı?
    - Aynı pool'da karışık domain task'leri var mı?

Her madde için PASS / FAIL / EKSIK. Kod yazma, sadece eksiklikleri söyle.
```

---

## Tamamlama kriterleri

- [ ] `Executors` factory metotlarının tuzaklarını ezberimde
- [ ] `ThreadPoolExecutor`'un 7 constructor parametresini tek tek tanımlayabiliyorum
- [ ] Thread yaratma sırasını (core → queue → max → reject) anlatabiliyorum
- [ ] Bounded queue ile pool yazdım, queue full reject reproduce ettim
- [ ] `NamedThreadFactory` yazdım, jstack çıktısında isimleri gördüm
- [ ] 4 default `RejectedExecutionHandler`'ın farklarını biliyorum, banking için hangisini ne zaman kullandığımı söyleyebiliyorum
- [ ] `ScheduledExecutor` silent kill bug'ını reproduce ettim, wrapper ile fix ettim
- [ ] `fixedRate` vs `fixedDelay` farkını örnekle açıklayabiliyorum
- [ ] `ForkJoinPool.commonPool` blocking tuzağını gördüm, kendi pool ile izole ettim
- [ ] Domain-specific pool layout (transfer/notification/audit/scheduled) tasarladım
- [ ] Graceful shutdown pattern'i kodumda var (Spring @PreDestroy veya manual)
- [ ] Pool monitoring metric'i ile alert eşiği belirleyebiliyorum
- [ ] `Future.get()` timeout her zaman pass ediyorum
- [ ] Interrupt swallow yasak — flag re-set ediyorum

Hepsi onaylı → Topic 3.5'e geç → [05-completable-future/](../05-completable-future/README.md)

---

## Defter notları

1. "`Executors.newFixedThreadPool`'un gizli tuzağı: ____. Banking'de bunun anlamı: ____."
2. "`ThreadPoolExecutor`'un 7 parametresi: ____."
3. "Thread yaratma sırası: ____ → ____ → ____ → ____. Unbounded queue'da hangi adım atlanır: ____."
4. "`NamedThreadFactory`'nin önemi (jstack perspektifi): ____."
5. "`RejectedExecutionHandler` 4 default'u: ____. Banking için yasak olanlar: ____."
6. "`scheduleAtFixedRate` exception fırlatınca: ____. Çözüm: ____."
7. "`parallelStream` ve `CompletableFuture` default executor: ____. Banking'de tuzağı: ____."
8. "`shutdown()` vs `shutdownNow()` farkı: ____."
9. "Banking için domain-specific pool layout: ____ ve gerekçesi: ____."
10. "Interrupt swallow neden yasak: ____. Doğru yaklaşım: ____."
