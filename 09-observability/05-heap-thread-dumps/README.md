# Topic 9.5 — Heap & Thread Dump Analysis

## Hedef

Production troubleshooting derinliği: heap dump alma + Eclipse MAT analizi, thread dump alma + jstack / FastThread.io, GC log analizi. Banking memory leak / deadlock / OOM / livelock incident senaryoları. JVM internals (Eden, Survivor, Old gen, Metaspace). 

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 9.4 (Profiling) bitti
- JVM heap model temel
- Thread state (Runnable, Blocked, Waiting, Timed_Waiting, Terminated)

---

## Kavramlar

### 1. JVM memory layout

```
Java Process Memory
├── Heap (managed)
│   ├── Young Generation
│   │   ├── Eden            (yeni allocate)
│   │   ├── S0 (Survivor)   (1. GC sonrası)
│   │   └── S1 (Survivor)
│   └── Old Generation       (long-lived)
├── Metaspace                (class metadata)
├── Code Cache               (JIT compiled)
├── Native Heap              (JNI, direct ByteBuffer)
├── Thread Stacks            (per thread ~1 MB)
├── Native Memory Tracking   (JVM internal)
└── Off-heap                 (Netty, Kafka direct)
```

**Banking common issues:**
- OutOfMemoryError: Java heap space → heap leak
- OOM: Metaspace → class leak
- OOM: GC overhead limit → GC thrashing
- OOM: Direct buffer memory → Netty / Kafka native leak
- StackOverflowError → deep recursion

### 2. Heap dump — when, how

**When to dump:**
- OOM error
- Memory growth pattern (gradual leak)
- Suspected leak (sustained high heap usage)
- Cache exploration

**How to dump:**

```bash
# jmap (alive heap)
jmap -dump:live,format=b,file=heap.hprof <pid>

# jcmd (modern alternative)
jcmd <pid> GC.heap_dump filename=heap.hprof

# Auto-dump on OOM (production setup)
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/dumps/heap-%p-%t.hprof \
     -jar app.jar
```

**Banking pratiği:** Production'da `HeapDumpPath` mounted volume (PVC). OOM → dump → pod restart → ops investigate.

**Size:** Banking app heap dump genelde 1-8 GB. Transfer download.

### 3. Eclipse MAT — analysis

Memory Analyzer Tool (Eclipse, free).

```
Open heap.hprof
  ├── Overview (largest objects, dominator tree)
  ├── Histogram (class count + size)
  ├── Dominator Tree (object retention)
  ├── Leak Suspects Report  ← automated guess
  └── Top Consumers
```

**Banking case study — caching leak:**

```
Heap usage: 4 GB

Leak Suspects:
  Problem 1: 3.5 GB (87%) accumulated in HashMap
    Path: GlobalCache.cache → HashMap → ...
```

Dominator Tree shows `GlobalCache.cache` holds 3.5 GB.

Histogram by class:
```
java.util.HashMap$Node    50,000,000 instances    1.2 GB
com.bank.Account          25,000,000 instances    2.0 GB
java.lang.String          ...
```

→ `GlobalCache` `Account` objelerini accumulate ediyor. **No eviction** → memory grows.

**Fix:** TTL-based eviction (Caffeine), size limit, weak references for cache values.

### 4. Common Java leak patterns

#### Pattern 1: Static collection growing

```java
public class GlobalCache {
    private static final Map<UUID, Account> CACHE = new HashMap<>();   // ❌ unbounded
}
```

**Fix:** Caffeine / EHCache with maximumSize + expireAfterWrite.

#### Pattern 2: ThreadLocal leak

```java
public class Service {
    private static final ThreadLocal<UserContext> CONTEXT = new ThreadLocal<>();
    
    public void doWork(UserContext ctx) {
        CONTEXT.set(ctx);
        // ... no remove
    }
}
```

Thread pool reuse → old context lingers → user objects retained.

**Fix:** Always `CONTEXT.remove()` in finally.

#### Pattern 3: Inner class capturing outer

```java
public class Service {
    private final HugeData data;
    
    public Runnable task() {
        return () -> System.out.println("hi");   // captures Service.this → HugeData
    }
}
```

`Runnable` instance pinned → `Service` pinned → `HugeData` pinned.

