# Topic 12.5 — Contract Testing: Spring Cloud Contract, Pact

## Hedef

Microservice'ler arası **contract** garantisi: Consumer-Driven Contracts (CDC) ile provider değişimi consumer'ı kırmaz. Spring Cloud Contract (banking yaygın), Pact (multi-language broker), Kafka contract, banking event schema evolution, breaking change detection, CI integration, contract broker.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Phase 6 (Messaging) bitti
- Phase 7 (Microservices) bitti
- JUnit 5 (Topic 12.1) bitti

---

## Kavramlar

### 1. Contract testing — niye?

**Problem:** Microservice'ler ayrı deploy. Provider değişimi → consumer kırılır → production'da fark edilir.

**Geleneksel çözümler:**
- **End-to-end tests:** Yavaş, flaky, çoklu service setup
- **Integration tests with real provider:** Provider runtime'da gerek

**Contract testing:**
- **Consumer** "ben bunu bekliyorum" beyan eder (contract)
- **Provider** contract'a göre stub verifies (CI'da)
- Provider değişikliği breaking olursa CI fail → production'a gitmeden yakala

```
Consumer (Transfer Service):
  "POST /accounts/{id}/debit
   Expected response: {accountId: UUID, balanceAfter: BigDecimal, status: 'OK'}"

Provider (Account Service):
  Contract verified in CI
  Response shape: {accountId: UUID, newBalance: BigDecimal, ok: true}
  → Mismatch detected → fail
```

### 2. Spring Cloud Contract — provider-side

Provider yazar contract, consumer kullanır stub. **TR banking yaygın** Spring ecosystem.

```xml
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>4.1.0</version>
    <extensions>true</extensions>
    <configuration>
        <baseClassForTests>com.bank.account.AccountContractBase</baseClassForTests>
        <testFramework>JUNIT5</testFramework>
        <testMode>MOCKMVC</testMode>
    </configuration>
</plugin>
```

**Contract DSL** (Groovy or YAML):

```groovy
// src/test/resources/contracts/account/shouldDebitAccount.groovy
Contract.make {
    description "Should debit account successfully"
    request {
        method POST()
        url("/v1/accounts/acc-001/debit") {
            headers {
                contentType applicationJson()
                header("X-Idempotency-Key", anyAlphaNumeric())
                header("Authorization", anyAlphaNumeric())
            }
            body([
                amount: 100.00,
                currency: "TRY",
                reference: $(anyUuid())
            ])
        }
    }
    response {
        status OK()
        headers {
            contentType applicationJson()
        }
        body([
            accountId: "acc-001",
            balanceAfter: $(producer(950.00), consumer(anyNumber())),
            status: "DEBITED",
            timestamp: $(producer(execute('localDateTime()')), consumer(anyDateTime()))
        ])
    }
}
```

**Provider base class:**

```java
public abstract class AccountContractBase {
    
    @Autowired
    protected AccountController accountController;
    
    @MockBean
    protected AccountService accountService;
    
    @BeforeEach
    void setup() {
        RestAssuredMockMvc.standaloneSetup(accountController);
        
        when(accountService.debit(eq("acc-001"), any())).thenReturn(
            new DebitResult("acc-001", new BigDecimal("950.00"), "DEBITED", Instant.now()));
    }
}
```

Plugin generates JUnit tests from contracts. Run `mvn test` → tests verify controller obeys contract.

### 3. Consumer side — stub usage

Stubs published to Maven repo / git.

```xml
<dependency>
    <groupId>com.bank</groupId>
    <artifactId>account-service</artifactId>
    <version>1.0.0</version>
    <classifier>stubs</classifier>
    <type>jar</type>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.bank:account-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class TransferServiceTest {
    
    @Autowired AccountClient accountClient;
    
    @Test
    void shouldUseAccountServiceForDebit() {
        DebitResult result = accountClient.debit("acc-001", new BigDecimal("100"));
        
        assertThat(result.balanceAfter()).isNotNull();   // From stub
        assertThat(result.status()).isEqualTo("DEBITED");
    }
}
```

Consumer test runs against **provider's published stub**. Real account service not needed.

### 4. Pact — multi-language broker model

Pact = consumer-driven, multi-language (Java, JS, Go, Ruby).

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
```

**Consumer side:**

```java
@ExtendWith(PactConsumerTestExt.class)
class TransferAccountContractTest {
    
