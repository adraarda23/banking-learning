# Topic 3.9 — JVM Internals: Memory, GC, JIT

## Hedef

JVM'in **iç mekanizmasını** banking-grade derinlikte anlamak. Heap layout, garbage collector seçimi (G1, ZGC, Shenandoah), JIT compilation (tiered), escape analysis ve memory area'larını öğrenmek. Production'da bir banking servisinin GC log'unu okuyup tune edebilmek.

## Süre

Okuma: 2.5 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 3.1-3.8 bitti
- "Heap, stack" temel ayrımı tanıdık
- `java -Xms... -Xmx...` flag'lerini gördün

---

## Kavramlar

### 1. JVM memory areas — büyük resim

```
JVM Process
├── Heap                       ← objelerin yaşadığı yer
│   ├── Young Generation
│   │   ├── Eden Space
│   │   └── Survivor Space (S0, S1)
│   └── Old Generation (Tenured)
├── Metaspace                  ← class metadata (Java 8+, eski PermGen)
├── Code Cache                 ← JIT'in derlediği native kod
├── Thread Stack (per-thread)  ← method call stack, primitive değerler
├── Native Memory              ← direct buffer, JNI
└── PC Register (per-thread)
```

**Banking servisi tipik bellek dağılımı:**

```
8GB JVM:
- Heap: 6GB (Xmx = 6g)
  - Young: 2GB
  - Old: 4GB
- Metaspace: 256MB
- Code Cache: 240MB
- Threads: ~500 thread × 1MB = 500MB
- Native (DirectByteBuffer): 500MB
```

### 2. Heap — generational hypothesis

**Generational hypothesis:** "Objelerin çoğu kısa ömürlüdür."

Banking örneği: bir HTTP request boyunca yaratılan DTO'lar, mapper sonuçları, intermediate hesaplama nesneleri — request bitince **çöp**. Sadece az sayıda obje "uzun yaşar" (cache entry'leri, connection pool, vb.).

GC bu varsayımdan beslenir:

1. **Young Generation:** Yeni objeler buraya yaratılır. Sık ve hızlı GC.
2. **Old Generation:** Young'dan **promote** olan objeler. Nadir ve maliyetli GC.

**Young layout:**

```
┌──────────┬────────┬────────┐
│   Eden   │   S0   │   S1   │
└──────────┴────────┴────────┘
```

- **Eden:** Yeni allocation buraya.
- **S0/S1:** Survivor space'ler. Minor GC'de yaşayan objeler S0/S1 arasında kopyalanır.

**Minor GC akışı:**

1. Eden + S0 → S1'e canlı objeleri kopyala
2. Eden + S0 boşalt
3. Bir sonraki minor GC'de Eden + S1 → S0'a (swap)
4. Bir obje N kez survive ederse (default ~15) → **Old** generation'a promote

**Major GC (Old GC):** Old generation'da yer tükenince. Çok daha yavaş.

### 3. Garbage collectors — banking için seçim

#### Serial GC (`-XX:+UseSerialGC`)

- Single-thread GC
- Stop-the-world (STW)
- Banking için **uygun değil** (latency yüksek)
- Sadece embedded, küçük heap senaryoları

#### Parallel GC (`-XX:+UseParallelGC`)

- Çok thread'li, throughput odaklı
- STW pause var ama daha kısa
- Banking için **batch job'larda** OK (EOD reconciliation, raporlama)
- Online işlemler için yetersiz (pause >1s görülür)

#### G1 GC (Garbage First) — Java 9+ default

- **Banking standardı.**
- Heap'i **region**'lara böler (1-32MB her biri).
- Young + Old aynı heap'te dağıtılır.
- Concurrent marking + STW pause kısa (genelde <200ms)
- **Tuning:**

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200    # hedef max pause
-Xms4g -Xmx4g                # heap sabit (banking pratiği)
```

`MaxGCPauseMillis` **target** — guarantee değil. Çok düşük (50ms) verirsen GC daha sık çalışır, throughput düşer.

#### ZGC (Z Garbage Collector) — Java 11+ stable, Java 21 generational

- **Pause-less.** Pauseları **<10ms** garantili (concurrent marking + relocation).
- **TB-ölçek** heap support (16TB'ye kadar).
- **CPU overhead:** ~15% throughput maliyeti.
- Banking için **düşük-latency** servisler (örn. fraud detection real-time) için ideal.

```bash
-XX:+UseZGC
-Xmx16g
```

Java 21'den itibaren **Generational ZGC** (`-XX:+ZGenerational`) — Young/Old ayrımı, throughput daha iyi.

#### Shenandoah — Red Hat

- ZGC benzeri, pause-less
- Daha düşük CPU overhead
- Java 12+ official (Red Hat builds'te)
- Banking'de OpenJDK + Red Hat ekosisteminde tercih

#### Banking için karar matrisi

| Profil | GC |
|---|---|
| API gateway, online transfer, p99 < 50ms | ZGC |
| Standart banking API (p99 < 200ms OK) | G1 (default) |
| EOD batch, raporlama (throughput) | Parallel |
| Düşük heap (<2GB) servisleri | G1 yine OK |
| Mevcut prod stable, dokunma | G1 |

### 4. GC tuning flag'leri

```bash
# Heap boyutu
-Xms6g -Xmx6g                          # initial = max (production)
-XX:NewRatio=2                         # Old:Young = 2:1

