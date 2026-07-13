# Faz 2 — PHASE TEST

```admonish question title="Bu test ne işe yarar?"
Faz 3'e geçmeden önce kendini sına. Bu bir sınav değil, **dürüstlük kontrolü**:
işaretleyemediğin her madde, hangi topic'e geri döneceğini gösterir.
Hepsine "evet" diyebiliyorsan hazırsın.
```

Faz 2 derindi: JPA internals, transactions, locking, N+1, connection pool, Hibernate performance. Bu konular **junior'dan mid-junior'a geçişin** ana göstergesi. Burayı sağlam atarsan Faz 3-9 arası çok daha hızlı geçer.

## Pratik test — projeden

- [ ] `core-banking` repo'sunda Phase 2 mini-project'in tüm değişiklikleri commit'li
- [ ] `mvn verify` lokalde geçiyor, JaCoCo coverage ≥ %75
- [ ] `docker compose up -d` ile PostgreSQL + PgBouncer + Prometheus + Grafana çalışıyor
- [ ] `mvn spring-boot:run` ile uygulama ayağa kalkıyor
- [ ] Grafana `localhost:3000`'da 3 panel (HikariCP, Hibernate, Business) canlı veri gösteriyor
- [ ] OpenAPI UI'da yeni endpoint'ler (`/v1/accounts/{id}/transactions`, `PATCH /v1/accounts/{id}`) görünür
- [ ] `/actuator/prometheus`'ta hikaricp_* ve hibernate_* metric'leri görünüyor
- [ ] 100 iterasyon A↔B transfer test'i 0 deadlock dönüyor
- [ ] 5 yeni smoke test senaryosu manuel olarak çalıştırıldı, screenshot'lar `docs/smoke-test-evidence/`'ta

---

## Konsept testi — sorular ve defterinde yazılı cevaplar

### JPA Fundamentals (Topic 2.1)

- [ ] Persistence Context'in 4 ana sorumluluğunu sayabilirim (identity map, dirty checking, transactional write-behind, lazy loading)
- [ ] Entity state lifecycle (transient → managed → detached → removed) ve transition'ları biliyorum
- [ ] `EntityManager.persist`, `merge`, `find`, `getReference`, `flush`, `clear`, `detach` farklarını net söyleyebilirim
- [ ] `merge` neden `persist` gibi davranmaz — argümanı geri döndürdüğü sebep
- [ ] First-level cache (L1) ne demek, ne zaman temizlenir
- [ ] Flush mode AUTO vs COMMIT vs MANUAL ne zaman hangisi
- [ ] `@OneToMany` mappedBy ile bidirectional ilişki ve owning side kavramı

### Spring Data JPA (Topic 2.2)

- [ ] `Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository` hierarchy
- [ ] Derived query method keyword'lerinden 5 tanesini söyleyebilirim (And, Or, GreaterThan, In, Between, OrderBy, Top10, ...)
- [ ] Derived query 4+ koşul olduğunda neden `@Query` tercih etmeliyim
- [ ] JPQL ile native SQL arasında seçim kriteri (DB-specific, complex CTE, performans)
- [ ] `@Modifying` neden zorunlu, `clearAutomatically` ne yapar
- [ ] `Page` vs `Slice` vs `List` farkı ve banking'de hangisi ne zaman
- [ ] Cursor-based vs offset-based pagination'ın trade-off'u
- [ ] Specification pattern'i ne zaman tercih ederim
- [ ] DTO projection (record + constructor expression) vs interface projection
- [ ] Auditing pattern'ini (CreatedDate, LastModifiedDate, CreatedBy, LastModifiedBy) doğru kurabilirim
- [ ] Soft delete pattern `@SQLDelete + @Where` ile, banking'de hangi tablolarda hangi tablolarda DEĞİL
- [ ] `getReferenceById` vs `findById` farkı + ne zaman hangisi

### Transactions (Topic 2.3)