    @Pact(consumer = "transfer-service")
    public RequestResponsePact debitAccount(PactDslWithProvider builder) {
        return builder
            .given("account acc-001 has balance 1000.00")
            .uponReceiving("a debit request for 100 TRY")
                .path("/v1/accounts/acc-001/debit")
                .method("POST")
                .matchHeader("Content-Type", "application/json")
                .body(new PactDslJsonBody()
                    .decimalType("amount", 100.00)
                    .stringValue("currency", "TRY")
                    .uuid("reference"))
            .willRespondWith()
                .status(200)
                .matchHeader("Content-Type", "application/json")
                .body(new PactDslJsonBody()
                    .stringValue("accountId", "acc-001")
                    .decimalType("balanceAfter", 900.00)
                    .stringValue("status", "DEBITED"))
            .toPact();
    }
    
    @Test
    @PactTestFor(providerName = "account-service", pactMethod = "debitAccount")
    void verifyDebitWorks(MockServer mockServer) {
        AccountClient client = new AccountClient(mockServer.getUrl());
        
        DebitResult result = client.debit("acc-001", new BigDecimal("100"), "TRY");
        
        assertThat(result.balanceAfter()).isEqualByComparingTo("900.00");
    }
}
```

Pact JSON generated → published to **Pact Broker**.

**Provider side verification:**

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Provider("account-service")
@PactBroker(host = "pact-broker.bank.com", port = "443", scheme = "https",
            authentication = @PactBrokerAuth(token = "${PACT_BROKER_TOKEN}"))
class AccountProviderVerificationTest {
    
    @LocalServerPort int port;
    
    @MockBean AccountService accountService;
    
    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @State("account acc-001 has balance 1000.00")
    void setupAccountState() {
        when(accountService.debit("acc-001", any())).thenReturn(
            new DebitResult("acc-001", new BigDecimal("900.00"), "DEBITED", Instant.now()));
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTest(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

CI provider verifies all consumer pacts.

### 5. Pact Broker — central registry

Broker stores:
- Pact contracts (consumer-published)
- Verification results (provider-published)
- Can-i-deploy queries

```
Consumer pushes pact:
  Consumer test pass → pact JSON → broker

Provider verifies:
  Provider tests → pact from broker → results → broker

Can-i-deploy:
  pact-broker can-i-deploy --pacticipant transfer-service --version 1.5
  → Yes if all provider verifications for v1.5 pass
  → No otherwise
```

Banking CI gate:
```bash
pact-broker can-i-deploy \
  --pacticipant transfer-service \
  --version $GIT_SHA \
  --to-environment production
```

### 6. Banking — contract for events (Kafka)

Spring Cloud Contract Kafka stub:

```groovy
Contract.make {
    description "Transfer initiated event"
    label "transferInitiated"
    input {
        triggeredBy "publishTransferInitiated()"
    }
    outputMessage {
        sentTo "transfer-events"
        body([
            transferId: $(anyUuid()),
            fromAccount: "acc-001",
            toAccount: "acc-002",
            amount: 100.00,
            currency: "TRY",
            timestamp: $(producer(execute('localDateTime()')), consumer(anyDateTime()))
        ])
        headers {
            header("traceparent", anything())
        }
    }
}
```

Consumer test:
```java
@AutoConfigureStubRunner(ids = "com.bank:transfer-service:+:stubs", repositoryRoot = "...")
class FraudServiceContractTest {
    
    @Test
    void shouldConsumeTransferInitiatedEvent() {
        stubFinder.trigger("transferInitiated");
        
        // Consumer receives mock event
        await().untilAsserted(() -> {
            assertThat(fraudService.processedCount()).isEqualTo(1);
        });
    }
}
```

Banking event schema evolution catch-able.

### 7. Schema registry — Avro contract

Kafka Schema Registry (Confluent / Apicurio):

```
Producer publishes schema → Registry stores
Consumer reads schema → Registry verifies compatibility

Compatibility modes:
- BACKWARD (default): New schema can read old data
- FORWARD: Old schema can read new data
- FULL: Both
- NONE: No check
```

Banking için BACKWARD compat (old consumers can keep reading after producer update).

Avro schema evolution:

```json
// v1
{
  "type": "record",
  "name": "TransferEvent",
  "fields": [
    {"name": "transferId", "type": "string"},
    {"name": "amount", "type": "string"}
  ]
}

