# Topic 2.5 — N+1 Query Problem

## Hedef

JPA/Hibernate'in en sık karşılaşılan ve en yıkıcı performans problemi olan **N+1 query problemini** banking domain'inde reprodüksiyon kodu ile görmek. Sebebinin (LAZY association iterated in loop) net olarak görselleştirilmesi, 4 farklı çözüm yöntemini (`JOIN FETCH`, `@EntityGraph`, batch fetching, DTO projection) karşılaştırarak hangisinin ne zaman doğru olduğunu öğrenmek. Hibernate statistics ve SQL log üzerinden N+1'i **tespit etmek** ve düzeltmek. `FetchType.EAGER` ile cascade fetch explosion arasındaki ilişkiyi anlamak. `GET /accounts/{id}/transactions` endpoint'i üzerinden gerçek banking endpoint'inde N+1 problemini reprodüksiyon edip 4 yaklaşımı SQL count'ları ile karşılaştırmak.

## Süre

Okuma: 1.5 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 2.1 (JPA Fundamentals) ve 2.2 (Spring Data JPA) bitti
- `FetchType.LAZY` ve `FetchType.EAGER` arasındaki temel farkı duydun
- Hibernate'in `@OneToMany`, `@ManyToOne` association tiplerini kullandın
- "Hibernate niye bu kadar query atıyor?" sorusunu en az bir kere sordun

---

## Kavramlar

### 1. N+1 nedir — somut bir banking örneği

Senaryo: Bir müşterinin tüm hesaplarını ve her hesabın son 10 transaction'ını rapor olarak listelemek istiyorsun.

```java
@Entity
@Table(name = "accounts")
public class AccountJpaEntity {
    @Id UUID id;
    @Column UUID ownerId;
    @Column String currency;
    @Column BigDecimal balanceAmount;
    
    @OneToMany(mappedBy = "account", fetch = FetchType.LAZY)
    private List<JournalLineJpaEntity> journalLines = new ArrayList<>();
    
    // getters
}

@Entity
@Table(name = "journal_lines")
public class JournalLineJpaEntity {
    @Id UUID id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "account_id")
    private AccountJpaEntity account;
    
    @Column BigDecimal amount;
    @Column String direction;   // DEBIT or CREDIT
    @Column Instant occurredAt;
    
    // getters
}
```

Naif servis:

```java
@Service
@Transactional(readOnly = true)
public class CustomerReportService {
    
    public List<CustomerAccountReport> generateReport(UUID ownerId) {
        List<AccountJpaEntity> accounts = accountRepo.findByOwnerId(ownerId);   // ① 1 query
        
        return accounts.stream()
            .map(acc -> new CustomerAccountReport(
                acc.getId(),
                acc.getBalanceAmount(),
                acc.getJournalLines().size()    // ② her hesap için 1 query  → N query
            ))
            .toList();
    }
}
```

SQL log'da:

```sql
-- ① Tüm hesapları çek
SELECT * FROM accounts WHERE owner_id = ?;
-- ② Her hesap için ayrı query (örnek: 50 hesap)
SELECT * FROM journal_lines WHERE account_id = ?;   -- account 1
SELECT * FROM journal_lines WHERE account_id = ?;   -- account 2
SELECT * FROM journal_lines WHERE account_id = ?;   -- account 3
-- ... 50 kere
```

**Toplam:** 1 + 50 = 51 query. "N+1" tam burdan: 1 ana query + N child query.

**Banking'de yıkıcı sonuçlar:**

- 50 hesabı olan bir kurumsal müşteri → 51 query
- 1000 müşterilik bir batch report → 1000 + 1000*50 = **51,000 query**
- Her query 2ms olsa → 102 saniye sadece SQL round-trip

P95 latency artar, DB CPU patlar, connection pool tüketilir, başka request bekler. **Outage senaryosu**.

---

### 2. N+1'in temel sebebi — LAZY association'un loop içinde dokunulması

Hibernate `@OneToMany(fetch = LAZY)` deyince, parent entity load edilirken child collection **proxy** olarak bırakılır. Collection'a ilk dokunduğunda (size(), iterator()) Hibernate bir SELECT atar.

Java tarafında **bir döngü** ile her parent'a dokunursan, her birinin collection'u ayrı SELECT'le yüklenir. **N+1'in mekaniği bu.**

**Aynı sorun `@ManyToOne` ile de var:**

```java
List<JournalLineJpaEntity> lines = lineRepo.findByOccurredAtAfter(yesterday);   // ① 1 query
for (var line : lines) {
    System.out.println(line.getAccount().getCurrency());   // ② her line için account SELECT
}
```

100 journal_line → 1 + 100 = 101 query.

---

### 3. N+1'i tespit etme

#### Yöntem A — SQL log

`application.yml`:

```yaml
spring:
  jpa:
    show-sql: false
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

Endpoint çağır, log'da **tekrarlayan SELECT'leri** gör. Eğer aynı SELECT 50 kere tekrarlanıyorsa N+1 alarmı.

**`show-sql` yerine `org.hibernate.SQL=DEBUG` tercih edilir** — sistemli logger, formatlanabilir, dinamik açıp kapanır.

#### Yöntem B — Hibernate slow query log

```yaml
spring:
  jpa:
    properties:
      hibernate:
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 10
```

10ms'ten uzun süren her query log'a düşer. N+1 durumunda her küçük query 1-2ms olduğu için bu görmez — ama yine de slow query monitoring olarak değerli.

#### Yöntem C — Hibernate Statistics

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

Endpoint çağırdıktan sonra:

```java
@Autowired EntityManagerFactory emf;

