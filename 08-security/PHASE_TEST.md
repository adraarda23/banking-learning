# Faz 8 — PHASE TEST

Faz 9'a geçmeden önce kendini sına.

## Pratik test

- [ ] Keycloak realm + 6 client + 5 role + composite + custom MFA flow
- [ ] 4 service Spring Security 6 + JWT validation + JwtAuthenticationConverter
- [ ] Gateway TokenRelay + per-service JWT revalidation
- [ ] Encryption: AES-GCM + envelope + KMS (LocalStack) + 3 service column encrypted
- [ ] PAN tokenization + audited detokenize role-based
- [ ] Blind index (HMAC-SHA256) searchable encryption
- [ ] Audit service + hash chain + Kafka topic + 14+ event type
- [ ] Step-up authentication >10k TL + international + new beneficiary
- [ ] Logging masking (TC, PAN, Authorization)
- [ ] Security headers (HSTS, CSP, XFO, XCT, Referrer-Policy)
- [ ] OWASP dependency-check + SpotBugs FindSecBugs + Trivy CI integration
- [ ] 25+ integration test
- [ ] OWASP ZAP baseline scan report
- [ ] 8 kasten kırma senaryosu reproduced + fixed + audit verified
- [ ] BDDK + KVKK + PCI-DSS compliance mapping doc

## Konsept testi

### Authentication
- [ ] Spring Security 6 filter chain anatomi (SecurityContextHolder, AuthenticationManager, AuthenticationProvider)
- [ ] UserDetailsService banking flexibility (username/email/phone)
- [ ] BCrypt strength 12 vs Argon2 OWASP 2023+
- [ ] DelegatingPasswordEncoder migration pattern
- [ ] BankingPasswordPolicy (12 char, complexity, history, breach check)
- [ ] HaveIBeenPwned k-anonymity API
- [ ] RedisLoginAttemptService (5/15 lockout) per-user + per-IP
- [ ] TOTP MFA (Google Authenticator) + SMS OTP
- [ ] MFA challenge flow + Redis state
- [ ] AuthenticationEventListener audit
- [ ] Session management stateless JWT

### JWT
- [ ] JWS vs JWE banking pratik
- [ ] HS256 vs RS256 microservice senaryosu
- [ ] Algorithm confusion attack + explicit enforcement
- [ ] alg=none attack + library default
- [ ] Standard claims (iss, sub, aud, exp, jti, nbf, iat)
- [ ] Access + Refresh pattern banking TTL
- [ ] Refresh rotation + compromise detection (reuse → revoke all)
- [ ] Token blocklist Redis + TTL
- [ ] JWKS endpoint key rotation
- [ ] Clock skew leeway
- [ ] aud check cross-service token reuse prevention
- [ ] PII payload anti-pattern (KVKK)

### OAuth2 + OIDC
- [ ] OAuth authorization vs OIDC authentication ayrımı
- [ ] Authorization Code + PKCE 7 step flow
- [ ] PKCE code_verifier + code_challenge
- [ ] Client Credentials service-to-service
- [ ] Refresh token rotation (reuseRefreshTokens(false))
- [ ] ID token vs Access token + custom claim
- [ ] UserInfo endpoint
- [ ] Discovery endpoint /.well-known/openid-configuration
- [ ] OAuth 2.1 deprecated (Implicit, Password)
- [ ] FAPI Advanced (mTLS, JAR, certificate-bound)
- [ ] BDDK Open Banking standards

### Keycloak
- [ ] Realm + client + role + group hierarchy
- [ ] Public (PKCE) vs confidential client
- [ ] Realm role vs client role + composite
- [ ] User attribute → token mapper → JWT claim
- [ ] Custom authentication flow + SPI risk assessment
- [ ] LDAP/AD federation READ_ONLY banking employee
- [ ] Event listener SPI → Kafka audit
- [ ] HA cluster Infinispan + PostgreSQL
- [ ] Banking theme customization (Türkçe, KVKK consent)
- [ ] Anti-pattern (direct access grants, wildcard URI, master realm)

### Encryption
- [ ] AES-256-GCM AEAD properties
- [ ] IV reuse catastrophic GCM
- [ ] Envelope encryption DEK + KEK rationale
- [ ] AWS KMS generateDataKey + encryption context
- [ ] HashiCorp Vault transit engine
- [ ] JPA AttributeConverter column-level
- [ ] Blind index (HMAC) searchable encryption trade-off
- [ ] Tokenization vs encryption PCI scope reduction
- [ ] FPE format-preserving (legacy systems)
- [ ] TLS 1.3 + forward secrecy + HSTS
- [ ] mTLS service-to-service (service mesh)
- [ ] Key rotation lazy re-encrypt
- [ ] Crypto-shredding KVKK silinme hakkı

