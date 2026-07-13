# Phase 12 Mini-Project — Banking Testing Excellence

## Hedef

Phase 7-11 banking microservice'leri için **comprehensive testing excellence**: JUnit 5 + Mockito + Testcontainers + ArchUnit + Contract tests + Mutation testing + Performance CI. Banking-grade quality bar — TR banka denetim hazır.

## Süre

12-15 gün (günde 3 saat)

## Önbilgi

- Phase 11 mini-project tamam
- Topic 12.1-12.7 bitti

---

## Görev listesi

### 1. JUnit 5 advanced setup (1 gün)

Per service:
- @ParameterizedTest matrix (IBAN, day count, fraud rules)
- @TestFactory dynamic (per bank code, per currency)
- @Nested organization (3+ level)
- Custom assertions (TransferAssert, LedgerAssert, AuditAssert)
- Custom extensions (LedgerInvariantExtension, MdcCleanupExtension)
- @Tag (unit / integration / slow / banking-domain)
- Maven profile selective

### 2. Mockito banking patterns (1 gün)

Per service unit tests:
- @MockitoSettings STRICT_STUBS
- @InjectMocks + @Captor banking pattern
- BDDMockito given/willReturn/then/should
- 5+ banking scenario per service (sanctions, KKB, CB, saga, idempotency)
- @Spy partial mock LedgerService
- InOrder verification (saga compensation)
- verifyNoInteractions PCI boundary

### 3. Testcontainers integration tests (2 gün)

```java
@SpringBootTest
@Testcontainers
abstract class AbstractBankingIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withInitScript("db/banking-init.sql")
        .withReuse(true);
    
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.1"));
    
    @Container
    static KeycloakContainer keycloak = new KeycloakContainer()
        .withRealmImportFile("/banking-realm.json")
        .withReuse(true);
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Container
    static LocalStackContainer localstack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:3.0"))
        .withServices(KMS, S3);
    
    @DynamicPropertySource
    static void dynamicProps(DynamicPropertyRegistry r) {
        r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> keycloak.getAuthServerUrl() + "/realms/banking");
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        r.add("aws.kms.endpoint", () -> localstack.getEndpointOverride(KMS).toString());
    }
    
    @BeforeEach
    @Sql(scripts = "/db/clean.sql")
    void cleanState() {}
    
    @AfterEach
    void verifyLedgerInvariant() {
        BigDecimal diff = trialBalanceService.compute();
        assertThat(diff).isEqualByComparingTo(ZERO);
    }
}
```

Banking integration test 25+ per service:
- Repository test (real Postgres)
- REST controller test
- Kafka producer/consumer roundtrip
- Keycloak token authorization
- LocalStack KMS encrypt/decrypt
- Cross-service via stub

### 4. ArchUnit rules — banking architecture (1 gün)

50+ ArchUnit rule:

**Hexagonal:**
- Domain isolation
- Application no adapter
- Layered architecture

**Naming:**
- Controller/Repository/Service/Port suffix

**Banking domain:**
- Money fields BigDecimal
- No double for money
- PII fields @Convert
- Ledger access only LedgerService
- PAN restricted to card module
- Audit immutable
- @PreAuthorize on sensitive
- @IdempotencyKey on POST transfer

**Security:**
- No field injection
- No System.out.println
- No catch-all exception
- No hardcoded secret

**Spring:**
- @Transactional on service
- No Repository in Controller

### 5. Contract tests — Spring Cloud Contract (2 gün)

10+ contract per service pair:

**Account ↔ Transfer:**
- Debit (success + insufficient)
- Credit
- Balance lookup
- Lock for transfer

**Transfer ↔ Ledger:**
- Journal post
- Reverse

**Card ↔ Compliance:**
- Authorization → fraud check

**Transfer ↔ Notification:**
- Transfer-events Kafka contract

Pact broker local + can-i-deploy gate.

Avro Schema Registry banking events.

### 6. Mutation testing — PIT (1 gün)

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses>
            <param>com.bank.transfer.domain.*</param>
            <param>com.bank.transfer.application.service.*</param>
            <param>com.bank.ledger.domain.*</param>
            <param>com.bank.loan.domain.InterestCalculator</param>
            <param>com.bank.loan.domain.AmortizationCalculator</param>
            <param>com.bank.compliance.rule.*</param>
        </targetClasses>
        <mutators>
            <mutator>STRONGER</mutator>
        </mutators>
        <mutationThreshold>85</mutationThreshold>
        <coverageThreshold>90</coverageThreshold>
        <features>
            <feature>+GIT(from=origin/main)</feature>   <!-- PR-only -->
        </features>
    </configuration>
