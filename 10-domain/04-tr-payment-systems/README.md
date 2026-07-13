# Topic 10.4 — TR Payment Systems: EFT, FAST, BKM, Troy, TCMB

## Hedef

TR finansal ekosistem altyapısını **detaylı** öğrenmek: EFT (TCMB), FAST (instant payment), Havale (intra-bank), BKM (kart switch), Troy (yerli kart şeması), TCMB (merkez bankası) rolleri, settlement window'ları, working hours/holidays, fee structures, IBAN formatı, FAST Karekod, BKM Express, Open Banking BDDK.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1-10.3 bitti
- TR bankacılık temel kullanıcı tecrübesi
- TCMB ne işe yarar duy

---

## Kavramlar

### 1. TR finansal ekosistem — overview

```
[Customers / Corporates]
       ↓
[Bankalar / PSP'ler]  (Garanti, Akbank, İşbank, YKB, vb. + Papara, İninal vb.)
       ↓
[Switch / Settlement]
   ├── TCMB (TR merkez bankası) — EFT/FAST routing + settlement
   ├── BKM (Bankalararası Kart Merkezi) — kart switch
   ├── KKB (Kredi Kayıt Bürosu) — credit score
   ├── TBB (Türkiye Bankalar Birliği) — meslek odası
   ├── BDDK (Bankacılık Düzenleme Denetleme Kurumu) — regulator
   ├── MASAK (Mali Suçları Araştırma Kurulu) — AML
   ├── SPK (Sermaye Piyasası Kurulu) — capital markets
```

### 2. Havale — intra-bank transfer

Same-bank transfer. **No clearing needed** (internal book transfer).

```
Customer A (Maven Bank) → Customer B (Maven Bank), 1000 TL

Banking ops:
- Debit Customer A
- Credit Customer B
- Internal journal entry (Topic 10.1)
- T+0 (instant)
- Generally FREE
```

24/7 available (internal operation).

### 3. EFT — Elektronik Fon Transferi (TCMB)

Inter-bank batched transfer system. TCMB switch.

**EFT-1:** Düşük tutar (limitsiz olarak)
**EFT-2:** Yüksek tutar critical settlement (banking için)

**Working hours:**
- Mon-Fri: 08:30 - 17:30
- Cumartesi: 08:30 - 12:00 (limited)
- Pazar / resmi tatil: KAPALI
- Yeni yıl, bayram tatilleri: bank holiday calendar

**Settlement model:**
- **DNS (Deferred Net Settlement)** — batch + multilateral netting
- T+0 same day if window open
- Cut-off → next business day

**Fee structure:**
- TCMB → bankalara fee (~0.10 TL)
- Bankalar → müşteri fee (~5-15 TL retail, free corporate package)

**Message format:**
- Legacy: TCMB proprietary
- New: ISO 20022 pacs.008 (migration in progress)

**Banking flow:**
```
[Customer A @ Maven Bank]
   ↓ "EFT 1000 TL to Customer B @ Other Bank, IBAN TR..."
[Maven Bank]
   ├── Debit Customer A
   ├── Credit EFT Outgoing Clearing
   └── Send EFT message to TCMB
[TCMB]
   ├── Validate
   ├── Net with other transactions
   └── Settle: debit Maven nostro, credit Other bank nostro
[Other Bank]
   ├── Receive EFT message
   ├── Credit Customer B
   └── Debit EFT Incoming Clearing
```

End-to-end < 30 dakika typical.

### 4. FAST — Fonların Anlık Transferi

TR equivalent of UK Faster Payments / EU SCT Inst.

**Launched:** Ocak 2021 (TCMB)

**Characteristics:**
- **Instant** (10 saniye SLA, 99% < 30 sn)
- **24/7/365** — pazar, gece, tatil all available
- **Per transaction limit:** 70,000 TL (2024)
- **Karekod** — QR code initiation
- **ISO 20022** pacs.008-based

**FAST Karekod:**
- QR code with payment intent
- Static (merchant fixed) or Dynamic (per-payment)
- Scan with mobile app → confirm → instant transfer

