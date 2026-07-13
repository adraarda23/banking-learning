# Topic 2.1 — JPA Fundamentals & Entity Lifecycle

## Hedef

JPA'nın (Hibernate'in) **iç çalışmasını** anlamak. Bir nesne ne zaman "managed", ne zaman "detached"? Hibernate session ne tutuyor, neden? Dirty checking nasıl çalışıyor, performans cezası nedir? Flush ne zaman tetikleniyor? Bunları cevaplamadan JPA'yla banking yazmak = mayın tarlasında dans etmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Phase 1 bitti
- JPA temel annotation'ları gördün (`@Entity`, `@Id`, `@Column`)
- Basit bir `JpaRepository.findById(...)` çağrısı yaptın

---

## Kavramlar

### 1. JPA ≠ Hibernate (kısaca)

**JPA (Jakarta Persistence API):** Standart, sadece interface'ler ve annotation'lar. Spec.

**Hibernate:** JPA'nın en yaygın implementation'ı. Spring Boot default. Bizim kullanacağımız.

Diğer implementation'lar: EclipseLink, OpenJPA. TR bankalarında %95+ Hibernate.

JPA spec'ine sadık kalmak = ileride implementation değiştirmek mümkün. Hibernate-specific feature'ları kullanırsan (örn. `@DynamicUpdate`) JPA-tarif yapamıyorsun. **Kural:** Önce JPA-standard çözüm dene, gerekirse Hibernate-specific çık.

### 2. EntityManager — kalbin atışı

`EntityManager` JPA'nın merkezi API'si. Şu sorumlukları var:

- Entity'leri persist et (`em.persist(account)`)
- Entity bul (`em.find(Account.class, id)`)
- Entity sil (`em.remove(account)`)
- Query çalıştır (`em.createQuery(...)`)
- Transaction yönetimi (`em.getTransaction().begin()`)

Spring Boot'ta direkt `EntityManager`'ı çok az kullanırsın — `JpaRepository` üzerinden dolaylı kullanırsın. Ama **arka planda ne olduğunu bilmek şart**.

```java
@Service
class AccountReportingService {
    
    @PersistenceContext
    private EntityManager em;
    
    public Account findByIdNative(UUID id) {
        return em.find(AccountJpaEntity.class, id);
    }
    
    public List<Account> complexQuery() {
        return em.createQuery(
            "SELECT a FROM AccountJpaEntity a WHERE a.balanceAmount > :minBalance",
            AccountJpaEntity.class
        )
        .setParameter("minBalance", new BigDecimal("10000"))
        .getResultList();
    }
}
```

`@PersistenceContext` Spring'in EntityManager inject etmesi için. `@Autowired` da çalışır ama persistence-specific olduğundan `@PersistenceContext` daha doğru.

### 3. Persistence Context — kavramların kavramı

**Persistence Context (PC)** = bir transaction (veya request) boyunca yönetilen entity'lerin tutulduğu **in-memory cache**.

EntityManager bir PC'i kontrol eder. PC bir `Map<Class, Map<Id, Entity>>` gibi düşün.

**Davranışı:**

```java
@Transactional
public void example(UUID id) {
    Account a1 = em.find(AccountJpaEntity.class, id);   // DB'den fetch, PC'e koy
    Account a2 = em.find(AccountJpaEntity.class, id);   // PC'ten al, DB'ye gitmez
    
    assertThat(a1 == a2).isTrue();   // SAME REFERENCE
}
```

**İlk sorgu** DB'ye gider, **ikincisi** PC'ten gelir. Bu **first-level cache** denir.

**Ölçek:** PC bir transaction (`@Transactional` method) boyunca yaşar. Transaction biterse PC kapanır, içindeki entity'ler **detached** olur.

### 4. Entity State Machine — 4 hâl

Bir JPA entity 4 farklı hâlde olabilir:

```
                  ┌──────────────┐
                  │  TRANSIENT   │  ← new ile yarattın, henüz persist yok
                  └──────┬───────┘
                         │ persist()
                         ↓
                  ┌──────────────┐
            ┌────→│   MANAGED    │ ← PC içinde, dirty checking aktif
            │     └──────┬───────┘
            │            │
            │  merge()   │ remove()  detach() / clear()
            │            ↓
            │     ┌──────────────┐    ┌──────────────┐
            └─────┤   DETACHED   │    │   REMOVED    │
                  └──────────────┘    └──────────────┘
                                              │
                                              │ flush → DELETE SQL
                                              ↓
                                          deleted in DB
```

