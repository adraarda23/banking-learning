# Topic 12.2 — Mockito Deep

## Hedef

Mockito framework banking-grade derinlikte: @Mock/@Spy/@Captor/@InjectMocks, when/thenReturn/thenThrow/thenAnswer, verify (interaction testing), ArgumentCaptor, ArgumentMatcher, BDDMockito (given/willReturn), MockedStatic, deep stubs, strict stubbing, banking domain scenarios (fraud rule mock, KKB stub, MASAK alert verify), anti-patterns.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- JUnit 5 (Topic 12.1) bitti
- Spring dependency injection kavramı
- Mock vs Stub vs Fake ayrımı duy

---

## Kavramlar

### 1. Test doubles — terminology

| Type | Anlam | Banking örnek |
|---|---|---|
| **Dummy** | Hiçbir şey yapmaz, placeholder | unused parameter |
| **Stub** | Pre-canned response | KKB returns 1500 score |
| **Mock** | Behavior verification | MASAK alert was called |
| **Spy** | Real object + intercept | Wrap LedgerService real |
| **Fake** | Working impl simplified | In-memory repo |

Mockito = mostly **Stub + Mock + Spy**. Fake genelde manuel sınıf.

### 2. Mockito setup

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
```

```java
@ExtendWith(MockitoExtension.class)
class TransferServiceTest {
    
    @Mock AccountRepository accountRepo;
    @Mock LedgerService ledgerService;
    @Mock KafkaTemplate<String, Object> kafkaTemplate;
    @Mock SanctionsService sanctionsService;
    
    @InjectMocks TransferService transferService;
    
    @Captor ArgumentCaptor<JournalEntryRequest> journalCaptor;
    
    @Test
    void shouldDebitAndCreditOnTransfer() {
        // given
        when(accountRepo.findById(any())).thenReturn(Optional.of(testAccount()));
        
        // when
        transferService.transfer(testRequest());
        
        // then
        verify(ledgerService).post(journalCaptor.capture());
        assertThat(journalCaptor.getValue().description()).contains("Transfer");
    }
}
```

### 3. Stubbing patterns

```java
// Basic
when(accountRepo.findById(accountId)).thenReturn(Optional.of(account));

// Multiple invocations — different responses
when(kkbService.getScore("12345678901"))
    .thenReturn(1500)
    .thenReturn(1600)   // 2nd call
    .thenThrow(new ServiceUnavailableException("KKB down"));   // 3rd

// Throw
when(sanctionsService.check(any()))
    .thenThrow(new SanctionsHitException("OFAC match"));

// Dynamic (Answer)
when(feeService.calculateFee(any()))
    .thenAnswer(invocation -> {
        TransferRequest req = invocation.getArgument(0);
        return req.amount().multiply(new BigDecimal("0.001"));
    });

// Void method
doNothing().when(notificationService).send(any());
doThrow(new NotificationException()).when(notificationService).send(any());
doAnswer(inv -> { ... return null; }).when(notificationService).send(any());
```

### 4. ArgumentMatchers

```java
// any()
when(accountRepo.findById(any())).thenReturn(Optional.of(account));
when(accountRepo.findById(any(UUID.class))).thenReturn(Optional.of(account));

// specific value
when(accountRepo.findById(eq(accountId))).thenReturn(Optional.of(account));

// banking specific
when(sanctionsService.check(argThat(req -> 
    req.country().equals("IR") || req.country().equals("KP"))))
    .thenReturn(SanctionsResult.hit());

// combined
when(transferService.process(
    eq(accountA),
    argThat(amt -> amt.compareTo(new BigDecimal("10000")) > 0)))
    .thenReturn(approveWithMfaRequired);
```

**Important:** Mixing literal + matcher illegal:
```java
when(repo.find(eq(1L), "x"))   // ❌ mixed
when(repo.find(eq(1L), eq("x")))   // ✓
```

### 5. verify — interaction testing

```java
@Test
void shouldNotifyOnHighValueTransfer() {
    TransferRequest req = TransferRequest.builder()
        .amount(new BigDecimal("500000"))
        .build();
    
    transferService.transfer(req);
    
    verify(notificationService).sendHighValueAlert(any());
    verify(complianceService, atLeastOnce()).recordTransaction(any());
}

