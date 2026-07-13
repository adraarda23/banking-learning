# Topic 12.3 — Testcontainers

## Hedef

Testcontainers ile **real services in Docker** test'lerde kullanmak. PostgreSQL, Kafka, Keycloak, Redis, LocalStack (AWS mock), MongoDB, Vault, banking-grade integration test patterns: shared vs per-test container, reuse mode, network setup, Spring Boot integration, performance optimization, CI integration.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 12.1, 12.2 bitti
- Docker temel
- Spring Boot test slices

---

## Kavramlar

### 1. Testcontainers — niye?

**Mock vs Embedded vs Testcontainers:**

| | Mock | H2 / Embedded | Testcontainers |
|---|---|---|---|
| Production parity | Düşük | Orta (H2 = Postgres değil) | Yüksek |
| Speed | Çok hızlı | Hızlı | Orta (Docker start) |
| Banking için | Unit OK | Integration risky | Integration ideal |
| Setup complexity | Düşük | Düşük | Orta (Docker gerekli) |
| CI cost | Sıfır | Düşük | Orta |

**Banking kanıtlanmış değer:**
- Postgres SQL dialect, lock behavior, partitioning → H2'de yok
- Kafka consumer group, transaction → embedded kafka eksik
- Keycloak realm + token format → mock fragile

### 2. Setup

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
```

### 3. PostgreSQL basic

```java
@Testcontainers
class LedgerRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("banking_test")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("db/init-banking-schema.sql");
    
    @DynamicPropertySource
    static void datasourceProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired LedgerRepository repo;
    
    @Test
    void shouldPersistJournalEntry() {
        JournalEntry entry = repo.save(testEntry());
        assertThat(entry.getId()).isNotNull();
    }
}
```

`@Container` static → shared across all tests in class.

### 4. Shared container — performance

Banking large suite: per-test container start ≥ 5 sec × 100 test = 500 sec. **Slow**.

**Pattern 1: Singleton container (per JVM)**

```java
public class SharedContainers {
    
    public static final PostgreSQLContainer<?> POSTGRES = 
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("banking_test")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);
    
    static {
        POSTGRES.start();   // Once per JVM
    }
}
```

```java
@SpringBootTest
@Import(BankingTestConfig.class)
abstract class AbstractIntegrationTest {
    
    @DynamicPropertySource
    static void registerProps(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", SharedContainers.POSTGRES::getJdbcUrl);
        r.add("spring.datasource.username", SharedContainers.POSTGRES::getUsername);
        r.add("spring.datasource.password", SharedContainers.POSTGRES::getPassword);
    }
}
```

**Pattern 2: Reuse mode**

`~/.testcontainers.properties`:
```
testcontainers.reuse.enable=true
```

`withReuse(true)`. Container stays running across test runs (faster locally).

CI: reuse off (clean state).

### 5. Kafka container

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.5.1"))
    .withEmbeddedZookeeper();

@DynamicPropertySource
static void kafkaProps(DynamicPropertyRegistry r) {
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}

@Test
void shouldPublishTransferEvent() {
    transferService.transfer(req);
    
    ConsumerRecord<String, String> record = KafkaTestUtils.getSingleRecord(
        testConsumer, "transfer-events", Duration.ofSeconds(5));
    
    assertThat(record.value()).contains(req.transferId().toString());
}
```

### 6. Keycloak container

```java
@Container
static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
    .withRealmImportFile("/banking-realm.json");

@DynamicPropertySource
static void keycloakProps(DynamicPropertyRegistry r) {
    r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
        () -> keycloak.getAuthServerUrl() + "/realms/banking");
}

@Test
void shouldAuthorizeWithValidJwt() {
    String token = obtainAccessToken("ahmet", "test123", "banking-web");
    
    mockMvc.perform(get("/v1/accounts/me")
            .header("Authorization", "Bearer " + token))
        .andExpect(status().isOk());
}
```