SessionFactory sf = emf.unwrap(SessionFactory.class);
Statistics stats = sf.getStatistics();
System.out.println("Query execution count: " + stats.getQueryExecutionCount());
System.out.println("Entity load count: " + stats.getEntityLoadCount());
System.out.println("Collection load count: " + stats.getCollectionLoadCount());
```

Aynı endpoint için query execution count > 10 ise alarm.

#### Yöntem D — `assertSelectCount` (test'te)

Test'te SQL sayısını assert et:

```java
@Test
@DataJpaTest
@Testcontainers
void shouldNotProduceNPlusOne() {
    // setup: 1 müşteri + 10 hesap + her hesaba 5 transaction
    
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();
    
    customerReportService.generateReport(ownerId);
    
    // 1 (accounts) + 1 (joined journal_lines) = 2 ideal
    assertThat(stats.getQueryExecutionCount()).isLessThanOrEqualTo(2);
}
```

Production'a girmeden önce **test'te yakalama** = en güvenli yol. CI'de aynı testler her PR'da koşar.

#### Yöntem E — Üretim profiler

Banking ortamlarında **APM araçları** (Datadog, New Relic, Dynatrace, Elastic APM). Endpoint başına SQL query count metric'i. N+1 anomalisi grafik olarak görünür.

---

### 4. Çözüm 1 — `JOIN FETCH` (JPQL)

İhtiyaç duyduğun association'u **explicit** olarak JOIN'le, single SQL ile yükle.

```java
interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    
    @Query("""
        SELECT DISTINCT a FROM AccountJpaEntity a 
        LEFT JOIN FETCH a.journalLines 
        WHERE a.ownerId = :ownerId
    """)
    List<AccountJpaEntity> findByOwnerIdWithLines(@Param("ownerId") UUID ownerId);
}
```

Üretilen SQL:

```sql
SELECT DISTINCT a.*, jl.* 
FROM accounts a 
LEFT OUTER JOIN journal_lines jl ON jl.account_id = a.id 
WHERE a.owner_id = ?;
```

**Tek query.** N+1 çözüldü.

**`DISTINCT` neden gerekli:**

Bir hesabın 5 journal_line'ı varsa, JOIN sonucu satır 5 kere tekrar eder. JPQL'in `DISTINCT` keyword'ü Hibernate'e *Java tarafında* duplicate'leri kaldırmasını söyler. SQL DISTINCT'i değildir (o farklı performans).

**Modern Hibernate:** `hibernate.query.passDistinctThrough = false` ile JPQL DISTINCT SQL'e geçmez (default). Hibernate 6'da artık otomatik bu davranış.

**Avantajlar:**

- Tek query
- Tam tipli entity döner
- Hibernate'in tüm feature'larını (cascade, dirty tracking) kullanır

**Dezavantajlar:**

- `MultipleBagFetchException`: Aynı parent'ın **iki ayrı collection**'ını JOIN FETCH yapmaya çalışırsan Hibernate kabul etmez. Cartesian product çok büyük olur.
- Pagination ile sorunlu: JOIN sonrası satır sayısı parent sayısı değil. `LIMIT 10` aslında 10 line, parent sayısı tahmin edilemez.
- 1-N JOIN'lerde data inflation (5 line = 5 satır, hepsi parent kolonlarını tekrar getirir → network/memory)

**Banking pratiği:** Tek collection fetch için ideal. İki collection için `@EntityGraph` veya iki ayrı query.

---

### 5. Çözüm 2 — `@EntityGraph`

JPA standardı (Hibernate-specific değil). Bir entity'nin **hangi association'larının load edileceğini** declare eder. `JOIN FETCH`'in deklaratif versiyonu.

#### Ad-hoc `@EntityGraph`

```java
interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    
    @EntityGraph(attributePaths = {"journalLines"})
    List<AccountJpaEntity> findByOwnerId(UUID ownerId);
}
```

Spring Data JPA, derived query veya `@Query` ile birleştirip otomatik JOIN ekler. JPQL yazmadan aynı sonuç.

**İki seviyeli (nested) attribute path:**

```java
@EntityGraph(attributePaths = {"journalLines", "journalLines.journalEntry"})
List<AccountJpaEntity> findByOwnerId(UUID ownerId);
```

Account → journalLines → journalEntry hepsi tek query'de.

#### Named `@EntityGraph`

Entity üzerinde tanımla:

```java
@Entity
@NamedEntityGraph(
    name = "Account.withLines",
    attributeNodes = @NamedAttributeNode("journalLines")
)
public class AccountJpaEntity {
    // ...
}

