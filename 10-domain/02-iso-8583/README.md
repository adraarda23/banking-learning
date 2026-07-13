# Topic 10.2 — ISO 8583: Card Transaction Messaging

## Hedef

ISO 8583 kart işlem mesajlaşma standardını banking-grade derinlikte öğrenmek: MTI (Message Type Indicator), bitmap, data elements, transaction flow (authorization, capture, refund, reversal), POS/ATM acquirer-issuer flow, BKM (Türk kart yapısı), Visa/Mastercard/Troy network entegrasyonu, EMV co-existence, ISO 8583 → ISO 20022 migration.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1 (Double-entry accounting) bitti
- Kart işleme temel kavramı
- Binary protokol bilmek bonus

---

## Kavramlar

### 1. ISO 8583 — what + why

ISO 8583 = **financial transaction card-originated message** standardı. Versions:
- ISO 8583:1987 — orijinal
- ISO 8583:1993 — güncellenmiş
- ISO 8583:2003 — modern (XML-friendly versions)

**Use cases:**
- POS terminal → acquirer → network (Visa/MC/Troy) → issuer
- ATM cash withdrawal
- E-commerce auth
- Refund / void / reversal
- Card authorization (online verification)
- Card capture (batch settlement)

**Banking ecosystem:**
```
[POS Terminal]                                 
   ↓ ISO 8583
[Merchant Acquirer Bank]                       
   ↓ ISO 8583
[Network (Visa/Mastercard/Troy/BKM)]           
   ↓ ISO 8583
[Issuer Bank (kart sahibi banka)]              
   ↓ approve/decline
... back to merchant
```

End-to-end < 2 saniye (target).

### 2. Message Type Indicator (MTI) — 4 digit

```
Position 1: Version
  0 — ISO 8583:1987
  1 — ISO 8583:1993
  2 — ISO 8583:2003

Position 2: Message class
  1 — Authorization
  2 — Financial
  3 — File actions
  4 — Reversal
  5 — Reconciliation
  6 — Administrative
  7 — Fee collection
  8 — Network management
  9 — Reserved

Position 3: Message function
  0 — Request
  1 — Request response
  2 — Advice
  3 — Advice response
  4 — Notification

Position 4: Message origin
  0 — Acquirer
  1 — Acquirer repeat
  2 — Issuer
  3 — Issuer repeat
  4 — Other
```

**Common MTI:**

| MTI | Anlam |
|---|---|
| `0100` | Authorization Request (online auth, ATM) |
| `0110` | Authorization Response |
| `0120` | Authorization Advice |
| `0200` | Financial Request (purchase, capture) |
| `0210` | Financial Response |
| `0220` | Financial Advice (offline forwarded) |
| `0400` | Reversal Request |
| `0410` | Reversal Response |
| `0500` | Reconciliation Request |
| `0800` | Network Management (echo, sign-on, sign-off) |
| `0810` | Network Management Response |

### 3. Bitmap — which fields are present

Each ISO 8583 message has 1-2 bitmaps (64 or 128 bits).

```
Primary bitmap (bits 1-64): mandatory
Secondary bitmap (bits 65-128): if bit 1 of primary set
```

Bit X set → field X present in message.

```
Hex bitmap example: 7220000102C04822
Binary:             0111 0010 0010 0000 0000 0000 ...

Bit 2 set → PAN present
Bit 3 set → Processing code present
Bit 4 set → Amount, transaction present
...
```

Parsing logic:
```java
public class IsoMessage {
    private String mti;
    private BitSet bitmap;
    private Map<Integer, String> fields;
    
    public void parse(byte[] data) {
        // MTI first 4 chars (BCD or ASCII)
        mti = readAscii(data, 0, 4);
        
        // Primary bitmap: bytes 4-12 (8 bytes = 64 bits)
        bitmap = parseBitmap(data, 4, 8);
        
        int offset = 12;
        if (bitmap.get(0)) {   // bit 1 = secondary bitmap
            BitSet secondary = parseBitmap(data, offset, 8);
            bitmap.or(secondary);   // 128 total
            offset += 8;
        }
        
        for (int i = 2; i <= 128; i++) {   // bit 1 is bitmap indicator
            if (bitmap.get(i - 1)) {
                FieldDefinition def = FIELD_DEFS.get(i);
                String value = readField(data, offset, def);
                fields.put(i, value);
                offset += def.length(value);
            }
        }
    }
}
```

### 4. Data elements — most important fields

