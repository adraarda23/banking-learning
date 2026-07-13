# Topic 2.2 — Spring Data JPA & Repositories

## Hedef

Spring Data JPA'nın `JpaRepository` üzerinden sağladığı abstraction'ları **iç mekanikleriyle** öğrenmek. Derived query'lerin sınırlarını, JPQL ve native SQL'in ne zaman gerektiğini, Specification'ın dinamik query için kullanımını, projection türlerini (interface, class, dynamic), pagination ve sort'u, ve auditing pattern'i banking örnekleriyle kavramak.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~4.5 saat

## Önbilgi

- Topic 2.1 (JPA Fundamentals) bitti
- EntityManager, persistence context, entity state'leri biliyorsun
- Basit `JpaRepository.findById`, `save` çağrıları yaptın

---

## Kavramlar

### 1. Spring Data JPA mimarisi

Spring Data JPA = JPA üzerine **opinion'lı abstraction**. Sana üç hediye verir:

1. **Repository interface'leri** — sen interface yazıyorsun, Spring implement ediyor (proxy)
2. **Derived query'ler** — method ismi okuyarak SQL üretiyor (`findByOwnerId`)
3. **Pagination & Sorting** — built-in

**Hierarchy:**

```
Repository<T, ID>                 ← marker
  └── CrudRepository<T, ID>       ← save, findById, deleteAll, count
       └── PagingAndSortingRepository<T, ID>   ← Pageable, Sort
            └── JpaRepository<T, ID>           ← flush, saveAll, deleteAllInBatch
```

`JpaRepository` JPA-specific en zengini. Genelde bunu extend ederiz.

**Banking pratiği:** Domain'in adapter katmanında `JpaRepository`'leri **package-private** tut:

```java
package com.mavibank.banking.account.adapter.out.persistence;

interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    // package-private, dışarıya sızmasın
}
```

Domain port (`AccountRepository`) public, JPA repository implementation detayı.

### 2. Derived query method'lar

Method ismi konvansiyonuna uyarsa Spring SQL üretir:

```java
interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    
    // SELECT * FROM accounts WHERE owner_id = ?
    List<AccountJpaEntity> findByOwnerId(UUID ownerId);
    
    // SELECT * FROM accounts WHERE owner_id = ? AND currency = ?
    List<AccountJpaEntity> findByOwnerIdAndCurrency(UUID ownerId, String currency);
    
    // SELECT * FROM accounts WHERE owner_id = ? OR currency = ?
    List<AccountJpaEntity> findByOwnerIdOrCurrency(UUID ownerId, String currency);
    
    // SELECT COUNT(*) FROM accounts WHERE status = ?
    long countByStatus(AccountStatusEntity status);
    
    // SELECT EXISTS(... WHERE owner_id = ?)
    boolean existsByOwnerId(UUID ownerId);
    
    // SELECT * FROM accounts WHERE balance_amount > ?
    List<AccountJpaEntity> findByBalanceAmountGreaterThan(BigDecimal threshold);
    
    // SELECT * FROM accounts WHERE balance_amount BETWEEN ? AND ?
    List<AccountJpaEntity> findByBalanceAmountBetween(BigDecimal min, BigDecimal max);
    
    // SELECT * FROM accounts WHERE currency IN (?, ?, ?)
    List<AccountJpaEntity> findByCurrencyIn(Collection<String> currencies);
    
    // SELECT * FROM accounts WHERE opened_at > ?
    List<AccountJpaEntity> findByOpenedAtAfter(Instant date);
    
    // SELECT * FROM accounts WHERE status = ? ORDER BY opened_at DESC
    List<AccountJpaEntity> findByStatusOrderByOpenedAtDesc(AccountStatusEntity status);
    
    // SELECT * FROM accounts WHERE status = ? LIMIT 10
    List<AccountJpaEntity> findTop10ByStatusOrderByOpenedAtDesc(AccountStatusEntity status);
    
    // SELECT * FROM accounts WHERE customer_reference IS NULL
    List<AccountJpaEntity> findByCustomerReferenceIsNull();
    
    // DELETE FROM accounts WHERE status = ?
    @Modifying
    @Transactional
    long deleteByStatus(AccountStatusEntity status);   // Banking'de fiziksel silme genelde anti-pattern
}
```

**Anahtar kelimeler:**

