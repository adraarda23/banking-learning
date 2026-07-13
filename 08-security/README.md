# Faz 8 — Security

## Bu faz neyle ilgili?

Banking backend'inin **güvenlik** katmanı. Müşteri verisi (TC No, telefon, kart bilgisi), hesap erişimi ve para hareketleri — hepsi düşmanca bir internet ortamında savunulmak zorunda. Bu faz "Spring Security uygulamasını **regülasyona uygun** ve **OWASP defense-in-depth** prensipleri ile nasıl kurarım?" sorusunun cevabıdır.

Bu fazda öğreneceklerin:

- **Spring Security 6** filter chain mimarisi, `SecurityFilterChain` bean tabanlı modern konfigürasyon
- **Authentication**: UserDetails, password encoding (BCrypt/Argon2), brute-force koruması
- **JWT**: yapı, signing (HMAC/RSA/ECDSA), access vs refresh token, revocation stratejileri
- **OAuth2 / OIDC**: Authorization Code + PKCE, Client Credentials, Device Authorization grant'leri
- **Keycloak**: realm/client/role mimarisi, Spring Security resource server entegrasyonu
- **Encryption**: at-rest (column-level), in-transit (TLS/mTLS), envelope encryption, KMS
- **OWASP Top 10**: banking perspektifinde her bir madde (IDOR, SQLi, mass assignment, vb.)

## Hedef seviye

Faz sonunda:

- "Bir banking API'sini Keycloak ile authenticate ettiriyorum, JWT validate ediyorum, müşteri sadece kendi hesabını görebiliyor" diyebileceksin
- TC No ve telefon numarasını DB'de **şifreli** tutan column-level encryption uygulamış olacaksın
- OWASP Top 10 maddelerinden her birine **somut** banking örneği verebileceksin
- Rate limiting ile brute-force ve DDoS savunmasını implement etmiş olacaksın
- "Production-grade banking security": regülatör denetimine çıkarabileceğin koddur

## Regülatör konteksti (Türkiye)

Banking güvenliği teknik bir tercih değil, **yasal yükümlülüktür**. Senin yazdığın kod bir gün denetlenecek. Bu faz boyunca aklında olması gereken regülatörler:

### BDDK (Bankacılık Düzenleme ve Denetleme Kurumu)

- **Bankaların Bilgi Sistemleri ve Elektronik Bankacılık Hizmetleri Hakkında Yönetmelik** (Resmi Gazete, 15 Mart 2020)
- **Bilgi Sistemleri Yönetimi Tebliği** — bilgi sistemleri risk yönetimi, erişim kontrolü, log yönetimi
- Önemli maddeler:
  - Müşteri kimlik doğrulama (multi-factor authentication zorunluluğu finansal işlemlerde)
  - Hassas veri şifreleme (transit + rest)
  - Erişim log'larının **5 yıl** saklanması
  - Penetration test yıllık zorunluluk
  - SOC (Security Operations Center) yapısı

### KVKK (Kişisel Verilerin Korunması Kanunu)

- **Kanun No: 6698**, yürürlük 2016
- **Veri Sorumluları Sicili (VERBİS)** kayıt zorunluluğu
- Müşteri verisi = kişisel veri. **TC No, ad-soyad, telefon, e-posta, adres, IP, log'lar** kişisel veri.
- **Hassas kişisel veri** ayrı kategori: sağlık, biyometrik, ırk, din, cinsel hayat. Banking'de pek karşılaşılmaz ama biyometrik (parmak izi, yüz tanıma) varsa GİRER.
- Yükümlülükler:
  - **Veri minimizasyonu** — gereksiz veri toplama
  - **Şifreleme** — özellikle hassas veri (TC No ve banka müşterisi olarak müşteri verisi)
  - **Erişim kontrolü** — kim, ne zaman, hangi veriye erişti loglanır
  - **Saklama süresi** — amaca uygun, sonra silme
  - **Veri ihlali bildirimi** — 72 saat içinde KVKK'ya bildirim
- Ceza: data breach'te şirket cirosuna göre yüksek para cezası + reputational damage.