@EntityGraph("Account.withLines")
List<AccountJpaEntity> findByOwnerId(UUID ownerId);
```

Reuse için: aynı graph birden fazla repository method'unda kullanılır.

#### Sub-graphs (deep hierarchy)

```java
@NamedEntityGraph(
    name = "Account.deep",
    attributeNodes = {
        @NamedAttributeNode(value = "journalLines", subgraph = "lines-subgraph")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "lines-subgraph",
            attributeNodes = {
                @NamedAttributeNode("journalEntry"),
                @NamedAttributeNode("createdAt")
            }
        )
    }
)
```

**`@EntityGraph` vs `JOIN FETCH`:**

| Kriter | `JOIN FETCH` | `@EntityGraph` |
|---|---|---|
| Standart | JPA + Hibernate | JPA standard |
| Yazım | JPQL içinde | Deklaratif annotation |
| Reusability | Düşük | Yüksek (named) |
| Dinamik | Hayır | Evet (ad-hoc) |
| Kontrol | Granular (alias, WHERE) | Sadece attribute paths |

**Banking pratiği:** Reporting endpoint'lerinde `@EntityGraph` deklaratif olduğu için tercih edilir. Kompleks WHERE / dinamik JOIN'de `JOIN FETCH` daha güçlü.

---

### 6. Çözüm 3 — Batch Fetching

N+1 yerine "**N+(N/batch_size)**". 100 query yerine 100/25 = 4 query. Mükemmel değil ama dramatik gelişme.

#### Entity-level `@BatchSize`

```java
@Entity
public class AccountJpaEntity {
    
    @OneToMany(mappedBy = "account", fetch = FetchType.LAZY)
    @BatchSize(size = 25)
    private List<JournalLineJpaEntity> journalLines = new ArrayList<>();
}
```

Hibernate'in davranışı: İlk hesap için `journalLines` access edildiğinde, hâlâ proxy halinde olan diğer 24 hesabın `journalLines`'larını da aynı `IN` clause ile yükler:

```sql
SELECT * FROM journal_lines WHERE account_id IN (?, ?, ?, ..., 25 adet);
```

Diğer 25'i için ikinci batch query. Toplam: 1 (accounts) + 4 (4 batch) = 5 query, 100 yerine.

#### Global `hibernate.default_batch_fetch_size`

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
```

Tüm LAZY association'lara default 25 batch size uygulanır.

**Avantajlar:**

- Kod değişikliği yok (sadece config veya bir annotation)
- Pagination'la uyumlu (parent sayısı değişmiyor)
- `MultipleBagFetchException` yok

**Dezavantajlar:**

- `JOIN FETCH` kadar hızlı değil (multiple round-trip)
- 25 ideal mi? Bağlama göre — 10 veya 50 daha iyi olabilir
- Cache pattern karmaşıklaşır

**Banking pratiği:** Default 16-25 batch size + spesifik N+1 nokta için JOIN FETCH. İkili yaklaşım.

---

### 7. Çözüm 4 — DTO Projection (entity bypass)

En **performant** yaklaşım: entity hiç load etme, sadece ihtiyacın olan kolonları tek query ile DTO'ya çek.

```java
public record CustomerAccountReportDto(
    UUID accountId,
    String currency,
    BigDecimal balance,
    long lineCount,
    BigDecimal totalCredit,
    BigDecimal totalDebit
) {}

interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    
    @Query("""
        SELECT new com.mavibank.banking.account.adapter.out.persistence.CustomerAccountReportDto(
            a.id,
            a.currency,
            a.balanceAmount,
            COUNT(jl.id),
            COALESCE(SUM(CASE WHEN jl.direction = 'CREDIT' THEN jl.amount ELSE 0 END), 0),
            COALESCE(SUM(CASE WHEN jl.direction = 'DEBIT' THEN jl.amount ELSE 0 END), 0)
        )
        FROM AccountJpaEntity a 
        LEFT JOIN a.journalLines jl 
        WHERE a.ownerId = :ownerId 
        GROUP BY a.id, a.currency, a.balanceAmount
    """)
    List<CustomerAccountReportDto> findReportByOwner(@Param("ownerId") UUID ownerId);
}
```

**Tek query, tek round-trip, sadece istediğin kolonlar.**

**Avantajlar:**

- Maksimum performans
- Memory efficient (entity yok, lazy proxy yok)
- DB-level aggregation (COUNT, SUM)
- Hibernate dirty checking yok (entity yok)

**Dezavantajlar:**

- Domain bypass — sadece reporting için
- Read-only (DTO'ya UPDATE yapamazsın)
- Constructor expression sözdizimi uzun
- DTO domain'e karışmamalı (separate folder/package)

**Banking pratiği:** Reporting endpoint'lerinin **çoğunluğu** DTO projection olmalı. Domain mutasyonu varsa entity, sadece okuma varsa DTO.

---

### 8. `FetchType.LAZY` vs `EAGER` — derin bakış

#### LAZY

Default for `@OneToMany` ve `@ManyToMany`. **Önerilen.**

Parent load olur, child collection proxy'dir. İlk dokunmada SELECT.

Avantaj: gereksiz veri yok.
Dezavantaj: PC kapandıktan sonra dokunursan `LazyInitializationException`.

#### EAGER

Default for `@ManyToOne` ve `@OneToOne`. **Banking'de sakıncalı.**

Parent yüklenirken, association da yüklenir.

```java
@Entity
public class AccountJpaEntity {
    
    @ManyToOne(fetch = FetchType.EAGER)   // default
    @JoinColumn(name = "owner_id")
    private CustomerJpaEntity owner;
}
```

Her `findById(account)` çağrısı bir `JOIN customer` yapar. `findAll()` ise tüm customers'ı da yükler.

#### EAGER cascade fetch explosion

```java
@Entity
public class AccountJpaEntity {
    @ManyToOne(fetch = EAGER) CustomerJpaEntity owner;
}

@Entity
public class CustomerJpaEntity {
    @OneToMany(fetch = EAGER) List<AddressJpaEntity> addresses;
    @ManyToOne(fetch = EAGER) BranchJpaEntity branch;
}

@Entity
public class BranchJpaEntity {
    @ManyToOne(fetch = EAGER) RegionJpaEntity region;
}
```

`accountRepo.findById(id)` aslında: account + customer + addresses + branch + region — **5 ayrı JOIN veya 5 ayrı query**. Bir hesap detayı için tüm hiyerarşi yüklenir.

İlk geliştirmede masum, prod'da yıkıcı. Test edilemez (entity'ler arası referans grafiği patlar).

