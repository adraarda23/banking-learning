# Topic 10.6 — Regulatory: BDDK, MASAK, KVKK, PCI-DSS, KKB, GDPR

## Hedef

TR + uluslararası banking regulatory landscape'i implementation perspektifinden öğrenmek: BDDK (bankacılık), MASAK (AML/CFT), KVKK (kişisel veri), PCI-DSS (kart), KKB (kredi bürosu), GDPR (AB veri), 5411 sayılı Kanun, 6493 Ödeme Hizmetleri Kanunu, basel uyumu, AML compliance program, audit trail, retention, reporting, incident notification.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 10.1-10.5 bitti
- Phase 8 security bitti
- TR mevzuat genel kavramı

---

## Kavramlar

### 1. Regulatory landscape — TR + global

```
Bank
├── TR Authorities
│   ├── BDDK     — Bankacılık Düzenleme ve Denetleme Kurumu
│   ├── TCMB     — Türkiye Cumhuriyet Merkez Bankası
│   ├── MASAK    — Mali Suçları Araştırma Kurulu
│   ├── KVKK     — Kişisel Verileri Koruma Kurumu
│   ├── SPK      — Sermaye Piyasası Kurulu
│   ├── HMB      — Hazine ve Maliye Bakanlığı
│   └── TBB      — Türkiye Bankalar Birliği (oda)
│
├── Global
│   ├── BCBS     — Basel Committee (capital, liquidity)
│   ├── FATF     — Financial Action Task Force (AML)
│   ├── PCI Council — kart standartları
│   ├── ISO      — uluslararası standart organ
│   └── SWIFT    — message standard + sanctions
│
├── Industry self-regulatory
│   ├── BKM      — kart switch
│   ├── KKB      — kredi bürosu
│   └── BÜMED    — internal industry alliances
```

### 2. BDDK — bankacılık regulator

**Authority basis:** 5411 sayılı Bankacılık Kanunu (2005).

**Key responsibilities:**
- License banks, M&A approval
- Capital adequacy (CAR — Sermaye Yeterlilik Oranı, Basel III)
- Liquidity (LCR, NSFR)
- Concentration limits (single counterparty)
- IT regulations (Bilgi Sistemleri Yönetimi Tebliği)
- Outsourcing notification
- Loan classification (gruplar: 1-2-3-4-5)
- Provision rules (specific + general)
- Stress test (annual)
- Open Banking (2020 tebliği)
- Consumer protection

**Banking IT implementation impact:**

```
BDDK Bilgi Sistemleri Yönetimi Tebliği (2020 güncellemesi):
- Data residency: TR-located primary
- Disaster recovery: hot/warm site + RTO/RPO targets
- Penetration test: yıllık + after major change
- IT audit: yıllık external + internal continuous
- Incident reporting: 24h after detection (significant)
- Change management documented
- Access control + segregation of duties
- Outsourcing notification + contract terms
- Backup + restore tested quarterly
- Capacity planning documented
```

**Backend implementation checklist:**

```java
// Audit log retention 10 yıl (BDDK)
@Entity
public class AuditEvent {
    @Id
    private Long id;
    
    @Column(nullable = false, updatable = false)
    private Instant occurredAt;
    
    @Column(nullable = false)
    private UUID userId;
    
    @Column(nullable = false)
    private String action;
    
    @Column(columnDefinition = "jsonb")
    private String details;
    
    // ... immutable, hash chain (Topic 8.7)
}

@Configuration
public class BankingRetentionConfig {
    @Bean
    public RetentionPolicy auditRetention() {
        return RetentionPolicy.builder()
            .source("audit_event")
            .retention(Period.ofYears(10))
            .archiveAfter(Period.ofDays(90))
            .tier1Days(7)
            .tier2Days(30)
            .tier3Days(90)
            .archiveTier("s3-glacier")
            .build();
    }
}
```

### 3. MASAK — AML/CFT

**Authority basis:** 5549 sayılı Suç Gelirlerinin Aklanmasının Önlenmesi Hakkında Kanun.