</plugin>
```

Banking critical classes ≥ 90% mutation score.

### 7. Performance CI (1.5 gün)

GitHub Actions:

```yaml
name: Banking Performance Gate

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'   # Nightly

jobs:
  jmh-regression:
    if: github.event_name == 'pull_request'
    runs-on: banking-benchmark-runner
    steps:
      - name: PR benchmark
      - name: Baseline benchmark  
      - name: Compare + comment + gate (10%)
  
  smoke:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy PR preview
      - name: k6 smoke test
  
  nightly-load:
    if: github.event_name == 'schedule'
    runs-on: banking-perf-runner
    steps:
      - name: Deploy staging fresh
      - name: Seed data
      - name: k6 load (30 min, 1000 RPS)
      - name: Compare baseline
      - name: Slack on regression
  
  weekly-soak:
    if: github.event_name == 'schedule' && contains(github.event.schedule, 'sunday')
    timeout-minutes: 480
    steps:
      - name: k6 soak (4 hours)
      - name: Memory grow check
      - name: JFR heap dump compare
```

JMH benchmarks:
- Money arithmetic (BigDecimal)
- Serialization (Jackson vs Avro)
- Lock (synchronized vs Atomic)
- Cache (Caffeine vs Redis)
- IBAN validation
- Encryption (AES-GCM)
- DB query (single vs batch)

### 8. Banking quality dashboard (1 gün)

Grafana panel:
- Test count per service
- Coverage % (Jacoco)
- Mutation score % (PIT)
- Failed contract verifications
- ArchUnit violations
- Benchmark trend (latest 30 days)
- Load test p99 / error rate
- Soak test memory grow trend

```promql
# Mutation score per service
banking_mutation_score{service=~".*"}

# Coverage trend
banking_coverage_percent{service=~".*"} - banking_coverage_percent{service=~".*"} offset 7d

# Benchmark trend
banking_jmh_score{benchmark="transferLatency"}

# Contract verifications
sum(banking_contract_verifications_total{status="success"}) / sum(banking_contract_verifications_total)
```

### 9. Banking-specific test scenarios (2 gün)

#### Senaryo 1: Ledger invariant under load
- 1000 concurrent transfers
- After: trial balance == 0
- No journal entry orphan
- All ledger entries balanced

#### Senaryo 2: Idempotency duplicate
- Same idempotency key 3x parallel
- Result: 1 transfer, 1 journal, 1 event

#### Senaryo 3: Sanctions screening real-time
- Sanctioned counterparty
- Transfer rejected
- Audit logged
- No ledger posting
- No event published

#### Senaryo 4: Fraud detection smurfing
- 5 transactions just below threshold in 1 hour
- MASAK alert generated
- Compliance officer notification
- STR draft

#### Senaryo 5: Saga compensation
- 3-step cross-bank transfer
- Step 3 fails
- Compensation reverse order
- Final ledger consistent

#### Senaryo 6: KVKK right to be forgotten
- Customer deletion request
- Crypto-shred verify
- Audit trail preserved (pseudonymized)
- Subsequent decrypt fails

#### Senaryo 7: Encryption tampering
- Tampered ciphertext at rest
- AEAD AuthenticationFailure
- Original data not exposed

#### Senaryo 8: Refresh token rotation reuse detection
- Used refresh re-used
- All tokens revoked
- Security alert
- User notification

### 10. CI integration end-to-end (0.5 gün)

```yaml
# .github/workflows/banking-quality-gate.yml
name: Banking Quality Gate

on:
  pull_request:
    branches: [main, develop]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: mvn -B test
  
  integration:
    runs-on: ubuntu-latest
    services: [postgres, kafka]
    steps:
      - run: mvn -B verify -Pintegration
  
  archunit:
    runs-on: ubuntu-latest
    steps:
      - run: mvn -B test -Dtest='*ArchitectureTest'
  
  contract:
    runs-on: ubuntu-latest
    steps:
      - run: mvn -B test -Dtest='*ContractTest'
      - run: pact-broker can-i-deploy --pacticipant transfer-service
  
  mutation:
    runs-on: ubuntu-latest
    steps:
      - run: mvn -B pitest:mutationCoverage -DwithHistory
  
  benchmark:
    runs-on: banking-benchmark-runner
    steps:
      - name: PR vs main JMH
      - name: 10% regression gate
  
  smoke:
    runs-on: ubuntu-latest
    needs: [unit, integration]
    steps:
      - name: k6 smoke
  
  quality-gate-merged:
    needs: [unit, integration, archunit, contract, mutation, benchmark, smoke]
    runs-on: ubuntu-latest
    steps:
      - name: All gates passed
        run: echo "PR ready to merge"
