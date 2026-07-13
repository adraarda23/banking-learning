# Topic 9.6 — JMH Deep & CI Integration

## Hedef

Java Microbenchmark Harness (JMH) banking-grade kullanımı: micro-benchmark design, mode selection, state scope, parameter sweep, profiler integration, JIT compilation behavior, CI integration ile performance regression detection. Banking-specific benchmark senaryoları (BigDecimal, locking, serialization).

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 9.4 (Profiling) bitti
- Java microbenchmark naive yaklaşımları neden bozuk biliyor olmak
- JIT compiler temel (HotSpot)

---

## Kavramlar

### 1. Niye JMH? Naive benchmark'lar neden bozuk?

```java
// ❌ Naive benchmark
public class BadBenchmark {
    public static void main(String[] args) {
        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            doSomething();
        }
        long elapsed = System.nanoTime() - start;
        System.out.println("Time: " + elapsed);
    }
}
```

**Sorunlar:**
1. **Dead code elimination** — JIT result kullanılmıyor görüyor → call'u optimize ediyor
2. **Constant folding** — JIT compile-time hesaplıyor
3. **JIT warmup** — first N call interpreted, sonra C1, sonra C2 → not steady state
4. **GC interference** — measurement window'da GC pause
5. **CPU frequency scaling** — turbo boost / thermal throttle
6. **OSR (On-Stack Replacement)** — loop iterations farklı optimize ediliyor
7. **Inlining heuristics** — caller context'e göre değişir
8. **Loop unrolling** — runtime farklı

JMH bunların hepsini handle eder.

### 2. JMH setup

```xml
<dependencies>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>1.37</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>1.37</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

Maven archetype (alternatif):
```bash
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=com.bank.bench \
  -DartifactId=banking-bench \
  -Dversion=1.0
```

### 3. JMH basic benchmark

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(value = 2, warmups = 1)
public class BigDecimalBenchmark {
    
    private BigDecimal a;
    private BigDecimal b;
    
    @Setup(Level.Trial)
    public void setup() {
        a = new BigDecimal("1234.5678");
        b = new BigDecimal("9876.5432");
    }
    
    @Benchmark
    public BigDecimal add() {
        return a.add(b);
    }
    
    @Benchmark
    public BigDecimal multiply() {
        return a.multiply(b);
    }
    
    @Benchmark
    public BigDecimal divide() {
        return a.divide(b, 4, RoundingMode.HALF_EVEN);
    }
}

public class BenchmarkRunner {
    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
            .include(BigDecimalBenchmark.class.getSimpleName())
            .forks(2)
            .result("results.json")
            .resultFormat(ResultFormatType.JSON)
            .build();
        new Runner(opt).run();
    }
}
```

Run:
```bash
mvn clean package
java -jar target/benchmarks.jar
```

Output:
```
Benchmark                          Mode  Cnt    Score    Error  Units
BigDecimalBenchmark.add            avgt   20   45.234 ±  1.234  ns/op
BigDecimalBenchmark.multiply       avgt   20  120.456 ±  5.234  ns/op
BigDecimalBenchmark.divide         avgt   20  450.789 ± 15.234  ns/op
```

### 4. Mode types

| Mode | Anlam | When |
|---|---|---|
| `Throughput` | Ops/sec | Throughput optimization |
| `AverageTime` | Time/op | Latency optimization |
| `SampleTime` | Distribution (p50, p99) | Latency distribution |
| `SingleShotTime` | One-shot (no warmup) | Cold start / setup overhead |
| `All` | All modes | Compare |

Banking:
- Transfer compute (CPU-bound): AverageTime
- Throughput stress: Throughput
- Latency SLO: SampleTime (p99)

### 5. State scope

| Scope | Anlam |
|---|---|
| `Scope.Benchmark` | Shared across threads |
| `Scope.Thread` | Per-thread instance |
| `Scope.Group` | Per @Group (split read/write threads) |

```java
@State(Scope.Thread)
public class ThreadState {
    BigDecimal x;
    @Setup public void s() { x = new BigDecimal("100"); }
}

@State(Scope.Benchmark)
public class SharedState {
    Map<String, Account> cache;   // Shared, but careful with mutation
}
```

