# Topic 3.10 — Performance Tools (JFR, async-profiler, MAT, jstack)

## Hedef

Production banking servisinde **performans sorunlarını teşhis etme** araçlarını öğrenmek. JFR (Java Flight Recorder), async-profiler ile CPU/allocation/lock profiling, MAT (Eclipse Memory Analyzer) ile heap leak hunting, jstack ile deadlock detection. Banking'de "uygulamamız yavaş" sorunu geldiğinde **5 dakika içinde root cause** bulabilen developer olmak.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 30 dk • Toplam: ~5 saat

## Önbilgi

- Topic 3.9 (JVM Internals) bitti
- Heap dump, thread dump kavramları tanıdık
- Bir process'in `pid`'ini bulmayı biliyorsun (`jps`, `ps`)

---

## Kavramlar

### 1. Performans sorunlarının taksonomisi

Banking'de gelen şikayet: "Uygulama yavaş." Önce **ne tür yavaşlık**?

| Şikayet | İlk şüpheli |
|---|---|
| Tüm endpoint yavaş | CPU bottleneck, GC, DB |
| Spesifik endpoint yavaş | O endpoint'in kod path'i |
| Zaman zaman yavaş ("hıçkırık") | Long GC pause, lock contention |
| Yavaşlık zaman geçtikçe artıyor | Memory leak, connection leak |
| OOM | Memory leak veya yetersiz heap |
| Deadlock — hiçbir şey ilerlemiyor | Thread blocked, lock issue |
| Yüksek CPU | Hot method, infinite loop |

Her birinin **farklı araç**ı var. Doğru aracı seçmek vakit kazandırır.

### 2. jps, jstack, jmap, jstat — JDK built-in araçlar

#### `jps` — Java process'leri listele

```bash
jps -lv

12345 com.mavibank.banking.CoreBankingApplication -Xmx4g -XX:+UseG1GC
67890 com.mavibank.fraud.FraudApplication -Xmx2g
```

#### `jstack <pid>` — thread dump

```bash
jstack 12345 > threads.txt
```

Bir snapshot tüm thread'lerin durumunu yazar. **Deadlock'u jstack otomatik tespit eder**:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x000000076a..., a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f... (object 0x000000076a..., a java.lang.Object),
  which is held by "Thread-1"
```

Banking pattern: Concurrent transfer A→B + B→A deadlock'u burada **karşına çıkar** (Topic 3.3'teki demo).

#### `jstack` thread state'leri

| State | Anlamı |
|---|---|
| RUNNABLE | CPU'da çalışıyor veya hazır |
| BLOCKED | Monitor lock için bekliyor |
| WAITING | `wait()`, `park()` çağrısı bekliyor |
| TIMED_WAITING | `sleep()`, `wait(timeout)` bekliyor |

**Banking analizi:** Tüm thread'ler BLOCKED → lock contention. Hepsi WAITING → idle pool. Pattern olarak bakman gereken şey.

#### `jmap` — heap dump

```bash
jmap -dump:live,format=b,file=heap.hprof 12345
```

`live` = sadece reachable objeleri dump'la (daha küçük, daha temiz). Production'da `live` kullan.

**Tuzak:** `jmap` STW yapar — production'da dikkat. **`jcmd <pid> GC.heap_dump` daha az invasive**.

#### `jstat` — GC monitoring

```bash
jstat -gc 12345 1000
```

Her saniye GC istatistiği:

```
 S0C    S1C    S0U    S1U    EC       EU       OC      OU    MC     MU
1024.0 1024.0  0.0   512.0  8192.0   3120.0  16384.0 12340.0  ...
```

- `S0C/S1C`: Survivor capacity
- `EC/EU`: Eden capacity/used
- `OC/OU`: Old capacity/used

Production'da Prometheus + Micrometer ile sürekli bunu izle (Phase 9).

#### `jcmd` — Swiss Army knife

```bash
jcmd <pid> help                       # listele
jcmd <pid> GC.heap_dump heap.hprof    # heap dump (jmap'ten iyi)
jcmd <pid> GC.run                     # full GC tetikle
jcmd <pid> Thread.print               # jstack equivalent
jcmd <pid> VM.flags                   # JVM flag'leri
jcmd <pid> VM.native_memory           # NMT
jcmd <pid> JFR.start                  # JFR başlat
jcmd <pid> JFR.dump filename=app.jfr
```

**Banking pratik tavsiyesi:** `jcmd`'i öğren. `jstack`, `jmap` ayrı ayrı yerine `jcmd` kapsayıcı.

### 3. Java Flight Recorder (JFR)

JFR JVM'in **dahili** profiling sistemi. **Production-safe** (~%1-2 overhead).

#### Continuous recording (production)

```bash
java -XX:StartFlightRecording=settings=profile,duration=2h,filename=banking.jfr \
     -jar app.jar