// v2 — add optional field (BACKWARD compatible)
{
  "type": "record",
  "name": "TransferEvent",
  "fields": [
    {"name": "transferId", "type": "string"},
    {"name": "amount", "type": "string"},
    {"name": "currency", "type": ["null", "string"], "default": null}
  ]
}
```

Banking pratiği: Avro + Schema Registry = strong contract.

### 8. OpenAPI contract test

Provider OpenAPI spec generated. Consumer verifies adherence.

```xml
<dependency>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
</dependency>
```

Generate consumer client from OpenAPI:
```bash
openapi-generator-cli generate -i account-service-openapi.json -g java -o ./generated-client
```

Schema diff CI:
```bash
openapi-diff old-spec.json new-spec.json --fail-on-incompatible
```

Banking REST API → OpenAPI **lightweight contract test**.

### 9. Banking contract patterns

**Pattern 1: Money precision contract**

```groovy
body([
    amount: $(producer(100.00), consumer(matching("\\d+\\.\\d{2}")))
])
```

Always 2 decimal places banking.

**Pattern 2: ISO date format**

```groovy
body([
    timestamp: $(producer(execute('iso8601')), consumer(matching("\\d{4}-\\d{2}-\\d{2}T.*Z")))
])
```

**Pattern 3: Optional vs mandatory**

```groovy
// Mandatory
body([
    transferId: $(anyUuid()),
    amount: $(any())
])

// Optional with default
body([
    transferId: $(anyUuid()),
    description: $(optional("Transfer"))
])
```

**Pattern 4: Status enum**

```groovy
body([
    status: $(producer("COMPLETED"), consumer(regex("INITIATED|COMPLETED|FAILED|REVERSED")))
])
```

### 10. Banking — contract testing anti-pattern'leri

**Anti-pattern 1: No contract / end-to-end only**
- Slow feedback. CDC essential for microservice.

**Anti-pattern 2: Contract test in same module**
- Provider value lost. Stub publish + consumer pull pattern.

**Anti-pattern 3: Loose matchers everywhere**
```groovy
body([
    amount: $(any())   // anything matches
])
```
Banking için **strict types** (regex with format, decimal precision).

**Anti-pattern 4: Schema migration without contract verify**
- Avro BACKWARD broken → consumer crash. CI gate.

**Anti-pattern 5: Provider state setup leak**
- @State setup modifies shared state. Per-test isolation.

**Anti-pattern 6: Pact broker public**
- Banking pact = API design (security sensitive). Private broker.

**Anti-pattern 7: Contract = bloated test**
- 50 fields per contract. Test core fields only.

**Anti-pattern 8: No can-i-deploy gate**
- CI passes but production fail because consumer's contract not verified. Gate.

**Anti-pattern 9: Mock provider not running stub**
- Mock = independent test data. Stub = published contract. Banking için stub.

**Anti-pattern 10: Stub version drift**
- Consumer uses old stub, provider new. Banking version pinning + CI verify.

---

## Önemli olabilecek araştırma kaynakları

- Spring Cloud Contract docs
- Pact docs + Pact Broker
- Confluent Schema Registry
- Apicurio Registry
- "Building Microservices" — Sam Newman (Ch. contract testing)

---

## Mini task'ler

### Task 12.5.1 — Spring Cloud Contract provider (60 dk)

Account service contract. POST /debit endpoint. Generated test pass.

### Task 12.5.2 — Stub publish + consumer use (60 dk)

Provider deploy stub to local maven repo. Consumer use @AutoConfigureStubRunner.

### Task 12.5.3 — Pact JUnit5 consumer (60 dk)

Transfer service → account service contract. Pact JSON generate.

### Task 12.5.4 — Pact Broker local (45 dk)

Docker broker. Publish + verify.

### Task 12.5.5 — Kafka contract (60 dk)

Transfer event contract. Provider publish stub + consumer verify.

### Task 12.5.6 — Avro Schema Registry (60 dk)

Confluent SR local. Avro schema evolution test (BACKWARD compat).

### Task 12.5.7 — OpenAPI contract diff (45 dk)

Generate OpenAPI. openapi-diff with breaking change detect.

### Task 12.5.8 — can-i-deploy gate CI (45 dk)

Pact broker can-i-deploy script. CI fail if not verified.

### Task 12.5.9 — Banking strict matchers (30 dk)

Money decimal precision, ISO date, enum regex.

### Task 12.5.10 — Provider state with @State (45 dk)

3 different states. Each test independent.

---

## Test yazma rehberi

```java
// Consumer test with Pact
@ExtendWith(PactConsumerTestExt.class)
class TransferAccountIntegrationTest {
    