| Bit | Field | Format | Banking örnek |
|---|---|---|---|
| 2 | PAN (Primary Account Number) | n..19 LLVAR | "4532148803436467" |
| 3 | Processing code | n6 | "000000" (purchase), "010000" (cash withdrawal) |
| 4 | Amount, transaction | n12 | "000000010000" = 100.00 TL (2 decimal implicit) |
| 5 | Amount, settlement | n12 | (after FX) |
| 6 | Amount, cardholder billing | n12 | (multi-currency) |
| 7 | Transmission date+time | n10 (MMDDhhmmss) | "0512103045" |
| 11 | Systems trace audit number (STAN) | n6 | "123456" — unique per device |
| 12 | Local transaction time | n6 (hhmmss) | "103045" |
| 13 | Local transaction date | n4 (MMDD) | "0512" |
| 14 | Expiration date | n4 (YYMM) | "2512" |
| 18 | Merchant type / MCC | n4 | "5411" = grocery |
| 22 | POS entry mode | n3 | "012" = magnetic stripe, "051" = EMV chip |
| 25 | POS condition code | n2 | "00" = normal, "02" = customer not present |
| 32 | Acquiring institution ID | n..11 LLVAR | "987654321" |
| 35 | Track 2 data | z..37 LLVAR | "4532...=2512..." (deprecated, PCI-DSS) |
| 37 | Retrieval reference number (RRN) | an12 | "240512103045" |
| 38 | Authorization ID response | an6 | "123456" — issued by issuer |
| 39 | Response code | an2 | "00" = approved, "05" = decline |
| 41 | Card acceptor terminal ID | ans8 | "TERM0001" |
| 42 | Card acceptor ID | ans15 | "MERCHANT0000001" |
| 43 | Card acceptor name/location | ans40 | "MARKET A123 ISTANBUL TR" |
| 49 | Currency code, transaction | n3 | "949" = TRY |
| 52 | PIN data | b8 | encrypted block |
| 54 | Additional amounts | an..120 | (cash back, tax) |
| 55 | ICC data (EMV) | an..255 LLLVAR | TLV EMV chip data |
| 62 | Custom data | an..999 LLLVAR | Network-specific |

**Format codes:**
- `n` = numeric
- `a` = alpha
- `an` = alphanumeric
- `ans` = alphanumeric + special
- `b` = binary
- `z` = track 2
- `LLVAR` = 2-digit length prefix
- `LLLVAR` = 3-digit length prefix

### 5. Response codes — common

| Code | Anlam | Banking |
|---|---|---|
| `00` | Approved | Authorized |
| `01` | Refer to card issuer | Manual review |
| `04` | Pickup card | Stolen / blocked |
| `05` | Do not honor | Generic decline |
| `12` | Invalid transaction | Bad data |
| `13` | Invalid amount | Limit exceeded |
| `14` | Invalid card number | PAN invalid |
| `15` | No such issuer | Routing fail |
| `41` | Lost card | Card status |
| `43` | Stolen card | Card status |
| `51` | Insufficient funds | Balance |
| `54` | Expired card | Date check |
| `55` | Invalid PIN | PIN verify |
| `57` | Transaction not permitted | Card type/CVV |
| `61` | Activity amount limit exceeded | Daily limit |
| `62` | Restricted card | Country block |
| `65` | Activity count limit exceeded | TX count |
| `75` | PIN tries exceeded | Block |
| `91` | Issuer or switch inoperative | Timeout |
| `96` | System malfunction | Generic |

### 6. Authorization flow — full example

#### Step 1: Cardholder swipes / inserts / taps card at POS

POS construct message:

```
MTI: 0100
Bit 2 (PAN): 4532148803436467
Bit 3 (Processing code): 000000 (purchase from primary account)
Bit 4 (Amount): 000000010000 (100.00 TL)
Bit 7 (Transmission): 0512103045
Bit 11 (STAN): 123456
Bit 12 (Local time): 103045
Bit 13 (Local date): 0512
Bit 14 (Expiry): 2512
Bit 22 (POS entry): 051 (EMV chip)
Bit 25 (POS cond): 00 (normal)
Bit 41 (Terminal ID): TERM0001
Bit 42 (Merchant ID): 0000001MARKETACME
Bit 49 (Currency): 949 (TRY)
Bit 55 (EMV chip data): <TLV EMV>
```

#### Step 2: POS → Acquirer bank (Maven Bank)