| Keyword | SQL eşdeğeri |
|---|---|
| `findBy`, `getBy`, `queryBy`, `readBy` | SELECT |
| `countBy` | SELECT COUNT |
| `existsBy` | SELECT EXISTS |
| `deleteBy` (with `@Modifying`) | DELETE |
| `And`, `Or` | AND, OR |
| `Is`, `Equals` | = |
| `GreaterThan`, `LessThan` | >, < |
| `GreaterThanEqual` | >= |
| `Between` | BETWEEN |
| `In`, `NotIn` | IN |
| `Like`, `NotLike` | LIKE |
| `IsNull`, `IsNotNull` | IS NULL |
| `OrderBy<Field>Asc/Desc` | ORDER BY |
| `Top<n>`, `First<n>` | LIMIT n |
| `Distinct` | SELECT DISTINCT |

**Sınırları:**

- Method ismi çok uzun olunca (3-4 koşuldan fazla) okunmaz
- Dinamik filtre (kullanıcının seçtiği alanlar) → Specification veya Criteria
- Aggregation, complex JOIN → `@Query` veya native

**Banking pratiği:** 2-3 koşula kadar derived query OK. Daha fazlası → `@Query`.

### 3. `@Query` — JPQL ve native SQL

#### JPQL

```java
@Query("""
    SELECT a FROM AccountJpaEntity a 
    WHERE a.ownerId = :ownerId 
    AND a.status = 'ACTIVE'
    AND a.balanceAmount > :minBalance
    ORDER BY a.openedAt DESC
""")
List<AccountJpaEntity> findActiveAccountsAboveBalance(
    @Param("ownerId") UUID ownerId,
    @Param("minBalance") BigDecimal minBalance
);
```

**JPQL ≠ SQL:**
- Table değil, **entity class ismi**
- Kolon değil, **field ismi**
- JOIN entity ilişkileri üzerinden (`a.journalLines`)
- Veritabanı-bağımsız

**Named parameters (`:name`) tercih edilir, positional (`?1`) anti-pattern.**

#### Native SQL

```java
@Query(value = """
    SELECT * FROM accounts 
    WHERE owner_id = :ownerId 
    AND balance_amount > :minBalance
""", nativeQuery = true)
List<AccountJpaEntity> findActiveNative(
    @Param("ownerId") UUID ownerId,
    @Param("minBalance") BigDecimal minBalance
);
```

Ne zaman native?
- DB-specific feature (PostgreSQL window functions, MERGE, JSON operators)
- Performance — JPQL'in üretemediği optimum SQL
- Complex query JPQL'le anlatılamayacak kadar

**Tuzak:** Native query DB değişikliğine kapalı — PostgreSQL'den Oracle'a geçersen rewrite.

#### `@Modifying` — UPDATE/DELETE

```java
@Modifying
@Transactional
@Query("UPDATE AccountJpaEntity a SET a.status = :status WHERE a.id = :id")
int updateStatus(@Param("id") UUID id, @Param("status") AccountStatusEntity status);
```

**Önemli detaylar:**
- `@Modifying` UPDATE/DELETE için zorunlu
- `@Transactional` ile çağrılmazsa hata
- Dönen `int` = etkilenen satır sayısı
- **PC bypass eder** — managed entity'lerin state'i güncellenmez. `clearAutomatically = true` veya `em.refresh()`

**Banking pratiği:** Bulk update'lerde dirty checking yetmez. `@Modifying` ile direkt SQL hızlı. Ama PC tutarsızlığı dikkat.

### 4. Pagination — `Pageable` ve `Page`

```java
interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    Page<AccountJpaEntity> findByStatus(AccountStatusEntity status, Pageable pageable);
}
```

Controller:

```java
@GetMapping
PageResponse<AccountResponse> list(
    @PageableDefault(size = 20, sort = "openedAt", direction = Sort.Direction.DESC)
    Pageable pageable
) {
    Page<AccountJpaEntity> page = repository.findByStatus(ACTIVE, pageable);
    return new PageResponse<>(
        page.getContent().stream().map(mapper::toResponse).toList(),
        page.getNumber(),
        page.getSize(),
        page.getTotalElements(),
        page.getTotalPages()
    );
}
```

**`Page` vs `Slice` vs `List`:**

- `Page<T>` — toplam sayım yapar (`SELECT COUNT(*)` ekstra query). Toplam page bilgisi var.
- `Slice<T>` — sayım yapmaz. Sadece "sonraki sayfa var mı" bilgisi (`hasNext()`). Performansı daha iyi.
- `List<T>` — sayfalama yapar ama metadata yok.

**Banking pratiği:** Transaction history (milyonlarca kayıt) için `Slice`. Hesap listesi (az) için `Page`. Mutlaka `List<T>` ile döndürdüğünde **limit'i clamp et** (max 1000) — yoksa client'ı OOM yapar.

#### Cursor-based pagination (alternatif)

