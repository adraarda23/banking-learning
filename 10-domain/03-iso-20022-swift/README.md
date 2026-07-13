# Topic 10.3 — ISO 20022 & SWIFT MT

## Hedef

Banking messaging için ISO 20022 (modern XML/JSON) ve SWIFT MT (legacy) standardlarını derinlemesine öğrenmek. pacs.008, pain.001, camt.053 vb. mesaj tipleri, SEPA, cross-border payment, SWIFT FIN, SWIFT GPI, real-time settlement, ISO 20022 migration timeline (Kasım 2025 sonu — coexistence end).

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1, 10.2 bitti
- XML temel
- Cross-border payment kavram

---

## Kavramlar

### 1. Banking messaging landscape

| Standard | Use Case | Status |
|---|---|---|
| **ISO 8583** | Kart işlemleri (POS/ATM) | Active (Topic 10.2) |
| **SWIFT MT** | Cross-border wire | Legacy, EOL Nov 2025 |
| **SWIFT FIN** | SWIFT network layer | Active |
| **ISO 20022** | Modern universal | Active, growing |
| **SEPA** | EUR payments in Europe | Active (ISO 20022 subset) |
| **NACHA / ACH** | US domestic | Active (US-specific) |
| **TR EFT/FAST** | TR domestic | Active (TR-specific, ISO 20022 inspired) |

### 2. SWIFT MT — Message Type (legacy)

3-digit codes:
- `MT103` — Customer Credit Transfer (en yaygın retail wire)
- `MT202` — Bank-to-bank fund transfer
- `MT202 COV` — Cover payment (correspondent banking)
- `MT900/910` — Debit/Credit confirmation
- `MT940/950` — Statement
- `MT196/199` — Free format (queries, advice)

**Structure:** Block-based plain text.

```
{1:F01BANKTRIS0XXX0000000000}
{2:I103OTHERBANKXXXXN}
{3:{108:MT103REF}}
{4:
:20:REF20240512001
:23B:CRED
:32A:240512TRY100000,00
:33B:TRY100000,00
:50K:/TR123456789012345678
JOHN DOE
ISTANBUL TR
:59:/DE987654321098765432
JANE SMITH
BERLIN DE
:70:INVOICE PAYMENT 12345
:71A:OUR
-}
{5:{CHK:ABCDEF123456}}
```

**Block breakdown:**
- Block 1: Basic header (sender BIC, terminal ID, session)
- Block 2: Application header (msg type, receiver BIC)
- Block 3: User header (optional, reference)
- Block 4: Text block (financial data — most important)
- Block 5: Trailer (MAC checksum)

**Field tags in Block 4:**
- `:20:` — Sender reference
- `:23B:` — Bank operation code
- `:32A:` — Value date, currency, amount (24051TR100000,00)
- `:33B:` — Currency/instructed amount
- `:50K:` — Ordering customer
- `:59:` — Beneficiary
- `:70:` — Remittance info
- `:71A:` — Details of charges (OUR/BEN/SHA)

### 3. SWIFT MT charges (`:71A:`)

- `OUR` — Sender pays all charges
- `BEN` — Beneficiary pays all charges
- `SHA` — Shared (sender pays own bank, beneficiary pays receiving bank)

Banking pratiği: TR çıkışlı çoğu **SHA** (cost optimization).

### 4. ISO 20022 — modern standard

XML-based (JSON variants growing). Structured, validated by XSD schema.

**Categorization (4-letter):**

| Prefix | Domain | Banking |
|---|---|---|
| `pacs` | Payments Clearing and Settlement | Inter-bank |
| `pain` | Payments Initiation | Customer → bank |
| `camt` | Cash Management | Statement, balance |
| `acmt` | Account Management | Open, close |
| `auth` | Authorities | Regulatory |
| `caaa` | Card / ATM | (overlap ISO 8583) |
| `tsmt` | Trade Services | Trade finance |
| `setr` | Securities | Securities |
| `colr` | Collateral | Repo |
| `fxtr` | FX Trade | FX |

**Message naming:** `{prefix}.{number}.{variant}.{version}`