### PCI-DSS (Payment Card Industry Data Security Standard)

- Kart datası (PAN, CVV, expiration date) işleyen her sistem için zorunlu standart
- **PCI-DSS Level 1**: yıllık 6 milyon+ transaction işleyen acquirer/issuer — Türk bankaları bu seviyede
- Önemli maddeler:
  - **PAN (Primary Account Number) hiçbir zaman cleartext saklanamaz** — şifreli, tokenize, veya truncate
  - **CVV asla saklanamaz** (transaction sonrası bile)
  - **Network segmentation** — kart işlem ortamı (CDE - Cardholder Data Environment) izole olmalı
  - **Encryption**: AES-256 in-transit ve at-rest
  - **Access control**: need-to-know prensibi
  - **Logging**: kart datasına erişen her işlem log'lanır
  - **Vulnerability scanning**: çeyreklik network scan + yıllık penetration test
- **Tokenization** PCI-DSS scope reduction için anahtar — PAN yerine token saklarsan kart dataset'i azalır.

### Diğer ilgili regülasyonlar

- **MASAK (Mali Suçları Araştırma Kurulu)** — AML/KYC, **şüpheli işlem bildirimi**. Backend'in bunu destekleyen kayıt yapısına sahip olmalı.
- **SWIFT CSP (Customer Security Programme)** — SWIFT mesajlaşması yapan bankalar için. Sen muhtemelen direkt karşılaşmazsın ama haberdar ol.
- **TCMB Ödeme ve Menkul Kıymet Sistemleri** — anlık transfer (FAST), takas sistemleri.
- **GDPR** — Avrupa'lı müşterin varsa.

### Sertifika ve denetimler

- **ISO 27001** — bilgi güvenliği yönetim sistemi (BGYS) sertifikası — neredeyse tüm bankalarda var
- **SOC 2 Type II** — third-party audit, özellikle bulut servisleri için
- **TS 13298** — Türk standardı, kurumsal e-belge yönetimi
- Yıllık **iç denetim** + **bağımsız denetim** (BDDK izinli denetim firmaları)

Yazdığın her güvenlik kontrolü **denetlenebilir** olmalı. Audit log, decision trail, change request — hepsi günün birinde bir denetçiye sunulacak.

## Bu fazın faz 1-7 ile ilişkisi

| Faz | Bu fazda ne kullanılıyor |
|---|---|
| 1 — Foundation | RFC 7807 error handling — auth/access denied response'larında |
| 2 — JPA & Transactions | Encrypted column'lar (AttributeConverter), audit table'lar |
| 3 — Concurrency | Brute-force koruması — distributed lock + rate limit |
| 4 — SQL & Oracle | Şifreli kolonlarda index, partial encryption, query performance |
| 5 — Batch | Encryption key rotation batch'i, log retention batch'i |
| 6 — Messaging | Service-to-service authentication (mTLS, JWT propagation), Kafka SASL/SSL |
| 7 — Microservices | API Gateway → JWT validation, service mesh, BFF pattern |

## Faz yapısı

### Topic 1 — Spring Security Architecture (~700 satır)
Spring Security 6 filter chain. `SecurityFilterChain` bean. `AuthenticationManager`, `AuthenticationProvider`, `UserDetailsService`, `GrantedAuthority`. Method security: `@PreAuthorize`, `@PostAuthorize`. SpEL. Role hierarchy. Banking: account erişim kontrolü ("müşteri sadece kendi hesabı").

### Topic 2 — Authentication (~750 satır)
`UserDetails`, `BCryptPasswordEncoder`, `Argon2PasswordEncoder`. Password policy (uzunluk, kompleksite, history). `DelegatingPasswordEncoder` ile migration. Salt, peppering. Brute force koruması (account lockout). Banking: müşteri authentication akışı, ATM PIN vs internet banking password farkı.

### Topic 3 — JWT (~900 satır)
JWT yapısı (header, payload, signature). JWS vs JWE. HMAC vs RSA vs ECDSA signing. Access token vs refresh token. Token rotation. Revocation stratejileri (blocklist, short TTL). JWT pitfall'ları (alg=none, weak secret). JJWT library. Spring Security 6 OAuth2 resource server. Banking: JWT içerik kararı.

