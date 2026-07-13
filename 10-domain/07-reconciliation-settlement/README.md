# Topic 10.7 — Reconciliation & Settlement

## Hedef

Banking reconciliation derinlikte: internal ledger vs external statement match, settlement vs clearing, Nostro/Vostro hesapları, EOD reconciliation job pattern, exception management, break investigation, multi-source reconciliation (Card network, EFT/FAST, SWIFT, internal), partial match algoritmaları, age-based break, escalation, Phase 5 (Batch) ile bağlantı.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1 (Double-entry) bitti
- Topic 10.2-10.4 (ISO 8583, ISO 20022, TR payments) bitti
- Phase 5 (Spring Batch) bitti

---

## Kavramlar

### 1. Clearing vs Settlement — kritik ayrım

**Clearing:** Transaction calculate net positions (who owes whom).
**Settlement:** Actual fund movement (debit/credit reserve accounts).

```
T+0 (Trade date):
  Bank A → Bank B 1000 transactions, total +5M TL
  Bank B → Bank A 800 transactions, total +3M TL
  
  Clearing (net): Bank A owes Bank B 2M TL net

T+1 (Settlement date):
  CBRT: Debit Bank A reserve 2M, Credit Bank B reserve 2M
  Settlement complete.
```

**Settlement models:**
- **DNS (Deferred Net Settlement):** Batch + multilateral netting, T+1 (EFT)
- **RTGS (Real Time Gross Settlement):** Tek tek, anında (FAST)
- **DvP (Delivery vs Payment):** Securities atomic swap

### 2. Nostro / Vostro / Loro

**Nostro** ("our" account at another bank):
```
Maven Bank has account at JPMorgan New York (USD nostro)
Maven Bank's books: "1103 Nostro - JPMorgan USD" — asset
```

**Vostro** ("your" account at our bank):
```
JPMorgan has account at Maven Bank (TRY vostro)
Maven Bank's books: "2200 Vostro - JPMorgan TRY" — liability
```

**Loro** (third-party reference): "Their account at another bank" (rare).

**Banking flow:**
```
Customer X (Maven Bank) sends 1000 USD to Customer Y (Citi London):

Maven Bank's perspective:
  Debit  Customer X (USD)              1000
  Credit Nostro - JPMorgan (USD)       1000   (use Maven's USD position)

JPMorgan executes via SWIFT to Citi:
  Maven's Nostro at JPM: -1000
  Citi's Nostro at JPM: +1000

Citi London:
  Debit  Nostro - JPM                  1000
  Credit Customer Y                    1000
```

End-to-end: Multiple bank ledgers + correspondent network + SWIFT.

### 3. Reconciliation — what + why

**Goal:** Internal ledger vs external statement match. Discrepancy = "break".

**Sources:**
- TCMB EFT statement (daily)
- TCMB FAST settlement
- BKM card network NDC (Net Detail Concentrator) daily file
- SWIFT MT940 / camt.053 nostro statement
- Visa/Mastercard settlement file
- Internal subsystem (separate microservice ledger sync)

**Reconciliation types:**
- **Cash recon:** Bank account balance vs internal cash ledger
- **Card recon:** Card network expected vs internal posted
- **Securities recon:** Trade vs custodian
- **Inter-company:** Subsidiary balances
- **Sub-ledger to GL:** Customer accounts vs general ledger

### 4. EOD reconciliation job — Spring Batch (Phase 5)

```java
@Configuration
@EnableBatchProcessing
public class CardReconciliationJobConfig {
    
    @Bean
    public Job cardReconciliationJob(JobRepository jobRepo, 
                                     Step loadStatementStep,
                                     Step matchStep,
                                     Step exceptionStep) {
        return new JobBuilder("cardReconciliation", jobRepo)
            .start(loadStatementStep)
            .next(matchStep)
            .next(exceptionStep)
            .build();
    }
    
    @Bean
    public Step loadStatementStep(JobRepository repo, PlatformTransactionManager tm) {
        return new StepBuilder("loadStatement", repo)
            .<NetworkStatementRecord, ExternalLedgerEntry>chunk(500, tm)
            .reader(networkStatementReader())
            .processor(statementToLedgerProcessor())
            .writer(externalLedgerWriter())
            .build();
    }
    
    @Bean
    @StepScope
    public FlatFileItemReader<NetworkStatementRecord> networkStatementReader(
        @Value("#{jobParameters['statementFile']}") String filePath
    ) {
        return new FlatFileItemReaderBuilder<NetworkStatementRecord>()
            .name("networkStatementReader")
            .resource(new FileSystemResource(filePath))
            .delimited()
            .delimiter("|")
            .names("rrn", "transmissionDateTime", "amount", "merchantId", 
                   "responseCode", "transactionType", "authId")
            .targetType(NetworkStatementRecord.class)
            .build();
    }
    
    @Bean
    public Step matchStep(JobRepository repo, PlatformTransactionManager tm) {
        return new StepBuilder("match", repo)
            .<InternalEntry, MatchResult>chunk(500, tm)
            .reader(internalEntryReader())
            .processor(matchingProcessor())
            .writer(matchResultWriter())
            .build();
    }
}
```