```

`settings=profile` — detaylı, `settings=default` — overhead'siz minimum.

#### On-demand recording

```bash
# Başlat
jcmd 12345 JFR.start name=banking duration=60s

# Status
jcmd 12345 JFR.check

# Dump
jcmd 12345 JFR.dump name=banking filename=banking.jfr
```

#### JFR events kategorileri

- **CPU:** Method sampling, native, etc.
- **Memory allocation:** TLAB, outside TLAB
- **GC:** All gc events, pause times
- **Lock:** Monitor wait, blocked, contention
- **Thread:** Sleep, park, end, dump
- **IO:** Socket read/write, file read/write
- **JIT:** Compilation, deopt
- **Custom:** Sen kendi event'ini fırlatabilirsin

#### JMC ile analiz

JDK Mission Control:
- Heap allocations → hot path
- Latency events → slow operations
- Lock contention → contended monitor
- TLAB allocations → hot constructor

#### Custom JFR event (banking örneği)

```java
import jdk.jfr.*;

@Name("com.mavibank.TransferExecuted")
@Label("Transfer Executed")
@Category("Banking")
public class TransferExecutedEvent extends Event {
    
    @Label("Amount")
    public double amount;
    
    @Label("Currency")
    public String currency;
    
    @Label("Duration ms")
    public long durationMs;
}

// Usage
TransferExecutedEvent event = new TransferExecutedEvent();
event.begin();
try {
    transferService.execute(...);
    event.amount = amount.doubleValue();
    event.currency = currency.getCurrencyCode();
} finally {
    event.commit();
}
```

JMC'de bu custom event'ler görünür, banking-specific tracing.

### 4. async-profiler — alternatif CPU/allocation profiler

async-profiler (https://github.com/async-profiler/async-profiler) — Linux'ta perf_events kullanır, çok düşük overhead.

#### CPU profiling

```bash
./profiler.sh -d 60 -f cpu.html 12345
```

60 saniye CPU profiling, HTML flame graph üretir.

#### Allocation profiling

```bash
./profiler.sh -e alloc -d 60 -f alloc.html 12345
```

#### Lock profiling

```bash
./profiler.sh -e lock -d 60 -f lock.html 12345
```

Hangi thread hangi monitor'da blocked kaldı.

#### Wall-clock profiling

```bash
./profiler.sh -e wall -d 60 -f wall.html 12345
```

CPU + blocked time — thread'in tam ne yaptığı.

### 5. Flame graph okuma

Flame graph y-ekseni call stack, x-ekseni zaman (genişlik = harcanan süre).

```
   ┌────────────────────────────────────────┐
   │       Account.deposit (5%)              │
   ├──────────────────────┬─────────────────┤
   │  Money.add (3%)      │ JpaRepo.save (40%)│
   ├──────────────────────┴───────┬─────────┤
   │  Hibernate flush (30%)        │ ...    │
   ├───────────────────────────────┤        │
   │  PreparedStatement.execute    │        │
   │  (28%)                        │        │
   └───────────────────────────────┴────────┘
```

**Yorum:** `JpaRepo.save` toplam zamanın 40%'ını alıyor. İçinde `Hibernate flush` 30%, onun da içinde `PreparedStatement.execute` 28%. **Bottleneck:** DB.

Renkler:
- Yeşil: Java
- Sarı: JVM
- Kırmızı: native code, kernel

**Banking pratiği:** Slow endpoint'i CPU profile et, flame graph'a bak, en geniş block neyse onunla başla.

### 6. MAT (Eclipse Memory Analyzer Tool)

Heap dump analizi için altın standart.

#### Workflow

1. Heap dump al (`jcmd` ile)
2. MAT'ı aç, `.hprof` dosyasını yükle
3. **Leak Suspects Report** otomatik çalışır

#### Leak Suspects

MAT'ın AI'sı (basit heuristic) "Bu nesne abartılı yer kaplıyor, leak şüphesi" der.

Banking örneği:
```
Problem Suspect 1
The class "java.util.concurrent.ConcurrentHashMap$Node[]" 
occupies 850,431,232 (85%) bytes.
The memory is accumulated in one instance of 
"java.util.concurrent.ConcurrentHashMap$Node[]" 
loaded by "<system class loader>".