```

### 11. Defter notları (20 madde)

1. "JUnit 5 advanced (parametric + dynamic + @Nested + custom assertions) banking matrix: ____."
2. "Mockito STRICT_STUBS + @Captor + BDDMockito + @Spy banking scenarios: ____."
3. "Testcontainers shared singleton + @ServiceConnection + per-test clean: ____."
4. "LedgerInvariantExtension @AfterEach trial balance check banking: ____."
5. "ArchUnit hexagonal + naming + banking-domain (money, PII, ledger access): ____."
6. "Spring Cloud Contract provider + stub + consumer banking pattern: ____."
7. "Pact Broker + can-i-deploy CI gate banking deploy safety: ____."
8. "Kafka event contract + Avro Schema Registry BACKWARD compat: ____."
9. "PIT STRONGER mutators + BIG_DECIMAL banking critical (Ledger/Interest/Fraud): ____."
10. "Banking mutation %90+ critical class + git diff PR-only: ____."
11. "JMH benchmark CI + PR vs main + 10% regression gate + PR comment: ____."
12. "k6 SLO-driven thresholds + banking realistic load model: ____."
13. "Nightly load + soak weekly + pre-release full suite banking: ____."
14. "Banking integration 25+ scenario per service (ledger + sanctions + fraud + saga): ____."
15. "8 banking-specific kasten kırma senaryo end-to-end test: ____."
16. "Banking quality dashboard Grafana (coverage + mutation + contract + benchmark): ____."
17. "Dedicated CI runner banking benchmark stable env (CPU pin + turbo off): ____."
18. "Test pyramid banking (unit ~70% + integration ~25% + e2e ~5%): ____."
19. "Banking test domain language (TransferAssert, LedgerAssert, AuditAssert): ____."
20. "Senior banking engineer mindset: behavior testing + property + mutation + contract: ____."

---

## Tamamlama kriterleri

- [ ] JUnit 5 advanced features per service
- [ ] Mockito STRICT_STUBS + banking scenarios
- [ ] Testcontainers full stack (PG + Kafka + Keycloak + Redis + LocalStack)
- [ ] LedgerInvariantExtension banking
- [ ] ArchUnit 50+ rule
- [ ] Contract tests (Spring Cloud Contract + Pact + Pact Broker + can-i-deploy)
- [ ] Avro Schema Registry banking events
- [ ] PIT mutation testing critical class ≥ 85%
- [ ] JMH PR regression gate
- [ ] k6 SLO-driven CI
- [ ] Nightly load + weekly soak
- [ ] 8 banking-specific kasten kırma senaryo
- [ ] Banking quality dashboard
- [ ] 20 defter notu

---

## Önemli not

Phase 12 = **quality assurance maturity**. Banking için bu phase:

- BDDK regulatory: test coverage + mutation testing audit-able
- KVKK / PCI-DSS: contract + mutation + ArchUnit privacy guard
- Production confidence: integration + contract + perf gates
- Senior engineer behavior: test design + quality bar set

TR bankalarında **mid-senior engineer testing skills** dağıtılır:
- Junior: writes tests
- Mid: writes good tests
- Senior: designs test strategy + quality gates + team conventions

Phase 12 → Senior banking engineer.

---

## 12 phase complete

Phase 1-12 tamamlayan **complete TR banking backend engineer**:

```
Phase 1: Foundation ✓
Phase 2: Hexagonal + DDD ✓
Phase 3: JPA + Postgres ✓
Phase 4: SQL + Oracle ✓
Phase 5: Spring Batch ✓
Phase 6: Kafka + Messaging ✓
Phase 7: Microservices ✓
Phase 8: Security ✓
Phase 9: Observability ✓
Phase 10: Banking Domain ✓
Phase 11: DevOps ✓
Phase 12: Testing ✓
```

**TR banka mid+ / senior backend engineer interview'a hazırsın.**

Sırada:
- Mock interview pratiği
- Real banking case study writeup
- Open-source contribution (Spring / banking lib)
- Conference talk submission (BDDK / Yazılım Tarafı / Devnot)