### 5. Matching algorithms

#### Exact match (RRN based)
```sql
SELECT i.id AS internal_id, e.id AS external_id
FROM internal_card_authorization i
INNER JOIN external_network_statement e
  ON i.rrn = e.rrn
  AND i.statement_date = e.statement_date;
```

#### Fuzzy match (tolerance)
Network sometimes truncates, banks sometimes round:
```sql
SELECT i.id, e.id
FROM internal_card_authorization i
INNER JOIN external_network_statement e
  ON i.merchant_id = e.merchant_id
  AND i.amount BETWEEN e.amount - 0.01 AND e.amount + 0.01    -- ±1 kuruş tolerance
  AND ABS(EXTRACT(EPOCH FROM (i.posted_at - e.posted_at))) < 60   -- ±1 min
  AND NOT EXISTS (SELECT 1 FROM matched WHERE matched.internal_id = i.id);
```

#### Multi-pass match
1. Pass 1: Exact RRN
2. Pass 2: Authorization ID match
3. Pass 3: Amount + merchant + time fuzzy
4. Remaining: exception/break

### 6. Break management

```sql
CREATE TABLE reconciliation_break (
    id BIGSERIAL PRIMARY KEY,
    recon_run_id UUID NOT NULL,
    recon_type VARCHAR(50),     -- 'card_network', 'eft', 'swift', ...
    break_type VARCHAR(50),     -- 'missing_internal', 'missing_external', 'amount_mismatch', 'date_mismatch'
    severity VARCHAR(20),       -- 'low', 'medium', 'high', 'critical'
    
    internal_ref VARCHAR(100),
    external_ref VARCHAR(100),
    
    internal_amount NUMERIC(19,4),
    external_amount NUMERIC(19,4),
    amount_difference NUMERIC(19,4),
    
    detected_at TIMESTAMPTZ DEFAULT now(),
    aged_days INT,
    
    status VARCHAR(20),         -- 'open', 'investigating', 'resolved', 'written_off'
    assigned_to VARCHAR(100),
    resolution_at TIMESTAMPTZ,
    resolution_journal_id BIGINT REFERENCES journal_entry(id),
    
    details JSONB
);

CREATE INDEX idx_break_status_aged ON reconciliation_break(status, aged_days DESC);
CREATE INDEX idx_break_type ON reconciliation_break(recon_type, break_type);
```

**Break types:**
- `missing_internal` — External has, internal lost (unfixed credit?)
- `missing_external` — Internal has, external silence (network reject?)
- `amount_mismatch` — Match by ref but amount differs (FX, fee)
- `date_mismatch` — Posted at different dates (cross-cutoff)
- `duplicate_internal` — Internal posted twice
- `duplicate_external` — Network sent twice (acquirer resend)

### 7. Aging + escalation

```sql
-- Daily update aged_days
UPDATE reconciliation_break
SET aged_days = aged_days + 1
WHERE status IN ('open', 'investigating');

-- Aged > 5 days → high severity
UPDATE reconciliation_break
SET severity = 'high'
WHERE status = 'open' AND aged_days >= 5;

-- Aged > 10 days → critical + escalate
UPDATE reconciliation_break
SET severity = 'critical'
WHERE status = 'open' AND aged_days >= 10;
```

```java
@Component
public class BreakEscalationService {
    
    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void escalateOpenBreaks() {
        List<ReconciliationBreak> critical = breakRepo.findByStatusAndSeverity("open", "critical");
        if (!critical.isEmpty()) {
            // Notify ops + compliance
            emailService.send("ops@bank.tr", 
                "URGENT: " + critical.size() + " critical recon breaks aged > 10 days",
                buildBreakReport(critical));
            
            // Page on-call
            opsgenie.alert(Severity.HIGH, "Critical recon breaks aged");
        }
        
        // Email summary daily
        SummaryReport report = breakRepo.dailySummary();
        emailService.send("recon-team@bank.tr", "Daily Break Summary", report);
    }
}
```