**Compliance program requirements:**
- Compliance officer (atanmış)
- Risk assessment annual
- Policy & procedure documented
- Customer Due Diligence (CDD) at onboarding
- Enhanced Due Diligence (EDD) for high-risk
- Ongoing monitoring
- Sanctions screening
- Reporting (STR, CTR)
- Training annual
- Audit (internal + external)

**STR (Şüpheli İşlem Bildirimi):**

Trigger örnekleri:
- Smurfing (structuring) — Birden fazla küçük tutar transfer ardarda
- Round-number transactions
- Geographically anomalous
- PEP large transactions
- Inconsistent with customer profile
- Anonymous beneficiary
- Cash-heavy
- Sanctions-listed counterparty

**Implementation:**

```java
@Service
public class TransactionMonitoringService {
    
    private final List<MonitoringRule> rules;
    
    @KafkaListener(topics = "transactions")
    public void onTransaction(TransactionEvent event) {
        for (MonitoringRule rule : rules) {
            RuleResult result = rule.evaluate(event);
            if (result.isMatch()) {
                createAlert(event, rule, result);
            }
        }
    }
    
    private void createAlert(TransactionEvent event, MonitoringRule rule, RuleResult result) {
        ComplianceAlert alert = ComplianceAlert.builder()
            .alertId(UUID.randomUUID())
            .transactionId(event.getId())
            .customerId(event.getCustomerId())
            .rule(rule.getName())
            .severity(result.getSeverity())
            .reason(result.getReason())
            .status("PENDING_REVIEW")
            .createdAt(Instant.now())
            .build();
        alertRepo.save(alert);
        
        if (result.getSeverity() == Severity.CRITICAL) {
            notifyComplianceOps(alert);
        }
    }
}

// Rule examples
public class SmurfingDetectionRule implements MonitoringRule {
    
    public RuleResult evaluate(TransactionEvent event) {
        // Count transactions for customer last 24h
        List<TransactionEvent> recent = txRepo.findByCustomerSince(
            event.getCustomerId(), 
            event.getTimestamp().minus(24, ChronoUnit.HOURS));
        
        BigDecimal totalAmount = recent.stream()
            .map(TransactionEvent::getAmount)
            .reduce(ZERO, BigDecimal::add);
        
        long countBelowThreshold = recent.stream()
            .filter(tx -> tx.getAmount().compareTo(new BigDecimal("10000")) < 0
                       && tx.getAmount().compareTo(new BigDecimal("9000")) > 0)
            .count();
        
        if (countBelowThreshold >= 5 && totalAmount.compareTo(new BigDecimal("40000")) > 0) {
            return RuleResult.match(
                Severity.HIGH,
                "Potential smurfing: " + countBelowThreshold + " transactions just below threshold");
        }
        return RuleResult.noMatch();
    }
}

public class LargeCashRule implements MonitoringRule {
    public RuleResult evaluate(TransactionEvent event) {
        if ("CASH_DEPOSIT".equals(event.getType()) 
            && event.getAmount().compareTo(new BigDecimal("75000")) > 0) {
            return RuleResult.match(Severity.MEDIUM, "Large cash deposit");
        }
        return RuleResult.noMatch();
    }
}

public class SanctionedCounterpartyRule implements MonitoringRule {
    public RuleResult evaluate(TransactionEvent event) {
        if (sanctionsService.isOnList(event.getRecipientName(), event.getRecipientCountry())) {
            return RuleResult.match(Severity.CRITICAL, "Sanctioned counterparty");
        }
        return RuleResult.noMatch();
    }
}
```

**STR submission:**

MASAK e-bildirim sistemi:

```java
@Async
public void submitStr(ComplianceAlert alert) {
    StrReport report = buildStrReport(alert);
    
    StrSubmissionResponse response = masakClient.submit(report);
    
    alert.setStrSubmittedAt(Instant.now());
    alert.setStrReportId(response.getReportId());
    alert.setStatus("STR_FILED");
    alertRepo.save(alert);
    
    auditService.log("STR_FILED", alert);
}
```