Acquirer:
- Validate message format
- Verify merchant active
- Apply fraud rules (velocity, geography)
- Route to network (Visa/Mastercard/Troy/BKM)

#### Step 3: Acquirer → Network → Issuer

Network routing by **BIN (Bank Identification Number)**, first 6-8 digits of PAN.

```
PAN: 4532-1488-0343-6467
BIN: 453214 → resolves to "ABC Bank issuer"
```

Network forwards to issuer bank.

#### Step 4: Issuer decision

Issuer:
- Card lookup (PAN encrypted/tokenized internally)
- Card status check (active, not blocked)
- Velocity check (daily TX count)
- Limit check (daily amount)
- Balance check (debit card) / credit available (credit card)
- Fraud score
- 3D Secure verify (e-commerce için)
- PIN/CVV verify

Decision: approve / decline → response code.

If approve: **hold amount** on card account (authorization hold).

#### Step 5: Issuer response

```
MTI: 0110
Bit 2-49: (echo)
Bit 38 (Auth ID): 123456 (issuer-generated)
Bit 39 (Response): 00 (approved) or other
```

#### Step 6: Back through network → acquirer → POS

POS: prints receipt, customer leaves with goods.

#### Step 7: Settlement (later, batch)

Acquirer batches authorizations (typically EOD):
```
MTI: 0220 (Advice)
... captured authorizations ...
```

Sent to network → cleared via central bank.

Issuer: actual debit (was hold, now confirmed).

### 7. Reversal flow — what when wrong

POS timeout, partial complete, customer cancellation:

```
MTI: 0400 (Reversal Request)
Bit 90 (Original data elements): MTI + STAN + transmission date+time + acquirer ID + forwarding ID
Bit 39: (reason code)
```

Issuer: release hold.

**Banking critical:**
- Reversal **idempotent** (network retry safe)
- 24-hour SLA reversal window
- Late reversal → chargeback dispute

### 8. EMV chip data — bit 55

EMV (Europay/Mastercard/Visa) chip data carried in bit 55 (TLV format):

```
Tag-Length-Value:

9F1A 02 0840           # Terminal country code
5A   08 4532148803436467  # PAN
9F26 08 ABCD1234...     # Application Cryptogram (ARQC)
9F27 01 80              # Cryptogram Information Data
9F36 02 00FF            # Application Transaction Counter
```

EMV cryptogram (ARQC) = card-generated, includes transaction details + dynamic counter → **replay attack prevention**.

Issuer verifies cryptogram with shared key → genuine card.

### 9. Banking — Türk kart altyapısı (BKM, Troy)

**BKM (Bankalararası Kart Merkezi):**
- TR domestic card network
- Switch between Turkish banks
- ATM/POS routing
- Settlement coordination

**Troy:**
- Turkish chip-and-PIN card brand (national)
- BKM operates
- Visa/MC alternative for domestic

**TR card flow specifics:**
- All POS traffic via BKM switch (regulation)
- 3D Secure mandatory online (BDDK requirement)
- KKB (Kredi Kayıt Bürosu) — credit score check before approval

### 10. Implementation — ISO 8583 Java library

`jPOS` (open source) en yaygın:

```xml
<dependency>
    <groupId>org.jpos</groupId>
    <artifactId>jpos</artifactId>
    <version>2.1.10</version>
</dependency>
```

Build message:

```java
ISOMsg msg = new ISOMsg();
msg.setMTI("0100");
msg.set(2, "4532148803436467");                 // PAN
msg.set(3, "000000");                           // Processing code
msg.set(4, "000000010000");                     // Amount 100.00 TL
msg.set(7, format(Instant.now(), "MMddHHmmss"));
msg.set(11, "123456");                          // STAN
msg.set(12, format(Instant.now(), "HHmmss"));
msg.set(13, format(Instant.now(), "MMdd"));
msg.set(22, "051");                             // EMV
msg.set(41, "TERM0001");                        // Terminal
msg.set(42, "MERCHANT0000001");
msg.set(49, "949");                             // TRY

// Pack to binary
ISOPackager packager = new ISO87BPackager();
msg.setPackager(packager);
byte[] data = msg.pack();
```

Parse incoming:
```java
ISOMsg incoming = new ISOMsg();
incoming.setPackager(packager);
incoming.unpack(data);

String mti = incoming.getMTI();
String pan = incoming.getString(2);
String amount = incoming.getString(4);
String responseCode = incoming.getString(39);
```