**TRANSIENT:** Sadece Java nesnesi. DB'de yok. ID henüz set edilmemiş (veya `@GeneratedValue` ise null).

```java
AccountJpaEntity a = new AccountJpaEntity();
a.setOwnerId(UUID.randomUUID());
// transient — JPA'nın haberi yok
```

**MANAGED:** Persistence context içinde. Hibernate her flush'ta dirty check yapıyor.

```java
em.persist(a);   // managed oldu, ID set edildi
a.setBalanceAmount(new BigDecimal("100"));   // dirty
// Transaction commit → UPDATE SQL otomatik fırlar
```

**DETACHED:** Daha önce managed'di, şimdi PC kapandığı için detached. State'i hâlâ Java'da ama JPA'nın haberdar olmuyor.

```java
@Transactional
public Account loadAccount(UUID id) {
    return em.find(AccountJpaEntity.class, id);
    // transaction biter → return edilen entity detached
}

// Caller'da:
Account a = svc.loadAccount(id);
a.setBalanceAmount(...);   // detached — UPDATE FIRTLAMAZ
```

Banking tuzağı: Service'ten dönen entity üzerinde değişiklik yapıp ekstra service çağırırsan iki ihtimal var:
1. Caller `@Transactional` değil → değişiklik kaybolur
2. Caller `@Transactional` ve aynı entity yeniden load ediliyor → değişikliğin üzerine yazılır

**Doğru pratik:** Service'ten domain object veya DTO dön, JPA entity asla.

**REMOVED:** `em.remove(entity)` çağrıldı, hâlâ PC'de ama "silinecek" işaretli. Flush'ta DELETE SQL'i.

### 5. Dirty Checking — sihir nasıl çalışıyor

Managed bir entity'nin field'ları değişirse, Hibernate transaction commit'inde **otomatik UPDATE SQL** fırlatır. Buna **dirty checking** denir.

```java
@Transactional
public void deposit(UUID accountId, BigDecimal amount) {
    AccountJpaEntity a = em.find(AccountJpaEntity.class, accountId);
    a.setBalanceAmount(a.getBalanceAmount().add(amount));
    // explicit save() ÇAĞIRMADIN — yine de UPDATE çıkar
}
```

**Sihir altı:**
1. `em.find` zamanı: Hibernate entity'nin **snapshot**'ını alır (kopyasını saklar)
2. Transaction commit'inde: mevcut entity vs snapshot karşılaştırılır
3. Fark varsa UPDATE SQL üretilir

**Performans uyarısı:**
- Snapshot tutmak hafıza yükü
- Karşılaştırma CPU yükü
- 1000 entity'i PC'de tutarsan 1000 snapshot, 1000 karşılaştırma