#### Pattern 4: Cache without eviction

```java
@Cacheable("accounts")   // Spring default = ConcurrentHashMap unbounded
public Account getAccount(UUID id) { ... }
```

**Fix:** Caffeine cache manager + maximumSize + expireAfterWrite.

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager mgr = new CaffeineCacheManager();
    mgr.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(15)));
    return mgr;
}
```

#### Pattern 5: Connection not closed

```java
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(...);
// ❌ no close → connection pinned + result set in memory
```

**Fix:** try-with-resources. (HikariCP throws warnings if leak.)

#### Pattern 6: Stream not closed

```java
Stream<String> lines = Files.lines(path);
lines.forEach(...);   // ❌ underlying FileChannel pinned
```

**Fix:** try-with-resources.

#### Pattern 7: Listener not unregistered

```java
eventBus.register(this);
// ❌ no unregister → this pinned forever
```

**Fix:** unregister in @PreDestroy / lifecycle hook.

### 5. Thread dump — when, how

**When to dump:**
- CPU 100% (find thread doing it)
- Application hung (find stuck thread)
- Deadlock suspected
- Slow performance investigation

```bash
# jstack
jstack <pid> > thread.dump

# Multiple samples (good practice)
for i in 1 2 3; do
  jstack <pid> > thread-$i.dump
  sleep 5
done

# jcmd
jcmd <pid> Thread.print > thread.dump

# kill -3 (Linux/Mac, dump to stdout/log)
kill -3 <pid>
```

### 6. Thread dump anatomy

```
"http-nio-8080-exec-5" #45 daemon prio=5 os_prio=0 tid=0x7f3b3c012800 nid=0x4242 RUNNABLE
   java.lang.Thread.State: RUNNABLE
     at java.util.regex.Pattern$BmpCharProperty.match(Pattern.java:3962)
     at java.util.regex.Pattern$Loop.match(Pattern.java:4807)
     at com.bank.IbanValidator.validate(IbanValidator.java:45)
     at com.bank.TransferService.transfer(TransferService.java:78)
     ...
   Locked ownable synchronizers:
     - <0x000000076b8a9b48> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
```

**Sections:**
- Thread name + ID + priority + native ID
- **State:** RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED
- Stack trace
- Locks (held, waiting for)

### 7. Thread states — banking interpretation

| State | Anlam | Banking örnek |
|---|---|---|
| RUNNABLE | CPU üzerinde / IO yapıyor | DB query, network I/O, compute |
| BLOCKED | Monitor lock bekleniyor | Synchronized contention |
| WAITING | Object.wait, LockSupport.park | Future.get, thread pool idle |
| TIMED_WAITING | sleep, wait(timeout) | Sleep, retry wait |
| TERMINATED | Bitmiş | - |

**Banking signal:**
- All `http-nio-*` threads RUNNABLE in DB code → DB slow
- Many threads BLOCKED on same monitor → lock contention
- Threads WAITING forever → deadlock veya stuck future

### 8. Deadlock detection

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x...x... (object 0x...x..., a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x...x... (object 0x...x..., a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
        at com.bank.AccountService.method1(AccountService.java:50)
        - waiting to lock <0x000000076b8a9c00> (a java.lang.Object)
        - locked <0x000000076b8a9bd0> (a java.lang.Object)
"Thread-2":
        at com.bank.AccountService.method2(AccountService.java:75)
        - waiting to lock <0x000000076b8a9bd0> (a java.lang.Object)
        - locked <0x000000076b8a9c00> (a java.lang.Object)
```

JVM otomatik deadlock detect (Java-level). **Banking saatten saate genelde lock ordering ihlali.**

**Fix:** Lock ordering rule (her zaman aynı sırada lock al).

```java
// ❌ Inconsistent lock order
public void transfer(Account from, Account to) {
    synchronized (from) {
        synchronized (to) { ... }
    }
}

// ✓ Lock ordering (canonical order)
public void transfer(Account a, Account b) {
    Account first = a.getId().compareTo(b.getId()) < 0 ? a : b;
    Account second = a.getId().compareTo(b.getId()) < 0 ? b : a;
    synchronized (first) {
        synchronized (second) { ... }
    }
}
```

### 9. Thread dump analysis tools

