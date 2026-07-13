# Topic 2.7 — Hibernate Performance Tuning

## Hedef

Hibernate'in performans karakteristiğini production-grade banking workload'larında **bilinçli optimize edebilmek**. Batch insert/update'lerin neden default olarak çalışmadığını (IDENTITY ID generation problemi) ve nasıl düzeltileceğini (SEQUENCE allocationSize) öğrenmek. Hibernate query plan cache'in (`hibernate.query.plan_cache_max_size`) memory leak senaryosunu görmek. Statement caching, second-level cache (Ehcache/Hazelcast/Caffeine), query cache pattern'lerini banking domain'inde ne zaman kullanmak/kullanmamak gerektiğini karar verecek matrisi oluşturmak. Hibernate Statistics'i production'da kullanmak. PostgreSQL `log_min_duration_statement` ile slow query tespiti, `pg_stat_statements` ile aggregate analiz. Bu topic Phase 2'nin "performance optimizasyon ustalığı" topic'i — burada öğrenilenler her banking projesinde günlük kullanılır.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 2.1-2.6 bitti — JPA mekaniği, transactions, locking, N+1, pool sizing biliyorsun
- Spring Boot uygulamanda PostgreSQL ve TestContainers ile çalışıyor
- `application.yml`'da Hibernate property'lerini set etmeyi gördün (`spring.jpa.properties.hibernate.*`)
- Performans deyince "P50, P95, P99 latency" ve "throughput RPS" terimlerine aşinasın

---

## Kavramlar

### 1. Batch insert — Hibernate'in en pahalı operasyonu

Banking'de bir batch job 100.000 transaction kaydı işliyor. Naif yaklaşım:

```java
@Transactional
public void importTransactions(List<TransactionDto> transactions) {
    for (TransactionDto dto : transactions) {
        TransactionJpaEntity entity = mapper.toEntity(dto);
        repo.save(entity);
    }
}
```

Bu kodda her `save` ayrı bir `INSERT` SQL üretir. 100.000 INSERT * 2ms = **200 saniye**. Aşırı yavaş.

**Hibernate batching aktif olsa:** 100.000 / 50 = 2000 batch query, her biri tek round-trip. **Teorik:** 20 saniye. **Pratik:** Çoğu zaman çalışmaz, neden?

#### Batching aktif değil — sebep çoğu zaman ID generation

```java
@Entity
public class TransactionJpaEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // ❌ batch'i kırar
    private Long id;
    // ...
}
```

`IDENTITY` strategy DB'nin auto-increment'ini kullanır. Hibernate **insert sonrası ID'yi okumak zorunda** — yani her insert'i tek tek yapar. Batch imkânsız.

#### `SEQUENCE` ile batch fix

```java
@Entity
public class TransactionJpaEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "txn_seq")
    @SequenceGenerator(
        name = "txn_seq",
        sequenceName = "transaction_sequence",
        allocationSize = 50   // ← kritik
    )
    private Long id;
}
```

Migration:

```sql
CREATE SEQUENCE transaction_sequence
    START WITH 1 
    INCREMENT BY 50 
    NO CYCLE;
```

**Mekanik:** Hibernate bir defada DB'den 50 ID alır (`nextval`), Java tarafında bir bir kullanır. Tüm INSERT'leri batch'leyebilir çünkü ID önceden biliniyor.

**`allocationSize` neden 50:**

- 1 → her insert için DB round-trip → batch çalışmaz
- 50 → 50 insert için 1 sequence call, batch için yeterli
- 1000 → çok büyük "ID kaybı" (uygulama restart'larında)

**Banking pratiği:** `allocationSize = 50` veya `100`. Allokasyon JDBC URL'de seçildiği gibi. Sequence'in DB-tarafı `INCREMENT BY` aynı değer olmalı — yoksa ID çakışması.

#### Batch config — `application.yml`

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
```

**Açıklama:**

- `jdbc.batch_size = 50` — JDBC PreparedStatement'a 50 INSERT eklenir, sonra bir kerede execute.
- `order_inserts = true` — Hibernate INSERT'leri **table'a göre sıralar**. Aynı table'a 50 art arda INSERT → tek batch. Farklı tabloya gidip gelmek batch'i bozar.
- `order_updates = true` — Aynı UPDATE'ler için.
- `batch_versioned_data = true` — `@Version`'lı entity'ler için batch update aktif (default false).

#### UUID ID ile batch

UUID kullanıyorsan IDENTITY problemi yok. Hibernate `@GeneratedValue(strategy = GenerationType.UUID)` (JPA 3.0+, Hibernate 6.2+) ile UUID atar Java tarafında, batch sorunsuz.

```java
@Entity
public class JournalLineJpaEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
}
```

**Banking pratiği:** Yeni entity'lerde UUID tercih (distributed sistem, debug edebilen ID). Eski sistemlerde Long + SEQUENCE.

#### Verifikasyon — batch gerçekten çalışıyor mu?

```yaml
logging:
  level:
    org.hibernate.engine.jdbc.batch.internal.BatchingBatch: DEBUG
