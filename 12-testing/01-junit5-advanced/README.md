# Topic 12.1 — JUnit 5 Advanced

## Hedef

JUnit 5 (Jupiter) framework'ünü banking-grade kullanım derinliğine getirmek: parametric tests (CsvSource, MethodSource, ArgumentsProvider), dynamic tests, @Nested organization, lifecycle (@BeforeAll/@BeforeEach + per-class), execution order + parallel, extensions (custom), assumptions, conditional tests, banking domain examples (IBAN validation matrix, day count convention matrix, MASAK rule scenarios), assertion strategies (AssertJ, JsonPath, custom matchers).

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- JUnit 4 → 5 farkını duy
- AssertJ kullandın
- Basic test yazıyorsun

---

## Kavramlar

### 1. JUnit 5 architecture

```
JUnit 5 = JUnit Platform + Jupiter (API) + Vintage (JUnit 4 bridge)

JUnit Platform: test launcher, IDE/build integration
Jupiter: new test API (@Test, @ParameterizedTest, etc.)
Vintage: run old JUnit 4 tests on Jupiter platform
```

Maven:
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
</dependency>
```

### 2. Lifecycle annotations

```java
class BankingServiceTest {
    
    @BeforeAll
    static void setupAll() {
        // Once per class — heavy setup (DB schema, etc.)
    }
    
    @BeforeEach
    void setUp() {
        // Before every test — fresh state
    }
    
    @Test
    void singleTest() { ... }
    
    @AfterEach
    void tearDown() { ... }
    
    @AfterAll
    static void tearDownAll() { ... }
}
```

**Per-class lifecycle:**

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class BankingServiceTest {
    
    @BeforeAll
    void setupAll() {   // Non-static OK
        // Shared instance across tests
    }
}
```

Banking pratiği: PER_METHOD default (fresh state); PER_CLASS for expensive setup.

### 3. Parameterized tests — high leverage

Aynı test logic, multiple input — banking için **kritik**.

#### @ValueSource

```java
@ParameterizedTest
@ValueSource(strings = {
    "TR320010009999987654321098",
    "TR430010009999987654321097",
    "TR540010009999987654321096"
})
void shouldValidateTrIban(String iban) {
    assertThat(IbanValidator.isValid(iban)).isTrue();
}
```

#### @CsvSource — multi-arg

```java
@ParameterizedTest(name = "IBAN {0} → bank code {1}")
@CsvSource({
    "TR320010009999987654321098, 00100",
    "TR320010009999987654321098, 00100",
    "TR430000209999987654321097, 00002"
})
void shouldExtractBankCode(String iban, String expectedBank) {
    assertThat(IbanValidator.extractBankCode(iban)).isEqualTo(expectedBank);
}
```

#### @CsvFileSource — file-based

```java
// resources/test-data/transfer-scenarios.csv
// fromAccount,toAccount,amount,currency,expectedStatus,expectedFee
ACC001,ACC002,100,TRY,SUCCESS,5.00
ACC001,ACC003,5000,TRY,SUCCESS,15.00
ACC001,ACC004,100000,TRY,FAILED,0.00
```

```java
@ParameterizedTest
@CsvFileSource(resources = "/test-data/transfer-scenarios.csv", numLinesToSkip = 1)
void shouldExecuteTransferScenarios(
    String from, String to, BigDecimal amount, String currency,
    String expectedStatus, BigDecimal expectedFee
) {
    TransferRequest req = new TransferRequest(from, to, amount, currency);
    TransferResult result = transferService.execute(req);
    
    assertThat(result.status()).isEqualTo(expectedStatus);
    assertThat(result.fee()).isEqualByComparingTo(expectedFee);
}
```

#### @MethodSource — programmatic