### 4. KVKK — kişisel veri

**Authority basis:** 6698 sayılı Kişisel Verilerin Korunması Kanunu (2016).

**Banking PII categories:**
- TC kimlik no
- Ad-soyad
- Adres
- Telefon, e-posta
- Doğum tarihi
- Mesleki bilgi
- Mali durum
- Sağlık (kredi sigortası için)
- Biometric (face/fingerprint)

**KVKK requirements implementation:**

1. **Açık rıza (explicit consent)** — Her veri işleme amacı için ayrı
2. **Aydınlatma yükümlülüğü** — Privacy notice gösterimi
3. **Veri minimizasyonu** — En az gerekli kadar
4. **Saklama süresi** — Amaca uygun, sonra silinmeli
5. **Veri güvenliği** — Encryption, access control
6. **Veri sorumlusu sicili** — VERBİS kayıt
7. **Veri ihlali bildirimi** — 72 saat KVKK + ilgililere
8. **Veri sahibi hakları** — Erişim, düzeltme, silme, taşıma
9. **Yurtdışı aktarım kısıtı** — Yeterli koruma + onay

**Implementation:**

```java
@Entity
public class CustomerConsent {
    @Id private UUID id;
    private UUID customerId;
    private String purpose;            // "marketing", "credit_check", "third_party_share", ...
    private boolean granted;
    private Instant grantedAt;
    private Instant revokedAt;
    private String legalBasis;          // KVKK Madde 5/2 (sözleşme), Madde 5/2 (kanun), Madde 6 (açık rıza)
}

@Service
public class ConsentService {
    
    public boolean hasConsent(UUID customerId, String purpose) {
        return consentRepo.findActiveConsent(customerId, purpose).isPresent();
    }
    
    public void revoke(UUID customerId, String purpose) {
        consentRepo.findActiveConsent(customerId, purpose)
            .ifPresent(c -> {
                c.setRevokedAt(Instant.now());
                consentRepo.save(c);
            });
    }
    
    public void recordConsent(UUID customerId, String purpose, String legalBasis) {
        consentRepo.save(CustomerConsent.builder()
            .id(UUID.randomUUID())
            .customerId(customerId)
            .purpose(purpose)
            .granted(true)
            .grantedAt(Instant.now())
            .legalBasis(legalBasis)
            .build());
        
        auditService.log("CONSENT_GRANTED", customerId, purpose);
    }
}

@Aspect
@Component
public class ConsentEnforcementAspect {
    
    @Around("@annotation(RequiresConsent)")
    public Object enforce(ProceedingJoinPoint pjp) throws Throwable {
        RequiresConsent annotation = ... ;
        UUID customerId = extractCustomerId(pjp);
        
        if (!consentService.hasConsent(customerId, annotation.purpose())) {
            throw new ConsentRequiredException(annotation.purpose());
        }
        
        return pjp.proceed();
    }
}

@Service
public class MarketingService {
    
    @RequiresConsent(purpose = "marketing")
    public void sendCampaign(UUID customerId, Campaign campaign) {
        // ...
    }
}
```

**Right to be forgotten (silinme hakkı):**

```java
@Transactional
public void forgetMe(UUID customerId) {
    // Crypto-shred (Topic 8.6) — destroy encryption key
    encryptionService.destroyCustomerKey(customerId);
    
    // Mark customer record (soft-delete'den farkı encrypted unreadable artık)
    Customer c = customerRepo.findById(customerId).orElseThrow();
    c.setRequestedForgottenAt(Instant.now());
    c.setStatus("FORGOTTEN");
    customerRepo.save(c);
    
    // Schedule retention compliance (some data must keep 10 yıl - BDDK)
    forgetScheduler.scheduleResidualPurge(customerId, Period.ofYears(10));
    
    auditService.log("CUSTOMER_FORGOTTEN", customerId);
}
```

**Trade-off:** KVKK silinme hakkı vs BDDK 10 yıl audit retention. Çözüm: anonymize, sadece functional data sil; audit trail pseudonymized.

### 5. PCI-DSS — kart standartları