Examples:
- `pacs.008.001.10` — Customer Credit Transfer (CCT)
- `pacs.002.001.12` — Payment Status Report
- `pacs.004.001.11` — Payment Return
- `pacs.009.001.10` — Financial Institution Credit Transfer
- `pain.001.001.11` — Customer Credit Transfer Initiation
- `pain.002.001.13` — Customer Status Report
- `camt.052.001.10` — Bank-to-Customer Account Report
- `camt.053.001.10` — Bank-to-Customer Statement
- `camt.054.001.10` — Bank-to-Customer Debit/Credit Notification
- `camt.056.001.10` — Payment Recall

### 5. ISO 20022 example — pacs.008

`pacs.008` (FI to FI Customer Credit Transfer) — interbank transfer:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:iso:std:iso:20022:tech:xsd:pacs.008.001.10">
    <FIToFICstmrCdtTrf>
        <GrpHdr>
            <MsgId>MAVEN-2024-05-12-001</MsgId>
            <CreDtTm>2024-05-12T10:30:45Z</CreDtTm>
            <NbOfTxs>1</NbOfTxs>
            <CtrlSum>100000.00</CtrlSum>
            <SttlmInf>
                <SttlmMtd>CLRG</SttlmMtd>
                <ClrSys>
                    <Cd>TRGT2</Cd>
                </ClrSys>
            </SttlmInf>
            <InstgAgt>
                <FinInstnId>
                    <BICFI>MAVNRIS0XXX</BICFI>
                </FinInstnId>
            </InstgAgt>
            <InstdAgt>
                <FinInstnId>
                    <BICFI>OTHERBKDEFFXXX</BICFI>
                </FinInstnId>
            </InstdAgt>
        </GrpHdr>
        <CdtTrfTxInf>
            <PmtId>
                <InstrId>INSTR-001</InstrId>
                <EndToEndId>E2E-20240512-001</EndToEndId>
                <TxId>TX-MAVEN-20240512-001</TxId>
                <UETR>db8a3a6b-7e7d-4cad-94c4-d3eaa9c6e7b1</UETR>
            </PmtId>
            <IntrBkSttlmAmt Ccy="EUR">100000.00</IntrBkSttlmAmt>
            <IntrBkSttlmDt>2024-05-13</IntrBkSttlmDt>
            <ChrgBr>SHAR</ChrgBr>
            <Dbtr>
                <Nm>John Doe</Nm>
                <PstlAdr>
                    <StrtNm>Atatürk Caddesi</StrtNm>
                    <BldgNb>15</BldgNb>
                    <PstCd>34000</PstCd>
                    <TwnNm>Istanbul</TwnNm>
                    <Ctry>TR</Ctry>
                </PstlAdr>
            </Dbtr>
            <DbtrAcct>
                <Id>
                    <IBAN>TR320010009999987654321098</IBAN>
                </Id>
            </DbtrAcct>
            <DbtrAgt>
                <FinInstnId>
                    <BICFI>MAVNRIS0XXX</BICFI>
                </FinInstnId>
            </DbtrAgt>
            <CdtrAgt>
                <FinInstnId>
                    <BICFI>OTHERBKDEFFXXX</BICFI>
                </FinInstnId>
            </CdtrAgt>
            <Cdtr>
                <Nm>Jane Smith</Nm>
                <PstlAdr>
                    <StrtNm>Friedrichstrasse</StrtNm>
                    <BldgNb>100</BldgNb>
                    <PstCd>10117</PstCd>
                    <TwnNm>Berlin</TwnNm>
                    <Ctry>DE</Ctry>
                </PstlAdr>
            </Cdtr>
            <CdtrAcct>
                <Id>
                    <IBAN>DE89370400440532013000</IBAN>
                </Id>
            </CdtrAcct>
            <RmtInf>
                <Ustrd>Invoice payment 12345</Ustrd>
            </RmtInf>
        </CdtTrfTxInf>
    </FIToFICstmrCdtTrf>