### 6. Parameter sweep

```java
@State(Scope.Benchmark)
public class ParamBenchmark {
    
    @Param({"10", "100", "1000", "10000"})
    private int size;
    
    @Param({"HashMap", "ConcurrentHashMap", "Caffeine"})
    private String cacheType;
    
    private Map<Long, Account> cache;
    
    @Setup
    public void setup() {
        cache = switch (cacheType) {
            case "HashMap" -> new HashMap<>();
            case "ConcurrentHashMap" -> new ConcurrentHashMap<>();
            case "Caffeine" -> Caffeine.newBuilder().maximumSize(size).<Long, Account>build().asMap();
        };
        for (long i = 0; i < size; i++) {
            cache.put(i, new Account(i));
        }
    }
    
    @Benchmark
    public Account get() {
        return cache.get(ThreadLocalRandom.current().nextLong(size));
    }
}
```

Result matrix: 4 size × 3 cache type = 12 combinations.

### 7. Blackhole + escape

```java
@Benchmark
public void wrong() {
    BigDecimal r = a.add(b);
    // r unused → JIT may eliminate
}

@Benchmark
public BigDecimal correct() {
    return a.add(b);   // Returned → consumer prevents elimination
}

@Benchmark
public void blackhole(Blackhole bh) {
    BigDecimal r = a.add(b);
    bh.consume(r);
}
```

**Multiple results:**
```java
@Benchmark
public void multipleConsume(Blackhole bh) {
    bh.consume(a.add(b));
    bh.consume(a.multiply(b));
}
```

### 8. CompilerControl — JIT inlining

```java
@CompilerControl(CompilerControl.Mode.DONT_INLINE)
@Benchmark
public BigDecimal noInline() {
    return helper();
}

@CompilerControl(CompilerControl.Mode.DONT_INLINE)
private BigDecimal helper() { ... }
```

Method dispatch / inlining overhead ölçmek için.

### 9. Async-profiler integration

```bash
java -jar target/benchmarks.jar \
  -prof async:event=cpu \
  -prof async:event=alloc \
  -prof gc \
  BigDecimalBenchmark
```

Output: Flame graph + GC stats per benchmark.

Banking: Hot benchmark'a kim allocate ediyor → fix.

### 10. Banking benchmarks — examples

#### Benchmark 1: Money operations

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class MoneyBenchmark {
    
    private BigDecimal amount;
    private BigDecimal rate;
    private MathContext mc;
    
    @Setup
    public void setup() {
        amount = new BigDecimal("12345.6789");
        rate = new BigDecimal("1.0234");
        mc = new MathContext(20, RoundingMode.HALF_EVEN);
    }
    
    @Benchmark
    public BigDecimal addBigDecimal() {
        return amount.add(amount);
    }
    
    @Benchmark
    public BigDecimal multiplyBigDecimal() {
        return amount.multiply(rate);
    }
    
    @Benchmark
    public BigDecimal divideWithMathContext() {
        return amount.divide(rate, mc);
    }
    
    @Benchmark
    public BigDecimal newMathContextEveryCall() {
        return amount.divide(rate, new MathContext(20, RoundingMode.HALF_EVEN));
    }
    
    @Benchmark
    public long longArithmetic() {
        return 123456L + 789012L;   // BigDecimal alternative for fixed-precision
    }
}
```

Expected result: `longArithmetic` ~1 ns, `addBigDecimal` ~50 ns, `divide` ~500 ns, `newMathContext` overhead.

#### Benchmark 2: Concurrent map

```java
@State(Scope.Benchmark)
@Threads(4)
public class ConcurrentMapBenchmark {
    
    @Param({"HashMap", "ConcurrentHashMap", "Caffeine"})
    private String type;
    
    private Map<Long, Account> map;
    
    @Setup
    public void setup() {
        // ... init map
    }
    
    @Benchmark
    @Group("readWrite")
    @GroupThreads(3)
    public Account read() {
        return map.get(ThreadLocalRandom.current().nextLong(10_000));
    }
    