### 7. Redis container

```java
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
    .withExposedPorts(6379);

@DynamicPropertySource
static void redisProps(DynamicPropertyRegistry r) {
    r.add("spring.data.redis.host", redis::getHost);
    r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
}
```

### 8. LocalStack — AWS services

```java
@Container
static LocalStackContainer localstack = new LocalStackContainer(
    DockerImageName.parse("localstack/localstack:3.0"))
    .withServices(KMS, S3, SQS);

@DynamicPropertySource
static void awsProps(DynamicPropertyRegistry r) {
    r.add("aws.kms.endpoint", () -> localstack.getEndpointOverride(KMS).toString());
    r.add("aws.s3.endpoint", () -> localstack.getEndpointOverride(S3).toString());
    r.add("aws.region", localstack::getRegion);
    r.add("aws.access-key", localstack::getAccessKey);
    r.add("aws.secret-key", localstack::getSecretKey);
}

@Test
void shouldEncryptViaKms() {
    String encrypted = encryptionService.encrypt("sensitive data");
    String decrypted = encryptionService.decrypt(encrypted);
    
    assertThat(decrypted).isEqualTo("sensitive data");
}
```

Banking KMS test → real KMS API surface, LocalStack fakes.

### 9. Vault container

```java
@Container
static VaultContainer<?> vault = new VaultContainer<>("hashicorp/vault:1.15")
    .withVaultToken("test-root-token")
    .withInitCommand("secrets enable transit",
                     "write -f transit/keys/banking-pii");

@DynamicPropertySource
static void vaultProps(DynamicPropertyRegistry r) {
    r.add("spring.cloud.vault.uri", () -> 
        "http://" + vault.getHost() + ":" + vault.getMappedPort(8200));
    r.add("spring.cloud.vault.token", () -> "test-root-token");
}
```

### 10. Docker Compose container — full stack

```java
@Container
static DockerComposeContainer<?> stack = new DockerComposeContainer<>(
    new File("src/test/resources/docker-compose-test.yml"))
    .withExposedService("postgres", 5432, Wait.forListeningPort())
    .withExposedService("kafka", 9092)
    .withExposedService("keycloak", 8080, 
        Wait.forHttp("/health/ready").forStatusCode(200));
```

Banking full stack integration test.

### 11. Network isolation

```java
@Container
static Network network = Network.newNetwork();

@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
    .withNetwork(network)
    .withNetworkAliases("banking-db");

@Container
static GenericContainer<?> app = new GenericContainer<>("banking/transfer-service:test")
    .withNetwork(network)
    .withEnv("DB_HOST", "banking-db")
    .dependsOn(postgres);
```

Banking PCI scope: NetworkPolicy emulate.

### 12. Database state management

```java
@SpringBootTest
@Testcontainers
class TransferIntegrationTest extends AbstractIntegrationTest {
    
    @BeforeEach
    @Sql(scripts = "/db/clean-banking-state.sql", 
         executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    void cleanDb() {
        // SQL clean executed before each test
    }
    
    // OR use @Transactional + rollback
}
```

**Strategy:**
- **Per-test rollback** (fast): `@Transactional` on test
- **Per-test SQL clean** (slower, more thorough)
- **Per-class fresh schema** (most expensive, isolation strong)

Banking: ledger invariants → per-test clean recommended.

### 13. Banking — full integration test

