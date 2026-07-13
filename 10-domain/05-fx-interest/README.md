# Topic 10.5 — FX & Interest Calculations

## Hedef

Banking finance hesaplamalarını derinlemesine öğrenmek: simple/compound interest, day count conventions (ACT/360, ACT/365, 30/360), Yield to Maturity, amortization (krediler), FX bid/ask spread, FX swap/forward, BSMV ve KKDF (TR vergi), accrual logic banking ledger (Topic 10.1), Effective Annual Rate (EAR), APR vs APY.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1 (Double-entry) bitti
- BigDecimal money handling
- Basic math (logarithm)

---

## Kavramlar

### 1. Simple interest

```
I = P × r × t

I: interest
P: principal
r: annual rate (decimal, 5% = 0.05)
t: time (years)
```

**Banking örnek:** 100,000 TL, %5 yıllık, 90 gün:

```
t = 90 / 365 = 0.2466 (assume ACT/365)
I = 100,000 × 0.05 × 0.2466 = 1,232.88 TL
```

Java:
```java
public BigDecimal simpleInterest(BigDecimal principal, BigDecimal rate, int days, int daysInYear) {
    return principal
        .multiply(rate)
        .multiply(BigDecimal.valueOf(days))
        .divide(BigDecimal.valueOf(daysInYear), 10, RoundingMode.HALF_EVEN);
}
```

### 2. Compound interest

```
A = P × (1 + r/n)^(n×t)

A: future value
P: principal
r: annual rate
n: compounding periods per year
t: time (years)
```

**Banking örnek:** 100,000 TL, %5 yıllık, 5 yıl, monthly compound:

```
n = 12
A = 100,000 × (1 + 0.05/12)^(12×5)
A = 100,000 × (1.00417)^60
A = 100,000 × 1.2834
A = 128,335.87 TL
```

Java:
```java
public BigDecimal compoundFutureValue(
    BigDecimal principal, 
    BigDecimal annualRate, 
    int compoundPerYear, 
    BigDecimal years
) {
    MathContext mc = new MathContext(20, RoundingMode.HALF_EVEN);
    
    // (1 + r/n)
    BigDecimal periodicRate = annualRate.divide(
        BigDecimal.valueOf(compoundPerYear), mc);
    BigDecimal base = BigDecimal.ONE.add(periodicRate);
    
    // n × t (total periods)
    int totalPeriods = years.multiply(BigDecimal.valueOf(compoundPerYear))
        .setScale(0, RoundingMode.HALF_EVEN).intValue();
    
    // pow
    BigDecimal factor = base.pow(totalPeriods, mc);
    
    return principal.multiply(factor)
        .setScale(2, RoundingMode.HALF_EVEN);
}
```

**Continuous compounding:**

```
A = P × e^(r×t)
```

Banking nadir (mostly theoretical), continuous-style fixed income models.

### 3. Day count conventions

**Critical for banking accuracy.** Aynı oran, farklı convention = farklı interest.

| Convention | Numerator | Denominator | Use case |
|---|---|---|---|
| **ACT/365** | Actual days | 365 | TR retail loans |
| **ACT/365F (Fixed)** | Actual days | 365 (even leap year) | Government bonds (some) |
| **ACT/360** | Actual days | 360 | TRY money market, USD short rate |
| **ACT/ACT** | Actual days | Actual days in year (365 or 366) | EUR bonds, US Treasury |
| **30/360** | 30 days/month assumed | 360 | US corporate bonds |
| **30E/360 (Eurobond)** | 30 days/month (European method) | 360 | European bonds |

**ACT/360 example:** 100,000 TL, %5, 90 gün:
```
I = 100,000 × 0.05 × 90/360 = 1,250.00 TL
```

ACT/365 ile karşılaştır (1,232.88 TL) — ACT/360 daha yüksek!