    @Benchmark
    @Group("readWrite")
    @GroupThreads(1)
    public Account write() {
        return map.put(ThreadLocalRandom.current().nextLong(10_000), new Account());
    }
}
```

3 reader thread + 1 writer thread concurrent.

#### Benchmark 3: Serialization

```java
@State(Scope.Benchmark)
public class SerializationBenchmark {
    
    private Transfer transfer;
    private ObjectMapper jackson;
    private byte[] avroSchema;
    private Serializer<Transfer> protobuf;
    
    @Setup
    public void setup() {
        transfer = createTransfer();
        jackson = new ObjectMapper();
        // ... protobuf, avro setup
    }
    
    @Benchmark
    public byte[] jacksonSerialize() throws Exception {
        return jackson.writeValueAsBytes(transfer);
    }
    
    @Benchmark
    public byte[] protobufSerialize() {
        return protobuf.serialize(transfer);
    }
    
    @Benchmark
    public byte[] avroSerialize() {
        // ... avro logic
    }
}
```

#### Benchmark 4: Lock comparison

```java
@State(Scope.Benchmark)
@Threads(8)
public class LockBenchmark {
    
    private final Object intrinsicLock = new Object();
    private final ReentrantLock reentrant = new ReentrantLock();
    private final StampedLock stamped = new StampedLock();
    private final AtomicReference<BigDecimal> atomic = new AtomicReference<>(BigDecimal.ZERO);
    
    private BigDecimal balance = BigDecimal.ZERO;
    
    @Benchmark
    public void synchronizedAdd() {
        synchronized (intrinsicLock) {
            balance = balance.add(BigDecimal.ONE);
        }
    }
    
    @Benchmark
    public void reentrantLockAdd() {
        reentrant.lock();
        try {
            balance = balance.add(BigDecimal.ONE);
        } finally {
            reentrant.unlock();
        }
    }
    
    @Benchmark
    public void atomicAdd() {
        atomic.updateAndGet(b -> b.add(BigDecimal.ONE));
    }
}
```

### 11. JMH CI integration — performance regression

```yaml
# .github/workflows/perf.yml
name: Performance Benchmark

on:
  pull_request:
    paths:
      - 'src/main/java/com/bank/**'
      - 'pom.xml'

jobs:
  benchmark:
    runs-on: ubuntu-latest-cpu-isolated   # Dedicated runner
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Run benchmarks (this branch)
        run: |
          mvn clean package -DskipTests
          java -jar target/benchmarks.jar \
            -rf json -rff branch-result.json \
            -wi 3 -i 5 -f 2
      
      - name: Get baseline (main)
        run: |
          git checkout main
          mvn clean package -DskipTests
          java -jar target/benchmarks.jar \
            -rf json -rff main-result.json \
            -wi 3 -i 5 -f 2
      
      - name: Compare
        run: |
          python compare.py main-result.json branch-result.json > diff.md
          cat diff.md
          # Fail if regression > 10%
          python compare.py --threshold 0.10 main-result.json branch-result.json
      
      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const diff = require('fs').readFileSync('diff.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '## Benchmark Diff\n\n' + diff
            });
```

**Critical: dedicated runner.** Shared CI noise → false regressions.

Banking için dedicated bare-metal CI runner.

### 12. Stable benchmark environment

```bash
# CPU affinity
taskset -c 0,1 java -jar benchmarks.jar

# Disable turbo boost
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# Disable HyperThreading sibling
echo 0 | sudo tee /sys/devices/system/cpu/cpu1/online

# Set CPU governor performance
sudo cpupower frequency-set -g performance

# Disable transparent huge pages
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

JMH options:
```java
.jvmArgsAppend("-XX:-RestrictContended")
.jvmArgsAppend("-XX:-TieredCompilation")   // Stable, no C1
.jvmArgsAppend("-XX:CICompilerCount=2")
.jvmArgsAppend("-Xms4g", "-Xmx4g")
.forks(3)
.warmupIterations(5)
.measurementIterations(10)
```

### 13. Statistical significance