```
[Customer A @ Maven Bank] scans Karekod from [Merchant @ Other Bank]

Karekod content (TLV-encoded):
  - Beneficiary IBAN
  - Beneficiary name
  - Amount (optional, dynamic)
  - Reference
  - Expiry timestamp

Maven Bank app:
  - Parse Karekod
  - Show confirm screen
  - User taps "Pay"
  - FAST send

10 saniye içinde:
  - TCMB FAST switch receive
  - Validate (sanctions, limits)
  - Route to Other Bank
  - Other Bank credit Merchant
  - Confirmation back
```

**Settlement:** RTGS-like, gross real-time.

**Banking fee:** 0-3 TL retail, often FREE (regulatory push for adoption).

**Use cases:**
- Bireysel anlık transfer
- E-commerce ödeme (Karekod)
- P2P (mobile-to-mobile)
- Bill payment (ücretsiz fatura)

### 5. IBAN — TR format

TR IBAN: 26 character.

```
TR    32    00100    9    9999987654321098
└──┘  └──┘  └────┘  └─┘  └──────────────┘
 CC   Check  Bank   Reserve  Account
                    digit
```

**Decode example:** `TR320010009999987654321098`

- `TR` — Country
- `32` — Check digit (MOD-97 result)
- `00100` — Bank code (5-digit; e.g. 00010 = Türkiye İş Bankası)
- `0` — Reserved (always 0)
- `9999987654321098` — Account number (16 digits)

**MOD-97 algorithm:**

```java
public class TurkishIbanValidator {
    
    public static boolean isValid(String iban) {
        iban = iban.replaceAll("\\s", "").toUpperCase();
        
        if (iban.length() != 26 || !iban.startsWith("TR")) return false;
        
        // Move first 4 chars to end
        String rearranged = iban.substring(4) + iban.substring(0, 4);
        
        // Replace letters with numbers (A=10, B=11, ..., T=29, R=27)
        StringBuilder numeric = new StringBuilder();
        for (char c : rearranged.toCharArray()) {
            if (Character.isLetter(c)) {
                numeric.append(c - 'A' + 10);
            } else {
                numeric.append(c);
            }
        }
        
        // Mod 97 (big number)
        BigInteger ibanNumber = new BigInteger(numeric.toString());
        return ibanNumber.mod(BigInteger.valueOf(97)).equals(BigInteger.ONE);
    }
    
    public static String extractBankCode(String iban) {
        return iban.substring(4, 9);
    }
    
    public static String extractAccountNumber(String iban) {
        return iban.substring(10);
    }
}
```

**TR bank codes (subset):**
- `00010` — Türkiye İş Bankası
- `00012` — TC Ziraat Bankası
- `00046` — Akbank
- `00062` — Türkiye Garanti Bankası
- `00064` — Türkiye İş Bankası (DUPLICATE — different historical)
- `00067` — Yapı Kredi
- `00111` — QNB Finansbank
- `00134` — Denizbank
- `00203` — Albaraka Türk
- `00205` — Kuveyt Türk
- `00206` — Türkiye Finans
- `00800` — Papara (PSP)
- `00802` — İninal

(Full list: TCMB / TBB website)

### 6. BKM — Bankalararası Kart Merkezi

TR domestic card switch.

**Roles:**
- POS/ATM transactions inter-bank routing
- Card scheme operations (Troy)
- Standardization for issuers/acquirers
- Statistical reporting (BKM reports monthly published)

**BKM Express:** Mobile wallet (deprecated, replaced by FAST Karekod)

**Banking integration:**
- Acquirers connect to BKM switch
- ISO 8583 over leased lines / VPN
- Settlement T+1 (debit Maven, credit Other bank)
- Daily settlement file (NDC reconciliation)

### 7. Troy — Türkiye'nin Ödeme Yöntemi

TR domestic card brand. Visa/MC alternative.

**Launched:** 2016 (BKM owned)

**Coverage:**
- Most TR ATMs
- Most TR POS terminals
- Limited international (some partners)

**Technical:**
- EMV chip standard
- Contactless NFC support
- Compatible with Visa/MC infrastructure (acceptance)

**Banking strategic value:**
- Cost reduction (no Visa/MC fees on TR domestic)
- Sovereignty (regulatory aligned)
- Data residency (TR jurisdiction)

### 8. KKB — Kredi Kayıt Bürosu

TR credit bureau.

**Services:**
- **Findeks Skor** (TR equivalent of FICO)
- Credit history lookup
- Risk reporting
- KYC verification support