</Document>
```

**Critical fields:**
- `MsgId` — message unique ID
- `EndToEndId` — customer end-to-end reference
- `UETR` — Unique End-to-end Transaction Reference (SWIFT GPI tracking)
- `BICFI` — Bank Identifier Code (8 or 11 char)
- `IBAN` — Account number international format
- `ChrgBr` — Charge bearer (SHAR/DEBT/CRED)

### 6. UETR — SWIFT GPI tracking

UETR = 36-char UUID. Cross-border payment için end-to-end traceability.

```
GPI Tracker URL:
https://tracker.swift.com/track?uetr=db8a3a6b-7e7d-4cad-94c4-d3eaa9c6e7b1

Status:
- Initiated (sender bank)
- In transit (correspondent)
- Credited (beneficiary bank)
- Settled
- Returned
- Status SLA target: 80% < 30 minutes
```

Banking pratiği: SWIFT GPI **mandatory** modern correspondent banking.

### 7. SEPA — Single Euro Payments Area

EUR-only ISO 20022 subset for European harmonized payments.

**SCT (SEPA Credit Transfer):**
- `pain.001` customer initiation
- `pacs.008` interbank
- D+1 settlement (max 1 business day)

**SCT Inst (Instant):**
- `pacs.008` with high priority
- 10 saniye target
- 24/7/365

**SDD (SEPA Direct Debit):**
- `pain.008` initiation
- `pacs.003` interbank
- Mandate-based recurring

**SEPA characteristics:**
- IBAN+BIC mandatory
- Charge: SHAR (shared)
- EUR only
- 33 European countries

Türkiye SEPA üyesi DEĞİL (EUR area dışı), ama SEPA-compatible TR bankalar var (correspondent network).

### 8. ISO 20022 migration — banking 2025

SWIFT MT → ISO 20022 migration **Kasım 2025 sonu**.

Timeline:
- **Mart 2023:** MX (ISO 20022) coexistence başladı
- **Kasım 2025:** MT100 series (MT103, MT202 vb.) **end-of-life**
- 2026+: MX-only

Banking sektörü preparation:
- MT/MX translator (legacy bağlantılar için)
- Rich data uplift (ISO 20022 daha zengin)
- Operational training
- Regulatory reporting yeni format
- Sanctions screening parser update

### 9. Java implementation — Prowide

[Prowide](https://www.prowidesoftware.com/) ISO 20022 + SWIFT MT Java library.

```xml
<dependency>
    <groupId>com.prowidesoftware</groupId>
    <artifactId>pw-swift-core</artifactId>
    <version>SRU2024-9.5.5</version>
</dependency>
<dependency>
    <groupId>com.prowidesoftware</groupId>
    <artifactId>pw-iso20022</artifactId>
    <version>SRU2024-9.5.5</version>
</dependency>
```

**SWIFT MT — build:**

```java
MT103 mt = new MT103()
    .append(new Field20("REF20240512001"))
    .append(new Field23B("CRED"))
    .append(new Field32A("240512", "EUR", "100000,00"))
    .append(new Field50K("/TR320010009999987654321098\nJOHN DOE\nISTANBUL TR"))
    .append(new Field59("/DE89370400440532013000\nJANE SMITH\nBERLIN DE"))
    .append(new Field70("INVOICE PAYMENT 12345"))
    .append(new Field71A("SHA"));

mt.setSender("MAVNRIS0");
mt.setReceiver("OTHERBKDEFF");

String swiftMessage = mt.message();
System.out.println(swiftMessage);
```

**SWIFT MT — parse:**

```java
MT103 parsed = MT103.parse(swiftMessage);
String reference = parsed.getField20().getValue();
String amount = parsed.getField32A().getAmount();
String currency = parsed.getField32A().getCurrency();
String beneficiaryName = parsed.getField59().getNameAndAddress().get(0);
```

**ISO 20022 pacs.008 — build:**

```java
Pacs00800110 pacs = new Pacs00800110();
FIToFICustomerCreditTransferV10 doc = new FIToFICustomerCreditTransferV10();