**Banking pratiği:**
- TR retail krediler: ACT/365
- TRY money market: ACT/360
- USD LIBOR (now SOFR): ACT/360
- EUR bonds: ACT/ACT

**Implementation:**

```java
public enum DayCountConvention {
    
    ACT_365 {
        public BigDecimal fraction(LocalDate from, LocalDate to) {
            long days = ChronoUnit.DAYS.between(from, to);
            return BigDecimal.valueOf(days)
                .divide(BigDecimal.valueOf(365), 10, RoundingMode.HALF_EVEN);
        }
    },
    
    ACT_360 {
        public BigDecimal fraction(LocalDate from, LocalDate to) {
            long days = ChronoUnit.DAYS.between(from, to);
            return BigDecimal.valueOf(days)
                .divide(BigDecimal.valueOf(360), 10, RoundingMode.HALF_EVEN);
        }
    },
    
    ACT_ACT {
        public BigDecimal fraction(LocalDate from, LocalDate to) {
            // Complex: split if year boundary crossed
            // ... see ISDA convention
        }
    },
    
    THIRTY_360 {
        public BigDecimal fraction(LocalDate from, LocalDate to) {
            int d1 = Math.min(from.getDayOfMonth(), 30);
            int d2 = (d1 == 30 && to.getDayOfMonth() == 31) 
                     ? 30 : Math.min(to.getDayOfMonth(), 30);
            
            int days = 360 * (to.getYear() - from.getYear())
                     + 30 * (to.getMonthValue() - from.getMonthValue())
                     + (d2 - d1);
            
            return BigDecimal.valueOf(days)
                .divide(BigDecimal.valueOf(360), 10, RoundingMode.HALF_EVEN);
        }
    };
    
    public abstract BigDecimal fraction(LocalDate from, LocalDate to);
}
```

### 4. Loan amortization

**French amortization (equal periodic payment):**

```
PMT = P × (r / (1 - (1 + r)^(-n)))

P: principal
r: periodic rate
n: number of periods
```

**Example:** 100,000 TL kredi, %15 yıllık, 60 ay:

```
r = 0.15 / 12 = 0.0125
n = 60
PMT = 100,000 × (0.0125 / (1 - (1.0125)^(-60)))
PMT = 100,000 × (0.0125 / 0.527594)
PMT = 100,000 × 0.0237899
PMT = 2,378.99 TL (monthly)
```

**Schedule:**

| Month | Beg.Balance | PMT | Interest | Principal | End Balance |
|---|---|---|---|---|---|
| 1 | 100,000.00 | 2,378.99 | 1,250.00 | 1,128.99 | 98,871.01 |
| 2 | 98,871.01 | 2,378.99 | 1,235.89 | 1,143.10 | 97,727.91 |
| ... |
| 60 | 2,348.50 | 2,378.99 | 29.36 | 2,349.63 | 0.00 |

Interest = Beg.Balance × r
Principal = PMT - Interest

```java
public class AmortizationSchedule {
    
    public List<Installment> generate(
        BigDecimal principal,
        BigDecimal annualRate,
        int months
    ) {
        MathContext mc = new MathContext(20, RoundingMode.HALF_EVEN);
        BigDecimal r = annualRate.divide(BigDecimal.valueOf(12), mc);
        BigDecimal onePlusR = BigDecimal.ONE.add(r);
        BigDecimal denominator = BigDecimal.ONE.subtract(onePlusR.pow(-months, mc));
        BigDecimal pmt = principal.multiply(r).divide(denominator, 2, RoundingMode.HALF_EVEN);
        
        List<Installment> schedule = new ArrayList<>();
        BigDecimal balance = principal;
        
        for (int i = 1; i <= months; i++) {
            BigDecimal interest = balance.multiply(r).setScale(2, RoundingMode.HALF_EVEN);
            BigDecimal principalPaid = pmt.subtract(interest);
            
            if (i == months) {
                // Last installment — adjust to clear residual
                principalPaid = balance;
                pmt = principalPaid.add(interest);
            }
            
            BigDecimal endBalance = balance.subtract(principalPaid);
            
            schedule.add(new Installment(i, balance, pmt, interest, principalPaid, endBalance));
            balance = endBalance;
        }
        
        return schedule;
    }
}
```