**Banking integration:**
- Credit application → KKB query → score returned
- Real-time API
- Cost per query

**FastBureau API example:**

```java
@Service
public class KkbService {
    
    private final WebClient kkbClient;
    
    public CreditScoreResponse getScore(CreditScoreRequest req) {
        return kkbClient.post()
            .uri("/api/v1/findeks-score")
            .header("Authorization", "Bearer " + bankAuthToken)
            .bodyValue(req)
            .retrieve()
            .bodyToMono(CreditScoreResponse.class)
            .block();
    }
}

public record CreditScoreRequest(String tcKimlik, String fullName) {}

public record CreditScoreResponse(
    int score,           // 0-1900
    String category,     // VERY_LOW, LOW, MEDIUM, HIGH, VERY_HIGH
    List<CreditEvent> recentEvents
) {}
```

### 9. TCMB — Türkiye Cumhuriyet Merkez Bankası

Central Bank of TR.

**Roles:**
- Monetary policy
- Currency printing
- Bank reserve management
- **EFT operation**
- **FAST operation**
- Interest rate setting
- FX intervention

**Banking integration:**
- Each bank holds reserve at TCMB
- EFT/FAST settle through TCMB reserves
- Daily reporting requirement (haftalık, aylık)
- Macroprudential measures (reserve req, FX limits)

### 10. BDDK — Bankacılık Düzenleme ve Denetleme Kurumu

Banking regulator. Authority over:
- Bank license, M&A
- Capital adequacy ratio (CAR)
- Risk management standards
- IT regulations (5411 sayılı kanun)
- Open Banking framework (2020 tebliği)

**IT requirements (banking critical):**
- Outsourcing notification
- Datacenter geographical (TR-located)
- Disaster recovery (RTO/RPO)
- IT audit annual
- Penetration test annual
- Incident reporting (significant outages)

**Open Banking (Açık Bankacılık):**
- BDDK 2020 tebliği
- PSD2 benzeri
- Customer consent-based data sharing
- API standardization (BKM coordinator)
- TPP (Third-Party Provider) license

### 11. MASAK — Mali Suçları Araştırma Kurulu

AML (Anti-Money Laundering) regulator.

**Reporting requirements:**
- **STR** (Suspicious Transaction Report) — şüpheli işlem bildirimi
- **CTR** (Currency Transaction Report) — yüksek tutar nakit
- **Sanctions screening** — daily list update

Banking pratiği: 
- Sanctions database (OFAC, EU, MASAK national)
- Real-time screening at transaction
- Customer onboarding KYC + ongoing monitoring

```java
@Service
public class MasakService {
    
    private final SanctionsListClient sanctionsClient;
    
    public ScreeningResult screen(ScreeningRequest req) {
        boolean nameMatch = sanctionsClient.searchByName(req.name());
        boolean tcKimlikMatch = sanctionsClient.searchById(req.tcKimlik());
        boolean addressMatch = sanctionsClient.searchByAddress(req.address());
        
        return new ScreeningResult(
            nameMatch || tcKimlikMatch || addressMatch,
            ...
        );
    }
    
    @Scheduled(cron = "0 0 6 * * *")   // Daily 06:00
    public void refreshSanctionsList() {
        sanctionsClient.downloadLatest();
        sanctionsRepo.replaceAll(sanctionsClient.parse());
    }
}
```

**STR submission:**

```java
@Async
public void submitStr(SuspiciousTransaction tx) {
    StrReport report = StrReport.builder()
        .customerId(tx.customerId())
        .tcKimlik(tx.tcKimlik())
        .transactionDetails(tx.details())
        .suspicionReason(tx.reason())
        .reportingPersonId(tx.reporterId())
        .build();
    
    masakApi.submit(report);
    auditRepo.save(StrAudit.from(report));
}
```

**MASAK fines for non-reporting:** Bank için **çok yüksek**. Audit + automation şart.

### 12. Customer onboarding — KYC TR

**Steps:**
1. **Identity** — TC Kimlik No verification (e-Devlet integration)
2. **Address** — adres beyanı + ikametgah
3. **Source of funds** — gelir belgesi
4. **PEP screening** — Politically Exposed Person
5. **Sanctions screening** — MASAK lists
6. **Risk score** — KKB + bank internal