@Test
void shouldNeverCallBlockedCustomer() {
    when(customerRepo.findById(any()))
        .thenReturn(Optional.of(blockedCustomer));
    
    assertThatThrownBy(() -> transferService.transfer(req))
        .isInstanceOf(CustomerBlockedException.class);
    
    verify(ledgerService, never()).post(any());
    verify(kafkaTemplate, never()).send(anyString(), any());
}

@Test
void shouldNotCallExternalApiInRollback() {
    // ...
    
    verifyNoInteractions(externalApiClient);
    // OR
    verifyNoMoreInteractions(externalApiClient);
}
```

### 6. ArgumentCaptor — banking verification depth

```java
@Test
void shouldPostBalancedJournalEntry() {
    transferService.transfer(buildRequest(customerA, customerB, "100", "TRY"));
    
    verify(ledgerService).post(journalCaptor.capture());
    JournalEntryRequest captured = journalCaptor.getValue();
    
    assertThat(captured.entries())
        .hasSize(2)
        .extracting(LedgerEntryRequest::accountCode)
        .containsExactly("2101-CustomerA", "2101-CustomerB");
    
    BigDecimal totalDebit = captured.entries().stream()
        .map(LedgerEntryRequest::debit).filter(Objects::nonNull)
        .reduce(ZERO, BigDecimal::add);
    BigDecimal totalCredit = captured.entries().stream()
        .map(LedgerEntryRequest::credit).filter(Objects::nonNull)
        .reduce(ZERO, BigDecimal::add);
    
    assertThat(totalDebit).isEqualByComparingTo(totalCredit);
}

@Test
void shouldEmitTransferEventWithCorrectPayload() {
    transferService.transfer(testRequest);
    
    ArgumentCaptor<TransferInitiatedEvent> eventCaptor = 
        ArgumentCaptor.forClass(TransferInitiatedEvent.class);
    verify(kafkaTemplate).send(eq("transfer-events"), eventCaptor.capture());
    
    TransferInitiatedEvent event = eventCaptor.getValue();
    assertThat(event.transferId()).isEqualTo(testRequest.transferId());
    assertThat(event.amount()).isEqualByComparingTo(testRequest.amount());
    assertThat(event.timestamp()).isAfter(Instant.now().minusSeconds(10));
}
```

Captor banking için **invaluable** — published event/journal/audit payload assertion.

### 7. @Spy — partial mock

```java
@Spy
LedgerService ledgerService = new LedgerService(realDependencies);

@Test
void shouldUseRealMethodButStubOne() {
    doReturn(BigDecimal.ZERO)
        .when(ledgerService)
        .balanceOf(anyString(), anyString());
    
    // Other methods run real
    ledgerService.post(realJournalRequest);
}
```

**Note:** `when().thenReturn()` on Spy → real method **runs** to determine return type, then overrides. For void or risky: `doReturn().when()`.

Banking: Spy real `LedgerService` ama `balanceOf` slow → stub balance.

### 8. BDDMockito — given/when/then

```java
import static org.mockito.BDDMockito.*;

@Test
void givenSufficientBalance_whenTransfer_thenSucceed() {
    // given
    given(accountRepo.findById(accountA)).willReturn(Optional.of(account));
    given(account.getBalance()).willReturn(new BigDecimal("1000"));
    
    // when
    TransferResult result = transferService.transfer(testRequest);
    
    // then
    then(ledgerService).should().post(any());
    then(kafkaTemplate).should().send(eq("transfer-events"), any());
    assertThat(result.status()).isEqualTo("COMPLETED");
}
```

Banking BDD style readability.

### 9. MockedStatic — static method mock

```java
@Test
void shouldUseFrozenClockInTest() {
    Instant frozen = Instant.parse("2024-05-12T10:30:00Z");
    
    try (MockedStatic<Instant> mocked = mockStatic(Instant.class)) {
        mocked.when(Instant::now).thenReturn(frozen);
        
        Transfer t = transferService.transfer(req);
        assertThat(t.createdAt()).isEqualTo(frozen);
    }
}
```

**Better:** Inject `Clock` to avoid static mock. Static mock = last resort.

```java
public class TransferService {
    private final Clock clock;   // ✓
    
