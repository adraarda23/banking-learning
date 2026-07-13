# Faz 10 — PHASE TEST

Faz 11'e geçmeden önce kendini sına.

## Pratik test

- [ ] Chart of Accounts + journal/ledger immutable + reverse pattern
- [ ] 8 banking transaction tipi (deposit, intra-customer, EFT, FAST, havale, FX, interest, fee)
- [ ] Card service ISO 8583 (0100/0210/0220/0400) + tokenization + EMV
- [ ] Transfer service TR (EFT/FAST/havale) + IBAN + working hours + 70k FAST limit
- [ ] Cross-border SWIFT pacs.008 + UETR + nostro reconciliation
- [ ] Loan service amortization + KKB + BSMV/KKDF + YMO + daily accrual
- [ ] FX service bid/ask + customer conversion + forward
- [ ] Compliance service MASAK rules (3+) + STR + KVKK consent + crypto-shred
- [ ] Reconciliation service 5 source + break aging + adjustment journal
- [ ] EOD master job orchestration (Phase 5 Spring Batch + ShedLock)
- [ ] Kafka event flow 8+ topic + outbox pattern (Topic 6.6)
- [ ] 7 production-like senaryo reproduce + verify

## Konsept testi

### Double-Entry Accounting
- [ ] Double vs single-entry banking neden zorunlu (audit, trial balance)
- [ ] Debit/credit kuralları bank perspective (asset/liability)
- [ ] Customer deposit liability sebebi
- [ ] Chart of Accounts hierarchy + intermediate clearing
- [ ] Journal vs Ledger + atomic posting
- [ ] Stripe immutable + reversal pattern
- [ ] Balance compute (sum/materialized/snapshot) trade-off
- [ ] Multi-currency segregation FX
- [ ] Accrual vs cash basis IFRS uyumu
- [ ] Anti-pattern (direct balance UPDATE, float money, cross-currency sum)

### ISO 8583
- [ ] Acquirer → network → issuer flow
- [ ] MTI 4-digit decoding
- [ ] Bitmap primary + secondary
- [ ] Critical fields (PAN, Amount, STAN, Response code, Auth ID)
- [ ] Authorization 0100/0110 + hold balance + EMV ARQC
- [ ] Capture 0220 advice + ledger posting
- [ ] Reversal 0400 idempotent 24h SLA
- [ ] PCI-DSS PAN tokenization + log masking
- [ ] TR — BKM switch + Troy + 3D Secure + KKB
- [ ] ISO 8583 → ISO 20022 migration

### ISO 20022 & SWIFT
- [ ] SWIFT MT vs ISO 20022 migration timeline (Nov 2025)
- [ ] ISO 20022 prefix (pacs/pain/camt) + numbering
- [ ] pacs.008 FI to FI customer credit transfer
- [ ] UETR SWIFT GPI end-to-end tracking
- [ ] BIC + IBAN validation pre-send
- [ ] SEPA SCT + SCT Inst + SDD
- [ ] ChrgBr OUR/SHA/BEN trade-off
- [ ] camt.053 EOD statement
- [ ] Outbox + async SWIFT delivery
- [ ] PII anti-pattern remittance info + sanctions before send

### TR Payment Systems
- [ ] TR ecosystem (TCMB + BKM + KKB + BDDK + MASAK + SPK) roles
- [ ] EFT vs FAST trade-off (hours, amount, fee, settlement)
- [ ] Havale intra-bank instant book transfer
- [ ] TR IBAN format MOD-97 + bank code
- [ ] FAST Karekod TLV + static/dynamic QR
- [ ] BKM kart switch + Troy strategic
- [ ] TCMB EFT/FAST operation + nostro
- [ ] BDDK 5411 + Open Banking + IT regulations
- [ ] MASAK STR + smurfing + daily refresh
- [ ] TR business calendar (bayram) banking critical

### FX & Interest
- [ ] Simple vs compound interest products
- [ ] Day count conventions (ACT/365, ACT/360, ACT/ACT, 30/360)
- [ ] French amortization + interest portion pattern
- [ ] BSMV %5 + KKDF %10 consumer loan TR
- [ ] APR (BKO) vs APY/EAR (YMO/NKO)
- [ ] FX bid/ask spread + customer buy/sell
- [ ] FX forward CIP TRY high rate scenario
- [ ] Daily interest accrual + ledger idempotency
- [ ] Time zone Europe/Istanbul + leap year + holiday
- [ ] TBB + BDDK consumer regulation YMO

### Regulatory
- [ ] TR regulatory landscape (BDDK + MASAK + KVKK + TCMB + SPK)
- [ ] BDDK 5411 + Bilgi Sistemleri Tebliği IT compliance
- [ ] MASAK STR + sanctions + 3 rule
- [ ] KVKK consent + right to be forgotten + crypto-shred
- [ ] PCI-DSS PAN tokenization + Level 1
- [ ] KKB Findeks + cache + cost optimization
- [ ] GDPR vs KVKK paralel + cross-border AB
- [ ] Basel III CAR + LCR + NSFR daily computation
- [ ] Incident notification 24h BDDK + 72h KVKK
- [ ] Audit trail 14+ event + retention 10 yıl + hash chain