**MERNİS integration:**

TC Kimlik database verification:
```java
public boolean verifyIdentity(String tcKimlik, String name, String surname, int birthYear) {
    return mernisClient.verify(MernisRequest.builder()
        .tcKimlik(tcKimlik)
        .ad(name)
        .soyad(surname)
        .dogumYili(birthYear)
        .build()).isValid();
}
```

**Banking KYC level:**
- **Basic:** Düşük tutar (< 5000 TL/ay), uzaktan açılış
- **Standard:** Normal limit
- **Enhanced:** Yüksek risk (corporate, PEP), branch visit + document review

### 13. Bank holiday calendar — TR

Banking için **iş günü hesabı kritik**:

```java
@Service
public class TrBusinessCalendar {
    
    private final Set<LocalDate> holidays;
    
    @PostConstruct
    public void init() {
        holidays = Set.of(
            LocalDate.of(2024, 1, 1),     // Yılbaşı
            LocalDate.of(2024, 4, 10),    // Ramazan Bayramı 1. gün
            LocalDate.of(2024, 4, 11),    // Ramazan Bayramı 2. gün
            LocalDate.of(2024, 4, 12),    // Ramazan Bayramı 3. gün
            LocalDate.of(2024, 4, 23),    // Ulusal Egemenlik ve Çocuk Bayramı
            LocalDate.of(2024, 5, 1),     // Emek ve Dayanışma Günü
            LocalDate.of(2024, 5, 19),    // Atatürk'ü Anma, Gençlik ve Spor Bayramı
            LocalDate.of(2024, 6, 17),    // Kurban Bayramı 1. gün
            LocalDate.of(2024, 6, 18),    // Kurban Bayramı 2. gün
            LocalDate.of(2024, 6, 19),    // Kurban Bayramı 3. gün
            LocalDate.of(2024, 6, 20),    // Kurban Bayramı 4. gün
            LocalDate.of(2024, 7, 15),    // Demokrasi ve Milli Birlik Günü
            LocalDate.of(2024, 8, 30),    // Zafer Bayramı
            LocalDate.of(2024, 10, 29)    // Cumhuriyet Bayramı
        );
    }
    
    public boolean isBusinessDay(LocalDate date) {
        DayOfWeek dow = date.getDayOfWeek();
        if (dow == DayOfWeek.SUNDAY) return false;
        if (dow == DayOfWeek.SATURDAY) return isSaturdayActive(date);
        return !holidays.contains(date);
    }
    
    public LocalDate nextBusinessDay(LocalDate date) {
        LocalDate next = date.plusDays(1);
        while (!isBusinessDay(next)) next = next.plusDays(1);
        return next;
    }
    
    public LocalDate addBusinessDays(LocalDate date, int days) {
        LocalDate result = date;
        for (int i = 0; i < days; i++) {
            result = nextBusinessDay(result);
        }
        return result;
    }
}
```

EFT cutoff: 17:30 + cumartesi öğlen. Cutoff sonrası → next business day.

FAST: 24/7 — calendar etkilemez.

### 14. Banking — TR anti-pattern'leri

**Anti-pattern 1: IBAN validation app-level yok**

Customer wrong IBAN → EFT fail + refund SLA. Pre-send validate.

**Anti-pattern 2: Working hours kontrol yok**

EFT 17:35'te ilk gönderilince → "neden gitmiyor?" customer support call'u. UI hour check.

**Anti-pattern 3: Holiday calendar hardcoded geçmiş yıl**

Yıl dönüm zamanı update unutuldu → wrong business day. Resmi gazete kaynaklı dynamic update.

**Anti-pattern 4: MASAK screening sync block**

Customer onboarding 30 saniye bekler. Async + later validation.

**Anti-pattern 5: KKB query every login**

Cost + KKB API rate limit. Cache (TTL günler).

**Anti-pattern 6: FAST limit ignored**

70k TL üstü FAST → reject. UI pre-validate + suggest EFT.

**Anti-pattern 7: BKM card transactions sync ledger update**

POS terminal timeout. Async ledger + reconcile.

**Anti-pattern 8: Bank holiday only Christian calendar**

Kurban / Ramazan bayramı resmi tatil. TR calendar mandatory.