    public Transfer transfer(...) {
        Instant now = Instant.now(clock);   // testable
        ...
    }
}
```

### 10. Strict stubbing — banking quality

```java
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.STRICT_STUBS)   // default
class TransferTest { ... }
```

Strict stubbing fail-fast:
- Unused stub → fail
- Argument mismatch → clear error

Banking: **STRICT_STUBS standard** — production-grade rigor.

### 11. Banking domain — Mockito scenarios

#### Scenario 1: MASAK alert verification

```java
@Test
void shouldRaiseMasakAlertOnSmurfingPattern() {
    List<Transaction> smurfingTxs = generateSmurfingPattern(customerA);
    
    when(transactionRepo.findByCustomerSince(any(), any()))
        .thenReturn(smurfingTxs);
    
    transactionMonitor.onTransaction(currentTx);
    
    ArgumentCaptor<ComplianceAlert> alertCaptor = ArgumentCaptor.forClass(ComplianceAlert.class);
    verify(alertRepo).save(alertCaptor.capture());
    
    ComplianceAlert alert = alertCaptor.getValue();
    assertThat(alert.getRule()).isEqualTo("SmurfingDetection");
    assertThat(alert.getSeverity()).isEqualTo(Severity.HIGH);
    
    verify(complianceOpsNotifier).notify(any());
}
```

#### Scenario 2: KKB cache hit

```java
@Test
void shouldCacheKkbScore() {
    when(kkbClient.getScore("12345678901")).thenReturn(new ScoreResponse(1500));
    
    kkbService.getScore("12345678901");
    kkbService.getScore("12345678901");
    kkbService.getScore("12345678901");
    
    verify(kkbClient, times(1)).getScore(any());   // Cached after 1st
}
```

#### Scenario 3: Circuit breaker fallback

```java
@Test
void shouldUseFallbackWhenRiskServiceDown() {
    when(riskClient.assess(any()))
        .thenThrow(new ResourceAccessException("Connection refused"));
    
    RiskAssessment result = riskService.assess(req);
    
    assertThat(result.score()).isEqualTo(DEFAULT_HIGH_RISK_SCORE);   // Fallback
    verify(metricsCounter).increment("risk_service_fallback");
}
```

#### Scenario 4: Saga compensation

```java
@Test
void shouldTriggerCompensationOnPartialFailure() {
    // Setup: first 2 steps succeed, 3rd fails
    doReturn(SagaStep.completed("debit"))
        .when(accountStep).execute(any());
    doReturn(SagaStep.completed("publish"))
        .when(kafkaStep).execute(any());
    doThrow(new RuntimeException("External error"))
        .when(remoteBankStep).execute(any());
    
    SagaResult result = sagaOrchestrator.run(testSaga);
    
    assertThat(result.status()).isEqualTo("COMPENSATED");
    
    // Verify compensation order: reverse of completed steps
    InOrder inOrder = inOrder(kafkaStep, accountStep);
    inOrder.verify(kafkaStep).compensate(any());
    inOrder.verify(accountStep).compensate(any());
}
```

#### Scenario 5: Idempotency

```java
@Test
void shouldReturnExistingResultOnDuplicateIdempotencyKey() {
    String idempotencyKey = "key-abc";
    Transfer existing = createTestTransfer();
    
    when(idempotencyRepo.findByKey(idempotencyKey))
        .thenReturn(Optional.of(existing));
    
    Transfer result = transferService.transferWithIdempotency(req, idempotencyKey);
    
    assertThat(result.id()).isEqualTo(existing.id());   // Same response
    verify(ledgerService, never()).post(any());          // No second posting
}
```

### 12. Banking — Mockito anti-pattern'leri

**Anti-pattern 1: Mocking value objects**
```java
@Mock BigDecimal amount;   // ❌
@Mock String iban;
```
Value objects = simple constructors. Use real.

**Anti-pattern 2: Mocking what you don't own**
```java
@Mock LocalDate fixedDate;   // ❌
@Mock Files files;
```
Wrap external in adapter. Mock your adapter.

**Anti-pattern 3: Overspecified verification**
```java
verify(repo).save(captor.capture());
verify(repo, times(1)).save(any());   // duplicate of above
verifyNoMoreInteractions(repo);        // overly strict
```
Banking için: business-relevant verifications only.

**Anti-pattern 4: when() everywhere**
```java
when(account.getId()).thenReturn(...);
when(account.getBalance()).thenReturn(...);
when(account.getOwner()).thenReturn(...);
```
Real object more readable. Mocking everything = anti-test smell.

**Anti-pattern 5: Test setup > test logic**
30 satır setup + 2 satır assertion. Test smell — over-mocking. Refactor production code.

**Anti-pattern 6: Mock'ing private methods**
PowerMock road. Don't go. Refactor to public/package-private.

**Anti-pattern 7: Mock static (excessive)**
MockedStatic OK for legacy. New code: inject Clock, dependency.

**Anti-pattern 8: Mock final classes via PowerMock**
Mockito-inline supports final. Don't need PowerMock.

**Anti-pattern 9: Test repeats production logic**
```java
when(feeCalc.compute(amount)).thenReturn(amount.multiply(0.001));
```
Test verifies "service called fee calc" not "fee value". Real fee calc class OK.

**Anti-pattern 10: No strict stubs**
Loose stubs leave dead code in tests. Strict by default.

---

## Önemli olabilecek araştırma kaynakları

- Mockito documentation
- "Pragmatic Unit Testing in Java 8 with JUnit" — Hunt/Thomas
- "Effective Software Testing" — Aniche
- Mockito FAQ / wiki

---

## Mini task'ler

### Task 12.2.1 — Basic stub + verify (30 dk)

TransferService test. accountRepo stub. ledgerService verify.

### Task 12.2.2 — ArgumentCaptor journal (60 dk)

Transfer → capture JournalEntryRequest → assert balanced + correct accounts.

### Task 12.2.3 — Multiple invocations stub (45 dk)

KKB query → first call returns 1500, second 1600, third throws.

### Task 12.2.4 — BDDMockito style (30 dk)

given/willReturn + then/should. Banking BDD readability.

### Task 12.2.5 — @Spy partial mock (45 dk)

LedgerService spy: balanceOf stubbed (slow), other methods real.

### Task 12.2.6 — MockedStatic + Clock alternative (45 dk)

Static Instant.now() mock + better: Clock injection refactor.

### Task 12.2.7 — Strict stubbing exposure (30 dk)

`STRICT_STUBS`. Unused stub fail.

### Task 12.2.8 — MASAK alert verify (60 dk)

Smurfing scenario. Alert captured + assert fields + notify call verified.

### Task 12.2.9 — Saga compensation InOrder (45 dk)

3-step saga. Step 3 fails. InOrder verify compensation reverse.

### Task 12.2.10 — Idempotency Mockito (45 dk)

Duplicate idempotency key. Verify ledger.post NEVER called second time.

---

## Test yazma rehberi

```java
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.STRICT_STUBS)
class TransferServiceTest {
    