- [ ] ACID properties (Atomicity, Consistency, Isolation, Durability) banking örneği ile her birini açıklayabilirim
- [ ] `@Transactional`'ın Spring proxy mekaniği (JDK Dynamic vs CGLIB) — ne zaman hangisi
- [ ] Self-invocation problemini reprodüksiyon ettim, 2 farklı çözüm uyguladım (self-injection, AspectJ)
- [ ] 7 propagation tipini (REQUIRED, REQUIRES_NEW, NESTED, MANDATORY, SUPPORTS, NOT_SUPPORTED, NEVER) banking use case'leri ile söyleyebilirim
- [ ] REQUIRES_NEW ile audit log pattern: "yetersiz bakiye olsa bile audit kalmalı" senaryosu
- [ ] NESTED ile REQUIRES_NEW farkı (savepoint vs ayrı TX)
- [ ] 4 isolation level + 3 phenomenon (dirty read, non-repeatable read, phantom read) eşleştirme matrisi
- [ ] PostgreSQL REPEATABLE_READ ile MySQL InnoDB REPEATABLE_READ farkı
- [ ] Checked vs unchecked exception default rollback davranışı — neden banking domain exception'ları `RuntimeException` extend etmeli
- [ ] `rollbackFor` ve `noRollbackFor` ne zaman gerek
- [ ] `readOnly = true` ne sağlar (Hibernate dirty checking kapanır, replica routing potansiyeli)
- [ ] `@TransactionalEventListener(AFTER_COMMIT)` ile event-based notification pattern
- [ ] TX içinde external HTTP call yapmamamın 3 sebebi (timeout, pool exhaustion, rollback paradoksu)

### Locking (Topic 2.4)

- [ ] Optimistic locking mekanizmasını (`@Version` + UPDATE WHERE version = ?) anlatabilirim
- [ ] Pessimistic `SELECT FOR UPDATE` ile DB-level row lock alır
- [ ] `LockModeType.PESSIMISTIC_READ`, `PESSIMISTIC_WRITE`, `PESSIMISTIC_FORCE_INCREMENT`, `OPTIMISTIC_FORCE_INCREMENT` farkları
- [ ] `@Lock` annotation ile JPA repository method'a lock ekleme
- [ ] `FOR UPDATE NOWAIT` vs `WAIT n` vs `SKIP LOCKED` — banking senaryoları
- [ ] `SKIP LOCKED` ile job queue pattern (multi-worker concurrent claim)
- [ ] Deadlock'un nasıl oluştuğunu (A→B + B→A) çizebilirim
- [ ] Lock ordering ile deadlock fix prensibi
- [ ] PostgreSQL SERIALIZABLE (SSI) ile pessimistic locking arasındaki algoritmik fark
- [ ] Retry pattern'in 5 unsuru: max attempts, exponential backoff, jitter, max delay cap, sadece concurrency exception
- [ ] Optimistic vs Pessimistic karar matrisi (conflict rate, TX duration, distributed mi)
- [ ] Banking'de money movement neden pessimistic, profile update neden optimistic

### N+1 Problem (Topic 2.5)

- [ ] N+1'in temel sebebi (LAZY collection iterated in loop)
- [ ] N+1'i tespit etmenin 3 yolu: SQL log, Hibernate statistics, test'te assertQueryCount
- [ ] `@ManyToOne` default'unun EAGER olduğunu ve banking'de **her zaman explicit LAZY** yapmam gerektiğini biliyorum
- [ ] EAGER cascade fetch explosion senaryosunu açıklayabilirim
- [ ] OSIV (Open Session In View) açık olduğunda neden gizli N+1 olur — production'da `open-in-view: false`
- [ ] `JOIN FETCH` (JPQL) ile `@EntityGraph` farkı — ne zaman hangisi
- [ ] `JOIN FETCH` ile DISTINCT keyword'ü neden gerek
- [ ] Pagination + collection JOIN FETCH'in tehlikesi (LIMIT in-memory) ve 2-query çözümü
- [ ] Batch fetching mekanizması (`@BatchSize` veya `default_batch_fetch_size`)
- [ ] DTO projection ne zaman tercih, entity'ye göre avantajları
- [ ] `MultipleBagFetchException` sebebi (2 @OneToMany JOIN FETCH) ve 2 çözümü
- [ ] 4 yöntemin (JOIN FETCH, EntityGraph, batch, DTO) trade-off matrisini çıkarabilirim

### Connection Pool — HikariCP (Topic 2.6)

