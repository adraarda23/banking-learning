# Topic 7.1 — DDD Strategic Design: Bounded Context, Context Map, Aggregate

## Hedef

Faz 1'de tek bir `core-banking` monolitinde gördüğün domain'i (account, transfer, ledger, fraud, notification) **stratejik DDD** lensiyle yeniden okumak. Bounded context'lerin sınırlarını bulmak, aralarındaki ilişkileri tipleri ile adlandırmak (context map), aggregate root'ların gerçek **transaction boundary** anlamını kavramak. Bu topic, Faz 7'nin **mimari temelidir** — buradaki kararlar 02-08 topic'lerinin nasıl çıkacağını belirler.

## Süre

Okuma: 2.5 saat • Kâğıt-kalem context map: 2 saat • Mini task'ler: 3 saat • Test: 1 saat • Toplam: ~8.5 saat (3-4 gün)

## Önbilgi

- Faz 1.1 — Hexagonal architecture bitirilmiş (port/adapter ayrımı net)
- Faz 2 — JPA + transaction yönetimi pratik var (`@Transactional` kullandın)
- Eric Evans'ın DDD'sinden Faz 1'de **kavramları** gördün — burada **stratejik tarafa** geçiyoruz
- "Bounded context" terimini duydun; bu topic'te onu **yaşayacaksın**

---

## Kavramlar

### 1. Stratejik DDD vs Taktiksel DDD — fark

Faz 1'de **taktiksel** DDD'ye dokunduk: entity, value object, aggregate, domain event, repository. Bunlar **tek bir bounded context içinde** kod yazma kuralları.

**Stratejik DDD** ise farklı bir seviye:

- Büyük domain'i **nasıl parçalayacaksın**?
- Parçalar (bounded context) arasında **nasıl ilişki kuracaksın**?
- Hangi parça hangi takıma ait olur (Conway's Law)?
- Hangi parça **microservice** olarak ayrılır?

Microservice mimarisi **stratejik DDD'nin sonucu**. Önce bounded context'leri bul, sonra "bu context'ler hangi servisler olur?" sorusunu cevapla.

> Eric Evans (DDD Blue Book, 2003): "Strategic design is about decisions that are global and structural. Tactical design is about expressing the model in code."

---

### 2. Bounded Context — tanım ve sınırlar

**Bounded context:** Bir modelin **anlamlı, tutarlı, sınırlı** olduğu bağlam. Sınırın ötesinde aynı kelime farklı anlama gelebilir.

Banking örneği — `Account` kelimesi:

| Bounded context | "Account" ne demek |
|---|---|
| **Core Banking (vadesiz hesap)** | Bakiyesi olan, debit/credit edilen ledger objesi. ID + currency + balance + status. |
| **Authentication / Kimlik** | Login bilgisi. username + password hash + MFA secret + last login. |
| **CRM / Customer 360** | Müşterinin "ana hesabı" — tüm hesapların referansını taşır, marketing tagi var. |
| **Card Management** | Kart ile ilişkili "card account" — limit, dönem, ödenmemiş tutar. |
| **Loan / Kredi** | Kredi hesabı — anapara, faiz, ödeme planı. |

**Aynı kelime, beş farklı anlam.** Bu farklılıkları **tek bir model**de birleştirmeye çalışırsan:

- `Account` class'ı 47 field'lı dev bir nesne olur
- "Account credit dedi → loan ödemesi mi yapıyoruz, vadesiz hesaba para mı yatırıyoruz?" sorusu netleşemez
- Her takım `Account` üzerinde değişiklik yaparken birbirine çarpar

**Bounded context çözümü:** Her birinin **kendi `Account`'u** olsun, **kendi terminolojisi** olsun, **kendi modeli** olsun. Birbirleriyle konuşurken **çevirmen (mapping/ACL)** olsun.

#### Bounded context'in pratik göstergeleri

Bir kodun ayrı bir bounded context olduğunu nereden anlarsın:

1. **Ubiquitous language farklı.** Bir takım "transfer", diğer takım "havale", üçüncüsü "EFT" diyor.
2. **Aynı kelimenin field'ları/davranışı farklı.** `Customer` bir tarafta `kyc_status` taşır, diğer tarafta `marketing_segment`.
3. **Lifecycle bağımsız.** Bir context değişiyor diye diğeri **deploy** edilmek zorunda olmamalı.
4. **Sahiplik farklı.** Farklı takım, farklı PO, farklı SLA.
5. **Veri tutarlılığı sınırı.** Bir transaction'ın bir context'in içinde tamamlanması mantıklı; context'ler arası `@Transactional` zaten çalışmaz.

Bu kriterlerin **en az 3'üne "evet" diyebiliyorsan**, ayrı bir bounded context olma kuvvetli ihtimal.

---

### 3. Banking için bounded context haritası

Faz 7'de odaklandığımız 4 servis + onların etrafındaki bounded context'ler:

```
                       ┌──────────────────┐
                       │   Identity /     │
                       │   Authentication │  ← out of scope (Faz 8)
                       └──────────────────┘
                                ↑
                                │ user identity
                                │
┌─────────────────┐    ┌────────┴───────────┐    ┌─────────────────┐
│   Customer      │    │   Account / Core   │    │     Transfer    │
│   (CRM, 360)    │←──→│   Banking Ledger   │←──→│   Orchestration │
│   out of scope  │    │  (account-service) │    │ (transfer-svc)  │
└─────────────────┘    └────────┬───────────┘    └────────┬────────┘
                                │                          │
                                ↓                          ↓
                       ┌──────────────────┐    ┌──────────────────┐
                       │     Card         │    │      Fraud       │
                       │   Management     │    │   Risk Engine    │
                       │   out of scope   │    │  (fraud-service) │
                       └──────────────────┘    └──────────────────┘
                                                        │
                                                        ↓
                                              ┌──────────────────┐
                                              │   Notification   │
                                              │   Delivery       │
                                              │ (notif-service)  │
                                              └──────────────────┘
```

**Faz 7'de implement edilecek 4 context:**

1. **Account / Core Banking Ledger** — vadesiz hesap, balance, double-entry ledger
2. **Transfer Orchestration** — para hareketinin orchestration'ı, idempotency, saga
3. **Fraud Risk** — kural motoru, skor, decision (APPROVE/REJECT/REVIEW)
4. **Notification Delivery** — SMS/email/push template + gönderim

**Out of scope (faz 8+):**
- Customer / KYC / 360
- Card management
- Loan / Kredi
- Authentication / IAM
- Reporting / Data warehouse

---

### 4. Ubiquitous Language — bounded context içinde

Her bounded context'in **kendi sözlüğü** olmalı. Banking'de gerçek hayat örneği:

#### Account / Core Banking dilinde:

- **Hesap (Account):** Para tutan ledger objesi. ID, currency, balance, status (ACTIVE/FROZEN/CLOSED).
- **Bakiye (Balance):** Hesabın anlık parası. `Money` value object.
- **Hesap hareketi (Journal Line):** Bir debit veya credit kaydı. Hep çift gelir.
- **Hesap defteri (Journal Entry):** Bir transaction'ın iki tarafını (DEBIT + CREDIT) birleştiren atomic kayıt.
- **Hesap durumu (Status):** ACTIVE, FROZEN (geçici dondurma), CLOSED (kalıcı kapama).

#### Transfer Orchestration dilinde:

- **Transfer:** İki hesap arasında para hareketi talebi. ID, fromAccount, toAccount, amount, currency, status.
- **Status:** PENDING, FRAUD_CHECK, EXECUTED, REJECTED, COMPENSATING, COMPENSATED, FAILED.
- **Saga:** Transfer'in yaşam döngüsünü yöneten state machine.
- **Compensation:** Başarısız bir adımı geri alma — reverse journal entry.
- **Idempotency Key:** Aynı transfer'in iki kez işlenmesini engelleyen client-sağlayan UUID.
- **Hold:** Transfer onayı sırasında bakiyenin "kilitlenmesi" — formal kayıt değil, ama mantıksal.

#### Fraud Risk dilinde:

- **Risk Score:** 0-100 arası, transfer'in riskini özetler.
- **Rule:** Bir karar kuralı — "günlük 50.000 TRY üstü → MANUAL REVIEW".
- **Decision:** APPROVE, REJECT, REVIEW.
- **Velocity:** Belirli bir zaman aralığında işlem sayısı (örn. son 5 dk'da 10 transfer).
- **Blacklist:** Engellenmiş hesap/IBAN/IP listesi.
- **Whitelist:** Müşteri tarafından beyan edilmiş güvenilir alıcılar.

#### Notification dilinde:

- **Channel:** SMS, EMAIL, PUSH (mobile bildirimi), IN_APP.
- **Template:** Parametrize edilmiş mesaj — "{name}, {amount} {currency} transfer onaylandı".
- **Delivery Attempt:** Bir kanaldan bir mesajı gönderme denemesi, başarı/başarısızlık + zaman.
- **Quiet Hours:** Müşterinin bildirim almak istemediği zaman aralığı.
- **Opt-out:** Müşterinin tamamen ya da kanal bazlı vazgeçmesi.

**Dikkat:** "Transfer" kelimesi Account context'te de geçer (bir journal entry'nin sebebi), ama oradaki "transfer" sadece bir **referans** (transferId UUID). Account, transfer'in **business logic'ini** bilmez. Bu sınır.

---

### 5. Context Map — ilişki tipleri

İki bounded context arasındaki ilişkiyi tanımlayan **9 standart pattern** var (Eric Evans). Microservice tasarlarken bu tipleri **bilinçli seç**:

```
Context A ─────[relationship type]─────► Context B
```

#### 5.1. Shared Kernel (Paylaşılan Çekirdek)

İki context **aynı küçük model parçasını paylaşır**. Sıkı bağlanma.

```
[Context A] ─┐
             ├──> [Shared Kernel]
[Context B] ─┘
```

**Banking örneği:** `Money`, `Currency`, `AccountId` (value object'ler) tüm servislerde aynı şekilde anlamlı. Bir `banking-common` Maven module/JAR olarak shared kernel.

**Risk:** Shared kernel her iki takım da değişikliği koordine etmek zorunda. Küçük tutulmalı.

**Banking pratiği:** Sadece **stabil value object'ler** (`Money`, `Currency`, ErrorCode enum'u, `IdempotencyKey`). Domain logic asla shared kernel'da olmamalı.

#### 5.2. Customer-Supplier

Bir context **müşteri** (downstream), diğeri **tedarikçi** (upstream). Tedarikçi, müşterinin ihtiyaçlarını dikkate alır.

```
[Customer / downstream] ◄────── [Supplier / upstream]
                                      ↑
                                tedarikçi müşteriyi
                                önemser
```

**Banking örneği:** `account-service` (supplier), `transfer-service` (customer). Transfer-service, account-service'in API'sini kullanır. Account-service breaking change yaparken transfer-service'i **uyarır**, takvim koordine eder.

**İşaret:** Aynı şirkette iki takım, iyi iletişim, ortak sprint koordinasyonu.

#### 5.3. Conformist

Müşteri context, tedarikçinin modelini **olduğu gibi** kabul eder. Adapter yok.

```
[Customer] ════adapts to════► [Supplier model]
```

**Banking örneği:** Bir TR bankası **MASAK** (Mali Suçları Araştırma Kurulu) raporlama sistemine veri yollar. MASAK'ın formatı bellidir, banka bunu **kabul etmek zorunda**, kendi şartlarını dayatamaz. Banka kodunda MASAK schema'sı doğrudan kullanılır.

**Risk:** Tedarikçi değişirse müşteri patlar. Kaçınılabilirse kaçın.

#### 5.4. Anti-Corruption Layer (ACL) — banking için çok önemli

Müşteri context, tedarikçinin "kirli" modelini kendine sokmaz. Arada **çevirmen katman** kurar.

```
[Clean Context] ◄──[ACL]──► [Legacy / dirty model]
```

**Banking örneği — KLASİK:**

TR bankalarında **legacy core banking** (COBOL mainframe, AS/400, Hogan, Bantas vb.) hâlâ canlı. Yeni microservice'ler legacy ile konuşmak zorunda ama legacy modeli berbat (her field 12 karakter, hesap numarası "ACCT-" prefix'li, currency "TR" yerine "TRL" diyor).

