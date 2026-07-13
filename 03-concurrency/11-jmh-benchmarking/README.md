# Topic 3.11 — JMH (Java Microbenchmark Harness)

## Hedef

JMH ile **doğru** Java microbenchmark yazmak. Constant folding, dead code elimination, JIT warmup gibi tuzaklara düşmemek. Banking domain'inde gerçek karar destekleyici benchmark'lar yazmak (BigDecimal vs long-based money, virtual vs platform thread, ConcurrentHashMap vs synchronized HashMap).

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 30 dk • Toplam: ~4 saat

## Önbilgi

- Topic 3.1-3.10 bitti
- Performans hakkında konuşmaya alışkınsın
- "Hızlı / yavaş" demek yerine "X mikrosaniye/op" demek senin için makul

---

## Kavramlar

### 1. Neden JMH — naive benchmark'lar yalan söyler

Junior'ın klasik benchmark:

```java
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    money.add(other);
}
long elapsed = System.nanoTime() - start;
System.out.println(elapsed + " ns total");
```

**Bu kod yalan söyler.** Çünkü:

1. **JIT warmup yok:** İlk 10000 invocation interpreter'de çalışır, yavaş. Daha sonra C2 compile eder.
2. **Dead code elimination (DCE):** Result kullanılmıyorsa JIT loop'u tamamen kaldırabilir.
3. **Constant folding:** Argümanlar sabitse JIT compile time'da hesaplar.
4. **Loop unrolling:** JIT loop'u optimize ederek benchmark scenario'sundan sapar.
5. **GC noise:** Test ortasında GC olunca outlier sonuç.
6. **CPU caches:** İlk iterasyon cache miss, sonrakiler hit — ölçüm farklı.
7. **OS scheduling:** Process'in CPU zamanı dağılımı belirsiz.

JMH bu tuzakların **hepsine** karşı önlem alır.

### 2. JMH setup

`pom.xml`:

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

JMH'in idiomatic yolu: **ayrı bir Maven module** (`banking-benchmarks`). Production kodunu kirletmez.

### 3. Temel JMH benchmark

```java
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)
@State(Scope.Benchmark)
public class MoneyAddBenchmark {
    
    Money tl100;
    Money tl50;
    
    @Setup
    public void setup() {
        tl100 = Money.of("100.00", "TRY");
        tl50 = Money.of("50.00", "TRY");
    }
    
    @Benchmark
    public Money baselineAdd() {
        return tl100.add(tl50);
    }
}
```

`@Benchmark` method'unun **dönen değeri** Blackhole tarafından tüketilir — DCE'den kaçınma yolu (return ile).

### 4. JMH annotation'ları

#### `@BenchmarkMode`

- `Mode.Throughput`: ops/saniye
- `Mode.AverageTime`: zaman/op
- `Mode.SampleTime`: zaman distribution (p50, p99)
- `Mode.SingleShotTime`: bir kez ölç (warmup-less senaryolar)
- `Mode.All`: hepsi

Banking için **AverageTime** ve **Throughput** en kullanışlı.

#### `@OutputTimeUnit`

Sonuçlar hangi unit'te raporlansın: `NANOSECONDS`, `MICROSECONDS`, `MILLISECONDS`.

Money operasyonu için NANOSECONDS, network çağrısı için MILLISECONDS.

#### `@Warmup` ve `@Measurement`

```java
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
```

- 5 warmup iteration (her biri 1 sn) — JIT ısınsın
- 10 measurement iteration (her biri 1 sn) — ortalama al

#### `@Fork`

```java
@Fork(2)
```

2 ayrı JVM instance'ında çalıştır → her birinde 10 iteration. Toplam 20 sample. Daha güvenilir (JIT compile'ı tek run'da deterministic değil).

`@Fork(value = 2, jvmArgs = {"-Xms2g", "-Xmx2g"})` — JVM flag'leri.

#### `@State(Scope)`

- `Scope.Benchmark`: tüm benchmark thread'leri aynı state
- `Scope.Thread`: her thread'in kendi state'i
- `Scope.Group`: thread group bazlı

Banking concurrent benchmark'larda `Scope.Thread` çok kullanılır.