**Banking pratiği:** PC'i küçük tut. Büyük batch işlemlerde periodic `em.flush()` + `em.clear()` ile PC'i temizle (faz 5 batch'te detay).

### 6. Flush — DB'ye ne zaman yazılıyor

**Flush** = PC'deki değişiklikleri DB'ye **SQL olarak gönderme**. **Commit'ten farklıdır** — flush sonrası rollback hâlâ mümkün.

Flush ne zaman tetiklenir?

1. **Transaction commit öncesi** (default)
2. **Query çalıştırılırken** (flush mode AUTO ise — default)
3. **Manuel `em.flush()` çağrısı**

**Flush mode'lar:**

```java
em.setFlushMode(FlushModeType.AUTO);     // default — query öncesi otomatik
em.setFlushMode(FlushModeType.COMMIT);   // sadece commit öncesi
```

`AUTO` ile şu durum olur:

```java
@Transactional
public void example() {
    AccountJpaEntity a = em.find(AccountJpaEntity.class, id);
    a.setBalanceAmount(...);
    
    // Yeni query çalıştır
    Long count = (Long) em.createQuery("SELECT COUNT(a) FROM AccountJpaEntity a WHERE a.balanceAmount > 0")
        .getSingleResult();
    // ← Bu noktada flush oldu, UPDATE SQL DB'ye gitti
    // Yoksa count yanlış olabilirdi (pending update SQL henüz uygulanmadı)
}
```

**Hibernate'in akıllı kısmı:** Sadece query'nin etkilendiği tabloları flush eder (Hibernate 5+).

**Banking pratiği:** Çoğu zaman `AUTO` ile yaşa. Sadece performans-kritik batch'lerde `COMMIT`'e geç.

### 7. `flush()` ve `clear()` — manuel kontrol

```java
@Transactional
public void batchInsertAccounts(List<NewAccountRequest> requests) {
    int counter = 0;
    for (NewAccountRequest req : requests) {
        AccountJpaEntity a = new AccountJpaEntity();
        // ... set fields
        em.persist(a);
        
        if (++counter % 50 == 0) {
            em.flush();   // SQL'leri DB'ye gönder
            em.clear();   // PC'i boşalt — eski entity'ler detach
        }
    }
}
```

**Neden:** 100,000 entity'i tek PC'de tutmak OOM (OutOfMemory). Periyodik flush+clear ile PC küçük kalır.

**Tehlike:** Clear sonrası eski entity reference'larıyla çalışırsan **detached** olduğunu unutma — manage etmek gerekirse `em.merge()`.

### 8. Cascade Types

Bir entity'i persist ederken ilişkili entity'leri ne yapacağız?

```java
@Entity
class JournalEntryJpaEntity {
    @Id UUID id;
    
    @OneToMany(mappedBy = "journalEntry", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<JournalLineJpaEntity> lines = new ArrayList<>();
}
```

**Cascade types:**

| Type | Davranış |
|---|---|
| `PERSIST` | parent persist → child da persist |
| `MERGE` | parent merge → child da merge |
| `REMOVE` | parent remove → child da remove |
| `REFRESH` | parent refresh → child da refresh |
| `DETACH` | parent detach → child da detach |
| `ALL` | hepsi |

**`orphanRemoval = true`:** Parent'ın collection'ından bir child çıkardığında **DB'den de silinir**.

**Banking örneği — journal entry:**

```java
JournalEntryJpaEntity entry = new JournalEntryJpaEntity();
entry.getLines().add(new JournalLineJpaEntity(...));   // debit
entry.getLines().add(new JournalLineJpaEntity(...));   // credit
em.persist(entry);   // CASCADE PERSIST sayesinde lines'lar da kaydedilir
```

**Tuzak:** `CascadeType.REMOVE` ile `orphanRemoval = true` farklı. İkincisi child collection'dan çıkarmayı izler, birincisi sadece parent silinmesini.

**Banking pratiği:** Journal entry → journal lines ilişkisinde `CascadeType.ALL + orphanRemoval = true` mantıklı (lines aggregate'in parçası).

### 9. Identity & Equality — entity için tehlikeli

İki `AccountJpaEntity` ne zaman eşittir?

```java
AccountJpaEntity a1 = em.find(AccountJpaEntity.class, id);
AccountJpaEntity a2 = em.find(AccountJpaEntity.class, id);
a1 == a2;       // true (aynı PC, aynı entity)
a1.equals(a2);  // ??? default Object.equals → identity check
```

**Eğer ID'ye göre equals override etmediysen** default identity-based equality. Çoğu zaman doğru çalışır AMA:

```java
Set<AccountJpaEntity> accounts = new HashSet<>();
accounts.add(em.find(AccountJpaEntity.class, id));
em.clear();
accounts.contains(em.find(AccountJpaEntity.class, id));   // FALSE!
// Farklı PC'lerden geldi, identity farklı, equals false
```

**Doğru çözüm:**

```java
@Entity
class AccountJpaEntity {
    @Id UUID id;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof AccountJpaEntity other)) return false;
        return id != null && id.equals(other.id);
    }
    
    @Override
    public int hashCode() {
        // ID null olabilir (transient state) — class hash kullan
        return getClass().hashCode();
    }
}
```

**Neden `hashCode` ID'ye dayalı değil:** Transient state'te ID null. HashSet'e koyarsan, persist sonrası ID atanırsa hashCode değişir → set bozulur.

**Banking pratiği:** Entity'i Set'e koymadan önce iki kere düşün. JPA entity bir collection'a sokulduğunda equality problemi başlar. Domain'de (`Account` aggregate) bunu hisseder, JPA entity'de büyük dert.

### 10. `@GeneratedValue` strategies

Primary key nasıl üretilir?

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

**Stratejiler:**

| Strategy | Davranış | Banking için |
|---|---|---|
| `AUTO` | Hibernate karar verir | Belirsiz, kullanma |
| `IDENTITY` | DB auto-increment column | PostgreSQL OK, **batch insert kötü** |
| `SEQUENCE` | DB sequence | PostgreSQL/Oracle, **batch insert için ideal** |
| `TABLE` | id_generator tablosu | Yavaş, kullanma |
| `UUID` | Java UUID (JPA 3.1+) | Distributed system için iyi |

**Banking örneği:**

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

**Neden UUID:** Distributed sistem (Phase 7'de microservices'e bölecek), DB-bağımsız, çakışma yok.

**Neden IDENTITY tehlikeli:** PostgreSQL `IDENTITY` ile Hibernate **batch insert yapamaz** — her INSERT'in dönüş değerini almak gerekiyor, bu da batching'i kırıyor. 100 hesap insert ediyorsan 100 ayrı round-trip.

**SEQUENCE:**

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "account_seq")
@SequenceGenerator(name = "account_seq", sequenceName = "account_id_seq", allocationSize = 50)
private Long id;
```

`allocationSize = 50` → Hibernate sequence'i 50'şer atlayarak çeker. 50 insert için 1 sequence call. Banking'de yaygın.

### 11. Lazy vs Eager loading

```java
@Entity
class AccountJpaEntity {
    @Id UUID id;
    
    @OneToMany(mappedBy = "account", fetch = FetchType.LAZY)   // default
    private List<JournalLineJpaEntity> journalLines;
}
```

**LAZY:** Collection'a erişene kadar DB'den çekilmez. `account.getJournalLines()` çağrılırsa SQL fırlatılır.

**EAGER:** Account fetch'inde **JOIN ile birlikte** çekilir.

**Default'lar:**
- `@OneToOne` → EAGER (TEHLİKE)
- `@ManyToOne` → EAGER (TEHLİKE)
- `@OneToMany` → LAZY
- `@ManyToMany` → LAZY

**Banking tuzağı:** `@ManyToOne` default EAGER, her hesap çekiminde `Customer` da çekilir, her transaction çekiminde Account da çekilir → cascade fetch felaketleri.

**Banking kuralı:**

```java
@ManyToOne(fetch = FetchType.LAZY)   // her zaman explicit LAZY
@JoinColumn(name = "customer_id")
private CustomerJpaEntity customer;
```

EAGER kullanma. Gerekirse query'de `JOIN FETCH` ile ad-hoc fetch yap.

**Detay:** N+1 problem'in temeli LAZY'dir. Topic 2.5'te detayla göreceğiz.

### 12. `@Transient` vs JPA transient state

**Karıştırma:**

```java
@Entity
class AccountJpaEntity {
    @Id UUID id;
    
    @Transient   // ← JPA annotation
    private String formattedBalance;   // DB'de kolon yok, JPA görmezden gel
}
```

`@Transient` = "bu field DB'ye yazılmasın, JPA görmezden gel" anlamında.

**Transient state** = "bu entity henüz persist edilmemiş" anlamında.

İkisi farklı kavram — annotation aynı kelime ama farklı bağlam.

### 13. `LazyInitializationException` — junior'ın baş ağrısı

```java
@Transactional
public AccountJpaEntity loadAccount(UUID id) {
    return em.find(AccountJpaEntity.class, id);
}

// caller (no transaction)
AccountJpaEntity a = svc.loadAccount(id);
a.getJournalLines();   // ← BAM! LazyInitializationException
```

**Sebep:** Service'te transaction bitti, PC kapandı, entity detached. Caller LAZY collection'a erişmek istiyor — PC yok, fetch edilemez.

**Yanlış çözümler:**
- `fetch = FetchType.EAGER` yapmak (her account fetch'inde tüm lines)
- `open-session-in-view` aktif etmek (Spring Boot'un eski default'ı — KAPATILMALI)
- Trans'ı caller'a taşımak (sorunu erteleme)

**Doğru çözüm:**

1. **JOIN FETCH veya `@EntityGraph`** ile fetch et:

```java
@Query("SELECT a FROM AccountJpaEntity a LEFT JOIN FETCH a.journalLines WHERE a.id = :id")
Optional<AccountJpaEntity> findByIdWithLines(UUID id);
```

2. **DTO projection** — service entity dönmesin, DTO dönsün:

```java
@Transactional
public AccountReport loadAccountReport(UUID id) {
    AccountJpaEntity entity = em.find(AccountJpaEntity.class, id);
    return new AccountReport(
        entity.getId(),
        entity.getBalanceAmount(),
        entity.getJournalLines().stream()        // ← transaction içindeyken
            .map(JournalLineDto::from)
            .toList()
    );
}
```

Banking pratiği: **Service entity dönmez. DTO veya domain object döner.** Phase 1'in hexagonal arch kararı budur.

### 14. `open-session-in-view` (OSIV) — neden kapatmalısın

Spring Boot default'ta `spring.jpa.open-in-view = true`. Bu **HTTP request boyunca** transaction-less PC tutar. Yani lazy collection'lar request'in herhangi yerinde fetch edilebilir.

**Görünüşte iyi.** Aslında felaket:

1. **N+1 query**'ler controller veya view layer'da gizli kalır (transaction sınırında değil)
2. **Performance kötü** — her lazy access bir SQL, gözle göremezsin
3. **Architecture leakage** — controller layer DB ile konuşur

**Banking için kural:**

```yaml
spring:
  jpa:
    open-in-view: false
```

Bunu **mutlaka kapat**. İlk başta LazyInitializationException patlar — bu iyi. Sana doğru architecture'ı zorla.

### 15. `EntityManager` vs `JpaRepository`

Genelde Spring Data JPA `JpaRepository` kullanıyoruz. EntityManager kullanmak ne zaman?

**JpaRepository yeterli:** %85
**EntityManager direkt gerekiyor:** %15
- Complex dynamic query (CriteriaBuilder)
- Multi-tenancy senaryosu
- Native SQL with custom result mapping
- Batch processing fine-grained control

İkisini bir arada **karıştırma** dediğim şey: bir method'da hem `repository.save(...)` hem `em.persist(...)` yapmak. Kafan karışır. Birini seç.

### 16. JPA + Records — neden olmaz

```java
@Entity
public record AccountJpaEntity(UUID id, ...) {}   // ❌ Çalışmaz
```

JPA spec gereği entity'ler:
- Public/protected no-arg constructor olmalı
- Proxy oluşturulabilmesi için **final olmamalı**
- Field'lar (veya getter/setter) JPA'nın access edebileceği şekilde

`record` final. JPA proxy üretemez. **Entity için record kullanma.**

**Çözüm:** JPA entity manuel class, domain object record. İkisi arasında mapper.

### 17. Banking anti-pattern'leri

**Anti-pattern 1: Entity'i HTTP response'a sızdırma**

Phase 1'de detayla anlattık. JPA entity hiçbir zaman Controller'dan dönmez.

**Anti-pattern 2: Service'ten entity dönme**

```java
@Service
class AccountService {
    @Transactional
    public AccountJpaEntity getById(UUID id) {   // ❌ entity döndü
        return em.find(...);
    }
}
```

Detached entity ile çalışmaya zorlanırsın → LazyInitializationException, dirty checking olmaz, vs.

**Çözüm:** Domain object veya DTO dön.

**Anti-pattern 3: Manuel ID set**

```java
AccountJpaEntity a = new AccountJpaEntity();
a.setId(UUID.randomUUID());   // ← manuel ID
em.persist(a);
```

**Sorun:** `@GeneratedValue` ile çakışırsa unpredictable. Manuel ID istiyorsan `@GeneratedValue` koyma.

**Anti-pattern 4: Equals/hashCode bypass**

Entity'i collection'a koyup ID'ye dayalı equals override etmemek. Yukarıda anlattım.

**Anti-pattern 5: `cascade = CascadeType.ALL` her yerde**

`@ManyToOne(cascade = ALL)` çocuğun parent'ı silmesini sağlar — felaket. Cascade her zaman düşünerek seç.

---

## Önemli olabilecek araştırma kaynakları

- "Java Persistence with Hibernate" (Christian Bauer) — referans kitap
- "High-Performance Java Persistence" (Vlad Mihalcea) — banking için altın
- Vlad Mihalcea blog (vladmihalcea.com) — günde 1 yazı okumak yıllık eğitim
- Hibernate User Guide
- Spring Data JPA reference
- "Java Persistence Performance" blog yazıları (Hibernate ekibi)
- IBM Developer "JPA Best Practices"

---

## Mini task'ler

### Task 2.1.1 — Persistence context deneyi (30 dk)

`AccountPlayground.java` (geçici):

```java
@SpringBootApplication
public class PersistenceContextDemo implements CommandLineRunner {
    
    @PersistenceContext EntityManager em;
    
    @Transactional
    public void run(String... args) {
        UUID id = createTestAccount();
        
        AccountJpaEntity a1 = em.find(AccountJpaEntity.class, id);
        AccountJpaEntity a2 = em.find(AccountJpaEntity.class, id);
        
        System.out.println("Same reference? " + (a1 == a2));
        System.out.println("a1 == a2? " + a1.equals(a2));
        
        a1.setBalanceAmount(new BigDecimal("999"));
        System.out.println("a2 balance: " + a2.getBalanceAmount());   // a1'in değişikliğini gör mü?
    }
}
```

Sonucu **defterine yaz**.

### Task 2.1.2 — Entity state geçişleri (30 dk)

`EntityLifecycleDemo.java`:

- New entity yarat (transient)
- `em.persist` (managed)
- Field değiştir, flush et — UPDATE SQL gör (SQL log aç)
- `em.detach` (detached)
- Field değiştir — UPDATE çıkmaz, gör
- `em.merge` (managed yeniden)
- `em.remove` (removed)
- commit → DELETE SQL gör

Her geçişte **defter notu** al.

### Task 2.1.3 — Dirty checking inceleme (20 dk)

`application-dev.yml`:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
    org.hibernate.stat: DEBUG
```

Bir deposit transaction'ı çalıştır. Log'da:
- Kaç SQL fırlandı?
- Hibernate statistics'te ne göreceksin?

### Task 2.1.4 — OSIV kapat ve test et (15 dk)

`application.yml`:

```yaml
spring:
  jpa:
    open-in-view: false
```

Bir reporting endpoint yaz, lazy collection'a controller'da erişmeye çalış. `LazyInitializationException`'ı **canlı gör**, defter notu al, sonra service'te fetch ederek çöz.

### Task 2.1.5 — Equals/hashCode entity için (30 dk)

`AccountJpaEntity`'ne ID-based equals + class-based hashCode ekle. Test:

```java
Set<AccountJpaEntity> set = new HashSet<>();
set.add(em.find(AccountJpaEntity.class, id));
em.clear();
boolean stillContains = set.contains(em.find(AccountJpaEntity.class, id));
```

`stillContains` true olmalı.

### Task 2.1.6 — Batch insert deneyi (45 dk)

100 account insert eden bir method yaz. İki versiyon:

**Versiyon A — flush/clear olmadan:**

```java
@Transactional
public void insertManyBad(int count) {
    for (int i = 0; i < count; i++) {
        em.persist(new AccountJpaEntity(...));
    }
}
```

**Versiyon B — periodic flush/clear:**

```java
@Transactional
public void insertManyGood(int count) {
    for (int i = 0; i < count; i++) {
        em.persist(new AccountJpaEntity(...));
        if (i % 50 == 49) {
            em.flush();
            em.clear();
        }
    }
}
```

Bellek tüketimi farkını ölç (`-Xmx128m` ile çalıştır, 10000 entity, A patlar, B çalışır).

### Task 2.1.7 — Cascade ve orphanRemoval (30 dk)

`JournalEntryJpaEntity`'ne `@OneToMany(cascade = ALL, orphanRemoval = true)` ile `lines` ekle.

Bir journal entry + 2 line yarat, persist. SQL log'da kaç INSERT'i gör.

Sonra entry'nin lines listesinden bir line `remove()` et, flush. DELETE SQL'i gör.

---

## Test yazma rehberi

### Test 2.1.1 — Persistence context cache test

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class PersistenceContextTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired EntityManager em;
    @Autowired TestEntityManager testEm;
    
    @Test
    void firstLevelCacheReturnsSameReference() {
        UUID id = UUID.randomUUID();
        AccountJpaEntity entity = createAccount(id);
        testEm.persist(entity);
        testEm.flush();
        testEm.clear();   // PC'i sıfırla
        
        AccountJpaEntity a1 = em.find(AccountJpaEntity.class, id);
        AccountJpaEntity a2 = em.find(AccountJpaEntity.class, id);
        
        assertThat(a1).isSameAs(a2);   // referans eşitliği
    }
    
    @Test
    void dirtyCheckingTriggersUpdate() {
        UUID id = UUID.randomUUID();
        testEm.persist(createAccount(id));
        testEm.flush();
        testEm.clear();
        
        AccountJpaEntity managed = em.find(AccountJpaEntity.class, id);
        managed.setBalanceAmount(new BigDecimal("100.00"));
        // setBalanceAmount sonrası explicit save YOK
        em.flush();   // UPDATE fırlamalı
        
        testEm.clear();
        AccountJpaEntity reloaded = em.find(AccountJpaEntity.class, id);
        assertThat(reloaded.getBalanceAmount()).isEqualByComparingTo("100.00");
    }
    
    @Test
    void detachedEntityChangeDoesNotPersist() {
        UUID id = UUID.randomUUID();
        testEm.persist(createAccount(id));
        testEm.flush();
        testEm.clear();
        
        AccountJpaEntity managed = em.find(AccountJpaEntity.class, id);
        em.detach(managed);
        
        managed.setBalanceAmount(new BigDecimal("999.00"));
        em.flush();   // UPDATE çıkmamalı (detached)
        
        testEm.clear();
        AccountJpaEntity reloaded = em.find(AccountJpaEntity.class, id);
        assertThat(reloaded.getBalanceAmount()).isNotEqualByComparingTo("999.00");
    }
}
```

### Test 2.1.2 — Equals/hashCode test

```java
@Test
void shouldUseIdBasedEquality() {
    UUID id = UUID.randomUUID();
    AccountJpaEntity a1 = new AccountJpaEntity();
    a1.setId(id);
    AccountJpaEntity a2 = new AccountJpaEntity();
    a2.setId(id);
    
    assertThat(a1).isEqualTo(a2);
    assertThat(a1).hasSameHashCodeAs(a2);
}