JMH output:
```
add  avgt   20   45.234 ±  1.234  ns/op
```

Score 45.234, **error ±1.234** (3σ confidence interval).

Two benchmark compare:
- A: 45 ± 2 ns
- B: 50 ± 3 ns

Overlap: yes (47-48 range) → **not statistically significant**.

Banking için ≥ 5% difference + non-overlapping CI → meaningful.

### 14. JMH for assertions

Spring Boot test'te JMH assertion:

```java
@Test
void transferShouldBeFast() throws Exception {
    Options opt = new OptionsBuilder()
        .include("transferBenchmark")
        .warmupIterations(2)
        .measurementIterations(3)
        .forks(1)
        .build();
    
    Collection<RunResult> results = new Runner(opt).run();
    RunResult r = results.iterator().next();
    double avgTime = r.getPrimaryResult().getScore();   // ns/op
    
    assertThat(avgTime).isLessThan(1_000_000);   // < 1 ms
}
```

CI'da regression detect.

### 15. Banking benchmark anti-pattern'leri

**Anti-pattern 1: Naive `System.nanoTime()` loop**

Yukarıda. JMH kullan.

**Anti-pattern 2: Single run, single fork**

Statistical noise. `forks=2-3, iterations=5-10`.

**Anti-pattern 3: Warmup yetersiz**

JIT 1000-10000 iteration sonra optimize. Min 3-5 warmup iteration.

**Anti-pattern 4: State scope yanlış**

`@State(Scope.Thread)` shared object için → wrong measurement.

**Anti-pattern 5: Result discard (no Blackhole)**

DCE elimination → benchmark anlamsız.

**Anti-pattern 6: I/O in benchmark**

DB query, network in `@Benchmark` → noise dominates. Profile production traffic + measure isolated component.

**Anti-pattern 7: Comparison without baseline**

Banking için: "X faster than Y" iddiası baseline run gerekli. CI integration.

**Anti-pattern 8: CI runner shared**

Noise. Dedicated runner.

**Anti-pattern 9: Premature optimization**

JMH gösterse bile, code-level micro-difference production bottleneck olmayabilir. Önce metric + profile (Topic 9.2, 9.4).

**Anti-pattern 10: Benchmark code production'da çalıştırmak**

JMH benchmark = test artifact. Production binary'de bulunmamalı.

---

## Önemli olabilecek araştırma kaynakları

- JMH official docs
- JMH samples (github)
- "Java Microbenchmarks Done Right" — Aleksey Shipilëv
- Async-profiler + JMH integration
- "Optimizing Java" — Benjamin Evans
- Shipilëv blog (Disruptor benchmark style)

---

## Mini task'ler

### Task 9.6.1 — JMH project setup (30 dk)

Maven archetype veya manual dependency. Basit benchmark çalışsın.

### Task 9.6.2 — BigDecimal benchmark suite (60 dk)

add, multiply, divide (mathcontext static vs per-call), longArithmetic comparison. Result analyze.

### Task 9.6.3 — Parameter sweep cache benchmark (60 dk)

HashMap vs ConcurrentHashMap vs Caffeine, 4 size. Matrix result.

### Task 9.6.4 — Serialization comparison (60 dk)

Jackson JSON vs Protobuf vs Avro for `Transfer` object. Size + serialize/deserialize speed.

### Task 9.6.5 — Lock benchmark (60 dk)

synchronized vs ReentrantLock vs AtomicReference for balance update. 1, 4, 8 thread.

### Task 9.6.6 — async-profiler integration (45 dk)

`-prof async:event=cpu` ile en yavaş benchmark'ın flame graph'i.

### Task 9.6.7 — CI integration (60 dk)

GitHub Actions workflow. PR'da benchmark çalışsın. baseline (main) vs PR comparison. Comment PR.

### Task 9.6.8 — Statistical significance check (30 dk)

Aynı benchmark 3 kez. Error margin'lerin overlap durumu interpret.

### Task 9.6.9 — Spring test entegrasyonu (45 dk)

JUnit test'te JMH run. assertion ile threshold check.