#### `@Setup` ve `@TearDown`

```java
@Setup(Level.Trial)        // tüm benchmark öncesi bir kez
public void setupTrial() { ... }

@Setup(Level.Iteration)    // her iteration öncesi
public void setupIteration() { ... }

@Setup(Level.Invocation)   // her invocation öncesi (riskli — overhead ekler)
public void setupInvocation() { ... }
```

### 5. Blackhole — DCE'den kaçınma

```java
@Benchmark
public void noReturn(Blackhole bh) {
    Money result = tl100.add(tl50);
    bh.consume(result);   // JIT bu çağrıyı kaldıramaz
}
```

Veya **return**:

```java
@Benchmark
public Money withReturn() {
    return tl100.add(tl50);   // dönen değer benchmark framework tarafından tüketilir
}
```

İkisi de OK. Return formu daha okunaklı.

**Tehlike:**

```java
@Benchmark
public void wrong() {
    tl100.add(tl50);          // ❌ sonuç hiçbir yere gitmiyor → DCE
}
```

JIT bu kodu **tamamen silebilir**. Benchmark "0 ns" gibi anlamsız sonuç verir.

### 6. Constant folding tuzağı

```java
@Benchmark
public Money constantFolded() {
    return Money.of("100.00", "TRY").add(Money.of("50.00", "TRY"));   // ❌ JIT compile time'da hesap
}
```

Tüm değerler sabit → JIT compile zamanında sonuç sabitleyebilir.

**Çözüm:** `@State` field'lar olarak değerleri tut, `@Setup`'ta yarat.

### 7. JMH ile parametrik benchmark

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class MoneyOpsBenchmark {
    
    @Param({"100.00", "1000.00", "1000000.00"})
    String amountStr;
    
    Money m1, m2;
    
    @Setup
    public void setup() {
        m1 = Money.of(amountStr, "TRY");
        m2 = Money.of("0.01", "TRY");
    }
    
    @Benchmark
    public Money add() {
        return m1.add(m2);
    }
}
```

Sonuç: 3 ayrı benchmark, her amount'ta ayrı sonuç.

### 8. Concurrent JMH benchmark

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Threads(8)
@State(Scope.Benchmark)
public class ConcurrentMapBenchmark {
    
    @Param({"ConcurrentHashMap", "SynchronizedHashMap"})
    String mapType;
    
    Map<UUID, BigDecimal> map;
    UUID[] keys;
    int idx;
    
    @Setup
    public void setup() {
        map = switch (mapType) {
            case "ConcurrentHashMap" -> new ConcurrentHashMap<>();
            case "SynchronizedHashMap" -> Collections.synchronizedMap(new HashMap<>());
            default -> throw new IllegalArgumentException();
        };
        
        keys = new UUID[1000];
        for (int i = 0; i < 1000; i++) {
            UUID k = UUID.randomUUID();
            keys[i] = k;
            map.put(k, BigDecimal.valueOf(i));
        }
    }
    
    @Benchmark
    public BigDecimal get() {
        UUID k = keys[(idx++) % keys.length];
        return map.get(k);
    }
}
```

`@Threads(8)` — 8 thread eş zamanlı.

