# Phase 10 Mini-Project — TR Banking Domain Integration

## Hedef

Phase 7-9 microservice + observability + security üzerine **TR banking domain** entegre etmek. Double-entry ledger, ISO 8583 kart, ISO 20022/SWIFT cross-border, EFT/FAST TR payment, FX/interest, regulatory (MASAK/KVKK/BDDK), reconciliation. **Mini bir TR banka backend'i**.

## Süre

15-20 gün (günde 3 saat)

## Önbilgi

- Phase 9 mini-project tamam
- Topic 10.1-10.7 bitti

---

## Görev listesi

### 1. Chart of Accounts + Ledger service (1.5 gün)

(Topic 10.1)

`ledger-service` microservice:

```sql
-- COA seed
INSERT INTO account (code, name, parent_code, account_type, currency) VALUES
('1', 'ASSETS', NULL, 'asset', NULL),
('1100', 'Cash & Equivalents', '1', 'asset', NULL),
('1101', 'Vault Cash', '1100', 'asset', 'TRY'),
('1102', 'Central Bank Reserve TRY', '1100', 'asset', 'TRY'),
('1103', 'Nostro JPMorgan USD', '1100', 'asset', 'USD'),
('1104', 'Nostro Deutsche Bank EUR', '1100', 'asset', 'EUR'),
('1200', 'Loans Receivable', '1', 'asset', NULL),
('1201', 'Consumer Loans', '1200', 'asset', NULL),
('1202', 'Mortgage Loans', '1200', 'asset', NULL),
-- ... 50+ accounts
('6100', 'EFT Outgoing Clearing', '6', 'intermediate', NULL),
('6200', 'FAST Outgoing Clearing', '6', 'intermediate', NULL),
('6300', 'Card Network Clearing', '6', 'intermediate', NULL),
('6400', 'SWIFT Outgoing Clearing', '6', 'intermediate', NULL);

-- journal_entry, ledger_entry tables (Topic 10.1)

-- Customer account = subaccount of 2101 (per customer)
-- account.parent_code = '2101', account.code = '2101-' || customer_id
```

`LedgerService`:
- `post(JournalEntryRequest)` — atomic + balance check
- `reverse(journalId)` — offset entry
- `balanceOf(accountId, asOfDate)` — sum/snapshot
- `trialBalance()` — global D vs C
- `transactionHistory(accountId, fromDate, toDate)`

Multi-currency segregation. Immutable trigger.

### 2. Card service — ISO 8583 (2 gün)

(Topic 10.2)

`card-service`:

- POST `/v1/cards/authorize` — POS/ATM authorization (REST API for simulation; production: ISO 8583 over jPOS)
- POST `/v1/cards/capture` — settlement capture (batch)
- POST `/v1/cards/reverse` — reversal
- POST `/v1/cards/refund` — refund

Internal jPOS server:
- 0100/0110 auth
- 0220 advice
- 0400/0410 reversal

Authorization logic:
- Card status check
- Limit check (daily, monthly)
- Balance check (debit card)
- 3D Secure (e-commerce)
- Fraud rules
- KKB check (credit card application)

Ledger posting (Topic 10.1):
```
Authorization:
  Hold (no journal)

Capture (T+1 batch):
  Debit  Customer Card Account
  Credit Card Network Clearing  (6300)

Settlement (T+2):
  Debit  Card Network Clearing  (6300)
  Credit Nostro - card network bank

Refund:
  Reverse of capture
```

PAN tokenization (Topic 8.6). PCI-DSS scope reduction.

### 3. Transfer service — TR EFT/FAST (2 gün)

(Topic 10.4)

`transfer-service`:

- POST `/v1/transfers/eft` — EFT initiate
- POST `/v1/transfers/fast` — FAST initiate
- POST `/v1/transfers/havale` — intra-bank
- GET `/v1/transfers/{id}/status` — status

Logic:
- IBAN validate (MOD-97)
- Bank code resolve
- Working hours check (EFT) / 24-7 (FAST)
- FAST limit check (70k TL)
- Sanctions screening (MASAK — Topic 10.6)
- Customer limit check
- BSMV calculation

Ledger posting:
```
Havale (intra-bank):
  Debit  Customer A
  Credit Customer B

EFT outgoing:
  Initiate:
    Debit  Customer A
    Credit EFT Outgoing Clearing  (6100)
  
  Settlement (T+0/T+1 from TCMB):
    Debit  EFT Outgoing Clearing
    Credit Central Bank Reserve   (1102)

FAST: similar to EFT but real-time (10 sec)
```

Outbox pattern (Topic 6.6) for TCMB integration (async).

### 4. Cross-border service — SWIFT (1.5 gün)

(Topic 10.3)

`xborder-service`:

- POST `/v1/transfers/swift` — cross-border initiate

Logic:
- SWIFT BIC resolve
- pacs.008 build (Prowide)
- XSD validate
- UETR generate
- Sanctions screening
- Currency conversion (Topic 10.5 FX)
- Correspondent routing

