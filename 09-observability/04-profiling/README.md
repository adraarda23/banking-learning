# Topic 9.4 — Profiling: JFR + async-profiler + Flame Graphs

## Hedef

Production-safe profiling banking serverlerda: JVM Flight Recorder (JFR), async-profiler, flame graph okuma. CPU hotspot, allocation hotspot, lock contention bulma. Continuous profiling (Pyroscope, Parca, Datadog Profiler) tools. Banking case studies (slow transfer, high CPU, GC pressure).

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 9.1-9.3 bitti
- JVM temel (heap, stack, GC duy)
- Thread, lock kavramı

---

## Kavramlar

### 1. Profiling vs Metrics

| | Metrics | Profiling |
|---|---|---|
| Granularity | Aggregate (timer p99) | Per-stack-frame |
| Use case | "Slow var mı?" | "Neresi slow?" |
| Cost | Cheap | Medium |
| Production | Always-on | On-demand or continuous |
| Question | "What" | "Why" |

Metric: "Transfer p99 latency 1.5s". Profile: "1.2s'i `BigDecimal.divide()` çağrısında geçiyor, ~80% CPU `MathContext` setup."

### 2. Sampling vs Instrumentation

**Sampling profiler:**
- Periodic snapshot (~10-100 Hz)
- Düşük overhead (1-3%)
- Production-safe
- async-profiler, JFR, Datadog Profiler

**Instrumentation profiler:**
- Bytecode injection
- Her method enter/exit kaydet
- Yüksek overhead (5-50%)
- Production'da risky
- VisualVM (instrumentation mode), YourKit

Banking pratiği: **Sampling** always; instrumentation sadece dev/CI.

### 3. JFR — Java Flight Recorder

JDK built-in. Low overhead. JDK 11+ free.

```bash
# Inline (when starting JVM)
java -XX:StartFlightRecording=duration=60s,filename=app.jfr,settings=profile \
     -jar app.jar

# Attach to running JVM
jcmd <pid> JFR.start name=banking duration=60s filename=app.jfr settings=profile

# Stop and dump
jcmd <pid> JFR.dump name=banking filename=app.jfr
jcmd <pid> JFR.stop name=banking
```

Settings:
- `default` — low overhead, continuous (~0.5%)
- `profile` — detailed, short-term (~2-3%)

**JFR events:**
- CPU usage
- Method profiling (sampling)
- Allocation (in TLAB)
- GC pause + stats
- Lock contention
- I/O (file, socket)
- Class loading
- Exception throw
- Network
- Thread state

Banking için JFR sürekli on (default settings) — incident'te dump.

### 4. JFR analysis — JDK Mission Control (JMC)

JMC = JFR analiz UI. Aspect'ler:
- Method Profiling (flame graph)
- Memory (TLAB allocations)
- GC (pauses, throughput)
- Thread (state, contention)
- I/O
- Exceptions
- Hot Classes

```
JMC opens app.jfr
  ├── Java Application
  │   ├── Method Profiling (hot methods)
  │   ├── Memory (allocation rate, TLAB)
  │   ├── GC (pause, throughput, generation)
  │   ├── Code (compilation, deoptimization)
  │   └── Threads (state, latency)
  └── JVM Internals
```

Banking örnek workflow:
```
1. Alert: p99 latency spike
2. jcmd <pid> JFR.dump filename=spike.jfr
3. Open JMC → Method Profiling
4. Hot method: BigDecimalUtil.normalize (40% CPU)
5. Drill down: MathContext allocation per call (anti-pattern)
6. Fix: Single MathContext static instance
```

### 5. async-profiler — superior alternative

Production'da daha popüler. Sampling, low overhead, CPU + alloc + lock + wall-clock + cache miss.

```bash
# Download
curl -L -o async-profiler.tar.gz \
  https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xf async-profiler.tar.gz

# Run
./profiler.sh -d 30 -f profile.html <pid>

# CPU profile
./profiler.sh -d 60 -e cpu -f cpu.html <pid>

# Allocation profile
./profiler.sh -d 60 -e alloc -f alloc.html <pid>

# Lock contention
./profiler.sh -d 60 -e lock -f lock.html <pid>

# Wall-clock (sees blocked threads too — banking I/O)
./profiler.sh -d 60 -e wall -f wall.html <pid>

# Continuous (background)
./profiler.sh -d 300 -f $(date +%s).html <pid>
```

**Wall-clock profiling** banking için **özellikle değerli** — I/O blocked (DB query, external API) görür. CPU profile sadece running thread'leri.

### 6. Flame graph reading