**Banking varyasyonlar:**
- **Eşit anapara:** Same principal, decreasing total payment
- **Balon:** Last payment large
- **Faizsiz dönemli:** First N months interest-only
- **Sabit faiz vs Değişken faiz** (TR LIBOR yerine TLREF — TR floating rate)

### 5. BSMV ve KKDF — TR banking vergi

**BSMV (Banka ve Sigorta Muameleleri Vergisi):**
- TR-only banking transactions tax
- Rate: %5 (varies by transaction type)
- Bank levies, remits to state monthly

**Applied to:**
- Loan interest income (BSMV %5 of interest)
- Commission income
- FX gains
- Money market income

**KKDF (Kaynak Kullanımı Destekleme Fonu):**
- Consumer loan support fund
- Rate: %10 (consumer loan interest)
- %0 (mortgage, vehicle loan in some cases)
- %15 (FX-indexed consumer loan)

**Banking impact:**

```
Customer takes 100,000 TL loan, 5 yıl, %2.5/ay:
  Pure interest: 17,250 TL (over loan)
  
  BSMV: 17,250 × 0.05 = 862.50 TL
  KKDF: 17,250 × 0.10 = 1,725.00 TL
  Total effective cost: 100,000 + 17,250 + 862.50 + 1,725 = 119,837.50 TL
```

**Effective rate calculation:**
- Stated rate (declared): %2.5/ay
- Effective rate (TÜFE compliant): include BSMV + KKDF
- TBB regulation: customer'a effective rate (yıllık maliyet oranı — YMO) gösterilmeli

```java
public BigDecimal calculateEffectiveAnnualCost(
    BigDecimal principal,
    BigDecimal monthlyRate,
    int months,
    BigDecimal bsmvRate,    // 0.05
    BigDecimal kkdfRate     // 0.10 consumer
) {
    BigDecimal totalInterest = ZERO;
    BigDecimal balance = principal;
    
    for (int i = 0; i < months; i++) {
        BigDecimal monthlyInterest = balance.multiply(monthlyRate);
        BigDecimal bsmv = monthlyInterest.multiply(bsmvRate);
        BigDecimal kkdf = monthlyInterest.multiply(kkdfRate);
        BigDecimal totalMonthly = monthlyInterest.add(bsmv).add(kkdf);
        
        totalInterest = totalInterest.add(totalMonthly);
        balance = balance.subtract(/* principal of installment */);
    }
    
    // YMO = (totalInterest / principal) * (12 / months) * 100
    return totalInterest
        .divide(principal, 10, RoundingMode.HALF_EVEN)
        .multiply(BigDecimal.valueOf(12))
        .divide(BigDecimal.valueOf(months), 4, RoundingMode.HALF_EVEN)
        .multiply(BigDecimal.valueOf(100));
}
```

### 6. APR vs APY (EAR)

**APR (Annual Percentage Rate):**
- Nominal rate, no compounding
- TR: BKO (Borçlanma Kâr Oranı)

**APY (Annual Percentage Yield) / EAR (Effective Annual Rate):**
- With compounding
- TR: NKO (Net Kâr Oranı) or YMO

**Relationship:**

```
EAR = (1 + APR/n)^n - 1

APR: 12%, monthly compound:
EAR = (1 + 0.12/12)^12 - 1 = 12.683%
```

Banking pratiği: Customer'a hem APR hem EAR göster (transparency regulation).

### 7. FX — bid/ask spread

Bank FX rates:

```
USD/TRY rates:
  Bank buys (bid): 32.40    (bank pays TRY for USD)
  Bank sells (ask): 32.60   (bank receives TRY for USD)
  Mid:             32.50
  Spread:          0.20 TRY (~62 bps)
```