**Anti-pattern 9: TC Kimlik MERNIS verification yok**

Fake identity onboarding. MERNIS şart.

**Anti-pattern 10: Open Banking BDDK uyumsuz custom API**

3rd party integrations standard BDDK API'leri kullanmalı. Custom = uyumsuz.

---

## Önemli olabilecek araştırma kaynakları

- TCMB web (tcmb.gov.tr) — EFT/FAST docs
- BKM web (bkm.com.tr) — kart switch + istatistik
- KKB API documentation
- BDDK web (bddk.org.tr) — bankacılık mevzuatı
- MASAK web (masak.hmb.gov.tr) — AML rehberi
- TBB Türkiye Bankalar Birliği
- 5411 sayılı Bankacılık Kanunu
- 6493 sayılı Ödeme Hizmetleri Kanunu

---

## Mini task'ler

### Task 10.4.1 — TR IBAN validator (45 dk)

MOD-97 + TR format check. Bank code extract. Test 10+ real IBAN.

### Task 10.4.2 — Bank code lookup table (30 dk)

`bank_code` table (00010 → Türkiye İş Bankası ...). API endpoint `/banks/{code}`.

### Task 10.4.3 — Business calendar service (60 dk)

TR holidays 2024-2025. isBusinessDay, nextBusinessDay, addBusinessDays. Test.

### Task 10.4.4 — EFT vs FAST router (60 dk)

Customer request → amount check + working hours check + cost compare → suggest EFT or FAST.

### Task 10.4.5 — FAST Karekod parser (45 dk)

TLV format. QR content → IBAN + name + amount + reference.

### Task 10.4.6 — MERNIS verification mock (45 dk)

TC Kimlik + name + surname + birthYear → MERNIS API call. Mock service.

### Task 10.4.7 — KKB Findeks score (45 dk)

Mock KKB API. Customer onboarding flow. Score-based limit assignment.

### Task 10.4.8 — MASAK sanctions screening (60 dk)

Sanctions list (OFAC sample). Name + ID match. Real-time screening per transfer.

### Task 10.4.9 — STR (suspicious transaction) detection + report (60 dk)

Rule: >50k cash deposit, fragmenting (smurfing) pattern. Auto STR draft.

### Task 10.4.10 — Open Banking consent flow (60 dk)

3rd party app → BDDK-compliant consent screen → access token (Topic 8.4) → API call.

---

## Test yazma rehberi

```java
@ParameterizedTest
@CsvSource({
    "TR320010009999987654321098, true",
    "TR320010009999987654321099, false",     // wrong checksum
    "TR3200100099998,            false",      // too short
    "DE89370400440532013000,     false"       // not TR
})
void shouldValidateIban(String iban, boolean expected) {
    assertThat(TurkishIbanValidator.isValid(iban)).isEqualTo(expected);
}

@Test
void shouldExtractBankCode() {
    String iban = "TR320010009999987654321098";
    String bankCode = TurkishIbanValidator.extractBankCode(iban);
    
    assertThat(bankCode).isEqualTo("00100");
}

@ParameterizedTest
@CsvSource({
    "2024-01-01, false",   // Yılbaşı
    "2024-01-02, true",    // Salı normal
    "2024-04-23, false",   // Ulusal egemenlik
    "2024-06-15, false",   // Cumartesi (assume off)
    "2024-06-17, false"    // Kurban Bayramı
})
void shouldRecognizeHolidays(LocalDate date, boolean isBusinessDay) {
    assertThat(calendar.isBusinessDay(date)).isEqualTo(isBusinessDay);
}

@Test
void shouldComputeT2BusinessDay() {
    LocalDate friday = LocalDate.of(2024, 5, 10);
    LocalDate t2 = calendar.addBusinessDays(friday, 2);
    
    assertThat(t2).isEqualTo(LocalDate.of(2024, 5, 14));   // Tuesday
}

@Test
void shouldSuggestFastDuringNonBusinessHours() {
    // Friday 18:00, EFT closed
    Clock clock = Clock.fixed(
        ZonedDateTime.of(2024, 5, 10, 18, 0, 0, 0, ZoneId.of("Europe/Istanbul")).toInstant(),
        ZoneId.of("Europe/Istanbul"));
    
    TransferRouter router = new TransferRouter(calendar, clock);
    Route route = router.suggest(amount: new BigDecimal("1000"), iban: validIban);
    
    assertThat(route.system()).isEqualTo("FAST");
}

@Test
void shouldRejectFastAboveLimit() {
    TransferRouter router = new TransferRouter(calendar, clock);
    Route route = router.suggest(amount: new BigDecimal("100000"), iban: validIban);
    
    assertThat(route.system()).isEqualTo("EFT");   // > 70k FAST limit
    assertThat(route.warning()).contains("FAST limit aşıldı");
}

@Test
void shouldBlockSanctionedRecipient() {
    sanctionsList.add(new SanctionedEntity("12345678901", "Sanctioned Person", null));
    
    TransferRequest req = TransferRequest.builder()
        .recipientName("Sanctioned Person")
        .recipientTcKimlik("12345678901")
        .build();
    
    assertThatThrownBy(() -> transferService.initiate(req))
        .isInstanceOf(SanctionsScreeningException.class);
}
```