**Banking kuralı:** **Hiç bir association EAGER olmamalı.** Her `@ManyToOne`'ı explicit olarak `LAZY` yap:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "owner_id")
private CustomerJpaEntity owner;
```

İhtiyaç noktasında `JOIN FETCH` veya `@EntityGraph` ile yükle. Bu **deklaratif performans** sağlar.

---

### 9. Banking endpoint reprodüksiyon: `GET /accounts/{id}/transactions`

Şimdi gerçek bir banking endpoint'i — bir hesabın transaction geçmişi, paginated.

**Naif kod:**

```java
@RestController
@RequestMapping("/v1/accounts")
public class AccountController {
    
    @GetMapping("/{id}/transactions")
    public PageResponse<TransactionResponse> getTransactions(
        @PathVariable UUID id,
        @PageableDefault(size = 20) Pageable pageable
    ) {
        AccountJpaEntity account = jpaRepo.findById(id).orElseThrow();
        Page<JournalLineJpaEntity> lines = lineRepo.findByAccountId(id, pageable);
        
        return new PageResponse<>(
            lines.getContent().stream()
                .map(line -> new TransactionResponse(
                    line.getId(),
                    line.getDirection(),
                    line.getAmount(),
                    line.getJournalEntry().getOccurredAt(),       // ❶ N+1: JournalEntry LAZY
                    line.getJournalEntry().getDescription(),
                    line.getJournalEntry().getCounterparty().getName()   // ❷ Daha derin N+1
                ))
                .toList(),
            pageable.getPageNumber(),
            pageable.getSize(),
            lines.getTotalElements(),
            lines.getTotalPages()
        );
    }
}
```

20 transaction → 1 (accounts findById) + 1 (lines page) + 20 (journal_entry per line) + 20 (counterparty per entry) = **42 query**.

Toplam ile gelecek 1 page (20 kayıt) için 42 query. Toplam SUM 20 + 20 = COUNT için ek query. Pagination total count için 1 query.

P95 ölçümü: 42 query * 1ms = 42ms minimum. Ağ ve concurrency ile 100-200ms.

#### Çözüm karşılaştırması — 4 yöntem aynı endpoint üzerinde

**A — `JOIN FETCH`:**

```java
@Query("""
    SELECT jl FROM JournalLineJpaEntity jl 
    JOIN FETCH jl.journalEntry je 
    JOIN FETCH je.counterparty 
    WHERE jl.account.id = :accountId 
    ORDER BY je.occurredAt DESC
""")
Page<JournalLineJpaEntity> findByAccountIdWithEntry(@Param("accountId") UUID id, Pageable p);
```

SQL: 1 (lines + entry + counterparty JOIN) + 1 (count for page) = **2 query** (pagination olmasa 1).

**B — `@EntityGraph`:**

```java
@EntityGraph(attributePaths = {"journalEntry", "journalEntry.counterparty"})
@Query("SELECT jl FROM JournalLineJpaEntity jl WHERE jl.account.id = :accountId ORDER BY jl.journalEntry.occurredAt DESC")
Page<JournalLineJpaEntity> findByAccountId(@Param("accountId") UUID id, Pageable p);
```

SQL: 2 query (data + count).

**C — Batch fetching:**

```java
@Entity
public class JournalLineJpaEntity {
    @ManyToOne(fetch = LAZY)
    @BatchSize(size = 25)
    private JournalEntryJpaEntity journalEntry;
}

@Entity
public class JournalEntryJpaEntity {
    @ManyToOne(fetch = LAZY)
    @BatchSize(size = 25)
    private CounterpartyJpaEntity counterparty;
}
```

20 line için → 1 (lines) + 1 (journal_entries IN 20) + 1 (counterparties IN 20) + 1 (count) = **4 query**.

**D — DTO projection:**

```java
public record TransactionDto(
    UUID lineId,
    String direction,
    BigDecimal amount,
    Instant occurredAt,
    String description,
    String counterpartyName
) {}