- [ ] Connection açmanın 100-300ms maliyeti (TCP, TLS, auth, init)
- [ ] HikariCP'nin hızının 3 sebebi: ConcurrentBag, thread-local fast-path, bytecode optimization
- [ ] Brian Goetz formülü: `connections = ((core_count * 2) + effective_spindle_count)`
- [ ] K8s replica * pool size = toplam DB connection — `max_connections`'ı geçmemeli
- [ ] Banking workload tipleri (API kısa TX, batch uzun TX, reporting read-heavy) için pool boyutu farkı
- [ ] `connectionTimeout` banking API için 2-5 saniye
- [ ] `maxLifetime` = DB-side timeout - 30 saniye formülü
- [ ] `keepaliveTime` neden firewall idle koruması için kritik
- [ ] `leakDetectionThreshold` dev/prod ayrımı
- [ ] Pool exhaustion 3 belirti: pending threads, timeout, P95 spike
- [ ] `jstack` ile blocked thread'leri tespit, hangi thread connection'ı tutuyor
- [ ] Connection leak'i `try-with-resources` ile önleme
- [ ] PgBouncer transaction pooling neden K8s multi-replica için gerek, session vs transaction vs statement mode farkları
- [ ] `prepareThreshold=0` PgBouncer transaction mode'da neden zorunlu
- [ ] HikariCP Micrometer metrics → Prometheus → Grafana stack'ini kurabilirim
- [ ] Gatling load test ile pool sweet spot bulma metodolojisi

### Hibernate Performance (Topic 2.7)

- [ ] Batch insert'i kırma sebebi: `IDENTITY` ID strategy — `SEQUENCE` allocationSize 50 ile çözüm
- [ ] `hibernate.jdbc.batch_size`, `order_inserts`, `order_updates`, `batch_versioned_data` config kombinasyonu
- [ ] `StatelessSession` ETL job'larda neden tercih (PC yok, dirty check yok, memory ucuz)
- [ ] `@Modifying(clearAutomatically = true)` bulk update sonrası PC tutarlılığı
- [ ] Query plan cache memory leak senaryosu (string concat JPQL) ve önleme (`:param`)
- [ ] `hibernate.query.plan_cache_max_size` banking için 4096+
- [ ] L2 cache (`use_second_level_cache: true`) — banking'de **hangi entity'lerde cache** ve **hangi entity'lerde KESINLIKLE cache değil**
- [ ] Cache concurrency strategy seçimi: lookup → READ_ONLY veya READ_WRITE, money → cache yok
- [ ] Cache provider seçimi: Ehcache (single instance), Hazelcast (K8s multi-replica), Redis (büyük cache)
- [ ] Query cache neden append-heavy banking için **kapalı** (sürekli invalidation)
- [ ] Hibernate Statistics Micrometer entegrasyonu ile production izleme
- [ ] PostgreSQL `log_min_duration_statement = 250` ve `pg_stat_statements` extension
- [ ] `idle_in_transaction_session_timeout` neden TX leak fail-fast için kritik
- [ ] JPQL %85, native SQL %15 rule — native ne zaman gerek (CTE, JSONB, UPSERT, window functions)

---

## Banking domain — concurrency & performance hakkında derinlemesine

- [ ] Para transferinde 3 entity (account_from, account_to, journal_entry) atomik TX'te değiştirilmek zorunda — sebebi ACID Atomicity
- [ ] Double-entry invariant (sum debit = sum credit) hem domain logic'te hem DB CHECK constraint ile garanti
- [ ] Optimistic locking + idempotency-key kombinasyonu retry'ı güvenli yapar (aynı işlem 2 kere kaydedilmez)
- [ ] Pessimistic + lock ordering pattern'i microservice'siz banking için altın standard
- [ ] PostgreSQL SERIALIZABLE alternatifi yüksek concurrency'de pessimistic'e göre throughput avantajı
- [ ] Account balance L2 cache'i neden YASAK (stale balance = yetersiz fonla işlem yapma riski)
- [ ] Currency, Country, Branch, FeeSchedule cache'lemeye uygun (statik veya yavaş değişen)
- [ ] Banking saatleri (07:00-22:00 high traffic, gece batch) pool sizing'i etkiler
- [ ] EFT vs Havale vs SWIFT — her birinin TX süresi farklı, pool sizing'i etkiler
- [ ] Regulatör isteği: audit kayıtları **physical delete yok**, soft delete bile sınırlı

---

## Test alışkanlığı (Phase 2 ekstra)

- [ ] `@DataJpaTest` + TestContainers PostgreSQL ile repository test'i yazıyorum
- [ ] `@SpringBootTest` + TestContainers ile full integration test
- [ ] H2 hiç bir test'te kullanılmıyor (PostgreSQL ile semantic difference)
- [ ] Concurrent test'lerde `ExecutorService` + `CountDownLatch` pattern'i biliyorum
- [ ] Hibernate Statistics ile `assertQueryCount` helper'ı test'lerime entegre ettim
- [ ] N+1 regression test'i CI'da koşuyor — yeni dev N+1 eklerse build kırılır
- [ ] Optimistic conflict, deadlock, pool exhaustion, leak detection reprodüksiyon test'leri var
- [ ] Test isimleri Türkçe veya İngilizce ama **net** (`shouldEliminateDeadlockWithLockOrdering`)

