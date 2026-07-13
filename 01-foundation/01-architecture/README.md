# Topic 1.1 — Mimari: Hexagonal, Ports & Adapters, Layered

```admonish info title="Bu bölümde"
- Layered ve hexagonal mimarilerin farkını ve **bağımlılık yönünün** neden kritik olduğunu öğreneceksin
- Port ve adapter kavramlarını (driving / driven) banking örnekleri üzerinden tanıyacaksın
- DDD'nin temel yapı taşlarını göreceksin: value object, entity, aggregate root, domain event
- Feature-first package yapısını ve ADR yazma alışkanlığını kazanacaksın
- Anemic Domain Model başta olmak üzere yaygın anti-pattern'leri tanımayı öğreneceksin
```

## Hedef

Bir Java backend uygulamasının kodunu **neye göre organize edersin** sorusunu derinlemesine cevaplamak. Banking domain'inde neden bu kararın "uygulamayı taşıyabilir/test edilebilir" yapmak için kritik olduğunu görmek.

## Süre

Okuma: 1.5-2 saat • Mini task'ler: 2 saat • Test: 30 dk • Toplam: ~4.5 saat

## Önbilgi

- Spring Boot temel (controller, service, repository annotation'larını gördün)
- Maven hakkında genel fikir

---

## Kavramlar

### 1. Neden mimari kararları önemli — banking perspektifi

Bankada bir hesap servisi 10 yıl yaşar. Veritabanı değişir (Oracle → PostgreSQL), framework değişir (Spring 5 → 6 → ?), arayüz değişir (SOAP → REST → gRPC). **Domain logic değişmez**: bir hesabın bakiyesi negatife düşemez, her transfer iki kayıt üretir, faiz hesabı çok kesin kurallara tabi.

Yanlış mimari = teknoloji değişikliği = uygulamayı sıfırdan yazmak.
Doğru mimari = teknoloji çevre katmanlarda değişir, çekirdek (domain) sağlam kalır.

Bu yüzden **bağımlılık yönü** mimarinin can damarı: çevre koda mı bağlı, yoksa çevre çekirdek koda mı bağlı?

---

### 2. Layered Architecture (3-katmanlı / N-tier)

**Tanım:** Kodu yatay katmanlara böler. Klasik versiyon:

```
┌─────────────────────────┐
│  Controller (web)       │  ← HTTP istekleri burada karşılanır
├─────────────────────────┤
│  Service (business)     │  ← İş kuralları burada
├─────────────────────────┤
│  Repository (data)      │  ← DB ile konuşan kod
├─────────────────────────┤
│  Database               │
└─────────────────────────┘
```

Bağımlılık yönü yukarıdan aşağı: Controller → Service → Repository → DB.

```mermaid
flowchart TD
    C["Controller - HTTP isteklerini karşılar"] --> S["Service - iş kuralları"]
    S --> R["Repository - DB ile konuşur"]
    R --> DB["Database"]
```

**Neden popüler:** Spring Boot tutorial'larının %95'i bu yapıdadır. Hızlı başlanır. Junior'ın aşina olduğu yapı.

**Basit örnek:**

```java
// Controller — HTTP'i bilir
@RestController
@RequestMapping("/accounts")
class AccountController {
    private final AccountService service;
    
    @PostMapping
    ResponseEntity<AccountResponse> create(@RequestBody CreateAccountRequest req) {
        return ResponseEntity.ok(service.create(req));
    }
}

// Service — business logic
@Service
class AccountService {
    private final AccountRepository repo;
    
    public AccountResponse create(CreateAccountRequest req) {
        var account = new Account(req.ownerId(), req.currency());
        account = repo.save(account);
        return new AccountResponse(account.getId(), account.getBalance());
    }
}

// Repository — DB'i bilir
interface AccountRepository extends JpaRepository<Account, Long> { }
```

**Sorunlar (banking için):**

```admonish warning title="Dikkat — Layered'ın banking için tuzakları"
1. **Domain modeli framework'e bağımlı.** `Account` entity'si `@Entity`, `@Id`, `@Column` annotation'ları ile JPA'ya yapıştı. JPA bırakırsan domain modelini yeniden yazarsın.

2. **Service'i unit test etmek zor.** `AccountRepository` Spring bean olduğu için mock veya `@DataJpaTest` gerekir. Saf bir Java birim testi yazamazsın.

3. **Çekirdek iş kuralları her yere dağılır.** Bakiye kontrolü Service'te, validation Controller'da, audit trigger Repository'de — kim hangi kuralı uyguluyor karışır.

4. **Dış sistem entegrasyonu sızar.** Bir external bank API'sini servise enjekte ettiğin anda, servisin domain logic'i ile network detayları birbirine girer.

5. **Bağımlılık yönü domain'e dayatma yapar.** Service, JPA repository'e bağımlıdır → service domain'i değil, persistence'ı tanır.
```

---

### 3. Hexagonal Architecture (Ports & Adapters)

Alistair Cockburn tarafından önerildi (2005). **Aynı şey** "Ports & Adapters", "Onion Architecture", "Clean Architecture" — küçük farklar var ama özü aynı.

**Tanım:** Uygulamanın **çekirdek domain logic'i** ortada durur. Tüm dış dünya (DB, HTTP, Kafka, dosya sistemi) **çevrede** durur. Çekirdek dış dünyaya **bağımlı değildir** — dış dünya çekirdeğe bağımlıdır.

```
                  ┌─────────────────┐
                  │   HTTP / REST   │
                  │   (adapter)     │
                  └────────┬────────┘
                           ↓
        ┌──────────────────────────────────┐
        │                                  │
        │       Application Core           │
        │   ┌─────────────────────────┐    │
        │   │      Domain Model       │    │  ← saf Java, framework yok
        │   │  Account, Transfer,     │    │
        │   │  JournalEntry...        │    │
        │   └─────────────────────────┘    │
        │                                  │
        │   Use Cases / Application        │  ← orchestration
        │   Services                       │
        │                                  │
        │   Ports (interfaces)             │  ← çekirdek "şuna ihtiyacım var" der
        │   - AccountRepository (port)     │
        │   - NotificationSender (port)    │
        └──────┬────────────────────┬──────┘
               ↓                    ↓
        ┌──────────────┐   ┌──────────────┐
        │ JPA Adapter  │   │ Kafka Adapter│
        └──────┬───────┘   └──────────────┘
               ↓
        ┌──────────────┐
        │  PostgreSQL  │
        └──────────────┘
```

Aynı yapının katman ve bağımlılık ilişkisi:

```mermaid
flowchart TD
    subgraph CORE["Application Core"]
        IN["Use Case - driving port"] --> SVC["Application Service"]
        SVC --> DOM["Domain Model - saf Java"]
        SVC --> OUT["AccountRepository - driven port"]
        SVC --> OUT2["NotificationSender - driven port"]
    end
    HTTP["HTTP Controller - driving adapter"] --> IN
    JPA["JPA Adapter - driven adapter"] -. "implement eder" .-> OUT
    KAFKA["Kafka Adapter - driven adapter"] -. "implement eder" .-> OUT2
    JPA --> PG["PostgreSQL"]
```

**Anahtar kavramlar:**

**Port:** Çekirdeğin "dış dünyadan beklediği" interface. Çekirdek tanımlar.
- *Driving port* (sol/üst): "Beni dışarıdan çağırın" — use case interface'i
- *Driven port* (sağ/alt): "Bana bunları sağlayın" — repository, message sender interface'i

**Adapter:** Port'u gerçek bir teknoloji ile somutlaştıran kod.
- *Driving adapter*: HTTP controller, gRPC handler, Kafka consumer — uygulamayı çağıran taraf
- *Driven adapter*: JPA repository implementation, Kafka producer, SMS gateway client — uygulamanın çağırdığı taraf

**Dependency inversion:** Çekirdek bir interface tanımlar, adapter onu implement eder. Çekirdek **adapter'ın varlığını bilmez**.

**Banking domain örneği:**

```java
// === DOMAIN === (framework yok, Spring yok, JPA yok)
// banking/domain/account/Account.java
public class Account {
    private final AccountId id;
    private final OwnerId ownerId;
    private Money balance;
    
    public void debit(Money amount) {
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException(id);
        }
        balance = balance.subtract(amount);
    }
    
    public void credit(Money amount) {
        balance = balance.add(amount);
    }
    // ... getters, no setters
}

// === PORT === (interface, domain'in dışarıdan istediği)
// banking/application/port/out/AccountRepository.java
public interface AccountRepository {
    Optional<Account> findById(AccountId id);
    void save(Account account);
}

// === PORT (use case) === (driving)
// banking/application/port/in/TransferMoneyUseCase.java
public interface TransferMoneyUseCase {
    void transfer(AccountId from, AccountId to, Money amount);
}

// === APPLICATION SERVICE === (orchestration)
// banking/application/service/TransferMoneyService.java
public class TransferMoneyService implements TransferMoneyUseCase {
    private final AccountRepository accountRepository;  // ← port, ne JPA bilir ne Spring
    
    public TransferMoneyService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }
    
    @Override
    public void transfer(AccountId from, AccountId to, Money amount) {
        var fromAccount = accountRepository.findById(from).orElseThrow();
        var toAccount = accountRepository.findById(to).orElseThrow();
        fromAccount.debit(amount);
        toAccount.credit(amount);
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
    }
}

// === ADAPTER (driven) === (JPA implementation)
// banking/adapter/out/persistence/JpaAccountRepository.java
@Component
class JpaAccountRepository implements AccountRepository {
    private final AccountJpaRepository jpaRepo;          // Spring Data JPA
    private final AccountPersistenceMapper mapper;
    
    @Override
    public Optional<Account> findById(AccountId id) {
        return jpaRepo.findById(id.value()).map(mapper::toDomain);
    }
    
    @Override
    public void save(Account account) {
        var entity = mapper.toEntity(account);
        jpaRepo.save(entity);
    }
}

// JPA entity'si domain'in dışında, sadece persistence katmanında
@Entity
@Table(name = "accounts")
class AccountJpaEntity {
    @Id Long id;
    Long ownerId;
    BigDecimal balanceAmount;
    String balanceCurrency;
    // ...
}

// === ADAPTER (driving) === (HTTP controller)
// banking/adapter/in/web/TransferController.java
@RestController
@RequestMapping("/transfers")
class TransferController {
    private final TransferMoneyUseCase transferMoney;    // ← port, somutu bilmez
    
    public TransferController(TransferMoneyUseCase transferMoney) {
        this.transferMoney = transferMoney;
    }
    
    @PostMapping
    public ResponseEntity<Void> transfer(@Valid @RequestBody TransferRequest req) {
        transferMoney.transfer(
            new AccountId(req.fromAccountId()),
            new AccountId(req.toAccountId()),
            Money.of(req.amount(), req.currency())
        );
        return ResponseEntity.ok().build();
    }
}
```

Bir transfer isteğinin bu yapıda uçtan uca akışı şöyle:

```mermaid
sequenceDiagram
    participant C as TransferController
    participant S as TransferMoneyService
    participant A as Account aggregate
    participant R as JpaAccountRepository
    participant DB as PostgreSQL
    C->>S: transfer via TransferMoneyUseCase
    S->>R: findById via AccountRepository port
    R->>DB: SELECT
    R-->>S: Account domain nesnesi
    S->>A: debit ve credit
    A-->>S: kurallar uygulandi
    S->>R: save via port
    R->>DB: UPDATE
    S-->>C: tamam
```

Controller sadece **port interface'ini** bilir, service sadece **port üzerinden** persistence'a ulaşır — hiçbir ok domain'den dışarı çıkmaz.

**Avantajlar:**

1. **Domain saf Java.** Spring veya JPA olmadan IDE'de çalıştırırsın. Unit test'leri mock'sız yazılır.
2. **Adapter değiştirilebilir.** JPA → MongoDB geçişi sadece adapter değişir, domain kalır.
3. **Test piramidi sağlıklı:** domain unit test (hızlı, çok), adapter test (TestContainers ile), integration test (az).
4. **Açık kontrat:** Port'lar bir interface, "uygulamanın dış dünyadan ihtiyaçları neler" sorusunun tek cevabı.

**Dezavantajlar:**

1. **Daha fazla kod.** Domain ↔ persistence için mapper yazarsın.
2. **Junior için karmaşık.** İlk bakışta over-engineering gibi gelir.
3. **Küçük projede gereksiz.** CRUD bir admin paneli için çok ağır.

```admonish tip title="İpucu — banking için neden değer"
Domain kuralları (negative balance, double-entry invariant, faiz formülü) **on yıllarca yaşayacak**. Teknoloji 3 yılda bir değişir. Hexagonal seni yaşatır.
```

---

### 4. Onion Architecture & Clean Architecture (kısaca)

**Onion Architecture (Jeffrey Palermo):** Hexagonal'in benzeri, sadece görselleştirmesi "soğan halkaları" şeklinde. Merkezde Domain, sonra Application Services, sonra Infrastructure. Aynı dependency inversion prensibi.

**Clean Architecture (Uncle Bob):** Yine aynı fikir. Katmanları "Entities → Use Cases → Interface Adapters → Frameworks & Drivers" diye adlandırır. **"The Dependency Rule"** der: bağımlılıklar sadece dışarıdan içeri akar.

Üçü de pratikte aynı yapıyı önerir. İsim farklı, fikir aynı: **domain merkez, framework çevre, bağımlılık içe doğru.**

---

### 5. Domain-Driven Design (DDD) — mimari değil ama bağlantılı

Eric Evans (2003). DDD bir **mimari** değil, **modelleme yaklaşımı**. Hexagonal ile çok iyi eşleşir.

**DDD'nin Faz 1'de bilmen gereken parçaları:**

- **Ubiquitous language:** Kod ile iş diliyle aynı kelimeleri kullanmak. Bankacı "EFT" diyorsa kod da `Eft` diyor, `Transfer` değil.
- **Bounded context:** Büyük domain'i parçalara böl. Account context, Card context, Transfer context — her birinin kendi modeli olabilir.
- **Aggregate:** Bir tutarlılık sınırı. Bir aggregate root (örn. `Account`), kendisine ait child'ları (`JournalEntry`'leri) yönetir. Aggregate root **dışarıdan yegane giriş noktasıdır**.
- **Entity vs Value Object:**
  - *Entity:* Kimliği var (`Account`, `Customer`). İki müşterinin aynı adı olsa da farklı entity'dir.
  - *Value Object:* Kimliği yok, değeri ile tanımlanır. `Money(100.00, TRY)` herhangi bir `Money(100.00, TRY)` ile eşittir. **Immutable** olmalı.