### Topic 4 — OAuth2 & OIDC (~850 satır)
OAuth2 grant'ları (Authorization Code + PKCE, Client Credentials, Device). OpenID Connect (OIDC). ID token vs access token. Scope vs claim. PKCE neden zorunlu. Refresh token rotation. Banking: mobile app → OIDC → banking API akışı.

### Topic 5 — Keycloak (~850 satır)
Keycloak mimarisi (realm, client, role, user, group). Docker setup. Realm config. Client (confidential vs public). Spring Security resource server entegrasyonu. Role mapping. Custom claim'ler. Keycloak admin REST API. Banking: multi-tenant bank realm, role hierarchy.

### Topic 6 — Encryption (~850 satır)
Encryption at rest vs in transit. TLS, mTLS (service-to-service). Column-level encryption (Hibernate `@ColumnTransformer`, `AttributeConverter`). Envelope encryption (DEK + KEK). KMS entegrasyonu (AWS KMS, HashiCorp Vault, Azure Key Vault). Key rotation. Tokenization (PCI-DSS scope reduction). Banking: TC No, telefon, kart numarası şifreleme.

### Topic 7 — OWASP Top 10 (~950 satır)
OWASP Top 10 (2021) banking perspektifi:
- A01 Broken Access Control (IDOR)
- A02 Cryptographic Failures
- A03 Injection (SQL, NoSQL, command, LDAP)
- A04 Insecure Design
- A05 Security Misconfiguration
- A06 Vulnerable Components
- A07 Authentication Failures
- A08 Software & Data Integrity
- A09 Logging Failures
- A10 SSRF

Plus banking-specific: mass assignment, card data exposure.

### Mini-project (~750 satır)
Phase 8 mini-project: Keycloak docker-compose + realm/client config + Spring Security resource server + JWT validation + `@PreAuthorize` ile account ownership + Hibernate `@ColumnTransformer` ile TC No + telefon şifreleme + Resilience4j RateLimiter + OWASP ZAP penetration test + ayrı audit log table.

### PHASE_TEST (~250 satır)
Self-assessment.

## Süre tahmini

Bu fazın tamamı: **3-4 hafta** (günde 2-3 saat).

- Topic 1-2 (Spring Security temel): ~5 gün
- Topic 3-4 (JWT, OAuth2): ~5 gün
- Topic 5 (Keycloak hands-on): ~3 gün
- Topic 6 (Encryption): ~4 gün
- Topic 7 (OWASP): ~5 gün
- Mini-project: ~5-7 gün

Banking security **çok geniş** bir konu. Bu faz seni "junior'ın bilmesi gerekenler"in üst sınırına çıkarır — gerçek banking security uzmanlığı ayrı bir kariyer dalıdır, sen burada **şu kadar bilen developer** olacaksın: "güvenli yazılım yazıyor, security ekibi ile aynı dili konuşabiliyor, kendi kodundaki bariz açıkları tanıyor."

## Önerilen okuma listesi (faz başlamadan önce göz at)

- **OWASP Top 10** (2021 edition) — official site, kısa
- **Spring Security Reference** — official, geniş ama referans olarak gerekli
- **OAuth 2.0 Simplified** (Aaron Parecki) — kitap, çok güzel pratik
- **OpenID Connect Core 1.0** spec — referans
- **RFC 6749** — OAuth 2.0
- **RFC 7519** — JWT
- **RFC 7636** — PKCE
- **NIST SP 800-63B** — authentication guidelines (password policy referansı)
- **PCI-DSS v4.0** — sektör standardı
- **BDDK Bilgi Sistemleri Yönetimi Tebliği** — Türkiye'ye özel

Faz boyunca bu kaynaklara döneceğiz.

## Başlama

→ Topic 1: [01-spring-security-architecture/](01-spring-security-architecture/README.md)

İyi öğrenmeler. Bu fazda kod yazmak kadar **düşünmek** de önemli — bir güvenlik kararı verirken "saldırgan ne yapabilir?" sorusunu sürekli kendine sor.