### 11. ISO 8583 server — TCP socket

jPOS Q2 framework:

```xml
<!-- q2 deploy file -->
<channel-adaptor name="acquirer-channel" 
                 class="org.jpos.q2.iso.ChannelAdaptor">
    <channel class="org.jpos.iso.channel.ASCIIChannel" 
             packager="org.jpos.iso.packager.ISO87BPackager">
        <property name="host" value="acquirer.bank.tr"/>
        <property name="port" value="9999"/>
    </channel>
    <in>send-queue</in>
    <out>receive-queue</out>
</channel-adaptor>

<server name="server" class="org.jpos.q2.iso.QServer">
    <attr name="port" type="java.lang.Integer">8000</attr>
    <channel class="org.jpos.iso.channel.NACChannel" 
             packager="org.jpos.iso.packager.ISO87BPackager"/>
    <request-listener class="com.bank.iso.AuthRequestListener">
        <property name="space" value="space:default"/>
        <property name="queue" value="auth-queue"/>
    </request-listener>
</server>
```

### 12. ISO 8583 in Spring Boot — modern wrapping

```java
@Service
public class CardAuthorizationService {
    
    private final AcquirerClient acquirerClient;
    
    public AuthorizationResult authorize(AuthorizationRequest req) {
        ISOMsg msg = new ISOMsg();
        msg.setMTI("0100");
        msg.set(2, req.pan());
        msg.set(3, processingCode(req.transactionType()));
        msg.set(4, formatAmount(req.amount(), 2));   // 2 decimal implicit
        // ... fields
        
        ISOMsg response = acquirerClient.send(msg, Duration.ofSeconds(5));
        
        String rc = response.getString(39);
        String authId = response.getString(38);
        
        return AuthorizationResult.builder()
            .approved("00".equals(rc))
            .responseCode(rc)
            .authorizationId(authId)
            .build();
    }
    
    private String formatAmount(BigDecimal amount, int decimals) {
        // 100.00 TL → "000000010000"
        BigDecimal scaled = amount.movePointRight(decimals);
        return String.format("%012d", scaled.toBigInteger());
    }
}
```

### 13. PCI-DSS + ISO 8583

ISO 8583 carries **cardholder data** (PAN, expiry, track data).

**PCI-DSS requirements:**
- PAN encrypted at rest (envelope encryption Topic 8.6)
- PAN masked in logs (last 4 only)
- Track data NEVER stored after authorization
- CVV NEVER stored
- Encryption in transit (mutual TLS)
- Logs without sensitive data
- Tokenization preferred (acquirer stores token, not PAN)

```java
@Component
public class IsoMessageMasker {
    
    public String maskForLog(ISOMsg msg) {
        ISOMsg copy = (ISOMsg) msg.clone();
        if (copy.hasField(2)) {
            String pan = copy.getString(2);
            copy.set(2, pan.substring(0, 6) + "******" + pan.substring(pan.length() - 4));
        }
        if (copy.hasField(35)) copy.set(35, "***track2***");
        if (copy.hasField(52)) copy.set(52, "***pin***");
        if (copy.hasField(55)) copy.set(55, "***emv***");
        return copy.toString();
    }
}
```

### 14. ISO 8583 → ISO 20022 migration

Industry moving to ISO 20022 (XML/JSON, richer):
- Visa: Visa Token Service moving partly
- Mastercard: MTN program
- TR (BKM): hybrid period

Banking için:
- Legacy ISO 8583 sürmeye devam (10+ yıl)
- Yeni features ISO 20022
- Bridge / adapter pattern (banking için yaygın)

### 15. Banking — ISO 8583 anti-pattern'leri

**Anti-pattern 1: PAN log'da plain**

```java
log.info("Authorization for PAN: {}", pan);   // ❌ PCI-DSS ihlali
```

**Anti-pattern 2: Track data store**

Authorization sonrası track data saklamak. PCI-DSS yasak.

**Anti-pattern 3: CVV log/store**

CVV asla saklanmaz. Authorization sırasında geçer, sonra silinir.

**Anti-pattern 4: Sync POS terminal timeout**

POS 5+ saniye bekler → terminal locked. Async ack + later reconcile.

**Anti-pattern 5: Reversal yok / late**

Timeout durumunda reversal şart. Banking 24h SLA.

**Anti-pattern 6: STAN reuse**

STAN per-terminal unique. Reuse = duplicate transaction risk.