---

## Mülakat hazırlığı — bu cümleleri rahat söyleyebilmelisin

- [ ] "Banking'de para transferi için pessimistic locking + lock ordering kullanırım, çünkü ____"
- [ ] "Profile update gibi düşük çakışmalı operasyonlarda optimistic locking + retry tercih ederim, çünkü ____"
- [ ] "N+1 problemini şu 3 yöntemle tespit ederim: ____, ____, ____"
- [ ] "HikariCP pool boyutunu şöyle belirlerim: Brian Goetz formülü + K8s replica + DB max_connections ____"
- [ ] "Hibernate batch insert için neden SEQUENCE + allocationSize gerek, IDENTITY niye işe yaramaz ____"
- [ ] "PostgreSQL serialization failure (40001) gördüğümde retry stratejim ____"
- [ ] "Connection leak'i nasıl tespit edip düzeltirim ____"
- [ ] "L2 cache'i Account üzerinde neden YASAK ____"
- [ ] "Spring `@Transactional` self-invocation problemini ____ ile çözüyorum"
- [ ] "Banking'de OSIV neden kapalı olmalı ____"

---

## Bu fazda kaç gün harcadım?

Tahmin: ~21 gün (günde 2-3 saat). Daha az veya çok olabilir. Faz 2 Faz 1'den **derindi** — acele etmek anlamsız.

Kendine sor: "Eğer bir TR bank mülakatında JPA, transaction, locking, N+1, pool tuning sorulsa, **rahatça** anlatabilir miyim? Kod yazsalar yazabilir miyim?"

Cevabın "evet" ise **Faz 3'e geç** → `../03-concurrency/`

Cevabın "hayır" veya emin değilim ise:
- Hangi konu zayıf, oraya geri dön
- Mini-project'in K1-K5 kasten kırma görevlerinden eksik olan varsa yap
- Pratik = ezberi yenen tek şey

---

## Bonus — mid-junior'a geçtiğinin işareti

Faz 2'yi bitirdiğinde, şu cümleleri **kendinden ödün vermeden** söyleyebiliyorsan mid-junior+ seviyesindesin:

- "Banking-grade transaction management'ı pratik olarak uygulayabilirim — propagation, isolation, lock ordering hepsi yerinde."
- "Hibernate'in performans karakteristiğini biliyorum: batch insert, query plan cache, L2 cache, slow query detection."
- "Connection pool tuning'i deneysel yapıyorum — Gatling ile load test, Grafana ile monitoring."
- "N+1 problem'i hem reprodüksiyon hem fix yapabilirim, 4 farklı yöntemi trade-off'larıyla biliyorum."
- "Concurrency problemlerini deadlock-free şekilde çözebilirim — lock ordering, retry with backoff, optimistic vs pessimistic seçimi."

Bunlar **mid-level Java backend developer**'ın bilmesi gerekenlerin önemli kısmı. Phase 1 + Phase 2 = mid-junior+. Phase 5 sonunda mid, Phase 9 sonunda mid+.

---

## Faz 3'te ne var?

Faz 3 = **JVM-level concurrency**. Şu ana kadar DB-level concurrency (locking, transactions) gördün. Faz 3'te:

- `synchronized` ve `volatile` JMM (Java Memory Model)
- `java.util.concurrent` (Lock, ReentrantLock, ReadWriteLock, StampedLock)
- Atomic classes (`AtomicInteger`, `AtomicReference`, CAS pattern)
- Thread pools, `ExecutorService`, `CompletableFuture`
- `BlockingQueue`, producer-consumer pattern
- Virtual threads (Java 21) — banking için ne anlama geliyor
- Thread-safe collection'lar (`ConcurrentHashMap`, `CopyOnWriteArrayList`)
- Reactive intro (Project Reactor) — Phase 4'te detay
- Banking pattern'ler: rate limiter, circuit breaker pattern (Resilience4j)

```admonish success title="Sonraki durak: Faz 3"
İyi şanslar. Phase 2'nin temeli sağlam ise Faz 3 ufkunu çok genişletecek —
banking'de high-throughput sistem nasıl yazılır sorusuna cevap verecek.
→ [Faz 3 — Concurrency & JVM](../03-concurrency/index.md)
```