Flame graph = stack frame y-axis, time x-axis.

```
         [main]
        /      \
   [doWork]  [other]
    /   \
[step1] [step2]
   |       |
[parse] [compute]
   |
[regex.match]    ← width = time spent here
```

**Genişlik = zaman / CPU**. Üstte ne kadar genişse, o kadar fazla zaman.

**Banking örnek:**

```
[Tomcat:exec-5]
  → SpringDispatcher.doDispatch
    → TransferController.transfer
      → TransferService.transfer
        → AccountRepository.findById                   30%
          → Hibernate query                            20%
          → BigDecimal.setScale                        10%   ← WAIT, why setScale here?
        → BalanceCalculator.calculate                  40%
          → BigDecimal.divide                          25%
            → MathContext.<init>                       15%   ← Hot allocation!
            → division                                 10%
        → KafkaProducer.send                           20%
          → Serializer.serialize                       15%
```

`MathContext.<init>` 15% = hot allocation per call → fix.

### 7. CPU profiling banking patterns

**Pattern 1: Hot method**

```
BigDecimalUtil.format     40% CPU
  ↓ called from
TransferController, AccountController, ...
```

Cache result veya hot path'i optimize.

**Pattern 2: Excessive logging**

```
Logback.encode               25% CPU
  ↓
TransferService.log.debug("...", expensiveToString())
```

Debug level production'da OFF (Topic 9.1).

**Pattern 3: Inefficient SQL**

```
Hibernate ResultSet.next()    35% CPU
```

→ N+1 query. EXPLAIN PLAN, JOIN FETCH (Phase 3).

**Pattern 4: Serialization**

```
Jackson.serialize           20% CPU
  └── BigDecimal.toPlainString    8%
```

→ Custom serializer for BigDecimal banking.

### 8. Allocation profiling

Allocation hotspot → GC pressure → pause time.

```bash
./profiler.sh -e alloc -d 60 -f alloc.html <pid>
```

**Banking örnek:**

```
[TransferService.transfer]
  → BigDecimal.<init>(String)          200 MB/s allocation
  → ArrayList.<init>(emptyList)        50 MB/s
  → String.<init>                      150 MB/s
```

200 MB/s BigDecimal allocation → young gen fills fast → GC frequent.

**Fix:**
- BigDecimal.valueOf (cached small values)
- StringBuilder reuse
- Object pool (banking için nadiren — usually OK)

### 9. Lock contention profiling

```bash
./profiler.sh -e lock -d 60 -f lock.html <pid>
```

**Banking örnek:**

```
ReentrantLock@xyz   500 ms contention
  ↓ holder
  AccountService.transfer (held for 200 ms)
  ↓ waiters
  20 threads, avg wait 50 ms
```

Bottleneck = AccountService.transfer locks too widely.

**Fix:** Lock granularity reduce, optimistic locking (Phase 3.6).

### 10. Continuous profiling — Pyroscope / Parca / Datadog

Always-on profiling. Per-second sample stored. Compare time windows.

```yaml
# Pyroscope agent
- PYROSCOPE_APPLICATION_NAME=transfer-service
- PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
- PYROSCOPE_LOG_LEVEL=debug
- PYROSCOPE_PROFILER_EVENT=itimer    # CPU
```

Java agent attach:
```bash
java -javaagent:pyroscope.jar -jar app.jar
```

**Advantages:**
- Yesterday baseline vs today
- Deploy artifact differential
- A/B test perf comparison
- Continuous = no need to wait for incident

Banking için Pyroscope / Parca yaygın choice.

### 11. Banking case study — slow transfer

**Initial signal:** Transfer p99 1.5s (SLO: 500ms).

**Step 1: Metrics — narrowing down**
```promql
histogram_quantile(0.99, 
  rate(banking_transfer_duration_seconds_bucket[5m]))
# = 1.5s
```

**Step 2: Trace — drill down**

Exemplar trace:
```
[banking.transfer]                  1.5s
  ├── account_lookup                100 ms
  ├── balance_check                  50 ms
  ├── limit_check                   200 ms
  ├── fraud_score (external)        800 ms   ← culprit
  └── kafka_publish                 100 ms
```

**Step 3: Profile fraud-service**

```bash
jcmd <pid> JFR.start duration=60s filename=fraud.jfr settings=profile
```

JFR → method profiling:
```
[FraudController.score]
  → FraudRules.evaluate                70% CPU
    → MlModel.predict                  50% CPU
      → DenseMatrix.multiply           45% CPU   ← hot
```

**Step 4: Allocation profile**