Keywords:
java.util.concurrent.ConcurrentHashMap
```

→ Bir `ConcurrentHashMap` heap'in 85%'ini tutuyor. Bu cache **bounded değil mi?**

#### Dominator Tree

Hangi obje ne kadar memory'i **retain** ediyor (kendisi + sadece kendi referans verdiği)?

```
Dominator Tree
├── com.mavibank.AccountCache (700MB retained)
│   └── ConcurrentHashMap (700MB)
│       └── 5,000,000 Node entries
├── HikariCP DataSource (50MB)
└── ...
```

`AccountCache` 700MB retained → ana suspect.

#### Path to GC Roots

"Bu obje neden hâlâ memory'de?" sorusu.

```
Path to GC Roots:
java.lang.Thread (main) 
  → AccountCache.instance (static field)
    → ConcurrentHashMap
      → Node[5000000]
```

`AccountCache.instance` static → JVM yaşadığı sürece tutuluyor.

#### OQL (Object Query Language)

SQL-like syntax ile heap'te query:

```sql
SELECT * FROM com.mavibank.Account WHERE balance > 1000000
```

Banking: "Heap'te kaç hesap var, kaçı 1M+ bakiyeli" gibi sorular.

### 7. FastThread.io — thread dump analysis

Thread dump'ı yükle, otomatik analiz:

- Thread state distribution (kaç RUNNABLE, kaç BLOCKED)
- Hot threads (en çok CPU)
- Lock contention
- Deadlock detection
- Thread pool saturation

Banking'de "uygulama yanıt vermiyor" şikayetinde 3-4 thread dump al (5 dk arayla), FastThread'e yükle, pattern gör.

### 8. Heap dump alma — production-safe yollar

#### Yöntem 1: jcmd (önerilen)

```bash
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

#### Yöntem 2: jmap

```bash
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>
```

`live` flag → STW yapar ama daha küçük dump.

#### Yöntem 3: HeapDumpOnOutOfMemoryError

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heap.hprof
```

OOM olunca otomatik dump.

#### Yöntem 4: Spring Boot Actuator endpoint

```yaml
management:
  endpoints:
    web:
      exposure:
        include: heapdump
```

```bash
curl http://localhost:8080/actuator/heapdump > heap.hprof
```

**Banking güvenlik:** Actuator endpoint'i prod'da yalnız internal/admin için aç.

### 9. Banking incident playbook

#### Incident: "API yavaş, p99 normalde 100ms şimdi 2s"

**5 dakikalık triaj:**

1. **jcmd Thread.print** — thread dump al
   - Çoğu thread BLOCKED? → lock contention
   - Çoğu thread WAITING? → external service slow, connection pool exhausted
   - Çoğu thread RUNNABLE? → CPU bottleneck
2. **jstat -gc** — GC frekans/duration
   - Sık major GC? → memory issue
3. **GC log son 5 dakika** — pause history
4. **JFR son 60 sn** — hot method'lar

#### Incident: "Memory sürekli büyüyor, restart sonrası 1 saatte OOM"

1. **jmap -histo** — class instance counts
   - Beklenmeyen büyük class collection?
2. **Heap dump al** — `jcmd GC.heap_dump`
3. **MAT analizi** — Dominator tree
4. **Leak suspect:** static collection, ThreadLocal, cache unbounded

#### Incident: "Uygulama yanıt vermiyor, hung"

1. **jcmd Thread.print** — birden fazla, 30 sn arayla
2. **Deadlock yakala** — jstack header'ında "Found Java-level deadlock"
3. **Tüm thread WAITING/BLOCKED** → external dependency yanıt vermiyor
4. **Tek bir lock'ta yığılma** → lock contention veya zombi lock holder

### 10. Production'da continuous monitoring (önbakış)

Phase 9'da detaylandıracağız ama:
- Prometheus + Grafana — metrik
- Distributed tracing (Jaeger, Tempo) — request flow
- Continuous JFR — sürekli kayıt
- APM (Elastic, Datadog) — paid alternatives

İlk araç bilgisi olmadan bunlar boş laf. Önce `jstack`, `jmap`, JFR, MAT öğren.

---

## Önemli olabilecek araştırma kaynakları

- "Java Performance: The Definitive Guide" (Scott Oaks)
- JDK Mission Control documentation
- async-profiler GitHub README
- Eclipse Memory Analyzer Tool documentation
- FastThread.io blog (thread dump örnekleri)
- Brendan Gregg's website (flame graph mucidi)
- Andrei Pangin (async-profiler yazarı) tech talks
- "JVM Anatomy" by Aleksey Shipilev — performance series

---

## Mini task'ler

### Task 3.10.1 — jcmd hızlı test (15 dk)

`core-banking` çalışırken:

```bash
jps -lv                                    # pid bul
jcmd <pid> help                            # mevcut command'ları gör
jcmd <pid> VM.flags                        # JVM flag'leri
jcmd <pid> Thread.print | head -100        # thread dump
jcmd <pid> GC.heap_info                    # heap detayı
```

Output'ları **defterine** yapıştır.

### Task 3.10.2 — Deadlock üret ve jstack ile yakala (45 dk)

Topic 3.3'teki deadlock kod parçasını çalıştır. `jstack <pid>` ile yakala. Header'da "Found one Java-level deadlock" satırını gör, stack trace'i incele.

Lock ordering uygulayıp yeniden çalıştır, deadlock kaybolduğunu doğrula.

### Task 3.10.3 — Memory leak ile MAT analizi (60 dk)

`core-banking`'e leaky cache ekle:

```java
@Component
public class LeakyCache {
    private final Map<UUID, byte[]> store = new HashMap<>();   // unbounded, no eviction
    