### 8. Resolution patterns

**Pattern 1: Auto-resolve (low severity, mechanism known)**
- Late network update — wait T+2 → match
- Rounding cent diff → write-off threshold (< 0.10 TL)

**Pattern 2: Investigation**
- Compliance officer reviews
- Trace via Topic 9.3 (distributed tracing) → root cause
- Source side correct? Refresh data?

**Pattern 3: Adjustment journal**
```java
public void resolveAmountMismatch(Long breakId, BigDecimal adjustment) {
    ReconciliationBreak b = breakRepo.findById(breakId).orElseThrow();
    
    JournalEntry adj = ledgerService.post(JournalEntryRequest.builder()
        .description("Recon adjustment break #" + breakId)
        .referenceType("recon_adjustment")
        .referenceId(breakId.toString())
        .entry(LedgerEntryRequest.debit("recon_clearing", adjustment, "TRY"))
        .entry(LedgerEntryRequest.credit("recon_offset", adjustment, "TRY"))
        .build());
    
    b.setResolutionJournalId(adj.getId());
    b.setStatus("resolved");
    b.setResolutionAt(Instant.now());
    breakRepo.save(b);
    
    auditService.log("RECON_BREAK_RESOLVED", breakId);
}
```

**Pattern 4: Write-off**
- Investigation impossible (old, no docs)
- Adjustment journal hit P&L (loss account)
- Approval workflow (manager + compliance)

### 9. Card network reconciliation flow

```
Day 1: POS transactions throughout day
  Internal: 10,000 card_authorization records
  
Day 1 EOD: Network sends NDC file
  File: 9,995 records (network's view)
  
Day 2 morning: Recon job runs
  Load NDC → external_network_statement
  Match against internal_card_authorization
  
  Results:
    Matched: 9,990
    Missing internal: 5 (we have, network missing — reject by network?)
    Missing external: 10 (network has, we missing — capture failed?)
    Amount mismatch: 3 (rounding, FX)
    Duplicate internal: 2 (auto-retry happened)
  
  Open breaks: 20
  
Day 2-7: Investigation
  Day 2: Auto-resolve 10 (late network confirms)
  Day 3: Manual resolve 5 (refund)
  Day 4: Write-off 3 (< 1 TL each)
  Day 7: 2 still open, aged → escalate
  
Day 7+: Critical break — pageoncall
```

### 10. EFT reconciliation

```
Internal: customer_transfer table (initiated transfers)
External: TCMB EFT statement (daily file)

Match by EFT reference number + amount + value date.

Common breaks:
- EFT failed (network reject) — internal "pending", external silent
- Late EFT (cross-cutoff) — internal day N, external day N+1
- Customer fee mismatch
- FX EFT (cross-currency)
```

### 11. SWIFT nostro reconciliation

```
Maven Bank's nostro at JPMorgan New York:
  Internal "1103 Nostro JPM USD" ledger balance: 5,234,890.50 USD
  
JPMorgan camt.053 statement (T+1):
  Opening balance: 5,234,890.50 USD
  Movements: list
  Closing balance: 4,891,420.30 USD

Reconciliation:
  Internal closing 4,891,420.30 USD = JPM closing → MATCH
  
If mismatch:
  Missing entries (incoming credit JPM has but Maven didn't post)
  Pending entries (Maven posted but JPM not confirmed)
```

Banking pratiği: **daily nostro recon** mandatory.

### 12. Multi-source recon — banking complex

```
Single transfer touches:
- Internal transfer-service
- Internal account-service
- Internal ledger
- TCMB FAST
- Beneficiary bank (via FAST)

Each system has own view. Recon = align all views.
```

Banking BIG bank: recon team 10-50 person, dedicated software.

### 13. Phase 5 (Batch) integration

EOD recon = Spring Batch job (Topic 5):
- Chunk processing (large files)
- Skip + retry
- Restartable (idempotent)
- Partitioned (parallel)
- Lock (ShedLock) — single instance
- Trace + metric

```java
@Bean
public Job cardReconJob(JobRepository repo) {
    return new JobBuilder("cardReconJob", repo)
        .start(downloadStatementStep())
        .next(loadStatementStep())
        .next(matchStep())
        .next(processBreaksStep())
        .next(notifyStep())
        .build();
}
```

### 14. Banking — reconciliation anti-pattern'leri

**Anti-pattern 1: Manual recon Excel**

Banking küçük → orta zaman çalışır, scale yok. Otomatize zorunlu.

**Anti-pattern 2: Daily recon yok**

Hafta sonu birikim → tespit gecikir, fraud risk. T+1 standart.