```java
@ParameterizedTest(name = "[{index}] {0}")
@MethodSource("dayCountFractionScenarios")
void shouldComputeDayCountFraction(
    String description,
    DayCountConvention convention,
    LocalDate from,
    LocalDate to,
    BigDecimal expectedFraction
) {
    BigDecimal actual = convention.fraction(from, to);
    assertThat(actual).isEqualByComparingTo(expectedFraction);
}

static Stream<Arguments> dayCountFractionScenarios() {
    return Stream.of(
        Arguments.of("90 days ACT/365",
            ACT_365, LocalDate.of(2024,1,1), LocalDate.of(2024,4,1),
            new BigDecimal("0.2493150685")),
        Arguments.of("90 days ACT/360",
            ACT_360, LocalDate.of(2024,1,1), LocalDate.of(2024,4,1),
            new BigDecimal("0.2527777778")),
        Arguments.of("Leap year Feb 28-29 ACT/365",
            ACT_365, LocalDate.of(2024,2,28), LocalDate.of(2024,2,29),
            new BigDecimal("0.0027397260")),
        Arguments.of("30/360 month end edge case",
            THIRTY_360, LocalDate.of(2024,1,31), LocalDate.of(2024,2,28),
            new BigDecimal("0.0777777778"))
    );
}
```

#### @EnumSource

```java
@ParameterizedTest
@EnumSource(value = TransferType.class, names = {"EFT", "FAST", "SWIFT"})
void shouldHaveFee(TransferType type) {
    assertThat(feeService.getFee(type)).isGreaterThan(ZERO);
}
```

#### @ArgumentsSource — custom provider

```java
@ParameterizedTest
@ArgumentsSource(MasakSmurfingScenarioProvider.class)
void shouldDetectSmurfingScenarios(SmurfingScenario scenario) {
    boolean detected = monitoringService.detectSmurfing(scenario.transactions());
    assertThat(detected).isEqualTo(scenario.expectedDetection());
}

public class MasakSmurfingScenarioProvider implements ArgumentsProvider {
    
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
        return Stream.of(
            Arguments.of(scenario("5 tx just below 10k threshold", 
                generateTransactions(5, 9500, ofHours(1)),
                true)),
            Arguments.of(scenario("Random amounts, no pattern",
                generateRandomTransactions(10),
                false)),
            Arguments.of(scenario("Single large legitimate transfer",
                List.of(new Tx(150000)),
                false)),
            Arguments.of(scenario("3 tx below threshold (less than 5)",
                generateTransactions(3, 9500, ofMinutes(30)),
                false))
        );
    }
}
```

### 4. Dynamic tests — runtime-generated

```java
@TestFactory
Stream<DynamicTest> shouldValidateAllRegisteredBanks() {
    List<BankCode> registered = bankCodeRepo.findAll();
    
    return registered.stream()
        .map(bank -> dynamicTest(
            "Validate " + bank.getName() + " (" + bank.getCode() + ")",
            () -> {
                String testIban = generateTestIban(bank.getCode());
                assertThat(IbanValidator.isValid(testIban))
                    .as("IBAN with bank %s", bank.getName())
                    .isTrue();
            }));
}

@TestFactory
Collection<DynamicNode> ledgerInvariants() {
    return List.of(
        dynamicContainer("Debit-credit balance per currency",
            ledgerRepo.findAllCurrencies().stream()
                .map(ccy -> dynamicTest(
                    ccy + " balanced",
                    () -> {
                        BigDecimal debits = ledgerRepo.sumDebits(ccy);
                        BigDecimal credits = ledgerRepo.sumCredits(ccy);
                        assertThat(debits).isEqualByComparingTo(credits);
                    }))),
        dynamicTest("Trial balance overall zero",
            () -> {
                BigDecimal diff = trialBalance.computeOverall();
                assertThat(diff).isEqualByComparingTo(ZERO);
            })
    );
}
```

**Banking use:** Runtime-discovered test scenarios (every registered bank, every active currency, every active customer segment).

### 5. @Nested — test organization