### Reconciliation & Settlement
- [ ] Clearing vs Settlement + DNS vs RTGS
- [ ] Nostro / Vostro correspondent banking
- [ ] Reconciliation 5 source (card, EFT, FAST, SWIFT, internal)
- [ ] Match algoritma (exact + fuzzy)
- [ ] Break types (missing internal/external, amount/date mismatch, duplicate)
- [ ] Aging + severity + escalation workflow
- [ ] Resolution adjustment journal + audit
- [ ] Auto-write-off threshold vs high-value approval
- [ ] Spring Batch (Phase 5) + ShedLock
- [ ] Banking recon team + compliance dashboard

## Banking domain skill check

- [ ] Bir transfer'in 6 service'i (transfer, account, ledger, compliance, recon, audit) gezdiği flow'u beyaz tahtaya çizebilirim
- [ ] Cross-border transfer customer + acquirer + network + issuer + nostro/vostro flow'unu açıklayabilirim
- [ ] Customer credit application'dan loan disbursement'a kadar KYC + KKB + risk + amortization + accrual'ı süreçleyebilirim
- [ ] MASAK STR senaryosu reproduce + investigation + submit pratik yaptım
- [ ] EOD master job orchestration design edebilirim
- [ ] Reconciliation break root cause walkthrough yapabilirim
- [ ] BDDK + KVKK + PCI-DSS + MASAK compliance matrix backend mapping
- [ ] Banking transaction lifecycle ledger impact biliyor

## Soft skills

- [ ] 20 mini-project defter notu yazılmış
- [ ] Architecture + flow diagrams çıkarıldı
- [ ] 7 senaryo end-to-end verified
- [ ] Banking jargon (acquirer, issuer, nostro, vostro, clearing, settlement, STR, CTR, YMO, BSMV, KKDF, FAST, EFT, EMV) rahat kullanım

## Kaç gün?

Tahmin: 40-50 gün (günde 3 saat). Phase 10 = **banking domain mastery**. Bu phase'i bitirenler **mid+ → senior** banking engineer profili kazanır.

---

Hepsine "evet" → **Faz 11'e geç → 11-devops/**

---

## Bonus — TR banking senior engineer işareti

Phase 10 bittikten sonra şu cümleleri söyleyebiliyorsan:

- "TR banking ecosystem TCMB/BKM/KKB/BDDK/MASAK roller ve interaction matrix'i çizebilirim."
- "Double-entry ledger banking-grade tasarımı (atomic, immutable, multi-currency, reversal) yapabilirim."
- "ISO 8583 card flow + PCI-DSS scope reduction + EMV cryptogram doğrulama uygulayabilirim."
- "ISO 20022 migration (Kasım 2025) plan + MT/MX coexistence/translator + UETR tracking."
- "TR EFT/FAST/havale routing logic + working hours + limits + sanctions screening."
- "Loan amortization French + BSMV/KKDF + YMO disclosure + daily accrual idempotent."
- "MASAK rule engine (smurfing, large cash, sanctions) + STR workflow."
- "KVKK consent + right to be forgotten + audit retention conflict resolution."
- "BDDK Basel III CAR/LCR daily compute + regulatory XML reporting."
- "Reconciliation 5 source + aging + adjustment journal + Spring Batch orchestration."
- "EOD master job orchestration + 24-hour incident notification."

TR bankalarında **senior banking backend engineer** olarak interview'a hazır. CV'de **"banking domain expertise"** ile teknik competence beraber sunmak — TR banka mid+/senior maaş skalası için **rekabetçi** profil.

---

## Banking için Phase 10'un yeri

Phase 10 = **banking domain genuine knowledge**. Sırf teknik (Phase 1-9) yetersiz — banking sektöründe **domain expertise** karşılığı yüksek.

- **BDDK denetim hazırlığı:** Compliance + audit code-level uygulanmış
- **MASAK uyum:** Sanctions + STR real-time + automated
- **KVKK uyum:** Consent + retention + crypto-shred
- **TR-specific:** EFT/FAST/IBAN/BSMV/KKDF — yerel knowledge
- **Cross-border:** SWIFT migration + correspondent banking
- **Banking ops:** EOD orchestration + reconciliation + adjustment

TR bankalarında engineer **iş bağlamında** çalışırken bu domain knowledge **günlük decision-making'i** yönlendirir. Phase 11 (DevOps) ve Phase 12 (Testing) operasyonel + quality dimension ekleyecek.

Senior banking engineer için Phase 10 **necessary but not sufficient** — Phase 11-12 ile birlikte **complete profile**.