Topic 8.6'da temel. Burada compliance perspektifinden:

**Levels (TR bank için Level 1 — large issuer/acquirer):**
- Level 1: > 6M kart işlemi/yıl
- Annual on-site QSA assessment
- Quarterly ASV scan
- Penetration test annual

**Banking compliance requirements:**

1. **Build secure network** — firewall, no default passwords
2. **Protect cardholder data** — PAN encryption, tokenization, no CVV store
3. **Vulnerability management** — patch, AV
4. **Strong access control** — RBAC, MFA, physical access logs
5. **Network monitoring** — IDS/IPS, FIM
6. **Security policy** — documented, reviewed

**Scope reduction via tokenization:**
- PAN stored only in payment service (PCI-DSS scope)
- Other services use token
- 95% backend out of PCI scope

### 6. KKB — kredi bürosu

**Findeks Skor (300-1900):**
- Banking onboarding ve kredi başvurusu için ZORUNLU sorgu
- Score-based pricing + limit
- API real-time

**Other KKB services:**
- KRS (Kredi Referans Sistemi) — past credit behavior
- Çek raporu — bounced check history
- GeoData — adres doğrulama
- KKB Sandbox / API marketplace

**Banking integration:**

```java
@Service
public class KkbService {
    
    @Cacheable(value = "kkb-score", key = "#tcKimlik", unless = "#result == null")
    public CreditScoreResponse getScore(String tcKimlik) {
        return kkbClient.post()
            .uri("/findeks/v3/score")
            .bodyValue(Map.of("tcKimlik", tcKimlik))
            .retrieve()
            .bodyToMono(CreditScoreResponse.class)
            .block();
    }
}

@Configuration
public class KkbCacheConfig {
    @Bean
    public CacheManager kkbCacheManager() {
        return CaffeineCacheManager.builder()
            .caffeine(Caffeine.newBuilder()
                .maximumSize(100_000)
                .expireAfterWrite(Duration.ofDays(7))   // Score 7 gün cache
                .recordStats())
            .build();
    }
}
```

**Cost optimization:** KKB query başına ücret. Caching + business rule (sadece kredi başvurusunda + büyük tutar kart başvurusu).

### 7. GDPR — AB veri koruma