```java
class TransferServiceTest {
    
    @Nested
    @DisplayName("Same-bank transfers (havale)")
    class HavaleTests {
        
        @Test
        void shouldDebitSourceAndCreditDestination() { ... }
        
        @Test
        void shouldSucceedWith24x7Availability() { ... }
        
        @Nested
        @DisplayName("with insufficient balance")
        class InsufficientBalance {
            
            @Test
            void shouldThrowAndNotModifyLedger() { ... }
            
            @Test
            void shouldAuditFailedAttempt() { ... }
        }
    }
    
    @Nested
    @DisplayName("EFT outgoing")
    class EftTests {
        
        @Test
        void shouldHoldInClearingAccount() { ... }
        
        @Test
        void shouldRejectOutsideWorkingHours() { ... }
        
        @Test
        void shouldApplyBsmv() { ... }
    }
    
    @Nested
    @DisplayName("FAST instant")
    class FastTests {
        
        @Test
        void shouldComplete10Seconds() { ... }
        
        @Test
        void shouldReject70kPlus() { ... }
    }
}
```

IDE'de tree shape görünür, readability artar.

### 6. @DisplayName + @DisplayNameGeneration

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class BankingTest {
    
    @Test
    void shouldDebitCustomerOnTransfer() { }
    // Display: "should Debit Customer On Transfer"
    
    @Test
    @DisplayName("Müşteri yetersiz bakiyede transfer reddedilmeli")
    void rejectInsufficientBalance() { }
    // Display: Türkçe açıklama
}
```

Banking: Türkçe display name + acceptance test readability.

### 7. Assertions strategy

#### Built-in assertions (yeterli değil banking için)

```java
assertEquals(expected, actual);
assertNotNull(obj);
```

#### AssertJ — fluent + rich

```java
assertThat(transfer.amount())
    .isPositive()
    .isLessThanOrEqualTo(new BigDecimal("100000"))
    .isEqualByComparingTo(expectedAmount);

assertThat(transfer)
    .extracting(Transfer::status, Transfer::currency)
    .containsExactly("COMPLETED", "TRY");

assertThat(ledgerEntries)
    .hasSize(2)
    .extracting(LedgerEntry::accountCode)
    .containsExactly("2101-A", "2101-B");

assertThat(response.headers())
    .containsKey("X-Idempotency-Key")
    .containsEntry("Content-Type", "application/json");
```

Banking için AssertJ **standard**.

#### Custom assertions — banking domain

```java
public class TransferAssert extends AbstractAssert<TransferAssert, Transfer> {
    
    public TransferAssert(Transfer actual) {
        super(actual, TransferAssert.class);
    }
    
    public static TransferAssert assertThat(Transfer actual) {
        return new TransferAssert(actual);
    }
    
    public TransferAssert isBalanced() {
        isNotNull();
        BigDecimal debits = actual.ledgerEntries().stream()
            .map(LedgerEntry::debit).reduce(ZERO, BigDecimal::add);
        BigDecimal credits = actual.ledgerEntries().stream()
            .map(LedgerEntry::credit).reduce(ZERO, BigDecimal::add);
        if (debits.compareTo(credits) != 0) {
            failWithMessage("Expected balanced (D == C) but D=%s, C=%s", debits, credits);
        }
        return this;
    }
    
    public TransferAssert hasFeeOf(BigDecimal expected) {
        isNotNull();
        if (actual.fee().compareTo(expected) != 0) {
            failWithMessage("Expected fee %s but was %s", expected, actual.fee());
        }
        return this;
    }
    
    public TransferAssert wasIdempotent() {
        // Check audit log for duplicate request
        ...
        return this;
    }
}

// Usage
TransferAssert.assertThat(result)
    .isBalanced()
    .hasFeeOf(new BigDecimal("5.00"))
    .wasIdempotent();
```

Test reads like spec.

### 8. Assumptions — conditional execution

```java
@Test
void integrationTestRequiresDatabase() {
    assumeTrue(isDatabaseAvailable(), "Database not available, skipping");
    
    // Test that needs DB
    ...
}

@Test
@EnabledIf("isBankingHours")
void onlyRunDuringBankingHours() { ... }

@Test
@DisabledOnOs(OS.WINDOWS)
void unixOnly() { ... }