Çözüm — `legacy-adapter` ACL:

```java
// === transfer-service domain (clean) ===
public class Transfer {
    private final TransferId id;
    private final Money amount;       // BigDecimal + Currency
    private final AccountId from;
    private final AccountId to;
    // ...
}

// === ACL — legacy çevirmeni ===
@Component
class LegacyHogonTransferAdapter {
    private final HogonClient hogonClient;
    private final HogonCurrencyMapper currencyMapper;  // "TR" → "TRY"
    private final HogonAccountMapper accountMapper;    // "ACCT-12345" → AccountId
    
    public LegacyTransferResponse send(Transfer transfer) {
        // Bizim clean domain'imizden legacy formatına çevir
        HogonTransferRequest legacyReq = new HogonTransferRequest();
        legacyReq.setFromAcct(accountMapper.toLegacy(transfer.getFrom()));  // "ACCT-12345"
        legacyReq.setToAcct(accountMapper.toLegacy(transfer.getTo()));
        legacyReq.setAmt(transfer.getAmount().amount().toPlainString());     // string
        legacyReq.setCcy(currencyMapper.toLegacy(transfer.getAmount().currency()));  // "TRL"
        legacyReq.setTrxDt(LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd")));
        
        var legacyResp = hogonClient.executeTransfer(legacyReq);
        
        // Legacy response'u clean domain'e çevir
        return new LegacyTransferResponse(
            new TransferReference(legacyResp.getTrxRef()),
            mapStatus(legacyResp.getStatusCd()),  // "00"→SUCCESS, "01"→PENDING
            Instant.parse(legacyResp.getProcDt())
        );
    }
}
```

**Mid-level mülakatta sorulur:** "ACL ne zaman lazım?" Cevap: legacy entegrasyon, üçüncü parti API (BKM, FAST, Visa), kendi başka bounded context'in çirkin/değişken modeli.

**Banking için kural:** Her external entegrasyon (3rd party, legacy, başka context) için **mutlaka ACL**. Kirli model domain'e SIZAMAZ.

#### 5.5. Open Host Service (OHS) + Published Language

Bir context, **birden fazla müşteri** tarafından kullanılacaksa standart bir protokol + dil yayınlar.

```
[OHS Provider] ────── [Published Language (REST/JSON, OpenAPI)] ──────► [Many consumers]
```

**Banking örneği:** `account-service` OpenAPI 3 ile `/v1/accounts` API'sini publish eder. Hangi consumer kim olduğunu bilmez (transfer-service, batch-service, reporting). Stable contract.

**Published Language genelde:**
- OpenAPI 3 (REST için)
- AsyncAPI (event-driven için)
- Protobuf (gRPC için)
- Avro schema (Kafka için)

#### 5.6. Partnership

İki context **karşılıklı bağımlı**, birlikte başarısız ya da birlikte başarılı. Takımlar gerçekten ortak çalışır.

**Banking örneği:** `transfer-service` ↔ `account-service` arasında. Transfer hesabı debit etmeden başarılı sayılamaz; hesap, transferı bilmeden balance güncellenmez. İki takım **aynı sprintte koordine**.

#### 5.7. Conformist'in tersi: Separate Ways

İki context birbirinden **bağımsız**. Hiç entegrasyon yok, paralel iki dünya.

**Banking örneği:** `notification-service` ile `card-management` doğrudan konuşmaz. Birbirini bilmezler. Karşılaştığı yer Kafka event'i (notification, kart eventlerini dinler ama kart-management notification'ı bilmez).

#### 5.8. Big Ball of Mud

Anti-pattern. Context sınırı yok, her şey her şeye bağlı. Tipik **monolitin** sonu.

Faz 1-6 monolit'imiz **küçük olduğu için** ball-of-mud değil. Ama büyüseydi olabilirdi.

#### 5.9. Banking için pratik context map

```
                              [banking-common]            ← Shared Kernel
                            (Money, Currency,
                             AccountId, ErrorCode)
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
       │  account-   │        │  transfer-  │        │   fraud-    │
       │  service    │        │   service   │        │   service   │
       └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
              │                      │                      │
              │  ◄──Customer-────────┤                      │
              │     Supplier         │                      │
              │                      │                      │
              │                      ├──Customer-Supplier──►│
              │                      │     (sync REST)      │
              │                      │                      │
              │ ◄──── async event ───┤ ◄──── async event ──┤
              │   (account.* topic)  │  (fraud.decided.*)   │
              │                                             │
              │                                             │
              │                ┌─────────────┐              │
              │                │ notification│              │
              └────async event─►   service   ◄──async event─┘
                               └─────────────┘
                                     │
                                     │  Separate Ways with:
                                     ▼
                              [Card Mgmt (out)]
```