```

Log'da:

```
DEBUG ... BatchingBatch - Reusing batch statement
INFO ... StatementInspector - SELECT nextval('transaction_sequence')
DEBUG ... BatchingBatch - Adding to batch (batch size: 50)
DEBUG ... BatchingBatch - Executing batch (size: 50)
```

`Adding to batch ... 50` ve `Executing batch (size: 50)` → batching aktif.

Eğer her satır için `INSERT INTO ...` görüyorsan **batch çalışmıyor**.

#### `saveAll` ile `save` farkı

```java
repo.saveAll(entities);   // ✅ batch'e uygun
entities.forEach(repo::save);   // ⚠️ flush davranışına göre
```

`saveAll` Spring Data'da `EntityManager.persist` döngüsü, **flush sona kadar ertelenir**. Tek `save` döngüsünde de aynıdır ama `@Transactional` sınırında flush. Pratikte ikisi de benzer ama `saveAll` niyetin net.

**Tuzak:** Yarıda `flush()` çağırma — `saveAll(batch1); em.flush(); saveAll(batch2);` → batch1'in batch'i bozulabilir.

---

### 2. Bulk update / delete — `@Modifying` ile JPQL

Persistence context'in dirty checking'i 100.000 entity'i UPDATE etmek için aşırı yavaş.

```java
@Modifying
@Query("UPDATE TransactionJpaEntity t SET t.status = :newStatus WHERE t.batchId = :batchId")
int markAsProcessed(@Param("batchId") UUID batchId, @Param("newStatus") String newStatus);
```

DB'ye tek `UPDATE` SQL. 100.000 satırı saniye altında günceller.

**Tuzak:** PC bypass eder. Aynı TX içinde sonra entity'i `findById` ile alırsan **eski state** görürsün (PC'deki cached). `@Modifying(clearAutomatically = true)` ile PC temizlenir, ya da manuel `em.clear()`.

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE ...")
int bulkUpdate(...);
```

**Banking pratiği:** Batch fee deduction, statement generation gibi mass operasyonlar için `@Modifying`. PC tutarlılığını dikkat et.

---

### 3. `StatelessSession` — Hibernate'in unutulan armağanı

Çok büyük ETL/import job'larında `StatelessSession` PC'i hiç kullanmaz, dirty checking yok, cascade yok.

```java
@Service
public class BulkImportService {
    
    @Autowired EntityManagerFactory emf;
    
    public void importHugeBatch(List<TransactionDto> dtos) {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        try (StatelessSession session = sf.openStatelessSession()) {
            Transaction tx = session.beginTransaction();
            try {
                for (int i = 0; i < dtos.size(); i++) {
                    TransactionJpaEntity entity = mapper.toEntity(dtos.get(i));
                    session.insert(entity);
                    if (i % 1000 == 0) {
                        // No flush needed; insert is immediate
                    }
                }
                tx.commit();
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }
}
```

**`StatelessSession` özellikleri:**

- PC yok → memory'i şişirmez
- Dirty checking yok → CPU tasarrufu
- Cascade çalışmaz → her entity manuel
- L1 cache yok → aynı entity tekrar load edilirse 2. SELECT
- L2 cache yok

**Banking pratiği:** Statement generation, monthly aggregation, partition migration gibi büyük ETL. Normal request handling'te kullanma.

---

### 4. Hibernate query plan cache

Hibernate her JPQL/HQL/Criteria query'sini bir kez parse edip "plan"a çevirir. Bu plan cache'lenir. Sonraki çağrılarda direkt plan kullanılır.

**Default:** `hibernate.query.plan_cache_max_size = 2048` (entries).

**Tuzak: Plan cache memory leak**

Eğer query'leri **dinamik string concat** ile üretiyorsan:

```java
String hql = "SELECT a FROM AccountJpaEntity a WHERE a.id = " + accountId;   // ❌
```

Her farklı `accountId` için yeni plan. 1 milyon farklı ID → 1 milyon plan → OOM.

**Çözüm:** Her zaman parameterized query:

```java
@Query("SELECT a FROM AccountJpaEntity a WHERE a.id = :id")
Optional<AccountJpaEntity> findById(@Param("id") UUID id);
```

Aynı plan, farklı parameter. Cache 1 entry kalır.

**Cache tuning:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        query:
          plan_cache_max_size: 4096
          plan_parameter_metadata_max_size: 256
