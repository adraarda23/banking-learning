# Topic 10.1 — Double-Entry Accounting (Banking Deep)

## Hedef

Phase 1'deki temel double-entry'i banking-grade derinlikte öğrenmek: T-account, debit/credit kuralları, Chart of Accounts (COA), journal vs ledger, trial balance, accrual vs cash basis, Stripe ledger pattern, immutable ledger design, banking event sourcing pattern, multi-currency accounting, intermediate accounts.

## Süre

Okuma: 2.5 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Phase 1 temel accounting bitti
- BigDecimal money handling
- Phase 3 JPA + transactional integrity

---

## Kavramlar

### 1. Double-entry — niye banking için ZORUNLU?

Single-entry (saymaca defter):
```
+100 (gelir)
-50 (gider)
=50 (bakiye)
```

Hata bulunamaz. Audit imkansız. Banking için **ASLA**.

Double-entry:
```
Her transaction = 2 entry
Sum(debits) == Sum(credits)   (her zaman)

Bir hesap arttı = başka bir hesap azaldı / arttı (denge)
```

**Banking örnek — transfer:**
```
Customer A → Customer B 100 TL

Entry 1: Debit  customer_A_account  100 TL    (A azaldı)
Entry 2: Credit customer_B_account  100 TL    (B arttı)

Sum debits = 100, Sum credits = 100. Denge.
```

### 2. T-account temelleri

```
       Account: Customer A
      ──────────────────────
       Debit  │  Credit
       100    │
       50     │
              │  200
       ─────────────────
       Total: 150 D, 200 C
       Balance: 50 C
```

Banking görsel: T-account. Sol Debit, sağ Credit.

### 3. Debit / Credit kuralları — banking semantic

Banking için **bank perspective'inden** bak:

| Hesap tipi | Debit | Credit |
|---|---|---|
| **Asset** (varlık) — Banka için müşteri borcu | ↑ | ↓ |
| **Liability** (borç) — Banka için müşteri parası | ↓ | ↑ |
| **Equity** (özkaynak) | ↓ | ↑ |
| **Revenue** (gelir) | ↓ | ↑ |
| **Expense** (gider) | ↑ | ↓ |

**Banking confusion alert:**

```
Customer ATM'de 100 TL yatırır.

Banka kitabında:
  Vault cash (asset)        +100  Debit
  Customer deposit (liab.)  +100  Credit

Niye customer deposit "liability"?
→ Customer parayı çekme hakkına sahip. Banka borçlu = banka için liability.
```

Müşteri perspektifinden: "Param arttı". Banka perspektifinden: "Borcum (sana) arttı, ama kasamda nakit arttı".

### 4. Chart of Accounts (COA) — banking

```
1. ASSETS                    (Banka varlıkları)
   1100 Cash & Equivalents
     1101 Vault Cash
     1102 Central Bank Reserve
     1103 Nostro Accounts        (banka'nın başka bankada hesabı)
   1200 Loans Receivable
     1201 Consumer Loans
     1202 Mortgage Loans
     1203 Corporate Loans
   1300 Securities
   1400 Premises & Equipment

2. LIABILITIES               (Banka borçları)
   2100 Customer Deposits
     2101 Demand Deposits        (vadesiz)
     2102 Savings Deposits
     2103 Time Deposits          (vadeli)
     2104 Foreign Currency Deposits
   2200 Vostro Accounts          (başka banka'nın bizde hesabı)
   2300 Borrowings (interbank)
   2400 Subordinated Debt

3. EQUITY
   3100 Share Capital
   3200 Retained Earnings
   3300 Reserves

4. REVENUE
   4100 Interest Income
     4101 Loan Interest
     4102 Investment Interest
   4200 Fee Income
     4201 Transfer Fees
     4202 Card Fees
   4300 FX Gains

5. EXPENSE
   5100 Interest Expense
     5101 Deposit Interest
     5102 Borrowing Interest
   5200 Operating Expense
     5201 Salaries
     5202 IT Infrastructure
   5300 Provisions for Loan Loss
   5400 FX Losses

6. INTERMEDIATE / CLEARING
   6100 EFT Clearing
   6200 FAST Clearing
   6300 Card Network Clearing
   6400 SWIFT Settlement Pending
```

Her account = unique number (1101, 4201, vb).