GroupHeader113 hdr = new GroupHeader113();
hdr.setMsgId("MAVEN-2024-05-12-001");
hdr.setCreDtTm(now());
hdr.setNbOfTxs("1");
hdr.setCtrlSum(new BigDecimal("100000.00"));
doc.setGrpHdr(hdr);

CreditTransferTransaction50 tx = new CreditTransferTransaction50();
PaymentIdentification7 pmtId = new PaymentIdentification7();
pmtId.setInstrId("INSTR-001");
pmtId.setEndToEndId("E2E-20240512-001");
pmtId.setUETR(UUID.randomUUID().toString());
tx.setPmtId(pmtId);

ActiveCurrencyAndAmount amt = new ActiveCurrencyAndAmount();
amt.setCcy("EUR");
amt.setValue(new BigDecimal("100000.00"));
tx.setIntrBkSttlmAmt(amt);

// ... debtor, creditor, agents

doc.getCdtTrfTxInf().add(tx);
pacs.setFIToFICstmrCdtTrf(doc);

String xml = MxParseUtils.write(pacs);
```

### 10. MT → MX translation (during migration)

Coexistence period için bridge:

```java
// MT103 → pacs.008 translate
MT103 mt = MT103.parse(swiftMessage);
Pacs00800110 mx = MtMxTranslator.toMx(mt);

// Reverse direction
Pacs00800110 mx2 = Pacs00800110.parse(xmlMessage);
MT103 mt2 = MtMxTranslator.toMt(mx2);
```

Trade-off: MT structured az → MX'e zengin info infer edilemez. Banking için **MX-first** yeni implementation.

### 11. camt.053 — bank-to-customer statement

```xml
<Document xmlns="urn:iso:std:iso:20022:tech:xsd:camt.053.001.10">
    <BkToCstmrStmt>
        <GrpHdr>
            <MsgId>STMT-20240512-001</MsgId>
            <CreDtTm>2024-05-12T23:59:00Z</CreDtTm>
        </GrpHdr>
        <Stmt>
            <Id>STMT-001</Id>
            <ElctrncSeqNb>1</ElctrncSeqNb>
            <FrToDt>
                <FrDtTm>2024-05-12T00:00:00Z</FrDtTm>
                <ToDtTm>2024-05-12T23:59:59Z</ToDtTm>
            </FrToDt>
            <Acct>
                <Id>
                    <IBAN>TR320010009999987654321098</IBAN>
                </Id>
                <Ccy>TRY</Ccy>
            </Acct>
            <Bal>
                <Tp><CdOrPrtry><Cd>OPBD</Cd></CdOrPrtry></Tp>
                <Amt Ccy="TRY">10000.00</Amt>
                <CdtDbtInd>CRDT</CdtDbtInd>
                <Dt><Dt>2024-05-12</Dt></Dt>
            </Bal>
            <Bal>
                <Tp><CdOrPrtry><Cd>CLBD</Cd></CdOrPrtry></Tp>
                <Amt Ccy="TRY">10500.00</Amt>
                <CdtDbtInd>CRDT</CdtDbtInd>
                <Dt><Dt>2024-05-12</Dt></Dt>
            </Bal>
            <TxsSummry>
                <TtlNtries>
                    <NbOfNtries>3</NbOfNtries>
                </TtlNtries>
            </TxsSummry>
            <Ntry>
                <Amt Ccy="TRY">1000.00</Amt>
                <CdtDbtInd>CRDT</CdtDbtInd>
                <Sts><Cd>BOOK</Cd></Sts>
                <BookgDt><Dt>2024-05-12</Dt></BookgDt>
                <ValDt><Dt>2024-05-12</Dt></ValDt>
                <BkTxCd>
                    <Domn>
                        <Cd>PMNT</Cd>
                        <Fmly><Cd>RCDT</Cd><SubFmlyCd>BOOK</SubFmlyCd></Fmly>
                    </Domn>
                </BkTxCd>
                <NtryDtls>
                    <TxDtls>
                        <Refs>
                            <EndToEndId>E2E-20240512-001</EndToEndId>
                        </Refs>
                        <RltdPties>
                            <Dbtr><Pty><Nm>John Doe</Nm></Pty></Dbtr>
                        </RltdPties>
                        <RmtInf>
                            <Ustrd>Salary May 2024</Ustrd>
                        </RmtInf>
                    </TxDtls>
                </NtryDtls>
            </Ntry>
            <!-- more Ntry -->
        </Stmt>
    </BkToCstmrStmt>