- **Domain event:** Domain'de bir şey olduğunda yayılan mesaj. `MoneyTransferred`, `AccountOpened` gibi.

**Banking örneği — value object:**

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount == null || currency == null) throw new IllegalArgumentException();
        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("Too many decimal places for " + currency);
        }
    }
    
    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
    }
    
    public boolean isLessThan(Money other) {
        requireSameCurrency(other);
        return amount.compareTo(other.amount) < 0;
    }
    
    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException(currency, other.currency);
        }
    }
}
```

**Banking örneği — aggregate root:**

```java
public class Account {
    private final AccountId id;
    private final OwnerId ownerId;
    private final Currency currency;
    private Money balance;
    private final List<DomainEvent> events = new ArrayList<>();
    
    public void deposit(Money amount, TransferId transferId) {
        if (!amount.currency().equals(this.currency)) {
            throw new CurrencyMismatchException(this.currency, amount.currency());
        }
        balance = balance.add(amount);
        events.add(new MoneyDeposited(id, amount, transferId, Instant.now()));
    }
    
    public void withdraw(Money amount, TransferId transferId) {
        if (!amount.currency().equals(this.currency)) {
            throw new CurrencyMismatchException(this.currency, amount.currency());
        }
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException(id);
        }
        balance = balance.subtract(amount);
        events.add(new MoneyWithdrawn(id, amount, transferId, Instant.now()));
    }
    
    public List<DomainEvent> pullEvents() {
        var pulled = List.copyOf(events);
        events.clear();
        return pulled;
    }
    // ...
}
```

```admonish warning title="Dikkat"
`Account` saf Java. `@Entity` yok. JPA ayrı bir `AccountJpaEntity` ile temsil edilir, mapper ile dönüşür.
```

---

### 6. Package yapısı — nasıl klasörleyeceksin

Hexagonal'i package'a yansıtmanın iki ana yolu var:

**Variant A — "feature first, then layer":**

```
com.mavibank.banking
├── account
│   ├── domain
│   │   ├── Account.java
│   │   ├── AccountId.java
│   │   └── Money.java
│   ├── application
│   │   ├── port
│   │   │   ├── in
│   │   │   │   └── OpenAccountUseCase.java
│   │   │   └── out
│   │   │       └── AccountRepository.java
│   │   └── service
│   │       └── OpenAccountService.java
│   └── adapter
│       ├── in
│       │   └── web
│       │       └── AccountController.java
│       └── out
│           └── persistence
│               ├── JpaAccountRepository.java
│               └── AccountJpaEntity.java
└── transfer
    ├── domain
    ├── application
    └── adapter