### 5. Journal vs Ledger

**Journal** — chronological transaction list.
**Ledger** — per-account aggregation.

```
Journal entry #12345 (2024-05-12 10:30:45):
  Description: Transfer A → B
  Debit:  Customer A Account (2101)    100.00 TL
  Credit: Customer B Account (2101)    100.00 TL

Ledger view Customer A:
  Date         | Description | Debit | Credit | Balance
  2024-05-12  | Transfer    | 100   |        | (X-100)
```

Banking DB tasarımı:

```sql
CREATE TABLE journal_entry (
    id BIGSERIAL PRIMARY KEY,
    posted_at TIMESTAMPTZ NOT NULL,
    description TEXT,
    reference_type VARCHAR(50),    -- 'transfer', 'fee', 'interest', ...
    reference_id UUID,
    posted_by UUID,                 -- user or system
    reversal_of BIGINT REFERENCES journal_entry(id),
    immutable BOOLEAN DEFAULT TRUE  -- audit safeguard
);

CREATE TABLE ledger_entry (
    id BIGSERIAL PRIMARY KEY,
    journal_entry_id BIGINT REFERENCES journal_entry(id) NOT NULL,
    account_id BIGINT REFERENCES account(id) NOT NULL,
    debit_amount NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit_amount NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL,
    posted_at TIMESTAMPTZ NOT NULL,
    CHECK (debit_amount >= 0 AND credit_amount >= 0),
    CHECK (debit_amount = 0 OR credit_amount = 0)   -- one or other, not both
);

CREATE INDEX idx_ledger_account_date ON ledger_entry(account_id, posted_at DESC);
CREATE INDEX idx_ledger_journal ON ledger_entry(journal_entry_id);
```

**Critical: journal balanced check.**

```sql
-- Trigger or app-level validation
SELECT 
    journal_entry_id,
    SUM(debit_amount) AS total_debit,
    SUM(credit_amount) AS total_credit
FROM ledger_entry
GROUP BY journal_entry_id
HAVING SUM(debit_amount) != SUM(credit_amount);
-- Returns rows = INVARIANT BROKEN
```

### 6. Stripe ledger pattern — immutable + idempotent

Stripe banking ledger reference (open public design):

**Principles:**
1. **Immutable journal entries** — never UPDATE, never DELETE
2. **Reversal via offset** — wrong entry → ekstra reverse entry yaz
3. **Idempotency** — same `(reference_type, reference_id)` aynı sonuç
4. **Atomic posting** — multiple ledger entries one transaction (DB ACID)
5. **Balance derived** — ledger_entry sum'dan
6. **Currency segregation** — different currency = different account

Reverse pattern:

```
Entry #12345 (wrong):
  Debit  A 100
  Credit B 100

Reverse:
Entry #12346 (reversal_of = 12345):
  Debit  B 100      (B'den geri al)
  Credit A 100      (A'ya geri ver)

Sonuç: Effective balance original'dan önceki halinde
```

### 7. Balance computation

Naive: ledger_entry'lerin sum'ı.

```sql
SELECT 
    SUM(credit_amount - debit_amount) AS balance
FROM ledger_entry
WHERE account_id = $1
  AND posted_at <= $2;
```

**Performance:** Milyonlarca entry için her balance query expensive.

**Materialized view (Topic 4):**

```sql
CREATE MATERIALIZED VIEW account_balance AS
SELECT 
    account_id,
    currency,
    SUM(credit_amount - debit_amount) AS balance,
    MAX(posted_at) AS last_posted_at
FROM ledger_entry
GROUP BY account_id, currency;

-- Refresh nightly veya event-driven
REFRESH MATERIALIZED VIEW CONCURRENTLY account_balance;
```

**Better: balance snapshot pattern:**

```sql
CREATE TABLE account_balance_snapshot (
    account_id BIGINT,
    as_of_date DATE,
    balance NUMERIC(19,4),
    PRIMARY KEY (account_id, as_of_date)
);

-- Nightly: snapshot her account
-- Online query: latest snapshot + last day's deltas
```

Banking için snapshot **EOD** sonrası. Online query latency optimization.

### 8. Banking transaction examples — full posting

#### Example 1: Customer deposit