#### jstack — base
Text dump.

#### FastThread.io
Browser upload → analiz report:
- Thread state breakdown chart
- Deadlock detection
- Repeating stack patterns
- Thread pool exhaustion check
- High CPU thread identification

Banking ops için **standard tool**.

#### IBM Thread Dump Analyzer (TDA)

#### Async-profiler (Topic 9.4) thread states profile

### 10. GC log analysis

Banking için GC log **always on**:

```bash
java -Xlog:gc*:file=/var/log/banking/gc.log:time,uptime,level,tags:filecount=10,filesize=100M \
     -jar app.jar
```

GC log entry:
```
[2024-05-12T10:30:45.123+0000][0.500s][info][gc] GC(0) Pause Young (Normal) 256M->128M(1024M) 12.345ms
```

**Tools:**
- **GCViewer** (open source) — visual analysis
- **GCEasy.io** (online) — upload + report
- **JMC GC tab**

**Banking interpretation:**

```
GCEasy report:
- Throughput: 98.5%   (target > 95%)
- Avg pause: 25 ms
- Max pause: 180 ms   (target p99 < 200 ms)
- Old gen growth: 5 MB/min   (concerning if grows forever)
- Full GC count: 2 in 24h (acceptable)
```

**Red flags:**
- Throughput < 90%
- p99 pause > 500 ms
- Full GC frequent
- Old gen never shrinks (leak)

### 11. JVM tuning — G1GC banking

```bash
java \
  -Xms4g -Xmx4g \                          # Same min/max (avoid resize pause)
  -XX:+UseG1GC \                            # G1 banking default
  -XX:MaxGCPauseMillis=200 \                # Target pause
  -XX:InitiatingHeapOccupancyPercent=45 \   # Start concurrent at 45%
  -XX:G1HeapRegionSize=16M \                # 16 MB regions
  -XX:+ParallelRefProcEnabled \
  -XX:+AlwaysPreTouch \                     # Touch heap on startup (avoid lazy alloc)
  -XX:+UseStringDeduplication \             # Reduce String memory
  -Xlog:gc*:file=...
```

**For low-latency banking (FAST payments):**

```bash
-XX:+UseZGC \                # Pause < 10 ms
-XX:+ZGenerational \
-Xms8g -Xmx8g
```

ZGC pause ≪ G1 ama throughput biraz düşük.

### 12. Native memory tracking

Off-heap leak (Netty direct buffer, Kafka, Compressor)?

```bash
java -XX:NativeMemoryTracking=summary -jar app.jar

jcmd <pid> VM.native_memory summary
```

Output:
```
Total: reserved=4524032KB, committed=1532812KB
-                  Java Heap (reserved=4194304KB, committed=1234567KB)
                            (mmap: reserved=4194304KB, committed=1234567KB)
-                     Class (reserved=1048576KB, committed=80520KB)
                            (classes #12345)
-                    Thread (reserved=204852KB, committed=204852KB)
                            (threads #100)
-                       GC  (...)
-                     Code  (...)
-                  Compiler (...)
-                  Internal (...)
-               Symbol      (...)
-    Native Memory Tracking (...)
-               Arena Chunk (...)
```

Banking common: **Direct buffer** (Netty, Kafka).

### 13. Banking case study — transfer service OOM

**Initial:**
```
Pod restart loop. Logs:
java.lang.OutOfMemoryError: Java heap space
```

**Step 1: Heap dump auto-collected**

`/var/dumps/heap-12345-2024-05-12.hprof` (3.8 GB).

**Step 2: MAT analysis**

```
Leak Suspects:
  Problem 1: 3.4 GB in ConcurrentHashMap retained by IdempotencyKeyCache
```

**Step 3: Code inspection**

```java
@Component
public class IdempotencyKeyCache {
    private final Map<String, IdempotencyRecord> cache = new ConcurrentHashMap<>();   // ❌ unbounded
    
    public void put(String key, IdempotencyRecord record) {
        cache.put(key, record);
    }
}
```

No eviction. Production: 10M transfer requests/day → 10M entries/day → fill heap in 4-5 days.

**Step 4: Fix**

