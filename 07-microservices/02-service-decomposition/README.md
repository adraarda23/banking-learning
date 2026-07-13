# Topic 7.2 — Service Decomposition

## Hedef

Monolit'i microservice'lere bölme **stratejilerini** öğrenmek. Hangi servisi ayır, hangisini birleştir kararını **gerekçeli** verebilmek. Banking 4-servis (account, transfer, fraud, notification) bölüşümünün **why**'ını anlamak. Database per service, shared library yönetimi, distributed monolith anti-pattern.

## Süre

Okuma: 1.5 saat • Mini task: 1.5 saat • Test: yok • Toplam: ~3 saat

## Önbilgi

- Topic 7.1 (DDD Strategic) bitti — bounded context, ubiquitous language, context map
- Phase 1-6 boyunca `core-banking` monolit'i (`AccountController`, `TransferService`, vb.) yazıldı
- Phase 6 outbox + Kafka event'ler hazır (decomposition'a kritik altyapı)

---

## Kavramlar

### 1. Decomposition kararı — kriter matrisi

Bir feature/modül microservice mi olsun, monolit içinde mi kalsın?

| Kriter | Microservice ayır | Birleşik tut |
|---|---|---|
| Ayrı team (Conway's Law) | ✓ | — |
| Ayrı release cycle | ✓ | — |
| Ayrı scalability needs | ✓ | — |
| Bağımsız deploy gerekli | ✓ | — |
| Yüksek coupling (sürekli birlikte değişir) | — | ✓ |
| Aynı transaction boundary | — | ✓ |
| Latency hassas (sync call) | — | ✓ |
| Farklı tech stack ihtiyacı | ✓ | — |
| Domain'in **stabil** çekirdek parçası | ? | bağımlı |
| External dependency (vendor) | ✓ (ACL) | — |
| Düşük volume + basit logic | — | ✓ |

**Karar:**
- 3+ "✓" → microservice
- 3+ "Birleşik" → modüler monolit
- Mixed → tartış, **erkenden ayırma**

### 2. Banking — neden 4 servis

`core-banking` monolit'inden 4 servise bölme rasyoneli:

#### account-service

**Sorumluluk:**
- Account aggregate (open, close, balance, status)
- JournalEntry + JournalLine (double-entry ledger)
- Balance reconciliation primitive
- gRPC `debit`, `credit`, `getBalance` internal API
- REST API `/accounts` CRUD

**Neden ayrı:**
- **Scaling pattern:** Read-heavy (10:1 read:write) — caching, replica
- **Stability:** Account domain çok stabil, nadiren değişir
- **Authority:** Bakiyenin **tek kaynağı**. Cross-service'lerden referans nokta.
- **Performance critical:** Her transfer çağrı yapacak → düşük latency

**Tech:**
- Spring Boot + Postgres (Oracle migration sonrası)
- HikariCP optimized (Phase 2)
- gRPC for internal calls (performance)
- REST for external

#### transfer-service

**Sorumluluk:**
- Transfer aggregate (orchestration)
- Idempotency-Key handling
- Saga coordinator (cross-bank)
- account-service'i çağırır
- fraud-service'i çağırır (pre-check)
- Outbox event publish (Phase 6)

**Neden ayrı:**
- **Volatility:** Transfer logic sık değişir (yeni transfer tipleri, regulatory)
- **Saga state:** Stateful, dedicated
- **Compute heavy:** Multi-step orchestration
- **Business critical:** Ayrı release & monitoring

**Tech:**
- Spring Boot + Postgres
- Saga orchestrator (Phase 6 Topic 6.7)
- Kafka producer (outbox)

#### fraud-service

**Sorumluluk:**
- Fraud rule engine
- Kafka Streams real-time scoring (Phase 6)
- transfer-service tarafından sync `scoreTransfer(transfer)` çağrılır
- async event-based scoring
- Alert generation

**Neden ayrı:**
- **Different scaling:** CPU-heavy (rule evaluation)
- **Different tech:** Kafka Streams stateful
- **ML potential:** Sonradan ML model eklenebilir, ayrı stack
- **Compliance:** Fraud team ayrı, ayrı release control

**Tech:**
- Spring Boot + Kafka Streams
- Read replica (account info için)
- Stateful (Kafka Streams state store + RocksDB)

#### notification-service

**Sorumluluk:**
- SMS, email, push gateway entegrasyonu
- Notification queue (Kafka consumer)
- Idempotent delivery
- Template management
- Retry + DLT

**Neden ayrı:**
- **External dependency:** SMS gateway (multiple providers)
- **Independent failure:** SMS gateway down → diğer servisler etkilenmesin
- **High throughput:** Parallel consumer threads
- **Simple domain:** Stateless processing

**Tech:**
- Spring Boot + Kafka consumer
- ResilientExternalGateway (Phase 7 Topic 5)

### 3. Hangi feature'ları ayırma — banking örnek

#### Ayırma:
- **Authentication** → auth-service ya da **Keycloak** (vendor)
- **Audit** → audit-service (centralized regulatory)
- **Reporting** → reporting-service (read replica, CQRS)
- **Customer KYC** → customer-service (compliance ekibi)

#### Birleşik tut:
- **Money + Currency value objects** → shared library (`banking-commons`)
- **Account + JournalEntry + JournalLine** → account-service içinde (aggregate boundary)
- **TransferRequest + Transfer + IdempotencyKey** → transfer-service içinde

### 4. Database per service

**Kural:** Her servis kendi DB schema'sını sahiplenir.

```
account-service     → account_db   (accounts, journal_entries, journal_lines)
transfer-service    → transfer_db  (transfers, saga_states, idempotency_keys, outbox_events)
fraud-service       → fraud_db     (fraud_rules, fraud_scores, fraud_alerts)
                                    + Kafka Streams state store
notification-service → notification_db (notification_log, processed_events)
audit-service       → audit_db     (audit_records — regulatory immutable)
```

**Faydaları:**
- **Data sovereignty** — her servis kendi schema'sını migrate eder, evolve eder
- **Schema değişiklik impact** — sadece o servis
- **Failure isolation** — bir DB down → bir servis affected
- **Polyglot persistence** — fraud için Cassandra, audit için BigQuery (eğer gerekirse)

#### Cross-service data access

**Yöntem 1: Sync HTTP/gRPC call**

```java
@Service
public class TransferService {
    @Autowired AccountServiceClient accountClient;
    
    public Transfer execute(TransferRequest req) {
        Account from = accountClient.getById(req.fromAccountId());   // network call
        // ...
    }
}
```

**Trade-off:** Latency + failure cascade riski. Resilience4j (Topic 7.5) ile mitigate.

**Yöntem 2: Async event + local read model**

```java
// account-service publishes
kafka.send("banking.accounts.updated", accountUpdatedEvent);

// transfer-service consumer + own read model
@KafkaListener("banking.accounts.updated")
public void onAccountUpdated(AccountUpdatedEvent event) {
    accountReadModel.upsert(event);   // local DB
}

// Transfer logic
public Transfer execute(req) {
    Account from = accountReadModel.findById(req.fromAccountId());   // local read
}
```

**Trade-off:** Eventual consistency, local read model maintenance.

**Banking pratiği:**
- **Real-time critical** (debit/credit) → sync gRPC + circuit breaker
- **Reference data** (account info display) → local read model
- **Bulk/reporting** → CQRS read replica

### 5. Anti-pattern: Shared database

```
account-service    →┐
                    ├→ shared_banking_db (accounts, transfers, audit, vb.)
transfer-service   →┤
                    │
audit-service      →┘
```

**Sorunlar:**
- Schema değişikliği **tüm servisleri** etkiler
- Coordination overhead — release cycle birbirine bağlanır
- Data sovereignty yok
- "Distributed monolith" oluşur

**Banking için YASAK.** Her servis kendi DB'si.

### 6. Distributed monolith — büyük tuzak

3 yıl önce **monolit**. Bugün 10 servise böldün. Hâlâ aynı **monolit-like** problemler:

**Belirtileri:**
- Bir feature için 5+ servis aynı anda deploy
- Bir servisin fail'i diğer 9'unu etkiliyor
- Cross-service sync HTTP call zinciri (5+ hop)
- Shared library version mismatch hell
- Cross-service transaction (manual orchestration veya 2PC)

**Sebepler:**
- Yanlış decomposition (yanlış sınırlar)
- Sync HTTP heavy usage
- Shared DB veya shared mutable state
- Versioned shared library zorla

**Çözüm:**
- Bounded context'leri yeniden çiz (DDD strategic)
- Sync HTTP → async event (Topic 6.6 outbox)
- Resilience pattern (Topic 7.5)
- API versioning + contract testing (Topic 12.5)

### 7. Shared library vs duplication

**Banking shared concepts:** Money, Currency, AccountId, OwnerId, TransferId.

#### Yaklaşım A: Shared library

```
banking-commons (Maven module)
├── Money.java
├── Currency.java
├── AccountId.java
├── ...
└── pom.xml (versioned)

account-service depends on banking-commons:1.2.0
transfer-service depends on banking-commons:1.2.0
fraud-service depends on banking-commons:1.2.0
```

**Avantaj:** Tek kaynak, type safety, no duplication.

**Dezavantaj:** Version coordination. Her library upgrade tüm servisleri etkiler.

#### Yaklaşım B: Duplication

```
account-service/Money.java
transfer-service/Money.java   (kopya)
fraud-service/Money.java      (kopya)
```

**Avantaj:** Servisler bağımsız evolve eder.

**Dezavantaj:** Drift riski, bug fix N kez tekrarlanır.

#### Pragmatik karar

**Banking:**
- **Stable value objects** (Money, Currency) → shared library
- **Domain entities** (Account, Transfer) → her servisin kendi tanımı (DDD bounded context)
- **DTO / event format** → published language (Avro schema registry)

`banking-commons` minimal: sadece truly stable value object'ler. Domain logic yok.

### 8. Cross-service contract

İki servis arası **kontrat** — değişiklikler ikinci tarafı kırmamalı.

**Tipi:**

#### REST OpenAPI spec

```yaml
openapi: 3.0.0
info: { title: Account Service }
paths:
  /accounts/{id}:
    get:
      parameters: [{ name: id, in: path, schema: { type: string } }]
      responses:
        "200":
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Account" }
```

Generated client + server stubs.

#### gRPC .proto

```protobuf
syntax = "proto3";
service AccountService {
  rpc GetById (GetAccountRequest) returns (Account);
  rpc Debit (DebitRequest) returns (DebitResponse);
}
```

Type-safe code generation. Banking için sync inter-service genelde gRPC.

#### Avro/Protobuf event schema

```json
{
  "type": "record",
  "name": "TransferCompleted",
  "fields": [...]
}
```

Schema Registry (Phase 6). Producer + consumer compatibility check.

**Banking pratiği:**
- Internal sync: gRPC
- Public sync: REST OpenAPI
- Async events: Avro + Schema Registry

### 9. Contract testing (Phase 12 referansı)

Provider değişimi consumer'ı kırmaz garantisi. Spring Cloud Contract veya Pact (Phase 12 Topic 5).

Banking pratiği: CI'da contract test gate. Provider değişikliği önce.

### 10. Decomposition strategies

#### Strategy 1: Decompose by Business Capability

İş yetkinliği ekseninde. Banking'de:
- Account management
- Transfer
- Card management
- Loan
- Customer (KYC)
- Audit
- Fraud
- Notification

**En yaygın** ve **DDD bounded context** ile uyumlu.

#### Strategy 2: Decompose by Subdomain

Eric Evans DDD:
- **Core subdomain:** Differentiating value (banking için: lending algorithms, fraud detection)
- **Supporting subdomain:** Core'u destekleyen ama differentiating değil (customer mgmt, KYC)
- **Generic subdomain:** Off-the-shelf alınabilir (auth, logging, monitoring)

Banking strateji:
- Core → **own implementation** (kompetitif avantaj)
- Supporting → **kendi yapımı** veya partial vendor
- Generic → **vendor** (Auth0, Datadog, vb.)

#### Strategy 3: Decompose by Aggregate

Her aggregate ayrı servis (1:1). Aşırı granular — micro-microservices, operational nightmare.

#### Strategy 4: Decompose by Team

Conway's Law: Sistemler organizasyon yapısını yansıtır. Team başına servis.

Banking realistic constraint. Team size 5-9 (two-pizza rule).

**Pragmatik karar:** Business capability primary, team constraint realistic.

### 11. Strangler Fig pattern — kademeli migration

Banking gerçeği: monolitten "big bang" rewrite **YASAK** (regulatory, business continuity).

Strangler Fig:

```
Phase 0: Monolit (legacy core banking system)

Phase 1: API Gateway koy, traffic monolit'e route
         + Yeni feature → microservice yaz, Gateway route

Phase 2: Eski feature'ları **kademeli** migrate
         Read-only first (CQRS-like)
         Sonra write paths
         Monolit'in o özelliği decommission

Phase 3: Monolit küçülür, sonunda decommission
```

**Banking örnek:**

```
2020: Legacy core banking (COBOL on mainframe)
2022: API Gateway. Yeni mobile API endpoint'leri → microservice
      Eski branch terminal → legacy
2024: Account read → microservice. Account write → hâlâ legacy
2025: Transfer microservice. Legacy decommission timeline 2027
```

**Banking için Strangler Fig standart.** Regulatory bunu zorunlu kılar.

### 12. Service boundaries — ne **birleşik tut**

Microservice yarış kazanmak değil. Bazı şeyler birleşik kalmalı:

#### Aggregate consistency boundary

Account + JournalEntry + JournalLine **tek aggregate**. Same DB, same transaction:

```java
@Transactional
public Transfer execute(...) {
    Account from = accountRepo.findByIdAndLock(...);
    Account to = accountRepo.findByIdAndLock(...);
    
    // Same TX:
    journalEntryRepo.save(new JournalEntry(...));
    journalLineRepo.save(new JournalLine(from, DEBIT, amount));
    journalLineRepo.save(new JournalLine(to, CREDIT, amount));
    accountRepo.save(from);
    accountRepo.save(to);
}
```

İki ayrı servis (`account-service` ve `journal-service`) yapma — distributed transaction'a düşersin.

#### Tight coupling = same service

Module A ve B sürekli birlikte değişiyor (her PR ikisini de modify) → **aslında aynı domain**. Birleşik tut.

### 13. Service granularity — sweet spot

**Çok büyük (monolit):** Tüm sorunlar, hiçbir avantaj.

**Çok küçük (nano-service):** Operational nightmare. 100 servis → 100 K8s deployment, 100 CI pipeline, 100 monitoring config.

**Sweet spot (banking):**
- 4-12 service
- Her servis 5-9 developer kapsamında
- Her servis 1-2 aggregate
- Her servis bağımsız deploy edilebilir
- Her servis kendi DB

### 14. Banking örnek — full decomposition map

```
Internet
   ↓
[API Gateway]   ← Spring Cloud Gateway, JWT auth, rate limiting
   ↓
┌────────────┬────────────┬────────────┬────────────┐
│ account    │ transfer   │ fraud      │ notification│
│ service    │ service    │ service    │ service    │
└─────┬──────┴─────┬──────┴─────┬──────┴─────┬──────┘
      ↓            ↓            ↓            ↓
   account_db  transfer_db  fraud_db   notification_db
      ↑            ↓            ↓            ↓
      ↑       Kafka (banking.transfers)
      ↑            ↓
      └────────────┘
        async event flow

External:
[Keycloak]    ← OAuth2 / OIDC
[Schema Registry]
[Prometheus + Grafana + Jaeger]
[Kafka Connect + Debezium]   ← Outbox CDC
```

### 15. Banking anti-pattern'leri

**Anti-pattern 1: Shared DB across services**

```
account-service →┐
                 ├→ shared_db
transfer-service →┘
```

Coupling, deploy interdependence. **YASAK.**

**Anti-pattern 2: Sync HTTP chain (3+ hop)**

```
Gateway → transfer-service → account-service → fraud-service → KKB
```

5 sync call. Bir tane fail → tümü fail. Network round-trip × N.

**Çözüm:** Bazı çağrıları async event'e çevir. Local read model.

**Anti-pattern 3: Distributed monolith**

10 servis, ama hepsi aynı release cycle'da, sync HTTP heavy, shared library lock. Microservice avantajı yok, monolit dezavantajı + distributed sistem complexity.

**Anti-pattern 4: Nano-service**

```
- user-name-service    (sadece user name CRUD)
- user-email-service   (sadece email)
- user-phone-service   (sadece phone)
```

Aşırı granular. Operational nightmare.

**Çözüm:** `customer-service` tek (her customer attribute orada).

**Anti-pattern 5: Big-bang decomposition**

Bir gecede monolitten 10 servise. Banking için **YASAK** (regulatory + business continuity). Strangler fig.

**Anti-pattern 6: Tek developer'ın 5 servisi**

Conway's Law. Servis sınırları team sınırını yansıtsın.

**Anti-pattern 7: Database in service container**

```yaml
service: account-service
  containers:
    - app
    - postgres   # ❌ DB internal
```

Stateful service, scale impossible. DB **ayrı** managed (RDS, Cloud SQL).

---

## Önemli olabilecek araştırma kaynakları

- "Microservices Patterns" (Chris Richardson) — Chapter 2 (Decomposition)
- "Building Microservices" (Sam Newman, 2nd Ed)
- Martin Fowler's microservices articles
- Eric Evans DDD blue book — strategic design chapter
- Vaughn Vernon — Strategic DDD
- Sam Newman — "Monolith to Microservices" book
- Netflix tech blog — decomposition stories

---

## Mini task'ler

### Task 7.2.1 — Decomposition decision matrix (30 dk)

`core-banking`'in her modülü için tablo:

| Modül | Team | Release cycle | Scaling | Coupling | Karar |
|---|---|---|---|---|---|
| Account | A | weekly | read-heavy | low | own service |
| Transfer | B | weekly | write-heavy | medium | own service |
| Money | — | stable | — | high | shared library |
| Audit | C | rare | append-only | low | own service |
| Customer | D | weekly | mixed | medium | own service |

10+ modül için karar gerekçeli yaz. **Defterimde** bu tablo.

### Task 7.2.2 — Maven multi-module setup (45 dk)

```
core-banking-parent/
├── pom.xml (parent)
├── banking-commons/
│   ├── pom.xml
│   └── src/main/java/.../{Money, Currency, AccountId, OwnerId, ...}
├── account-service/
│   └── pom.xml (depends on banking-commons)
├── transfer-service/
│   └── pom.xml (depends on banking-commons)
├── fraud-service/
├── notification-service/
└── api-gateway/
```

`mvn install` çalışmalı. `account-service` `banking-commons.Money` import edebilmeli.

### Task 7.2.3 — Database per service migration (45 dk)

Mevcut tek schema'dan 4 schema'ya bölme:

```sql
CREATE SCHEMA account_db;
CREATE SCHEMA transfer_db;
CREATE SCHEMA fraud_db;
CREATE SCHEMA notification_db;

-- Account tables → account_db
ALTER TABLE accounts SET SCHEMA account_db;
ALTER TABLE journal_entries SET SCHEMA account_db;
ALTER TABLE journal_lines SET SCHEMA account_db;

-- Transfer tables → transfer_db
ALTER TABLE transfers SET SCHEMA transfer_db;
ALTER TABLE saga_states SET SCHEMA transfer_db;
ALTER TABLE outbox_events SET SCHEMA transfer_db;
ALTER TABLE idempotency_keys SET SCHEMA transfer_db;

-- vs.
```

Flyway migration `V_split_schemas.sql`.

### Task 7.2.4 — Strangler fig planning (30 dk)

`core-banking` monolit'inden 4 servise migration planı:

**Sprint 1-2:** API Gateway + transfer-service ayır (yeni feature olduğu için)
**Sprint 3-4:** notification-service ayır (consumer pattern, basit)
**Sprint 5-7:** account-service ayır (en kritik, dikkat)
**Sprint 8-9:** fraud-service ayır (Kafka Streams)

Her sprint için: feature flag, rollback plan, parallel-run pattern.

**Defterimde** sprint-by-sprint plan.

### Task 7.2.5 — Cross-service data access strategy (30 dk)

3 senaryo + strategy:

1. **Transfer servisi account balance bilmesi gerek:** sync gRPC + circuit breaker?
2. **Audit servisi account name görmek istiyor (rapor):** async event + local read model?
3. **Fraud servisi customer KYC status:** read replica veya sync gRPC?

Her birinin trade-off'unu **defterimde** yaz.

### Task 7.2.6 — Distributed monolith risk assessment (30 dk)

Mevcut `core-banking`'i analiz et — distributed monolith'e döner mi?

Checklist:
- [ ] Bir feature için 3+ servis aynı anda deploy gerekiyor mu?
- [ ] Sync HTTP chain 3+ hop var mı?
- [ ] Shared library upgrade tüm servisleri etkileyecek mi?
- [ ] Cross-service sync transaction'a ihtiyaç var mı?

Riskli yerleri **defterimde** mark et.

---

## Test yazma rehberi

Bu topic daha çok **architecture decision**. Test'ler ArchUnit (Topic 12.4) ile:

```java
@AnalyzeClasses(packages = "com.mavibank.banking")
public class ServiceDecompositionTest {
    
    @ArchTest
    static final ArchRule modules_should_be_independent =
        slices().matching("com.mavibank.banking.(*)..")
            .should().notDependOnEachOther()
            .ignoreDependency(...)
            .check(classes);
    
    @ArchTest
    static final ArchRule no_cycles_between_services =
        slices().matching("com.mavibank.banking.(*)..")
            .should().beFreeOfCycles();
    
    @ArchTest
    static final ArchRule account_service_should_not_depend_on_transfer =
        noClasses().that().resideInAPackage("..account..")
            .should().dependOnClassesThat().resideInAPackage("..transfer..");
}
```

CI'da fail → cross-service dependency unintended.

---

## Claude-verify prompt

```
Microservice decomposition tasarımımı banking-grade kriterlere göre değerlendir:

1. 4 servis ayrımı:
   - Business capability ekseninde mi?
   - Her servis bir bounded context'i implement ediyor mu?
   - Aggregate'lar service içinde (split yok)?

2. Database per service:
   - Her servisin kendi schema'sı (veya DB) mı?
   - Shared DB anti-pattern yok mu?

3. Shared library:
   - banking-commons minimal (sadece stable value objects)?
   - Domain logic shared library'de YOK?
   - Version management strategy var mı?

4. Cross-service communication:
   - Real-time critical için sync gRPC + circuit breaker?
   - Reference data için async event + read model?
   - Reporting için CQRS?

5. Strangler fig:
   - Big-bang migration anti-pattern'inden kaçınılmış mı?
   - Sprint-by-sprint plan var mı?

6. Anti-pattern detection:
   - Distributed monolith risk değerlendirmesi yapıldı mı?
   - Sync HTTP chain 3+ hop var mı?
   - Nano-service yok mu (her servis substantial)?

7. Service granularity:
   - 4-12 service sweet spot içinde mi?
   - Her servis 1-2 aggregate?

8. Conway's Law:
   - Team yapısı servis sınırlarını yansıtıyor mu?
   - Team size 5-9 (two-pizza)?

9. ArchUnit:
   - Cross-service dependency tests yazılmış mı?
   - Cycle detection?
   - Layered architecture enforcement?

10. Banking-specific:
    - Account aggregate consistency boundary (account + journal) korunmuş mu?
    - Audit centralized (regulatory)?
    - Fraud isolated (compliance team)?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] 10+ modül için decomposition decision matrix (defterimde)
- [ ] Maven multi-module setup
- [ ] banking-commons module (Money, Currency, ID'ler)
- [ ] 4 service ayrı Maven module
- [ ] Database per service migration
- [ ] Strangler fig sprint plan
- [ ] Cross-service data access strategy (3 senaryo)
- [ ] Distributed monolith risk assessment
- [ ] ArchUnit cross-service dependency tests

---

## Defter notları (10 madde)

1. "Service ayır vs birleşik tut karar matrisi (10 kriter): ____."
2. "Banking 4-servis (account/transfer/fraud/notification) ayrım rasyoneli + her birinin scaling pattern: ____."
3. "Database per service prensibinin 4 avantajı: ____."
4. "Cross-service data access stratejileri (sync gRPC, async event, CQRS) ne zaman hangisi: ____."
5. "Distributed monolith 7 belirtisi: ____."
6. "Strangler fig pattern banking regulatory için neden zorunlu: ____."
7. "Shared library yönetimi — minimal scope kararı: ____."
8. "Decomposition strategy (capability vs subdomain vs aggregate vs team) banking için seçim: ____."
9. "Nano-service vs right-sized service operational cost: ____."
10. "Conway's Law banking org structure ve servis yapısı uyumu: ____."