---

## Claude-verify prompt

```
TR payment systems implementation'ımı banking-grade kriterlere göre değerlendir:

1. IBAN:
   - MOD-97 validation?
   - TR format (26 char) check?
   - Bank code extract + lookup?
   - Real-time validation (pre-send)?

2. Business calendar:
   - TR holidays (resmi tatil) güncel?
   - isBusinessDay / addBusinessDays utility?
   - EFT cutoff (17:30) check?
   - FAST 24/7 awareness?

3. EFT vs FAST routing:
   - Working hours check?
   - Amount limit (FAST 70k) check?
   - Cost suggestion?
   - User UX (banking app guidance)?

4. FAST Karekod:
   - TLV parser?
   - Static + dynamic QR support?
   - Expiry check?

5. KYC integration:
   - MERNIS TC Kimlik verification?
   - KKB Findeks score?
   - Cache strategy (TTL günler)?
   - PEP screening?

6. MASAK / sanctions:
   - Sanctions list daily refresh?
   - Real-time screening at transfer?
   - STR detection rules (smurfing, large cash)?
   - STR report submit API?

7. BDDK:
   - Open Banking 2020 tebliği uyumlu?
   - IT regulations (DR, audit, pentest)?
   - Outsourcing notification?

8. Ledger integration:
   - EFT clearing account (Topic 10.1)?
   - FAST clearing real-time?
   - BKM card settlement T+1?

9. Bank holiday handling:
   - Resmi tatil dynamic update?
   - Yarım gün cumartesi?
   - Bayram tatilleri?

10. Anti-pattern:
    - IBAN validation skip YOK?
    - Working hours ignore YOK?
    - Hardcoded geçmiş yıl holidays YOK?
    - MASAK sync block YOK?
    - KKB query per request YOK?
    - FAST limit ignore YOK?
    - TC Kimlik MERNIS yok YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] TR IBAN validator + bank code lookup
- [ ] TR business calendar service (holidays)
- [ ] EFT vs FAST router
- [ ] FAST Karekod TLV parser
- [ ] MERNIS verification mock
- [ ] KKB Findeks score + caching
- [ ] MASAK sanctions screening (daily refresh + screening)
- [ ] STR detection + report submit
- [ ] Open Banking consent flow
- [ ] BDDK IT regulatory checklist (DR, audit)
- [ ] 8+ integration test (IBAN, holiday, routing, sanctions)

---

## Defter notları (10 madde)

1. "TR finansal ekosistem (TCMB, BKM, KKB, BDDK, MASAK) rol matrisi: ____."
2. "EFT vs FAST trade-off (working hours, amount, fee, settlement model): ____."
3. "Havale intra-bank instant internal book transfer: ____."
4. "TR IBAN format MOD-97 + 26 char + bank code extract: ____."
5. "FAST Karekod TLV format + static/dynamic QR + 10 sn SLA: ____."
6. "BKM kart switch + Troy yerli kart şeması banking strategic: ____."
7. "TCMB EFT/FAST operation + reserve settlement banking nostro: ____."
8. "BDDK 5411 + Open Banking 2020 + IT regulations (DR, audit, pentest): ____."
9. "MASAK sanctions screening + STR (smurfing) + daily refresh banking pratik: ____."
10. "TR business calendar (holidays, bayram) banking ops kritik: ____."