### OWASP Top 10
- [ ] A01 Broken Access Control: IDOR + repo-level fix + deny-by-default + Not Found NOT Forbidden
- [ ] A02 Cryptographic Failures: TLS + AES-GCM + KMS + log masking + BCrypt
- [ ] A03 Injection: parametrize SQL + @Valid + Thymeleaf escape + CSP
- [ ] A04 Insecure Design: step-up auth + threat model STRIDE + misuse cases
- [ ] A05 Security Misconfiguration: actuator lockdown + default creds + stack trace + headers
- [ ] A06 Vulnerable Components: dependency-check + SBOM + Trivy + patch policy
- [ ] A07 Auth Failures: brute force + session fixation + cookie flags
- [ ] A08 Software/Data Integrity: ObjectInputFilter + webhook HMAC + audit hash chain
- [ ] A09 Logging/Monitoring: SIEM + retention + tamper-proof + PII mask
- [ ] A10 SSRF: allowlist + private IP block + DNS rebind + IMDSv2

## Banking compliance

- [ ] KVKK requirements: encryption at rest, audit, silinme hakkı (crypto-shred)
- [ ] BDDK güvenlik tebliği: MFA, audit retention 5-10 yıl, IT continuity
- [ ] PCI-DSS: PAN tokenization, key rotation yıllık, AES-256 min, audit, NETWORK segmentation
- [ ] MASAK: KYC, customer due diligence, transaction monitoring
- [ ] Open Banking BDDK (2020+): PSD2 benzeri, FAPI alignment

## Soft skills

- [ ] 15 mini-project defter notu yazılmış
- [ ] OWASP Top 10 her madde banking örnek + fix kafamda
- [ ] Compliance regulatory bağlama yapabilirim
- [ ] Threat modeling (STRIDE) banking feature için yapabilirim
- [ ] Security incident response plan'i taslak çizebilirim

## Kaç gün?

Tahmin: 25-30 gün (günde 3 saat). Phase 8 = phase 7 ile birlikte **mid+ seniora yakın seviyenin imzası**.

---

Hepsine "evet" → **Faz 9'a geç → 09-observability/**

---

## Bonus — senior banking security engineer işareti

Phase 8 bittikten sonra şu cümleleri söyleyebiliyorsan:

- "Banking realm Keycloak setup'ı, custom MFA flow + risk SPI dahil, sıfırdan kurabilirim."
- "Envelope encryption + KMS + blind index pattern'ini banking PII için uygulayabilirim."
- "PAN tokenization service'i ayrı microservice + PCI scope isolation ile tasarlayabilirim."
- "OWASP Top 10 her risk için banking-specific defense + reproduce + fix gösterebilirim."
- "Audit hash chain + Kafka + tamper detection tasarımı + verify scheduled job çıkarabilirim."
- "Step-up authentication banking yüksek değer işlem için OTP challenge + Redis state."
- "JWT compromise detection (refresh reuse) + tokens revoke + user alert flow."
- "BDDK + KVKK + PCI-DSS compliance mapping ile feature design yapabilirim."
- "Security headers (HSTS, CSP, XFO) + actuator lockdown + ProblemDetail standardı."
- "SAST (FindSecBugs) + SCA (dependency-check) + DAST (ZAP) CI pipeline kurabilirim."

TR bankalarında **senior backend / security engineer** rolüne hazır profilsin.

---

## Banking için Phase 8'in yeri

Phase 8 = **TR banking regulator + auditor expectation karşılayan baseline**. 

- **BDDK denetimi:** Audit log 5-10 yıl, MFA, encryption mandatory
- **KVKK uyumu:** PII encryption, log masking, silinme hakkı
- **PCI-DSS:** PAN tokenization, key rotation, network isolation
- **MASAK:** Transaction audit + suspicious activity
- **Open Banking:** OAuth + FAPI + consent management

Mid+ developer banking güvenliğini bilmeden ilerleyemez. Senior'a giderken **security mindset** zorunlu. TR bank interview'lerinde **OWASP Top 10**, **encryption at rest**, **MFA flow**, **JWT pitfalls** sorulan top konular.

Phase 8 → Phase 9 (Observability) doğal devam: audit log + metrics + tracing **integrated security observability**.
