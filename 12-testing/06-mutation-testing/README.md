# Topic 12.6 — Mutation Testing: PIT

## Hedef

Mutation testing ile **test kalitesini** ölçmek: coverage %85 ama mutation %50 = test'ler aslında zayıf. PIT (Pitest) banking için yaygın, Spring Boot/Maven entegrasyonu, mutator types, mutation score, banking critical class hedef, CI integration, performance, false positive analizi.

## Süre

Okuma: 1.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- JUnit 5 + Mockito (Topic 12.1, 12.2) bitti
- Code coverage Jacoco yetersizliğini biliyor

---

## Kavramlar

### 1. Mutation testing — niye?

**Code coverage:** "Kod satırı çalıştı." Anlamı sınırlı.

```java
public int divide(int a, int b) {
    return a / b;
}

@Test
void test() {
    divide(10, 2);   // Coverage: %100. Assertion yok!
}
```

Coverage **execution**, not **verification**.

**Mutation testing:**

1. Production code'a değişiklik yap (mutant)
2. Test suite run
3. Mutant **caught** (test fail) → good (mutant killed)
4. Mutant **survives** (test pass) → bad (test does NOT verify behavior)

```
Original: return a / b;
Mutant 1: return a * b;   (replace / with *)
Mutant 2: return a + b;
Mutant 3: return a - b;
Mutant 4: return 0;

If test was: assertEquals(5, divide(10, 2)) → mutants 1,2,3,4 all killed.
If test was: divide(10, 2) (no assert) → 0 mutants killed.

Mutation score = killed / total = 0/4 = %0
```

**Mutation score** = test quality metric.

### 2. PIT (Pitest) — banking yaygın

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.bank.transfer.domain.*</param>
            <param>com.bank.transfer.application.*</param>
        </targetClasses>
        <targetTests>
            <param>com.bank.transfer.*Test</param>
        </targetTests>
        <mutators>
            <mutator>DEFAULTS</mutator>
        </mutators>
        <mutationThreshold>80</mutationThreshold>   <!-- Banking gate -->
        <coverageThreshold>85</coverageThreshold>
        <outputFormats>
            <param>HTML</param>
            <param>XML</param>
        </outputFormats>
        <timestampedReports>false</timestampedReports>
    </configuration>
</plugin>
```

Run:
```bash
mvn org.pitest:pitest-maven:mutationCoverage
```

Output:
```
================================================================================
- Mutators
================================================================================
> org.pitest.mutationtest.engine.gregor.mutators.ConditionalsBoundaryMutator
> org.pitest.mutationtest.engine.gregor.mutators.IncrementsMutator
> org.pitest.mutationtest.engine.gregor.mutators.MathMutator
> org.pitest.mutationtest.engine.gregor.mutators.NegateConditionalsMutator
> org.pitest.mutationtest.engine.gregor.mutators.ReturnValsMutator
...

================================================================================
- Statistics
================================================================================
>> Generated 234 mutations Killed 198 (85%)
>> Ran 312 tests (1.34 tests per mutation)
>> Mutation score: 85% (threshold 80) PASS
```

HTML report: `target/pit-reports/index.html`.

### 3. Mutator types

**Default mutators (PIT standard):**

| Mutator | Anlam |
|---|---|
| `CONDITIONALS_BOUNDARY` | `<` ↔ `<=` |
| `INCREMENTS` | `i++` ↔ `i--` |
| `INVERT_NEGS` | `-i` ↔ `i` |
| `MATH` | `+` ↔ `-`, `*` ↔ `/`, etc. |
| `NEGATE_CONDITIONALS` | `==` ↔ `!=`, `>` ↔ `<=` |
| `RETURN_VALS` | `return true` ↔ `return false`, etc. |
| `VOID_METHOD_CALLS` | Remove void method calls |
| `EMPTY_RETURNS` | `return ""` ↔ `return null` |
| `FALSE_RETURNS` | `return false` ↔ `return true` |
| `TRUE_RETURNS` | Same flip |
| `NULL_RETURNS` | `return null` |
| `PRIMITIVE_RETURNS` | `return 0`, `return 1.0`, etc. |

**STRONGER mutators (banking için recommended):**

```xml
<mutators>
    <mutator>STRONGER</mutator>