```

**Variant B — "layer first, then feature":**

```
com.mavibank.banking
├── domain
│   ├── account
│   └── transfer
├── application
│   ├── account
│   └── transfer
└── adapter
    ├── in.web
    │   ├── account
    │   └── transfer
    └── out.persistence
        ├── account
        └── transfer
```

```admonish tip title="İpucu — hangisini seçmeli?"
A (feature-first). Sebep:
- Bir feature'a dokunmak istediğinde tüm kod aynı yerde
- Microservice'e ayırmak gerektiğinde feature'ı tek klasör çıkarırsın
- Bounded context fikriyle daha iyi eşleşir

Bu projede Variant A kullanacağız.
```

---

### 7. Modüllerin Maven multi-module ile fiziksel ayrılığı (opsiyonel ama güçlü)

Sadece package değil, **Maven module** olarak da ayırabilirsin:

```
core-banking/
├── pom.xml                          (parent)
├── banking-domain/                  (sadece domain)
│   └── pom.xml
├── banking-application/             (use cases + ports)
│   └── pom.xml                      (banking-domain'e depend)
├── banking-adapter-persistence/     (JPA adapter)
│   └── pom.xml                      (application'a depend, Spring Data JPA dep)
├── banking-adapter-web/             (HTTP adapter)
│   └── pom.xml                      (application'a depend, Spring Web dep)
└── banking-app/                     (main class, configuration)
    └── pom.xml                      (tüm adapter'lara depend)
```

Modüller arası bağımlılık yönü — dikkat et, tüm oklar domain'e doğru akar:

```mermaid
flowchart TD
    APP["banking-app"] --> WEB["banking-adapter-web"]
    APP --> PERS["banking-adapter-persistence"]
    WEB --> APPL["banking-application"]
    PERS --> APPL
    APPL --> DOM["banking-domain"]
```

**Avantaj:** Compile-time enforcement. Domain modülünde Spring import'una **derleyici izin vermez**, çünkü Spring dependency'si yok.

**Dezavantaj:** Setup biraz daha ağır. Junior için A variant'ı package-level ile başlamak yeterli.

Bu projede başlangıçta **tek module + package separation** kullanacağız. Faz 7'de microservices'e bölerken multi-module'a geçireceğiz.

---

### 8. Architectural Decision Records (ADR)

Mimari kararlarını **yazılı kayıt** olarak tutmak. `core-banking/docs/adr/` klasörü açar, her karar için bir markdown dosyası.

Template:

```markdown
# ADR-001: Hexagonal architecture seçimi

Date: 2025-MM-DD
Status: Accepted

## Context
TR bank backend rolü hedefli core-banking projesi. Domain (account, transfer, ledger) uzun ömürlü olacak. Adapter'lar (DB, mesajlaşma) değişebilir.

## Decision
Hexagonal architecture (Ports & Adapters) kullanılacak. Domain saf Java tutulacak, JPA entity'leri ayrı persistence katmanında olacak.

## Consequences
+ Domain unit test'leri mock'suz yazılır
+ JPA değişikliği domain'i etkilemez
- Mapper kodu yazma yükü
- Junior için ilk başta karmaşık
```

```admonish tip title="İpucu"
Banking şirketlerinde ADR çok yaygın — production'a giren her major karar belgelenir. Bu alışkanlığı şimdiden kazan.
```

---

### 9. Anti-pattern'ler — ne yapma

**Anti-pattern 1: "Anemic Domain Model"**

Domain class'ı sadece getter/setter'dan oluşur, hiçbir davranış yoktur. Tüm logic Service'e dağılır.

```java
// ❌ Kötü
class Account {
    private BigDecimal balance;
    public BigDecimal getBalance() { return balance; }
    public void setBalance(BigDecimal b) { balance = b; }
}

class AccountService {
    public void withdraw(Long id, BigDecimal amount) {
        Account a = repo.findById(id);
        if (a.getBalance().compareTo(amount) < 0) throw new ...;
        a.setBalance(a.getBalance().subtract(amount));
        repo.save(a);
    }
}
```

Sorun: `Account` saçma. Onun **iç tutarlılığı** dışarıdan korunmak zorunda. Başka bir servis aynı kuralı uygulamayı unutursa, bakiye negatife düşer.

Doğrusu: `account.withdraw(amount)` çağrısı, kuralı içinde tutar.

**Anti-pattern 2: "Service İçinde Repository Çağrısı + Business Logic Karışık"**

Service hem orchestration yapar, hem DB sorgusu yazar, hem matematik yapar, hem HTTP çağrısı yapar.

Çözüm: orchestration servise, kural domain'e, DB adapter'a, HTTP başka bir adapter'a.

**Anti-pattern 3: "Entity'i HTTP'ye Sızdırma"**

```java
// ❌ Kötü
@GetMapping("/{id}")
public AccountJpaEntity get(@PathVariable Long id) { ... }
```

Sorunlar: lazy loading patlaması, internal field'lar dışarı sızar, JSON formatın DB schema'na bağlanır, breaking change riski yüksek.

Çözüm: ayrı `AccountResponse` DTO'su (sonraki topic'te detaylı).

**Anti-pattern 4: "Circular Dependency Between Layers"**

```admonish warning title="Dikkat"
Domain, persistence adapter'a import yapar. Bu hexagonal'i çürütür. Domain hiçbir şeye bağımlı olmamalı.
```

---

## Önemli olabilecek araştırma kaynakları (kuralın gereği: senin için keyword, kendi bul)

- "Hexagonal Architecture" Alistair Cockburn original article
- "Get Your Hands Dirty on Clean Architecture" Tom Hombergs (kitap — küçük ve odaklı)
- "Domain-Driven Design" Eric Evans (mavi kitap — referans, baştan sona okumak şart değil)
- "Implementing Domain-Driven Design" Vaughn Vernon (kırmızı kitap — daha pratik)
- Reflectoring.io blog (Tom Hombergs) — hexagonal Spring Boot örnekleri
- "Clean Architecture" Robert C. Martin
- ArchUnit dokümantasyonu (mimariyi test etme)

---

## Mini task'ler

Bunları sırayla `~/projects/core-banking/` içinde yap. Henüz Spring Boot kurmaya gerek yok, sadece domain sınıfları yazıyoruz.

### Task 1.1.1 — `Money` value object'i yaz (30 dk)

`banking/domain/common/Money.java` oluştur. Şu kuralları sağlasın:

- `amount` (BigDecimal) ve `currency` (java.util.Currency) tutar
- Immutable (record kullanabilirsin)
- `add(Money other)` ve `subtract(Money other)` metodları
- Farklı para birimleriyle aritmetik → `CurrencyMismatchException` fırlat
- Negatif amount construction'da kabul edilebilir (refund senaryosu için) ama dikkatli ol
- `isLessThan`, `isGreaterThanOrEqual`, `isZero`, `isNegative` metodları
- `Money.zero(Currency)` static factory
- `Money.of(BigDecimal, Currency)` static factory
- Scale, currency'nin default fraction digits'i ile sınırlı (TRY: 2, JPY: 0, BHD: 3)

### Task 1.1.2 — `AccountId`, `OwnerId`, `TransferId` value object'leri (15 dk)

Hepsi `record` olarak yaz. Null kabul etmesin. UUID veya Long bazlı — seçimin senin.

### Task 1.1.3 — `Account` aggregate root'u (45 dk)

`banking/domain/account/Account.java` oluştur:

- `id` (AccountId), `ownerId` (OwnerId), `currency` (Currency), `balance` (Money) tutar
- Constructor private veya factory ile çağrılır: `Account.open(OwnerId owner, Currency currency)` → balance 0 ile başlar
- `deposit(Money amount)` ve `withdraw(Money amount)` metodları
- Currency mismatch ve insufficient funds için kendi exception'larını fırlat
- `domain/account/exception/` paketine `InsufficientFundsException`, `CurrencyMismatchException` koy
- Setter yok. Tüm değişiklik metot ile.

### Task 1.1.4 — Klasör yapısını oluştur (10 dk)

`~/projects/core-banking/` içinde **henüz boş** olsa bile şu klasörleri oluştur (Maven yapısı henüz değil, sadece package planı):

```
core-banking/
└── src/
    └── main/
        └── java/
            └── com/
                └── mavibank/
                    └── banking/
                        ├── account/
                        │   ├── domain/
                        │   ├── application/
                        │   │   ├── port/
                        │   │   │   ├── in/
                        │   │   │   └── out/
                        │   │   └── service/
                        │   └── adapter/
                        │       ├── in/
                        │       │   └── web/
                        │       └── out/
                        │           └── persistence/
                        ├── transfer/
                        │   └── ... (account ile aynı yapı)
                        └── common/
                            └── domain/
```

### Task 1.1.5 — ADR yaz (15 dk)

`core-banking/docs/adr/0001-use-hexagonal-architecture.md` oluştur. Yukarıdaki template'i kullan. Kendi kararını kendi cümlelerinle yaz.

---

## Test yazma rehberi

Bu topic'te framework olmadığı için **saf JUnit 5 + AssertJ** ile başlıyoruz. Test class'larını `src/test/java/com/mavibank/banking/...` altında entity ile aynı paket yapısında yaz.

### Test 1.1.1 — `MoneyTest`

Şu senaryoları yaz (her biri ayrı `@Test`):

1. `Money.of(100.00, TRY)` + `Money.of(50.00, TRY)` = `Money.of(150.00, TRY)` ✓
2. `Money.of(100.00, TRY).add(Money.of(50.00, USD))` → `CurrencyMismatchException` fırlatır
3. `Money.zero(TRY).isZero()` true döner
4. `Money.of(-10.00, TRY).isNegative()` true döner
5. `Money.of(100.00, TRY).isLessThan(Money.of(50.00, TRY))` false
6. `Money.of(100.123, TRY)` construction → exception (scale > 2)
7. `Money.of(100, JPY)` → scale 0 OK
8. `Money.of(100.000, BHD)` → scale 3 OK
9. `null` currency ile construction → NullPointerException veya IllegalArgumentException

AssertJ kullan:
```java
import static org.assertj.core.api.Assertions.*;

assertThat(result).isEqualTo(Money.of(new BigDecimal("150.00"), TRY));
assertThatThrownBy(() -> money1.add(money2)).isInstanceOf(CurrencyMismatchException.class);
```

### Test 1.1.2 — `AccountTest`

1. `Account.open(ownerId, TRY)` → balance 0 TRY
2. Account'a `deposit(100 TRY)` → balance 100 TRY
3. Account'a `deposit(100 USD)` → `CurrencyMismatchException` (account TRY ise)
4. Account balance 100 TRY iken `withdraw(150 TRY)` → `InsufficientFundsException`
5. Account balance 100 TRY iken `withdraw(40 TRY)` → balance 60 TRY
6. Birden fazla deposit/withdraw kombinasyonu sonrası balance doğru
7. Withdraw amount 0 ise → IllegalArgumentException (sınır durumu)

### Test 1.1.3 — Pattern öğrenmek için: Test Data Builder

Test'lerinde her sefer `Account.open(new OwnerId(UUID.randomUUID()), Currency.getInstance("TRY"))` yazmak yorucu. Test helper class'ı yaz:

```java
class AccountTestBuilder {
    private OwnerId ownerId = new OwnerId(UUID.randomUUID());
    private Currency currency = Currency.getInstance("TRY");
    private Money balance = Money.zero(currency);
    
    public static AccountTestBuilder anAccount() { return new AccountTestBuilder(); }
    public AccountTestBuilder withCurrency(String code) { 
        this.currency = Currency.getInstance(code); 
        return this; 
    }
    public AccountTestBuilder withBalance(String amount) {
        this.balance = Money.of(new BigDecimal(amount), currency);
        return this;
    }
    public Account build() {
        var account = Account.open(ownerId, currency);
        if (balance.amount().compareTo(BigDecimal.ZERO) > 0) {
            account.deposit(balance, new TransferId(UUID.randomUUID()));
        }
        return account;
    }
}
```

Kullanım:
```java
var account = anAccount().withCurrency("TRY").withBalance("100.00").build();
```

```admonish tip title="İpucu"
Bu pattern (Object Mother / Test Data Builder) banka tarafında **çok yaygın**, alışkanlık kazan.
```

---

## Claude-verify prompt

Topic'i bitirdiğinde, kodunu Claude'a verify ettir. **Sadece verify**, kod yazdırma. Aşağıdaki prompt'u kopyala, kodunu (yapıştırarak veya repo linki ile) ver:

```
Aşağıdaki Java kodum hexagonal architecture & DDD prensiplerine göre yazılmış bir 
banking domain modülü. Lütfen şu kriterlere göre değerlendir ve EKSİKLERİ söyle, 
düzeltmek için kod yazma:

1. Domain class'larında hiç framework annotation'ı (@Entity, @Component, @Autowired, 
   vb.) var mı? Olmamalı.

2. `Money` value object:
   - Immutable mi (record veya tüm field'lar final)?
   - `add` ve `subtract` farklı currency'ler için exception fırlatıyor mu?
   - Scale validation (currency'nin default fraction digits'ine göre) var mı?
   - Construction'da null kontrolü var mı?
   - `equals` ve `hashCode` doğru çalışıyor mu (record ise otomatik)?

3. `Account` aggregate root:
   - `balance` field'ı dışarıya setter ile expose ediliyor mu? (Olmamalı)
   - `deposit` ve `withdraw` metotları currency mismatch ve insufficient funds 
     durumlarını kontrol ediyor mu?
   - Constructor public mi yoksa factory method (`Account.open(...)`) var mı?
   - Anemic domain model'e kaymış mı (sadece getter/setter)?

4. Paket yapısı:
   - `domain/`, `application/`, `adapter/` ayrımı yapılmış mı?
   - `application/port/in/` ve `application/port/out/` var mı?

5. Exception'lar:
   - Domain-specific exception class'ları var mı (`InsufficientFundsException` gibi) 
     yoksa generic `RuntimeException` mu kullanılmış?

6. Test'ler:
   - `MoneyTest` ve `AccountTest` var mı?
   - AssertJ kullanılmış mı?
   - Currency mismatch ve insufficient funds için ayrı test'ler var mı?
   - Test Data Builder pattern uygulanmış mı?

7. ADR:
   - `docs/adr/0001-use-hexagonal-architecture.md` var mı?
   - Context, Decision, Consequences bölümleri yazılmış mı?

Her madde için PASS / FAIL / EKSIK olarak işaretle ve kısa açıklama yap. Kod yazıp 
düzeltme. Sadece neyin eksik/yanlış olduğunu söyle. Ben düzelteceğim.
```

---

## Tamamlama kriterleri (kendine sor)

- [ ] `Money`, `AccountId`, `OwnerId`, `TransferId`, `Account` class'larını yazdım
- [ ] Domain class'larımın hiçbirinde `@Entity` veya Spring annotation'ı yok
- [ ] `InsufficientFundsException` ve `CurrencyMismatchException` yazdım, kullanıyorum
- [ ] Paket yapısı `account/domain`, `account/application/port/{in,out}`, `account/application/service`, `account/adapter/{in/web, out/persistence}` şeklinde
- [ ] `MoneyTest` ve `AccountTest` yazdım, AssertJ kullandım
- [ ] Test Data Builder pattern'i öğrendim ve kullandım
- [ ] `docs/adr/0001-use-hexagonal-architecture.md` yazdım
- [ ] Hexagonal architecture'ın "dependency direction"ını birine 2 dakikada anlatabilirim
- [ ] Anti-pattern'leri tanıyabiliyorum (Anemic Domain Model özellikle)

Hepsi onaylı → Topic 1.2'ye geç → [02-project-setup/](../02-project-setup/README.md)

---

## Notlar (defterine yaz)

Aşağıdaki cümleleri **kendi kelimelerinle** doldur:

1. "Hexagonal architecture'ın temel prensibi ____ ve bunun banking projesinde önemi ____ çünkü ____."
2. "Bir port ve bir adapter farkı şu: ____. Driving port örneği ____, driven port örneği ____."
3. "Anemic Domain Model'in problemi ____. Bunun yerine ____ yaparım."
4. "DDD'nin Entity ve Value Object farkı ____. `Money` value object'tir çünkü ____."
5. "Aggregate root nedir, neden önemli? ____."

---

```admonish success title="Bölüm Özeti"
- Mimarinin can damarı **bağımlılık yönüdür**: hexagonal'de dış dünya çekirdeğe bağımlıdır, çekirdek dış dünyayı bilmez
- **Port** çekirdeğin tanımladığı interface, **adapter** onu bir teknolojiyle somutlaştıran koddur; driving taraf uygulamayı çağırır, driven taraf uygulama tarafından çağrılır
- Domain modeli **saf Java** kalır: `@Entity` gibi framework annotation'ları domain'e girmez, JPA entity'si ayrı tutulup mapper ile dönüştürülür
- `Money` gibi **value object'ler immutable** olur ve kendi kurallarını (currency mismatch, scale) içinde taşır; `Account` gibi **aggregate root'lar** iç tutarlılığı kendi metotlarıyla korur
- Package yapısında **feature-first** (Variant A) tercih edilir; bounded context ile eşleşir ve microservice'e bölünmeyi kolaylaştırır
- **Anemic Domain Model**'den kaç: logic getter/setter'la dışarıda değil, `account.withdraw(amount)` gibi davranış olarak domain'in içinde yaşar
```