**Sonuç beklenisi:** `ConcurrentHashMap` 5-10x daha hızlı (read'ler lock-free).

### 9. Banking benchmark örnekleri

#### Benchmark 1: BigDecimal vs long-based money

Bazı banking projeleri "long ile cent saklayalım, BigDecimal yavaş" der. **Gerçek mi?**

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class MoneyImplBenchmark {
    
    BigDecimal bdA, bdB;
    long longA, longB;
    
    @Setup
    public void setup() {
        bdA = new BigDecimal("100.50");
        bdB = new BigDecimal("50.25");
        longA = 10050L;   // kuruş olarak
        longB = 5025L;
    }
    
    @Benchmark
    public BigDecimal bigDecimalAdd() {
        return bdA.add(bdB);
    }
    
    @Benchmark
    public long longAdd() {
        return longA + longB;
    }
}
```

**Tipik sonuç:** BigDecimal ~30ns, long ~1ns. **30x fark.**

Karar: Banking'de **doğruluk > performans**. BigDecimal kazanır. Ama hot inner loop'larda (örn. faiz hesabı milyonlarca kayıt için) düşünülebilir.

#### Benchmark 2: Virtual vs platform threads (I/O bound)

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
public class ThreadBenchmark {
    
    @Param({"100", "1000", "10000"})
    int concurrency;
    
    @Benchmark
    public void platformThreads() throws InterruptedException {
        ExecutorService exec = Executors.newFixedThreadPool(concurrency);
        runTasks(exec);
        exec.shutdown();
        exec.awaitTermination(1, TimeUnit.MINUTES);
    }
    
    @Benchmark
    public void virtualThreads() throws InterruptedException {
        try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
            runTasks(exec);
        }
    }
    
    private void runTasks(ExecutorService exec) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(concurrency);
        for (int i = 0; i < concurrency; i++) {
            exec.submit(() -> {
                try { Thread.sleep(10); } catch (InterruptedException e) {}
                latch.countDown();
            });
        }
        latch.await();
    }
}
```

Beklenti:
- 100 task: platform ≈ virtual
- 1000: virtual 2-3x daha hızlı
- 10000: platform OOM veya massive context switch overhead

#### Benchmark 3: ConcurrentHashMap vs synchronized

Yukarıda anlatıldı. Throughput karşılaştırması.

#### Benchmark 4: HashMap vs ConcurrentHashMap (single thread)

```java
@Benchmark
public BigDecimal hashMapGet() {
    return hashMap.get(key);
}

@Benchmark
public BigDecimal concurrentHashMapGet() {
    return concurrentMap.get(key);
}
```

Sonuç: Tek thread'de **fark çok az** (~%5). `ConcurrentHashMap` single-thread'de bile rakip.

### 10. JMH profiler integration

```bash
mvn clean install
java -jar target/benchmarks.jar -prof gc          # GC profile
java -jar target/benchmarks.jar -prof stack       # stack profiler
java -jar target/benchmarks.jar -prof jfr         # JFR çıkarır
java -jar target/benchmarks.jar -prof async       # async-profiler
```

`-prof gc` çıktısı:

```
MoneyOpsBenchmark.add:gc.alloc.rate     8123 MB/sec
MoneyOpsBenchmark.add:gc.alloc.rate.norm  168 B/op
```

`168 B/op` → her operation 168 byte allocate ediyor. BigDecimal immutable, her add yeni instance yaratır.

### 11. JMH sonuçlarını yorumlama

JMH çıktı örneği:

```
Benchmark                          Mode  Cnt   Score   Error  Units
MoneyOpsBenchmark.add              avgt   20  28.123 ± 1.245  ns/op
MoneyOpsBenchmark.add:gc.alloc.rate avgt 20  8123.45 ± 145.78  MB/sec
```

- `Cnt`: total iterations (fork × measurement = 2×10 = 20)
- `Score`: mean
- `Error`: 99.9% confidence interval (mean ± 1.245 ns)
- `Score - Error` ile `Score + Error` aralığı **güvenilir aralık**

**Iki benchmark karşılaştırması:**

```
BigDecimal.add:  28.123 ± 1.245 ns/op  → aralık [26.878, 29.368]
Long.add:        1.025 ± 0.123 ns/op  → aralık [0.902, 1.148]
```

Aralıklar overlap etmiyor → fark istatistiksel anlamlı. Long ~27x daha hızlı.

### 12. JMH anti-pattern'leri

**Anti-pattern 1: Dönen değeri kullanmamak**

Yukarıda anlatıldı — DCE riski.

**Anti-pattern 2: Setup'ta constant değerler**

```java
@Benchmark
public BigDecimal benchmarkAdd() {
    BigDecimal a = new BigDecimal("100");   // ❌ her invocation'da yarat
    return a.add(BigDecimal.ONE);
}
```

`new` overhead'i ölçüme dahil oluyor. `@Setup` ile field'a koy.

**Anti-pattern 3: Fork'suz**

```java
@Fork(0)   // ❌ tek JVM
```

Tek JVM'in JIT decision'ı deterministic değil. `@Fork(2)` minimum.

**Anti-pattern 4: Çok kısa benchmark süresi**

```java
@Warmup(iterations = 1, time = 1, timeUnit = TimeUnit.NANOSECONDS)   // ❌ 1 ns warmup
```

Realistic warmup süresi 1-5 saniye.

**Anti-pattern 5: Concurrent benchmark'ta Scope.Benchmark + mutable state**

```java
@State(Scope.Benchmark)
public class Bench {
    private int counter;   // ❌ multiple thread access — race
    
    @Benchmark
    public void inc() { counter++; }
}
```

`Scope.Thread` yap veya atomic kullan.

**Anti-pattern 6: Production code'a JMH dependency sızdırmak**

JMH benchmark'lar **ayrı module**. Production JAR'ında JMH olmasın.

---

## Önemli olabilecek araştırma kaynakları

- JMH official documentation (openjdk.org/projects/code-tools/jmh/)
- JMH Samples (GitHub: openjdk/jmh) — 38 örnek
- "Java Performance: The Definitive Guide" — Microbenchmarking chapter
- Aleksey Shipilev (JMH yazarı) blog posts
- JMH presentations at JVM Language Summit
- "The art of benchmarking" Brian Goetz

---

## Mini task'ler

### Task 3.11.1 — Setup ve ilk benchmark (30 dk)

`core-banking/banking-benchmarks` adlı ayrı Maven module yarat. JMH dependency'sini ekle. `MoneyAddBenchmark` yaz (yukarıdaki örnek). Çalıştır:

```bash
mvn clean package
java -jar target/benchmarks.jar MoneyAddBenchmark
```

Sonuç **defterine** yazıştır.

### Task 3.11.2 — BigDecimal vs long benchmark (45 dk)

`MoneyImplBenchmark` yaz (yukarıda örnek). Çalıştır, sonucu defter'e yaz. **Defterine kararını yaz**: banking'de hangisini neden tercih ettin?

### Task 3.11.3 — Virtual vs platform thread (45 dk)

`ThreadBenchmark` yaz (yukarıda örnek). `@Param` ile 100/1000/10000 concurrency level dene.

Çalıştır:

```bash
java -jar target/benchmarks.jar ThreadBenchmark
```

Sonucu tabloya yaz, **defterine** yorum.

### Task 3.11.4 — ConcurrentHashMap throughput (45 dk)

`ConcurrentMapBenchmark` yaz (yukarıda örnek). `@Threads(8)` ile concurrent get/put.

Üç implementation: ConcurrentHashMap, synchronizedMap, HashMap (ile single thread).

### Task 3.11.5 — GC profile (15 dk)

```bash
java -jar target/benchmarks.jar MoneyOpsBenchmark -prof gc
```

Output'ta `gc.alloc.rate.norm` (byte/op) görsen. **Defterine** banking objesinin allocation rate'ini yaz.

### Task 3.11.6 — Confidence interval ile karar (30 dk)

İki Money implementation arasında JMH benchmark yazarken iki sonucun 99.9% confidence interval'ları overlap etmemeli. Birinin **gerçekten** daha hızlı olduğunu doğrula. Eğer overlap varsa, fark istatistiksel anlamlı değil.

---

## Test yazma rehberi

JMH benchmark'ları **kendi test'i**. Bunun üstüne unit test yazmaya gerek yok. Ama:

### Test 3.11.1 — Benchmark logic sanity test

```java
@Test
void benchmarkSetupShouldCreateExpectedState() {
    MoneyAddBenchmark bench = new MoneyAddBenchmark();
    bench.setup();
    
    assertThat(bench.tl100).isEqualTo(Money.of("100.00", "TRY"));
    assertThat(bench.tl50).isEqualTo(Money.of("50.00", "TRY"));
    
    Money result = bench.baselineAdd();
    assertThat(result).isEqualTo(Money.of("150.00", "TRY"));
}
```

Benchmark'ın **doğru iş yaptığını** doğrula. Hızı boş ver burada.

### Test 3.11.2 — CI'da benchmark threshold

```yaml
# .github/workflows/perf.yml
- name: Run JMH benchmarks
  run: |
    cd banking-benchmarks
    mvn clean package
    java -jar target/benchmarks.jar -rf json -rff results.json
    
- name: Check performance regression
  run: |
    SCORE=$(jq '.[] | select(.benchmark | endswith(".add")) | .primaryMetric.score' results.json)
    if (( $(echo "$SCORE > 50" | bc -l) )); then
      echo "Regression detected: Money.add > 50ns"
      exit 1
    fi
```

Banking'de performance regression CI'da yakalanmalı (Phase 9 detay).

---

## Claude-verify prompt

```
Aşağıdaki JMH benchmark kodumu banking-grade kriterlere göre değerlendir. Sadece 
eksikleri ve yanlışları işaretle:

1. Setup:
   - JMH ayrı Maven module'da mı (production code'una bulaşmıyor mu)?
   - `jmh-generator-annprocess` provided scope mu?

2. Benchmark structure:
   - `@BenchmarkMode` doğru (AverageTime, Throughput, vs)?
   - `@OutputTimeUnit` mantıklı (operation latency'sine uygun)?
   - `@Warmup` ve `@Measurement` realistic süreli mi (1-5 sn)?
   - `@Fork(2)` veya daha fazla mı (deterministic değil tek fork)?
   - `@State(Scope)` benchmark için doğru mu?

3. DCE'den kaçınma:
   - Return value var mı veya Blackhole.consume kullanılmış mı?
   - Sonuç hiçbir yere gitmeyen benchmark var mı? (Olmamalı)

4. Constant folding'den kaçınma:
   - Sabit literal'lar benchmark method'unun içinde mi? (Olmamalı — @Setup'a koy)
   - Test data @State field'da mı, dynamic mı?

5. Banking benchmark'ları:
   - BigDecimal vs long karşılaştırması var mı?
   - Virtual vs platform thread benchmark mevcut mu?
   - ConcurrentHashMap vs synchronized karşılaştırması yapılmış mı?
   - Allocation rate (-prof gc) ölçülmüş mü?

6. Concurrent benchmark'lar:
   - `@Threads(N)` belirli mi?
   - Mutable state Thread scope'ta mı (Benchmark DEĞİL)?

7. Sonuç yorumlama:
   - Confidence interval (± Error) okunup overlap kontrol edilmiş mi?
   - Sonuç sadece "score" değil, "score ± error" tartışılmış mı?

8. Anti-pattern:
   - `@Fork(0)` veya `@Fork(1)` mi? (deterministic değil)
   - Çok kısa warmup mı?
   - Mutable shared state race condition'a açık mı?

9. CI integration:
   - Benchmark sonuçları JSON olarak çıkarılıyor mu?
   - Performance regression threshold check var mı?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `banking-benchmarks` Maven module yarattım
- [ ] JMH dependency'leri pom.xml'de
- [ ] `MoneyAddBenchmark` çalışıyor, sonuç deftere yazıldı
- [ ] BigDecimal vs long benchmark sonucunu **rakamla** biliyorum
- [ ] Virtual vs platform thread @Param ile 100/1000/10000 dene
- [ ] ConcurrentHashMap vs synchronized throughput karşılaştırması yaptım
- [ ] `-prof gc` ile allocation rate ölçtüm
- [ ] Confidence interval okuyabiliyorum, anlamlı fark vs noise ayırabiliyorum
- [ ] DCE, constant folding, fork tuzaklarını biliyorum
- [ ] CI'da benchmark threshold check için yöntem hazırladım

---

## Defter notları

1. "Naive `System.nanoTime()` benchmark'ın 5 sorunu: ____."
2. "JIT warmup neden gerekli (interpreter → C1 → C2): ____."
3. "Dead code elimination'dan kaçınma yolu (return / Blackhole): ____."
4. "Constant folding nasıl olur, nasıl önlerim: ____."
5. "`@Fork(2)` neden minimum (single fork güvensizliği): ____."
6. "`@State(Scope.Thread)` ne zaman, `Scope.Benchmark` ne zaman: ____."
7. "Confidence interval (Error) ile mean'in birlikte yorumlanması: ____."
8. "Banking: BigDecimal vs long benchmark sonucum (rakam): ____."
9. "Banking: virtual vs platform thread fark hangi concurrency'den itibaren: ____."
10. "`-prof gc` allocation rate'in banking için anlamı: ____."