```java
@Component
public class IdempotencyKeyCache {
    private final Cache<String, IdempotencyRecord> cache = Caffeine.newBuilder()
        .maximumSize(1_000_000)
        .expireAfterWrite(Duration.ofHours(24))   // 24h idempotency window
        .recordStats()
        .build();
    
    // Better: Use Redis (multi-instance shared, persistent)
}
```

Real fix → **Redis** for distributed idempotency (Topic 7.7 IdempotencyKeyFilter).

**Step 5: Post-fix verify**

Heap dump in steady state: 800 MB (was 3.5 GB).

### 14. Banking — heap/thread anti-pattern'leri

**Anti-pattern 1: Unbounded cache**

In-memory Map cache without eviction. **Caffeine + max + expire.**

**Anti-pattern 2: ThreadLocal.remove() unutmak**

Thread pool reuse → leak.

**Anti-pattern 3: Lock without timeout**

```java
lock.lock();   // ❌ no timeout
```

Deadlock = forever block.

```java
if (lock.tryLock(5, TimeUnit.SECONDS)) { ... }   // ✓
```

**Anti-pattern 4: synchronized on wide scope**

```java
public synchronized void transfer(...) { ... }   // Whole method
```

Contention. Narrow scope or finer lock.

**Anti-pattern 5: No timeout on Future.get**

```java
future.get();   // ❌ forever wait
future.get(5, TimeUnit.SECONDS);   // ✓
```

**Anti-pattern 6: GC log not enabled**

Production'da GC log şart. Incident'te tek bilgi kaynağı.

**Anti-pattern 7: -Xmx >> -Xms**

JVM lazy commit → GC time'da resize pause. Same min/max.

**Anti-pattern 8: HeapDumpOnOutOfMemoryError yok**

OOM olunca evidence yok. Always on.

**Anti-pattern 9: Production thread dump ignore**

Ops dump tutmuyor. Banking için periodic capture script.

**Anti-pattern 10: Tomcat thread pool yetersiz config**

```yaml
server.tomcat.threads.max: 200   # Default
```

Banking blocking I/O heavy → 200 yetersiz olabilir. Profile-based tune.

---

## Önemli olabilecek araştırma kaynakları

- Eclipse Memory Analyzer (MAT) docs
- FastThread.io
- GCEasy.io
- "Java Performance" — Scott Oaks
- "Optimizing Java" — Benjamin Evans
- Plumbr leak detector (commercial)

---

## Mini task'ler

### Task 9.5.1 — HeapDumpOnOutOfMemory setup (15 dk)

JVM flags `-XX:+HeapDumpOnOutOfMemoryError` + `-XX:HeapDumpPath=...`. K8s'de mounted volume.

### Task 9.5.2 — Intentional leak + heap dump (60 dk)

Unbounded Map cache code yaz. JMeter ile flood. `jmap -dump:live`. MAT ile aç.

### Task 9.5.3 — MAT analysis walkthrough (45 dk)

Leak Suspects report, dominator tree, histogram by class. "Account" class retention path.

### Task 9.5.4 — Deadlock reproduce + thread dump (60 dk)

İki account arası transfer'de lock ordering yanlış. 2 thread concurrent transfer → deadlock. `jstack` → "Found Java-level deadlock". Fix lock ordering.

### Task 9.5.5 — Lock contention via thread dump (45 dk)

50 thread concurrent transfer synchronized method. 3 sample jstack → çoğu BLOCKED on same monitor. Fix granularity.

### Task 9.5.6 — FastThread.io upload + report (30 dk)

Thread dump → fastthread.io. Report read.

### Task 9.5.7 — GC log + GCEasy (45 dk)

GC log enable. Load test ile data. GCEasy.io upload. Throughput + pause analysis.

### Task 9.5.8 — Native memory tracking (30 dk)

`NativeMemoryTracking=summary` + jcmd. Heap dışı memory.

### Task 9.5.9 — GC tuning experiment (60 dk)

G1GC default vs `MaxGCPauseMillis=100`. ZGC compare. Pause time impact.

### Task 9.5.10 — Production runbook write (30 dk)

Markdown: "How to diagnose OOM in production banking service". Steps + commands.

---

## Test yazma rehberi