@Test
void shouldUseClassHashCodeForTransient() {
    AccountJpaEntity a1 = new AccountJpaEntity();   // ID null
    AccountJpaEntity a2 = new AccountJpaEntity();   // ID null
    
    // Hash aynı olmalı (class bazlı), equals false
    assertThat(a1.hashCode()).isEqualTo(a2.hashCode());
    assertThat(a1).isNotEqualTo(a2);   // farklı transient entity
}
```

### Test 2.1.3 — LazyInitializationException

```java
@SpringBootTest
class LazyLoadingTest {
    
    @Autowired AccountService accountService;
    @Autowired EntityManager em;
    
    @Test
    void lazyAccessOutsideTransactionShouldThrow() {
        // service entity dönsün (bilerek anti-pattern test ediyor)
        AccountJpaEntity loaded = accountService.findEntityById(testAccountId);   // detached
        
        assertThatThrownBy(() -> loaded.getJournalLines().size())
            .isInstanceOf(LazyInitializationException.class);
    }
    
    @Test
    void joinFetchShouldAvoidLazyException() {
        AccountWithLines result = accountService.findWithLinesById(testAccountId);
        // service içinde JOIN FETCH yapıldı, DTO döndü
        assertThat(result.lines()).isNotNull();
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki JPA kodumu banking-grade JPA fundamentals kriterlerine göre değerlendir. 
Sadece eksik veya yanlışları işaretle, kod yazma:

1. EntityManager kullanımı:
   - `@PersistenceContext` ile inject ediliyor mu (`@Autowired` DEĞİL — gerçi çalışır)?
   - EntityManager ve JpaRepository karışık kullanılıyor mu (Olmamalı)?

2. Entity state'ler:
   - Service method'larda transient → managed → detached geçişleri bilinçli mi?
   - Service'ten entity döndürülüyor mu (Olmamalı — DTO veya domain dön)?
   - Detached entity'lerle çalışmaya zorlanan bir akış var mı?

3. Equals/hashCode:
   - Entity'lerde equals ID bazlı mi?
   - hashCode CLASS bazlı mı (ID DEĞİL — transient state için)?
   - Lombok `@Data` veya `@EqualsAndHashCode` ile yanlış eq/hash üretilmiş mi?

4. Cascade ve orphanRemoval:
   - `cascade = ALL` gereksiz yere her yerde mi (sadece aggregate child için olmalı)?
   - `orphanRemoval = true` ile `CascadeType.REMOVE` farkı doğru kullanılmış mı?

5. Fetch type'lar:
   - `@ManyToOne` ve `@OneToOne` default EAGER'a güveniliyor mu? Explicit `fetch = LAZY` var mı?
   - LAZY collection access'leri transaction içinde mi?
   - JOIN FETCH veya `@EntityGraph` kullanılmış mı?

6. Konfigürasyon:
   - `spring.jpa.open-in-view: false` mu? (Mutlaka false)
   - `ddl-auto: validate` mu? (create/update OLMAMALI)
   - SQL logging dev/test profile'da aktif mi?
   - `generate_statistics: true` ile Hibernate stats görünüyor mu?

7. ID generation:
   - `@GeneratedValue` strategy ne (IDENTITY, SEQUENCE, UUID)?
   - PostgreSQL IDENTITY ile batch insert kullanılıyorsa performans bilinçli mi?
   - SEQUENCE ile `allocationSize` 1'den fazla mı (50+)?

8. Record ile JPA entity:
   - Hiçbir entity `record` mı (yanlış — record entity olmaz)?
   - Entity'ler `final` mı (olmamalı — proxy gerekiyor)?

9. Batch processing:
   - 1000+ entity insert eden method'da periodic flush/clear var mı?
   - Yok mu — OOM riski var mı?

10. Anti-pattern:
    - Entity HTTP response'a dönüyor mu?
    - Controller layer'da lazy collection'a erişim var mı (OSIV kapalı olmasa bile yanlış)?
    - Manuel ID set + `@GeneratedValue` karışık mı?

Her madde için PASS / FAIL / EKSIK işaretle. Kod yazmadan açıklama yap.
```

---

## Tamamlama kriterleri

- [ ] PC cache davranışı kendi gözümle gördüm (`a1 == a2` deneyi)
- [ ] 4 entity state'in geçişlerini SQL log'unda izledim
- [ ] OSIV'i kapadım (`open-in-view: false`)
- [ ] LazyInitializationException'ı bilerek aldım, fix yöntemlerini biliyorum
- [ ] Entity'ye ID-based equals + class-based hashCode yazdım
- [ ] Cascade type'ları kavradım, hangi entity'de neyi neden kullandığımı söyleyebilirim
- [ ] `@GeneratedValue` strategy seçim kararımı verebiliyorum
- [ ] Batch insert için flush/clear pattern'i biliyorum
- [ ] Entity'ye `record` kullanmama sebebini biliyorum
- [ ] Dirty checking iç çalışma mantığını anlatabiliyorum

---

## Defter notları

1. "JPA ve Hibernate ilişkisi: ____."
2. "Persistence context = ____. Yaşam süresi = ____."
3. "Entity'nin 4 state'i ve geçiş tetikleyicileri: ____."
4. "Dirty checking nasıl çalışır (snapshot mekaniği): ____."
5. "Flush ne zaman tetiklenir (3 senaryo): ____."
6. "OSIV'i neden kapatıyorum: ____."
7. "LazyInitializationException'ın gerçek sebebi ve doğru çözüm: ____."
8. "JPA entity için equals'i ID-based, hashCode'u class-based yapmamın sebebi: ____."
9. "`@OneToMany` default LAZY ama `@ManyToOne` default EAGER — bu ne gibi sorunlar yaratır: ____."
10. "Entity'ye `record` neden kullanılamaz: ____."