</mutators>
```

Includes additional:
- `EXPERIMENTAL_REMOVE_INCREMENTS`
- `REMOVE_CONDITIONALS`
- `EXPERIMENTAL_BIG_DECIMAL`
- `EXPERIMENTAL_BIG_INTEGER`

**Banking için BIG_DECIMAL mutator özellikle değerli:**
- `BigDecimal.add()` → `BigDecimal.subtract()`
- `setScale(2, HALF_EVEN)` → `setScale(2, HALF_UP)` (different rounding!)

### 4. Banking-critical class analysis

**High-priority for mutation:**
- `LedgerService` (Topic 10.1)
- `InterestCalculator` (Topic 10.5)
- `FraudDetectionRule` (Topic 10.6)
- `AmortizationService`
- `IbanValidator` (Topic 10.4)
- `MaskingService` (PII)
- `EncryptionService`

These functions have business-critical logic. Mutation score **>= 90%** target.

**Lower-priority:**
- DTO mapping
- Configuration
- Logging

Mutation %60-70 OK.

### 5. Target class selection

```xml
<targetClasses>
    <!-- Banking core -->
    <param>com.bank.transfer.domain.*</param>
    <param>com.bank.transfer.application.service.*</param>
    
    <!-- Exclude DTO, config -->
    <excludedClasses>
        <param>*Dto</param>
        <param>*Config</param>
        <param>*Application</param>
    </excludedClasses>
</targetClasses>
```

### 6. PIT incremental analysis

Full mutation slow (10+ min banking projeleri).

```xml
<configuration>
    <withHistory>true</withHistory>
    <historyInputFile>target/pit-history.bin</historyInputFile>
    <historyOutputFile>target/pit-history.bin</historyOutputFile>
</configuration>
```

Subsequent runs skip unchanged classes.

**Pull request mode (PIT 1.15+):**

```xml
<features>
    <feature>+GIT(from=origin/main)</feature>
</features>
```

Only mutate **changed code** in PR → fast.

CI:
```bash
mvn pitest:mutationCoverage -DwithHistory -DhistoryInputFile=cache/pit-history.bin
```

### 7. Banking mutation example

```java
public class InterestCalculator {
    
    public BigDecimal computeInterest(BigDecimal principal, BigDecimal rate, int days) {
        return principal
            .multiply(rate)
            .multiply(BigDecimal.valueOf(days))
            .divide(BigDecimal.valueOf(365), 2, RoundingMode.HALF_EVEN);
    }
}
```

**Mutations:**
1. `multiply(rate)` → `divide(rate)` — Test catches?
2. `multiply(days)` → `multiply(days+1)` — Edge case test?
3. `BigDecimal.valueOf(365)` → `BigDecimal.valueOf(360)` — ACT/360 vs ACT/365 test?
4. `RoundingMode.HALF_EVEN` → `RoundingMode.HALF_UP` — Rounding test?
5. `setScale(2, ...)` → `setScale(4, ...)` — Precision test?

Strong test suite kills all 5. Weak suite kills 2-3.

**Test impact:**

```java
@Test
void shouldCalculateInterest() {
    BigDecimal result = calc.computeInterest(
        new BigDecimal("100000"),
        new BigDecimal("0.05"),
        365);
    
    assertThat(result).isEqualByComparingTo("5000.00");
}
```

Kills mutation 2 (days+1 yields 5013.70). Doesn't kill mutation 3 (multiply vs divide if test only checks one value).

**Better:**

```java
@ParameterizedTest
@CsvSource({
    "100000, 0.05, 365, 5000.00",     // Full year
    "100000, 0.05, 90, 1232.88",      // 90 days
    "100000, 0.05, 0, 0.00",          // Zero days
    "100000, 0.10, 365, 10000.00",    // Different rate
    "0,      0.05, 365, 0.00"          // Zero principal
})
void shouldCalculateInterestForVariousScenarios(...) {
    BigDecimal result = calc.computeInterest(...);
    assertThat(result).isEqualByComparingTo(expected);
}

@Test
void shouldUseHalfEvenRounding() {
    BigDecimal result = calc.computeInterest(
        new BigDecimal("100000"),
        new BigDecimal("0.05"),
        100);
    
    // 100000 * 0.05 * 100 / 365 = 1369.86301... 
    // HALF_EVEN rounds 1369.86 (5 → even)
    // HALF_UP rounds 1369.87
    assertThat(result).isEqualByComparingTo("1369.86");
}
```

Now kills mutators 3, 4, 5 too. Mutation score climbs.

### 8. PIT CI integration

```yaml
- name: Mutation Testing
  run: |
    mvn -B pitest:mutationCoverage \
      -DwithHistory \
      -DhistoryInputFile=cache/pit-history.bin \
      -DhistoryOutputFile=cache/pit-history.bin
  
- name: Upload PIT report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: pit-report
    path: target/pit-reports/
  