```
Customer A deposits 1000 TL cash at branch:

Journal entry #1:
  Description: ATM/Branch deposit
  Debit:  Vault Cash (1101)              1000.00 TL
  Credit: Customer A Deposit (2101-A)    1000.00 TL
```

#### Example 2: Inter-customer transfer (same bank)

```
Customer A → Customer B 100 TL:

Journal entry:
  Debit:  Customer A Deposit (2101-A)   100.00 TL
  Credit: Customer B Deposit (2101-B)   100.00 TL
```

Bank's total liability unchanged.

#### Example 3: Inter-bank transfer (EFT) — outgoing

```
Customer A → Customer X at another bank, 100 TL:

Initiation (T+0):
  Debit:  Customer A Deposit (2101-A)        100.00 TL
  Credit: EFT Outgoing Clearing (6100)       100.00 TL

Settlement with CBRT (T+1):
  Debit:  EFT Outgoing Clearing (6100)       100.00 TL
  Credit: Central Bank Reserve (1102)        100.00 TL
```

Intermediate clearing account temporarily holds liability until settled.

#### Example 4: Interest accrual

```
Loan account, 5% annual, monthly accrual:

Loan balance: 100,000 TL
Monthly interest: 100,000 * 5% / 12 = 416.67 TL

Journal entry (last day of month):
  Debit:  Customer A Loan Receivable (1201-A)    416.67 TL
  Credit: Interest Income (4101)                 416.67 TL
```

Note: This is **accrual** — money not collected yet, but earned.

#### Example 5: FX transaction

```
Customer A buys 100 USD at 32.50 TL/USD:

Customer pays: 3250.00 TL
Customer receives: 100.00 USD

Journal entry:
  Debit:  Customer A TRY Deposit (2101-A-TRY)    3250.00 TL
  Credit: Bank FX Position TRY (1104-TRY)        3250.00 TL
  
  Debit:  Bank FX Position USD (1104-USD)         100.00 USD
  Credit: Customer A USD Deposit (2104-A-USD)     100.00 USD
```

İki entry — TRY ve USD. Currency segregation.

FX gain/loss bank'ın internal position'ında recognized.

#### Example 6: Fee charged

```
Transfer fee 5 TL:

Journal entry (along with transfer):
  Debit:  Customer A Deposit                       5.00 TL
  Credit: Transfer Fee Income (4201)               5.00 TL
```

### 9. Atomic posting — DB transaction

```java
@Service
public class LedgerService {
    
    private final EntityManager em;
    private final JournalEntryRepository journalRepo;
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public JournalEntry post(JournalEntryRequest req) {
        validateBalance(req);   // Sum debits == Sum credits
        validateIdempotency(req);
        
        JournalEntry journal = JournalEntry.builder()
            .postedAt(Instant.now())
            .description(req.description())
            .referenceType(req.referenceType())
            .referenceId(req.referenceId())
            .postedBy(currentUserId())
            .build();
        journalRepo.save(journal);
        
        for (LedgerEntryRequest entry : req.entries()) {
            LedgerEntry ledger = LedgerEntry.builder()
                .journalEntry(journal)
                .accountId(entry.accountId())
                .debitAmount(entry.debitAmount() != null ? entry.debitAmount() : ZERO)
                .creditAmount(entry.creditAmount() != null ? entry.creditAmount() : ZERO)
                .currency(entry.currency())
                .postedAt(journal.getPostedAt())
                .build();
            em.persist(ledger);
        }
        
        em.flush();
        return journal;
    }
    
    private void validateBalance(JournalEntryRequest req) {
        Map<String, BigDecimal> debitsByCcy = req.entries().stream()
            .filter(e -> e.debitAmount() != null)
            .collect(groupingBy(LedgerEntryRequest::currency,
                reducing(ZERO, LedgerEntryRequest::debitAmount, BigDecimal::add)));
        
        Map<String, BigDecimal> creditsByCcy = req.entries().stream()
            .filter(e -> e.creditAmount() != null)
            .collect(groupingBy(LedgerEntryRequest::currency,
                reducing(ZERO, LedgerEntryRequest::creditAmount, BigDecimal::add)));
        
        if (!debitsByCcy.equals(creditsByCcy)) {
            throw new LedgerInvariantException(
                "Journal not balanced. Debits: " + debitsByCcy + ", Credits: " + creditsByCcy);
        }
    }
}
```