`OFFSET 1000000 LIMIT 20` çok yavaş — DB 1M kayıt skip eder. Banking'de transaction history için **cursor-based** daha iyi:

```java
@Query("""
    SELECT t FROM TransactionJpaEntity t 
    WHERE t.accountId = :accountId 
    AND (t.occurredAt < :cursorTime OR (t.occurredAt = :cursorTime AND t.id < :cursorId))
    ORDER BY t.occurredAt DESC, t.id DESC
    LIMIT :limit
""")
List<TransactionJpaEntity> findByAccountAfterCursor(
    @Param("accountId") UUID accountId,
    @Param("cursorTime") Instant cursorTime,
    @Param("cursorId") UUID cursorId,
    @Param("limit") int limit
);
```

Cursor = (last seen `occurredAt`, last seen `id`). Phase 2'de detaylandırmayacağız, ama bil.

### 5. `Sort` — dinamik sıralama

```java
Page<AccountJpaEntity> page = repository.findAll(
    PageRequest.of(0, 20, Sort.by("openedAt").descending().and(Sort.by("currency").ascending()))
);
```

URL'den:

```
GET /v1/accounts?sort=openedAt,desc&sort=currency,asc&page=0&size=20
```

Spring otomatik parse eder. **Tehlike:** Kullanıcının istediği herhangi field'da sort yapma — sensitive bilgi sızdırabilir veya performans kaybı. **Whitelist** yap:

```java
private static final Set<String> ALLOWED_SORT_FIELDS = Set.of("openedAt", "balanceAmount", "currency");

private Sort sanitize(Sort sort) {
    return Sort.by(sort.stream()
        .filter(o -> ALLOWED_SORT_FIELDS.contains(o.getProperty()))
        .toList());
}
```

### 6. Specifications — dinamik query

Kullanıcı 5 farklı filtre seçebilir (currency, status, balance min, balance max, owner). Hangisini gönderir bilmiyoruz. Derived query'le anlatılamaz, JPQL'de string concat anti-pattern.

**Specification pattern:**

```java
interface AccountJpaRepository extends 
    JpaRepository<AccountJpaEntity, UUID>,
    JpaSpecificationExecutor<AccountJpaEntity> { }   // ← ek interface
```

Specification builder:

```java
public class AccountSpecifications {
    
    public static Specification<AccountJpaEntity> hasOwnerId(UUID ownerId) {
        return (root, query, cb) -> 
            ownerId == null ? null : cb.equal(root.get("ownerId"), ownerId);
    }
    
    public static Specification<AccountJpaEntity> hasCurrency(String currency) {
        return (root, query, cb) -> 
            currency == null ? null : cb.equal(root.get("currency"), currency);
    }
    
    public static Specification<AccountJpaEntity> hasStatus(AccountStatusEntity status) {
        return (root, query, cb) -> 
            status == null ? null : cb.equal(root.get("status"), status);
    }
    
    public static Specification<AccountJpaEntity> balanceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min != null && max != null) return cb.between(root.get("balanceAmount"), min, max);
            if (min != null) return cb.greaterThanOrEqualTo(root.get("balanceAmount"), min);
            if (max != null) return cb.lessThanOrEqualTo(root.get("balanceAmount"), max);
            return null;
        };
    }
}
```

Kullanım:

```java
public Page<Account> search(AccountFilter filter, Pageable pageable) {
    Specification<AccountJpaEntity> spec = Specification
        .where(AccountSpecifications.hasOwnerId(filter.ownerId()))
        .and(AccountSpecifications.hasCurrency(filter.currency()))
        .and(AccountSpecifications.hasStatus(filter.status()))
        .and(AccountSpecifications.balanceBetween(filter.minBalance(), filter.maxBalance()));
    
    return repository.findAll(spec, pageable);
}
```

Null'lar otomatik ignore edilir. Type-safe (refactoring güvenli).

**Banking pratiği:** Search/filter endpoint'leri için ideal. Reporting filtreleri, admin panelleri.

### 7. Criteria API — düşük seviyeli

Specification'ın altında **JPA Criteria API** var. Hibernate-specific değil, JPA standard.

```java
public List<AccountJpaEntity> findAccountsCriteria(UUID ownerId, BigDecimal minBalance) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<AccountJpaEntity> cq = cb.createQuery(AccountJpaEntity.class);
    Root<AccountJpaEntity> root = cq.from(AccountJpaEntity.class);
    
    List<Predicate> predicates = new ArrayList<>();
    if (ownerId != null) predicates.add(cb.equal(root.get("ownerId"), ownerId));
    if (minBalance != null) predicates.add(cb.greaterThanOrEqualTo(root.get("balanceAmount"), minBalance));
    
    cq.where(predicates.toArray(new Predicate[0]));
    cq.orderBy(cb.desc(root.get("openedAt")));
    
    return em.createQuery(cq).getResultList();
}
```