    @Mock AccountRepository accountRepo;
    @Mock LedgerService ledgerService;
    @Mock SanctionsService sanctionsService;
    @Mock KafkaTemplate<String, Object> kafkaTemplate;
    @Mock IdempotencyRepository idempotencyRepo;
    
    @InjectMocks TransferService transferService;
    
    @Captor ArgumentCaptor<JournalEntryRequest> journalCaptor;
    @Captor ArgumentCaptor<TransferInitiatedEvent> eventCaptor;
    
    @Test
    void shouldExecuteValidTransfer() {
        // given
        Account fromAccount = createAccount(customerA, new BigDecimal("1000"));
        Account toAccount = createAccount(customerB, new BigDecimal("500"));
        TransferRequest req = TransferRequest.builder()
            .fromAccount(fromAccount.getId())
            .toAccount(toAccount.getId())
            .amount(new BigDecimal("100"))
            .currency("TRY")
            .build();
        
        given(accountRepo.findById(fromAccount.getId()))
            .willReturn(Optional.of(fromAccount));
        given(accountRepo.findById(toAccount.getId()))
            .willReturn(Optional.of(toAccount));
        given(sanctionsService.check(any())).willReturn(SanctionsResult.clear());
        given(idempotencyRepo.findByKey(any())).willReturn(Optional.empty());
        
        // when
        TransferResult result = transferService.transfer(req);
        
        // then
        assertThat(result.status()).isEqualTo("COMPLETED");
        
        then(ledgerService).should().post(journalCaptor.capture());
        JournalEntryRequest journal = journalCaptor.getValue();
        assertThat(journal.entries()).hasSize(2);
        
        then(kafkaTemplate).should().send(eq("transfer-events"), eventCaptor.capture());
        TransferInitiatedEvent event = eventCaptor.getValue();
        assertThat(event.amount()).isEqualByComparingTo("100");
    }
    