</Document>
```

EOD statement → customer download / API integration.

### 12. ISO 20022 schema validation

XSD schema vendor sağlar (paid). Banking implementation:

```java
SchemaFactory factory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = factory.newSchema(new File("schemas/pacs.008.001.10.xsd"));

Validator validator = schema.newValidator();
validator.validate(new StreamSource(new StringReader(xmlMessage)));
// Throws SAXException on invalid
```

Banking ingestion pipeline: validation **mandatory**.

### 13. Banking integration — pacs.008 transfer flow

```java
@Service
public class CrossBorderTransferService {
    
    private final BicResolver bicResolver;
    private final SwiftConnector swiftConnector;
    private final LedgerService ledgerService;
    
    @Transactional
    public TransferResult initiate(CrossBorderTransferRequest req) {
        // 1. Validate IBAN, BIC
        validateIban(req.creditorIban());
        String creditorBic = bicResolver.fromIban(req.creditorIban());
        
        // 2. Sanctions screening (Topic 10.6)
        sanctionsScreener.check(req.creditorName(), req.creditorCountry());
        
        // 3. Construct pacs.008
        Pacs00800110 message = buildPacs008(req, creditorBic);
        
        // 4. Validate XSD
        validateSchema(message);
        
        // 5. Persist outbox (Topic 6.6 dual-write safe)
        outboxRepo.save(OutboxEvent.builder()
            .aggregate("transfer")
            .aggregateId(req.transferId().toString())
            .payload(MxParseUtils.write(message))
            .destination("swift")
            .build());
        
        // 6. Ledger post (Topic 10.1)
        ledgerService.post(
            JournalEntry.builder()
                .debit(req.debtorAccount(), req.amount(), req.currency())
                .credit(SWIFT_OUTGOING_CLEARING, req.amount(), req.currency())
                .build());
        
        // 7. Outbox publisher will send via SwiftConnector to SWIFTNet
        return TransferResult.builder()
            .uetr(message.getFIToFICstmrCdtTrf().getCdtTrfTxInf().get(0).getPmtId().getUETR())
            .status("PENDING")
            .build();
    }
    