**ACL'ler:**
- `transfer-service` → `legacy-core-bank-acl` (eğer eski Hogon hâlâ varsa)
- `fraud-service` → `external-blacklist-acl` (3rd party blacklist provider)
- `notification-service` → `sms-provider-acl` (Iletimerkezi vb.)

---

### 6. Aggregate — derinlemesine

Faz 1'de aggregate'i "consistency boundary" diye gördük. Şimdi **microservice perspektifinden** derinleşiyoruz.

#### 6.1. Aggregate boundary kuralı (Vaughn Vernon)

> **Bir transaction içinde sadece BİR aggregate değiştirilir.**

Banking örneği — bir transfer:

```
1 transfer = 2 hesap değişikliği = 2 aggregate
```

İlk bakışta `transferMoney()` metodunda iki account'u da değiştirmek istersin:

```java
// ❌ Aggregate kuralını ihlal ediyor
@Transactional
public void transfer(AccountId from, AccountId to, Money amount) {
    Account fromAcc = accountRepo.findById(from);   // aggregate 1
    Account toAcc   = accountRepo.findById(to);     // aggregate 2
    fromAcc.withdraw(amount);
    toAcc.deposit(amount);
    accountRepo.save(fromAcc);
    accountRepo.save(toAcc);
}
```

Monolit'te bu **çalışır** (DB transaction iki row'u atomic değiştirir). Ama:

- Konkürans sorunu: iki account aynı anda farklı transfer'lere katılırsa **deadlock** riski (Faz 3'te gördün)
- **Aggregate kuralının ruhu:** her aggregate kendi state'inden sorumlu. İki aggregate'i atomic güncellemek, bu sorumluluğu erozyona uğratır.
- Microservice perspektifinden: iki account farklı servislerde olsaydı bu kodu yazamazdın (distributed transaction yok).

**Doğru yaklaşım:**

```java
// ✓ Her aggregate ayrı transaction
@Transactional
public void debitFromAccount(AccountId from, Money amount, TransferId tid) {
    Account fromAcc = accountRepo.findById(from);
    fromAcc.withdraw(amount, tid);  // domain event üretir
    accountRepo.save(fromAcc);
    // Event yayınla (Outbox via Faz 6)
    eventPublisher.publish(new MoneyDebited(from, amount, tid));
}

@Transactional
public void creditToAccount(AccountId to, Money amount, TransferId tid) {
    Account toAcc = accountRepo.findById(to);
    toAcc.deposit(amount, tid);
    accountRepo.save(toAcc);
    eventPublisher.publish(new MoneyCredited(to, amount, tid));
}
```

İki operasyon **ayrı transaction**. Aralarında **eventual consistency**. Tutarlılığı **saga** (Topic 7.7) ile garanti edeceğiz.

**Mid-level mülakatta sorulur:** "İki account'u aynı anda atomik güncelleyemiyorsan, ortada bir an için para 'kayıp' değil mi?" Cevap: Evet, **transit state**'tedir. Çift girişli kayıt sisteminde bu için özel "in-transit" hesap (suspense account) açılır. From hesabından debit edildi → suspense'a credit. Suspense'tan debit → to hesabına credit. Suspense her zaman 0'a dönmek zorunda; muhasebe günsonu kontrol eder.

#### 6.2. Aggregate'in microservice ile ilişkisi

İki yaklaşım var:

**Service per aggregate:** Her aggregate kendi servisinde.
- account-service → Account aggregate
- card-service → Card aggregate

Saf, ama aşırıya kaçabilir. 50 aggregate = 50 servis = operasyonel kâbus.

**Service per bounded context:** Bir servis, bir bounded context'in **tüm aggregate'lerini** içerir.
- account-service → Account aggregate + JournalEntry aggregate
- transfer-service → Transfer aggregate + IdempotencyKey aggregate (basit, ayrı)

**Banking'de tercih:** Service per bounded context. Aggregate granularity çok ince, servis granularity'si daha iri olmalı (Topic 7.2'de detay).

#### 6.3. Aggregate root içinde child entity, ama aggregate dışına direct erişim YOK

```java
public class Transfer {  // aggregate root
    private TransferId id;
    private List<SagaStep> sagaSteps;  // child entity'ler
    
    // ✓ External code Transfer üzerinden SagaStep'lere erişir
    public void recordStepCompleted(SagaStepType type) {
        sagaSteps.stream()
            .filter(s -> s.getType() == type)
            .findFirst()
            .ifPresent(SagaStep::markCompleted);
    }
    
    // ❌ External code'a SagaStep'leri direkt expose etme
    // public List<SagaStep> getSagaSteps() { return sagaSteps; }
}
```

**Banking pratiği:** Repository sadece aggregate root için olmalı (`TransferRepository`, **`SagaStepRepository` YOK**).

---

### 7. Domain Event — bounded context'ler arası iletişim

Domain event = "domain'de bir şey oldu" mesajı. Bounded context'ler arası iletişimin **en sağlıklı yolu**.

```java
// account-service domain'de
public class Account {
    public void withdraw(Money amount, TransferId tid) {
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException();
        }
        balance = balance.subtract(amount);
        events.add(new MoneyDebited(id, amount, tid, Instant.now()));
    }
}
```

**Event'in 3 yönü:**

1. **Past tense** — `MoneyDebited` (oldu, command değil)
2. **Immutable** — bir kez yayılır, değiştirilmez
3. **Self-contained** — event'i alan, başka servise sormaya gerek duymadan iş yapabilmeli (yeterli context taşı)

**Banking örnekleri:**

- `AccountOpened(accountId, ownerId, currency, openedAt)`
- `MoneyDebited(accountId, amount, transferId, occurredAt)`
- `MoneyCredited(accountId, amount, transferId, occurredAt)`
- `TransferRequested(transferId, fromAccountId, toAccountId, amount, currency, requestedAt)`
- `TransferExecuted(transferId, executedAt)`
- `TransferRejected(transferId, reasonCode, rejectedAt)`
- `FraudCheckCompleted(transferId, decision, riskScore, completedAt)`
- `NotificationSent(notificationId, recipientId, channel, sentAt)`

#### Domain event ↔ integration event farkı

- **Domain event:** Aggregate içinde üretilir, **aynı bounded context** içinde dinlenir. İç tutarsızlığa yol açmaz.
- **Integration event:** Bir context'ten diğerine **kasten** yayınlanır. Cross-context kontrat.

Banking pratiği: aggregate domain event'leri **dışarı sızdırma**. Bir **integration event** dönüştürücüsü olsun (Outbox publisher, Faz 6'da gördün).