# G1
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:G1NewSizePercent=20                # Young min %
-XX:G1MaxNewSizePercent=40             # Young max %
-XX:G1ReservePercent=10                # promotion için yedek

# ZGC
-XX:+UseZGC
-XX:+ZGenerational                     # Java 21+

# GC log (production critical!)
-Xlog:gc*:file=/var/log/banking/gc.log:time,uptime:filecount=10,filesize=100M

# Heap dump on OOM (production critical!)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/banking/heapdump.hprof
```

**Banking kural:** Production'da **`Xms = Xmx`**. Heap dinamik büyümesin — GC pause'a sebep olur.

### 5. GC log analizi

G1 GC log örneği:

```
[2.143s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 
  100M->20M(2048M) 5.231ms
```

Açıklama:
- `Pause Young (Normal)`: Minor GC
- `100M->20M`: Heap kullanımı 100MB'tan 20MB'ye düştü
- `(2048M)`: Toplam heap kapasitesi
- `5.231ms`: Pause süresi

**Sağlıklı pattern:**
- Minor GC pause < 100ms
- Major GC pause < 1s (G1 mixed GC)
- Frequency: minor 10-30 sn, major 5-30 dk

**Sorunlu pattern'ler:**

```
GC(42) Pause Full (Allocation Failure) 1800M->1750M(2048M) 8500ms
```

Full GC + uzun pause → heap çok dolu, fragmentation, memory leak şüphesi.

```
GC(43) Pause Young (Normal) 90M->85M(2048M) 250ms
```

Young space çoğu obje promote olmaya gidiyor → tenuring threshold çok düşük veya Young çok küçük.

**Araçlar:**

- **GCViewer** — open source, gc.log parse edip görselleştirir
- **GCEasy.io** — web tabanlı, ücretsiz, AI öneri verir
- **JClarity Censum** — ticari, derin analiz
- **gcplot.com** — alternative

### 6. JIT compilation — interpreter → C1 → C2

Java kod execution'ın yaşam döngüsü:

```
1. javac → .class (bytecode)
2. JVM bytecode'u INTERPRET eder (yavaş)
3. "Hot" method (>10000 invocation) → C1 (client compiler) — basit, hızlı compile
4. Çok hot → C2 (server compiler) — agresif optimization, yavaş compile
5. Code Cache'e yerleşir, native execution
```

**Tiered compilation** (default Java 8+): Hem C1 hem C2 birlikte.

```bash
-XX:+TieredCompilation       # default
-XX:CompileThreshold=10000   # interpret → compiled (default)
```

#### Optimization'lar

**Inlining:** Küçük method'ları çağrı yerine kopyala.

```java
public BigDecimal getAmount() { return amount; }   // 5 satır method
// JIT bunu çağıran kod'a inline eder, method call yok
```

**Escape analysis:** Bir nesne sadece bir method içinde kullanılıyorsa, **heap yerine stack'te** yarat.

```java
public BigDecimal compute() {
    Money temp = Money.of("100", "TRY");   // sadece burada kullanılıyor
    return temp.amount();
}
// JIT: temp object'i stack'te yarat, GC yok, allocation cost yok
```

**Scalar replacement:** Bir objeyi field'larına böl.

```java
Money m = Money.of("100", "TRY");
// JIT: m.amount → local variable, m.currency → local variable, m heap'te değil
```

**Banking impact:** Banking app'inde inner loop'lardaki `BigDecimal` operasyonları — eğer ara değişkenler **leak etmiyor**sa JIT bunları stack'e koyar, allocation yükü kaybolur.

#### JIT log'lama

```bash
-XX:+PrintCompilation         # her compile event'i
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintInlining            # inline kararları
```

```
123  45    3   com.mavibank.banking.Money::add (12 bytes)
124  46    4   com.mavibank.banking.Account::deposit (45 bytes)
```

`3` = C1 tier, `4` = C2 tier. Method compile edildi.

### 7. Class loading & Metaspace

Java 7 ve öncesi: **PermGen** (sabit boyut, OOM kaynağı).
Java 8+: **Metaspace** (native memory, dynamic grow).

Metaspace içerir:
- Class metadata
- Method bytecode
- Reflection objeleri
- Class loader'lar

```bash
-XX:MaxMetaspaceSize=512m    # üst limit
-XX:MetaspaceSize=128m        # initial
```

**Banking tuzağı:** Çok class yüklenen app'ler (Spring + JPA + ek lib'ler) Metaspace shrink **OOM** olabilir. Default unbounded ama **MaxMetaspaceSize set et**.

**Class loader leak:** Spring DevTools restart ile classloader leak — Metaspace büyür, OOM. Production'da DevTools yok.

### 8. Code Cache

JIT'in compile ettiği native kod buraya gider.

```bash
-XX:ReservedCodeCacheSize=240m   # default
-XX:InitialCodeCacheSize=64m
```

**Banking tuzağı:** Code cache full → JIT compilation durur, performance düşer (sessiz).

```
CodeCache: size=245760Kb used=243512Kb max_used=245456Kb free=2248Kb
```

`used` ≈ `size` → genişlet:

```bash
-XX:ReservedCodeCacheSize=512m
```

### 9. Thread stack

Her thread için ayrı stack. Default size **1MB**.

```bash
-Xss512k   # her thread 512KB stack
```

**Banking impact:** 1000 platform thread × 1MB = 1GB native memory.

Virtual thread'lerde stack çok küçük (~kbyte) — Loom'un en büyük kazancı.

### 10. Native memory tracking

JVM'in heap dışı memory kullanımını görmek:

```bash
java -XX:NativeMemoryTracking=detail ...

jcmd <pid> VM.native_memory
```

Output:

```
Total: reserved=8GB, committed=6GB
- Java Heap (reserved=6GB, committed=6GB)
- Class (reserved=1GB, committed=200MB)
- Thread (reserved=500MB, committed=500MB)
- Code (reserved=240MB, committed=120MB)
- GC (reserved=100MB, committed=80MB)
- Direct (reserved=512MB, committed=300MB)  ← DirectByteBuffer
```

**Banking tuzağı:** Off-heap memory (DirectByteBuffer, Netty) heap'te görünmez ama OS belleği yer. **Track et.**

### 11. JDK Flight Recorder (JFR) — production-safe profiling

JFR JVM'in **dahili** profiling sistemi. Production'da %1-2 overhead ile çalışır.

```bash
# Continuous recording
-XX:StartFlightRecording=duration=60s,filename=banking.jfr

# Dump anlık
jcmd <pid> JFR.start name=banking
jcmd <pid> JFR.dump name=banking filename=banking.jfr
```

JFR events:
- Allocation (TLAB, outside TLAB)
- GC events
- Lock contention
- Thread states
- File I/O
- Network I/O
- JIT compilation
- Class loading

**Banking pratiği:** Production'da JFR sürekli açık olabilir, sorun yaşanınca son N dakikalık recording'i indirip JMC'de incele.

### 12. JIT decompile & deoptimization

JIT compile ettiği kodu bazen **deoptimize** eder (interpreter'e geri döner):

- Type assumption fail (örn. virtual method monomorphic compile, sonra polymorphic geldi)
- Uncommon trap (sıra dışı code path)

```bash
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintCompilation
-XX:+LogCompilation
```

Banking impact: nadiren önemli ama "uygulamamız hızlandı sonra yavaşladı" gizemli senaryolarda bakman gerekebilir.

### 13. Banking örnekleri — gerçek tuning

#### Senaryo 1: API service, p99 hedef 100ms

```bash
java -server \
     -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=50 \
     -XX:+ParallelRefProcEnabled \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/banking/heap.hprof \
     -Xlog:gc*:file=/var/log/banking/gc.log:time,uptime:filecount=10,filesize=100M \
     -jar core-banking.jar
```

#### Senaryo 2: Düşük-latency fraud detection (p99 hedef 10ms)

```bash
java -server \
     -Xms8g -Xmx8g \
     -XX:+UseZGC \
     -XX:+ZGenerational \
     -XX:-OmitStackTraceInFastThrow \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/fraud/heap.hprof \
     -jar fraud-service.jar
```

#### Senaryo 3: EOD batch (throughput maksimum)

```bash
java -server \
     -Xms16g -Xmx16g \
     -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -jar eod-job.jar
```

### 14. Anti-pattern'ler

**Anti-pattern 1: `-Xmx` çok büyük, `-Xms` küçük**

```bash
-Xms512m -Xmx16g
```

Sorun: Heap büyürken **GC pause** olur. Production'da `Xms = Xmx`.

**Anti-pattern 2: GC log kapalı production'da**

OOM olduğunda neden olduğunu bilemezsin. Her zaman GC log aktif.

**Anti-pattern 3: HeapDumpOnOutOfMemoryError yok**

OOM oldu → process restart → kanıt yok. Heap dump otomatik kaydet.

**Anti-pattern 4: Yanlış GC seçimi**

Düşük-latency servise Parallel GC takmak (uzun STW pause), batch job'a ZGC takmak (CPU overhead).

**Anti-pattern 5: `MaxGCPauseMillis` çok düşük**

```bash
-XX:MaxGCPauseMillis=10
```

G1 buna ulaşmak için Young'ı küçültür, sık minor GC yapar, **throughput düşer**. Realistic hedef (50-200ms).

---

## Önemli olabilecek araştırma kaynakları

- "Java Performance: The Definitive Guide" (Scott Oaks) — bütün JVM internals
- "Garbage Collection Handbook" — derin GC teori
- Oracle G1 GC tuning guide
- "JVM Anatomy" by Aleksey Shipilev (blog series)
- ZGC documentation (OpenJDK)
- Shenandoah GC paper (Red Hat)
- GCEasy.io blog post arşivi
- JEP 439: Generational ZGC

---

## Mini task'ler

### Task 3.9.1 — GC log aktif et ve oku (30 dk)

`core-banking` app'inde:

```bash
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -Xlog:gc*:file=gc.log:time,uptime \
     -jar core-banking.jar
```

100 transfer request gönder. `gc.log`'u aç, manuel oku.

Sonra GCEasy.io'ya yükle, raporu **defterine kopyala**.

### Task 3.9.2 — Heap dump on OOM (30 dk)

`-Xmx128m` ile çalıştır (kasten küçük). Bir endpoint ekle, OOM tetikle:

```java
@GetMapping("/_oom")
String oom() {
    List<byte[]> hold = new ArrayList<>();
    while (true) hold.add(new byte[10_000_000]);   // 10MB her seferde
}
```

```bash
java -Xmx128m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./oom.hprof \
     -jar core-banking.jar
```

OOM tetikle. `oom.hprof` dosyası oluştu mu? MAT (Eclipse Memory Analyzer) ile aç. **Defterine** dominator tree'yi yaz.

### Task 3.9.3 — G1 vs ZGC karşılaştırma (45 dk)

`core-banking`'i iki konfigürasyonla çalıştır:

```bash
# G1
java -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=100 ...

# ZGC
java -Xmx4g -XX:+UseZGC -XX:+ZGenerational ...
```

Aynı Gatling load test'i (200 req/sec, 5 dk) — her ikisinde p50, p95, p99 latency ölç. **Defterine** karşılaştırma yaz.

### Task 3.9.4 — JFR continuous recording (30 dk)

```bash
java -XX:StartFlightRecording=duration=2m,filename=banking.jfr \
     -jar core-banking.jar
```

200 transfer request gönder. JMC ile `banking.jfr` aç. İncele:
- Allocation hot paths
- Lock contention
- GC events

### Task 3.9.5 — Native memory tracking (15 dk)

```bash
java -XX:NativeMemoryTracking=detail ...
jcmd <pid> VM.native_memory
```

Output'u defterine yapıştır. Heap dışında ne kadar memory? (Class, Thread, Code, Direct).

### Task 3.9.6 — JIT compile log (30 dk)

```bash
java -XX:+PrintCompilation ...
```

Output'un ilk 50 satırını incele. Hangi method'lar C1, hangileri C2 oldu? Banking domain method'ları var mı (örn. `Money.add`, `Account.deposit`)?

---

## Test yazma rehberi

### Test 3.9.1 — Heap allocation tracking

```java
@Test
void shouldNotAllocateExcessivelyInHotPath() {
    // Allocation profile için JMH + -prof gc
    // Manuel test zor, JMH ile yap (Topic 3.11)
}
```

### Test 3.9.2 — Memory leak detection

```java
@Test
void cacheShouldNotGrowUnboundedly() throws Exception {
    AccountBalanceCache cache = new AccountBalanceCache();
    
    long before = getUsedHeap();
    
    for (int i = 0; i < 100_000; i++) {
        cache.put(UUID.randomUUID(), BigDecimal.valueOf(i));
    }
    
    System.gc();
    Thread.sleep(100);
    System.gc();
    
    long after = getUsedHeap();
    long growth = after - before;
    
    // Cache bounded olmalı (örn. 10000 entry limit)
    // 100k put sonrası heap büyümesi makul olmalı
    assertThat(growth).isLessThan(50_000_000);   // 50MB threshold
}

private long getUsedHeap() {
    Runtime r = Runtime.getRuntime();
    return r.totalMemory() - r.freeMemory();
}
```

Bu test fragile — System.gc() guarantee değil. Production'da MAT analizi tercih.

### Test 3.9.3 — TestContainers ile JVM flag karşılaştırma

Belirli bir endpoint için iki farklı JVM config ile timing karşılaştırma. CI/CD pipeline'da:

```yaml
# .github/workflows/perf.yml
- name: Run with G1
  run: java -Xmx2g -XX:+UseG1GC -jar app.jar &
  
- name: Run with ZGC
  run: java -Xmx2g -XX:+UseZGC -jar app.jar &
```

Daha çok integration test. JMH (Topic 3.11) daha kontrollü.

---

## Claude-verify prompt

```
Aşağıdaki JVM tuning ve memory analizi çalışmamı banking-grade kriterlere göre 
değerlendir. Sadece eksikleri ve yanlışları işaretle:

1. Heap konfigürasyonu:
   - `Xms = Xmx` mi (production)?
   - Heap size servisin profiline uygun mu (API vs batch)?
   - Heap dump on OOM aktif mi?

2. GC seçimi:
   - Online API için G1 veya ZGC kullanılmış mı (Parallel/Serial DEĞİL)?
   - Düşük-latency için ZGC tercih edilmiş mi?
   - Batch için Parallel tercih edilmiş mi?
   - MaxGCPauseMillis realistic mi (10ms çok düşük)?

3. GC log:
   - Production'da GC log aktif mi?
   - Log rotation (filecount, filesize) yapılandırılmış mı?
   - Log analiz edilmiş mi (GCEasy/GCViewer)?

4. Metaspace ve Code Cache:
   - MaxMetaspaceSize set mi (unbounded değil)?
   - ReservedCodeCacheSize yeterli mi?

5. Profiling:
   - JFR continuous recording production'da aktif mi?
   - Native memory tracking ile heap dışı memory görünüyor mu?
   - Allocation hot path'lerin profili çıkarılmış mı?

6. Banking-specific:
   - API gateway için ZGC + düşük pause hedef mi?
   - EOD batch için Parallel GC + büyük heap mi?
   - Fraud detection real-time için ZGC mi?

7. Anti-pattern:
   - Xms küçük + Xmx büyük (heap dynamic grow) var mı?
   - GC log production'da kapalı mı?
   - HeapDumpOnOutOfMemoryError eklenmiş mi?
   - MaxGCPauseMillis çok düşük mü (throughput katlıyor)?
   - DevTools production'da var mı? (Olmamalı — class loader leak)

8. Memory leak handling:
   - Memory leak şüphesinde MAT ile heap dump analiz akışı net mi?
   - Cache'ler bounded mi?
   - ThreadLocal'lar cleanup edilmiş mi?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] GC log aktif `core-banking`'te
- [ ] GCEasy.io ile GC raporu okudum
- [ ] HeapDumpOnOutOfMemoryError eklendi
- [ ] MAT ile heap dump analiz akışını **bir kez** uyguladım
- [ ] G1 vs ZGC karşılaştırma yaptım (latency)
- [ ] JFR continuous recording çalıştırdım, JMC ile inceledim
- [ ] Native Memory Tracking ile heap dışı belleği gördüm
- [ ] JIT compile log'unda C1 vs C2 method'ları tanıyabilirim
- [ ] `Xms = Xmx` pattern'ini benimsedim
- [ ] Banking için GC seçim matrisini biliyorum (API/batch/real-time)

---

## Defter notları

1. "Generational hypothesis ve banking için neden geçerli: ____."
2. "Young → Old promote akışı (minor GC + tenuring): ____."
3. "G1 vs ZGC vs Parallel — banking için karar kriterleri: ____."
4. "MaxGCPauseMillis target vs guarantee farkı: ____."
5. "Metaspace vs eski PermGen farkı: ____."
6. "Code cache full olunca ne olur: ____."
7. "Native memory tracking ile heap dışı bellek (Direct buffer, Thread stack): ____."
8. "Escape analysis + scalar replacement banking kodunda nereden tasarruf yapar: ____."
9. "JFR'ın production'da güvenli olmasının sebebi (overhead): ____."
10. "Xms = Xmx neden production'da kural: ____."
