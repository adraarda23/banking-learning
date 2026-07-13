# Topic 3.7 — Virtual Threads (Project Loom)

## Hedef

Java 21'in en önemli yeniliği — virtual thread — banking-grade seviyede anlamak. Platform thread vs virtual thread farkını, carrier thread mekanizmasını, **pinning** tuzağını ve bunu nasıl tespit edip çözeceğini öğrenmek. `ScopedValue` ile ThreadLocal alternatifini, banking domain'inde 10k+ concurrent transfer senaryosunu virtual thread ile çözmeyi kavramak.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 3.1-3.6 bitti (JMM, executor, locks, concurrent collections biliyorsun)
- "Thread = OS thread" varsayımıyla yıllardır yaşıyorsun
- `ExecutorService` kullanıyorsun
- Java 21 SDK kurulu (`java -version`)

---

## Kavramlar

### 1. Platform thread'in maliyeti — neden Loom var

Bir geleneksel Java thread = **bir OS thread**. Her OS thread:

- **~1-2 MB stack** (varsayılan)
- **OS kernel'in scheduling unit'i** (context switch maliyeti)
- **Kaynak sınırlı** (Linux'ta `ulimit -u` typically 4096-65k)
- **Kernel-mode geçişi** her context switch için maliyetli

Banking örneği — "1 thread per request" model:

```
1000 concurrent request → 1000 OS thread
1000 × 1MB stack = 1GB sadece thread stack
+ kernel scheduler overhead
+ context switching katlamaya başlar
```

**Sonuç:** Java backend'lerinde **10k+ concurrent request'i** tutamadık. NIO, reactive programming (Project Reactor, RxJava) bu yüzden ortaya çıktı — async kod yazarak az thread ile çok bağlantı tutmak. Ama:

```java
// Reactive — okumak zor
Mono.fromCallable(() -> accountRepo.findById(id))
    .flatMap(account -> Mono.fromCallable(() -> transferRepo.save(...)))
    .subscribeOn(Schedulers.boundedElastic())
    .doOnError(e -> log.error(...))
    .subscribe();
```

Reactive **çalışıyor** ama:
- Stack trace okumak imkânsız
- Debug zor
- Mental model dağınık
- Imperative kod yazımı kaybolmuş

### 2. Project Loom — virtual thread fikri

**Çözüm:** Thread'i JVM-managed yap. OS thread'lerle 1:1 değil, **çok-1** ilişki.

- **Platform thread:** Geleneksel OS thread. 1:1 OS thread eşlemesi.
- **Virtual thread:** JVM içinde yaşar, **carrier** platform thread'in üstünde çalışır. Bloklayıcı işlemde **unmount** olur, carrier başka virtual thread'e geçer.

```
Platform thread × 16 (CPU core sayısı kadar tipik)
   ↑
   │ carrier
   ↓
Virtual thread × 10,000 (kullanım gerektiği kadar)
```

**Sonuç:** 10k concurrent request → 10k virtual thread → 16 platform thread → 16 OS thread. **Imperative kod ama scale**.

### 3. Virtual thread oluşturma

```java
// Method 1: factory
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("Hello from virtual thread");
});

// Method 2: short form
Thread vt = Thread.startVirtualThread(() -> System.out.println("Hi"));

// Method 3: ExecutorService
try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> doWork());
}

// Method 4: explicit factory in ThreadPoolExecutor
ThreadFactory factory = Thread.ofVirtual().name("banking-vt-", 0).factory();
```

**Banking örneği — REST endpoint per request:**

```java
// Spring Boot 3.2+ — virtual thread per request
spring:
  threads:
    virtual:
      enabled: true
```

Hepsi bu kadar. Her HTTP request artık bir virtual thread'de çalışır.

### 4. Carrier thread mekanizması

Bir virtual thread çalışırken:

1. Bir **carrier** (platform) thread üzerine **mount** olur
2. Kod çalışır
3. **Blocking call** (I/O, sleep, lock acquire) ile karşılaşınca:
   - Virtual thread carrier'dan **unmount** olur
   - Carrier başka bir virtual thread'i mount eder
4. Blocking operasyon bittiğinde virtual thread **yeniden mount** olur (aynı veya farklı carrier'a)

**Carrier pool:** Default `ForkJoinPool.commonPool()` ile aynı boyutta (genelde CPU core sayısı).

```java
-Djdk.virtualThreadScheduler.parallelism=16   // carrier count
-Djdk.virtualThreadScheduler.maxPoolSize=256   // max
```

### 5. Pinning — virtual thread'in en büyük tuzağı

**Pinning:** Virtual thread carrier'dan **unmount olamaz**. Carrier thread blocked kalır, başka virtual thread'ler bekler.

#### Pinning sebepleri (Java 21)

**Sebep 1: `synchronized` blok içinde blocking call**

```java
synchronized (lock) {
    Thread.sleep(1000);   // ← PINNING — carrier 1 sn kilitli
}
```

JVM monitor lock'ı virtual thread parking için yeniden tasarlanmadı (Java 21'de). Eğer `synchronized` blok içinde:
- `Thread.sleep`
- Network I/O
- File I/O
- Lock acquire

→ Virtual thread **pinned** kalır.

**Sebep 2: Native method**

```java
synchronized (lock) {
    // herhangi bir native call → pinned
}
```

JNI çağrıları doğal olarak pinning yapar.

**Sebep 3: Class initialization**

```java
class HeavyClass {
    static {
        Thread.sleep(1000);   // static init bloğu içinde
    }
}
```

#### Banking örneği — pinning bug

```java
// ❌ Klasik banking kodu — pinning içerir
public class AccountService {
    
    @Transactional
    public synchronized void transfer(UUID from, UUID to, BigDecimal amount) {
        Account fromAcc = repo.findById(from);   // ← JDBC blocking
        Account toAcc = repo.findById(to);       // ← JDBC blocking
        fromAcc.withdraw(amount);
        toAcc.deposit(amount);
        repo.save(fromAcc);                       // ← JDBC blocking
        repo.save(toAcc);                         // ← JDBC blocking
    }
}
```

`synchronized` method + içinde JDBC call'ları → **her transfer süresince carrier thread kilitli**. Throughput'un yüksek olmayacak.

#### Tespit — JFR ile

```bash
java -XX:StartFlightRecording=duration=60s,filename=app.jfr \
     -XX:+UnlockDiagnosticVMOptions \
     -Djdk.tracePinnedThreads=full \
     -jar app.jar
```

`jdk.tracePinnedThreads=full` — pinning olduğunda stack trace'i yazar.

```
[VirtualThread[#42]/runnable@ForkJoinPool-1-worker-3] Thread is pinned
  at com.example.AccountService.transfer(AccountService.java:25)
  at java.base/java.util.concurrent...
```

JFR'da event: `jdk.VirtualThreadPinned`. JMC veya CLI ile aç:

```bash
jfr print --events jdk.VirtualThreadPinned app.jfr
```

#### Çözüm — `ReentrantLock`

`synchronized` yerine `ReentrantLock` virtual thread-aware:

```java
private final ReentrantLock lock = new ReentrantLock();

@Transactional
public void transfer(UUID from, UUID to, BigDecimal amount) {
    lock.lock();
    try {
        // I/O, blocking calls OK — unmount olur
        Account fromAcc = repo.findById(from);
        // ...
    } finally {
        lock.unlock();
    }
}
```

**Banking kural:** Virtual thread çağıran service code'da `synchronized` kullanma. `ReentrantLock` tercih et.

### 6. Pinning — Java 23+ güncellemesi

Java 21'de `synchronized` pinning yaratıyor. **Java 23+** ile bu büyük ölçüde çözüldü (JEP 491): synchronized içinde virtual thread artık unmount olabilir.

Ancak hâlâ pinning yaratan durumlar var:
- **Native method calls** (JNI)
- **`MethodHandle.invokeExact` belirli koşullarda**
- **Some I/O operations** (synchronization primitive yerleştirilmemiş)

**Banking gerçeği:** Bazı eski library'ler hâlâ `synchronized` ile JDBC connection yönetir. Bunları kullanırken Java versiyonu önemli.

### 7. ThreadLocal vs ScopedValue (Java 21 preview)

Virtual thread varsayım: **çok fazla** olabilir. Her birinde ThreadLocal değerleri tutmak hafıza tüketir.

```java
private static final ThreadLocal<UserContext> userContext = new ThreadLocal<>();
// 10k virtual thread = 10k UserContext referansı

userContext.set(ctx);   // her request'te
try {
    doWork();
} finally {
    userContext.remove();   // ← cleanup unutursan memory leak
}
```

**ScopedValue** (Java 21 incubator):

```java
private static final ScopedValue<UserContext> USER_CONTEXT = ScopedValue.newInstance();

ScopedValue.where(USER_CONTEXT, ctx).run(() -> {
    doWork();
    UserContext current = USER_CONTEXT.get();
});
// scope dışına çıkınca otomatik unbinding
```

**Avantajlar:**
- **Immutable** — bir kez set, scope içinde değişmez (bypass için `where().run()` ile yeni scope)
- **Otomatik cleanup** — scope sonunda kaybolur
- **Yapısal aktarım** — child scope'lar parent value'sunu okuyabilir, dışarı sızmaz
- **Performance** — lookup ThreadLocal'dan hızlı

**Banking örneği — request context:**

```java
public record BankingContext(UUID userId, String traceId, String tenantId) {}

private static final ScopedValue<BankingContext> CONTEXT = ScopedValue.newInstance();

@Component
class RequestContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        BankingContext ctx = extractContext((HttpServletRequest) req);
        try {
            ScopedValue.where(CONTEXT, ctx).run(() -> {
                try { chain.doFilter(req, res); }
                catch (Exception e) { throw new RuntimeException(e); }
            });
        } catch (Exception e) { /* unwrap */ }
    }
}

// Service'te erişim
@Service
class TransferService {
    public void transfer(...) {
        BankingContext ctx = CONTEXT.get();
        log.info("Transfer by user {} in tenant {}", ctx.userId(), ctx.tenantId());
    }
}
```

**Tuzak:** ScopedValue Java 21'de **preview/incubator**. Production'da `--enable-preview` flag gerekir, API değişebilir. Spring Framework 6.1+ entegrasyonu var.

### 8. Virtual thread doğru kullanım pattern'leri

#### Pattern 1: HTTP request handling

Spring Boot 3.2+ otomatik:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Tomcat her request'i virtual thread'de çalıştırır. **Tek satır config, instant scaling.**

#### Pattern 2: Massive parallel I/O

```java
List<UUID> accountIds = ...;  // 10,000 hesap

try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Account>> futures = accountIds.stream()
        .map(id -> exec.submit(() -> repo.findById(id)))
        .toList();
    
    List<Account> accounts = futures.stream()
        .map(f -> { try { return f.get(); } catch (Exception e) { throw new RuntimeException(e); } })
        .toList();
}
```

10k DB call paralel, throughput DB-bound. Eski model: 10k OS thread = patlama.

#### Pattern 3: External API fan-out

```java
List<FxProvider> providers = List.of(tcmbProvider, ecbProvider, fixerProvider);

try (var scope = new StructuredTaskScope.ShutdownOnSuccess<FxRate>()) {
    providers.forEach(p -> scope.fork(() -> p.getRate("USD", "TRY")));
    scope.join();
    FxRate fastest = scope.result();
}
```

Structured concurrency (Topic 3.8) ile daha temiz.

### 9. Virtual thread anti-pattern'leri

**Anti-pattern 1: CPU-bound iş için virtual thread**

```java
try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 1000).forEach(i -> exec.submit(() -> {
        // CPU-intensive computation, no blocking
        heavyMatrix.multiply();
    }));
}
```

Virtual thread CPU-bound iş için **avantaj sağlamaz** — carrier thread aynen iş yapıyor, virtual thread overhead ekliyor. **`ForkJoinPool` veya bounded thread pool kullan.**

**Anti-pattern 2: Pool'lamak**

```java
// ❌ Hatalı düşünce
ExecutorService pool = Executors.newFixedThreadPool(100);
// "Virtual thread expensive, pool'ayım"
```

Virtual thread **ucuz** — pool'lamak anlamsız. `newVirtualThreadPerTaskExecutor()` kullan. Lifecycle JVM'in işi.

**Anti-pattern 3: `synchronized` + I/O**

Yukarıda detaylandırıldı. Pinning üretir.

**Anti-pattern 4: ThreadLocal leak**

```java
ThreadLocal<HeavyObject> tl = new ThreadLocal<>();
tl.set(heavy);
// remove unutuldu → virtual thread sonlandığında heavy hâlâ referansta
```

Virtual thread sona erdiğinde GC olur ama scope dışı leak'leri **ScopedValue ile** çöz.

**Anti-pattern 5: Carrier thread name'ine güvenmek**

```java
log.info("Thread: {}", Thread.currentThread().getName());
// Virtual thread'lerde isim default boş veya generic
```

Log'larda virtual thread'i ayırt etmek istiyorsan **explicit name** ver:

```java
Thread.ofVirtual().name("transfer-vt-", 0).start(...);
```

### 10. JMH benchmark — virtual vs platform

```java
@State(Scope.Benchmark)
public class TransferBenchmark {
    
    @Param({"100", "1000", "10000"})
    int concurrency;
    
    @Benchmark
    public void platformThreads() throws InterruptedException {
        ExecutorService exec = Executors.newFixedThreadPool(concurrency);
        runTransfers(exec);
        exec.shutdown();
        exec.awaitTermination(1, TimeUnit.MINUTES);
    }
    
    @Benchmark
    public void virtualThreads() throws InterruptedException {
        try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
            runTransfers(exec);
        }
    }
    
    private void runTransfers(ExecutorService exec) {
        CountDownLatch latch = new CountDownLatch(concurrency);
        for (int i = 0; i < concurrency; i++) {
            exec.submit(() -> {
                simulateIo();   // 10ms blocking
                latch.countDown();
            });
        }
        try { latch.await(); } catch (InterruptedException e) {}
    }
    
    private void simulateIo() {
        try { Thread.sleep(10); } catch (InterruptedException e) {}
    }
}
```

Beklenen sonuçlar (yüksek I/O wait scenario):
- 100 concurrency: virtual ≈ platform
- 1000 concurrency: virtual 5-10x daha iyi
- 10000 concurrency: platform thread OOM olur, virtual çalışır

### 11. Spring Boot 3.2+ virtual thread integration

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Bu config:
- Tomcat connector → virtual thread per request
- `@Async` method'lar → virtual thread executor
- Scheduled task'lar → virtual thread

**Manuel kontrol:**

```java
@Configuration
class VirtualThreadConfig {
    
    @Bean
    AsyncTaskExecutor virtualThreadExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
    
    @Bean
    TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> protocolHandler.setExecutor(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }
}
```

### 12. Banking pattern — high-throughput service

**Senaryo:** Notification service. 1k notification/sec, her biri SMS gateway external call yapıyor (300ms latency).

```
Eski (platform thread, 100 thread pool):
1000 req/s × 0.3s = 300 thread gerekiyor ama 100 var → queue dolar, latency artar

Virtual thread:
Her notification → virtual thread → SMS gateway call → unmount → carrier başkasına geçer
Throughput limit: SMS gateway'in dayanabildiği kadar (DB connection pool yetmesi vs.)
```

10k concurrent notification, ~16 carrier thread ile rahatça.

### 13. Connection pool tuning + virtual threads

Virtual thread çok olur, ama **DB connection** hâlâ sınırlı. HikariCP `maximumPoolSize` çok yükseltme.

**Tehlike:**

```java
// 10000 virtual thread, hepsi DB call
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
for (int i = 0; i < 10000; i++) {
    exec.submit(() -> repo.findById(...));
}
```

10000 thread, 10000 connection request → HikariCP `maximumPoolSize=10` → 9990 thread queue'da bekler.

**Çözüm:** Semaphore ile DB call'ları **rate limit** et:

```java
private final Semaphore dbLimit = new Semaphore(50);   // max 50 concurrent DB call

public Account findAccount(UUID id) throws InterruptedException {
    dbLimit.acquire();
    try {
        return repo.findById(id);
    } finally {
        dbLimit.release();
    }
}
```

Veya `BlockingQueue` ile job queue pattern.

---

## Önemli olabilecek araştırma kaynakları

- JEP 444: Virtual Threads (Java 21 final)
- JEP 491: Synchronize Virtual Threads without Pinning (Java 24 preview)
- "Java Loom" — Inside Java podcast series
- Project Loom official documentation
- Cay Horstmann — "Modern Java in Action" — Virtual Threads chapter
- Brian Goetz — JavaOne talks on Project Loom
- Spring Boot 3.2 virtual thread integration docs
- `--enable-preview` flag usage (ScopedValue)

---

## Mini task'ler

### Task 3.7.1 — Platform vs virtual thread sayım (20 dk)

`ThreadComparison.java`:

```java
public class ThreadComparison {
    public static void main(String[] args) throws InterruptedException {
        // Try: how many platform threads?
        try {
            for (int i = 0; i < 100_000; i++) {
                new Thread(() -> {
                    try { Thread.sleep(60_000); } catch (InterruptedException e) {}
                }).start();
            }
        } catch (OutOfMemoryError e) {
            System.out.println("Platform OOM at: ?");
        }
    }
}
```

Sonra aynısını virtual thread ile yap:

```java
for (int i = 0; i < 1_000_000; i++) {
    Thread.startVirtualThread(() -> {
        try { Thread.sleep(60_000); } catch (InterruptedException e) {}
    });
}
```

Memory tüketimini gözle (`jcmd <pid> VM.native_memory`). **Defterine yaz.**

### Task 3.7.2 — Pinning reproduction (45 dk)

`PinningDemo.java`:

```java
public class PinningDemo {
    private final Object lock = new Object();
    
    public void blockingInSynchronized() throws InterruptedException {
        synchronized (lock) {
            Thread.sleep(100);   // pinning
        }
    }
    
    public void blockingWithReentrantLock() throws InterruptedException {
        ReentrantLock rl = new ReentrantLock();
        rl.lock();
        try {
            Thread.sleep(100);   // OK — unmount eder
        } finally {
            rl.unlock();
        }
    }
}
```

100 virtual thread'le iki versiyonu çağır, toplam süreyi ölç:

```bash
java -Djdk.tracePinnedThreads=full ...
```

Pinning olduğunda log'da uyarı. **Defterine** sonucu yaz.

### Task 3.7.3 — JFR ile pinning yakalama (30 dk)

```bash
java -XX:StartFlightRecording=duration=60s,filename=pinning.jfr \
     -XX:+UnlockDiagnosticVMOptions \
     -jar app.jar
```

Bir akış sırasında pinning üret, sonra:

```bash
jfr print --events jdk.VirtualThreadPinned pinning.jfr
```

Stack trace'i incele, hangi `synchronized` blok'unun pin'lediğini bul.

### Task 3.7.4 — Spring Boot virtual thread enable (30 dk)

`core-banking` projende:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Bir HTTP endpoint çağır, controller'da:

```java
log.info("Thread: {}, isVirtual: {}", 
    Thread.currentThread().getName(), 
    Thread.currentThread().isVirtual());
```

Log'da `isVirtual=true` olduğunu doğrula.

### Task 3.7.5 — Parallel FX rate fetch (45 dk)

`FxRateService` yaz, 3 farklı provider'dan paralel kur çek (mock):

```java
public Money convert(Money source, Currency target) {
    try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
        List<Future<FxRate>> futures = providers.stream()
            .map(p -> exec.submit(() -> p.getRate(source.currency(), target)))
            .toList();
        
        // İlk başarılı sonuç
        for (Future<FxRate> f : futures) {
            try { return source.convert(f.get(1, TimeUnit.SECONDS), target); }
            catch (Exception e) { /* continue */ }
        }
    }
    throw new NoFxProviderAvailableException();
}
```

Hemen sonra Topic 3.8'de bunu **StructuredTaskScope** ile yeniden yazacağız — daha temiz.

### Task 3.7.6 — DB connection pool + virtual thread (30 dk)

`core-banking`'te HikariCP `maximumPoolSize=5` yap. 100 paralel HTTP request gönder (Gatling veya `ab`). Log'da:
- Connection wait time gözle
- Throughput ölç

Sonra `maximumPoolSize=50`'ye çıkar, tekrar dene. Farkı **defterine** yaz.

---

## Test yazma rehberi

### Test 3.7.1 — Virtual thread isVirtual assertion

```java
@Test
void virtualThreadShouldReportIsVirtual() throws InterruptedException {
    AtomicBoolean isVirtual = new AtomicBoolean();
    CountDownLatch done = new CountDownLatch(1);
    
    Thread.startVirtualThread(() -> {
        isVirtual.set(Thread.currentThread().isVirtual());
        done.countDown();
    });
    
    done.await();
    assertThat(isVirtual).isTrue();
}

@Test
void platformThreadShouldNotBeVirtual() throws InterruptedException {
    AtomicBoolean isVirtual = new AtomicBoolean(true);
    Thread t = new Thread(() -> {
        isVirtual.set(Thread.currentThread().isVirtual());
    });
    t.start();
    t.join();
    assertThat(isVirtual).isFalse();
}
```

### Test 3.7.2 — Pinning indirect test (timing)

```java
@Test
void synchronizedShouldPinVirtualThread() throws InterruptedException {
    int taskCount = 100;
    long sleepMs = 100;
    
    // Platform thread baseline
    long start1 = System.currentTimeMillis();
    runTasksSynchronized(taskCount, sleepMs, true);    // virtual
    long virtualSyncTime = System.currentTimeMillis() - start1;
    
    long start2 = System.currentTimeMillis();
    runTasksWithLock(taskCount, sleepMs, true);        // virtual, ReentrantLock
    long virtualLockTime = System.currentTimeMillis() - start2;
    
    // Virtual + lock should be much faster (concurrency yüksek, pinning yok)
    assertThat(virtualLockTime).isLessThan(virtualSyncTime / 2);
}
```

### Test 3.7.3 — Massive concurrency

```java
@Test
void virtualThreadsShouldHandleMassiveConcurrency() throws Exception {
    int taskCount = 10_000;
    CountDownLatch done = new CountDownLatch(taskCount);
    AtomicInteger completed = new AtomicInteger();
    
    try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < taskCount; i++) {
            exec.submit(() -> {
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                completed.incrementAndGet();
                done.countDown();
            });
        }
        boolean finished = done.await(5, TimeUnit.SECONDS);
        assertThat(finished).isTrue();
    }
    
    assertThat(completed).hasValue(taskCount);
}
```

### Test 3.7.4 — Spring Boot virtual thread integration

```java
@SpringBootTest
@AutoConfigureMockMvc
class VirtualThreadControllerTest {
    
    @Autowired MockMvc mockMvc;
    
    @Test
    void requestShouldRunOnVirtualThread() throws Exception {
        mockMvc.perform(get("/v1/accounts/_thread-info"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.virtual").value(true));
    }
}

// Controller'da:
@GetMapping("/_thread-info")
Map<String, Object> threadInfo() {
    Thread t = Thread.currentThread();
    return Map.of("name", t.getName(), "virtual", t.isVirtual());
}
```

---

## Claude-verify prompt

```
Aşağıdaki Java virtual thread kodumu banking-grade kriterlere göre değerlendir. 
Sadece eksikleri ve yanlışları işaretle:

1. Virtual thread oluşturma:
   - `Executors.newVirtualThreadPerTaskExecutor()` veya `Thread.ofVirtual()` 
     kullanılmış mı?
   - Virtual thread'leri pool'lamaya çalışılmış mı? (Olmamalı)
   - Try-with-resources içinde ExecutorService kullanılmış mı?

2. Pinning kontrolü:
   - `synchronized` blok içinde blocking call (sleep, I/O, lock) var mı?
   - `ReentrantLock` tercih edilmiş mi virtual thread çağrılan kodda?
   - `jdk.tracePinnedThreads=full` ile test edilmiş mi?
   - JFR `jdk.VirtualThreadPinned` event'i analiz edilmiş mi?

3. CPU-bound vs I/O-bound:
   - Virtual thread CPU-bound iş için kullanılmış mı? (Olmamalı — ForkJoinPool tercih)
   - I/O-bound iş için doğru kullanılmış mı?

4. ScopedValue:
   - ThreadLocal yerine ScopedValue tercih edilmiş mi modern code'da?
   - ScopedValue.where().run() pattern'i doğru mu?
   - `--enable-preview` flag eklenmiş mi (Java 21'de)?

5. Spring Boot integration:
   - `spring.threads.virtual.enabled: true` config'i eklenmiş mi?
   - Manual override gerekirse `AsyncTaskExecutor` yapılandırılmış mı?

6. Banking pattern'ler:
   - Per-request virtual thread (Tomcat) etkin mi?
   - Massive parallel I/O için virtual thread fan-out kullanılmış mı?
   - DB connection pool + virtual thread mismatch'i (10k vt, 10 connection) Semaphore 
     veya benzeri ile yönetilmiş mi?

7. Anti-pattern:
   - `Executors.newFixedThreadPool` virtual thread yerine kullanılmış mı?
   - CPU-bound iş virtual thread'e gönderilmiş mi?
   - ThreadLocal cleanup unutulmuş mu?
   - `synchronized` + JDBC call kombinasyonu var mı?

8. Test:
   - Massive concurrency (10k) test edilmiş mi?
   - `Thread.currentThread().isVirtual()` ile virtual thread doğrulanmış mı?
   - Pinning timing comparison test'i var mı?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `Thread.startVirtualThread` veya `newVirtualThreadPerTaskExecutor` kullanıyorum
- [ ] `synchronized` blok içinde JDBC call yapmıyorum (pinning yok)
- [ ] `ReentrantLock`'a geçtim virtual thread çağrılan kodda
- [ ] JFR ile bir pinning olayını canlı yakaladım
- [ ] `jdk.tracePinnedThreads=full` ile uyarıları gördüm
- [ ] `core-banking`'te `spring.threads.virtual.enabled=true`
- [ ] 10k concurrent virtual thread denedim, sistem kalktı
- [ ] CPU-bound vs I/O-bound iş ayrımını yapıyorum
- [ ] ScopedValue ile request context taşıdım (en az bir örnek)
- [ ] DB connection pool sınırını virtual thread sayısıyla ölçeklemeden tutuyorum (Semaphore)

---

## Defter notları

1. "Platform thread'in maliyeti (stack, kernel overhead): ____."
2. "Virtual thread - carrier ilişkisi (mount/unmount): ____."
3. "Pinning nedir, hangi durumlarda olur (3 sebep): ____."
4. "Pinning'i JFR ile nasıl yakalarım: ____."
5. "`synchronized` yerine `ReentrantLock` neden virtual thread için iyi: ____."
6. "ScopedValue vs ThreadLocal (3 fark): ____."
7. "Virtual thread CPU-bound iş için neden uygun değil: ____."
8. "Spring Boot 3.2+ virtual thread enable etmek için tek satır config: ____."
9. "10k virtual thread + 10 DB connection mismatch'i nasıl çözerim: ____."
10. "Java 21 vs Java 23+ virtual thread + synchronized davranış farkı: ____."