**Anti-pattern 3: Break threshold yok**

0.01 TL diff için investigation manual → time waste. Threshold auto-write-off.

**Anti-pattern 4: Recon adjustment ledger bypass**

Manual UPDATE balance → audit broken. Always via ledger (Topic 10.1).

**Anti-pattern 5: Aging yok**

10 gün açık break → kayıp para risk. Aging + escalate.

**Anti-pattern 6: Single-source recon**

Card network reconcile, EFT yok, SWIFT yok → drift. Holistic.

**Anti-pattern 7: Recon late (post-close)**

EOD recon ertesi gün 14:00 → ops investigation kayıplı. Early morning T+1.

**Anti-pattern 8: Investigation eksiz**

Break "resolved" yazıldı ama kim, ne yaptı bilinmiyor. Workflow + audit.

**Anti-pattern 9: Failed recon ignore (next day same)**

Job fail → operations dashboard sessiz. Alert + retry + escalate.

**Anti-pattern 10: Compliance officer dahil değil**

Yüksek tutar break compliance review gerek. Workflow approval.

---

## Önemli olabilecek araştırma kaynakları

- BIS (Bank for International Settlements) settlement docs
- TCMB EFT/FAST operasyon kılavuzu
- BKM NDC dosya formatı dokümanı
- SWIFT MT940 / camt.053 spec
- "Cash Management" — bankacılık textbooks
- Phase 5 batch documentation

---

## Mini task'ler

### Task 10.7.1 — reconciliation_break table + indices (30 dk)

Schema yukarıdan. Test insert + query.

### Task 10.7.2 — NDC file reader (45 dk)

CSV-like format. Spring Batch FlatFileItemReader. 10k record load.

### Task 10.7.3 — Card recon match algorithm (60 dk)

Exact RRN + fallback amount+merchant fuzzy. Match table populate. Unmatched → break.

### Task 10.7.4 — Break categorize + severity (45 dk)

missing_internal, missing_external, amount_mismatch, date_mismatch detection.

### Task 10.7.5 — Aging + escalation (45 dk)

Daily update aged_days. Severity ratchet. Email + pager.

### Task 10.7.6 — Break resolution adjustment journal (60 dk)

Manual resolve → journal entry (Topic 10.1). Auto-write-off < 0.10 TL threshold.

### Task 10.7.7 — EFT reconciliation (60 dk)

TCMB EFT statement format mock. Customer transfer match. Break processing.

### Task 10.7.8 — Nostro SWIFT camt.053 recon (60 dk)

camt.053 parse (Topic 10.3). Opening balance + entries + closing balance check.

### Task 10.7.9 — Recon job Spring Batch + ShedLock (60 dk)

Partitioned chunk processor. ShedLock single-instance. Restartable.

### Task 10.7.10 — Compliance dashboard Grafana (60 dk)

Recon breaks open / aged / by type. Recon job last run status. Critical alerts.

---

## Test yazma rehberi

```java
@Test
@Transactional
void exactMatchByRrn() {
    InternalAuthorization internal = createInternal("RRN-123", new BigDecimal("100.00"));
    ExternalStatement external = createExternal("RRN-123", new BigDecimal("100.00"));
    
    ReconciliationResult result = reconService.run();
    
    assertThat(result.getMatched()).contains(new MatchPair(internal.getId(), external.getId()));
}

@Test
@Transactional
void fuzzyMatchByMerchantAndAmount() {
    InternalAuthorization i = createInternal("RRN-A", new BigDecimal("100.00"), "MERCH-1");
    i.setPostedAt(Instant.parse("2024-05-12T10:30:00Z"));
    ExternalStatement e = createExternal("RRN-B", new BigDecimal("100.00"), "MERCH-1");
    e.setPostedAt(Instant.parse("2024-05-12T10:30:30Z"));   // 30 sec later
    
    ReconciliationResult result = reconService.run();
    
    assertThat(result.getMatched()).hasSize(1);   // Fuzzy match
}

@Test
@Transactional
void shouldDetectMissingInternal() {
    ExternalStatement onlyExternal = createExternal("RRN-X", new BigDecimal("50.00"));
    
    ReconciliationResult result = reconService.run();
    
    assertThat(result.getBreaks()).hasSize(1);
    ReconciliationBreak b = result.getBreaks().get(0);
    assertThat(b.getBreakType()).isEqualTo("missing_internal");
    assertThat(b.getExternalRef()).isEqualTo("RRN-X");
}

@Test
@Transactional
void shouldAgeAndEscalate() {
    ReconciliationBreak b = createBreak();
    b.setAgedDays(11);
    breakRepo.save(b);
    
    escalationService.escalateOpenBreaks();
    
    ReconciliationBreak after = breakRepo.findById(b.getId()).orElseThrow();
    assertThat(after.getSeverity()).isEqualTo("critical");
}

@Test
@Transactional
void resolutionShouldPostAdjustmentJournal() {
    ReconciliationBreak b = createAmountMismatchBreak(new BigDecimal("5.00"));
    
    breakResolutionService.resolveAmountMismatch(b.getId(), new BigDecimal("5.00"));
    
    ReconciliationBreak resolved = breakRepo.findById(b.getId()).orElseThrow();
    assertThat(resolved.getStatus()).isEqualTo("resolved");
    assertThat(resolved.getResolutionJournalId()).isNotNull();
    
    JournalEntry adj = journalRepo.findById(resolved.getResolutionJournalId()).orElseThrow();
    assertThat(adj.getReferenceType()).isEqualTo("recon_adjustment");
}

@Test
void autoWriteOffBelowThreshold() {
    ReconciliationBreak smallBreak = createAmountMismatchBreak(new BigDecimal("0.05"));
    
    reconService.autoResolveSmallBreaks();
    
    ReconciliationBreak after = breakRepo.findById(smallBreak.getId()).orElseThrow();
    assertThat(after.getStatus()).isEqualTo("written_off");
}
```