    public void put(UUID key, byte[] value) {
        store.put(key, value);
    }
}
```

Bir endpoint ekle: her çağrıda 1MB veri cache'le.

```bash
java -Xmx256m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./oom.hprof \
     -jar core-banking.jar
```

300 request gönder → OOM → `oom.hprof` oluşur.

MAT ile aç:
1. Leak Suspects Report — leaky class'ı bul
2. Dominator Tree — toplam retained görüş
3. Path to GC Roots — neden tutulduğu

**Defterine** screenshot ve yorum yaz.

### Task 3.10.4 — JFR ile production-safe profiling (45 dk)

```bash
java -XX:StartFlightRecording=settings=profile,duration=2m,filename=banking.jfr \
     -jar core-banking.jar
```

2 dakika boyunca Gatling ile 100 req/sec yük ver.

JMC ile `banking.jfr` aç:
- **CPU sample** view → hot method'lar
- **GC** view → pause times
- **Lock contention** → çekişme
- **Allocation** → büyük allocation'lar

**Defterine** en hot 3 method'u yaz.

### Task 3.10.5 — async-profiler ile flame graph (45 dk)

async-profiler indir (https://github.com/async-profiler/async-profiler).

```bash
./profiler.sh -d 60 -f cpu.html <pid>
```

60 saniye yük altında profile, `cpu.html`'i tarayıcıda aç. Flame graph'ı incele:
- Genişliği en büyük block neyi gösteriyor?
- Banking-specific method'lar üst seviyede mi?

### Task 3.10.6 — Custom JFR event ekle (30 dk)

`core-banking`'in transfer service'ine `TransferExecutedEvent` ekle (yukarıdaki örnek). 100 transfer yap, JFR kaydı al, JMC'de bu custom event'i gör.

---

## Test yazma rehberi

Bu topic daha çok manuel diagnostic tool kullanımı. Test yazımı diğer topic'lerden farklı:

### Test 3.10.1 — Manual diagnostic snippet

```java
@Test
@Disabled("Manuel — heap dump alıp MAT'la incelemek için")
void manualHeapDumpCapture() throws IOException, AttachNotSupportedException {
    String pid = ManagementFactory.getRuntimeMXBean().getName().split("@")[0];
    
    HotSpotDiagnosticMXBean bean = ManagementFactory.newPlatformMXBeanProxy(
        ManagementFactory.getPlatformMBeanServer(),
        "com.sun.management:type=HotSpotDiagnostic",
        HotSpotDiagnosticMXBean.class
    );
    
    bean.dumpHeap("./manual-heap.hprof", true);
    System.out.println("Heap dump: ./manual-heap.hprof");
}
```

### Test 3.10.2 — Programmatic JFR

```java
@Test
void jfrCustomEventShouldBeCaptured() throws Exception {
    try (Recording recording = new Recording()) {
        recording.enable("com.mavibank.TransferExecuted");
        recording.start();
        
        // Trigger event
        TransferExecutedEvent ev = new TransferExecutedEvent();
        ev.begin();
        ev.amount = 100.0;
        ev.currency = "TRY";
        ev.commit();
        
        recording.stop();
        recording.dump(Path.of("test.jfr"));
        
        // Read back
        List<RecordedEvent> events;
        try (RecordingFile file = new RecordingFile(Path.of("test.jfr"))) {
            events = new ArrayList<>();
            while (file.hasMoreEvents()) {
                events.add(file.readEvent());
            }
        }
        
        assertThat(events).hasSize(1);
        RecordedEvent first = events.get(0);
        assertThat(first.getDouble("amount")).isEqualTo(100.0);
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki performance tooling çalışmamı banking-grade kriterlere göre değerlendir. 
Sadece eksikleri ve yanlışları işaretle:

1. JDK built-in tools:
   - jcmd, jstack, jmap, jstat ile pratik deneyim var mı?
   - `jcmd Thread.print` çıktısını okuyabiliyor musun?
   - Heap dump için `jcmd GC.heap_dump` mu kullandın (jmap'ten daha güvenli)?

2. JFR:
   - Production-safe continuous recording konfigürasyonu var mı?
   - `settings=profile` (detaylı) veya `settings=default` (minimum) seçimi bilinçli mi?
   - JMC ile analiz akışı denenmiş mi?
   - Custom JFR event ile banking-domain instrumentation var mı?

3. MAT (Memory Analyzer):
   - Bir heap dump'ı MAT ile analiz etmiş misin?
   - Leak Suspects Report yorumlanmış mı?
   - Dominator Tree ile retained heap görülmüş mü?
   - Path to GC Roots ile leak source bulunmuş mu?

4. async-profiler:
   - CPU flame graph oluşturulmuş mu?
   - Allocation profiling (`-e alloc`) yapılmış mı?
   - Lock contention (`-e lock`) profile edilmiş mi?

5. Production safety:
   - jmap STW etkisi bilinçli mi (production'da dikkatli)?
   - Heap dump dosyaları güvenli yerde mi (PII içerebilir)?
   - Spring Boot Actuator /heapdump endpoint'i auth korumalı mı?

6. Banking incident playbook:
   - Slow endpoint için triaj sırası net mi?
   - Memory leak için heap dump → MAT akışı denenmiş mi?
   - Deadlock için jstack analiz akışı uygulanmış mı?

7. Continuous monitoring (Phase 9 önbakış):
   - Production'da JFR sürekli açık mı?
   - GC log rotation yapılandırılmış mı?
   - Heap dump dizini disk space alarmı ile izleniyor mu?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `jcmd help` ile mevcut komutları gördüm
- [ ] `jcmd Thread.print` ile thread dump aldım, state'leri okudum
- [ ] Bilerek deadlock üretip jstack ile yakaladım
- [ ] `jcmd GC.heap_dump` ile heap dump aldım
- [ ] MAT ile bir leaky cache senaryosunu analiz ettim
- [ ] JFR continuous recording konfigüre ettim
- [ ] JMC ile JFR dosyasını açıp hot path'leri inceledim
- [ ] async-profiler ile flame graph oluşturdum
- [ ] Custom JFR event yazıp banking-specific tracing ekledim
- [ ] FastThread.io'ya thread dump yükleyip analiz aldım
- [ ] Banking incident playbook'unu deftere yazdım

---

## Defter notları

1. "Performans sorunu için araç seçim matrisi (memory leak / slow endpoint / deadlock): ____."
2. "`jstack`'in thread state'leri (RUNNABLE/BLOCKED/WAITING/TIMED_WAITING) banking interpretation: ____."
3. "`jcmd` ile `jmap`/`jstack` farkı (neden jcmd tercih): ____."
4. "JFR'ın production-safe olmasının sebebi (overhead): ____."
5. "MAT Leak Suspects Report'un altında yatan heuristic: ____."
6. "Dominator Tree ile shallow heap vs retained heap farkı: ____."
7. "async-profiler vs JFR farkı: ____."
8. "Flame graph okumayı 30 saniyede özetle: ____."
9. "Custom JFR event ile banking-specific tracing neden değerli: ____."
10. "Production heap dump için /actuator/heapdump endpoint güvenliği: ____."