```
DenseMatrix.<init>(int, int)    500 MB/s   ← new matrix per request
```

Per-request **fresh matrix allocation** → GC pressure → CPU + pause time.

**Step 5: Fix**

- Reuse matrix (object pool)
- Model warmup on startup
- Cache results (5-min TTL for same user)

**Step 6: Verify**

Re-profile after deploy. p99 200 ms ✓.

### 12. JVM tuning hints from profiling

**Symptom: Frequent GC**
- Allocation rate yüksek → reduce allocations
- Heap küçük → `-Xmx` artır
- Young gen küçük → ratio config

**Symptom: Long GC pauses (>1s)**
- G1GC tune: `-XX:MaxGCPauseMillis=200`
- Or switch ZGC / Shenandoah for low-pause
- Heap çok büyük → split application

**Symptom: High CPU sustained**
- Profile CPU
- Check infinite loop / thread spin
- Excessive logging

**Symptom: High allocation**
- Allocation profile
- Common: BigDecimal, String, primitive box

**Symptom: Lock contention**
- Lock profile
- Granularity reduce
- Lock-free data structure

### 13. Banking — profiling anti-pattern'leri

**Anti-pattern 1: Instrumenting profiler production'da**

Latency spike + memory bloat. Async-profiler / JFR sampling kullan.

**Anti-pattern 2: Sampling süre çok kısa**

5 saniye = statistical noise. Banking min 30-60 sn.

**Anti-pattern 3: Single JVM profile aggregate olarak yorumlamak**

Multi-tenant / multi-traffic-pattern → single profile yanıltır. Per-tenant profile veya per-endpoint.

**Anti-pattern 4: Profile production'da unattended**

JFR dosyası disk doldurabilir. Rotate + limit.

**Anti-pattern 5: Profile sensitive data leak**

JFR exception stack trace + arg → PII içerebilir. Banking için profile dosyaları **sıkı erişim** (S3 + IAM + encryption).

**Anti-pattern 6: Wall-clock profile yerine sadece CPU**

Banking I/O-heavy (DB, external API). Wall-clock görür blocked time. CPU profile blocked thread'leri kaçırır.

**Anti-pattern 7: Optimize before measure**

"Bence String yerine StringBuilder olur" → profile çıkmadan değişiklik. Önce ölç.

**Anti-pattern 8: Profile dev environment**

Dev workload != prod workload. Hot path farklı. Staging veya prod (controlled).

**Anti-pattern 9: Compare profiles different load**

Karşılaştırma yaparken aynı load. JMH bunun için (Topic 9.6).

**Anti-pattern 10: Continuous profiling cost ignore**

Pyroscope storage + retention. Banking için tier (daily detail, weekly aggregate).

---

## Önemli olabilecek araştırma kaynakları

- JDK Flight Recorder docs
- JDK Mission Control
- async-profiler GitHub
- Brendan Gregg flame graph (origin)
- Pyroscope docs
- Datadog Continuous Profiler
- "Java Performance" — Scott Oaks

---

## Mini task'ler

### Task 9.4.1 — JFR continuous recording (30 dk)

Spring Boot app `-XX:StartFlightRecording=settings=default,maxage=1h,filename=...` ile başlat. JMC ile aç.

### Task 9.4.2 — Hot CPU method JFR (45 dk)

Banking app'e BigDecimal-heavy compute endpoint. JFR profile çek (60 sn). JMC method profiling hot path identify.

### Task 9.4.3 — async-profiler CPU flame graph (45 dk)

Same scenario. async-profiler `-e cpu -d 60 -f cpu.html`. Browser'da flame graph aç. JFR ile compare.

### Task 9.4.4 — Allocation profiling (45 dk)

`-e alloc -d 60 -f alloc.html`. Banking BigDecimal allocation hotspot bul.

### Task 9.4.5 — Lock contention (45 dk)

Multi-thread test scenario (synchronized block). `-e lock -d 60`. Contention point.

### Task 9.4.6 — Wall-clock profile (30 dk)

External API call mock 200ms delay. `-e wall -d 60`. CPU profile'da görünmeyen blocked time.

### Task 9.4.7 — Pyroscope / Parca local (60 dk)

Pyroscope docker. Banking app agent attach. UI'da continuous profile timeline.

### Task 9.4.8 — Diff profile (45 dk)

Pre-optimization profile. Apply fix (cache result, static instance). Post-profile. Pyroscope diff view.

### Task 9.4.9 — JFR + Mission Control GC analysis (45 dk)

GC pause distribution. Long pause events. G1GC tune `-XX:MaxGCPauseMillis=200`.