---

## Claude-verify prompt

```
Reconciliation implementation'ımı banking-grade kriterlere göre değerlendir:

1. Schema:
   - reconciliation_break table?
   - external_statement source table per recon type?
   - matched table separate?
   - aged_days + severity tracking?

2. Match algorithms:
   - Exact match by reference (RRN, EFT ref, EndToEndId)?
   - Fuzzy match (amount tolerance, time tolerance)?
   - Multi-pass strategy?

3. Break types:
   - missing_internal / missing_external?
   - amount_mismatch?
   - date_mismatch?
   - duplicate_internal / duplicate_external?

4. Aging + severity:
   - Daily aged_days increment?
   - Severity ratchet (low → high → critical)?
   - Auto-escalation > 10 days?

5. Resolution:
   - Adjustment journal entry (Topic 10.1)?
   - Auto-write-off small threshold?
   - Compliance officer approval high value?
   - Audit log?

6. Recon types covered:
   - Card network (NDC)?
   - EFT (TCMB)?
   - FAST (TCMB)?
   - SWIFT nostro (camt.053)?
   - Cross-service internal?

7. Spring Batch (Phase 5):
   - Chunk-oriented job?
   - Restartable + idempotent?
   - ShedLock single-instance?
   - Partitioned for parallel?

8. Operations:
   - Daily T+1 schedule?
   - Compliance dashboard?
   - Email + pager escalation?
   - Investigation workflow?

9. Audit:
   - Recon run audit?
   - Break resolution audit?
   - Adjustment journal audit (chain to Topic 10.1)?

10. Anti-pattern:
    - Manual Excel YOK?
    - Daily skip YOK?
    - Recon adjust direct balance UPDATE YOK?
    - Aging yok YOK?
    - Single-source recon YOK?
    - Failed job silent YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] reconciliation_break + external_statement schema
- [ ] NDC card network file reader
- [ ] Card recon match algorithm (exact + fuzzy)
- [ ] EFT recon
- [ ] SWIFT nostro recon (camt.053)
- [ ] Break categorization (4+ type)
- [ ] Aging + escalation
- [ ] Resolution adjustment journal
- [ ] Auto-write-off threshold
- [ ] Spring Batch + ShedLock
- [ ] Compliance dashboard
- [ ] 8+ integration test

---

## Defter notları (10 madde)

1. "Clearing vs Settlement banking ayrımı + DNS vs RTGS: ____."
2. "Nostro / Vostro hesapları banking correspondent banking: ____."
3. "Reconciliation 5 source (card, EFT, FAST, SWIFT, internal) banking: ____."
4. "Match algoritma (exact ref + fuzzy amount+merchant+time) banking: ____."
5. "Break types (missing internal/external, amount/date mismatch, duplicate): ____."
6. "Aging + severity ratchet + escalation banking ops workflow: ____."
7. "Break resolution adjustment journal (Topic 10.1) + audit: ____."
8. "Auto-write-off threshold vs compliance officer approval high value: ____."
9. "Spring Batch (Phase 5) + ShedLock + idempotent restartable: ____."
10. "Banking recon team 10-50 person + compliance dashboard + dedicated software: ____."