    @KafkaListener(topics = "swift-status-updates")
    public void onStatusUpdate(SwiftStatusUpdate update) {
        // Receive pacs.002 status report
        Transfer t = transferRepo.findByUetr(update.uetr()).orElseThrow();
        
        if ("ACSC".equals(update.statusCode())) {   // AcceptedSettlementCompleted
            ledgerService.post(
                JournalEntry.builder()
                    .debit(SWIFT_OUTGOING_CLEARING, t.getAmount(), t.getCurrency())
                    .credit(NOSTRO_ACCOUNT_TARGET, t.getAmount(), t.getCurrency())
                    .build());
            t.markCompleted();
        } else if ("RJCT".equals(update.statusCode())) {
            // Rejected — refund debtor
            ledgerService.reverse(t.getInitialJournalId());
            t.markRejected(update.reasonCode());
        }
    }
}
```

### 14. Banking — ISO 20022/SWIFT anti-pattern'leri

**Anti-pattern 1: MT-only forever**

Kasım 2025 sonrası MT message reject edilir. Migration plan zorunlu.

**Anti-pattern 2: XSD validation skip**

Production'da invalid message → downstream reject + reconcile chaos. Always validate.

**Anti-pattern 3: UETR generate yok**

GPI tracking için UETR şart. Yeni payment her zaman UETR.

**Anti-pattern 4: PII in `Ustrd` (remittance info)**

```xml
<Ustrd>Payment to Jane Smith TC: 12345678901</Ustrd>   <!-- ❌ KVKK -->
```

Remittance info **invoice number, reference** olmalı, PII değil.

**Anti-pattern 5: Charge bearer karışıklığı**

OUR/SHA/BEN trade-off açık olmalı customer onayında. UX kritik.

**Anti-pattern 6: BIC validation yok**

Invalid BIC → SWIFT reject + refund SLA gecikir. BIC validation library kullan.

**Anti-pattern 7: IBAN check digit hesaplama yok**

IBAN MOD-97 check digit. App-level validate, DB write öncesi.

**Anti-pattern 8: Synchronous SWIFT call**

SWIFT round-trip dakikalar/saatler. Outbox pattern (Topic 6.6) + async.

**Anti-pattern 9: Idempotency yok**

SWIFT mesajı duplicate gönderim → double settlement. Idempotency key.

**Anti-pattern 10: Reconciliation eksik**

Sent SWIFT + status update mismatch → ledger drift. EOD reconciliation (Topic 10.7).

---

## Önemli olabilecek araştırma kaynakları

- ISO 20022 official (iso20022.org)
- SWIFT documentation (swift.com)
- Prowide library docs
- SWIFT MX migration guide 2025
- SEPA Rulebook (EBA Clearing)
- CBRT (TCMB) SWIFT operations rehberi

---

## Mini task'ler

### Task 10.3.1 — Prowide setup + MT103 build (45 dk)

Maven dep. MT103 message build + parse roundtrip. Field validation.

### Task 10.3.2 — pacs.008 ISO 20022 build (60 dk)

Cross-border transfer XML. Prowide ISO 20022 API. XSD validate.

### Task 10.3.3 — UETR generate + tracking (30 dk)

UUID per transfer. DB store. Status update event.

### Task 10.3.4 — MT103 ↔ pacs.008 translator (60 dk)

MT/MX bridge. Translator library kullan ya da manuel field mapping.

### Task 10.3.5 — camt.053 statement parse (45 dk)

EOD statement XML parse. Entry list, balance breakdown.

### Task 10.3.6 — pacs.002 status report handle (45 dk)

ACSC, RJCT status codes. Ledger update accordingly.

### Task 10.3.7 — pain.001 customer initiation (45 dk)

Customer → bank batch transfer initiation. pain.001 XML.

### Task 10.3.8 — IBAN MOD-97 validator (30 dk)

IBAN check digit algorithm. TR format support.

### Task 10.3.9 — BIC validator (30 dk)

8 or 11 char. Country code (chars 5-6). Bank code, location.

### Task 10.3.10 — Outbox + SWIFT async (60 dk)

pacs.008 message outbox table. Publisher → SwiftConnector (mock). Status update event consume.

---

## Test yazma rehberi

```java
@Test
void shouldBuildMT103() {
    MT103 mt = new MT103()
        .append(new Field20("TEST123"))
        .append(new Field32A("240512", "EUR", "100000,00"))
        // ... 
        ;
    
    String message = mt.message();
    
    MT103 parsed = MT103.parse(message);
    assertThat(parsed.getField20().getValue()).isEqualTo("TEST123");
    assertThat(parsed.getField32A().getAmount()).isEqualTo("100000,00");
}

@Test
void shouldBuildAndValidatePacs008() throws Exception {
    Pacs00800110 msg = buildTestPacs008();
    String xml = MxParseUtils.write(msg);
    
    SchemaFactory factory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
    Schema schema = factory.newSchema(getClass().getResource("/schemas/pacs.008.001.10.xsd"));
    Validator validator = schema.newValidator();
    validator.validate(new StreamSource(new StringReader(xml)));
    // No exception = valid
}

@Test
void shouldGenerateUniqueUetr() {
    Pacs00800110 msg1 = buildTestPacs008();
    Pacs00800110 msg2 = buildTestPacs008();
    
    String uetr1 = uetrOf(msg1);
    String uetr2 = uetrOf(msg2);
    
    assertThat(uetr1).isNotEqualTo(uetr2);
    assertThat(uetr1).matches("[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}");
}

@ParameterizedTest
@ValueSource(strings = {
    "TR320010009999987654321098",
    "DE89370400440532013000",
    "GB29NWBK60161331926819"
})
void shouldAcceptValidIban(String iban) {
    assertThat(IbanValidator.isValid(iban)).isTrue();
}