```java
@Test
void shouldNotLeakOnRepeatedCalls() throws Exception {
    long beforeUsed = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
    
    for (int i = 0; i < 10_000; i++) {
        transferService.transfer(testRequest);
    }
    
    System.gc();
    Thread.sleep(1000);
    
    long afterUsed = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
    long growth = afterUsed - beforeUsed;
    
    assertThat(growth).isLessThan(100 * 1024 * 1024);   // < 100 MB growth
}

@Test
void shouldDetectDeadlockEarly() throws Exception {
    AtomicBoolean deadlockDetected = new AtomicBoolean(false);
    
    Thread monitor = new Thread(() -> {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        while (!Thread.currentThread().isInterrupted()) {
            long[] deadlocked = bean.findDeadlockedThreads();
            if (deadlocked != null && deadlocked.length > 0) {
                deadlockDetected.set(true);
                break;
            }
            try { Thread.sleep(100); } catch (InterruptedException e) { break; }
        }
    });
    monitor.setDaemon(true);
    monitor.start();
    
    // Run scenario that should NOT deadlock with correct fix
    runConcurrentTransfers();
    
    Thread.sleep(2000);
    monitor.interrupt();
    
    assertThat(deadlockDetected.get()).isFalse();
}
```

---

## Claude-verify prompt

```
Heap/Thread dump readiness'imi banking-grade kriterlere göre değerlendir:

1. JVM flags:
   - HeapDumpOnOutOfMemoryError on?
   - HeapDumpPath mounted volume (production)?
   - GC log enabled + rotation?
   - NativeMemoryTracking optional?

2. JVM tuning:
   - -Xms == -Xmx (no resize pause)?
   - G1GC + MaxGCPauseMillis target?
   - ZGC banking ultra-low-latency?
   - AlwaysPreTouch?
   - StringDeduplication?

3. Memory anti-pattern audit:
   - Unbounded cache (HashMap)?
   - Caffeine maximumSize + expireAfterWrite?
   - ThreadLocal.remove() finally?
   - Listener unregister @PreDestroy?
   - try-with-resources stream/connection?

4. Concurrency:
   - Lock ordering canonical (account compareTo)?
   - tryLock with timeout?
   - Future.get with timeout?
   - synchronized scope narrow?

5. Production troubleshooting:
   - JFR continuous on (Topic 9.4)?
   - Heap dump capture procedure?
   - Thread dump capture procedure?
   - GC log analysis (GCEasy / GCViewer)?
   - FastThread.io workflow?

6. Banking-specific:
   - Idempotency cache via Redis (not in-memory)?
   - Connection pool tuning (HikariCP)?
   - Tomcat thread pool tuning?
   - Direct buffer leak check (Netty/Kafka)?

7. Runbook:
   - OOM diagnosis steps documented?
   - Deadlock diagnosis documented?
   - GC tuning playbook?
   - Sample analysis case study?

8. Test:
   - No-leak regression test?
   - No-deadlock regression test?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] HeapDumpOnOOM + path mounted
- [ ] GC log enabled + rotation
- [ ] Intentional leak reproduce + MAT analiz
- [ ] Deadlock reproduce + jstack tespit + fix
- [ ] Lock contention reproduce + tear apart
- [ ] FastThread.io report walkthrough
- [ ] GCEasy report analysis
- [ ] Native memory tracking baseline
- [ ] G1GC tune experiment + ZGC compare
- [ ] Production runbook (OOM, deadlock)
- [ ] 2+ regression test

---

## Defter notları (10 madde)

1. "JVM memory layout (heap, metaspace, native, off-heap) banking issue map: ____."
2. "MAT Leak Suspects → Dominator Tree → Histogram banking workflow: ____."
3. "Common Java leak pattern (unbounded cache, ThreadLocal, inner class): ____."
4. "Thread state (RUNNABLE, BLOCKED, WAITING, TIMED_WAITING) banking interpretation: ____."
5. "Deadlock detect (jstack auto + ThreadMXBean) + lock ordering fix: ____."
6. "FastThread.io + thread pool exhaustion + repeating pattern: ____."
7. "GC log + GCEasy throughput + pause analysis banking targets: ____."
8. "G1GC vs ZGC banking trade-off (throughput vs pause): ____."
9. "Native memory tracking + Netty direct buffer leak: ____."
10. "Banking idempotency cache OOM case study + Redis fix: ____."
