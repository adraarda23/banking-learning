# Topic 1.1 — Mimari: Hexagonal, Ports & Adapters, Layered

```admonish info title="Bu bölümde"
- Layered ve hexagonal mimarilerin farkını ve **bağımlılık yönünün** neden kritik olduğunu öğreneceksin
- Port ve adapter kavramlarını (driving / driven) banking örnekleri üzerinden tanıyacaksın
- DDD'nin temel yapı taşlarını göreceksin: value object, entity, aggregate root, domain event
- Feature-first package yapısını ve ADR yazma alışkanlığını kazanacaksın
- Anemic Domain Model başta olmak üzere yaygın anti-pattern'leri tanımayı öğreneceksin
```

## Hedef

"Bir Java backend uygulamasının kodunu neye göre organize edersin?" sorusuna derinlemesine cevap vermek — ve banking domain'inde bu kararın uygulamayı neden "taşınabilir ve test edilebilir" yaptığını görmek.

## Süre

Okuma: 1.5-2 saat • Kendini sına: 30 dk • Pratik (opsiyonel): 2-2.5 saat

## Önbilgi

- Spring Boot temel (controller, service, repository annotation'larını gördün)
- Maven hakkında genel fikir

---

## Kavramlar

### 1. Neden mimari kararları önemli — banking perspektifi

Bankada bir hesap servisi 10 yıl yaşar. O sürede veritabanı değişir (Oracle → PostgreSQL), framework değişir (Spring 5 → 6 → ?), arayüz değişir (SOAP → REST → gRPC). Değişmeyen tek şey **domain logic**: bakiye negatife düşemez, her transfer iki kayıt üretir, faiz hesabı kesin kurallara tabidir.

Yanlış mimaride teknoloji değişikliği "uygulamayı sıfırdan yaz" demektir. Doğru mimaride teknoloji çevre katmanlarda değişir, çekirdek sağlam kalır.

Bu yüzden mimarinin can damarı **bağımlılık yönü**: çekirdek kod çevreye mi bağlı, yoksa çevre çekirdeğe mi? Bu bölümdeki her şey bu tek sorunun etrafında dönüyor.

---

### 2. Layered Architecture (3-katmanlı / N-tier)

Spring Boot tutorial'larının %95'i bu yapıdadır, o yüzden buradan başlıyoruz. Fikir basit: kodu yatay katmanlara böl, her katman bir alttakini çağırsın.

```mermaid
flowchart TD
    C["Controller - HTTP isteklerini karşılar"] --> S["Service - iş kuralları"]
    S --> R["Repository - DB ile konuşur"]
    R --> DB["Database"]
```

Bağımlılık yönü yukarıdan aşağı: Controller → Service → Repository → DB. Hızlı başlanır, junior'ın aşina olduğu yapıdır.

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
```

Controller'ın tek işi HTTP isteğini karşılayıp servise devretmek — buraya kadar sorun yok. Asıl kritik nokta bir alt katmanda:

```java
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
```

Service iş kurallarını taşıyor ama enjekte ettiği `AccountRepository` doğrudan bir JPA interface'i:

```java
// Repository — DB'i bilir
interface AccountRepository extends JpaRepository<Account, Long> { }
```

Peki sorun ne? Dikkat et: Service, JPA repository'e bağımlı. Yani iş kurallarını taşıyan katman, **persistence teknolojisini tanıyor**. Banking'de bu şu tuzaklara dönüşür:

```admonish warning title="Dikkat — Layered'ın banking için tuzakları"
1. **Domain modeli framework'e yapışır.** `Account` entity'si `@Entity`, `@Id`, `@Column` ile JPA'ya bağlı. JPA bırakırsan domain modelini yeniden yazarsın.

2. **Service'i unit test etmek zor.** Repository Spring bean olduğu için mock veya `@DataJpaTest` gerekir; saf Java birim testi yazamazsın.

3. **İş kuralları her yere dağılır.** Bakiye kontrolü Service'te, validation Controller'da, audit trigger Repository'de — kim hangi kuralı uyguluyor karışır.

4. **Dış sistem entegrasyonu sızar.** External bank API'sini servise enjekte ettiğin anda domain logic ile network detayları birbirine girer.
```

---

### 3. Hexagonal Architecture (Ports & Adapters)

Şimdi bağımlılık yönünü ters çevirelim. Alistair Cockburn'ün 2005'te önerdiği fikir: **çekirdek domain logic ortada durur**, tüm dış dünya (DB, HTTP, Kafka) çevrede. <mark>Çekirdek dış dünyaya bağımlı değildir — dış dünya çekirdeğe bağımlıdır.</mark>

İki anahtar kavram var:

- **Port:** Çekirdeğin dış dünyayla konuşma **kontratı** — bir interface, ve onu çekirdek tanımlar.
  - *Driving port:* "Beni dışarıdan çağırın" — use case interface'i
  - *Driven port:* "Bana bunları sağlayın" — repository, message sender interface'i
- **Adapter:** Port'u gerçek bir teknolojiyle somutlaştıran kod.
  - *Driving adapter:* HTTP controller, gRPC handler, Kafka consumer — uygulamayı **çağıran** taraf
  - *Driven adapter:* JPA implementation, Kafka producer, SMS gateway client — uygulamanın **çağırdığı** taraf

Resmin tamamı: driving taraf solda çekirdeği çağırır, driven taraf sağda çekirdeğin tanımladığı port'ları implement eder.

```mermaid
flowchart LR
    subgraph DRIVING["Driving taraf"]
        HTTP["HTTP Controller"]
    end
    subgraph CORE["Application Core"]
        UC["Use Case interface - driving port"] --> SVC["Application Service"]
        SVC --> DOM["Domain Model - saf Java"]
        SVC --> REPO["AccountRepository - driven port"]
        SVC --> NOTIF["NotificationSender - driven port"]
    end
    subgraph DRIVEN["Driven taraf"]
        JPA["JPA Adapter"] --> PG["PostgreSQL"]
        KAFKA["Kafka Adapter"]
    end
    HTTP --> UC
    JPA -. "implement eder" .-> REPO
    KAFKA -. "implement eder" .-> NOTIF
```

Kesikli oklara dikkat: JPA adapter sağda durur ama ok **çekirdeğe doğru** akar. Çekirdek bir interface tanımlar, adapter onu implement eder; çekirdek adapter'ın varlığını bile bilmez. Buna **dependency inversion** denir ve hexagonal'in bütün sırrı budur.

**Banking domain örneği:**

Merkezden başlayalım. Domain sınıfı saf Java — iş kuralı (insufficient funds) metodun içinde yaşıyor:

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
```

Çekirdek, dış dünyadan ne istediğini iki port ile ilan eder: driven port `AccountRepository` ("bana persistence sağlayın") ve driving port `TransferMoneyUseCase` ("beni bu kontrat üzerinden çağırın"):

```java
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
```

Application service use case'i implement eder ve persistence'a **sadece port üzerinden** ulaşır — constructor'a gelen `AccountRepository`'nin arkasında JPA mı MongoDB mi olduğunu bilmez:

```java
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
```

Driven adapter, çekirdeğin tanımladığı port'u JPA ile somutlaştırır; mapper, JPA entity'si ile domain nesnesi arasında çeviri yapar:

```java
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
```

`@Entity` annotation'ı domain'e değil, sadece persistence katmanında yaşayan ayrı bir sınıfa yapışır:

```java
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
```

Son parça driving adapter: HTTP controller use case **port'unu** çağırır — somut service sınıfını değil, interface'i tanır:

```java
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

Bir transfer isteğinin uçtan uca akışı:

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

Controller sadece port interface'ini bilir, service persistence'a sadece port üzerinden ulaşır — hiçbir ok domain'den dışarı çıkmaz.

**Kazandıkların:**

1. **Domain saf Java.** <mark>Spring ve JPA olmadan çalışır; unit test'ler mock'sız yazılır.</mark>
2. **Adapter değiştirilebilir.** JPA → MongoDB geçişinde sadece adapter değişir.
3. **Test piramidi sağlıklı:** domain unit test (hızlı, çok), adapter test (TestContainers), integration test (az).
4. **Açık kontrat:** port'lar, "uygulamanın dış dünyadan ihtiyaçları neler" sorusunun tek cevabı.

**Bedeli:** mapper kodu yazarsın (daha fazla kod), ilk bakışta over-engineering gibi gelir ve küçük bir CRUD admin paneli için gerçekten de ağırdır.

```admonish tip title="İpucu — banking için neden değer"
Domain kuralları (negative balance, double-entry invariant, faiz formülü) **on yıllarca yaşayacak**. Teknoloji 3 yılda bir değişir. Hexagonal seni yaşatır.
```

---

### 4. Onion Architecture & Clean Architecture (kısaca)

Bu isimleri iş ilanlarında ve mülakatlarda göreceksin, paniğe gerek yok:

- **Onion Architecture** (Jeffrey Palermo): aynı fikir, "soğan halkaları" görseliyle — merkezde Domain, sonra Application Services, sonra Infrastructure.
- **Clean Architecture** (Uncle Bob): katmanları "Entities → Use Cases → Interface Adapters → Frameworks & Drivers" diye adlandırır ve **"The Dependency Rule"** der: bağımlılıklar sadece dışarıdan içeri akar.

Üçü de pratikte aynı yapıyı önerir: **domain merkez, framework çevre, bağımlılık içe doğru.**

---

### 5. Domain-Driven Design (DDD) — mimari değil ama bağlantılı

Hexagonal sana "domain'i ortaya koy" der ama o domain'i **nasıl modelleyeceğini** söylemez. Orada DDD devreye girer (Eric Evans, 2003). **DDD** bir mimari değil, modelleme yaklaşımıdır — hexagonal ile çok iyi eşleşir.

Faz 1'de bilmen gereken parçaları:

- **Ubiquitous language:** Kodda iş diliyle aynı kelimeleri kullan. Bankacı "EFT" diyorsa kod da `Eft` der, `Transfer` değil.
- **Bounded context:** Büyük domain'i parçalara böl — Account context, Card context, Transfer context. Her birinin kendi modeli olabilir.
- **Aggregate:** Bir tutarlılık sınırı. Aggregate root (örn. `Account`) kendi child'larını (`JournalEntry`'leri) yönetir ve **dışarıdan yegane giriş noktasıdır**.
- **Entity vs Value Object:** *Entity*'nin kimliği vardır (`Account`, `Customer`) — iki müşterinin adı aynı olsa da farklı entity'dir. *Value object* değeriyle tanımlanır — her `Money(100.00, TRY)` birbirine eşittir ve **immutable** olmalıdır.
- **Domain event:** Domain'de bir şey olduğunda yayılan mesaj: `MoneyTransferred`, `AccountOpened`.

**Banking örneği — value object:**

`Money`'nin bütün doğruluğu construction anında garanti altına alınır — compact constructor null ve scale kontrolü yapar, geçersiz bir `Money` nesnesi hiç var olamaz:

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount == null || currency == null) throw new IllegalArgumentException();
        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("Too many decimal places for " + currency);
        }
    }
```

Aritmetik metotlar mevcut nesneyi değiştirmez, **yeni bir `Money` döndürür** — immutability tam olarak budur:

```java
    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
    }
```

Karşılaştırma dahil her operasyon önce `requireSameCurrency` ile para birimi eşitliğini zorlar — kural tek yerde tanımlı, her yerden zorunlu:

```java
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

<details>
<summary>Tam kod: Money (~30 satır)</summary>

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

</details>

Dikkat: currency mismatch kuralı `Money`'nin **içinde** yaşıyor. <mark>Onu kullanan hiçbir servis bu kontrolü unutamaz.</mark>

**Banking örneği — aggregate root:**

`Account` durumunu private tutar ve her durum değişikliğinde bir domain event biriktirir. `deposit` önce invariant'ı (currency eşitliği) kontrol eder:

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
```

`withdraw` ek olarak insufficient-funds kuralını uygular — "bakiye negatife düşemez" invariant'ı setter'la dışarıdan değil, davranışın kendisiyle korunur:

```java
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
```

Biriken event'leri application service tek seferde çeker: liste kopyalanır ve temizlenir, böylece aynı event iki kez yayınlanmaz:

```java
    public List<DomainEvent> pullEvents() {
        var pulled = List.copyOf(events);
        events.clear();
        return pulled;
    }
    // ...
}
```

<details>
<summary>Tam kod: Account (~34 satır)</summary>

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

</details>

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

### 7. Maven multi-module ile fiziksel ayrılık (opsiyonel ama güçlü)

Package ayrımı bir söz; **multi-module** ise **derleyicinin zorladığı** bir sözleşme. Katmanları ayrı Maven module'lerine bölersin:

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

Modüller arası bağımlılık yönü — tüm oklar domain'e doğru akar:

```mermaid
flowchart TD
    APP["banking-app"] --> WEB["banking-adapter-web"]
    APP --> PERS["banking-adapter-persistence"]
    WEB --> APPL["banking-application"]
    PERS --> APPL
    APPL --> DOM["banking-domain"]
```

Avantajı compile-time enforcement: domain modülünde Spring dependency'si olmadığı için derleyici Spring import'una **izin vermez**. Bedeli biraz daha ağır setup — bu yüzden başlangıçta **tek module + package separation** kullanacağız, Faz 7'de microservices'e bölerken multi-module'a geçeceğiz.

---

### 8. Architectural Decision Records (ADR)

Altı ay sonra "biz neden hexagonal seçmiştik?" diye soran olacak — belki de o kişi sen olacaksın. **ADR**, mimari kararları yazılı kayıt altına alma alışkanlığıdır: `core-banking/docs/adr/` klasörü, her karar için bir markdown dosyası.

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

**Anti-pattern 1: Anemic Domain Model**

Domain class'ı sadece getter/setter'dan oluşur, hiçbir davranışı yoktur; tüm logic Service'e dağılır.

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

Sorun: `Account`'un iç tutarlılığı dışarıdan korunmak zorunda. Başka bir servis aynı kontrolü unutursa bakiye negatife düşer. Doğrusu: `account.withdraw(amount)` — <mark>kural nesnenin içinde yaşar</mark>.

**Anti-pattern 2: Her şeyi yapan Service**

Service hem orchestration yapar, hem DB sorgusu yazar, hem matematik yapar, hem HTTP çağrısı atar. Çözüm: orchestration servise, kural domain'e, DB adapter'a, HTTP başka bir adapter'a.

**Anti-pattern 3: Entity'i HTTP'ye sızdırma**

```java
// ❌ Kötü
@GetMapping("/{id}")
public AccountJpaEntity get(@PathVariable Long id) { ... }
```

Sonuçları: lazy loading patlaması, internal field'ların dışarı sızması, JSON formatının DB schema'sına bağlanması ve yüksek breaking change riski. Çözüm: ayrı `AccountResponse` DTO'su (sonraki topic'te detaylı).