@ParameterizedTest
@ValueSource(strings = {
    "TR320010009999987654321099",   // Wrong checksum
    "TR3200100099998",                // Too short
    "XX320010009999987654321098"      // Invalid country
})
void shouldRejectInvalidIban(String iban) {
    assertThat(IbanValidator.isValid(iban)).isFalse();
}

@Test
void shouldNotIncludeSensitiveDataInRemittanceInfo() {
    CrossBorderTransferRequest req = ...;
    Pacs00800110 msg = service.buildMessage(req);
    
    String xml = MxParseUtils.write(msg);
    assertThat(xml).doesNotContain("12345678901");   // No TC
    assertThat(xml).doesNotContain(req.debtorCardPan());
}
```

---

## Claude-verify prompt

```
ISO 20022 + SWIFT implementation'ımı banking-grade kriterlere göre değerlendir:

1. Library:
   - Prowide ISO 20022 / SWIFT?
   - XSD schema validation?
   - SRU (Standards Release Update) up to date?

2. Message types:
   - SWIFT MT103 customer credit transfer?
   - MT202 bank-to-bank?
   - ISO 20022 pacs.008 (FI to FI)?
   - ISO 20022 pain.001 (customer initiation)?
   - ISO 20022 camt.053 (statement)?
   - ISO 20022 pacs.002 (status report)?

3. Critical fields:
   - UETR generated for every transfer?
   - BIC validated (8 or 11 char)?
   - IBAN MOD-97 check digit validated?
   - ChrgBr (charge bearer) explicit?
   - EndToEndId customer reference?

4. Migration:
   - MT → MX translator for coexistence?
   - Kasım 2025 EOL aware?
   - New implementations MX-first?

5. Banking flow:
   - Cross-border transfer pacs.008 initiation?
   - Sanctions screening before send?
   - Outbox pattern (Topic 6.6)?
   - Ledger posting with clearing account?
   - Status update (pacs.002) consume?
   - Refund on rejection?

6. Schema validation:
   - XSD validation pipeline?
   - Reject invalid messages?

7. PII protection:
   - Remittance info no PII?
   - Customer data minimum required (KVKK)?

8. SWIFT GPI:
   - UETR tracking?
   - Status updates consume?

9. SEPA awareness:
   - SCT / SCT Inst / SDD differences?
   - EUR-only constraint?

10. Anti-pattern:
    - MT-only post-2025 YOK?
    - XSD validation skip YOK?
    - UETR yok YOK?
    - PII in remittance YOK?
    - Sync SWIFT call YOK?
    - Idempotency yok YOK?
    - Reconciliation eksik YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Prowide setup
- [ ] MT103 build + parse roundtrip
- [ ] pacs.008 ISO 20022 XML + XSD validate
- [ ] UETR generate + tracking
- [ ] MT ↔ MX translator
- [ ] camt.053 statement parse
- [ ] pacs.002 status report handle (ACSC, RJCT)
- [ ] pain.001 customer initiation
- [ ] IBAN + BIC validator
- [ ] Outbox + async SWIFT delivery
- [ ] PII-free remittance info audit
- [ ] 8+ integration test

---

## Defter notları (10 madde)

1. "SWIFT MT vs ISO 20022 banking 2025 migration timeline: ____."
2. "ISO 20022 prefix (pacs/pain/camt) + message numbering banking: ____."
3. "pacs.008 cross-border FI to FI customer credit transfer: ____."
4. "UETR SWIFT GPI tracking + end-to-end visibility: ____."
5. "BIC + IBAN MOD-97 validation banking pre-send: ____."
6. "SEPA SCT + SCT Inst + SDD EUR area harmonization: ____."
7. "ChrgBr OUR/SHA/BEN trade-off banking customer UX: ____."
8. "camt.053 bank-to-customer statement EOD daily: ____."
9. "Outbox pattern + async SWIFT delivery + status consume: ____."
10. "PII anti-pattern remittance info + sanctions screening before send: ____."