**Customer perspective:**
- Sell 1000 USD → receive 32,400 TL (bank buys at bid)
- Buy 1000 USD → pay 32,600 TL (bank sells at ask)

**Banking nedeniyle:**
- Cover risk (FX move during settlement)
- Operational cost
- Profit margin

**Spread factors:**
- Currency pair (major vs exotic) — TRY/USD daha geniş USD/EUR'dan
- Market volatility (spread widens in stress)
- Transaction size (large block tighter)
- Time of day (Asian close TRY wide)

**Java FX service:**

```java
@Service
public class FxRateService {
    
    private final FxRateRepository repo;
    
    public BigDecimal convertCustomerSellsForeign(
        BigDecimal foreignAmount,
        String foreignCurrency
    ) {
        // Customer sells foreign → bank buys at bid
        FxRate rate = repo.getRate(foreignCurrency, "TRY", FxSide.BID);
        return foreignAmount.multiply(rate.getRate())
            .setScale(2, RoundingMode.HALF_EVEN);
    }
    
    public BigDecimal convertCustomerBuysForeign(
        BigDecimal foreignAmount,
        String foreignCurrency
    ) {
        // Customer buys foreign → bank sells at ask
        FxRate rate = repo.getRate(foreignCurrency, "TRY", FxSide.ASK);
        return foreignAmount.multiply(rate.getRate())
            .setScale(2, RoundingMode.HALF_EVEN);
    }
}
```

### 8. FX forward — future rate

**Spot:** Anlık settlement (T+2 standard).
**Forward:** Future date locked rate.

**Forward rate (covered interest parity):**

```
F = S × (1 + r_TRY × t) / (1 + r_USD × t)

F: forward rate
S: spot rate
r_TRY: TRY interest rate
r_USD: USD interest rate
t: time to forward date (years)
```

**Banking örnek:** Spot USD/TRY 32.50, TRY rate %50, USD rate %5, 3 ay:

```
t = 0.25
F = 32.50 × (1 + 0.50 × 0.25) / (1 + 0.05 × 0.25)
F = 32.50 × 1.125 / 1.0125
F = 32.50 × 1.1111
F = 36.11
```

Forward 36.11 (3 ay sonra USD/TRY). Spot'tan yüksek = TRY discount (high rate).

**Forward use:**
- Corporate USD payment 3 ay sonra → lock rate today
- Hedge FX risk
- Speculative position

### 9. FX swap

**FX Swap:** Spot + opposite forward.

```
Day 1: Bank buys USD spot @ 32.50, sells TRY
       Bank sells USD 3-month forward @ 36.11, receives TRY

Net effect: Bank borrows TRY for 3 months (paying difference as cost)
           Equivalent to TRY money market borrowing
```

Banking için liquidity management, balance sheet optimization.

### 10. Interest accrual — banking ledger

**Daily accrual** banking standard. Faiz **hak edildiği gün** journal'a yazılır.

```java
@Service
public class InterestAccrualService {
    
    private final LedgerService ledgerService;
    
    @Scheduled(cron = "0 30 23 * * *")   // Daily 23:30
    public void accrueDailyInterest() {
        LocalDate today = LocalDate.now(zone);
        
        List<LoanAccount> loans = loanRepo.findActiveLoans();
        
        for (LoanAccount loan : loans) {
            BigDecimal dailyInterest = computeDailyInterest(loan, today);
            
            if (dailyInterest.signum() == 0) continue;
            
            ledgerService.post(JournalEntryRequest.builder()
                .description("Daily interest accrual " + loan.getId())
                .referenceType("interest_accrual")
                .referenceId(UUID.nameUUIDFromBytes(
                    (loan.getId() + today).getBytes()).toString())   // Idempotent
                .entry(LedgerEntryRequest.debit(
                    "loan_receivable_" + loan.getId(), dailyInterest, "TRY"))
                .entry(LedgerEntryRequest.credit(
                    "interest_income_4101", dailyInterest, "TRY"))
                .build());
        }
    }
    
    private BigDecimal computeDailyInterest(LoanAccount loan, LocalDate date) {
        BigDecimal balance = loan.getOutstandingBalance(date);
        BigDecimal annualRate = loan.getAnnualRate();
        DayCountConvention convention = loan.getDayCountConvention();
        
        BigDecimal fraction = convention.fraction(date.minusDays(1), date);  // 1 day
        return balance.multiply(annualRate).multiply(fraction)
            .setScale(2, RoundingMode.HALF_EVEN);
    }
}
```