**Anti-pattern 4: Katmanlar arası circular dependency**

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

## Kendini Sına

Aşağıdaki sorular mülakat tarzındadır. Önce kendi cevabını sesli veya yazılı ver, sonra cevabı açıp karşılaştır.

**S1. Layered mimaride `AccountService`'in doğrudan `JpaRepository`'e bağımlı olması banking'de hangi somut sorunlara yol açar?**

<details>
<summary>Cevabı göster</summary>

İş kurallarını taşıyan katman persistence teknolojisini tanımış olur ve bunun dört somut bedeli vardır: (1) domain modeli `@Entity`, `@Id` gibi annotation'larla JPA'ya yapışır — teknoloji değişiminde domain'i yeniden yazarsın; (2) Service'i unit test etmek için mock veya `@DataJpaTest` gerekir, saf Java birim testi yazamazsın; (3) iş kuralları katmanlara dağılır (validation Controller'da, bakiye kontrolü Service'te, audit Repository'de); (4) dış sistem entegrasyonu domain logic'in içine sızar. Bankada bir hesap servisi 10 yıl yaşar ve o sürede DB/framework değişir — bu bağımlılık "uygulamayı sıfırdan yaz" riskine dönüşür.

</details>

**S2. "Hexagonal mimaride bağımlılık yönü ters çevrilir" ne demek? JPA adapter örneğiyle açıkla.**

<details>
<summary>Cevabı göster</summary>

Çekirdek, ihtiyacını bir interface (port) olarak kendisi tanımlar: `AccountRepository`. `JpaAccountRepository` adapter'ı bu interface'i **implement eder** — yani derleme bağımlılığı adapter'dan çekirdeğe doğru akar. Adapter çekirdeği import eder ama çekirdek adapter'ın varlığını bile bilmez. Buna **dependency inversion** denir. Sonucu: JPA'yı MongoDB ile değiştirmek sadece yeni bir adapter yazmaktır; çekirdek koduna ve domain kurallarına dokunulmaz.

</details>

**S3. Port ile adapter farkı nedir? Driving ve driven taraflar için birer banking örneği ver.**

<details>
<summary>Cevabı göster</summary>

Port, çekirdeğin dış dünyayla konuşma **kontratıdır** (çekirdeğin tanımladığı interface); adapter ise o kontratı gerçek bir teknolojiyle somutlaştıran koddur. Driving port "beni dışarıdan çağırın" der — örnek: `TransferMoneyUseCase`; onu çağıran driving adapter HTTP `TransferController`'dır (gRPC handler veya Kafka consumer da olabilir). Driven port "bana bunları sağlayın" der — örnek: `AccountRepository`, `NotificationSender`; onları implement eden driven adapter'lar `JpaAccountRepository` ve Kafka producer'dır. Kısaca: driving taraf uygulamayı çağırır, driven taraf uygulama tarafından çağrılır.

</details>

**S4. Entity ile value object farkı nedir? `Money` neden immutable bir value object ve currency-mismatch kontrolü neden `Money`'nin içinde yaşamalı?**

<details>
<summary>Cevabı göster</summary>

Entity kimliğiyle tanımlanır — iki müşterinin adı aynı olsa da farklı entity'dirler (`Account`, `Customer`). Value object değeriyle tanımlanır — her `Money(100.00, TRY)` birbirine eşittir. `Money` immutable olmalıdır çünkü `add`/`subtract` mevcut nesneyi değiştirmeyip yeni bir `Money` döndürür; böylece paylaşılan bir bakiye nesnesi yan etkiyle bozulamaz. Kural içeride olunca da onu kullanan hiçbir servis kontrolü unutamaz:

```java
public Money add(Money other) {
    requireSameCurrency(other);   // kural tek yerde, her operasyonda zorunlu
    return new Money(amount.add(other.amount), currency);
}
```

</details>

**S5. Aggregate root nedir? `withdraw` logic'inin service yerine `Account`'un içinde olması neden kritik — bunun ihlaline hangi anti-pattern denir?**

<details>
<summary>Cevabı göster</summary>

Aggregate root, bir tutarlılık sınırının **yegane giriş noktasıdır**; child nesnelerini (örn. `JournalEntry`'leri) kendisi yönetir. `account.withdraw(amount)` insufficient-funds ve currency-mismatch invariant'larını nesnenin içinde uygular — hiçbir çağıran kuralı atlayamaz. Kuralın `getBalance`/`setBalance` üzerinden serviste uygulanması **Anemic Domain Model** anti-pattern'idir: domain sınıfı davranışsız bir veri torbasına dönüşür ve başka bir servis aynı kontrolü unutursa bakiye negatife düşer.

</details>

**S6. Domain sınıfına `@Entity` koymak yerine ayrı bir `AccountJpaEntity` + mapper tutmanın kazancı ve bedeli nedir?**

<details>
<summary>Cevabı göster</summary>

Kazanç: domain saf Java kalır — Spring/JPA olmadan derlenir ve unit test'ler mock'suz yazılır; persistence teknolojisi değişince sadece adapter, JPA entity ve mapper değişir; test piramidi sağlıklı olur (çok sayıda hızlı domain testi, az sayıda integration testi). Bedel: mapper kodu yazma yükü ve küçük bir CRUD admin paneli için gerçek bir over-engineering riski. Banking'de tercih nedeni: domain kuralları on yıllarca yaşar, teknoloji ~3 yılda bir değişir — ayrım seni yaşatır.

</details>

**S7. Package yapısında feature-first (Variant A) yaklaşımını layer-first'e neden tercih ediyoruz?**

<details>
<summary>Cevabı göster</summary>

Üç sebep: (1) bir feature'a dokunmak istediğinde domain, service ve adapter kodunun tamamı aynı klasörün altında; (2) ileride microservice'e bölmek gerektiğinde feature'ı tek klasör halinde çıkarırsın; (3) DDD'nin bounded context fikriyle birebir eşleşir — `account` ve `transfer` kendi modelleriyle ayrı yaşar. Layer-first'te ise tek bir feature değişikliği üç ayrı package ağacına dokunmayı gerektirir.

</details>

**S8. ADR nedir, hangi bölümlerden oluşur ve neden yazılır?**

<details>
<summary>Cevabı göster</summary>

Architectural Decision Record — mimari bir kararın markdown dosyası olarak kayıt altına alınmasıdır (`docs/adr/` klasöründe, karar başına bir dosya). Tipik bölümleri: başlık, tarih ve status; **Context** (kararın alındığı ortam ve kısıtlar), **Decision** (ne seçildi), **Consequences** (artılar ve eksiler — dürüstçe, eksiler dahil). Amacı: altı ay sonra "biz neden hexagonal seçmiştik?" sorusunun cevabının kaybolmaması. Banking şirketlerinde production'a giren her major karar bu şekilde belgelenir.

</details>

---

## Tamamlama kriterleri (kendine sor)

- [ ] "Kendini Sına" bölümündeki tüm soruları cevaba bakmadan yanıtlayabildim
- [ ] Hexagonal architecture'ın "dependency direction"ını birine 2 dakikada anlatabilirim
- [ ] Port ile adapter farkını driving/driven örnekleriyle açıklayabiliyorum
- [ ] Value object ile entity farkını ve `Money`'nin neden immutable olduğunu açıklayabiliyorum
- [ ] Anti-pattern'leri tanıyabiliyorum (Anemic Domain Model özellikle)
- [ ] (Pratik yaptıysan) `Money` ve `Account`'u framework annotation'sız yazdım, `MoneyTest`/`AccountTest` AssertJ ile geçiyor, ADR'ımı yazdım

Hepsi onaylı → Topic 1.2'ye geç → [02-project-setup/](../02-project-setup/index.md)

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

---

## Pratik yapmak istersen

Kavramları koda dökmek en kalıcı öğrenme yöntemidir. `~/projects/core-banking/` altında `Money` value object'ini, `AccountId`/`OwnerId`/`TransferId` record'larını ve `Account` aggregate root'unu **framework'süz saf Java** ile yaz (Kavramlar bölümündeki örnekleri referans al, `docs/adr/0001-use-hexagonal-architecture.md` için ADR template'ini kullan). Sonra aşağıdaki rehberle test et ve Claude'a verify ettir.

<details>
<summary>Test yazma rehberi</summary>

Bu topic'te framework olmadığı için **saf JUnit 5 + AssertJ** ile başlıyoruz. Test class'larını `src/test/java/com/mavibank/banking/...` altında entity ile aynı paket yapısında yaz.

**Test 1 — `MoneyTest`**

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

**Test 2 — `AccountTest`**

1. `Account.open(ownerId, TRY)` → balance 0 TRY
2. Account'a `deposit(100 TRY)` → balance 100 TRY
3. Account'a `deposit(100 USD)` → `CurrencyMismatchException` (account TRY ise)
4. Account balance 100 TRY iken `withdraw(150 TRY)` → `InsufficientFundsException`
5. Account balance 100 TRY iken `withdraw(40 TRY)` → balance 60 TRY
6. Birden fazla deposit/withdraw kombinasyonu sonrası balance doğru
7. Withdraw amount 0 ise → IllegalArgumentException (sınır durumu)

**Test 3 — Pattern öğrenmek için: Test Data Builder**

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

> **İpucu:** Bu pattern (Object Mother / Test Data Builder) banka tarafında **çok yaygın**, alışkanlık kazan.

</details>

<details>
<summary>Claude-verify prompt</summary>

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

</details>