```

**Banking pratiği:** Banking'de yüzlerce repository method var, her biri 1 plan. 4096 default'tan büyük ama yeterli. Dinamik Specification kullanıyorsan farklı predicate kombinasyonu = farklı plan; bu durumda plan cache büyütülmeli (8192+).

**İzleme:**

```yaml
spring.jpa.properties.hibernate.generate_statistics: true
```

```java
Statistics stats = sf.getStatistics();
System.out.println("Query plan cache hit count: " + stats.getQueryPlanCacheHitCount());
System.out.println("Query plan cache miss count: " + stats.getQueryPlanCacheMissCount());
```

Hit ratio > %95 → sağlıklı. < %50 → dinamik query problemi var.

---

### 5. Statement caching — JDBC PreparedStatement

Bu Hibernate'in değil **JDBC driver'ının** feature'ı. PgJDBC PreparedStatement'ları cache eder; benzer SQL geldiğinde driver compilation atlatır.

**PostgreSQL JDBC URL:**

```
jdbc:postgresql://localhost:5432/db?prepareThreshold=5
```

`prepareThreshold = 5` → aynı SQL 5 kez kullanıldıktan sonra DB-side prepared statement'a dönüşür (`PREPARE`/`EXECUTE`).

**Avantajlar:**

- Plan re-use DB tarafında
- Hibernate'in `?` parameter binding'i hızlı

**Tuzak:** PgBouncer transaction pooling mode'da `prepareThreshold > 0` patlamalara yol açar (Topic 2.6). PgBouncer arkasında **`prepareThreshold = 0`**.

**Banking pratiği:**
- Direkt PostgreSQL: `prepareThreshold = 5`
- PgBouncer transaction mode: `prepareThreshold = 0`
- AWS RDS Proxy: kontrol et, genelde `0` öneriliyor

---

### 6. Second-level cache (2L) — banking'de neye yarar, neye yaramaz

**L1 cache (PC, default):** Session/TX bazlı. TX kapanınca biter.

**L2 cache:** SessionFactory bazlı, **birden fazla TX/thread arası paylaşılır**. Aynı entity'i tekrar load etmek DB hit gerektirmez.

#### Konfigürasyon

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

`application.yml`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

Entity:

```java
@Entity
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class CurrencyJpaEntity {
    @Id String code;     // "TRY", "USD"
    String name;
    int fractionDigits;
}
```

Şimdi `currencyRepo.findById("TRY")` ilk çağrıda DB'ye gider, sonrakiler cache'ten okur.

#### Cache concurrency strategy'leri

- `READ_ONLY` — değişmeyen veri (currency, country codes). En hızlı.
- `READ_WRITE` — değişebilir, transaction-aware. Bizim case.
- `NONSTRICT_READ_WRITE` — stale'e tolerans var. Banking için **YASAK** (stale balance = felaket).
- `TRANSACTIONAL` — JTA transaction. Banking için overkill.

#### Cache provider seçimi

| Provider | Özellik | Banking kullanımı |
|---|---|---|
| **Ehcache** | Local JVM heap + disk. Off-heap opsiyonu. | Tek-instance servisler için ideal |
| **Caffeine** | High-performance local. Sadece in-memory. | Lookup data için (currency, country) |
| **Hazelcast** | Distributed, multi-instance senkronize. | K8s multi-replica için |
| **Redis** | Distributed, network. Hibernate'in resmi entegrasyonu sınırlı. | Genelde Spring Cache'le manuel |
| **Infinispan** | Red Hat. Banking enterprise sık seçim. | JBoss/WildFly ortamlarında |

**Banking pratiği:**

- **Tek instance:** Ehcache veya Caffeine (basit, hızlı).
- **K8s 5-10 replica:** Hazelcast (cache invalidation tüm node'lara yayılır).
- **K8s 50+ replica veya cache büyük:** Redis (memory ayrı, dedicated).

#### Banking'de ne cache, ne cache'leme

**Cache:**

- `CurrencyJpaEntity` (TRY, USD, EUR... — 200 satır, hiç değişmez) ✅
- `CountryJpaEntity` (ISO 3166 listesi) ✅
- `BranchJpaEntity` (banka şubeleri — yılda 1-2 değişir) ✅
- `FeeScheduleJpaEntity` (ücret yapısı — günde 1 değişir, expire kısa) ✅
- `ConfigurationJpaEntity` (sistem parametreleri) ✅

**Cache'leme:**

- `AccountJpaEntity` — bakiye dinamik, stale = problem ❌
- `JournalLineJpaEntity` — sık eklenir, append-only, cache faydası az ❌
- `CustomerJpaEntity` — PII, regülatör cache'i istemez ❌
- `CardJpaEntity` — security state'i hızlı değişir ❌

**Banking kuralı:**

> Money değişen şeyleri cache'leme. Lookup veya konfigürasyon türü statik veriyi cache'le.

#### Stale cache'in banking'de potansiyel zararı

- Account.balance cache'lenir → user 1000 TL var sanıyor → withdraw → DB'de 500 → "yetersiz" → ama cache 1000 dedi → hata mesajı tutarsız
- FeeSchedule cache'lenmiş eski tarife → fee yanlış hesaplandı → mali rapor hatası

**Çare:** Cache TTL kısa + invalidation event'leri. Banking'de cache **dikkatli kullanılan bir araç**.

---

### 7. Query cache — bambaşka bir hayvan

Query cache, **belirli bir query'nin sonuçlarını** (entity ID'leri) cache'ler. Sonra ID'lerle entity'i 2L cache'ten okur.

```java
@Query("SELECT a FROM AccountJpaEntity a WHERE a.ownerId = :ownerId")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<AccountJpaEntity> findByOwnerId(@Param("ownerId") UUID ownerId);
```

**Avantaj:** Aynı query parametreli tekrar çağrıldığında sonuç cache'ten.

**Tuzaklar:**

1. **Invalidation aşırı agresif:** Tablo herhangi bir kayıt eklenince/silinince tüm query cache entries silinir. Banking'in append-heavy yapısında query cache **sürekli invalidate**, performans kazancı yok.
2. **Memory leak:** Her farklı parameter combinasyonu yeni cache entry. Plan cache benzeri leak riski.
3. **Sadece exact match:** Sıra önemli, pagination farklı → farklı entry.

**Banking pratiği:** Query cache nadiren değerli. Sadece **read-only lookup query**'lerinde (örnek: tüm aktif currency'leri listele). Append-heavy tablolar (journal_lines, accounts) için kapalı.

---

### 8. Hibernate statistics — production izleme

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

Statistics nesnesi:

```java
@Component
public class HibernateMetricsRecorder {
    