- name: Check mutation threshold
  run: |
    SCORE=$(xmllint --xpath "string(//mutationCoverage/totals/percentageMutationCoverage)" \
        target/pit-reports/mutations.xml)
    echo "Mutation score: $SCORE%"
    if (( $(echo "$SCORE < 80" | bc -l) )); then
      echo "Mutation score below threshold"
      exit 1
    fi
```

PIT fails build if `mutationThreshold` not met (config).

### 9. False positives — equivalent mutants

```java
public int absoluteValue(int n) {
    if (n < 0) return -n;
    return n;
}

Mutant: if (n <= 0) return -n;
```

For `n=0`: original returns `0`, mutant returns `-0 = 0`. Behavior **same** → no test can kill this mutant.

PIT marks as **equivalent**. Banking için: review report, accept some equivalent mutants.

### 10. Banking — mutation testing anti-pattern'leri

**Anti-pattern 1: Coverage'a güvenmek**
- %85 coverage ≠ %85 quality. Mutation kanıt.

**Anti-pattern 2: Mutation all-or-nothing**
- All classes %95+ unrealistic. Critical class %90+, others %60+.

**Anti-pattern 3: PIT every CI run (slow)**
- Full mutation 10-30 dk. Incremental + git diff.

**Anti-pattern 4: Ignore equivalent mutants**
- Review false positives. Suppress via PIT config.

**Anti-pattern 5: Refactor test to kill mutants**
- Test driven by mutation = mutation test smell. Test driven by behavior.

**Anti-pattern 6: Default mutators only banking için**
- BIG_DECIMAL mutator banking için kritik. STRONGER mutators.

**Anti-pattern 7: PIT for getters/setters**
- Trivial code, low value. Exclude.

**Anti-pattern 8: No threshold gate**
- Score reported ama enforce edilmiyor. Banking için CI gate.

**Anti-pattern 9: Mutation testing without good test**
- Mutation = test quality measure. Test isn't ready → mutation report meaningless.

**Anti-pattern 10: Mutation testing replacing other testing**
- Mutation = white-box. Doesn't replace integration / contract / e2e.

---

## Önemli olabilecek araştırma kaynakları

- PIT documentation
- "Mutation Testing for Java" — academic papers
- pitest.org blog

---

## Mini task'ler

### Task 12.6.1 — PIT setup + first run (30 dk)

Maven plugin. Simple class + test. Mutation score baseline.

### Task 12.6.2 — Banking InterestCalculator strong test (60 dk)

Yukarıdaki örnek. Mutators kill, score >= 90.

### Task 12.6.3 — STRONGER mutators (30 dk)

BIG_DECIMAL + REMOVE_CONDITIONALS etc. Banking için.

### Task 12.6.4 — Incremental + git diff (30 dk)

History file + `+GIT(from=origin/main)`. PR-only mutation.

### Task 12.6.5 — Exclude DTO/Config (30 dk)

Target class config. Critical banking classes only.

### Task 12.6.6 — IBAN validator mutation (45 dk)

MOD-97 algorithm. Mutate boundaries. Test kills all.

### Task 12.6.7 — Fraud rule mutation (60 dk)

Smurfing detection threshold. Mutate count/amount. Test kills.

### Task 12.6.8 — Rounding mode mutation (45 dk)

BigDecimal HALF_EVEN. Mutate to HALF_UP. Test detect.

### Task 12.6.9 — CI integration + threshold (30 dk)

CI fail < 80%. Upload report artifact.

### Task 12.6.10 — Equivalent mutant analysis (30 dk)

Identify 2 equivalent mutants. Suppress via PIT config.

---

## Test yazma rehberi

```java
@DisplayName("InterestCalculator mutation-strong tests")
class InterestCalculatorTest {
    
    InterestCalculator calc = new InterestCalculator();
    
    @ParameterizedTest(name = "P={0} R={1} days={2} → I={3}")
    @CsvSource({
        "100000.00, 0.05, 365,  5000.00",
        "100000.00, 0.05, 90,   1232.88",
        "100000.00, 0.05, 1,    13.70",
        "100000.00, 0.05, 0,    0.00",
        "0,         0.05, 365,  0.00",
        "100000.00, 0,    365,  0.00",
        "100000.00, 0.10, 365,  10000.00",
        "100000.00, 0.05, 366,  5013.70",   // catches days+1 mutation
        "100000.00, 0.05, 730,  10000.00"
    })
    void shouldCalculateInterest(BigDecimal p, BigDecimal r, int d, BigDecimal expected) {
        BigDecimal actual = calc.compute(p, r, d);
        assertThat(actual).isEqualByComparingTo(expected);
    }
    