**Idempotency:** `referenceId = hash(loanId + date)` → same day re-run = no duplicate.

### 11. Banking interest products

**Vadeli mevduat (Time deposit):**
- Customer locks money for term (1, 3, 6, 12 ay)
- Higher rate than vadesiz
- Pre-mature withdrawal: penalty or no interest

**Vadesiz mevduat (Demand deposit):**
- Withdrawal anytime
- Low/no interest
- Banking: cheap funding (most banks)

**Kredi (Loan):**
- Customer borrows, pays interest
- Term, schedule, collateral (mortgage)
- BSMV + KKDF apply

**Kredi kartı (Credit card):**
- Revolving credit
- Minimum payment monthly
- High rates (gecikme faizi BDDK regulated max)

**Konut kredisi (Mortgage):**
- Long term (5-30 yıl)
- Lower rate
- Collateral: gayrimenkul
- KKDF %0 (regulatory)

**Otomobil kredisi (Auto loan):**
- Medium term (3-5 yıl)
- Collateral: araç
- KKDF varies

### 12. Banking — FX/interest anti-pattern'leri

**Anti-pattern 1: double for interest calc**

```java
double interest = principal * rate * days / 365;   // ❌
```

Precision loss. **BigDecimal HALF_EVEN.**

**Anti-pattern 2: Day count convention assumption**

Code "365 hardcoded" — ACT/360 contract için yanlış.

**Anti-pattern 3: BSMV + KKDF ignore TR products**

Customer şikayet, regulatory non-compliance.

**Anti-pattern 4: APR/APY confusion**

Customer "%12 yıllık" duyduğu, gerçek effective %13.5 → güven kaybı + UMÇK şikayet.

**Anti-pattern 5: FX rate spread ignore**

Mid-rate kullan → bank kaybeder. Bid/ask explicit.

**Anti-pattern 6: Daily interest accrual yok**

Customer pre-mature withdraw → manual calculate karmaşık. Daily accrual baseline.

**Anti-pattern 7: Leap year ignore**

Şubat 29 → 365 değil 366 gün. ACT/365 vs ACT/365F farkı.

**Anti-pattern 8: Time zone yanlış**

UTC vs Europe/Istanbul. Daily accrual cutoff TR zaman dilimi.

**Anti-pattern 9: Idempotency yok accrual'da**

Job re-run → double interest. Idempotency key per (loan, date).

**Anti-pattern 10: FX forward stale rate**

Real-time rate yok → market move sonrası eski rate ile işlem. Rate refresh + expiry.

---

## Önemli olabilecek araştırma kaynakları

- ISDA day count conventions
- "Fixed Income Mathematics" — Fabozzi
- TCMB MK (Mevduat Kanunu)
- BSMV / KKDF mevzuat
- TBB Tüketici Kredisi Sözleşmesi rehberi
- BDDK tüketici kredisi tebliği
- TRLIBOR / TLREF reference rates
- "Options, Futures, and Other Derivatives" — Hull

---

## Mini task'ler

### Task 10.5.1 — Simple + compound interest (45 dk)

BigDecimal implementation. Test edge cases (zero rate, zero principal, very long period).

### Task 10.5.2 — Day count conventions enum (60 dk)