@Test
@EnabledIfEnvironmentVariable(named = "RUN_INTEGRATION", matches = "true")
void integrationTest() { ... }
```

### 9. @TestMethodOrder + @Order

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OnboardingFlowTest {
    
    @Test
    @Order(1)
    void shouldCreateCustomer() { ... }
    
    @Test
    @Order(2)
    void shouldVerifyMernis() { ... }
    
    @Test
    @Order(3)
    void shouldQueryKkb() { ... }
    
    @Test
    @Order(4)
    void shouldOpenAccount() { ... }
}
```

**Warning:** Order dependency = anti-pattern usually. Each test should be **independent**. Order only for happy-path acceptance demos.

### 10. Parallel execution

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.config.dynamic.factor=2
```

**Banking caveat:**
- Test isolation şart (DB state, MDC, static state)
- `@Execution(SAME_THREAD)` mark for non-thread-safe tests
- Testcontainers per-class shared OK with REUSABLE option

```java
@Execution(ExecutionMode.SAME_THREAD)
class GlobalStateMutatingTest { ... }

@Execution(ExecutionMode.CONCURRENT)
class StatelessTest { ... }
```

### 11. Tagging — selective run

```java
@Tag("unit")
class UnitTest { }

@Tag("integration")
@Tag("slow")
class IntegrationTest { }

@Tag("banking-domain")
class LedgerTest { }
```

Maven:
```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <groups>unit</groups>           <!-- default test -->
        <excludedGroups>slow</excludedGroups>
    </configuration>
</plugin>
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <configuration>
        <groups>integration</groups>    <!-- verify phase -->
    </configuration>
</plugin>
```

```bash
# Unit tests only (fast, CI per PR)
mvn test

# Integration (verify, nightly)
mvn verify -Pintegration

# Specific tag
mvn test -Dgroups="banking-domain"
```

### 12. Custom extensions

Test-wide cross-cutting concerns (banking):

```java
public class BankingLedgerExtension implements 
    BeforeEachCallback, AfterEachCallback {
    
    @Override
    public void beforeEach(ExtensionContext context) {
        // Seed COA + clean ledger
        LedgerSeed.cleanAndSeed();
    }
    
    @Override
    public void afterEach(ExtensionContext context) {
        // Verify ledger invariants
        BigDecimal diff = trialBalance.compute();
        if (diff.compareTo(ZERO) != 0) {
            throw new AssertionError("Trial balance violation after test: " + diff);
        }
    }
}