**Specification > Criteria** çoğu zaman. Criteria daha verbose, daha düşük seviye. Yine de bil — JPA spec'inin parçası.

### 8. Projection türleri

Reporting endpoint'inde tüm entity field'larını çekmek israf. **DTO projection** önerilir.

#### Interface-based projection

```java
public interface AccountSummary {
    UUID getId();
    String getCurrency();
    BigDecimal getBalanceAmount();
}

interface AccountJpaRepository extends JpaRepository<AccountJpaEntity, UUID> {
    List<AccountSummary> findByOwnerId(UUID ownerId);
}
```

Spring otomatik proxy üretir. **Sadece istenen kolonları SELECT eder** — performans.

**Nested projection:**

```java
public interface AccountWithOwnerSummary {
    UUID getId();
    BigDecimal getBalanceAmount();
    OwnerSummary getOwner();
    
    interface OwnerSummary {
        String getName();
        String getEmail();
    }
}
```

#### Class-based (DTO) projection

```java
public record AccountSummaryDto(
    UUID id,
    String currency,
    BigDecimal balanceAmount
) {}

@Query("""
    SELECT new com.mavibank.banking.account.adapter.out.persistence.AccountSummaryDto(
        a.id, a.currency, a.balanceAmount
    )
    FROM AccountJpaEntity a
    WHERE a.ownerId = :ownerId
""")
List<AccountSummaryDto> findSummariesByOwner(@Param("ownerId") UUID ownerId);
```

JPQL constructor expression. **Performansı interface-based ile aynı**, tip-güvenli.

**Banking pratiği:** Record DTO + constructor expression — net, type-safe.

#### Dynamic projection

```java
<T> List<T> findByOwnerId(UUID ownerId, Class<T> type);

// Çağrı:
List<AccountSummary> summaries = repo.findByOwnerId(ownerId, AccountSummary.class);
List<AccountJpaEntity> full = repo.findByOwnerId(ownerId, AccountJpaEntity.class);
```

Aynı method farklı projection döner. Aşırı esnek ama nadiren gerekli.

### 9. Auditing — kim ne zaman değiştirdi

Banking'de **her satırın audit'i** gerekir:
- Kim oluşturdu?
- Ne zaman oluşturuldu?
- Son güncelleyen kim?
- Son güncelleme zamanı?

Spring Data JPA built-in destek:

`@EnableJpaAuditing` ekle:

```java
@SpringBootApplication
@EnableJpaAuditing
public class CoreBankingApplication { }
```

Entity:

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "updated_by", nullable = false, length = 100)
    private String updatedBy;
    
    // getters
}

@Entity
class AccountJpaEntity extends AuditableEntity {
    @Id UUID id;
    // ...
}
```

`AuditorAware` provider:

```java
@Component
class SecurityAuditorAware implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .or(() -> Optional.of("system"));
    }
}
```

Bunu activate et:

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "securityAuditorAware")
class AuditingConfig { }
```

Sonuç: Her `save` çağrısında `createdAt`, `updatedAt`, `createdBy`, `updatedBy` otomatik dolar.

**Banking pratiği:** Tüm entity'ler `AuditableEntity` extend etsin. Audit kolonları DB'de NOT NULL.

### 10. Soft delete pattern

Banking'de **fiziksel silme yok**. Hesap kapanır, kayıt kalır.

```java
@Entity
class AccountJpaEntity {
    @Id UUID id;
    
    @Column(name = "deleted_at")
    private Instant deletedAt;   // null → aktif, dolu → silinmiş
    
    // getters
}
```

Hibernate `@SQLDelete` ve `@Where` ile transparent soft delete:

```java
@Entity
@SQLDelete(sql = "UPDATE accounts SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
class AccountJpaEntity {
    // ...
}
```

`repo.delete(account)` artık UPDATE çıkarır, gerçek DELETE yok. Default query'ler `deleted_at IS NULL` filter ekler.

**Tuzak:** `@Where` Hibernate-specific. JPA standard'da yok. Spring Boot 3 + Hibernate 6 ile `@SoftDelete` (deneysel) geliyor.

**Banking pratiği:** Account/Customer için soft delete ideal. Audit kayıtları için fiziksel delete ASLA yok (regülatör isteği).

### 11. Custom repository implementation

Bazen Spring Data'nın imkânsız sağladığı şey gerekir (çok complex query, custom JDBC). Custom impl pattern:

```java
// 1. Public repository
interface AccountJpaRepository extends 
    JpaRepository<AccountJpaEntity, UUID>,
    AccountJpaRepositoryCustom { }

// 2. Custom interface
interface AccountJpaRepositoryCustom {
    Page<AccountJpaEntity> findByComplexCriteria(SearchCriteria criteria);
}

// 3. Implementation — naming convention: <RepoName>Impl
class AccountJpaRepositoryImpl implements AccountJpaRepositoryCustom {
    
    @PersistenceContext
    private EntityManager em;
    
    @Override
    public Page<AccountJpaEntity> findByComplexCriteria(SearchCriteria criteria) {
        // Custom logic with em
        ...
    }
}
```

Spring otomatik birleştirir. Senin kullanım açından tek `AccountJpaRepository` var.

### 12. `@Transactional(readOnly = true)`

Sadece okuyan service method'larına `readOnly = true` koy:

```java
@Service
@Transactional(readOnly = true)   // class-level default read-only
class AccountReportingService {
    
    public Account getById(AccountId id) { ... }
    
    @Transactional   // method-level override — write için
    public void updateStatus(AccountId id, Status status) { ... }
}
```

**Faydaları:**
- Hibernate dirty checking yapmaz (snapshot tutmaz) → performans
- DB driver bazı durumlarda read-only optimization yapar (Oracle bunu hisseder)
- Read-only replica'ya yönlendirme yapılabilir (Phase 9)

**Banking pratiği:** Reporting service'lerinin tümü `readOnly = true`. Write service'ler default.

### 13. `findById` vs `getReferenceById` (eski `getOne`)

```java
Optional<AccountJpaEntity> opt = repo.findById(id);       // SELECT fırlatır
AccountJpaEntity proxy = repo.getReferenceById(id);       // Lazy proxy döner
```

`getReferenceById`:
- DB'ye gitmez (henüz)
- Sadece foreign key kurmak için kullan: `transaction.setAccount(repo.getReferenceById(id))`
- Field'a erişirsen lazy load → DB call

**Banking örneği:**

```java
public void createJournalLine(UUID journalId, UUID accountId, BigDecimal amount) {
    JournalLineJpaEntity line = new JournalLineJpaEntity();
    line.setJournalEntry(journalRepo.getReferenceById(journalId));   // SELECT YOK
    line.setAccount(accountRepo.getReferenceById(accountId));         // SELECT YOK
    line.setAmount(amount);
    lineRepo.save(line);
}
```

3 SELECT yerine sadece 1 INSERT. Performans kazancı.

**Tuzak:** `getReferenceById` ID DB'de yoksa **save zamanı** patlar (foreign key constraint). `findById` ile var olduğunu doğrulamak istersen onu kullan.

### 14. Entity callbacks — `@PrePersist`, `@PreUpdate`

```java
@Entity
class AccountJpaEntity {
    @Id UUID id;
    
    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
        if (openedAt == null) openedAt = Instant.now();
    }
    
    @PreUpdate
    void preUpdate() {
        // örnek: status değişikliğinde audit log
    }
}
```

**Mevcut callback'ler:**

- `@PrePersist`, `@PostPersist`
- `@PreUpdate`, `@PostUpdate`
- `@PreRemove`, `@PostRemove`
- `@PostLoad`