```
[account-aggregate]
       │
       │ MoneyDebited (domain event)
       ↓
[domain event handler]
       │
       │ AccountBalanceChangedV1 (integration event)
       ↓
[Outbox table]
       │
       │ Kafka publisher
       ↓
[Kafka topic: account.balance.v1]
       │
       ↓
[fraud-service, notification-service, ... consumers]
```

Niye dönüşüm? Çünkü:
- Domain event'in field'ları zaman içinde değişir (refactor)
- Integration event **versionlu kontrat** (V1, V2) — eski consumer'ları bozma
- Internal state dışarı sızmamalı (örn. internal `versionNumber` field'ı dış dünyaya gerekmez)

---

### 8. Banking — gerçek bir context map exercise

Senin için somut bir egzersiz. Aşağıdaki 8 sistem **TR bankasında** yan yana yaşar:

1. **Core Banking** — vadesiz/vadeli hesap ledger
2. **Card Management** — kredi kartı, debit kartı, limit, dönem
3. **Customer 360** — müşteri kimlik, KYC, segmentasyon
4. **Loan / Kredi** — bireysel, ticari kredi yaşam döngüsü
5. **Transfer / Payment Hub** — havale, EFT, FAST, SWIFT
6. **Fraud Risk Engine** — risk skorlama, kural motoru
7. **Notification Hub** — SMS, email, push, IVR
8. **Reporting / DWH** — MASAK, BDDK raporlama, BI

#### Bu 8 context arası ilişki — çözümün

| From | To | İlişki tipi | Neden |
|---|---|---|---|
| Customer 360 | Account | Customer-Supplier | Account müşteri ID'sini Customer 360'tan alır |
| Account | Transfer | Customer-Supplier | Transfer, account API'sini kullanır |
| Transfer | Fraud | Customer-Supplier (sync) | Transfer, fraud onayı için bekler |
| Transfer | Notification | Separate Ways (event) | Async event ile, doğrudan bağ yok |
| Account | Notification | Separate Ways (event) | Account balance change → event |
| Card | Account | Partnership | Card transaction = account hareketi (sıkı koordine) |
| Loan | Account | Partnership | Kredi ödemesi = account debit (sıkı koordine) |
| Reporting | All | Conformist (event consumer) | Tüm sistemler event yayar, reporting kabul eder |
| Core | Legacy Hogon | ACL | Legacy modeli korunmaz, çevrilir |

**Defterine çiz:** Yukarıdaki tabloyu kâğıt-kalem ile **3 kez** çiz. Her çizimde bir context map ilişki tipini sözlü olarak (kendine) açıkla. Bu egzersizi yapmadan Topic 7.2'ye geçme.

---

### 9. Anti-pattern'ler

#### 9.1. "Distributed Big Ball of Mud"

Servisleri ayırdın ama her servis her servise sync REST atıyor. Sonuç: monolitten daha kötü, çünkü bir servis düştüğünde tüm sistem düşer ve **network gecikmeleri** eklendi.

**Çözüm:** Async event-driven iletişim (Kafka) %70+ olsun, sync sadece **gerçekten lazımsa** (örn. fraud check transfer'i bloke etmeli).

#### 9.2. "Shared Database"

İki servis aynı DB'yi paylaşıyor. Schema değişikliği koordine edilmek zorunda → microservice değil, sadece kod ayrı; aslında monolit.

**Banking için bilinçli istisna:** **Reporting / DWH**. Tüm servislerin DB'sine read-only erişim verilebilir (operational DB → analytical DB ETL). Ama bu **özel bir karar**, varsayılan değil.

#### 9.3. "Entity Service"

Bir servis sadece bir entity'i CRUD'lar, hiçbir domain logic taşımaz.

```
GET /v1/customers/{id}
POST /v1/customers
PUT /v1/customers/{id}
DELETE /v1/customers/{id}
```

Bu **anemic service**. Müşteri kayıt etmek bir business operation — değil sadece DB satırı eklemek. KYC kontrolü? Doğrulama? Birden fazla aggregate?

Banking'de service tasarımı **business capability** odaklı olmalı, entity odaklı değil. "customer-service" değil, "customer-onboarding-service" + "customer-360-service" gibi.

#### 9.4. "God Service"

Bir servis 3000 endpoint taşıyor, 50 aggregate'i yönetiyor. Microservice değil — sadece "bir servis", monolit'in eşiti.

**Kural:** Bir servis bir takıma sığmalı (5-9 kişi, two-pizza team). Eğer sığmıyorsa böl.

#### 9.5. "Synchronous Saga"

Saga'yı tüm sync REST çağrıları olarak yazmak. İlk adım yavaşsa zincirin tamamı yavaş.

**Çözüm:** Saga adımları event-driven (Kafka), durum DB'de persist.

#### 9.6. "Aggregate Reaching"

Bir aggregate'in içinden başka bir aggregate'in ID'sini alıp **direkt** repository ile sorgulamak.

```java
// ❌ Aggregate boundary kıran
public class Transfer {
    private AccountId fromAccountId;
    private final AccountRepository accountRepo;  // Domain'de repository YOK
    
    public boolean canExecute() {
        Account from = accountRepo.findById(fromAccountId);  // ❌
        return from.getBalance().isGreaterThan(amount);
    }
}
```

**Çözüm:** Application service orchestrate eder:

```java
public class ExecuteTransferService {
    public void execute(Transfer transfer) {
        Account from = accountRepo.findById(transfer.getFromAccountId());
        from.withdraw(transfer.getAmount(), transfer.getId());  // aggregate kendi state'ini yönetir
        // ...
    }
}
```

---

## Önemli olabilecek araştırma kaynakları (keyword)

- "Domain-Driven Design" Eric Evans — özellikle 2. ve 3. kısım (strategic)
- "Implementing Domain-Driven Design" Vaughn Vernon — pragmatik versiyon, "Effective Aggregate Design" makaleleri
- "Domain-Driven Design Distilled" Vaughn Vernon — kısa, hızlı tekrar
- "Building Microservices" Sam Newman (2. baskı) — Bölüm 2-3 (modeling, decomposition)
- "Strategic Monoliths and Microservices" Vaughn Vernon
- DDD-Crew GitHub: bounded context canvas, context mapping templates
- Martin Fowler — "BoundedContext", "ContextMap" wiki sayfaları
- Eric Evans — "DDD Reference" (kısa PDF, ücretsiz)
- Greg Young — Event Sourcing makaleleri (domain event'leri derinleştirmek için)
- "Context Mapping" — DDD Europe konuşmaları (YouTube)

---

## Mini task'ler

Bu task'leri sırayla yap. Kâğıt-kalem işin **en önemlisi** — kod yazmadan modellemen lazım.

### Task 7.1.1 — Kâğıt-kalem bounded context tespiti (45 dk)

Mevcut `core-banking` monolitinin paket yapısını aç (Faz 1 sonu durumu). Şu listeyi yap:

1. Hangi paket hangi domain kavramına bağlı?
2. Aşağıdaki kelimelerden hangileri tek bir anlamda kullanılıyor, hangileri **birden fazla anlam** içeriyor?
   - Account
   - Customer (eğer varsa)
   - Transfer
   - Transaction
   - Balance
   - Status

3. Hangi paketler **birlikte deploy** edilmek zorunda hissediyor, hangileri ayrılabilir görünüyor?

Bunu **defterine yaz**. Liste formatında, bir-iki cümle her bir madde için.

### Task 7.1.2 — Context map çizimi (60 dk)

Aşağıdaki bounded context'leri içeren bir context map çiz (kâğıt-kalem, sonra dijital olarak da):

- account
- transfer
- fraud
- notification
- (out-of-scope ama gösterilecek: customer, card, legacy-core)

Her ilişki için ilişki tipini etiketle (Customer-Supplier, ACL, Partnership, Shared Kernel, Separate Ways, Conformist).

Çizim aracı: kağıt + telefonla foto, veya draw.io / Excalidraw / Miro.

**Çıktı:** `docs/context-map.png` veya `docs/context-map.drawio` projende.

### Task 7.1.3 — Banking-common module (45 dk)

Shared Kernel'i implement et. Faz 1'deki `common/` paketi şu an monolit içinde. Bunu **ayrı bir Maven module** yap (henüz multi-repo değil, mono-repo içinde):

```
core-banking/
├── pom.xml                    (parent)
├── banking-common/            (yeni module)
│   ├── pom.xml
│   └── src/main/java/com/mavibank/banking/common/
│       ├── Money.java
│       ├── Currency.java (helper)
│       ├── AccountId.java
│       ├── TransferId.java
│       ├── ErrorCode.java
│       └── BankingException.java
├── account-service/           (gelecekte)
├── transfer-service/          (gelecekte)
└── ...
```

**Kural:** `banking-common`'da **hiçbir framework dependency'si yok** (Spring yok, JPA yok, sadece JDK + Bean Validation API olabilir).

Maven dependency:

```xml
<!-- transfer-service/pom.xml -->
<dependency>
    <groupId>com.mavibank.banking</groupId>
    <artifactId>banking-common</artifactId>
    <version>0.1.0</version>
</dependency>
```

Bu task hemen tüm servisleri ayırmıyor — Topic 7.2'de tam ayrım yapacağız. Şimdilik sadece **shared kernel'ı dışarı çıkar**.

### Task 7.1.4 — Ubiquitous language sözlüğü yaz (30 dk)

`docs/ubiquitous-language.md` oluştur. Her bounded context için ayrı bölüm:

```markdown
# Ubiquitous Language

## Account / Core Banking Ledger

- **Hesap (Account):** ...
- **Bakiye (Balance):** ...
- **Hesap durumu (Status):** ACTIVE, FROZEN, CLOSED
  - ACTIVE: ...
  - FROZEN: ...
  - CLOSED: ...
- **Journal Entry:** ...
- **Journal Line:** ...

## Transfer Orchestration

- **Transfer:** ...
- **Idempotency Key:** ...
- **Saga:** ...
- **Compensation:** ...
- **Hold:** ...

## Fraud Risk

- ...

## Notification Delivery

- ...
```

Her terim için **2-3 cümle** yaz. Hangi field'lar, hangi davranışlar.

**Kritik:** Aynı terim iki context'te varsa, **farkı vurgula**. Örn:
- "Account (Core Banking)": balance taşır, debit/credit edilir
- "Account (Authentication, out of scope)": login bilgisi, marketing tag'i

### Task 7.1.5 — ADR yaz (30 dk)

`docs/adr/0007-bounded-context-identification.md`:

```markdown
# ADR-0007: Bounded Context Tespiti (Phase 7)

Date: 2025-MM-DD
Status: Accepted

## Context
Faz 7 microservice mimariye geçiş. Monolit içindeki domain'i bounded context'lere 
ayırmamız gerekiyor.

## Decision
4 bounded context tanımlandı:
1. account (core-banking ledger)
2. transfer (orchestration, idempotency, saga)
3. fraud (risk engine)
4. notification (delivery)

Out-of-scope: customer, card, loan, identity (Faz 8+).

Shared Kernel: banking-common (Money, Currency, AccountId, ErrorCode).

İlişkiler:
- transfer → account: Customer-Supplier (sync REST + async event)
- transfer → fraud: Customer-Supplier (sync REST, kritik kararlar bloke)
- transfer → notification: Separate Ways (async event)
- account → notification: Separate Ways (async event)

## Consequences
+ Her context kendi DB'sine sahip → bağımsız evrim
+ Sync vs async ayrımı net
- Eventual consistency'yi notification'da kabul ettik
- Customer ID sadece referans olarak taşınır, customer modeli yok
```

### Task 7.1.6 — Aggregate transaction boundary refactor (60 dk)

Mevcut monolit'teki `ExecuteTransferService` (Faz 1 sonu) iki aggregate'i tek `@Transactional` ile güncelliyor:

```java
@Transactional
public void execute(...) {
    fromAccount.withdraw(...);
    toAccount.deposit(...);
    accountRepo.save(fromAccount);
    accountRepo.save(toAccount);
}
```

Bunu **bir aggregate per transaction** kuralına çek:

```java
@Service
class ExecuteTransferService {
    
    public void execute(...) {
        // Step 1: separate transaction — debit
        debitService.debit(fromAccountId, amount, transferId);  // @Transactional içinde
        
        // Step 2: separate transaction — credit
        creditService.credit(toAccountId, amount, transferId);  // @Transactional içinde
        
        // Step 3: transfer'i complete olarak mark et
        markCompleted(transferId);
    }
}
```

**Bilerek bırak:** Step 1 başarılı, Step 2 patlarsa? — Şimdilik kabul; **Topic 7.7'de Saga ile çözeceğiz**. Yorum satırı bırak: `// TODO: Phase 7.7 — Saga compensation`

Bu task'ta amaç: aggregate kuralının ruhunu görmek. Şimdi para "in-transit" — saga ile düzelteceğiz.

---

## Test yazma rehberi

### Test 7.1.1 — Bounded context "leak" test'i (ArchUnit)

ArchUnit ile **mimari kuralları compile-time'da kontrol et**:

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version>
    <scope>test</scope>
</dependency>
```

```java
@AnalyzeClasses(packages = "com.mavibank.banking", importOptions = ImportOption.DoNotIncludeTests.class)
class BoundedContextArchTest {
    
    // Account context, transfer/fraud/notification context'in domain'ine bağımlı olmamalı
    @ArchTest
    static final ArchRule accountShouldNotDependOnOtherContexts =
        noClasses().that().resideInAPackage("..account.domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..transfer.domain..",
                "..fraud.domain..",
                "..notification.domain.."
            );
    
    // Banking-common (shared kernel) hiçbir context'e bağımlı olmamalı
    @ArchTest
    static final ArchRule commonShouldNotDependOnContexts =
        noClasses().that().resideInAPackage("..common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..account..",
                "..transfer..",
                "..fraud..",
                "..notification.."
            );
    
    // Banking-common'da Spring annotation YOK
    @ArchTest
    static final ArchRule commonShouldNotUseSpring =
        noClasses().that().resideInAPackage("..common..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
    
    // Account context, transfer context'in private API'sini import etmemeli
    @ArchTest
    static final ArchRule transferShouldNotLeakInternals =
        classes().that().resideInAPackage("..transfer.application.service..")
            .should().notBePublic();
}
```

Bu test'leri çalıştır, **fail eden noktaları gör**. Hangi paket nereye sızıyor — düzelt.

### Test 7.1.2 — Aggregate boundary kural'ı test'i

```java
class AggregateTransactionBoundaryTest {
    
    @Test
    void debitAndCreditShouldBeInSeparateTransactions() {
        // Spring @Transactional propagation REQUIRES_NEW kullanılıyor mu?
        var debitMethod = ReflectionUtils.findMethod(DebitService.class, "debit", ...);
        var ann = AnnotationUtils.findAnnotation(debitMethod, Transactional.class);
        
        assertThat(ann).isNotNull();
        assertThat(ann.propagation()).isEqualTo(Propagation.REQUIRES_NEW);
    }
}
```

### Test 7.1.3 — Domain event yayılma test'i

```java
@Test
void withdrawShouldRaiseMoneyDebitedEvent() {
    Account account = AccountTestBuilder.anAccount()
        .withCurrency("TRY").withBalance("1000.00").build();
    TransferId tid = TransferId.generate();
    
    account.withdraw(Money.of("100.00", "TRY"), tid);
    
    List<DomainEvent> events = account.pullEvents();
    assertThat(events).hasSize(1);
    assertThat(events.get(0))
        .isInstanceOf(MoneyDebited.class)
        .satisfies(e -> {
            var debited = (MoneyDebited) e;
            assertThat(debited.amount()).isEqualTo(Money.of("100.00", "TRY"));
            assertThat(debited.transferId()).isEqualTo(tid);
        });
}
```

### Test 7.1.4 — Ubiquitous language consistency test'i

Bu hassas: kodu okuyarak ubiquitous language'a uymayan terim ararsın. **Manuel review** test'tir:

```
PR review checklist:
- [ ] account context'inde "Transfer" class'ı var mı? (Olmamalı, sadece TransferId referansı)
- [ ] transfer context'inde "Balance" değişkeni var mı? (Olmamalı, account servisine sor)
- [ ] notification context'inde "Account" alanı var mı? (Olmamalı, sadece accountId referansı)
```

Bunu `docs/code-review-checklist.md` olarak yaz.

---

## Claude-verify prompt

```
Aşağıdaki banking projemi DDD strategic design kriterleri ile değerlendir. Phase 7'nin 
ilk topic'inde bounded context tespiti ve context map çizimi yaptım. SADECE eksikleri 
işaretle, kod yazma:

1. Bounded context tespiti:
   - 4 context tanımlı mı (account, transfer, fraud, notification)?
   - Her context'in ubiquitous language sözlüğü docs/ubiquitous-language.md'de var mı?
   - Aynı terim iki context'te farklı anlamda kullanılırsa explicit belirtilmiş mi?
   - Out-of-scope context'ler (customer, card, loan) işaretli mi?

2. Context map:
   - docs/context-map.{png,drawio,md} dosyası var mı?
   - Her ilişki bir context-mapping pattern ile etiketli mi (Customer-Supplier, ACL, 
     Shared Kernel, Separate Ways, Partnership)?
   - Sync vs async iletişim ayrımı var mı?
   - Legacy/external ACL ihtimalleri belirtilmiş mi?

3. Shared kernel:
   - banking-common module oluşturulmuş mu?
   - banking-common'da Spring/JPA dependency YOK mu?
   - Money, Currency, AccountId, TransferId, ErrorCode burada mı?
   - Domain logic (Account aggregate, Transfer aggregate) banking-common'da DEĞİL mi?

4. Aggregate boundary:
   - ExecuteTransferService iki aggregate'i ayrı transaction'da mı güncelliyor?
   - @Transactional(propagation = REQUIRES_NEW) kullanılıyor mu?
   - Aggregate'in içinden başka aggregate'in repository'sine erişim YOK mu?
   - Domain event (MoneyDebited, MoneyCredited) aggregate içinde yayılıyor mu?

5. ArchUnit testleri:
   - Bounded context leak test'leri var mı?
   - banking-common Spring kullanmama test'i var mı?
   - Test'ler geçiyor mu?

6. ADR:
   - 0007-bounded-context-identification.md var mı?
   - Context-mapping kararları gerekçeli mi?

7. Domain event vs integration event:
   - Aggregate domain event yayar, ama dış servise gönderilen event ayrı (integration 
     event) mi?
   - Integration event versionlu (V1) mi?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster (dosya yolu). Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] 4 bounded context tespit edildi, isimleri ve sınırları net
- [ ] `docs/ubiquitous-language.md` her context için ayrı bölüm halinde yazıldı
- [ ] `docs/context-map.{png,drawio}` çizildi, 9 ilişki tipinden uygun olanlar etiketli
- [ ] `banking-common` Maven module oluşturuldu, Spring/JPA dependency YOK
- [ ] `Money`, `Currency`, `AccountId`, `TransferId`, `ErrorCode` banking-common'a taşındı
- [ ] `ExecuteTransferService` iki aggregate'i ayrı `@Transactional(REQUIRES_NEW)` ile güncelliyor (Saga TODO yorumu var)
- [ ] Aggregate domain event'ler (`MoneyDebited`, `MoneyCredited`) `pullEvents()` ile yayılıyor
- [ ] ArchUnit bounded context test'leri yazıldı, hepsi geçiyor
- [ ] ADR `0007-bounded-context-identification.md` yazıldı
- [ ] Aggregate transaction boundary kuralını birine 2 dakikada anlatabilirim
- [ ] Bounded context vs microservice ilişkisini ayırt edebiliyorum
- [ ] Banking domain'de ACL'in ne zaman gerekli olduğunu örnekle açıklayabilirim

---

## Defter notları

Aşağıdaki cümleleri **kendi kelimelerinle** doldur:

1. "Bir bounded context'in başka bir bounded context'ten farklı olduğunu **3 göstergesi**: ____."
2. "Banking'de aynı kelimenin (`Account`) farklı bounded context'lerde farklı anlamı olmasının pratik sonucu ____. Bunu adresleyen yaklaşım ____."
3. "Anti-Corruption Layer (ACL) ne zaman lazım? Cevabım ____. Banking'de tipik ACL örneği ____."
4. "Shared Kernel ile diğer ilişki tiplerinin farkı ____. Banking-common'da neyi paylaşırım, neyi paylaşmam ____."
5. "Aggregate boundary kuralı (one aggregate per transaction) banking'de neden önemli? ____. Bunu ihlal edersem ____."
6. "Domain event ve integration event'in farkı ____. Niye ikisini ayırırım ____."
7. "Bounded context = microservice eşitliği her zaman doğru mu? Cevabım ____ çünkü ____."
8. "Conway's Law banking'de pratiğe nasıl yansır? Örnek: ____."
9. "Customer-Supplier ile Partnership farkı ____. Banking'de tipik örnekleri ____."
10. "Aggregate'in içinden başka aggregate'in repository'sine erişmenin neden yanlış olduğunu, alternatifi ile birlikte açıkla: ____."

---

Tamamlandı → [02-service-decomposition/](../02-service-decomposition/index.md)