    @Test
    void shouldRejectWhenSanctioned() {
        given(sanctionsService.check(any()))
            .willReturn(SanctionsResult.hit("OFAC list"));
        
        assertThatThrownBy(() -> transferService.transfer(req))
            .isInstanceOf(SanctionsViolationException.class)
            .hasMessageContaining("OFAC");
        
        then(ledgerService).should(never()).post(any());
        then(kafkaTemplate).should(never()).send(anyString(), any());
    }
    
    @Test
    void shouldReturnExistingOnDuplicateIdempotencyKey() {
        Transfer existing = Transfer.builder().id(UUID.randomUUID()).build();
        given(idempotencyRepo.findByKey("key-abc"))
            .willReturn(Optional.of(existing));
        
        Transfer result = transferService.transferWithKey(req, "key-abc");
        
        assertThat(result.id()).isEqualTo(existing.id());
        then(ledgerService).should(never()).post(any());
    }
}
```

---

## Claude-verify prompt

```
Mockito test setup'ımı banking-grade kriterlere göre değerlendir:

1. Setup:
   - @ExtendWith(MockitoExtension.class)?
   - @MockitoSettings strictness=STRICT_STUBS?
   - @Mock/@InjectMocks/@Spy/@Captor proper usage?

2. Stubbing:
   - when().thenReturn() basic?
   - thenThrow for negative scenarios?
   - thenAnswer for dynamic responses?
   - doReturn/doThrow for void or spy?

3. Verification:
   - verify() interaction count?
   - verify(never()) negative?
   - InOrder for sequence?
   - verifyNoInteractions for isolation?

4. ArgumentCaptor:
   - Banking journal entry capture?
   - Event payload capture?
   - Field-level assertion?

5. ArgumentMatcher:
   - any() / eq() / argThat() correctly?
   - All-or-nothing rule respected?

6. Banking scenarios:
   - MASAK alert verify?
   - KKB cache verify?
   - Circuit breaker fallback?
   - Saga compensation InOrder?
   - Idempotency check?

7. @Spy:
   - Partial mock real method + stub specific?
   - doReturn().when() for spy?

8. BDDMockito:
   - given/willReturn/then/should?
   - Banking readability?

9. MockedStatic:
   - Only legacy / Clock injection preferred?

10. Anti-pattern:
    - Mocking value objects YOK?
    - Mocking what you don't own YOK?
    - Overspecified verification YOK?
    - Mock everywhere (no real obj) YOK?
    - Test repeats production logic YOK?
    - Mock private/static excessive YOK?
    - No strict stubs YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] @ExtendWith MockitoExtension + STRICT_STUBS
- [ ] @Mock + @InjectMocks pattern
- [ ] @Captor banking journal + event capture
- [ ] BDDMockito given/willReturn/then/should
- [ ] @Spy partial mock + doReturn
- [ ] MockedStatic + Clock alternative
- [ ] 5 banking domain scenario (MASAK, KKB, CB, Saga, Idempotency)
- [ ] InOrder verification
- [ ] verifyNoInteractions banking isolation
- [ ] 10+ test

---

## Defter notları (10 madde)

1. "Mockito test doubles (Mock/Stub/Spy/Fake) banking ayrımı: ____."
2. "@MockitoSettings STRICT_STUBS production-grade default: ____."
3. "when/thenReturn/thenThrow/thenAnswer stubbing banking variants: ____."
4. "ArgumentCaptor banking journal + event payload field-level assert: ____."
5. "ArgumentMatchers (any/eq/argThat) + all-or-nothing rule: ____."
6. "verify (times/never/atLeastOnce) + InOrder banking saga compensation: ____."
7. "@Spy partial mock + doReturn().when() for safe stub: ____."
8. "BDDMockito given/willReturn/then/should banking readability: ____."
9. "MockedStatic legacy + Clock injection preferred testable: ____."
10. "Anti-pattern (mock value, what you don't own, overspecified verify, over-mock): ____."