**Banking tuzağı:** Callback içinde başka entity manage etme, DI servisi çağırma. Callback minimal kalsın (basit default'lar set, validation). Business logic burada **OLMAMALI**.

### 15. Banking anti-pattern'leri

**Anti-pattern 1: Repository'i Controller'a inject etme**

```java
@RestController
class AccountController {
    private final AccountJpaRepository repo;   // ❌ persistence sızdı
    
    @GetMapping("/{id}")
    AccountJpaEntity get(@PathVariable UUID id) { ... }   // ❌ entity döndü
}
```

Hexagonal mimari kuralı: Controller → Service (use case) → Repository (port). Direkt JPA repo kullanma.

**Anti-pattern 2: `findAll()` korkusuzca**

```java
List<AccountJpaEntity> all = repo.findAll();   // 10M kayıt = OOM
```

Banking'de bir tablo milyonlarca satır olur. `findAll` asla `Pageable`'sız çağrılmaz.

**Anti-pattern 3: N+1 üreten loop**

```java
List<UUID> ids = ...;
List<AccountJpaEntity> accounts = ids.stream()
    .map(id -> repo.findById(id).orElseThrow())   // N+1!
    .toList();
```

Çözüm: `findAllById(ids)` (tek query, IN clause).

**Anti-pattern 4: `@Query` string concat**

```java
@Query("SELECT a FROM AccountJpaEntity a WHERE a.id = " + accountId)   // SQL injection!
```

Her zaman `:param` named parameters.

**Anti-pattern 5: Spring Data'yı bırakıp her şeye native SQL**

Junior bazen "JPA çok karışık" deyip her şeyi native yazar. JPA'nın dirty checking, identity map, cascade gibi faydaları kaybolur. **Önce JPQL, gerekirse native.**

---

## Önemli olabilecek araştırma kaynakları

- Spring Data JPA reference documentation
- "Spring Data JPA: Beyond CRUD" (Eugen Paraschiv)
- Vlad Mihalcea JPA Performance series
- "Spring Data Specifications" Baeldung
- Hibernate ORM 6 user guide
- "Effective Java" — record immutability for DTO projection

---

## Mini task'ler

### Task 2.2.1 — Domain repository port'unu detaylandır (30 dk)

`banking/account/application/port/out/AccountRepository.java` interface'ini şu method'larla genişlet:

```java
public interface AccountRepository {
    Optional<Account> findById(AccountId id);
    Account save(Account account);
    List<Account> findByOwnerId(OwnerId ownerId);
    Page<Account> findByOwnerIdAndStatus(OwnerId ownerId, AccountStatus status, Pageable pageable);
    boolean existsById(AccountId id);
    long countActiveByOwnerId(OwnerId ownerId);
}
```

`Pageable` ve `Page` Spring Data class'ları — port'un Spring Data'ya bağlı olmasını mı tercih edersin yoksa kendi `PageRequest` abstraction'ını mı? **Defterine** karar yaz. (Pragmatik: Spring Data class'larını port'ta kullanmak normaldir, tam DDD purist arz etmez.)

### Task 2.2.2 — JPA repository'i implement et (30 dk)

`banking/account/adapter/out/persistence/AccountJpaRepository.java` (package-private):

- `JpaRepository<AccountJpaEntity, UUID>` extend et
- Custom derived query'leri ekle (yukarıdaki listeden)

`JpaAccountRepository.java` (adapter class):

- `AccountRepository` (port) implement et
- `AccountJpaRepository` inject et
- Mapper ile `Account ↔ AccountJpaEntity` çevir

### Task 2.2.3 — Pagination endpoint (45 dk)

`GET /v1/accounts?ownerId={uuid}&status=ACTIVE&page=0&size=20&sort=openedAt,desc`

Controller'da `Pageable` parameter al, repository'e ilet, `Page<Account>` döndür.

Response:

```java
public record PageResponse<T>(
    List<T> items,
    int page,
    int size,
    long totalElements,
    int totalPages
) {}
```

OpenAPI'da pagination parameter'larını dokümante et.

### Task 2.2.4 — Specification ile arama (45 dk)

`AccountSpecifications` class'ı yaz (yukarıda örnek). 5 filter ile:
- `ownerId`
- `status`
- `currency`
- `minBalance`
- `maxBalance`

Search endpoint: `POST /v1/accounts/search` body içinde filter. Specifications birleştir, paginated dön.

### Task 2.2.5 — Projection ile efficient reporting (30 dk)

`AccountSummaryDto` record yaz. `@Query` constructor expression ile sadece (id, currency, balanceAmount) çek.

İki versiyon karşılaştır:
1. `List<AccountJpaEntity> findAll()` → tüm field'lar SELECT
2. `List<AccountSummaryDto> findSummaries()` → sadece 3 field SELECT

SQL log'da farkı **gör**, **defterine yaz**.

### Task 2.2.6 — Auditing entegrasyonu (45 dk)

`AuditableEntity` mapped superclass yaz. Tüm entity'ler extend etsin.

Migration ekle (V4):

```sql
ALTER TABLE accounts ADD COLUMN created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW();
ALTER TABLE accounts ADD COLUMN updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW();
ALTER TABLE accounts ADD COLUMN created_by VARCHAR(100) NOT NULL DEFAULT 'system';
ALTER TABLE accounts ADD COLUMN updated_by VARCHAR(100) NOT NULL DEFAULT 'system';

ALTER TABLE journal_entries ADD COLUMN created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW();
-- ...
```

`AuditorAware` provider yaz. `@EnableJpaAuditing` ekle. Bir hesap insert/update et, kolonların dolduğunu doğrula.

### Task 2.2.7 — Soft delete (30 dk)

`AccountJpaEntity`'ye `deletedAt` field ekle. `@SQLDelete` ve `@Where` annotation'ları ile soft delete davranışı.

Migration:

```sql
ALTER TABLE accounts ADD COLUMN deleted_at TIMESTAMP WITH TIME ZONE;
```

Test:
- Hesap aç
- `repo.delete(account)` çağır
- SQL log'da DELETE değil UPDATE gör
- `repo.findById(id)` → empty (filter aktif)
- DB'de manuel SELECT → satır hâlâ var

### Task 2.2.8 — `getReferenceById` deneyi (15 dk)

Iki versiyon:

```java
// A — findById
Account from = accountRepo.findById(fromId).orElseThrow();
Account to = accountRepo.findById(toId).orElseThrow();
JournalEntry entry = new JournalEntry(from, to, amount);
journalRepo.save(entry);
// SQL'leri say: 2 SELECT + 1 INSERT

// B — getReferenceById
AccountJpaEntity fromRef = jpaRepo.getReferenceById(fromId);
AccountJpaEntity toRef = jpaRepo.getReferenceById(toId);
JournalEntry entry = ...;
journalRepo.save(entry);
// SQL'leri say: 0 SELECT + 1 INSERT
```

Farkı **defterine** yaz. Ne zaman A, ne zaman B?

---

## Test yazma rehberi

### Test 2.2.1 — Repository test (`@DataJpaTest` + TestContainers)

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class AccountJpaRepositoryTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired AccountJpaRepository repo;
    @Autowired TestEntityManager em;
    
    @Test
    void findByOwnerIdShouldReturnMatching() {
        UUID owner = UUID.randomUUID();
        em.persist(createAccountEntity(owner, "TRY"));
        em.persist(createAccountEntity(owner, "USD"));
        em.persist(createAccountEntity(UUID.randomUUID(), "TRY"));   // başka owner
        em.flush();
        
        List<AccountJpaEntity> result = repo.findByOwnerId(owner);
        
        assertThat(result).hasSize(2);
        assertThat(result).allMatch(a -> a.getOwnerId().equals(owner));
    }
    
    @Test
    void pagingShouldWork() {
        UUID owner = UUID.randomUUID();
        for (int i = 0; i < 25; i++) {
            em.persist(createAccountEntity(owner, "TRY"));
        }
        em.flush();
        
        Page<AccountJpaEntity> page0 = repo.findByOwnerId(owner, 
            PageRequest.of(0, 10, Sort.by("openedAt")));
        
        assertThat(page0.getContent()).hasSize(10);
        assertThat(page0.getTotalElements()).isEqualTo(25);
        assertThat(page0.getTotalPages()).isEqualTo(3);
    }
    
    @Test
    void specificationShouldFilter() {
        // ...
    }
}
```

### Test 2.2.2 — Projection test

```java
@Test
void summaryProjectionShouldOnlyFetchProjectedFields() {
    // Setup
    em.persist(createAccountEntity());
    em.flush();
    em.clear();
    
    // Reset Hibernate statistics
    SessionFactory sf = em.getEntityManagerFactory().unwrap(SessionFactory.class);
    sf.getStatistics().clear();
    
    List<AccountSummaryDto> summaries = repo.findSummariesByOwner(ownerId);
    
    // Bir tane prepare statement ama hiç entity load değil
    assertThat(sf.getStatistics().getPrepareStatementCount()).isEqualTo(1);
    assertThat(sf.getStatistics().getEntityLoadCount()).isZero();
}
```

### Test 2.2.3 — Auditing test

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class AuditingTest {
    
    @MockBean SecurityAuditorAware auditorAware;
    
    @Test
    void shouldPopulateAuditFieldsOnInsert() {
        when(auditorAware.getCurrentAuditor()).thenReturn(Optional.of("test-user"));
        
        Account account = Account.open(ownerId, Currency.getInstance("TRY"));
        Account saved = accountRepository.save(account);
        
        AccountJpaEntity entity = jpaRepo.findById(saved.getId().value()).orElseThrow();
        assertThat(entity.getCreatedAt()).isNotNull();
        assertThat(entity.getCreatedBy()).isEqualTo("test-user");
        assertThat(entity.getUpdatedAt()).isNotNull();
        assertThat(entity.getUpdatedBy()).isEqualTo("test-user");
    }
    
    @Test
    void shouldUpdateOnlyUpdatedFieldsOnModify() {
        // createdAt initial save'de set olsun, update'te değişmesin
        // updatedAt update'te değişsin
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki Spring Data JPA kodumu banking-grade kriterlere göre değerlendir. 
Eksik veya yanlışları işaretle, kod yazma:

1. Repository pattern:
   - `JpaRepository` extend eden interface package-private mi (dışarıya sızmıyor)?
   - Domain port (`AccountRepository`) ayrı, public mi?
   - Adapter class (`JpaAccountRepository`) port'u implement edip JPA repo'yu inject ediyor mu?
   - Controller'a JPA repo direkt inject edilmiş mi? (Olmamalı)

2. Derived queries:
   - Method ismi 4+ koşula uzanmış mı (okunmaz)?
   - Bunlar `@Query` ile JPQL'e çevrilmiş mi?
   - Pagination eklenmiş mi (`Pageable` parameter)?

3. `@Query`:
   - Named parameters (`:param`) mu, positional (`?1`) mu (named tercih)?
   - SQL string concat (`+`) ile parameter eklemiş mi? (Olmamalı — injection)
   - JPQL mi yoksa native mi — gerekçeli mi?

4. Pagination:
   - `Page` ve `Slice` kullanım yerleri bilinçli ayrılmış mı?
   - `findAll()` Pageable olmadan kullanılmış mı? (Olmamalı)
   - Endpoint'lerde `PageResponse` veya benzeri sınırlı response dönülüyor mu?

5. Sort:
   - Kullanıcı input'undan gelen sort field'lar whitelist'le filtreleniyor mu?
   - Sensitive field'larda sort'a izin var mı (sızıntı riski)?

6. Specification:
   - Dinamik filter'lı search endpoint Specification ile mi?
   - Specification helper class temiz mi (her field için ayrı static method)?
   - Null check'ler doğru (null → no filter, predicate null dön)?

7. Projection:
   - Reporting endpoint'lerde tam entity yerine projection (interface veya DTO) kullanılmış mı?
   - DTO projection `record` ile mi yapılmış?
   - JPQL constructor expression doğru mu (paket yolu tam)?

8. Auditing:
   - `@EnableJpaAuditing` aktif mi?
   - Tüm entity'ler `AuditableEntity` extend ediyor mu?
   - `AuditorAware` provider security context'ten user okuyor mu?
   - Audit kolonları DB'de NOT NULL mu?
   - `updatable = false` createdAt/createdBy'da var mı?

9. Soft delete:
   - Banking entity'lerinde fiziksel DELETE yerine soft delete kullanılmış mı?
   - `@SQLDelete` ve `@Where` ile transparent mi?
   - Audit kayıtlarında soft delete YOK mu (regülatör için fiziksel kayıt kalmalı)?

10. Performance:
    - `findById` ile FK kurmak için kullanılmış mı (gereksiz SELECT)?
    - `getReferenceById` performans-kritik yerlerde kullanılıyor mu?
    - `@Transactional(readOnly = true)` reporting service'lerinde var mı?
    - Loop içinde `findById` kullanılmış mı (N+1 sebebi)? `findAllById(ids)` tercih mi?

11. Anti-pattern:
    - SQL injection riski (string concat) var mı?
    - `findAll()` Pageable'sız var mı?
    - Tüm query'ler native mi (JPQL'i bilinçli mi atlamış)?
    - Entity callback'lerde business logic var mı (olmamalı)?

Her madde için PASS / FAIL / EKSIK işaretle ve neden olduğunu belirt. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] Domain port (`AccountRepository`) + JPA adapter (`JpaAccountRepository`) ayrımı yapıldı
- [ ] Derived query'ler 2-3 koşulla, daha fazlası `@Query`
- [ ] Pagination endpoint çalışıyor, `Page` veya `Slice` bilinçli seçildi
- [ ] Sort whitelist ile sanitize ediliyor
- [ ] Specification ile dinamik search çalışıyor
- [ ] DTO projection ile reporting query SQL log'da daha küçük SELECT üretiyor
- [ ] Auditing aktif, her entity audit kolonlarıyla
- [ ] Soft delete `accounts` üzerinde çalışıyor (DELETE → UPDATE)
- [ ] `getReferenceById` performans deneyimi yapıldı
- [ ] Repository test'leri `@DataJpaTest` + TestContainers ile yazıldı
- [ ] OSIV kapalı, hâlâ tüm endpoint'ler düzgün çalışıyor (eksik fetch'ler giderildi)

---

## Defter notları

1. "JpaRepository hierarchy: ____."
2. "Derived query method ismi 4+ koşulla yazınca ne yapmalı: ____."
3. "JPQL ile native SQL arasında karar verirken bakacağım kriter: ____."
4. "`Page` vs `Slice` vs `List` ne zaman hangisi: ____."
5. "Cursor-based pagination'ın offset-based'a göre avantajı: ____."
6. "Specification pattern'in derived query'e göre avantajı: ____."
7. "Interface projection vs DTO projection (class) farkı: ____."
8. "`getReferenceById` ile `findById` farkı + ne zaman: ____."
9. "Auditing için 4 standart kolon ve `updatable = false` neden: ____."
10. "Banking'de soft delete sebebi, hangi tablolar HARIÇ: ____."