    @Pact(consumer = "transfer-service", provider = "account-service")
    public RequestResponsePact debitContract(PactDslWithProvider builder) {
        return builder
            .given("account acc-001 exists with balance 1000")
            .uponReceiving("a debit of 100 TRY")
                .path("/v1/accounts/acc-001/debit")
                .method("POST")
                .body(new PactDslJsonBody()
                    .decimalMatcher("amount", "\\d+\\.\\d{2}", 100.00)
                    .stringValue("currency", "TRY")
                    .uuid("reference"))
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .stringValue("accountId", "acc-001")
                    .decimalMatcher("balanceAfter", "\\d+\\.\\d{2}", 900.00)
                    .stringMatcher("status", "DEBITED|FAILED", "DEBITED"))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "debitContract")
    void shouldDebitViaAccountService(MockServer mockServer) {
        AccountClient client = new AccountClient(mockServer.getUrl());
        
        DebitResult result = client.debit("acc-001", new BigDecimal("100"), "TRY");
        
        assertThat(result.balanceAfter()).isEqualByComparingTo("900.00");
        assertThat(result.status()).isEqualTo("DEBITED");
    }
}
```

---

## Claude-verify prompt

```
Contract testing setup'ımı banking-grade kriterlere göre değerlendir:

1. Framework:
   - Spring Cloud Contract veya Pact?
   - Banking polyglot need varsa Pact?
   - Pact Broker / Spring Cloud Contract stubs?

2. Coverage:
   - REST endpoint contracts?
   - Kafka event contracts?
   - Avro Schema Registry?
   - OpenAPI diff?

3. Banking patterns:
   - Money decimal precision matcher?
   - ISO date format?
   - Status enum regex?
   - UUID format?

4. CI integration:
   - Contract tests in CI?
   - can-i-deploy gate?
   - Schema compatibility check?

5. Multi-environment:
   - Pact broker tags (environment)?
   - Verify against current production version?
   - Block deploy if not verified?

6. State management:
   - Provider @State independent?
   - No shared mutable state?

7. Stub distribution:
   - Maven classifier stubs?
   - Pact broker?
   - Version pinning?

8. Anti-pattern:
   - End-to-end only YOK?
   - Loose matchers YOK?
   - Schema migration uncontrolled YOK?
   - Provider state leak YOK?
   - Bloated contracts YOK?
   - No can-i-deploy gate YOK?

9. Banking event evolution:
   - Avro BACKWARD compat?
   - Default values for new fields?
   - Deprecated field strategy?

10. Documentation:
    - Contract = API design doc?
    - Auto-generated from contracts?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Spring Cloud Contract setup (provider)
- [ ] Stub publish + consumer use
- [ ] Pact consumer + provider
- [ ] Pact Broker integration
- [ ] Kafka event contract
- [ ] Avro Schema Registry compat
- [ ] OpenAPI diff
- [ ] can-i-deploy CI gate
- [ ] Banking strict matchers (money, ISO date, enum)
- [ ] 5+ contract test

---

## Defter notları (10 madde)

1. "Consumer-Driven Contracts vs end-to-end test microservice trade-off: ____."
2. "Spring Cloud Contract provider-driven vs Pact consumer-driven banking pick: ____."
3. "Stub publish (Maven classifier) + consumer @AutoConfigureStubRunner workflow: ____."
4. "Pact Broker can-i-deploy CI gate banking deploy safety: ____."
5. "Kafka event contract + Spring Cloud Contract message stub: ____."
6. "Avro Schema Registry BACKWARD compat banking event evolution: ____."
7. "Banking strict matcher (money decimal, ISO date, enum) precision: ____."
8. "OpenAPI diff REST contract breaking change detect: ____."
9. "Provider @State independent isolation per pact verification: ____."
10. "Contract = API design doc + auto-generated docs banking pattern: ____."