ACT/365, ACT/360, ACT/ACT, 30/360. Test her convention için fraction calculation.

### Task 10.5.3 — Loan amortization schedule (60 dk)

French method. 60-month consumer loan. Schedule table. Test: last installment clears balance.

### Task 10.5.4 — BSMV + KKDF effective cost (45 dk)

YMO calculation. TR consumer loan. Customer-facing display.

### Task 10.5.5 — APR ↔ APY conversion (30 dk)

EAR formula. Test: monthly vs daily vs continuous compound.

### Task 10.5.6 — FX bid/ask conversion (45 dk)

Customer buys/sells foreign. Rate table seed. Spread display.

### Task 10.5.7 — FX forward calculator (45 dk)

CIP formula. Spot + 2 rate inputs + tenor → forward.

### Task 10.5.8 — Daily interest accrual job (60 dk)

Spring `@Scheduled` 23:30. Ledger post via Topic 10.1 service. Idempotency. Test rerun.

### Task 10.5.9 — Loan early payoff calculator (45 dk)

Customer wants to close loan early. Outstanding balance + accrued interest to date.

### Task 10.5.10 — TR consumer loan calculator UI/API (60 dk)

Input: amount, term, rate. Output: schedule + BSMV/KKDF + YMO. Customer-facing API.

---

## Test yazma rehberi

```java
@Test
void shouldCalculateSimpleInterest() {
    BigDecimal interest = financeService.simpleInterest(
        new BigDecimal("100000"),
        new BigDecimal("0.05"),
        90,
        365);
    
    assertThat(interest).isEqualByComparingTo("1232.88");
}

@ParameterizedTest
@CsvSource({
    "365, ACT_365, 1232.88",
    "360, ACT_360, 1250.00",
    "365, THIRTY_360, 1250.00"   // 90 days under 30/360 = same as 90/360
})
void shouldUseDayCountConvention(int daysInYear, DayCountConvention convention, String expected) {
    BigDecimal interest = financeService.interest(
        new BigDecimal("100000"),
        new BigDecimal("0.05"),
        LocalDate.of(2024, 1, 1),
        LocalDate.of(2024, 4, 1),    // 91 days
        convention);
    
    // close compare
    assertThat(interest.subtract(new BigDecimal(expected)).abs())
        .isLessThan(new BigDecimal("5"));
}

@Test
void shouldGenerateAmortizationSchedule() {
    List<Installment> schedule = financeService.amortize(
        new BigDecimal("100000"),
        new BigDecimal("0.15"),
        60);
    
    assertThat(schedule).hasSize(60);
    
    Installment first = schedule.get(0);
    assertThat(first.getInstallment()).isEqualByComparingTo("2378.99");
    assertThat(first.getInterest()).isEqualByComparingTo("1250.00");
    
    Installment last = schedule.get(59);
    assertThat(last.getEndBalance()).isEqualByComparingTo("0.00");
    
    BigDecimal totalPaid = schedule.stream()
        .map(Installment::getInstallment)
        .reduce(ZERO, BigDecimal::add);
    assertThat(totalPaid).isGreaterThan(new BigDecimal("100000"));
}

@Test
void shouldComputeEffectiveCostWithBsmvKkdf() {
    BigDecimal ymo = financeService.effectiveAnnualCost(
        principal: new BigDecimal("100000"),
        monthlyRate: new BigDecimal("0.025"),
        months: 60,
        bsmvRate: new BigDecimal("0.05"),
        kkdfRate: new BigDecimal("0.10"));
    
    assertThat(ymo).isGreaterThan(new BigDecimal("30"));   // %30+ effective
}

@Test
void shouldUseAskRateForCustomerBuyingForeign() {
    fxRateRepo.save(new FxRate("USD", "TRY", new BigDecimal("32.40"), new BigDecimal("32.60")));
    
    BigDecimal tryToPay = fxService.convertCustomerBuysForeign(
        new BigDecimal("1000"), "USD");
    
    assertThat(tryToPay).isEqualByComparingTo("32600.00");
}

@Test
void dailyInterestAccrualShouldBeIdempotent() {
    LoanAccount loan = createLoan(new BigDecimal("100000"), new BigDecimal("0.15"));
    
    interestService.accrueDailyInterest();
    interestService.accrueDailyInterest();   // Re-run
    
    BigDecimal accrued = balanceService.balanceOf(loan.getInterestAccount(), "TRY");
    BigDecimal expectedOneDay = new BigDecimal("100000")
        .multiply(new BigDecimal("0.15"))
        .divide(new BigDecimal("365"), 2, RoundingMode.HALF_EVEN);
    
    assertThat(accrued).isEqualByComparingTo(expectedOneDay);   // Exactly one day, not two
}
```