    private final SessionFactory sessionFactory;
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedDelay = 30000)
    public void recordMetrics() {
        Statistics stats = sessionFactory.getStatistics();
        
        meterRegistry.gauge("hibernate.query.count", stats.getQueryExecutionCount());
        meterRegistry.gauge("hibernate.query.slowestTime", stats.getQueryExecutionMaxTime());
        meterRegistry.gauge("hibernate.entity.load.count", stats.getEntityLoadCount());
        meterRegistry.gauge("hibernate.entity.insert.count", stats.getEntityInsertCount());
        meterRegistry.gauge("hibernate.entity.update.count", stats.getEntityUpdateCount());
        meterRegistry.gauge("hibernate.second_level_cache.hit.ratio", 
            (double) stats.getSecondLevelCacheHitCount() / 
            (stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount() + 1));
        meterRegistry.gauge("hibernate.flush.count", stats.getFlushCount());
        meterRegistry.gauge("hibernate.connection.count", stats.getConnectCount());
        meterRegistry.gauge("hibernate.transaction.count", stats.getTransactionCount());
        meterRegistry.gauge("hibernate.session.open.count", stats.getSessionOpenCount());
        meterRegistry.gauge("hibernate.prepare.statement.count", stats.getPrepareStatementCount());
        
        // Slow query
        String slowestQuery = stats.getQueryExecutionMaxTimeQueryString();
        if (slowestQuery != null && stats.getQueryExecutionMaxTime() > 100) {
            log.warn("Slow Hibernate query ({}ms): {}", 
                stats.getQueryExecutionMaxTime(), slowestQuery);
        }
    }
}
```

**Banking-relevant metrics:**

- `query.count` — total query sayısı, throughput proxy
- `query.slowestTime` — en yavaş query'nin ms'i
- `entity.load.count / query.count` — bir query başına entity yükü (N+1 göstergesi)
- `second_level_cache.hit.ratio` — L2 cache etkisi
- `flush.count / transaction.count` — TX başına flush (1 ideal)

**Performans alarmı:** `slowestTime > 500ms` → log.warn, devops'a bildir.

---

### 9. PostgreSQL slow query log

DB tarafında **kanıt**. `postgresql.conf`:

```
log_min_duration_statement = 100   # 100ms üzeri her query log
log_statement_stats = on            # statistical info (CPU, IO)
log_lock_waits = on                 # lock bekleme uyarısı
deadlock_timeout = 1s
```

Log'da:

```
2025-05-12 14:30:00 UTC [12345] LOG: duration: 2543.123 ms 
statement: SELECT ... FROM accounts a LEFT JOIN journal_lines jl ... WHERE a.owner_id = $1
```

**Banking pratiği:** Production'da `log_min_duration_statement = 250` (250ms üzeri). Daha düşük olunca log explosion.

---

### 10. `pg_stat_statements` — aggregate query analiz

PostgreSQL extension. Tüm query'lerin **özetini** tutar — execution count, total time, mean time.

```sql
CREATE EXTENSION pg_stat_statements;