### Task 9.4.10 — Production incident simulation (60 dk)

Slow endpoint senaryo. Metric alert → trace exemplar → JFR dump → root cause → fix.

---

## Test yazma rehberi

```java
@Test
void shouldNotHaveExcessiveBigDecimalAllocation() throws Exception {
    // Run profiling via java agent or AsyncProfiler API
    AsyncProfiler profiler = AsyncProfiler.getInstance();
    profiler.start("alloc", 1_000_000);   // 1ms sampling
    
    for (int i = 0; i < 10_000; i++) {
        transferService.transfer(testRequest);
    }
    
    String output = profiler.stop();
    
    // Parse output, look for BigDecimal allocation rate
    long bigDecimalBytes = parseAllocFor(output, "java.math.BigDecimal");
    assertThat(bigDecimalBytes).isLessThan(100_000_000);   // < 100 MB
}

@Test
void noLockContentionOnHotPath() throws Exception {
    AsyncProfiler profiler = AsyncProfiler.getInstance();
    profiler.start("lock", 1_000_000);
    
    ExecutorService pool = Executors.newFixedThreadPool(50);
    for (int i = 0; i < 1000; i++) {
        pool.submit(() -> transferService.transfer(testRequest));
    }
    pool.shutdown();
    pool.awaitTermination(60, TimeUnit.SECONDS);
    
    String output = profiler.stop();
    
    long lockContention = parseLockFor(output, "TransferService");
    assertThat(lockContention).isLessThan(100_000_000);   // < 100ms aggregate
}
```

---

## Claude-verify prompt

```
Profiling stack'imi banking-grade kriterlere göre değerlendir:

1. Tooling:
   - JFR continuous (default settings)?
   - async-profiler available?
   - Continuous profiling (Pyroscope/Parca/Datadog)?

2. Profile types:
   - CPU profile?
   - Allocation profile?
   - Lock contention profile?
   - Wall-clock (banking I/O için kritik)?

3. Banking analysis:
   - Hot method identification?
   - Allocation rate (BigDecimal, String)?
   - Lock contention (transfer service)?
   - GC pause distribution?

4. Workflow:
   - Metric alert → trace → profile → root cause flow?
   - Pre/post-fix differential profile?

5. Production-safe:
   - Sampling overhead < 3%?
   - JFR sürekli on (default settings)?
   - Profile dump on incident automated?

6. Banking domain checks:
   - BigDecimal allocation hotspot tarandı?
   - Logging overhead (Logback, JSON serialize)?
   - DB query hot path (Hibernate)?
   - Kafka serialization hot path?

7. JVM tuning:
   - G1GC `MaxGCPauseMillis` set?
   - Heap size + young gen ratio?
   - GC log enabled?
   - ZGC / Shenandoah considered for low-pause?

8. Anti-pattern:
   - Instrumentation profiler prod YOK?
   - Sampling < 30 sn YOK?
   - PII profile leak (S3 IAM + encryption)?
   - Compare profiles different load YOK?
   - Optimize before measure YOK?

9. Documentation:
   - 1 case study walkthrough (slow transfer → fixed)?
   - Runbook: how to dump JFR on prod?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] JFR continuous on
- [ ] JMC ile JFR analiz pratiği
- [ ] async-profiler CPU + alloc + lock + wall flame graph
- [ ] Pyroscope/Parca continuous setup
- [ ] Banking hot path BigDecimal allocation profile
- [ ] Lock contention scenario reproduce + profile
- [ ] Pre/post-fix differential profile (Pyroscope)
- [ ] 1 production-like incident workflow walkthrough
- [ ] Runbook: JFR dump on prod
- [ ] 2+ integration test (allocation rate, no lock contention)

---

## Defter notları (10 madde)

1. "Sampling vs instrumentation profiler banking pick + overhead: ____."
2. "JFR continuous (default) + on-demand profile dump workflow: ____."
3. "async-profiler 4 mode (cpu, alloc, lock, wall) banking kullanım: ____."
4. "Flame graph okuma — genişlik = zaman, alt-üst stack: ____."
5. "Wall-clock profile (I/O blocked görünür) banking sebebi: ____."
6. "Continuous profiling (Pyroscope/Parca) baseline vs today diff: ____."
7. "BigDecimal allocation banking hotspot + fix patterns (cache, valueOf): ____."
8. "Lock contention reduce — granularity + optimistic locking: ____."
9. "JVM tuning from profile (G1GC pause target, ZGC low-pause): ____."
10. "Production incident workflow (alert → trace → profile → fix → verify): ____."