**Anti-pattern 7: Manual field positions hardcoded**

Magic numbers. Use packager + constants.

**Anti-pattern 8: ISO message in DB / Kafka as binary blob**

Audit zor. Parse + structured store + masked.

**Anti-pattern 9: Issuer response code generic**

Specific codes (51 insufficient, 54 expired, 61 limit) > generic 05. UX impact.

**Anti-pattern 10: EMV cryptogram verify yok**

Magstripe-only flow → cloning vulnerable. EMV ARQC verify şart.

---

## Önemli olabilecek araştırma kaynakları

- ISO 8583 specification (paid)
- jPOS documentation
- "ISO 8583 Implementation Guide" — vendor docs
- BKM (Bankalararası Kart Merkezi) docs
- Visa/Mastercard operating regulations
- EMV books (4 cilt EMVCo)
- PCI-DSS v4.0

---

## Mini task'ler

### Task 10.2.1 — jPOS setup + first message (45 dk)

Maven dependency. Build `0100` MTI message with required fields. Pack/unpack roundtrip.

### Task 10.2.2 — Bitmap parse (30 dk)

Hex `7220000102C04822` bitmap → which bits set → field list.

### Task 10.2.3 — Field formatter (45 dk)

Amount BigDecimal → 12-digit fixed format. Date format. PAN length-prefix.

### Task 10.2.4 — POS simulator (60 dk)

CLI tool: PAN + amount input → 0100 message construct → send to mock acquirer.

### Task 10.2.5 — Acquirer server (60 dk)

jPOS Q2 server. Listen port 8000. Parse 0100. Validate. Return 0110 with response code.

### Task 10.2.6 — Issuer authorization logic (60 dk)

Card balance check, limit check, fraud check (velocity), response code map. Approve/decline.

### Task 10.2.7 — Reversal flow (45 dk)

0400 reversal request. Find original by STAN + transmission date. Release hold. Idempotent.

### Task 10.2.8 — Settlement (capture) batch (45 dk)

EOD job: authorized but uncaptured → 0220 advice batch. Ledger posting (Topic 10.1).

### Task 10.2.9 — PCI-DSS masking (30 dk)

PAN, track, CVV, PIN, EMV cryptogram masking in logs. Test.

### Task 10.2.10 — Network management (echo) (30 dk)

0800 echo request → 0810 response. Heartbeat keep-alive.

---

## Test yazma rehberi

```java
@Test
void shouldBuildAuthorizationRequest() throws Exception {
    ISOMsg msg = new ISOMsg();
    msg.setMTI("0100");
    msg.set(2, "4532148803436467");
    msg.set(3, "000000");
    msg.set(4, "000000010000");
    msg.set(7, "0512103045");
    msg.set(11, "123456");
    msg.set(41, "TERM0001");
    msg.set(42, "MERCHANT0000001");
    msg.set(49, "949");
    
    ISOPackager packager = new ISO87BPackager();
    msg.setPackager(packager);
    byte[] packed = msg.pack();
    
    // Verify roundtrip
    ISOMsg parsed = new ISOMsg();
    parsed.setPackager(packager);
    parsed.unpack(packed);
    
    assertThat(parsed.getMTI()).isEqualTo("0100");
    assertThat(parsed.getString(2)).isEqualTo("4532148803436467");
    assertThat(parsed.getString(4)).isEqualTo("000000010000");
}

@Test
void shouldApproveValidAuthorization() {
    AuthorizationRequest req = AuthorizationRequest.builder()
        .pan("4532148803436467")
        .amount(new BigDecimal("100.00"))
        .currency("TRY")
        .terminalId("TERM0001")
        .merchantId("MERCHANT0000001")
        .build();
    
    AuthorizationResult result = authService.authorize(req);
    
    assertThat(result.isApproved()).isTrue();
    assertThat(result.responseCode()).isEqualTo("00");
    assertThat(result.authorizationId()).isNotEmpty();
}

@Test
void shouldDeclineWhenInsufficientFunds() {
    cardRepo.setBalance("4532148803436467", new BigDecimal("50.00"));
    
    AuthorizationResult result = authService.authorize(
        AuthorizationRequest.builder().pan("4532...").amount(new BigDecimal("100.00")).build());
    
    assertThat(result.isApproved()).isFalse();
    assertThat(result.responseCode()).isEqualTo("51");
}

@Test
void shouldHandleReversalIdempotently() {
    ISOMsg auth = sendAuthorization(...);
    
    ISOMsg reversal = sendReversal(auth);
    ISOMsg reversalRetry = sendReversal(auth);   // Same reversal
    
    // Hold released exactly once
    assertThat(cardRepo.getHoldCount(pan)).isEqualTo(0);
}

@Test
void shouldMaskPanInLogs(CapturedOutput output) {
    sendAuthorization(...);
    
    assertThat(output.getOut()).doesNotContain("4532148803436467");
    assertThat(output.getOut()).contains("453214");
    assertThat(output.getOut()).contains("******");
    assertThat(output.getOut()).contains("6467");
}

@Test
void shouldNeverStoreTrackData() {
    sendAuthorization(...);   // with track 2 in field 35
    
    List<CardAuthorization> stored = authRepo.findAll();
    stored.forEach(a -> {
        assertThat(a.getTrack2()).isNull();
        assertThat(a.getCvv()).isNull();
        assertThat(a.getPan()).isNotEqualTo("4532148803436467");   // tokenized
    });
}
```

