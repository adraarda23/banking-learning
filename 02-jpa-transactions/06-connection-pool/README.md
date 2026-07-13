# Topic 2.6 — HikariCP & Connection Pool Tuning

## Hedef

Connection pool'un (özellikle HikariCP) **iç mekaniğini** (ConcurrentBag, fast-path, housekeeping) anlamak. Banking workload'una uygun pool sizing yapabilmek (Brian Goetz formülü + banking adaptasyonu). Tüm kritik HikariCP parametrelerini (`maximumPoolSize`, `connectionTimeout`, `idleTimeout`, `maxLifetime`, `keepaliveTime`, `leakDetectionThreshold`, `validationTimeout`) banking context'inde doğru ayarlamak. Connection leak'i `leakDetectionThreshold` ile yakalamak, pool exhaustion'ı `jstack` ile teşhis etmek. Micrometer ile pool metric'lerini Prometheus'a yazıp Grafana'da gözlemlemek. K8s multi-replica deployment'larında PgBouncer'ın transaction pooling mode'unu neden ve nasıl kullandığını öğrenmek.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1.5 saat • Toplam: ~6 saat

## Önbilgi

- Topic 2.1-2.5 bitti — JPA mekaniği, transaction, locking, N+1 konularını biliyorsun
- "Connection pool" deyimini duydun, Spring Boot'un default'ta HikariCP kullandığını biliyorsun
- DB'ye bağlanmanın TCP handshake + TLS + authentication ile maliyetli olduğunu en az bir kere düşündün
- Threading'de pool kavramına aşinasın (`ExecutorService`, thread pool)

---

## Kavramlar

### 1. Neden connection pool — banking perspektifi

Her DB connection açma maliyeti:

- TCP 3-way handshake (~1-50ms, network'e göre)
- TLS handshake (varsa, +50-200ms)
- DB authentication (PostgreSQL pg_hba check, password verify, ~5-20ms)
- DB session initialization (timezone, search_path, ~5ms)

Toplam: 100-300ms her connection. Bir banking API endpoint'i 50ms hedef SLA'da çalışıyorsa, her request'te yeni connection açmak kabul edilemez.

**Pool çözümü:**

1. Uygulama başlatılınca pool N adet connection açar (`minimumIdle`).
2. Her request başlatıldığında pool'dan connection ödünç alır.
3. Request bitince connection pool'a geri verilir, kapanmaz.
4. İhtiyaç olursa pool büyür (`maximumPoolSize`'a kadar).

**Banking için neden kritik:**

- Yüksek throughput (saniyede binlerce request) — her birinde connection açmak imkânsız
- DB connection limit'i sınırlı (PostgreSQL default `max_connections = 100`) — savurganlık DB'yi öldürür
- Predictable latency — connection ödünç alma 0.5ms, açma 100-300ms

---

### 2. HikariCP — Spring Boot default

HikariCP (Hikari = "ışık" Japonca) Brett Wooldridge tarafından yazılmış, **en hızlı** connection pool. Spring Boot 2.0+'tan beri default.

**Neden hızlı:**

1. **ConcurrentBag** — özel bir lock-free data structure. Birden fazla thread aynı anda connection alıp/iade edebilir.
2. **Thread-local fast-path** — son kullanılan thread, son kullandığı connection'a öncelikli erişim. Cache-friendly.
3. **Bytecode optimization** — proxy class'lar `Wrapper` pattern yerine custom Javassist ile, minimum overhead.
4. **Lazy connection acquisition** — connection request anında bağlanır, batch acquisition yok.

**Diğer pool'lar (kıyas için):**

- **Apache DBCP2** — eski, slower, daha az feature
- **C3P0** — eski, "feature-rich" ama yavaş
- **Tomcat JDBC Pool** — Tomcat'ın default'u, HikariCP ortaya çıkana kadar yaygın
- **Oracle UCP** — Oracle DB'ye özel, RAC awareness

**Banking pratiği:** HikariCP. Tartışmasız.

---

### 3. HikariCP mimarisi — ConcurrentBag içgörü

```
                  ┌───────────────────────────────────┐
                  │    HikariCP                        │
                  │                                    │
                  │   ┌──────────────────────────┐     │
                  │   │   ConcurrentBag           │    │
                  │   │  ┌──┬──┬──┬──┬──┬──┐     │    │
                  │   │  │C1│C2│C3│C4│C5│C6│ ... │    │
                  │   │  └──┴──┴──┴──┴──┴──┘     │    │
                  │   │  ThreadLocal handoff      │    │
                  │   └──────────────────────────┘     │
                  │              ↑                     │
                  │   ┌─────────────────────────┐      │
                  │   │  Housekeeping thread     │     │
                  │   │  (idle timeout, max life) │    │
                  │   └─────────────────────────┘      │
                  └───────────────────────────────────┘
                              ↑
                  ┌───────────┴───────────┐
                  │   Application threads  │
                  │   (Tomcat NIO workers) │
                  └───────────────────────┘
```

**`getConnection()` flow:**

1. Thread-local'da daha önce kullanılan connection var mı? → varsa direkt al (fast-path, ~50ns).
2. ConcurrentBag'ta `STATE_NOT_IN_USE` connection var mı? → varsa CAS ile alın.
3. Pool maksimuma ulaşmadıysa yeni connection yarat.
4. Aksi takdirde `connectionTimeout` kadar bekle. Timeout olursa exception.

`return connection`: ConcurrentBag'a iade, `STATE_NOT_IN_USE`. Thread-local'a referans bırak (next time fast-path için).

**Banking implikasyonu:** Yüksek throughput (saniyede binlerce `getConnection`) için thread-local fast-path **çok değerli**. Yine de yanlış sizing → ConcurrentBag bekleme → kuyruk.

---

### 4. Kritik parametreler — banking-grade kalibrasyon

#### `maximumPoolSize`

Pool'un maksimum connection sayısı. **En önemli parametre.**

**Brian Goetz formülü:**

```
connections = ((core_count * 2) + effective_spindle_count)
```

- `core_count` = DB sunucusunun CPU core sayısı (Java app'inin değil!)
- `effective_spindle_count` = disk paralelliği (HDD: 1, SSD: 4-8, NVMe: 8-16)

Örnek: DB 8 core, SSD → `(8 * 2) + 4 = 20`.

**Banking adaptasyonu:**

Bu formül *worker thread* sayısı, *uygulama instance* sayısı değil. K8s'te 4 replica varsa, her replica 20 alırsa DB'ye toplam 80 connection gider. DB `max_connections = 100` ise kalan 20 başka servisler için. Plan:

```
per_replica_pool_size = (db_max_connections - reserved) / replica_count
```

`reserved` = admin, monitoring, başka uygulamalar için (~10-20%).

```
20 = (100 - 20) / 4
```

**Banking kuralı:**
- API service (kısa transaction) → 10-20 connection per replica
- Batch service (uzun transaction) → 5-10 per replica
- Reporting service (read-heavy) → 15-25 per replica

**Aşırı büyük pool tehlikeleri:**

1. **Context switching overhead** — DB sunucusunda binlerce backend process, scheduler bunalır.
2. **Connection limit'i aşımı** — DB diğer servisleri reddeder.
3. **Memory** — her PostgreSQL connection ~10MB shared memory + 5-50MB query plan cache.
4. **Replication lag** — write replica üzerinde çok çakışma.

**Aşırı küçük pool tehlikeleri:**

1. **`getConnection` timeout** — kuyruk birikir, request fail.
2. **P99 latency patlama** — bazı request'ler bekler.
3. **Throughput tavanı** — uygulama hızlı ama pool dar.

**Sizing süreci:**
1. Gatling/k6 ile load test
2. `HikariPool.PendingThreads` metric'i artıyor mu? → pool dar
3. P95 latency artıyor mu? → pool dar
4. DB CPU/load saturate mı? → DB darboğaz, pool büyütme yardım etmez

#### `minimumIdle`

Pool'da minimum kaç connection idle bekler. **HikariCP önerisi:** `maximumPoolSize` ile aynı (yani fixed-size pool).

```yaml
hikari:
  maximum-pool-size: 20
  minimum-idle: 20
```

**Neden fixed-size:** Pool'un büyüme-küçülme döngüsü her zaman 100-300ms gecikme yaratır (yeni bağlantı açma). Banking gibi predictable latency isteyen sistemlerde sürekli max'a tutmak iyi.

**Dinamik pool ne zaman:** Düşük trafik gece (1-2 connection yeter), yüksek trafik gündüz (20). Cost-saving.

#### `connectionTimeout`

`getConnection()` çağrısının bekleyebileceği maksimum süre. Pool dolduğunda, free connection için kaç ms bekler.

**Default:** 30000ms (30 saniye). **Banking için yüksek.**

**Banking önerisi:**

- API endpoint: 2-5 saniye. Daha uzun süre yan etkiler (HTTP timeout, kullanıcı bekler).
- Batch job: 30-60 saniye OK.

```yaml
hikari:
  connection-timeout: 3000   # 3 saniye
```

**Timeout aşımında:** `SQLException` ("HikariPool-1 - Connection is not available, request timed out after 3000ms"). Spring tarafında `CannotGetJdbcConnectionException`. Request 503 dönmeli (downstream sorun).

#### `idleTimeout`

Idle bir connection ne kadar süre boşta kalırsa kapanır. Sadece `minimumIdle < maximumPoolSize` olduğunda anlamlı.

**Default:** 600000ms (10 dk).

**Banking önerisi:** Fixed-size pool kullanıyorsan 0 (idle timeout off). Dynamic ise 5-10 dk.

#### `maxLifetime` — en sık kaçırılan parametre

Bir connection maksimum kaç ms yaşar. Kapanış nazikçe yapılır (idle olduğunda, ya da kullanıcı return ettiğinde).

**Neden gerekli:**

1. **DB-side timeout protection** — PostgreSQL'in `idle_in_transaction_session_timeout` veya firewall'un session timeout'una takılmadan önce pool kendisi kapatır.
2. **DB driver memory leak'leri** — JDBC driver bazı bug'lara sahip, connection'ı recycle etmek temizler.
3. **DB upgrade durumunda** — eski versiyonun connection'larını yenilemek.

**Banking formülü:** `maxLifetime = DB-side timeout - 30 saniye`.

Örnek: PostgreSQL `idle_in_transaction_session_timeout = 30 min`, AWS RDS proxy 5 dakika timeout.

```yaml
hikari:
  max-lifetime: 270000    # 4.5 dk (RDS proxy 5dk - 30s safety)
```

**Default:** 1800000ms (30 dk). Çoğu durumda OK.

**Asla DB-side timeout'tan büyük olmasın** — yoksa "connection has been closed" hataları başlar.

#### `keepaliveTime`

Connection'ı keep-alive ping ile canlı tutar (firewall idle timeout'undan korur). Spring Boot 2.3+, HikariCP 4.0+.

```yaml
hikari:
  keepalive-time: 30000    # 30 saniye
```

**Banking pratiği:** K8s + cloud provider firewall (idle 60s drop) varsa zorunlu.

#### `leakDetectionThreshold`

Bir connection bu süreden uzun süre ödünç alınırsa (return edilmediyse), log'a stack trace düşer. Default 0 (off).

**Banking önerisi:** Dev'de 30000 (30s), prod'da 60000 (60s).

```yaml
hikari:
  leak-detection-threshold: 30000
```

Çıktı:

```
WARN  c.z.h.p.ProxyLeakTask - Connection leak detection triggered for 
connection org.postgresql.jdbc.PgConnection@7a8c, stack trace follows:
  at com.mavibank.banking.account.service.AccountService.someMethod(AccountService.java:42)
  ...
```

**Banking pratiği:** Dev environment'ta `leakDetectionThreshold` aktif, prod'da daha yüksek değerle (cron job'lar uzun TX gerekebilir).

**Tuzak:** Bu log gelirse **kod bug'u var**. Connection ödünç alıp `close()` çağrılmadı (genelde `try-with-resources` eksikliği). Hemen düzelt.

#### `validationTimeout`

Pool connection'ı validate ederken (uygulama başlatma veya periyodik check) kaç ms bekler.

**Default:** 5000ms.

**Banking pratiği:** Default OK.

#### `autoCommit`

```yaml
hikari:
  auto-commit: true   # default
```

Spring `@Transactional` kullanıyorsa autoCommit her zaman false ayarlanır TX süresince. Yani **bu parametre büyük etki yapmaz** — Spring transaction abstraction zaten kontrol ediyor.

#### `transaction-isolation`

```yaml
hikari:
  transaction-isolation: TRANSACTION_READ_COMMITTED
```

Pool seviyesinde default isolation. PostgreSQL'in default'u READ_COMMITTED. Banking için bunu açıkça yazmak iyi (explicit is better than implicit).

---

### 5. Pool connection lifecycle

```
                ┌─────────────────────┐
                │     CREATED         │  (housekeeping yeni connection)
                └──────────┬──────────┘
                           ↓
                ┌─────────────────────┐
                │     IDLE            │  (havuzda bekliyor)
                └──┬──────────────┬───┘
                   ↓              ↑ (close → return)
                ┌─────────┐    ┌──────────┐
                │ BORROWED │←──│ RESET    │
                └─────────┘    └──────────┘
                   ↓
                Application uses
                   ↓
                close() → reset → IDLE
                
                IDLE + maxLifetime exceeded → CLOSED
                IDLE + idleTimeout + (count > minimumIdle) → CLOSED
```

**Reset davranışı:** HikariCP connection'ı iade ederken default'ları restore eder (`autoCommit`, `transaction isolation`, `read-only`, vb.). Bu yüzden uygulama tarafında bunları değiştirmek pool'un işine yarar — sonraki kullanıcı için tutarlı state.

---

### 6. Banking workload tipleri ve sizing

#### Tip A — API service (kısa transaction)

- Her endpoint: 50-200ms execution, ~10-50ms TX
- Yüksek RPS (saniyede 100-1000+)
- **Pool size formülü:** `RPS * avg_tx_time_seconds = concurrent_connections`
- 500 RPS * 0.05s = 25 connection per replica

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 25
      minimum-idle: 25
      connection-timeout: 3000
      max-lifetime: 270000
      keepalive-time: 30000
      leak-detection-threshold: 60000
```

#### Tip B — Batch service (uzun transaction)

- Her job: 5 dakika - 1 saat
- Düşük concurrency (1-10 paralel job)
- Pool **küçük** (her connection uzun süre ödünç)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 30000    # batch için tolerans
      max-lifetime: 1800000        # 30 dk OK
      leak-detection-threshold: 600000   # 10 dk
```

#### Tip C — Reporting service (read-heavy)

- Çok sayıda concurrent select
- TX kısa
- Replica'ya yönlendirilebilir (Phase 9 detay)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 30
      connection-timeout: 5000
      max-lifetime: 270000
```

---

### 7. Monitoring — Micrometer + Prometheus + Grafana

HikariCP `MetricRegistry` ile entegre. Spring Boot Actuator + Micrometer ile otomatik.

`pom.xml`:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

`application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  metrics:
    enable:
      hikaricp: true
```

**Önemli metric'ler:**

| Metric | Anlam | Alarm eşiği |
|---|---|---|
| `hikaricp.connections.active` | Şu anda kullanılan | `> pool_size * 0.9` |
| `hikaricp.connections.idle` | Boş bekleyenler | `< 1` (pool tükeniyor) |
| `hikaricp.connections.pending` | Connection bekleyen thread | `> 0` darboğaz |
| `hikaricp.connections.timeout` | Timeout sayısı (cumulative) | `> 0/min` |
| `hikaricp.connections.acquire` | Connection alma süresi histogram | P95 > 100ms |
| `hikaricp.connections.usage` | Connection ödünç süresi histogram | P95 > 500ms (uzun TX uyarısı) |
| `hikaricp.connections.creation` | Yeni connection açma süresi | İlk açılışta ölçülür |

`/actuator/prometheus` endpoint'inde scrape edilen örnek:

```
hikaricp_connections_active{pool="HikariPool-1"} 18.0
hikaricp_connections_idle{pool="HikariPool-1"} 2.0
hikaricp_connections_pending{pool="HikariPool-1"} 5.0
```

**Grafana dashboard fikirleri:**

- Active vs idle stack chart
- Pending threads alert (> 0 → uyarı)
- Acquire P95 / P99 — pool throughput sağlığı
- Connection age (ortalama) — `maxLifetime` çalışıyor mu kontrol

---

### 8. Pool exhaustion troubleshooting

**Belirtiler:**
- Endpoint'lerde sporadic 503 / 504
- Log'da `HikariPool-1 - Connection is not available`
- P95 latency dramatic spike

**Tanı 1: Metrics**

```
hikaricp_connections_active = max_pool_size
hikaricp_connections_pending > 0
hikaricp_connections_timeout artıyor
```

**Tanı 2: jstack — neden connection'lar tutuluyor?**

```bash
jps                                     # Java PID bul
jstack -l <pid> > thread-dump.txt
```

Çıktıda `getConnection` çağrısında BLOCKED thread'ler ve hangi connection'ı tuttuğu görünür.

Aramak: `at com.zaxxer.hikari.pool.HikariPool.getConnection`

Bloklananlar pool'dan connection bekliyor. Hangi thread'ler connection'ı *tutuyor* — onlar genelde uzun süreli DB call yapıyor olur:

```
"http-nio-8080-exec-5" #45 prio=5 ...
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.read(SocketInputStream.java:171)
        ...
        at org.postgresql.jdbc.PgPreparedStatement.executeQuery
        ...
        at com.mavibank.banking.report.ReportService.generateLongQuery
```

Bu thread `generateLongQuery` içinde 30 saniye bekliyor. Pool tüketici.

**Çözüm yolları:**

1. Long query'i optimize et (index, JOIN, batch fetching)
2. Query timeout koy (`@QueryHints` veya `statement_timeout`)
3. TX kısalt — uzun TX'i kır parçalara
4. Pool size'ı artır (geçici, kök neden değil)
5. Long-running query'i farklı service'e taşı (reporting service)

**Banking kuralı:** Pool exhaustion daima **kök neden** (uzun TX, N+1, lock contention). Pool size artırma sadece **palyatif**.

---

### 9. Connection leak — gerçek bug

```java
@Service
public class BadService {
    @Autowired DataSource dataSource;
    
    public void badMethod() {
        try {
            Connection conn = dataSource.getConnection();   // ❌ try-with-resources yok
            // ... use conn
        } catch (SQLException e) {
            log.error(e);
        }
        // conn.close() çağrılmadı → leak
    }
}
```

`leakDetectionThreshold` ile yakalanır. Düzeltme:

```java
public void goodMethod() {
    try (Connection conn = dataSource.getConnection()) {   // ✅ try-with-resources
        // ...
    } catch (SQLException e) {
        // ...
    }
    // conn.close() otomatik
}
```

**Modern Java/Spring uygulamasında** doğrudan `dataSource.getConnection()` çağırma — Spring'in `JdbcTemplate`, `TransactionManager`, `JpaRepository` zaten lifecycle'ı yönetir. Manuel `getConnection` = code smell.

**Tuzak:** `JpaRepository` veya `EntityManager` kullanırken leak yapmak zor — Spring yönetir. Ama bazen developer "direkt SQL gerek" diye `JdbcTemplate` veya `dataSource.getConnection` çağırır → bunu kontrol et.

---

### 10. PgBouncer — K8s multi-replica deployment için neden gerekli

#### Problem

K8s'te app 10 replica. Her replica HikariCP 20 connection. **Toplam: 200 connection.**

PostgreSQL `max_connections = 100`. **Patlama.**

Pool size'ı düşürmek? 8 connection per replica → çok az, application istemcileri bekler.

#### Çözüm: PgBouncer

PgBouncer DB ile uygulama arasında bir **lightweight proxy**. Uygulama PgBouncer'a bağlanır, PgBouncer DB'ye az sayıda gerçek connection açar.

```
Uygulama 1 (HikariCP 20)
Uygulama 2 (HikariCP 20)      ┌──────────────┐      ┌─────────────┐
Uygulama 3 (HikariCP 20)  →   │  PgBouncer    │  →  │  PostgreSQL │
...                            │  (50 backend) │      │  max_conn=60 │
Uygulama 10 (HikariCP 20)      └──────────────┘      └─────────────┘
toplam 200                      50 backend            60 backend
```

PgBouncer 200 "client connection"u 50 "server connection"a multiplex eder.

#### Pooling mode'ları

**Session pooling:**
- Client connect olduğu sürece aynı backend connection
- En transparent (PostgreSQL session feature'ları çalışır)
- Multiplexing sınırlı

**Transaction pooling:** (en yaygın banking için)
- Transaction süresince backend connection bağlı
- Transaction bittiğinde backend serbest, başka client kullanır
- Çok daha iyi multiplexing
- **Limit:** Session-level state (prepared statements, temp tables, advisory locks) çalışmaz veya dikkat gerekir

**Statement pooling:**
- Her statement için backend swap
- En agresif multiplexing
- Transaction izolasyonu çalışmaz — banking için **YASAK**

**Banking pratiği:**
- Transaction pooling (mode = transaction)
- Prepared statement'ları client-side cache kapalı (`prepareThreshold=0` PostgreSQL JDBC URL'de)
- Session-level config (search_path, vb) connection init'te yap

#### `pgbouncer.ini` örneği

```ini
[databases]
core_banking = host=postgres-primary port=5432 dbname=core_banking pool_size=50

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = scram-sha-256
pool_mode = transaction
max_client_conn = 500
default_pool_size = 50
reserve_pool_size = 5
server_lifetime = 3600
server_idle_timeout = 600
```

Uygulama JDBC URL:

```
jdbc:postgresql://pgbouncer:6432/core_banking?prepareThreshold=0
```

#### K8s deployment

PgBouncer pod'u sidecar veya StatefulSet olarak. HikariCP `max-lifetime`'ı PgBouncer `server_lifetime`'tan düşük tut.

#### Banking'de PgBouncer ne zaman gerek değil?

- Tek instance, küçük load → direkt PostgreSQL
- AWS RDS Proxy zaten benzer iş yapıyor (managed)
- Connection count yönetilebilir (<100 toplam)

---

### 11. AWS RDS Proxy, GCP SQL Proxy alternatifleri

**AWS RDS Proxy:** Cloud-native. PgBouncer + failover + IAM auth. AWS recommended for serverless / Lambda.

**GCP Cloud SQL Auth Proxy:** Authentication + connection routing. Limited pooling.

**Banking pratiği:** Self-hosted PgBouncer daha esnek (transaction mode), cloud proxy'ler daha managed (kolay) ama mode kısıtlı.

---

### 12. Anti-pattern'ler

**Anti-pattern 1: Maximum pool size çok büyük**

```yaml
hikari:
  maximum-pool-size: 200
```

DB'yi bunaltır. Brian Goetz formülüne sadık kal.

**Anti-pattern 2: `connectionTimeout: 0` veya çok yüksek**

```yaml
connection-timeout: 0     # ❌ sonsuza kadar bekle
connection-timeout: 60000 # ❌ banking API için çok uzun
```

API için 2-5 saniye.

**Anti-pattern 3: `maxLifetime` DB timeout'tan büyük**

```yaml
max-lifetime: 7200000  # 2 saat, ama DB session timeout 1 saat
```

"Connection has been closed" hatalarına yol açar.

**Anti-pattern 4: `dataSource.getConnection()` doğrudan çağırmak**

Spring abstraction'ı bypass. Leak riski.

**Anti-pattern 5: Pool monitoring yok**

Production'da metric'ler izlenmiyorsa pool exhaustion'ı kullanıcı bildirir (geç).

**Anti-pattern 6: HikariCP'yi başka pool ile değiştirmek (sebep yok)**

DBCP2 veya C3P0 = kayıp performans. Sebep yok değiştirme.

**Anti-pattern 7: Replica başına aynı `maxPoolSize` = DB connection overflow**

K8s'te scaling yaparken pool sizing'i unutmak. PgBouncer veya azalt.

**Anti-pattern 8: TX içinde external HTTP call**

```java
@Transactional
public void method() {
    repo.save(entity);            // connection ödünç alındı
    externalApi.call();           // 5 saniye sürdü
    repo.save(other);
}
```

Connection 5 saniye boyunca bağlı, başka request bekler. **External call TX dışına.**

---

### 13. Pool sizing experiment — Gatling ile

Pool sizing teori değil, **deneyseldir.** Gatling load test ile:

1. Pool = 5 → 50 req/sec → P95 ölç
2. Pool = 10 → 50 req/sec → P95 ölç
3. Pool = 20 → 50 req/sec → P95 ölç
4. Pool = 40 → 50 req/sec → P95 ölç

**Eğri:** Pool artarken P95 düşer, bir noktada **plato** olur (DB darboğaz), sonra artmaya başlar (context switching).

**Sweet spot:** P95'in plato başlangıcında. Banking'de bunu environment'a göre kalibre et — pre-prod'da load test, prod'da metric'lerle doğrula.

---

## Önemli olabilecek araştırma kaynakları

- HikariCP wiki — https://github.com/brettwooldridge/HikariCP/wiki
- "About Pool Sizing" HikariCP wiki sayfası (Brian Goetz formülü)
- Vlad Mihalcea "Connection Pool Sizing"
- PgBouncer official documentation
- PostgreSQL `max_connections` and `shared_buffers` tuning
- AWS RDS Proxy documentation
- Spring Boot Actuator + Micrometer guide
- Grafana HikariCP dashboard (community 6083, 19062)
- "Why You Don't Need 100s of Connections" Heroku Postgres blog

---

## Mini task'ler

### Task 2.6.1 — Default HikariCP config'i tanı (15 dk)

`application.yml`'a:

```yaml
logging:
  level:
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.HikariConfig: DEBUG
```

Uygulamayı başlat. Log'larda Hikari init config görünmeli. Tüm parametre default'larını **defterine** yapıştır.

### Task 2.6.2 — Banking-grade config (30 dk)

`application.yml`'ı şu değerlerle güncelle:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
      connection-timeout: 3000
      idle-timeout: 0
      max-lifetime: 270000
      keepalive-time: 30000
      leak-detection-threshold: 30000
      validation-timeout: 3000
      pool-name: CoreBankingHikariCP
      register-mbeans: true
```

PostgreSQL connection URL'ye:

```
jdbc:postgresql://localhost:5432/core_banking?prepareThreshold=0&socketTimeout=30
```

Açıklama defterine: Her parametre niye bu değer?

### Task 2.6.3 — Pool exhaustion reprodüksiyon (45 dk)

Pool'u 2'ye düşür:

```yaml
maximum-pool-size: 2
connection-timeout: 5000
```

`SlowEndpoint`:

```java
@GetMapping("/slow")
@Transactional
public void slow() throws InterruptedException {
    Thread.sleep(10000);   // 10 saniye boyunca connection tut
}
```

Curl ile 3 paralel istek:

```bash
curl localhost:8080/slow &
curl localhost:8080/slow &
curl localhost:8080/slow &
```

3. istek 5 saniye sonra timeout almalı: `Connection is not available, request timed out after 5000ms`.

`jstack` ile thread dump al — log'a yapıştır.

### Task 2.6.4 — Connection leak reprodüksiyon (30 dk)

Kasten leak'li kod:

```java
@RestController
class LeakController {
    @Autowired DataSource dataSource;
    
    @GetMapping("/leak")
    public void leak() throws SQLException {
        Connection conn = dataSource.getConnection();
        // conn.close() çağrılmadı
    }
}
```

`leakDetectionThreshold: 5000`. Endpoint'i 1 kere çağır. 5 saniye sonra log'da `ProxyLeakTask` warning ve stack trace görmeli.

Düzelt: try-with-resources.

### Task 2.6.5 — Micrometer + Prometheus + Grafana setup (1 saat)

`pom.xml`:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

`application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
```

Curl `localhost:8080/actuator/prometheus | grep hikari` — metric'leri gör.

`docker-compose.yml`'a Prometheus ve Grafana servisi ekle:

```yaml
prometheus:
  image: prom/prometheus
  ports: ["9090:9090"]
  volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]

grafana:
  image: grafana/grafana
  ports: ["3000:3000"]
```

`prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'core-banking'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

Grafana'ya HikariCP dashboard'u import et (ID 6083). Pool exhaustion testi sırasında grafiklerde aktivite görmeli.

### Task 2.6.6 — Gatling pool sizing experiment (1.5 saat)

`gatling-charts-highcharts-bundle` indir.

Simulation:

```scala
class TransferSimulation extends Simulation {
    val httpProtocol = http.baseUrl("http://localhost:8080")
    
    val scn = scenario("Transfer load")
        .exec(http("transfer")
            .post("/v1/transfers")
            .body(StringBody("""{ ... }""")).asJson
            .check(status.is(201))
        )
    
    setUp(
        scn.inject(constantUsersPerSec(50) during (1 minute))
    ).protocols(httpProtocol)
}
```

Pool size = 2 ile çalıştır → Gatling raporu (P95, P99, error rate). 
Pool = 5, 10, 20, 40 ile tekrar et.

Sonuçları `docs/pool-sizing-experiment.md`'a tablo olarak yaz:

| Pool size | P50 | P95 | P99 | Error rate | DB CPU |
|---|---|---|---|---|---|
| 2 | ? | ? | ? | ? | ? |
| 5 | ? | ? | ? | ? | ? |
| ... |  |  |  |  |  |

Sweet spot'u **defterine** yaz.

### Task 2.6.7 — PgBouncer kurulumu (1 saat)

`docker-compose.yml`'a:

```yaml
pgbouncer:
  image: edoburu/pgbouncer:1.21.0
  environment:
    DB_HOST: postgres
    DB_PORT: 5432
    DB_USER: banking
    DB_PASSWORD: banking_password
    POOL_MODE: transaction
    MAX_CLIENT_CONN: 500
    DEFAULT_POOL_SIZE: 25
  ports:
    - "6432:6432"
```

Uygulamanın JDBC URL'sini değiştir:

```
jdbc:postgresql://localhost:6432/core_banking?prepareThreshold=0
```

Aynı Gatling test'ini tekrar koş. PgBouncer transaction pooling ile **az gerçek connection** harcanmalı (PostgreSQL'de `pg_stat_activity` kontrol).

```sql
SELECT count(*), application_name FROM pg_stat_activity GROUP BY application_name;
```

PgBouncer aktif → 25 backend connection. PgBouncer'sız → 50+ direkt connection.

### Task 2.6.8 — `prepareThreshold=0` deneyi (30 dk)

`prepareThreshold=0`'sız (default 5) ile `prepareThreshold=0` arasındaki farkı **transaction pooling** mode'da gözle.

`prepareThreshold > 0` + PgBouncer transaction mode = "prepared statement does not exist" hataları (çünkü backend swap olur, prepared statement diğer backend'de yok).

Hatayı reprodüksiyon et, sonra `prepareThreshold=0` ile çöz. Defter notuna yaz.

---

## Test yazma rehberi

### Test 2.6.1 — Pool config verification

```java
@SpringBootTest
@Testcontainers
class HikariConfigTest {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired DataSource dataSource;
    
    @Test
    void hikariConfigShouldMatchBankingStandards() {
        assertThat(dataSource).isInstanceOf(HikariDataSource.class);
        HikariDataSource hikari = (HikariDataSource) dataSource;
        
        assertThat(hikari.getMaximumPoolSize()).isBetween(5, 30);
        assertThat(hikari.getConnectionTimeout()).isLessThanOrEqualTo(5000);
        assertThat(hikari.getMaxLifetime()).isLessThanOrEqualTo(1800000);
        assertThat(hikari.getLeakDetectionThreshold()).isGreaterThan(0);
    }
}
```

### Test 2.6.2 — Pool exhaustion timeout

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.datasource.hikari.maximum-pool-size=2",
    "spring.datasource.hikari.connection-timeout=1000"
})
class PoolExhaustionTest {
    
    @Autowired DataSource ds;
    
    @Test
    void shouldThrowTimeoutWhenPoolExhausted() throws Exception {
        Connection c1 = ds.getConnection();
        Connection c2 = ds.getConnection();
        
        long start = System.currentTimeMillis();
        assertThatThrownBy(() -> ds.getConnection())
            .isInstanceOf(SQLTransientConnectionException.class)
            .hasMessageContaining("Connection is not available");
        long elapsed = System.currentTimeMillis() - start;
        
        assertThat(elapsed).isBetween(900L, 1500L);
        
        c1.close();
        c2.close();
    }
}
```

### Test 2.6.3 — Leak detection

```java
@Test
@Slf4j
void leakDetectionShouldTriggerWarning() throws Exception {
    Logger hikariLogger = (Logger) LoggerFactory.getLogger("com.zaxxer.hikari.pool.ProxyLeakTask");
    ListAppender<ILoggingEvent> appender = new ListAppender<>();
    appender.start();
    hikariLogger.addAppender(appender);
    
    Connection conn = dataSource.getConnection();
    // Kasten kapatmıyoruz
    
    Thread.sleep(5500);   // leakDetectionThreshold + buffer
    
    boolean leakLogged = appender.list.stream()
        .anyMatch(e -> e.getFormattedMessage().contains("Connection leak detection"));
    assertThat(leakLogged).isTrue();
    
    conn.close();   // cleanup
}
```

---

## Claude-verify prompt

```
Aşağıdaki HikariCP + connection pool kodumu banking-grade kriterlere göre değerlendir. 
PASS / FAIL / EKSIK işaretle, KOD YAZMA:

1. Pool sizing:
   - `maximum-pool-size` Brian Goetz formülüne göre kalibre edilmiş mi (gerekçeli)?
   - K8s replica sayısı * pool size DB max_connections'ı aşıyor mu?
   - Banking workload tipi (API/Batch/Reporting) için doğru pool size mı?

2. Kritik parametreler:
   - `connection-timeout` 2-5 saniye arası mı (API için)?
   - `max-lifetime` DB-side timeout - 30s formülüne uygun mu?
   - `keepalive-time` set edilmiş mi (firewall idle koruması)?
   - `leak-detection-threshold` aktif mi (en azından dev'de)?
   - `minimum-idle = maximum-pool-size` (fixed-size) mı, dynamic mi (gerekçeli)?

3. Configuration:
   - `pool-name` set mi (log/metric'te ayırt etmek için)?
   - `register-mbeans` JMX için aktif mi (dev'de)?
   - JDBC URL `prepareThreshold=0` mı (PgBouncer transaction mode varsa)?
   - `socketTimeout` JDBC URL'de var mı?

4. Monitoring:
   - Micrometer Prometheus registry aktif mi?
   - `/actuator/prometheus` endpoint'inde hikaricp_* metric'leri görünüyor mu?
   - Grafana dashboard kurulmuş mu?
   - Alert kuralları (pending > 0, timeout > 0/min) tanımlı mı?

5. Pool exhaustion:
   - Reprodüksiyon test'i yazılmış mı?
   - `jstack` analizi `docs/` altında mı?
   - Long TX nedeni tespit edilmiş mi (N+1, external call, lock contention)?

6. Connection leak:
   - `leak-detection-threshold` log alarmı test ile doğrulanmış mı?
   - Manuel `dataSource.getConnection()` kullanımı var mı (olmamalı)?
   - try-with-resources alışkanlığı yerleşmiş mi?

7. PgBouncer / proxy:
   - Multi-replica deployment varsa PgBouncer veya RDS Proxy kullanımı düşünülmüş mü?
   - Transaction pooling mode mu (banking için)?
   - HikariCP `max-lifetime` < PgBouncer `server_lifetime` mi?
   - `prepareThreshold=0` ile prepared statement uyumu var mı?

8. Anti-pattern:
   - Pool size > 100 mü (çok büyük)?
   - `connection-timeout = 0` veya çok uzun mu?
   - `max-lifetime` > DB session timeout mu?
   - TX içinde external HTTP call var mı?

9. Banking-grade kalite:
   - Gatling/k6 ile load test yapılmış mı?
   - Pool sizing experiment dokümante mi (`docs/pool-sizing-experiment.md`)?
   - Pre-prod ve prod environment'larda farklı pool config var mı (sizing'e göre)?
   - Pool config ADR'da yazılı mı (karar gerekçesi)?

10. Sağlık kontrolü:
    - `/actuator/health` HikariCP health indicator'ı kontrol ediyor mu?
    - DB unavailable senaryosunda 503 dönüyor mu (K8s liveness probe için)?

Her madde için PASS / FAIL / EKSIK ve kısa gerekçe. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] HikariCP banking-grade config (`max-pool-size`, `connection-timeout`, `max-lifetime`, vs.) yazıldı, her değer gerekçeli
- [ ] Pool exhaustion reprodüksiyon ettim, `jstack` analizi `docs/`'a kaydedildi
- [ ] Connection leak reprodüksiyon ettim, leak detection ile yakalandı
- [ ] Micrometer + Prometheus + Grafana stack kurdum, dashboard'da pool metric'lerini gördüm
- [ ] Gatling ile pool sizing experiment yaptım, sweet spot belirledim
- [ ] PgBouncer'ı docker-compose'a ekledim, transaction pooling mode'unda test ettim
- [ ] `prepareThreshold=0` ile prepared statement sorunlarını anladım
- [ ] Brian Goetz formülünü banking'e uyarladığım ADR yazdım
- [ ] Banking workload tipi (API/Batch/Reporting) için pool farklılaşması düşünülmüş
- [ ] Anti-pattern listesi rahat — özellikle "TX içinde external call", "pool size > 100"

Hepsi onaylı → Topic 2.7'ye geç → [07-hibernate-performance/](../07-hibernate-performance/index.md)

---

## Defter notları

1. "Connection pool'un temel amacı: ____. Banking'de neden kritik: ____."
2. "HikariCP'nin hızının 3 sebebi: ConcurrentBag, ____, ____."
3. "Brian Goetz formülü: ____. Banking'de adaptasyon nasıl yapıyorum: ____."
4. "`maxLifetime` neden DB-side timeout'tan küçük tutulur: ____."
5. "`connectionTimeout` banking API için tavsiye edilen aralık: ____."
6. "`leakDetectionThreshold` ne yapar: ____. Dev/prod ayrımı: ____."
7. "Pool exhaustion 3 belirti: ____, ____, ____."
8. "`jstack` analizinde aranması gereken pattern: ____."
9. "PgBouncer transaction pooling mode'unun avantajı: ____. Banking için neden statement pooling YASAK: ____."
10. "Pool sizing experiment'in 5 ölçüm boyutu: P50, P95, P99, error rate, ____."