```java
@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
class BankingFullStackTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withInitScript("db/init.sql");
    
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.1"));
    
    @Container
    static KeycloakContainer keycloak = new KeycloakContainer()
        .withRealmImportFile("/banking-realm.json");
    
    @DynamicPropertySource
    static void keycloakProps(DynamicPropertyRegistry r) {
        r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> keycloak.getAuthServerUrl() + "/realms/banking");
    }
    
    @Autowired MockMvc mockMvc;
    @Autowired LedgerService ledger;
    @Autowired KafkaTestConsumer kafkaConsumer;
    
    @Test
    void endToEndTransferFlow() throws Exception {
        String token = obtainToken("ahmet", "test123");
        
        // Initiate transfer
        mockMvc.perform(post("/v1/transfers/havale")
                .header("Authorization", "Bearer " + token)
                .header("X-Idempotency-Key", UUID.randomUUID().toString())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"fromAccount":"acc-A","toAccount":"acc-B","amount":"100","currency":"TRY"}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("COMPLETED"));
        
        // Verify ledger
        assertThat(ledger.balanceOf("acc-A", "TRY"))
            .isEqualByComparingTo("900");
        assertThat(ledger.balanceOf("acc-B", "TRY"))
            .isEqualByComparingTo("600");
        
        // Verify event published
        ConsumerRecord<String, String> event = kafkaConsumer.pollOne(
            "transfer-events", Duration.ofSeconds(5));
        assertThat(event.value()).contains("acc-A").contains("acc-B");
    }
}
```

`@ServiceConnection` (Spring Boot 3.1+) — auto-configure Spring properties.

### 14. CI optimization

```yaml
# GitHub Actions
- name: Cache Testcontainers
  uses: actions/cache@v4
  with:
    path: ~/.testcontainers
    key: tc-${{ runner.os }}
    
- name: Run integration tests
  env:
    TESTCONTAINERS_RYUK_DISABLED: "false"   # Cleanup containers
    TESTCONTAINERS_REUSE_ENABLE: "false"     # CI: clean
  run: ./mvnw -B verify -Pintegration
```

Banking CI: dedicated runner, parallel suite split (slow tests separate).

### 15. Banking — Testcontainers anti-pattern'leri

**Anti-pattern 1: H2 for integration tests**
- Postgres SQL dialect, lock, partitioning different. Banking için Testcontainers.

**Anti-pattern 2: Per-test container start**
- Slow. Singleton + clean state per-test.

**Anti-pattern 3: Hardcoded host/port**
- Mapped port dynamic. `getJdbcUrl()`, `getMappedPort()`.

**Anti-pattern 4: No container cleanup**
- Ryuk daemon auto-cleanup. Don't disable.

**Anti-pattern 5: Container in @BeforeEach**
- Start cost × test count. Use class-level static.

**Anti-pattern 6: Production image in test**
- Test uses official lightweight image. Production custom image separate.

**Anti-pattern 7: No init script**
- Empty DB → tests fragile. Init schema + seed.

**Anti-pattern 8: Test depends on previous test state**
- Per-test clean state. Banking ledger invariants.

**Anti-pattern 9: Mixing Testcontainers and Spring's @AutoConfigureTestDatabase**
- H2 auto-config overrides Testcontainers. Disable.

**Anti-pattern 10: Wrong Wait strategy**
- `Wait.forListeningPort()` says "TCP open" — not "app ready". Use health endpoint.

---

## Önemli olabilecek araştırma kaynakları

- Testcontainers documentation
- Spring Boot Testcontainers integration
- "Practical Test-Driven Development" — Edward Garson
- LocalStack docs (AWS mock)

---

## Mini task'ler

### Task 12.3.1 — PostgreSQL container basic (30 dk)

@Container + JDBC. Repository test save/find roundtrip.

### Task 12.3.2 — Singleton + reuse (30 dk)

Shared static container. Reuse mode local. Speed compare.

### Task 12.3.3 — Kafka container (45 dk)

Publish event + consume verify.

### Task 12.3.4 — Keycloak realm import (60 dk)

KeycloakContainer + banking-realm.json. Obtain token + use against secured endpoint.

### Task 12.3.5 — Redis container (30 dk)

Cache test (Caffeine + Redis). Cache hit verify.

### Task 12.3.6 — LocalStack KMS (45 dk)

Banking encryption Topic 8.6. Envelope encrypt + decrypt roundtrip.