Ledger:
```
Initiate:
  Debit  Customer A (USD)
  Credit SWIFT Outgoing Clearing  (6400-USD)

Settlement (T+1 via nostro):
  Debit  SWIFT Outgoing Clearing
  Credit Nostro JPMorgan USD     (1103)
```

pacs.002 status update consume → completion / rejection.

### 5. Loan service — interest + amortization (1 gün)

(Topic 10.5)

`loan-service`:

- POST `/v1/loans/apply` — application
- GET `/v1/loans/{id}/schedule` — amortization table
- POST `/v1/loans/{id}/early-payoff` — pre-mature

Logic:
- KKB Findeks score (Topic 10.4)
- Risk-based pricing
- Amortization (French)
- BSMV + KKDF calculation
- YMO disclosure

Daily interest accrual job (Topic 10.5):
```
@Scheduled(cron = "0 30 23 * * *")
public void accrueDailyInterest() { ... }

Ledger entry:
  Debit  Loan Receivable
  Credit Interest Income  (4101)
```

BSMV monthly recognize → state liability.

### 6. FX service (1 gün)

(Topic 10.5)

`fx-service`:

- GET `/v1/fx/rates` — rate board (bid/ask)
- POST `/v1/fx/convert` — customer FX
- POST `/v1/fx/forward` — forward contract

Rate provider (TCMB rate feed mock):
- Refresh every 5 minutes
- Bid/ask spread per currency pair
- Spread widens in stress

Ledger:
```
Customer buys 1000 USD at 32.60:
  Debit  Customer A TRY            32600.00
  Credit Bank FX Position TRY      32600.00
  
  Debit  Bank FX Position USD       1000.00
  Credit Customer A USD             1000.00
```

### 7. Compliance service — MASAK + KVKK + audit (1.5 gün)

(Topic 10.6)

`compliance-service`:

- Real-time transaction monitoring (Kafka consumer)
- Sanctions screening
- STR workflow (alert → review → submit)
- Consent management
- Right to be forgotten (crypto-shred — Topic 8.6)
- Audit hash chain (Topic 8.7)

Daily sanctions refresh:
- OFAC SDN list
- EU sanctions
- MASAK national

Rules (Topic 10.6):
- Smurfing
- Large cash
- Sanctions counterparty
- PEP transaction
- Cross-border anomaly

### 8. Reconciliation service (1.5 gün)

(Topic 10.7)

`recon-service`:

EOD jobs:
- Card NDC reconciliation
- EFT reconciliation (TCMB statement)
- FAST reconciliation
- SWIFT nostro reconciliation (camt.053 parse)

Spring Batch (Phase 5):
- Chunk + restart + ShedLock
- Match: exact ref + fuzzy
- Break categorize
- Aging + escalation
- Resolution adjustment journal

Compliance dashboard:
- Open breaks
- Aged breaks
- Critical alerts

### 9. EOD master job (1 gün)

Orchestration:

```
EOD Master Job (23:30 - 02:00):
1. Stop online traffic gracefully
2. Run interest accrual (Topic 10.5)
3. Run BSMV/KKDF monthly close (last day of month)
4. Generate statements (camt.053)
5. Submit MASAK reports
6. Run reconciliations (cards, EFT, SWIFT)
7. Compute regulatory metrics (CAR, LCR)
8. Backup ledger snapshot
9. Restart online traffic
```

Each step Spring Batch + ShedLock + retry + audit.

### 10. Kafka event flow (1 gün)

Inter-service events:
- `customer-events` — KYC, onboarding, profile change
- `card-events` — authorization, capture, reversal, fraud alert
- `transfer-events` — EFT/FAST/havale lifecycle
- `loan-events` — application, approval, accrual, payment
- `fx-events` — rate change, customer conversion
- `compliance-events` — alert, STR, sanctions hit
- `recon-events` — break, resolution
- `ledger-events` — journal posted, balance change

Outbox pattern (Topic 6.6) per service. Idempotent consumers.

### 11. Production-like senaryolar (1.5 gün)

#### Senaryo 1: Customer journey — onboarding to first transfer
1. Customer apply → KYC (MERNIS + KKB) → account open
2. Cash deposit 10,000 TL → ledger posting
3. Card issued + activate
4. Online purchase 500 TL → card authorization → settlement T+1 → recon match
5. EFT 1000 TL → TCMB → recipient bank → completed
6. Audit trail complete

#### Senaryo 2: Cross-border transfer with FX
1. Customer requests 1000 USD transfer to Germany
2. FX rate 32.50 → 32,500 TL debit
3. Sanctions screening pass
4. pacs.008 generated + UETR
5. SWIFT outbox → mock SWIFTNet
6. Status update pacs.002 → settled
7. Ledger updated
8. SWIFT nostro recon match

#### Senaryo 3: Fraud alert
1. Customer pattern: small transactions in 24h adding to 50k → smurfing
2. MASAK rule triggers
3. Compliance officer review → STR draft
4. Officer approve → STR submit
5. Customer accounts flag → restrict
6. Audit trail