@Query("""
    SELECT new com.mavibank.banking.account.adapter.out.persistence.TransactionDto(
        jl.id, jl.direction, jl.amount, 
        je.occurredAt, je.description, c.name
    )
    FROM JournalLineJpaEntity jl 
    JOIN jl.journalEntry je 
    JOIN je.counterparty c 
    WHERE jl.account.id = :accountId 
    ORDER BY je.occurredAt DESC
""")
Page<TransactionDto> findTransactionDtos(@Param("accountId") UUID id, Pageable p);
```

SQL: 2 query (data + count). Ama sadece ihtiyaç duyulan kolonlar SELECT'te.

**Karşılaştırma tablosu (20 kayıt page için):**

| Yöntem | Query sayısı | Yüklenen field | Bytes | Kompleksite |
|---|---|---|---|---|
| Naif | 42 | Tüm entity field'ları | Yüksek | Düşük |
| JOIN FETCH | 2 | Tüm entity field'ları | Orta-yüksek | Orta |
| @EntityGraph | 2 | Tüm entity field'ları | Orta-yüksek | Düşük |
| Batch fetch | 4 | Tüm entity field'ları | Orta | Çok düşük |
| DTO projection | 2 | Sadece DTO field'ları | Düşük | Orta |

**Banking pratiği:**

- Reporting endpoint = DTO projection
- Mutation endpoint öncesi (read for update) = JOIN FETCH veya @EntityGraph (entity gerekli)
- Genel default = `default_batch_fetch_size = 25`
- Explicit N+1 noktası = JOIN FETCH

---

### 10. `MultipleBagFetchException` — iki collection JOIN FETCH

```java
@Query("""
    SELECT a FROM AccountJpaEntity a 
    JOIN FETCH a.journalLines 
    JOIN FETCH a.cards 
    WHERE a.id = :id
""")
```

Hibernate fırlatır: `MultipleBagFetchException: cannot simultaneously fetch multiple bags`.

**Sebep:** İki `@OneToMany` JOIN'i Cartesian product yapar. 10 line * 5 card = 50 satır. Hibernate doğru parent grouping yapamaz.

**Çözüm 1:** Collection'ları `Set` yap (sıra önemli değilse):

```java
@OneToMany(mappedBy = "account", fetch = LAZY)
private Set<JournalLineJpaEntity> journalLines = new HashSet<>();

@OneToMany(mappedBy = "account", fetch = LAZY)
private Set<CardJpaEntity> cards = new HashSet<>();
```

Set Cartesian'ı tolere eder. Ama hâlâ data inflation var (50 satır geliyor, network yükü).

**Çözüm 2:** İki ayrı query:

```java
@EntityGraph(attributePaths = "journalLines")
Optional<AccountJpaEntity> findAccountWithLines(UUID id);

@EntityGraph(attributePaths = "cards")
Optional<AccountJpaEntity> findAccountWithCards(UUID id);
```

İkisini ayrı çağır, ana service'te birleştir. Daha temiz.

**Çözüm 3:** Hibernate 6'da `MultipleBagFetchException` kaldırıldı (varsayılan: Set olarak çalışır). Spring Boot 3+ ile bu hata azaldı.

---

### 11. OSIV (Open Session In View) — N+1'i gizleyen tuzak

`spring.jpa.open-in-view: true` (default!) ile Hibernate session HTTP request boyunca açık kalır. Service'ten dönen entity'ye Controller'da hâlâ dokunabilirsin → LAZY collection access OK → ama her dokunma yeni query → **N+1 controller seviyesinde**.

**Sorun:**

1. Controller view layer'da DB query'leri patlatır (boundary belirsiz)
2. TX kapalı, lock yok, isolation yok — anormal davranış
3. N+1 dev'in farkına varamadığı yerde olur
4. Connection pool tüketilir (her request session'ı tutar)

**Çözüm:** Production'da **kapat**:

```yaml
spring:
  jpa:
    open-in-view: false