### Task 12.3.7 — Vault transit (45 dk)

Vault container + transit engine. Encryption-as-a-service.

### Task 12.3.8 — Full stack (postgres + kafka + keycloak) (60 dk)

3 container + Spring Boot end-to-end test.

### Task 12.3.9 — Network isolation (45 dk)

Custom network + multiple containers + alias.

### Task 12.3.10 — Banking ledger invariant (60 dk)

After each test: trial balance check. Extension or @AfterEach.

---

## Test yazma rehberi

```java
@SpringBootTest
@Testcontainers
class BankingIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withInitScript("db/init-banking.sql")
        .withReuse(true);
    
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.1"));
    
    @BeforeEach
    @Sql(scripts = "/db/clean.sql")
    void cleanState() {}
    
    @AfterEach
    void verifyLedgerInvariant() {
        BigDecimal diff = trialBalanceService.compute();
        assertThat(diff)
            .as("Trial balance violation")
            .isEqualByComparingTo(ZERO);
    }
    
    @Test
    void completeBankingTransferFlow() {
        // ... full e2e test
    }
}
```

---

## Claude-verify prompt

```
Testcontainers setup'ımı banking-grade kriterlere göre değerlendir:

1. Container types:
   - PostgreSQL banking DB?
   - Kafka transactions/events?
   - Keycloak auth?
   - Redis cache/idempotency?
   - LocalStack KMS/S3?
   - Vault transit (banking encryption)?

2. Optimization:
   - Shared (singleton) container?
   - withReuse(true) local?
   - @ServiceConnection (Spring Boot 3.1+)?
   - CI reuse off / local on?

3. State management:
   - Per-test clean SQL?
   - Init script seed?
   - Banking ledger invariant after each?

4. Wait strategy:
   - Wait.forHttp("/health/ready") application-ready?
   - Not Wait.forListeningPort() (TCP only)?

5. Network:
   - Custom Network for multi-container?
   - Aliases?

6. Spring integration:
   - @DynamicPropertySource?
   - @ServiceConnection auto-config?

7. Banking-specific:
   - Realm import Keycloak?
   - Banking schema init?
   - Demo customer seed?
   - KMS key setup?

8. CI:
   - Dedicated runner?
   - Cache strategy?
   - Parallel suite split?

9. Cleanup:
   - Ryuk enabled (don't disable)?
   - Container leak yok?

10. Anti-pattern:
    - H2 for integration YOK?
    - Per-test container YOK?
    - Hardcoded host/port YOK?
    - No init script YOK?
    - Test state dependency YOK?
    - Wrong wait strategy YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Postgres + Kafka + Keycloak + Redis + LocalStack + Vault containers
- [ ] Singleton + reuse pattern
- [ ] @ServiceConnection Spring Boot 3.1+
- [ ] Init script + per-test clean
- [ ] Ledger invariant @AfterEach
- [ ] Network isolation (multi-container)
- [ ] Banking full-stack end-to-end test
- [ ] CI integration tested
- [ ] 8+ integration test
- [ ] LocalStack KMS encryption roundtrip

---

## Defter notları (10 madde)

1. "Testcontainers vs H2/Mock — production parity banking sebebi: ____."
2. "Singleton + withReuse + @ServiceConnection performance optimization: ____."
3. "Per-test state clean (SQL or @Transactional rollback) banking ledger: ____."
4. "Banking ledger invariant @AfterEach trial balance check: ____."
5. "Keycloak realm import + token obtain + secured endpoint test: ____."
6. "LocalStack KMS banking encryption integration test pattern: ____."
7. "Vault transit engine encryption-as-a-service test: ____."
8. "@ServiceConnection Spring Boot 3.1+ auto-config syntax: ____."
9. "Wait.forHttp + health-ready vs Wait.forListeningPort fragility: ____."
10. "CI Testcontainers cache + reuse off + dedicated runner banking: ____."