TR bank için relevant olduğu durumlar:
- EU customer (vatandaş, mukim)
- EU operasyon (şube, leştirme)
- EU data residency (cross-border bank Türkiye'de hosted)

GDPR vs KVKK paralel (KVKK aslında GDPR esinli):
- Lawful basis
- Consent
- Data subject rights
- DPO appointment
- Breach notification 72h
- Cross-border transfer (TRIA — AB yeterli koruma değerlendirmesi)

### 8. Basel III — capital + liquidity

BDDK Basel III uyguluyor.

**Banking metrics:**
- **CAR (Sermaye Yeterlilik Oranı):** Tier 1 + Tier 2 / RWA ≥ 12% (TR minimum)
- **LCR (Liquidity Coverage Ratio):** HQLA / Net cash outflows (30-day) ≥ 100%
- **NSFR (Net Stable Funding Ratio):** Available stable funding / Required stable funding ≥ 100%
- **Leverage Ratio:** Tier 1 / Exposure ≥ 3%

Banking IT implication: data quality + EOD calculation + regulatory reporting.

```sql
-- Daily LCR calculation EOD job
SELECT 
    SUM(CASE WHEN asset_type IN ('cash', 'central_bank_reserve') THEN amount * 1.0
             WHEN asset_type = 'government_bond' THEN amount * 1.0
             WHEN asset_type = 'corporate_bond_aa' THEN amount * 0.85
             ELSE 0 END) AS hqla,
    SUM(CASE WHEN outflow_type = 'retail_deposit_stable' THEN balance * 0.05
             WHEN outflow_type = 'retail_deposit_less_stable' THEN balance * 0.10
             WHEN outflow_type = 'wholesale' THEN balance * 0.40
             ELSE 0 END) AS net_outflows
FROM regulatory_position
WHERE as_of_date = CURRENT_DATE;
```

Banking için **regulatory reporting service** ayrı microservice yaygın.

### 9. Reporting — BDDK, MASAK, TCMB

**BDDK günlük/haftalık/aylık raporlar:**
- Bilanço
- Sermaye yeterliliği
- Kredi-mevduat oranı
- Liquidity ratios

**MASAK aylık raporlar + ad-hoc STR/CTR.**

**TCMB raporları:**
- Aktif/pasif yapı
- Para arzı verisi
- FX position
- Interest rate

Banking IT: regulatory reporting service. Format: XBRL, XML, CSV (regulatory specific).

### 10. Incident notification

**BDDK:** 24 saat içinde significant outage / security incident.
**KVKK:** 72 saat içinde veri ihlali.
**MASAK:** STR → 10 iş günü.

Implementation: incident management system + automated alerting + manual review.

### 11. Audit trail — banking critical

Compliance audit için trail:
- WHO (user ID + IP)
- WHAT (action + resource)
- WHEN (timestamp)
- HOW (channel: mobile/web/branch/ATM)
- RESULT (success/fail + reason)
- PRE/POST state (for data changes)

```sql
CREATE TABLE compliance_audit (
    id BIGSERIAL PRIMARY KEY,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    user_id UUID,
    customer_id UUID,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id VARCHAR(100),
    channel VARCHAR(20),
    ip_address INET,
    user_agent TEXT,
    result VARCHAR(20),
    reason TEXT,
    pre_state JSONB,
    post_state JSONB,
    trace_id VARCHAR(100),
    hash_prev CHAR(64),
    hash_curr CHAR(64)
);
```

**Required audit events (BDDK + MASAK):**
- Customer onboarding + KYC steps
- Login (success + fail + lockout)
- Logout
- Password change, MFA enrollment/disable
- Profile changes (KVKK)
- Permission/role changes
- Transfer initiated/completed/failed
- Card operations (block, unblock, limit change)
- Loan application + approval/decline
- Account opening/closing
- Large transactions (BDDK threshold)
- Cross-border transfers
- Sanctions hit
- STR generated
- Admin actions (any)
- Configuration changes
- Data export/extract (KVKK)

### 12. Banking — regulatory anti-pattern'leri

**Anti-pattern 1: Audit log mutable**

UPDATE/DELETE possible → tamper. Immutable + hash chain.

**Anti-pattern 2: Retention insufficient**

Application log 30 gün only. Audit/compliance 5-10 yıl gerekli.

**Anti-pattern 3: PII export uncontrolled**

Bulk customer data export → KVKK ihlali. Approval workflow + audit.

**Anti-pattern 4: Sanctions screening daily batch only**

Real-time gerek. Daily list refresh + per-transaction screening.

**Anti-pattern 5: STR detection rules statik**

Yeni typoloji typology updates kaçırılır. Rule engine + tunable.

**Anti-pattern 6: Customer consent implicit**

"Site kullanımı = onay" — KVKK ihlali. Explicit checkboxes per purpose.

**Anti-pattern 7: Cross-border veri aktarımı uncontrolled**

Cloud provider AB → TR'den AB'ye aktarım onay gerekli. Hosted in TR strategy banking için yaygın.

**Anti-pattern 8: Incident notification manual**

Manual süre kaçar. Automated alerting + workflow.

**Anti-pattern 9: Compliance officer notification yok**

Material decision compliance officer dahil edilmez. Workflow approval gate.

**Anti-pattern 10: Penetration test annual yok**

BDDK requirement. Annual + after major change.

---

## Önemli olabilecek araştırma kaynakları

- 5411 sayılı Bankacılık Kanunu
- 6698 sayılı KVKK
- 5549 sayılı MASAK Kanunu
- 6493 Ödeme Hizmetleri Kanunu
- BDDK Bilgi Sistemleri Yönetimi Tebliği
- MASAK Tedbirler Yönetmeliği
- Basel III framework
- PCI-DSS v4.0
- GDPR full text
- BDDK Open Banking Tebliği (2020)
- TCMB Ödeme Sistemleri Genelgesi

---

## Mini task'ler

### Task 10.6.1 — Audit table + hash chain (60 dk)

Topic 8.7'den. Compliance audit dedicated table. Hash chain verify scheduled.

### Task 10.6.2 — Consent management (60 dk)

CustomerConsent entity + service. `@RequiresConsent` aspect.

### Task 10.6.3 — Right to be forgotten (45 dk)

Crypto-shred via Topic 8.6 KMS. Functional data delete. Audit retain pseudonymized.

### Task 10.6.4 — MASAK rule engine (90 dk)

Smurfing, large cash, sanctions counterparty. Real-time on transaction event. Alert table.

### Task 10.6.5 — STR submit workflow (45 dk)

Alert review → compliance officer approve → STR submit MASAK API (mock). Audit.

### Task 10.6.6 — Sanctions list daily refresh (45 dk)

OFAC + national list download. Replace table. Per-transfer screening.

### Task 10.6.7 — KVKK data export (60 dk)

Customer right to access. JSON export. PDF generate. Audit data export action.

### Task 10.6.8 — Regulatory reporting (60 dk)

Daily LCR + CAR computation. XML output (BDDK format). Scheduled.

### Task 10.6.9 — Incident notification workflow (45 dk)

Detected incident → BDDK 24h timer → KVKK 72h timer → automated escalation.

### Task 10.6.10 — Compliance dashboard (60 dk)

Grafana: open alerts, pending STRs, sanctions hits, consent grants/revokes.

---

## Test yazma rehberi

```java
@Test
void shouldDetectSmurfing() {
    UUID customerId = UUID.randomUUID();
    for (int i = 0; i < 5; i++) {
        txService.send(TransactionEvent.builder()
            .customerId(customerId)
            .amount(new BigDecimal("9500"))
            .timestamp(Instant.now().minus(i, ChronoUnit.HOURS))
            .build());
    }
    
    awaitForAlerts();
    
    List<ComplianceAlert> alerts = alertRepo.findByCustomer(customerId);
    assertThat(alerts).hasSize(1);
    assertThat(alerts.get(0).getRule()).isEqualTo("SmurfingDetection");
}

@Test
void shouldEnforceConsentBeforeMarketingCall() {
    UUID customerId = UUID.randomUUID();
    
    assertThatThrownBy(() -> marketingService.sendCampaign(customerId, campaign))
        .isInstanceOf(ConsentRequiredException.class);
    
    consentService.recordConsent(customerId, "marketing", "KVKK Madde 6");
    marketingService.sendCampaign(customerId, campaign);   // No exception
}

@Test
void shouldRevokeConsent() {
    UUID customerId = UUID.randomUUID();
    consentService.recordConsent(customerId, "marketing", "KVKK Madde 6");
    consentService.revoke(customerId, "marketing");
    
    assertThatThrownBy(() -> marketingService.sendCampaign(customerId, campaign))
        .isInstanceOf(ConsentRequiredException.class);
}

@Test
@Transactional
void rightToBeForgottenCryptoShreds() {
    UUID customerId = createCustomer();
    Customer before = customerRepo.findById(customerId).orElseThrow();
    String tcBefore = before.getTcKimlik();
    
    forgetService.forgetMe(customerId);
    
    em.clear();
    Customer after = customerRepo.findById(customerId).orElseThrow();
    
    // tcKimlik unreadable after crypto-shred
    assertThatThrownBy(after::getTcKimlik)
        .isInstanceOf(EncryptionException.class);
    
    assertThat(after.getStatus()).isEqualTo("FORGOTTEN");
}

@Test
void auditChainShouldDetectTamper() {
    auditService.log(userId, "LOGIN", null);
    auditService.log(userId, "TRANSFER", txId);
    auditService.log(userId, "LOGOUT", null);
    
    assertThat(auditService.verifyChain()).isTrue();
    
    // Tamper
    em.createNativeQuery("UPDATE compliance_audit SET action = 'FAKE' WHERE id = 2")
        .executeUpdate();
    
    assertThat(auditService.verifyChain()).isFalse();
}

@Test
void shouldFlagSanctionedRecipient() {
    sanctionsService.addToList(new SanctionedEntity("12345678901", "Sanctioned Person"));
    
    RuleResult result = sanctionsRule.evaluate(TransactionEvent.builder()
        .recipientName("Sanctioned Person")
        .recipientTcKimlik("12345678901")
        .build());
    
    assertThat(result.isMatch()).isTrue();
    assertThat(result.getSeverity()).isEqualTo(Severity.CRITICAL);
}
```

---

## Claude-verify prompt

```
Regulatory implementation'ımı banking-grade kriterlere göre değerlendir:

1. Audit trail:
   - Compliance audit table separate?
   - Immutable (no UPDATE/DELETE)?
   - Hash chain tamper detect?
   - 14+ audit event types (login, transfer, MFA, admin, ...)?
   - Retention 5-10 yıl (BDDK)?

2. MASAK / AML:
   - Real-time transaction monitoring?
   - 3+ rule (smurfing, large cash, sanctions)?
   - Sanctions daily refresh?
   - STR workflow (alert → review → submit)?
   - Compliance officer approval gate?

3. KVKK / GDPR:
   - Consent management (granular per purpose)?
   - @RequiresConsent aspect?
   - Right to be forgotten (crypto-shred)?
   - Data export (right to access)?
   - 72h breach notification workflow?
   - VERBİS kayıt?

4. PCI-DSS:
   - PAN tokenization (Topic 8.6)?
   - No CVV store?
   - Network segmentation?
   - Quarterly ASV scan automated?
   - Annual pentest scheduled?

5. KKB:
   - Findeks score integration?
   - Cache (7 days TTL)?
   - Score-based limit?

6. BDDK regulatory:
   - Daily LCR computation?
   - CAR calculation EOD?
   - XML/XBRL output format?
   - Submission workflow?

7. Basel III:
   - LCR + NSFR + CAR + Leverage tracked?
   - HQLA classification?

8. Incident notification:
   - 24h BDDK timer?
   - 72h KVKK timer?
   - Automated escalation?

9. Cross-border:
   - Data residency TR?
   - Cross-border consent?
   - Cloud provider AB → TR aktarım onayı?

10. Anti-pattern:
    - Audit mutable YOK?
    - Retention insufficient YOK?
    - PII export uncontrolled YOK?
    - Sanctions daily-only YOK?
    - Consent implicit YOK?
    - Pentest annual yok YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Compliance audit table + hash chain
- [ ] Consent management + @RequiresConsent
- [ ] Right to be forgotten (crypto-shred)
- [ ] MASAK rule engine (3+ rule)
- [ ] STR submission workflow
- [ ] Sanctions list daily refresh + screening
- [ ] KVKK data export
- [ ] Regulatory reporting (LCR + CAR XML)
- [ ] Incident notification automated (24h/72h)
- [ ] Compliance dashboard Grafana
- [ ] 10+ integration test
- [ ] Audit chain verify

---

## Defter notları (10 madde)

1. "TR regulatory landscape (BDDK + MASAK + KVKK + TCMB + SPK) banking impact: ____."
2. "BDDK 5411 + Bilgi Sistemleri Tebliği IT compliance (DR, pentest, audit, retention): ____."
3. "MASAK STR + sanctions screening + 3 rule (smurfing, cash, sanctions): ____."
4. "KVKK 6698 + consent + right to be forgotten + crypto-shred: ____."
5. "PCI-DSS PAN tokenization + scope reduction + Level 1 compliance: ____."
6. "KKB Findeks score + cache 7 gün + cost optimization: ____."
7. "GDPR vs KVKK paralel uyum + cross-border transfer AB: ____."
8. "Basel III CAR + LCR + NSFR banking metric + daily computation: ____."
9. "Incident notification BDDK 24h + KVKK 72h automated workflow: ____."
10. "Audit trail 14+ event type + retention 10 yıl + hash chain tamper-proof: ____."