---

## Claude-verify prompt

```
ISO 8583 implementation'ımı banking-grade kriterlere göre değerlendir:

1. jPOS setup:
   - ISO87BPackager veya custom?
   - MTI builder + bitmap auto?
   - Q2 framework or standalone?

2. Message types:
   - 0100/0110 Authorization?
   - 0200/0210 Financial?
   - 0220 Advice?
   - 0400/0410 Reversal?
   - 0800/0810 Network Management?

3. Field handling:
   - PAN (bit 2) length-prefix?
   - Amount (bit 4) 12-digit fixed BigDecimal conversion?
   - Date/time format conversion?
   - STAN (bit 11) unique per terminal?
   - Response code (bit 39) specific?

4. Banking transaction flow:
   - POS → acquirer → network → issuer end-to-end test?
   - Hold balance on auth?
   - Release on settlement / reversal?
   - Ledger entry (Topic 10.1) posted?

5. Reversal:
   - 0400 request with original data?
   - Idempotent (retry safe)?
   - Bit 90 original data elements?
   - 24h SLA tracked?

6. EMV:
   - Bit 55 TLV parsing?
   - ARQC cryptogram verify?
   - Magstripe deprecated for new cards?

7. PCI-DSS:
   - PAN encrypted at rest (tokenization)?
   - PAN masked in logs (first 6 + last 4)?
   - Track data NEVER stored?
   - CVV NEVER stored?
   - mTLS transport?

8. Response codes:
   - 51 insufficient funds?
   - 54 expired?
   - 61 limit exceeded?
   - 91 issuer down (timeout vs explicit)?

9. TR specifics:
   - BKM routing?
   - 3D Secure mandatory for CNP?
   - KKB check for credit?

10. Anti-pattern:
    - PAN log plain YOK?
    - Track/CVV store YOK?
    - STAN reuse YOK?
    - EMV verify YOK?
    - Sync POS timeout > 5s YOK?
    - Late reversal YOK?
```

---

## Tamamlama kriterleri

- [ ] jPOS dependency + custom packager
- [ ] 0100/0110 authorization roundtrip
- [ ] 0220 advice settlement
- [ ] 0400/0410 reversal idempotent
- [ ] 0800/0810 echo network management
- [ ] POS simulator + acquirer server + issuer logic
- [ ] EMV bit 55 TLV parse
- [ ] PCI-DSS masking in logs
- [ ] Tokenization (no PAN stored)
- [ ] Ledger posting on capture (Topic 10.1)
- [ ] 8+ integration test
- [ ] Response code map (10+)

---

## Defter notları (10 madde)

1. "ISO 8583 acquirer → network → issuer flow banking ecosystem: ____."
2. "MTI 4-digit (version + class + function + origin) decoding: ____."
3. "Bitmap primary + secondary 64+64 bit field presence: ____."
4. "Critical fields (PAN, Amount, STAN, Response code, Auth ID): ____."
5. "Authorization flow (0100 → 0110) + hold balance + EMV ARQC: ____."
6. "Capture (0220 advice) settlement + ledger posting Topic 10.1: ____."
7. "Reversal (0400) idempotent 24h SLA + bit 90 original data: ____."
8. "PCI-DSS PAN tokenization + log masking + no track/CVV: ____."
9. "TR specifics — BKM switch + Troy domestic + 3D Secure + KKB: ____."
10. "ISO 8583 → ISO 20022 migration banking transitional period: ____."