#### Senaryo 4: Loan lifecycle
1. Customer apply 100k TL loan
2. KKB Findeks score 1500 → eligible
3. Loan approve, amortization generate (60 month)
4. YMO disclosure
5. Daily accrual posting
6. Monthly auto-debit installment
7. Early payoff at month 24 → outstanding calc + bonus
8. Loan closed → ledger zero balance

#### Senaryo 5: Recon break resolution
1. Card NDC file shows 100 TL extra (missing internal)
2. Recon break critical aged 5 days
3. Investigation: missing capture due to outbox lag
4. Adjustment journal: debit customer, credit clearing
5. Recon resolution
6. Audit

#### Senaryo 6: EOD master job
1. Stop online traffic
2. Interest accrual 50k loans
3. Daily statements 500k customers
4. MASAK monthly batch report
5. Card recon T-1 settle
6. SWIFT nostro recon
7. CAR/LCR calc + BDDK report
8. All complete in 2.5 hours

#### Senaryo 7: KVKK right to be forgotten
1. Customer requests deletion
2. Crypto-shred customer encryption key
3. Audit trail preserved (pseudonymized)
4. Regulatory retention (10 yıl) maintained for transaction history only
5. Marketing consent revoked
6. Confirmation customer

### 12. Defter notları (20 madde)

1. "Banking double-entry ledger 6 tip transaction (deposit, transfer, EFT, interest, FX, fee): ____."
2. "Chart of Accounts hierarchy + intermediate clearing accounts banking design: ____."
3. "ISO 8583 card authorization flow (0100/0110) + EMV ARQC + PCI-DSS scope: ____."
4. "PAN tokenization + acquirer/issuer/network roles: ____."
5. "TR payment systems EFT/FAST/havale routing + working hours + limits: ____."
6. "TR IBAN MOD-97 + bank code 5 digit + 26 char format: ____."
7. "SWIFT pacs.008 cross-border + UETR + correspondent (nostro/vostro): ____."
8. "ISO 20022 migration Kasım 2025 EOL + MX-first new implementation: ____."
9. "FX bid/ask spread + customer perspective + forward CIP: ____."
10. "BSMV %5 + KKDF %10 consumer loan TR-specific tax: ____."
11. "YMO (effective annual cost) BDDK + TBB disclosure regulation: ____."
12. "Day count conventions (ACT/365, ACT/360, ACT/ACT, 30/360) product map: ____."
13. "Loan amortization French equal payment + last installment residual: ____."
14. "Daily interest accrual + ledger idempotency + time zone: ____."
15. "MASAK STR + sanctions screening + smurfing detection rules: ____."
16. "KVKK consent + right to be forgotten + crypto-shred + retention conflict: ____."
17. "BDDK 5411 + Bilgi Sistemleri + retention 10 yıl + 24h incident report: ____."
18. "Reconciliation 5 source (card, EFT, FAST, SWIFT, internal) + break aging: ____."
19. "EOD master job orchestration (interest, statement, recon, regulatory): ____."
20. "Senior banking engineer mind-map: domain + tech stack birleştirme: ____."

---

## Tamamlama kriterleri

- [ ] ledger-service (Chart of Accounts + journal/ledger immutable)
- [ ] card-service (ISO 8583 0100/0210/0220/0400 + tokenization)
- [ ] transfer-service (EFT/FAST/havale + IBAN + working hours + sanctions)
- [ ] xborder-service (SWIFT pacs.008 + UETR + nostro recon)
- [ ] loan-service (KKB + amortization + BSMV/KKDF + daily accrual)
- [ ] fx-service (rate board + customer conversion + forward)
- [ ] compliance-service (MASAK rules + STR + consent + audit chain)
- [ ] recon-service (5 source recon + Spring Batch + break aging)
- [ ] EOD master job orchestration
- [ ] Kafka event flow (8+ topic) + outbox pattern
- [ ] 7 senaryo reproduce + verify (customer journey, cross-border, fraud, loan, recon break, EOD, KVKK)
- [ ] 20 defter notu
- [ ] Architecture diagram + flow diagrams

---

## Önemli not

Phase 10 = **banking domain expert** transition. Bu mini-project'i bitiren:

- TR banking ecosystem'ini end-to-end anlamış (TCMB, BKM, BDDK, MASAK, KKB)
- Banking transaction lifecycle + accounting impact biliyor
- Regulatory compliance code-level uygulayabiliyor
- Cross-border SWIFT modernization (ISO 20022) takip ediyor

Bu seviye TR bankalarında **mid+/senior backend engineer** pozisyonlar için **ayırt edici**. Salt teknik (Phase 1-9) + domain (Phase 10) = TR bankası **interview'e hazır** profil.

Phase 11 (DevOps) + Phase 12 (Testing) ile **production deploy + quality assurance**'ı tamamlayacağız.