-- En yavaş 10 query (total time'a göre)
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Banking-grade analiz:

```sql
-- Mean time > 50ms olanlar (slow individually)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 50
ORDER BY total_exec_time DESC;

-- Çok sık çağrılan query'ler (top calls)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

**Banking pratiği:** Weekly query budget review. Yeni N+1 veya slow query CI'de yakalanmadıysa burada yakalanır.

---

### 11. JPQL/Native SQL ne zaman hangisi — performans bakışı

| Senaryo | JPQL | Native SQL |
|---|---|---|
| Standard CRUD | ✅ Tercih | ❌ Overkill |
| Complex JOIN (3+ tablo) | ✅ Çoğu zaman | ⚠️ Performans gerekiyorsa |
| DB-specific feature (window functions, CTE) | ❌ Desteklemez | ✅ Zorunlu |
| Aggregation (SUM, COUNT, AVG) | ✅ Destekler | ⚠️ Karmaşıksa |
| Bulk update | ✅ `@Modifying` JPQL | ⚠️ Performans için bazen |
| Multi-row INSERT (INSERT ... SELECT) | ❌ Desteklemez | ✅ Native |
| MERGE / UPSERT | ❌ JPQL'da yok | ✅ Native + ON CONFLICT |
| PostgreSQL JSONB queries | ❌ JPQL'da yok | ✅ Native + JSONB operators |
| Recursive CTE | ❌ | ✅ |

**Banking pratiği:** JPQL %85, native %15. Native gerektiğinde adapter katmanında izole — domain'e sızmasın.

#### PostgreSQL UPSERT örneği

```java
@Modifying
@Query(value = """
    INSERT INTO idempotency_keys (key, response_body, created_at)
    VALUES (:key, :body, NOW())
    ON CONFLICT (key) DO NOTHING
""", nativeQuery = true)
int upsertIdempotencyKey(@Param("key") String key, @Param("body") String body);
```

Return değeri 1 = yeni kayıt, 0 = zaten vardı (idempotency-key bulundu).

---

### 12. Connection-level query optimization

#### `statement_timeout`

PostgreSQL'in DB tarafı query timeout. SQL çok yavaşsa kullanıcı beklemez, DB öldürür.

```sql
SET statement_timeout = '5s';
```

Hibernate property:

```yaml
spring:
  datasource:
    hikari:
      data-source-properties:
        socketTimeout: 30
        loginTimeout: 5
```

Veya `application.yml` Spring-level:

```yaml
spring:
  jpa:
    properties:
      jakarta.persistence.query.timeout: 5000   # 5 saniye
```

Banking pratiği: API endpoint için 5 saniye, batch için 5 dakika.

#### `idle_in_transaction_session_timeout`

PostgreSQL'in "TX açtı ama 10 dk hiç bir şey yapmadı" timeout'u. Banking için:

```sql
SET idle_in_transaction_session_timeout = '30s';
```

Aşıldığında PostgreSQL session'ı kill. Open TX leak'ini fail-fast yapar.

---

### 13. Hibernate 6 yenilikleri (Spring Boot 3.x)

**Hibernate 6 ile gelen önemli değişiklikler:**

1. **Yeni query parser** (JPA Criteria desteği daha sıkı, eski Hibernate-specific feature'lar bazen kırıldı)
2. **SQM (Semantic Query Model)** — JPQL'in iç temsili yenilendi
3. **`@Cache` annotation Jakarta migration** — `jakarta.persistence` namespace
4. **`@SoftDelete` annotation (experimental)** — `@SQLDelete` + `@Where`'in standardı
5. **Yeni `@JdbcType` annotation** — custom type mapping (örnek PostgreSQL JSONB, ENUM)
6. **MultipleBagFetchException artık fırlatılmıyor** — bazı durumlarda Cartesian product gelir, dikkat

**Banking migration:** Spring Boot 2.x → 3.x geçişte Hibernate 6 query'leri yeniden test edilmeli. Eski `JOIN FETCH DISTINCT` davranışı farklı (gerek kalmadı).

---

### 14. Profiling — JFR + flame graph

JVM Flight Recorder ile Hibernate workload profile:

```bash
java -XX:StartFlightRecording=duration=60s,filename=banking.jfr -jar core-banking.jar
```

`banking.jfr` dosyasını **JDK Mission Control** veya **async-profiler** ile aç. CPU flame graph'ta:

- `Hibernate.SQL` çok zaman alıyor → JPA optimization gerek
- `JDBC.getConnection` → pool darboğaz
- `PostgreSQL.Socket.read` → DB darboğaz veya network

Banking'de production profiling **opsiyonel ama altın değer**. Phase 9'da APM ile detay.

---

### 15. Anti-pattern'ler

**Anti-pattern 1: IDENTITY ID + batch beklemek**

Yukarıda detaylı anlatıldı. IDENTITY + batch = imkânsız. SEQUENCE allocate.

**Anti-pattern 2: String concat ile JPQL/HQL**

```java
em.createQuery("SELECT a FROM AccountJpaEntity a WHERE a.id = " + id)   // ❌
```

Plan cache leak + SQL injection. Daima `:param`.

**Anti-pattern 3: 2L cache balance veya money field'larda**

`@Cache` Account, JournalLine üzerinde **YASAK**. Stale balance = banking faciası.

**Anti-pattern 4: `findAll()` pageable'sız**

Phase 2.2'de değinildi. Tekrar: 1M+ row tabloda OOM.

**Anti-pattern 5: 1 TX'te 1M satır işlemek**

```java
@Transactional
public void process() {
    for (int i = 0; i < 1_000_000; i++) {
        repo.save(...);   // PC 1M entity → OOM
    }
}
```

Çare: chunk'larla (10.000'lik batch), `em.flush(); em.clear();` her batch sonu. Veya `StatelessSession`.

**Anti-pattern 6: Persistence context'i temizlememek (batch'te)**

```java
for (int i = 0; i < 100_000; i++) {
    repo.save(entity);
    if (i % 1000 == 0) {
        em.flush();
        // em.clear() ← burada eksik → PC şişer
    }
}
```

`em.flush() + em.clear()` her batch.

**Anti-pattern 7: 2L cache invalidation eventi yok**

Cache aktif, ama veriyi external source'tan değiştirdiğinde (örnek: DB direkt UPDATE) cache stale kalır. Domain event veya scheduled refresh ile invalidation şart.

**Anti-pattern 8: `@Modifying` UPDATE/DELETE sonrası PC bypass**

`clearAutomatically = true` eksik → sonraki `findById` eski version döner.

**Anti-pattern 9: Native SQL her yerde**

JPA'nın faydalarını kaybedersin (PC, cascade, dirty checking). Sadece gerektiği yerde native.

---

## Önemli olabilecek araştırma kaynakları

- Vlad Mihalcea "High-Performance Java Persistence" (kitap)
- Hibernate User Guide — Performance chapter
- Vlad Mihalcea blog — batch insert, SEQUENCE, allocation size series
- "Practical Hibernate" Thorben Janssen (Patreon + blog)
- PostgreSQL `pg_stat_statements` documentation
- `log_min_duration_statement` PostgreSQL config
- Hibernate 6 migration guide
- "Java Performance" Scott Oaks (kitap)
- JFR + flame graph workflow
- Ehcache, Caffeine, Hazelcast documentation

---

## Mini task'ler

### Task 2.7.1 — IDENTITY → SEQUENCE migration (45 dk)

Mevcut entity'lerden birinin (örnek `TransactionJpaEntity`) ID strategy'sini `IDENTITY` → `SEQUENCE` + `allocationSize = 50` ile değiştir.

Migration V6:

```sql
CREATE SEQUENCE transaction_sequence START WITH 1000 INCREMENT BY 50;
ALTER TABLE transactions ALTER COLUMN id DROP DEFAULT;
-- (production'da eski IDENTITY ile uyumluluk için karmaşık migrate gerekir)
```

Test öncesi/sonrası: 1000 transaction insert eden test yaz. Önce IDENTITY ile süreyi ölç, sonra SEQUENCE+batch ile. Defter notuna yaz.

### Task 2.7.2 — Batch config aktif (30 dk)

`application.yml`'a batch config ekle:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true

logging:
  level:
    org.hibernate.engine.jdbc.batch.internal.BatchingBatch: DEBUG
```

1000 entity insert eden test. Log'da `Executing batch (size: 50)` görmeli, ~20 batch.

### Task 2.7.3 — `StatelessSession` import service (45 dk)

`BulkImportService` yaz. `StatelessSession` ile 100.000 random transaction insert et.

Karşılaştır:
- Normal `EntityManager` + batch + flush+clear: süre ?
- `StatelessSession`: süre ?
- Memory: `jconsole` veya `jstat -gc` ile heap kullanımı izle

Defter notu: hangi senaryoda hangisi?

### Task 2.7.4 — Bulk update `@Modifying` (30 dk)

`markTransactionsAsProcessed(batchId)` JPQL `UPDATE` query'si yaz, `@Modifying(clearAutomatically = true)` ile.

10.000 transaction'ı `PENDING` → `PROCESSED` etmek için iki yöntem karşılaştır:
- Domain entity ile her birini find + update + save: süre ?
- `@Modifying` bulk: süre ?

### Task 2.7.5 — Query plan cache leak reprodüksiyon (45 dk)

Kasten plan cache'i şişiren kod:

```java
for (int i = 0; i < 5000; i++) {
    String hql = "SELECT a FROM AccountJpaEntity a WHERE a.id = '" + UUID.randomUUID() + "'";
    em.createQuery(hql, AccountJpaEntity.class).getResultList();
}
```

`hibernate.query.plan_cache_max_size = 100` ile çalıştır. Heap dump al (`jmap -dump:format=b,file=heap.bin <pid>`), MAT (Eclipse Memory Analyzer) ile aç. `QueryPlanCache` referansını ara.

Düzelt: `:param` ile parametric.

### Task 2.7.6 — Ehcache 2L kurulumu (1 saat)

`Currency` (kod, isim, fractionDigits) entity'si ekle. 2L cache READ_ONLY ile aktif et.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax.cache.provider: org.ehcache.jsr107.EhcacheCachingProvider
```

`ehcache.xml`:

```xml
<config xmlns:jsr107="http://www.ehcache.org/v3/jsr107">
    <cache alias="com.mavibank.banking.currency.adapter.out.persistence.CurrencyJpaEntity">
        <expiry><ttl unit="hours">1</ttl></expiry>
        <heap unit="entries">200</heap>
    </cache>
</config>
```

Test: `currencyRepo.findById("TRY")` 100 kere çağır. SQL log'da yalnız **1 SELECT** olmalı (sonrası cache).

`/actuator/metrics/cache.gets?tag=cache:Currency` ile hit/miss count'u gör.

### Task 2.7.7 — Hibernate Statistics + Micrometer (45 dk)

`HibernateMetricsRecorder` yaz (yukarıdaki örnek). `@Scheduled` ile 30 saniyede bir Statistics'i çekip Micrometer'a yazsın.

`/actuator/metrics/hibernate.query.count` endpoint'inde değeri görmeli.

Grafana'da panel ekle: hibernate.query.count rate'i, hibernate.second_level_cache.hit.ratio.

### Task 2.7.8 — PostgreSQL slow query log + `pg_stat_statements` (1 saat)

`postgresql.conf` (docker-compose `command`'a ekle):

```yaml
postgres:
  command: |
    postgres
    -c shared_preload_libraries=pg_stat_statements
    -c log_min_duration_statement=100
    -c log_lock_waits=on
```

Migration:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Bir batch endpoint çalıştır (yavaş query üretsin), sonra:

```sql
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

Sonuçları `docs/slow-queries.md`'a yapıştır.

---

## Test yazma rehberi

### Test 2.7.1 — Batch insert verification

```java
@SpringBootTest
@Testcontainers
class BatchInsertTest {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired TransactionJpaRepository repo;
    @Autowired EntityManagerFactory emf;
    
    @Test
    @Transactional
    void should_execute_inserts_in_batches() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        sf.getStatistics().clear();
        
        List<TransactionJpaEntity> batch = IntStream.range(0, 1000)
            .mapToObj(i -> createEntity())
            .toList();
        
        repo.saveAll(batch);
        repo.flush();
        
        Statistics stats = sf.getStatistics();
        // 1000 entity, batch_size=50 → ~20 prepared statement çağrısı
        assertThat(stats.getEntityInsertCount()).isEqualTo(1000);
        assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(25);  // ~20 batch + biraz sequence
    }
}
```

### Test 2.7.2 — Bulk update PC bypass davranışı

```java
@Test
@Transactional
void bulkUpdate_shouldClearPersistenceContext_whenAnnotated() {
    UUID id = createEntityWithStatus("PENDING");
    TransactionJpaEntity entity = repo.findById(id).orElseThrow();
    assertThat(entity.getStatus()).isEqualTo("PENDING");
    
    repo.markAsProcessedBulk(id);   // @Modifying(clearAutomatically = true)
    
    // Aynı TX'te tekrar findById — PC clear edildi, fresh load
    TransactionJpaEntity afterUpdate = repo.findById(id).orElseThrow();
    assertThat(afterUpdate.getStatus()).isEqualTo("PROCESSED");
}
```

### Test 2.7.3 — L2 cache hit/miss

```java
@Test
void l2CacheShouldHitOnSecondLoad() {
    SessionFactory sf = emf.unwrap(SessionFactory.class);
    sf.getStatistics().clear();
    
    // İlk find — DB hit
    currencyRepo.findById("TRY").orElseThrow();
    long missAfter1 = sf.getStatistics().getSecondLevelCacheMissCount();
    
    // İkinci find — cache hit (TX kapanmış, yeni TX)
    transactionTemplate.executeWithoutResult(s -> 
        currencyRepo.findById("TRY").orElseThrow()
    );
    long hitAfter2 = sf.getStatistics().getSecondLevelCacheHitCount();
    
    assertThat(missAfter1).isEqualTo(1);
    assertThat(hitAfter2).isGreaterThanOrEqualTo(1);
}
```

### Test 2.7.4 — Plan cache stability

```java
@Test
void parameterizedQuery_shouldNotInflatePlanCache() {
    SessionFactory sf = emf.unwrap(SessionFactory.class);
    
    for (int i = 0; i < 1000; i++) {
        UUID randomId = UUID.randomUUID();
        accountRepo.findById(randomId);   // parametric query
    }
    
    // Plan cache'te sadece 1 entry olmalı
    long planCount = sf.getStatistics().getQueryPlanCacheHitCount() 
                   + sf.getStatistics().getQueryPlanCacheMissCount();
    assertThat(sf.getStatistics().getQueryPlanCacheHitCount()).isGreaterThan(900);
}
```

---

## Claude-verify prompt

```
Aşağıdaki Hibernate performans kodumu banking-grade kriterlere göre değerlendir. 
PASS / FAIL / EKSIK işaretle, KOD YAZMA:

1. Batch insert:
   - ID generation strategy IDENTITY mi (batch'i kırıyor) yoksa SEQUENCE/UUID mi?
   - SEQUENCE kullanılıyorsa `allocationSize` >= 50 mi?
   - `hibernate.jdbc.batch_size` set mi (>= 25)?
   - `order_inserts` ve `order_updates` true mu?
   - `batch_versioned_data` versionlu entity'ler için true mu?
   - Test'te `Executing batch (size: X)` log'u görülüyor mu?

2. Persistence context yönetimi:
   - Büyük loop'larda `em.flush(); em.clear();` her batch sonu var mı?
   - `StatelessSession` ETL job'larda kullanılıyor mu?
   - 1M+ kayıt tek TX'te işleniyor mu? (Olmamalı)

3. Bulk update/delete:
   - `@Modifying` JPQL bulk update için var mı?
   - `clearAutomatically = true` PC tutarlılığı için set mi?
   - `flushAutomatically = true` gerektiği yerde mi?

4. Query plan cache:
   - JPQL string concat var mı? (Olmamalı, SQL injection + plan leak)
   - Parameterized query'ler `:param` ile mi?
   - `plan_cache_max_size` 4096+ mı (banking için)?
   - Plan cache hit ratio test'te > %95 mi?

5. Statement caching:
   - JDBC URL `prepareThreshold` ayarı var mı?
   - PgBouncer transaction mode varsa `prepareThreshold = 0` mı?

6. Second-level cache:
   - L2 cache aktif mi (`use_second_level_cache: true`)?
   - Provider seçimi mantıklı mı (Ehcache tek instance, Hazelcast multi-replica)?
   - Hangi entity'ler cache'li (lookup data: currency, country)?
   - Hangi entity'ler cache'siz (Account, JournalLine — banking kuralı)?
   - Cache concurrency strategy money entity'lerde NONSTRICT_READ_WRITE değil mi? (Olmamalı)

7. Query cache:
   - Banking'in append-heavy yapısı için query cache **kapalı** mı? (Default kapalı OK)
   - Açıksa, sadece read-only lookup query'lerinde mi?

8. Statistics & monitoring:
   - `hibernate.generate_statistics: true` aktif mi?
   - Micrometer'a hibernate metric'leri yazılıyor mu?
   - Slowest query log alarmı kuruldu mu?

9. PostgreSQL tarafı:
   - `log_min_duration_statement` set mi (100-250 ms)?
   - `pg_stat_statements` extension yüklü mü?
   - `log_lock_waits` aktif mi?
   - `idle_in_transaction_session_timeout` set mi?

10. JPQL vs Native SQL:
    - JPQL dominant (~%85) mi, gerektiğinde native mi?
    - Native SQL gerekçeli mi (CTE, JSONB, UPSERT)?
    - Native query DB-agnostic mi (kaçınılmazsa) yoksa PostgreSQL-specific ve dokümante mi?

11. Anti-pattern:
    - IDENTITY + batch beklemek? (Olmamalı)
    - String concat JPQL/SQL? (Olmamalı)
    - 2L cache money/balance entity'de? (Olmamalı)
    - `findAll()` pageable'sız? (Olmamalı)
    - 1 TX'te 1M+ satır? (Olmamalı)
    - PC clear'sız bulk update + sonra find? (Olmamalı)
    - 2L cache invalidation event'i eksik? (Stale risk)

12. Banking-grade kalite:
    - Performance ADR yazılı mı (allocationSize, batch_size, L2 cache stratejisi)?
    - Slow query CI'de yakalanır mı (test'te Hibernate Statistics assertion)?
    - Production'da JFR profile aldın mı?
    - APM (Datadog/New Relic) entegrasyonu hazır mı (Phase 9'da detay)?

Her madde için PASS / FAIL / EKSIK ve kısa gerekçe. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] IDENTITY → SEQUENCE migration yaptım, `allocationSize=50` ile batch çalışıyor
- [ ] `hibernate.jdbc.batch_size`, `order_inserts/updates`, `batch_versioned_data` aktif
- [ ] BatchingBatch DEBUG log'da batch execution gördüm
- [ ] `StatelessSession` ile mass insert örneği yazdım
- [ ] `@Modifying(clearAutomatically = true)` bulk update kullanıyorum
- [ ] Query plan cache leak'i reprodüksiyon ve düzelttim
- [ ] Ehcache ile lookup entity'leri (Currency) 2L cache'te
- [ ] Money/balance entity'lere 2L cache koymadığımı doğruladım (banking kuralı)
- [ ] `HibernateMetricsRecorder` Micrometer'a yazıyor
- [ ] PostgreSQL `log_min_duration_statement` aktif, `pg_stat_statements` çalışıyor
- [ ] Anti-pattern listesi rahat (IDENTITY+batch, string concat, money cache, vs.)

Hepsi onaylı → Phase 2 mini-project'e geç → [../mini-project/](../mini-project/index.md)

---

## Defter notları

1. "IDENTITY ID strategy'sinin batch insert'i neden kırdığını açıklayan tek cümle: ____."
2. "`allocationSize` 50 olunca Hibernate nasıl davranır: ____. 1 olsa fark: ____."
3. "`hibernate.order_inserts = true` ne zaman değer ekler: ____."
4. "`StatelessSession` PC kullanmadığı için elde ettiğin 2 fayda: ____, ____. Kaybettiklerin: ____."
5. "`@Modifying(clearAutomatically = true)` neden gerek: ____."
6. "Query plan cache leak senaryosu: ____. Önleme: ____."
7. "L2 cache banking'de hangi entity'lerde uygundur (3 örnek): ____. Hangilerine YASAK (gerekçeli): ____."
8. "Cache concurrency strategy seçimi: lookup data ____, mutable data ____."
9. "PostgreSQL `pg_stat_statements`'in `slow query log`'a göre avantajı: ____."
10. "`idle_in_transaction_session_timeout` neden banking için kritik: ____."