---

## Claude-verify prompt

```
Finance calculations implementation'ımı banking-grade kriterlere göre değerlendir:

1. Money handling:
   - BigDecimal everywhere (no double)?
   - HALF_EVEN rounding?
   - MathContext explicit?

2. Interest:
   - Simple + compound implementation?
   - Day count conventions enum (ACT/365, ACT/360, ACT/ACT, 30/360)?
   - Convention switchable per product?

3. Amortization:
   - French equal payment?
   - Eşit anapara variant?
   - Last installment adjustment (clear residual)?
   - Schedule table generation?

4. TR specifics:
   - BSMV calculation (interest income)?
   - KKDF calculation (consumer loan)?
   - YMO (effective annual cost) display?
   - TBB consumer regulation compliance?

5. APR vs APY:
   - EAR formula?
   - Customer display both?

6. FX:
   - Bid/ask spread explicit?
   - Customer buys → ask?
   - Customer sells → bid?
   - Rate refresh strategy?
   - Forward CIP calculation?

7. Accrual:
   - Daily accrual scheduled job?
   - Ledger posting (Topic 10.1)?
   - Idempotency per (loan, date)?
   - Time zone (Europe/Istanbul)?

8. Edge cases:
   - Leap year handling?
   - Pre-mature payoff calculation?
   - Floating rate (TLREF) update?
   - Holiday day skip vs accrue?

9. Compliance:
   - YMO customer display?
   - BDDK consumer loan tebliği?
   - TBB sözleşme template?

10. Anti-pattern:
    - double money YOK?
    - Hardcoded 365 days YOK?
    - BSMV/KKDF skip YOK?
    - APR/APY confusion YOK?
    - Mid-rate FX YOK?
    - Time zone error YOK?
    - Idempotency yok accrual YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Simple + compound interest BigDecimal
- [ ] Day count conventions (4 enum)
- [ ] Amortization French + eşit anapara
- [ ] BSMV + KKDF + YMO calculation
- [ ] APR ↔ APY conversion
- [ ] FX bid/ask service
- [ ] FX forward CIP
- [ ] Daily interest accrual job + ledger + idempotency
- [ ] Pre-mature payoff calculator
- [ ] TR consumer loan calculator API
- [ ] 10+ integration test

---

## Defter notları (10 madde)

1. "Simple vs compound interest banking products (vadeli vs kredi): ____."
2. "Day count conventions (ACT/365, ACT/360, ACT/ACT, 30/360) product map: ____."
3. "French amortization formula + interest portion decreasing pattern: ____."
4. "BSMV + KKDF TR vergi banking customer cost impact: ____."
5. "APR (BKO) vs APY/EAR (YMO/NKO) banking regulation transparency: ____."
6. "FX bid/ask spread + customer buy → ask + sell → bid: ____."
7. "FX forward CIP formula + TRY high rate scenario: ____."
8. "Daily interest accrual + ledger (Topic 10.1) + idempotency: ____."
9. "Time zone Europe/Istanbul + leap year + holiday accrual karar: ____."
10. "TBB + BDDK consumer regulation YMO + sözleşme uyumu: ____."