### 10. Trial balance

Tüm account'ların debit total + credit total. **Eşit olmalı** (her zaman).

```sql
SELECT 
    'TOTAL' AS account,
    SUM(debit_amount) AS total_debit,
    SUM(credit_amount) AS total_credit,
    SUM(debit_amount) - SUM(credit_amount) AS difference
FROM ledger_entry;
-- difference MUST be 0
```

Banking EOD job → trial balance run → reconciliation alert if != 0.

### 11. Accrual vs Cash basis

**Cash basis:** Money in/out görünce kaydet.
**Accrual basis:** Earned/incurred görünce kaydet (gerçek time'da değil, hak edildiği zamanda).

Banking **accrual** kullanır:
- Faiz hesabı her gün accrue olur (collected aylık)
- Komisyon gelir kazanıldığında recognize (charge anında)
- Provisioning loan loss (potansiyel kayıp, gerçek değil)

Türk muhasebe (TMS/IFRS) accrual-based. BDDK requirement.

### 12. Event sourcing pattern — banking ledger

```sql
CREATE TABLE ledger_event (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID,
    event_type VARCHAR(50),
    payload JSONB,
    occurred_at TIMESTAMPTZ,
    sequence_no BIGINT
);
```

Event types:
- `AccountOpened`
- `DepositMade`
- `WithdrawalMade`
- `TransferInitiated`
- `TransferCompleted`
- `InterestAccrued`
- `FeeCharged`

Balance computed from event replay:
```java
BigDecimal balance = events.stream()
    .map(e -> switch (e.type()) {
        case "DepositMade" -> e.amount();
        case "WithdrawalMade" -> e.amount().negate();
        ...
    })
    .reduce(ZERO, BigDecimal::add);
```

**Trade-off:**
- Audit complete (all history)
- Replay for repair / what-if analysis
- Storage growth (years of events)
- Need snapshots for performance

Banking pratiği: Mixed — current balance materialized, events stored for audit.

### 13. Banking immutable design

```sql
-- Prevent UPDATE/DELETE at DB level
CREATE OR REPLACE FUNCTION prevent_ledger_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'ledger_entry is immutable. Use reversal pattern.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ledger_no_update
BEFORE UPDATE OR DELETE ON ledger_entry
FOR EACH ROW EXECUTE FUNCTION prevent_ledger_modification();
```

**App level:**
```java
@Entity
@Immutable   // Hibernate hint
public class LedgerEntry { ... }
```

### 14. Banking — accounting anti-pattern'leri

**Anti-pattern 1: Direct balance UPDATE**

```sql
UPDATE account SET balance = balance + 100 WHERE id = ?;   -- ❌
```

No audit, no double-entry. Banking için **ASLA**. Always journal + ledger.

**Anti-pattern 2: Single-entry**

`transaction(account, amount, type)` table. Audit impossible.

**Anti-pattern 3: Mutable ledger**

UPDATE on ledger_entry. Audit broken. Reverse pattern kullan.

**Anti-pattern 4: Negative number tricks**

Debit = positive, Credit = negative. Confusing. Banking conventional: ayrı kolonlar.

**Anti-pattern 5: Cross-currency balance**

```
USD entry + TRY entry → balance summed in TRY?
```

Currency segregation. Different currency = different account or separate.

**Anti-pattern 6: Floating point money**

`double balance`. Precision loss. **BigDecimal HALF_EVEN** (Phase 1).

**Anti-pattern 7: Journal not balanced check skipped**

App veya DB level invariant. Banking için **mandatory**.

**Anti-pattern 8: Async ledger posting**

Async = eventually consistent → balance race. Banking critical path → synchronous + transaction.

**Anti-pattern 9: No intermediate clearing accounts**

EFT outgoing → direct customer → CBRT. T+1 settle? Intermediate clearing account holds.

**Anti-pattern 10: Soft delete on ledger**

`deleted=true` flag. Audit broken. Hard immutable + reversal.

---

## Önemli olabilecek araştırma kaynakları

- "Accounting for Computer Scientists" — Martin Kleppmann blog
- Stripe ledger engineering blog
- Square / Block ledger design docs
- IFRS / TFRS standards
- BDDK muhasebe rehberi
- "Double-Entry Bookkeeping" — Pacioli (1494, ilk kaynak)

---

## Mini task'ler

### Task 10.1.1 — Chart of Accounts table (45 dk)

`account` table + hierarchical (parent_account_id). Seed banking COA (1101-5400, +6100-6400).

### Task 10.1.2 — journal_entry + ledger_entry tables (45 dk)

DDL schema. Constraints (debit_amount + credit_amount only one). Index account_id + posted_at.

### Task 10.1.3 — Immutable trigger (15 dk)

PL/pgSQL trigger BEFORE UPDATE/DELETE → exception.

### Task 10.1.4 — LedgerService atomic posting (60 dk)

Service: validate balanced, idempotency, atomic insert. Test: balanced journal accept, unbalanced reject.

### Task 10.1.5 — Banking transactions implement (90 dk)

6 example yukarıdan: deposit, intra-customer transfer, inter-bank transfer (clearing), interest accrual, FX, fee. Her biri için service method + test.

### Task 10.1.6 — Reverse pattern (30 dk)

`reverseEntry(journalId)` — yeni entry yaz with opposite debit/credit. `reversal_of` link.

### Task 10.1.7 — Trial balance SQL (30 dk)

Query: sum all debit, sum all credit, difference. EOD job için scheduled.

### Task 10.1.8 — Balance computation 3 yöntem (60 dk)

1. Naive sum query
2. Materialized view
3. Snapshot + delta

Performance compare (1M entry).

### Task 10.1.9 — Multi-currency account (45 dk)

Currency segregation. USD + TRY entries same customer = separate account or separate balance per currency.

### Task 10.1.10 — Event sourcing variant (60 dk)

`ledger_event` table. Event replay → balance. Snapshot her 1000 event.

---

## Test yazma rehberi

```java
@Test
@Transactional
void shouldEnforceBalancedJournalEntry() {
    JournalEntryRequest unbalanced = JournalEntryRequest.builder()
        .description("test")
        .entry(of("CUSTOMER_A", debit(100), "TRY"))
        .entry(of("CUSTOMER_B", credit(50), "TRY"))   // mismatch
        .build();
    
    assertThatThrownBy(() -> ledgerService.post(unbalanced))
        .isInstanceOf(LedgerInvariantException.class)
        .hasMessageContaining("not balanced");
}

@Test
@Transactional
void shouldPostDeposit() {
    JournalEntry entry = ledgerService.post(depositRequest(customerA, "1000", "TRY"));
    
    BigDecimal vaultBalance = balanceService.balanceOf(VAULT_CASH, "TRY");
    BigDecimal customerBalance = balanceService.balanceOf(customerA, "TRY");
    
    assertThat(vaultBalance).isEqualByComparingTo("1000.00");
    assertThat(customerBalance).isEqualByComparingTo("1000.00");
}

@Test
@Transactional
void shouldHandleTransferAcrossCustomers() {
    setupBalance(customerA, "TRY", "1000.00");
    setupBalance(customerB, "TRY", "500.00");
    
    ledgerService.post(transferRequest(customerA, customerB, "100", "TRY"));
    
    assertThat(balanceService.balanceOf(customerA, "TRY")).isEqualByComparingTo("900.00");
    assertThat(balanceService.balanceOf(customerB, "TRY")).isEqualByComparingTo("600.00");
}

@Test
@Transactional
void shouldUseClearingAccountForInterbankTransfer() {
    setupBalance(customerA, "TRY", "1000.00");
    
    JournalEntry initiate = ledgerService.post(
        eftOutgoingInitiateRequest(customerA, "100", recipientIban));
    
    assertThat(balanceService.balanceOf(customerA, "TRY")).isEqualByComparingTo("900.00");
    assertThat(balanceService.balanceOf(EFT_CLEARING, "TRY")).isEqualByComparingTo("100.00");
    
    JournalEntry settle = ledgerService.post(
        eftOutgoingSettleRequest(initiate.getId()));
    
    assertThat(balanceService.balanceOf(EFT_CLEARING, "TRY")).isEqualByComparingTo("0.00");
    assertThat(balanceService.balanceOf(CENTRAL_BANK_RESERVE, "TRY")).isEqualByComparingTo("-100.00");
}

@Test
@Transactional
void shouldReverseEntry() {
    JournalEntry original = ledgerService.post(transferRequest(...));
    
    JournalEntry reversal = ledgerService.reverse(original.getId(), "Customer dispute");
    
    assertThat(reversal.getReversalOf()).isEqualTo(original.getId());
    assertThat(balanceService.balanceOf(customerA, "TRY")).isEqualByComparingTo("1000.00");   // Restored
}

@Test
void trialBalanceShouldBeZero() {
    // Many random transactions
    for (int i = 0; i < 1000; i++) {
        randomTransaction();
    }
    
    BigDecimal diff = trialBalanceService.calculate();
    assertThat(diff).isEqualByComparingTo("0.00");
}

@Test
void shouldRejectModificationOfLedgerEntry() {
    JournalEntry entry = ledgerService.post(...);
    LedgerEntry ledger = entry.getLedgerEntries().get(0);
    
    assertThatThrownBy(() -> {
        em.createNativeQuery("UPDATE ledger_entry SET debit_amount = 200 WHERE id = " + ledger.getId())
            .executeUpdate();
    }).hasMessageContaining("immutable");
}
```

---

## Claude-verify prompt

```
Double-entry accounting implementation'ımı banking-grade kriterlere göre değerlendir:

1. Chart of Accounts:
   - Hierarchical (asset/liability/equity/revenue/expense/intermediate)?
   - Banking-realistic numbering?
   - Intermediate clearing accounts (EFT, FAST, SWIFT)?

2. Schema:
   - journal_entry + ledger_entry separate tables?
   - Debit/credit separate columns?
   - currency column?
   - immutable trigger?
   - reference_type + reference_id?
   - reversal_of FK?

3. Posting:
   - Atomic transaction (DB ACID)?
   - Balance check (sum debits == sum credits per currency)?
   - Idempotency (reference_type + reference_id unique)?
   - SERIALIZABLE or repeatable read?

4. Banking transactions:
   - Deposit/withdrawal?
   - Intra-customer transfer?
   - Inter-bank transfer with clearing account?
   - Interest accrual?
   - FX with currency segregation?
   - Fee charged?

5. Balance:
   - Computed from ledger sum (no direct balance UPDATE)?
   - Materialized view or snapshot for performance?
   - Per-currency balance?

6. Reversal:
   - Reverse entry pattern (offset, not delete)?
   - Reversal_of FK tracked?
   - Original immutable?

7. Audit:
   - posted_by tracked?
   - posted_at timestamp?
   - description meaningful?
   - reference linked to source?

8. Trial balance:
   - EOD job?
   - Sum debit == sum credit (across all accounts) check?
   - Alert if mismatch?

9. Multi-currency:
   - Each currency separate account or balance per currency?
   - FX gain/loss recognized?

10. Anti-pattern:
    - Direct balance UPDATE YOK?
    - Single-entry YOK?
    - Floating point money YOK?
    - Soft delete on ledger YOK?
    - Async ledger posting YOK?
    - Cross-currency sum YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] COA table + seed (asset/liability/equity/revenue/expense/intermediate)
- [ ] journal_entry + ledger_entry schema
- [ ] Immutable trigger
- [ ] LedgerService atomic posting + balance validation
- [ ] 6 banking transaction implement (deposit, intra-customer, EFT, interest, FX, fee)
- [ ] Reverse pattern
- [ ] Trial balance SQL
- [ ] 3 balance computation method + compare
- [ ] Multi-currency support
- [ ] Event sourcing variant
- [ ] 10+ integration test

---

## Defter notları (10 madde)

1. "Double-entry vs single-entry banking için neden zorunlu (audit, trial balance): ____."
2. "Debit/credit kuralları bank perspective: asset ↑ debit, liability ↑ credit: ____."
3. "Customer deposit niye liability (bank borçludur): ____."
4. "Chart of Accounts hierarchy + intermediate clearing accounts (EFT, FAST): ____."
5. "Journal vs Ledger + atomic posting transaction: ____."
6. "Stripe immutable + reversal pattern banking adoption: ____."
7. "Balance compute (sum query / materialized view / snapshot) trade-off: ____."
8. "Multi-currency segregation FX banking pratik: ____."
9. "Accrual vs cash basis banking IFRS/TFRS uyumu: ____."
10. "Anti-pattern (direct balance UPDATE, float money, cross-currency sum): ____."