@ExtendWith(BankingLedgerExtension.class)
class TransferServiceTest { ... }
```

**Banking domain extensions:**
- LedgerInvariantExtension (trial balance check)
- AuditChainExtension (hash chain verify)
- MdcCleanupExtension (Topic 9.1)
- TimeFreezeExtension (deterministic dates)

### 13. Banking — JUnit anti-pattern'leri

**Anti-pattern 1: Test depends on order**
```java
@Test void test1() { customer = create(); }
@Test void test2() { update(customer); }   // ❌ depends on test1
```
Each test independent.

**Anti-pattern 2: Shared mutable static state**
```java
static List<Transaction> transactions = new ArrayList<>();
```
Parallel + ordering issues. Use @BeforeEach fresh state.

**Anti-pattern 3: Magic numbers**
```java
assertThat(result).isEqualTo(123.45);
```
What is 123.45? Named constant:
```java
private static final BigDecimal EXPECTED_FEE_WITH_BSMV = new BigDecimal("123.45");
```

**Anti-pattern 4: Multiple assertions checking unrelated things**
```java
@Test
void everything() {
    assertThat(transfer.amount()).isPositive();
    assertThat(customer.email()).isNotNull();
    assertThat(account.balance()).isGreaterThan(0);
}
```
Split into focused tests.

**Anti-pattern 5: Test method names not descriptive**
```java
@Test void test1() { }
@Test void doStuff() { }
```
`shouldXxxWhenYyy` pattern.

**Anti-pattern 6: assertTrue without message**
```java
assertTrue(transferService.isAllowed(req));
```
On failure: "expected true got false". Useless. Use AssertJ with `as("...")`.

**Anti-pattern 7: Sleep in test**
```java
@Test void async() {
    triggerEvent();
    Thread.sleep(5000);   // ❌
    assertThat(...);
}
```
Flaky. Use Awaitility or proper async.

**Anti-pattern 8: System.out.println in test**
Logger usage. Or just AssertJ — no need to print.

**Anti-pattern 9: Mocking what you don't own**
```java
@Mock LocalDate fixedDate;   // ❌
```
Wrap external in adapter, mock adapter. Or use Clock injected.

**Anti-pattern 10: No banking-specific assertions**
Banking semantic checks (ledger balance, BSMV correct, MASAK trigger) wrapped as custom assertions.

---

## Önemli olabilecek araştırma kaynakları

- JUnit 5 user guide
- AssertJ documentation
- "Effective Software Testing" — Maurício Aniche
- "Growing Object-Oriented Software, Guided by Tests" — Freeman/Pryce
- xUnit Patterns — Gerard Meszaros

---

## Mini task'ler

### Task 12.1.1 — Parameterized test matrix (45 dk)

@CsvSource ile IBAN validation matrix (10 valid + 10 invalid case).

### Task 12.1.2 — @MethodSource banking (45 dk)

Day count fraction scenarios. 4 convention x 5 date range.

### Task 12.1.3 — @CsvFileSource transfer scenarios (45 dk)

CSV file 50 transfer case. Banking transfer service test.

### Task 12.1.4 — @TestFactory dynamic (60 dk)

Each registered bank → IBAN validation dynamic test.

### Task 12.1.5 — @Nested organization (60 dk)

TransferServiceTest 3 level nested (transfer type → success/failure → reason).

### Task 12.1.6 — Custom AssertJ extension (60 dk)

`TransferAssert` + `LedgerAssert` banking-domain.

### Task 12.1.7 — Custom extension (45 dk)

`LedgerInvariantExtension` after each verify trial balance.

### Task 12.1.8 — Parallel execution config (30 dk)

junit-platform.properties enable. Race condition reproduce + @Execution SAME_THREAD fix.

### Task 12.1.9 — Tagging + Maven profile (45 dk)

@Tag unit/integration/slow/banking. Maven profile selective.

### Task 12.1.10 — ArgumentsProvider MASAK (60 dk)

MasakSmurfingScenarioProvider with 6+ scenario.

---

## Test yazma rehberi

```java
@DisplayName("TransferService — banking transfer scenarios")
@TestMethodOrder(MethodOrderer.DisplayName.class)
class TransferServiceTest {
    
    @RegisterExtension
    static BankingLedgerExtension ledger = new BankingLedgerExtension();
    
    static TransferService service;
    
    @BeforeAll
    static void setupOnce() {
        service = new TransferService(...);
    }
    
    @Nested
    @DisplayName("Same-bank havale")
    class HavaleTests {
        
        @ParameterizedTest(name = "{0} TL transfer between same-bank customers")
        @ValueSource(strings = {"100", "1000", "10000", "100000"})
        @DisplayName("Various amount sizes")
        void shouldTransferSameBank(String amountStr) {
            BigDecimal amount = new BigDecimal(amountStr);
            TransferResult result = service.havale(testRequest(amount));
            
            TransferAssert.assertThat(result)
                .isBalanced()
                .isCompleted();
        }
        
        @Test
        @DisplayName("Reject when insufficient balance")
        void shouldRejectInsufficient() {
            setupAccountBalance(customerA, new BigDecimal("50.00"));
            
            assertThatThrownBy(() -> service.havale(transferOf(customerA, "100.00")))
                .isInstanceOf(InsufficientBalanceException.class)
                .hasMessageContaining("50.00")
                .hasMessageContaining("100.00");
            
            // Ledger should not have been modified
            assertThat(balanceService.balanceOf(customerA, "TRY"))
                .isEqualByComparingTo("50.00");
        }
    }
    