```

Kapattıktan sonra Controller'da lazy collection dokunma → `LazyInitializationException`. Bu **iyi** — N+1'i fail-fast yapıyor. Service'te ihtiyacın olan her şeyi yükle veya DTO döndür.

**Banking pratiği:** `open-in-view: false` mutlaka. Bu Phase 1'in `application.yml`'inde olmalıydı, değilse şimdi düzelt.

---

### 12. Anti-pattern'ler ve sık hatalar

**Anti-pattern 1: `@ManyToOne(fetch = EAGER)` default'u**

`@ManyToOne` default'u EAGER'dir. Bunu **HER ZAMAN** explicit LAZY yap:

```java
@ManyToOne(fetch = FetchType.LAZY)   // ✅ explicit
@JoinColumn(...)
private CustomerJpaEntity owner;
```

ArchUnit veya custom checkstyle ile compile-time enforce et.

**Anti-pattern 2: LAZY collection'a domain'de erişim**

```java
public class Account {   // domain class
    public BigDecimal totalDebit() {
        return journalLines.stream()    // ❌ N+1 ihtimali
            .filter(l -> l.getDirection() == DEBIT)
            .map(JournalLine::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

Domain class JPA'yı bilmemeli. Aggregation logic application service'te ya da DB'de (`COALESCE(SUM)`) olmalı.

**Anti-pattern 3: `findAll().stream().map(JOIN FETCH yok)`**

```java
accountRepo.findAll().stream()
    .map(a -> a.getJournalLines().size())   // N+1!
    .toList();
```

İlk reaksiyonda `JOIN FETCH` ekle, ya da DTO projection.

**Anti-pattern 4: Controller'da entity döndürmek**

OSIV açık → Controller'da entity field'a erişim → lazy load → N+1. Phase 1'de DTO öğrenildi, yine de bir dev "shortcut" yapıyorsa burası kaynaktır.

**Anti-pattern 5: Pagination + JOIN FETCH (LIMIT 10 sürprizi)**

```java
@Query("SELECT a FROM AccountJpaEntity a JOIN FETCH a.journalLines")
Page<AccountJpaEntity> findAll(Pageable p);   // ❌
```

Hibernate uyarı verir: `firstResult/maxResults specified with collection fetch; applying in memory`. SQL'de LIMIT uygulanmaz, **tüm sonuç memory'e çekilir** sonra Java'da sayfalanır.

Çözüm: 2 query — ilk parent'ları LIMIT'le çek, sonra `WHERE id IN (...)` ile child'ları JOIN FETCH ile getir.

```java
@Query("SELECT a.id FROM AccountJpaEntity a WHERE a.ownerId = :ownerId")
Page<UUID> findIds(@Param("ownerId") UUID id, Pageable p);

@Query("SELECT a FROM AccountJpaEntity a LEFT JOIN FETCH a.journalLines WHERE a.id IN :ids")
List<AccountJpaEntity> findByIdsWithLines(@Param("ids") List<UUID> ids);
```

**Anti-pattern 6: `hibernate.enable_lazy_load_no_trans = true`**

Bu config "lazy load detached entity'lerde de çalışsın" der. Çalışır ama her access yeni session açar → N+1 saklanır. **Kullanma.**

**Anti-pattern 7: `@Fetch(FetchMode.SUBSELECT)` her yerde**

`SUBSELECT` mode child'ları tek query'de çeker (`WHERE parent_id IN (SELECT parent_id FROM ...)`). Faydalı ama her yerde kullanmak memory patlaması.

---

### 13. Best practices özeti

1. **Tüm `@ManyToOne` explicit LAZY.** Default EAGER'ı asla kabul etme.
2. **OSIV kapalı** (`open-in-view: false`).
3. **Repository test'lerinde `@Sql` setup + Hibernate statistics assertion.** N+1 build'i patlatsın.
4. **Default batch fetch size = 16 veya 25** as baseline.
5. **Reporting endpoint'lerinde DTO projection.**
6. **Mutation endpoint'lerinde JOIN FETCH veya @EntityGraph.**
7. **Pagination + collection JOIN FETCH yasak** (LIMIT in-memory). 2 query yöntemi.
8. **APM monitoring**: Endpoint başına query count, P95 latency.

---

## Önemli olabilecek araştırma kaynakları

- Vlad Mihalcea "N+1 query problem" article series
- Vlad Mihalcea "High-Performance Java Persistence" book — Fetching chapter
- Hibernate User Guide — Fetching strategies
- Thorben Janssen "How to detect and fix N+1 query problem"
- Spring Data JPA reference — Entity Graphs
- "Java Persistence with Hibernate" Christian Bauer
- Hypersistence Utils library (Hibernate types, query plan cache helpers)
- `hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS` property

---

## Mini task'ler

### Task 2.5.1 — N+1 reprodüksiyon (45 dk)

`CustomerReportService.generateReport(ownerId)` yaz. Naif versiyon — LAZY collection size'a loop'ta dokun.

Test setup: 1 owner + 10 hesap + her hesaba 5 journal_line.

```yaml
logging.level.org.hibernate.SQL: DEBUG
```

Endpoint çağır, log'da SQL sayısını **say**. 1 + 10 = 11 query görmelisin. Screenshot al, defterine yapıştır.

### Task 2.5.2 — `assertQueryCount` test'i (30 dk)

`StatisticsAssertions` helper class yaz:

```java
public class StatisticsAssertions {
    public static void assertQueryCount(SessionFactory sf, int expected) {
        Statistics stats = sf.getStatistics();
        long actual = stats.getQueryExecutionCount();
        assertThat(actual).as("Hibernate query count").isEqualTo(expected);
    }
}
```

`CustomerReportServiceTest`'te `generateReport`'tan önce `clear()`, sonra `assertQueryCount(sf, 11)`. Test geçsin.

### Task 2.5.3 — JOIN FETCH ile çöz (30 dk)

`findByOwnerIdWithLines` repository method'unu yaz (JPQL + DISTINCT). Service'te bunu kullan.

Aynı test'i çalıştır → `assertQueryCount(sf, 1)` veya 2 olmalı. Karşılaştır:
- Önceki: 11 query
- Sonraki: 1-2 query
- Süre farkı: lokal DB'de muhtemelen 5-10ms → 1ms.

### Task 2.5.4 — `@EntityGraph` versiyonu (30 dk)

Aynı method'u `@EntityGraph(attributePaths = {"journalLines"})` ile yaz. JOIN FETCH versiyonu ile aynı sonucu vermeli.

Defter notu: Hangisini production'da seçersin, neden?

### Task 2.5.5 — Batch fetching ile çöz (30 dk)

`@BatchSize(size = 25)` ekle `Account.journalLines`'a. Naif kodu olduğu gibi bırak (loop'ta size() çağrısı).

Aynı test'i çalıştır → query count beklenen: 1 + 1 = 2 (10 hesap < 25 batch size).

Sonra setup'ı **50 hesap** yap. Query count: 1 + 2 = 3 (50 / 25 = 2 batch).

`hibernate.default_batch_fetch_size: 25` global config ile aynı sonucu elde et — `@BatchSize` annotation'ı kaldır.

### Task 2.5.6 — DTO projection ile çöz (45 dk)

`CustomerAccountReportDto` record yaz. JPQL constructor expression ile sadece (id, currency, balance, COUNT, SUM) çek.

Query count = 1.

Bytes karşılaştır:
- Naif: tüm Account + tüm JournalLine field'ları → ~20KB
- DTO: sadece 6 field → ~2KB

`/actuator/metrics/jvm.memory.used` ile JVM heap kullanımını karşılaştır (büyük dataset'te).

### Task 2.5.7 — Banking endpoint reprodüksiyon (1 saat)

`GET /v1/accounts/{id}/transactions?page=0&size=20` endpoint'ini ekle.

Naif versiyon yaz (Counterparty da var: JournalEntry → Counterparty `@ManyToOne` LAZY).

Test: 20 transaction page → query count yaklaşık 42 (1 + 20 + 20 + 1 count).

Sonra 4 yöntem ile **ayrı ayrı** düzelt:
- (a) JOIN FETCH versiyonu → 2 query
- (b) @EntityGraph versiyonu → 2 query
- (c) Batch fetching (size 25) → 4 query
- (d) DTO projection → 2 query

Her birinin SQL count'unu test ile assert et. `docs/n-plus-one-comparison.md`'a tablo yapıştır.

### Task 2.5.8 — OSIV kapatma deneyi (30 dk)

`spring.jpa.open-in-view: false` ekle (zaten varsa doğrula).

Naif endpoint'i tekrar çalıştır. `LazyInitializationException` bekle. Hangi noktada patlıyor?

Çözüm: Service'i `@Transactional(readOnly = true)` ile sarma, ihtiyacın olan tüm field'ları service içinde load et veya DTO döndür.

### Task 2.5.9 — `MultipleBagFetchException` reprodüksiyon (30 dk)

`Account` entity'sine `@OneToMany cards` ekle (Card entity yarat).

`SELECT a FROM AccountJpaEntity a JOIN FETCH a.journalLines JOIN FETCH a.cards WHERE a.id = :id` query'sini yaz.

Test çalıştır → `MultipleBagFetchException` (Hibernate 5.x) veya başarı (Hibernate 6.x).

Çözüm: Set'e dönüştür veya 2 ayrı query.

### Task 2.5.10 — Default batch fetch size global config (15 dk)

`application.yml`:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 16
```

Tüm `@OneToMany` ve `@ManyToOne` LAZY association'larında 16'lık batch otomatik aktif. Tüm endpoint'leri tekrar test et — N+1 azalır, ek değişiklik yapmadan.

---

## Test yazma rehberi

### Test 2.5.1 — Query count assertion (`@DataJpaTest`)

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class NPlusOneDetectionTest {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired EntityManagerFactory emf;
    @Autowired AccountJpaRepository accountRepo;
    @Autowired CustomerReportService service;
    
    @BeforeEach
    void setup() {
        UUID owner = UUID.randomUUID();
        for (int i = 0; i < 10; i++) {
            AccountJpaEntity a = createAccount(owner);
            for (int j = 0; j < 5; j++) {
                createJournalLine(a);
            }
        }
    }
    
    @Test
    void naiveCodeShouldShowNPlusOne() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();
        stats.clear();
        
        service.generateReportNaive(owner);
        
        // 1 (parent) + 10 (children) = 11
        assertThat(stats.getQueryExecutionCount()).isGreaterThanOrEqualTo(11);
    }
    
    @Test
    void joinFetchShouldEliminateNPlusOne() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();
        stats.clear();
        
        service.generateReportWithJoinFetch(owner);
        
        // 1 query (JOIN ile)
        assertThat(stats.getQueryExecutionCount()).isLessThanOrEqualTo(2);
    }
    
    @Test
    void dtoProjectionShouldUseSingleQuery() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();
        stats.clear();
        
        service.generateReportDto(owner);
        
        assertThat(stats.getQueryExecutionCount()).isEqualTo(1);
        // Entity load count 0 olmalı (DTO sadece)
        assertThat(stats.getEntityLoadCount()).isZero();
    }
}
```

### Test 2.5.2 — SQL count'ı endpoint seviyesinde

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class AccountTransactionsEndpointTest {
    
    @Autowired TestRestTemplate rest;
    @Autowired EntityManagerFactory emf;
    
    @Test
    void getTransactionsShouldUseAtMostTwoQueries() {
        UUID accountId = setupAccountWith20Transactions();
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        sf.getStatistics().clear();
        
        var response = rest.getForEntity(
            "/v1/accounts/{id}/transactions?page=0&size=20", 
            String.class, accountId
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(sf.getStatistics().getQueryExecutionCount()).isLessThanOrEqualTo(3);
    }
}
```

### Test 2.5.3 — `LazyInitializationException` ile OSIV kapalı doğrulama

```java
@Test
void osIvDisabledShouldFailOnLazyAccessOutsideTx() {
    Account account = accountRepository.findById(id).orElseThrow();
    // Service dışına çıktık, TX kapalı
    
    assertThatThrownBy(() -> account.getJournalLines().size())
        .isInstanceOf(LazyInitializationException.class);
}
```

Bu test başarılıysa `open-in-view: false` aktif demektir. OSIV açıksa exception fırlamaz — test fail.

---

## Claude-verify prompt

```
Aşağıda banking domain'inde JPA fetching kodum var. N+1 problem'ine yönelik şu kriterlerle 
değerlendir, PASS / FAIL / EKSIK işaretle, KOD YAZMA:

1. EAGER kullanımı:
   - Hiç bir `@ManyToOne` default EAGER mı kalmış? (Olmamalı, hepsi explicit LAZY)
   - Hiç bir `@OneToOne` EAGER mı?
   - Bunlar code review checklist'ine eklenmiş mi (ADR veya doc)?

2. OSIV:
   - `spring.jpa.open-in-view: false` mü?
   - Service'lerden Controller'a entity döndürülüyor mu (DTO mu)?
   - LazyInitializationException test ile doğrulanmış mı?

3. N+1 tespiti:
   - SQL log aktif (DEBUG level)?
   - Hibernate generate_statistics açık mı (en azından test'lerde)?
   - Test'lerde `assertQueryCount` benzeri assertion var mı?

4. Çözüm yöntemleri:
   - Reporting endpoint'leri DTO projection mu kullanıyor?
   - Mutation öncesi read'lerde JOIN FETCH veya @EntityGraph mi?
   - `default_batch_fetch_size` config'de set mi?
   - Spesifik N+1 noktalarında JOIN FETCH yazılmış mı?

5. JOIN FETCH:
   - JPQL'de DISTINCT keyword'ü var mı (gerekli olduğunda)?
   - Pagination ile JOIN FETCH kullanılmış mı (LIMIT in-memory tuzağı)? — eğer kullanıldıysa 
     2-query pattern (IDs + IN clause) tercih edilmiş mi?

6. @EntityGraph:
   - Named graph veya ad-hoc seçimi mantıklı mı (reuse durumu)?
   - Nested attribute path doğru yazılmış mı?
   - Sub-graphs gerekiyorsa kullanılmış mı?

7. Batch fetching:
   - Banking-grade default size (16 veya 25) mü?
   - Hot path'te @BatchSize override edilmiş mi (özel ihtiyaç varsa)?

8. DTO projection:
   - JPQL constructor expression doğru paket yoluyla mı?
   - Record kullanılmış mı (immutable)?
   - DTO'lar persistence package'da, domain'e karışmamış mı?

9. MultipleBagFetchException:
   - İki @OneToMany JOIN FETCH durumu kontrol edilmiş mi?
   - Çözüm: Set'e çevirme mi yoksa 2 ayrı query mi (banking'de hangi daha uygun)?

10. Endpoint-spesifik:
    - GET /v1/accounts/{id}/transactions endpoint'i hangi yöntem kullanıyor?
    - Endpoint query count'u test ile garanti altında mı?
    - Pagination 20 kayıt için maksimum 3 query mi (data + count + ek)?

11. Anti-pattern:
    - Domain class'ta LAZY collection iterasyonu var mı? (Olmamalı)
    - `hibernate.enable_lazy_load_no_trans` açık mı? (Kapalı olmalı)
    - Pagination + JOIN FETCH (collection) memory in-memory yapıyor mu?
    - `findAll()` Pageable'sız çağrılıyor mu (Phase 2.2 tekrar)?

12. Banking-grade kalite:
    - APM (Datadog/New Relic) ile endpoint query count monitor ediliyor mu?
    - Slow query log etkin mi (production'da log_min_duration_statement)?
    - N+1 regresyonu için CI'de test koşuyor mu?

Her madde için PASS / FAIL / EKSIK ve kanıt. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] N+1 problemini SQL log + Hibernate statistics ile reprodüksiyon ettim
- [ ] `assertQueryCount` helper'ı yazdım, test'lerimde kullanıyorum
- [ ] JOIN FETCH ile 11 query → 1-2 query indirgemesini gördüm
- [ ] `@EntityGraph` ile aynı sonucu elde ettim, ad-hoc vs named farkını biliyorum
- [ ] Default batch fetch size 16-25 olarak yapılandırıldı
- [ ] DTO projection ile reporting endpoint'i sadece istenen field'lar üzerinden
- [ ] `GET /accounts/{id}/transactions` endpoint'inde 4 yöntemi karşılaştırdım, sonuçlar `docs/`'da
- [ ] `MultipleBagFetchException` reprodüksiyon ettim ve çözüm yöntemini biliyorum
- [ ] `open-in-view: false` aktif, `LazyInitializationException` test'i ile doğrulanmış
- [ ] Tüm `@ManyToOne` explicit LAZY (code review yaptım)
- [ ] Anti-pattern listesi rahat — özellikle "pagination + collection JOIN FETCH", "EAGER cascade"

Hepsi onaylı → Topic 2.6'ya geç → [06-connection-pool/](../06-connection-pool/index.md)

---

## Defter notları

1. "N+1 problem temel sebebi: ____ (LAZY association'a loop içinde dokunma)."
2. "N+1'i tespit etmenin 3 yolu: ____, ____, ____."
3. "JOIN FETCH ile @EntityGraph farkı: ____. Hangisini ne zaman: ____."
4. "Pagination + collection JOIN FETCH neden problemli: ____. Çözüm: ____."
5. "Batch fetching mekaniği: ____ (IN clause). Default `default_batch_fetch_size` değerim: ____."
6. "DTO projection ne zaman tercih: ____. Entity'e göre avantajları: ____."
7. "OSIV açık olunca neden gizli N+1 olur: ____. Production'da OSIV: ____."
8. "FetchType.LAZY vs EAGER default farkı (associtaion type'a göre): @ManyToOne ____, @OneToMany ____."
9. "MultipleBagFetchException sebebi: ____. 2 çözüm: ____ ve ____."
10. "Banking-grade N+1 koruması (test + monitoring) için checklist: ____."