    @Test
    @DisplayName("Uses HALF_EVEN rounding (banker's rounding)")
    void shouldRoundHalfEven() {
        // 100000 * 0.05 * 100 / 365 = 1369.86301...
        // HALF_EVEN: 1369.86 (last even)
        // HALF_UP: 1369.87 (mutation)
        BigDecimal result = calc.compute(
            new BigDecimal("100000.00"), new BigDecimal("0.05"), 100);
        
        assertThat(result).isEqualByComparingTo("1369.86");
    }
    
    @Test
    @DisplayName("Uses ACT/365 day count (not ACT/360)")
    void shouldUse365DaysInYear() {
        // 100000 * 0.05 * 90 / 365 = 1232.88
        // ACT/360: 1250.00 (mutation)
        BigDecimal result = calc.compute(
            new BigDecimal("100000.00"), new BigDecimal("0.05"), 90);
        
        assertThat(result).isEqualByComparingTo("1232.88");
    }
    
    @Test
    @DisplayName("Negative days handled — input validation")
    void shouldRejectNegativeDays() {
        assertThatThrownBy(() -> calc.compute(
            new BigDecimal("100000"), new BigDecimal("0.05"), -1))
            .isInstanceOf(IllegalArgumentException.class);
    }
    
    @Test
    @DisplayName("Boundary at 1 day")
    void shouldHandleOneDayBoundary() {
        BigDecimal result = calc.compute(
            new BigDecimal("100000"), new BigDecimal("0.05"), 1);
        
        assertThat(result).isEqualByComparingTo("13.70");
    }
}
```

Expected PIT score for this test class: **95%+**.

---

## Claude-verify prompt

```
Mutation testing setup'ımı banking-grade kriterlere göre değerlendir:

1. PIT setup:
   - Maven plugin?
   - JUnit 5 plugin?
   - HTML + XML output?

2. Mutators:
   - DEFAULTS or STRONGER?
   - BIG_DECIMAL banking için?
   - REMOVE_CONDITIONALS?

3. Target selection:
   - Banking critical classes (Ledger, Interest, Fraud, IBAN)?
   - Exclude DTO/Config/Application?
   - High vs medium priority by domain?

4. Thresholds:
   - mutationThreshold (80+ recommended)?
   - coverageThreshold (85+)?
   - CI gate fail below?

5. Performance:
   - withHistory incremental?
   - Git diff (PR-only)?
   - PIT cache in CI?

6. Banking-strong tests:
   - InterestCalculator parametrize multiple scenarios?
   - Day count convention test kills 360 vs 365?
   - Rounding mode test (HALF_EVEN vs HALF_UP)?
   - IBAN check digit boundary?
   - Fraud rule thresholds?

7. CI integration:
   - PIT run in CI?
   - Report artifact upload?
   - Threshold enforce?
   - PR comment mutation score?

8. False positives:
   - Equivalent mutant review?
   - Suppress documented?

9. Banking critical %:
   - Ledger 95%+?
   - Interest calc 95%+?
   - Fraud rule 90%+?
   - DTO 60%+ (or excluded)?

10. Anti-pattern:
    - Coverage trust (mutation skip) YOK?
    - Full mutation every CI run YOK?
    - Refactor test just to kill mutant YOK?
    - Mutation for getters/setters YOK?
    - No threshold gate YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] PIT Maven plugin setup
- [ ] STRONGER mutators including BIG_DECIMAL
- [ ] Banking critical class target list
- [ ] Mutation threshold gate
- [ ] Incremental + git diff
- [ ] Banking-strong test (InterestCalculator)
- [ ] HALF_EVEN rounding test
- [ ] Day count convention test
- [ ] CI integration + threshold enforce
- [ ] Equivalent mutant suppression

---

## Defter notları (10 madde)

1. "Coverage vs Mutation score — execution vs verification banking sebebi: ____."
2. "PIT mutator types (DEFAULT vs STRONGER + BIG_DECIMAL banking): ____."
3. "Banking critical class hedef (Ledger, Interest, Fraud, IBAN, Encryption): ____."
4. "Mutation %90+ critical class + %60+ DTO trade-off: ____."
5. "Incremental + git diff PR-only PIT performance: ____."
6. "HALF_EVEN vs HALF_UP rounding mutation banking detect: ____."
7. "ACT/360 vs ACT/365 day count mutation banking detect: ____."
8. "CI gate mutation threshold + PR comment score: ____."
9. "Equivalent mutants false positive review + suppress: ____."
10. "Mutation = test quality metric + don't replace integration/e2e: ____."