### Task 9.6.10 — Banking real-world (60 dk)

Production'da yavaş method (gerçek veya simulated). Profile → suspect method JMH'la → fix candidate JMH ile compare.

---

## Test yazma rehberi

```java
@Test
@Tag("benchmark")
void bigDecimalAddShouldBeUnder100Ns() throws Exception {
    Options opt = new OptionsBuilder()
        .include(BigDecimalBenchmark.class.getSimpleName() + "\\.add")
        .warmupIterations(3)
        .measurementIterations(5)
        .forks(1)
        .build();
    
    Collection<RunResult> results = new Runner(opt).run();
    RunResult r = results.iterator().next();
    double score = r.getPrimaryResult().getScore();
    
    assertThat(score).isLessThan(100.0);   // ns
}
```

Maven:
```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludedGroups>benchmark</excludedGroups>   <!-- Regular test'te skip -->
    </configuration>
</plugin>
```

CI nightly: `mvn test -Dgroups=benchmark`.

---

## Claude-verify prompt

```
JMH setup'ımı banking-grade kriterlere göre değerlendir:

1. Setup:
   - JMH dependency + annotation processor?
   - Benchmark jar build?
   - JUnit entegrasyonu (assertion threshold)?

2. Benchmark design:
   - Mode (AverageTime / Throughput / SampleTime)?
   - State scope doğru (Thread vs Benchmark)?
   - Blackhole DCE prevention?
   - Forks ≥ 2?
   - Warmup ≥ 3 iteration?

3. Banking benchmarks:
   - BigDecimal operations?
   - Concurrent map (read/write groups)?
   - Serialization (Jackson/Protobuf/Avro)?
   - Lock comparison (synchronized/ReentrantLock/Atomic)?

4. Parameter sweep:
   - @Param matrix?
   - Cardinality reasonable (combinatorial explosion YOK)?

5. Profiler integration:
   - -prof async:event=cpu?
   - -prof async:event=alloc?
   - -prof gc?

6. Environment:
   - Dedicated runner CI?
   - CPU affinity (taskset)?
   - Turbo boost / HT disable considered?
   - Same JVM flags across runs?

7. Statistical:
   - Score ± Error reported?
   - Compare with non-overlapping CI?
   - 5%+ threshold for "meaningful"?

8. CI integration:
   - PR benchmark vs main baseline?
   - Regression > 10% fail CI?
   - PR comment with diff?

9. Banking real-world tie:
   - Production profile → JMH benchmark hot method?
   - Fix → JMH verify improvement?

10. Anti-pattern:
    - Naive System.nanoTime() YOK?
    - I/O in @Benchmark YOK?
    - Result discard (no Blackhole) YOK?
    - Single fork single run YOK?
    - Shared CI runner YOK?
    - Premature optimization YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] JMH setup + benchmark jar
- [ ] BigDecimal benchmark suite (5+ method)
- [ ] Parameter sweep cache benchmark
- [ ] Serialization comparison
- [ ] Lock comparison (multi-thread groups)
- [ ] async-profiler integration
- [ ] Statistical significance interpret
- [ ] CI integration (PR vs baseline + comment)
- [ ] JUnit assertion threshold test
- [ ] 1 production-tied real-world case study

---

## Defter notları (10 madde)

1. "Naive `System.nanoTime()` benchmark neden bozuk (DCE, JIT, GC, frequency): ____."
2. "JMH Mode selection (AverageTime, Throughput, SampleTime) banking: ____."
3. "State scope (Thread vs Benchmark vs Group) ne zaman hangisi: ____."
4. "Blackhole + return value DCE prevention: ____."
5. "Parameter sweep @Param matrix banking design (cache type x size): ____."
6. "async-profiler integration `-prof async:event=cpu` flame graph: ____."
7. "Dedicated CI runner + CPU affinity + turbo boost off (stable env): ____."
8. "Score ± Error statistical significance + non-overlapping CI: ____."
9. "PR vs baseline benchmark + regression % threshold + PR comment: ____."
10. "Production profile → JMH benchmark → fix → re-benchmark verify workflow: ____."