    @TestFactory
    @DisplayName("Day count conventions accuracy")
    Stream<DynamicTest> shouldComputeAllDayCountConventions() {
        return Stream.of(DayCountConvention.values())
            .map(convention -> dynamicTest(
                convention.name() + " for 90-day standard period",
                () -> {
                    BigDecimal fraction = convention.fraction(
                        LocalDate.of(2024, 1, 1),
                        LocalDate.of(2024, 4, 1));
                    assertThat(fraction)
                        .as("Fraction for %s", convention)
                        .isBetween(new BigDecimal("0.24"), new BigDecimal("0.26"));
                }));
    }
    
    @Test
    @Tag("integration")
    @Tag("slow")
    void fullEndToEndWithLedger() {
        // Comprehensive scenario
    }
}
```

---

## Claude-verify prompt

```
JUnit 5 test setup'ımı banking-grade kriterlere göre değerlendir:

1. Parameterized tests:
   - @ValueSource / @CsvSource / @MethodSource / @CsvFileSource banking matrix?
   - @ArgumentsProvider custom banking scenario?
   - Display name template ({0}, {1}) readable?

2. Dynamic tests:
   - @TestFactory runtime scenarios?
   - dynamicContainer hierarchy?

3. Organization:
   - @Nested logical grouping?
   - @DisplayName Türkçe veya readable?
   - @DisplayNameGeneration?

4. Assertions:
   - AssertJ fluent?
   - Custom domain assertions (TransferAssert, LedgerAssert)?
   - .as() failure message banking context?
   - Multiple assertions split?

5. Lifecycle:
   - @BeforeAll / @BeforeEach proper scope?
   - PER_METHOD vs PER_CLASS rationale?
   - @AfterEach cleanup?

6. Extensions:
   - Custom extensions banking domain?
   - LedgerInvariantExtension?
   - MdcCleanupExtension?

7. Tagging:
   - @Tag unit/integration/slow?
   - Maven profile separate run?

8. Parallel:
   - junit-platform.properties enabled?
   - @Execution SAME_THREAD non-thread-safe?
   - Testcontainers reusable?

9. Banking-specific:
   - IBAN validation matrix?
   - Day count convention matrix?
   - MASAK rule scenarios?
   - Ledger invariants?

10. Anti-pattern:
    - Order dependency YOK?
    - Shared mutable static YOK?
    - Magic numbers YOK?
    - Multi-assertion unrelated YOK?
    - test1/doStuff name YOK?
    - Sleep in test YOK?
    - Mocking what you don't own YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] @ParameterizedTest 4 source types kullanım
- [ ] @TestFactory dynamic banking
- [ ] @Nested 2+ level
- [ ] Custom AssertJ (TransferAssert, LedgerAssert)
- [ ] Custom extension (LedgerInvariantExtension)
- [ ] Parallel execution config + SAME_THREAD where needed
- [ ] @Tag unit/integration/slow + Maven profile
- [ ] CSV test data file
- [ ] ArgumentsProvider MASAK scenarios
- [ ] 10+ integration test
- [ ] DisplayName Turkish banking domain

---

## Defter notları (10 madde)

1. "JUnit 5 architecture (Platform + Jupiter + Vintage) + Jupiter API: ____."
2. "@ParameterizedTest 4 source (Value/Csv/Method/CsvFile) banking matrix: ____."
3. "@TestFactory dynamic tests runtime-discovered banking: ____."
4. "@Nested + @DisplayName test organization readability banking: ____."
5. "AssertJ fluent + custom domain assertions banking pattern: ____."
6. "@BeforeAll/@BeforeEach lifecycle + PER_CLASS vs PER_METHOD scope: ____."
7. "Custom extensions (LedgerInvariant, AuditChain, MdcCleanup) cross-cutting: ____."
8. "Parallel execution config + @Execution SAME_THREAD non-thread-safe: ____."
9. "@Tag unit/integration/slow + Maven Surefire/Failsafe profile: ____."
10. "Banking test anti-pattern (order dep, sleep, magic numbers, shared static): ____."
