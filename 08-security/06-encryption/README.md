# Topic 8.6 — Encryption: At Rest, In Transit, Envelope, KMS

## Hedef

Banking encryption pratiğini derinlemesine öğrenmek: column-level encryption (KVKK + PCI-DSS), envelope encryption (DEK + KEK), KMS integration (AWS KMS, HashiCorp Vault), tokenization, format-preserving encryption, TLS/mTLS, key rotation. Java/JCE/Bouncy Castle implementation.

## Süre

Okuma: 2.5 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Symmetric vs asymmetric crypto temel
- AES, RSA, SHA algoritmalarını duymuş ol
- Banking compliance: PCI-DSS, KVKK, BDDK

---

## Kavramlar

### 1. Encryption — neden banking için kritik

Banking verisi sınıflandırması:

| Veri | Risk | Encryption gereği |
|---|---|---|
| TC kimlik no | KVKK + reputational | Column-level + KMS |
| Card PAN | PCI-DSS | Tokenization mandatory |
| Card CVV | PCI-DSS | Asla saklama (cardholder data) |
| Card expiry | PCI-DSS | Encrypt |
| Account balance | Internal | TLS in transit; rest OK |
| Customer email/phone | KVKK | Column-level |
| Transaction amount | Internal | TLS in transit |
| Password | OWASP | BCrypt/Argon2 hash (not encrypt) |
| Session token | OWASP | TLS only; opaque |

**KVKK + BDDK requirements:**
- Sensitive PII at rest **encrypted**
- Key management separated from data
- Audit log encryption operations
- Key rotation **yıllık min**
- Crypto-shredding (key destroy = data destroy)

**PCI-DSS:**
- PAN encryption mandatory
- Key management documented
- Encryption keys themselves protected
- Annual key rotation
- Strong cryptography (AES-256, RSA 2048+)

### 2. Symmetric vs asymmetric

**Symmetric (AES):**
- Aynı key encrypt + decrypt
- Hızlı (1-10 GB/s)
- Banking için **column-level encryption**, **file encryption**

**Asymmetric (RSA, ECC):**
- Public key encrypt → private key decrypt
- Yavaş (~1 MB/s)
- Banking için **TLS handshake**, **JWT signing**, **digital signature**, **key exchange**

**Hybrid (envelope):**
- Symmetric ile data encrypt
- Asymmetric ile symmetric key encrypt
- En yaygın production pattern

### 3. AES — banking standard

**Block cipher** — 128 bit block.

**Key size:**
- AES-128 (legacy)
- AES-192
- AES-256 (banking standard) ← önerilir

**Modes:**

| Mode | Banking |
|---|---|
| **ECB** | ❌ NEVER (same plaintext → same ciphertext, pattern leak) |
| **CBC** | ✓ Eski. IV gerekli. Padding oracle attack risk. |
| **CTR** | ✓ Parallel. IV gerekli. |
| **GCM** | ✓✓ AEAD (Authenticated Encryption). Banking modern standard. |
| **CCM** | ✓ AEAD alternative. |
| **XTS** | Disk encryption (LUKS, FileVault). |

**AES-GCM** banking için pick. AEAD = encryption + integrity (no separate HMAC needed).

### 4. AES-GCM example (Java JCE)

```java
public class AesGcmEncryption {
    
    private static final int GCM_IV_LENGTH = 12;        // 96 bits — GCM standard
    private static final int GCM_TAG_LENGTH = 128;      // bits
    private static final SecureRandom RANDOM = new SecureRandom();
    
    public static EncryptedData encrypt(byte[] plaintext, byte[] key, byte[] aad) {
        try {
            byte[] iv = new byte[GCM_IV_LENGTH];
            RANDOM.nextBytes(iv);
            
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(key, "AES");
            GCMParameterSpec gcmSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, gcmSpec);
            
            if (aad != null) {
                cipher.updateAAD(aad);   // Authenticated but not encrypted
            }
            
            byte[] ciphertext = cipher.doFinal(plaintext);
            
            return new EncryptedData(iv, ciphertext);
        } catch (Exception e) {
            throw new EncryptionException("AES-GCM encrypt failed", e);
        }
    }
    
    public static byte[] decrypt(EncryptedData data, byte[] key, byte[] aad) {
        try {
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(key, "AES");
            GCMParameterSpec gcmSpec = new GCMParameterSpec(GCM_TAG_LENGTH, data.iv());
            cipher.init(Cipher.DECRYPT_MODE, keySpec, gcmSpec);
            
            if (aad != null) {
                cipher.updateAAD(aad);
            }
            
            return cipher.doFinal(data.ciphertext());
        } catch (AEADBadTagException e) {
            // Integrity failure — tampering OR wrong key
            throw new IntegrityException("GCM authentication tag mismatch", e);
        } catch (Exception e) {
            throw new EncryptionException("AES-GCM decrypt failed", e);
        }
    }
}

public record EncryptedData(byte[] iv, byte[] ciphertext) {}
```

**CRITICAL: IV reuse**

AES-GCM aynı key + aynı IV = **catastrophic failure**. Saldırgan ciphertext'leri XOR'layabilir.

```java
// ❌ WRONG
byte[] iv = "fixed_iv_12_".getBytes();   // Sabit IV

// ✓ CORRECT
byte[] iv = new byte[12];
new SecureRandom().nextBytes(iv);
```

Banking pratiği: Her encryption fresh IV. IV ciphertext ile birlikte stored (gizli değil, ama unique).

### 5. Envelope encryption — KMS pattern

**Problem:** Bir master key tüm veriyi şifreliyor →
- Key leak → tüm data açık
- Key rotation → tüm veri yeniden encrypt (massive)
- Application key yönetimi karmaşık

**Çözüm — Envelope:**

```
Data Encryption Key (DEK):  Random AES-256, per-record
Key Encryption Key (KEK):   Master key, KMS'de, rarely rotates

Encrypt flow:
  1. Generate fresh DEK (random AES-256)
  2. Encrypt data with DEK
  3. Encrypt DEK with KEK (KMS)
  4. Store: encrypted_data + encrypted_DEK

Decrypt flow:
  1. KMS decrypt(encrypted_DEK) → plain DEK
  2. Decrypt data with DEK
  3. Discard DEK (in-memory only)
```

**Avantajlar:**
- Performance: KMS sadece DEK için çağrılır
- Rotation: KEK rotate → re-encrypt sadece DEK'leri (kısa)
- Crypto-shredding: KEK destroy = tüm data destroy

```java
@Service
public class EnvelopeEncryptionService {
    
    private final KmsClient kmsClient;
    private final String kekId;
    private final SecureRandom random = new SecureRandom();
    
    public EncryptedRecord encrypt(byte[] plaintext, String aad) {
        // 1. Generate fresh DEK
        byte[] dek = new byte[32];   // 256-bit
        random.nextBytes(dek);
        
        try {
            // 2. Encrypt data with DEK (AES-GCM)
            EncryptedData encrypted = AesGcmEncryption.encrypt(
                plaintext, dek, aad.getBytes(UTF_8));
            
            // 3. Encrypt DEK with KEK (KMS)
            EncryptResponse kmsResp = kmsClient.encrypt(EncryptRequest.builder()
                .keyId(kekId)
                .plaintext(SdkBytes.fromByteArray(dek))
                .encryptionContext(Map.of("purpose", "banking-pii"))
                .build());
            
            byte[] encryptedDek = kmsResp.ciphertextBlob().asByteArray();
            
            return new EncryptedRecord(
                encrypted.iv(),
                encrypted.ciphertext(),
                encryptedDek,
                kekId);
        } finally {
            // 4. Zero DEK in memory
            Arrays.fill(dek, (byte) 0);
        }
    }
    
    public byte[] decrypt(EncryptedRecord record, String aad) {
        byte[] dek = null;
        try {
            // 1. KMS decrypt DEK
            DecryptResponse kmsResp = kmsClient.decrypt(DecryptRequest.builder()
                .ciphertextBlob(SdkBytes.fromByteArray(record.encryptedDek()))
                .keyId(record.kekId())
                .encryptionContext(Map.of("purpose", "banking-pii"))
                .build());
            
            dek = kmsResp.plaintext().asByteArray();
            
            // 2. Decrypt data with DEK
            EncryptedData encrypted = new EncryptedData(record.iv(), record.ciphertext());
            return AesGcmEncryption.decrypt(encrypted, dek, aad.getBytes(UTF_8));
        } finally {
            if (dek != null) {
                Arrays.fill(dek, (byte) 0);
            }
        }
    }
}

public record EncryptedRecord(
    byte[] iv, 
    byte[] ciphertext, 
    byte[] encryptedDek, 
    String kekId) {}
```

**Performance optimization — DEK caching:**

Aynı record'da multiple op? KMS round-trip için cache:

```java
@Service
public class DekCachingService {
    
    private final Cache<String, byte[]> dekCache;  // Caffeine
    private final KmsClient kmsClient;
    
    public DekCachingService() {
        this.dekCache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(5))   // Short!
            .build();
    }
    
    public byte[] getDek(String dekId, byte[] encryptedDek) {
        return dekCache.get(dekId, k -> kmsDecrypt(encryptedDek));
    }
}
```

**Trade-off:** Cache hit → fast. Cache miss → KMS call. Banking için TTL kısa (5-15 dk).

### 6. AWS KMS integration

```java
@Configuration
public class KmsConfig {
    
    @Bean
    public KmsClient kmsClient() {
        return KmsClient.builder()
            .region(Region.EU_CENTRAL_1)
            .credentialsProvider(DefaultCredentialsProvider.create())
            .build();
    }
    
    @Bean
    public String bankingKekId() {
        return "arn:aws:kms:eu-central-1:123456789012:key/abc-def-...";
    }
}
```

**Generate Data Key (KMS native):**

```java
GenerateDataKeyResponse resp = kmsClient.generateDataKey(GenerateDataKeyRequest.builder()
    .keyId(kekId)
    .keySpec(DataKeySpec.AES_256)
    .encryptionContext(Map.of("tenant", "TR", "purpose", "pii"))
    .build());

byte[] plaintextDek = resp.plaintext().asByteArray();
byte[] encryptedDek = resp.ciphertextBlob().asByteArray();
```

Tek call: plain DEK + encrypted DEK döner. Sonra plain DEK'i in-app encrypt için, encrypted DEK'i store et.

### 7. HashiCorp Vault (alternatif KMS)

```hcl
# Transit secret engine — encryption as service
vault secrets enable transit
vault write -f transit/keys/banking-pii \
  type=aes256-gcm96 \
  exportable=false \
  auto_rotate_period=8760h    # 1 year
```

```java
@Service
public class VaultEncryptionService {
    
    private final VaultTemplate vaultTemplate;
    
    public String encrypt(String plaintext) {
        Map<String, String> body = Map.of(
            "plaintext", Base64.getEncoder().encodeToString(plaintext.getBytes(UTF_8)),
            "context", Base64.getEncoder().encodeToString("banking-pii".getBytes(UTF_8))
        );
        
        VaultResponse response = vaultTemplate.write("transit/encrypt/banking-pii", body);
        return (String) response.getData().get("ciphertext");
        // "vault:v1:base64ciphertext..."
    }
    
    public String decrypt(String ciphertext) {
        Map<String, String> body = Map.of(
            "ciphertext", ciphertext,
            "context", Base64.getEncoder().encodeToString("banking-pii".getBytes(UTF_8))
        );
        
        VaultResponse response = vaultTemplate.write("transit/decrypt/banking-pii", body);
        String base64Plain = (String) response.getData().get("plaintext");
        return new String(Base64.getDecoder().decode(base64Plain), UTF_8);
    }
}
```

Vault encryption-as-a-service:
- Key Vault'tan çıkmaz
- Auto-rotation
- Audit log built-in
- HA cluster

Banking trade-off: Vault on-prem mümkün, AWS KMS cloud only. Türk bankası **on-prem** sebebi ile Vault yaygın.

### 8. Column-level encryption — JPA AttributeConverter

PII field'lara otomatik encrypt:

```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    
    private static EnvelopeEncryptionService encryptionService;
    private static String tenant;
    
    @Autowired
    public void setService(EnvelopeEncryptionService service) {
        EncryptedStringConverter.encryptionService = service;
    }
    
    @Override
    public String convertToDatabaseColumn(String plaintext) {
        if (plaintext == null) return null;
        
        EncryptedRecord record = encryptionService.encrypt(
            plaintext.getBytes(UTF_8), tenant);
        
        return Base64.getEncoder().encodeToString(serialize(record));
    }
    
    @Override
    public String convertToEntityAttribute(String dbValue) {
        if (dbValue == null) return null;
        
        EncryptedRecord record = deserialize(Base64.getDecoder().decode(dbValue));
        byte[] plaintext = encryptionService.decrypt(record, tenant);
        
        return new String(plaintext, UTF_8);
    }
}

@Entity
public class Customer {
    @Id
    private UUID id;
    
    private String name;   // Plain
    
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "tc_kimlik_encrypted")
    private String tcKimlik;
    
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "email_encrypted")
    private String email;
    
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "phone_encrypted")
    private String phone;
    
    // ...
}
```

DB row:
```
id          | name        | tc_kimlik_encrypted              | email_encrypted
abc-123     | Ahmet       | base64(iv+ciphertext+enc_dek)    | base64(...)
```

DBA `tc_kimlik` görmesin — encryption transparent application'da.

**Searchable encryption challenge:**

```sql
SELECT * FROM customers WHERE tc_kimlik = '12345678901';
```

DB'de encrypted column. Plain text aramaz! Çözüm: **deterministic encryption** (aynı plaintext → aynı ciphertext, but pattern leak risk) veya **blind index** (HMAC of plaintext, lookup).

```java
@Entity
public class Customer {
    @Convert(converter = EncryptedStringConverter.class)
    private String tcKimlik;        // GCM (different each time)
    
    @Column(name = "tc_kimlik_hash", length = 64)
    private String tcKimlikHash;    // HMAC-SHA256(tcKimlik, secret) — searchable
    
    @PrePersist
    @PreUpdate
    void computeHash() {
        if (tcKimlik != null) {
            tcKimlikHash = hmac(tcKimlik);
        }
    }
}
```

Query:
```java
public Optional<Customer> findByTcKimlik(String tcKimlik) {
    String hash = hmac(tcKimlik);
    return repo.findByTcKimlikHash(hash);
}
```

### 9. Tokenization — PCI-DSS

PAN (kart numarası) banking için **stored DEĞİL**. Yerine **token**.

```
PAN: 4532-1488-0343-6467
↓ tokenize
Token: tok_5f9a2b8c3d1e4f7g

Token → PAN mapping vault'ta (vault = ayrı, sıkı korunmuş).
PAN burst point sadece vault'a access olan service.
```

**Tokenization vs encryption:**

| | Encryption | Tokenization |
|---|---|---|
| Reversibility | Key ile | Vault lookup |
| Output format | Random ciphertext | Format-preserving |
| Search | Hard | Direct |
| Scope | Any data | Mostly PAN |
| Compliance | PCI scope still | Reduces PCI scope |

Banking: PCI scope **azaltmak** için tokenization. App PAN görmez, sadece token.

```java
@Service
public class CardTokenizationService {
    
    private final VaultStorage vault;   // Strict access, dedicated service
    
    public String tokenize(String pan) {
        validatePan(pan);
        String token = generateToken();   // Random format-preserving
        vault.store(token, encrypt(pan));
        return token;
    }
    
    public String detokenize(String token) {
        // Sıkı access control — only payment processor service
        SecurityContext.requireRole("payment-processor");
        return decrypt(vault.retrieve(token));
    }
    
    private String generateToken() {
        // tok_<16 random chars>
        return "tok_" + RandomStringUtils.randomAlphanumeric(16);
    }
}
```

Banking pratiği: Tokenization service **ayrı microservice**, **ayrı network zone**, **ayrı KMS key**.

### 10. Format-Preserving Encryption (FPE)

```
PAN: 4532-1488-0343-6467  →  FPE encrypt  →  9876-5432-1098-7654
   (16 digits)                              (still 16 digits)
```

Card processor sistemler 16 digit PAN bekler — encrypt etse output 16 digit kalmalı. **FFX, FF1, FF3-1** modları (NIST 800-38G).

Banking nadir kullanır (legacy uyumluluk), tokenization daha yaygın.

### 11. TLS — in transit

Banking için **TLS 1.2 minimum**, **TLS 1.3 preferred**.

```yaml
# Spring Boot application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: banking-api
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
    ciphers: 
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
```

**Banking TLS requirements:**
- HSTS header (`Strict-Transport-Security`)
- TLS 1.0 / 1.1 disabled (deprecated)
- Weak ciphers disabled (RC4, 3DES, CBC mode)
- Forward secrecy (ECDHE)
- Certificate from trusted CA (or internal PKI)
- Certificate pinning mobile app'te
- OCSP stapling

### 12. mTLS — service-to-service

```yaml
server:
  ssl:
    client-auth: need   # mTLS mandatory
    trust-store: classpath:truststore.jks
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

Client side:
```java
SSLContext sslContext = SSLContextBuilder.create()
    .loadKeyMaterial(clientKeyStore, keyPassword)
    .loadTrustMaterial(trustStore, null)
    .build();

WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create().secure(s -> s.sslContext(sslContext))))
    .build();
```

Banking pratiği: **Service mesh (Istio, Linkerd)** otomatik mTLS. Cert rotation otomatik.

### 13. Key rotation

**Why rotate:**
- Crypto best practice (key compromise window azalt)
- Regulatory requirement (yıllık min)
- Personnel changes

**Rotation strategy:**

1. **Generate new key** (v2)
2. **Use new key for new data**
3. **Lazy re-encrypt** old data (background batch)
4. **Old key kept** for decryption only
5. **Audit complete** → destroy old key

```java
@Service
public class KeyRotationService {
    
    @Scheduled(cron = "0 0 2 * * SAT")   // Weekly batch
    public void rotateOldRecords() {
        List<EncryptedRecord> oldVersion = repo.findByKekId(oldKekId, limit: 1000);
        
        for (EncryptedRecord record : oldVersion) {
            byte[] plaintext = encryptionService.decryptWithKek(record, oldKekId);
            EncryptedRecord newRecord = encryptionService.encryptWithKek(plaintext, newKekId);
            repo.replace(record.id(), newRecord);
        }
    }
}
```

KMS bunu da abstract eder — KMS internal'da key version yönetir.

### 14. Crypto-shredding

GDPR right-to-be-forgotten, KVKK silinme hakkı:

Naive: User'ın tüm data'sını DB'den sil → backup'lar, log'lar, replica'lar?

**Crypto-shredding:** User'a özel encryption key. **Key destroy = data unreadable.**

```
User A's data encrypted with DEK_A
DEK_A encrypted with KEK
User A "delete me" → destroy DEK_A
  Backup'lar hala var ama decrypt edilemez
```

KVKK için powerful pattern.

### 15. Banking — encryption anti-pattern'leri

**Anti-pattern 1: ECB mode**

```java
Cipher.getInstance("AES/ECB/PKCS5Padding")   // ❌
```

Same plaintext → same ciphertext. Pattern visible. **GCM kullan.**

**Anti-pattern 2: IV reuse**

GCM same key + same IV = catastrophic. Her encrypt fresh IV.

**Anti-pattern 3: Hardcoded key**

```java
private static final String KEY = "my-secret-key-12";   // ❌
```

Source code'da key = leak. KMS / Vault / env variable.

**Anti-pattern 4: Self-implemented crypto**

```java
// "I'll just XOR with my password"   ❌
```

Banking için JCE / Bouncy Castle / battle-tested library.

**Anti-pattern 5: Encryption WITHOUT authentication**

CBC mode integrity yok. Padding oracle attack. **AEAD (GCM)** kullan.

**Anti-pattern 6: Weak random**

```java
Random random = new Random();   // ❌ predictable
```

**SecureRandom** kullan.

**Anti-pattern 7: Key + ciphertext same place**

Tehlikenin tüm yumurta sepeti = aynı yerde key + data. KMS ayır.

**Anti-pattern 8: Algorithm string typo**

```java
Cipher.getInstance("AES")   // → defaults to ECB!
```

Always explicit: `"AES/GCM/NoPadding"`.

**Anti-pattern 9: Key rotation yok**

Banking yıllık rotation min. Schedule otomatik.

**Anti-pattern 10: PAN encryption (tokenization yerine)**

PAN encrypt + key in same system = PCI scope hala büyük. Tokenization ile PCI scope **isolation**.

---

## Önemli olabilecek araştırma kaynakları

- NIST SP 800-38D — GCM mode
- NIST SP 800-38G — FPE
- NIST SP 800-57 — Key management
- PCI-DSS v4 — Encryption requirements
- KVKK Tedbirler Rehberi
- AWS KMS Developer Guide
- HashiCorp Vault docs
- OWASP Cryptographic Storage Cheat Sheet
- Bouncy Castle JCE provider

---

## Mini task'ler

### Task 8.6.1 — AES-GCM encrypt/decrypt utility (45 dk)

`AesGcmEncryption.encrypt/decrypt` implement. Fresh IV per call. Test: tampered ciphertext → AEADBadTagException.

### Task 8.6.2 — Envelope encryption service (60 dk)

DEK generate (SecureRandom), KEK encrypt simulate (local AES). EncryptedRecord serialize/deserialize.

### Task 8.6.3 — AWS KMS integration (or LocalStack) (60 dk)

`kmsClient.generateDataKey` → DEK alma. Real envelope flow. (LocalStack için KMS mock OK.)

### Task 8.6.4 — JPA AttributeConverter encrypted columns (60 dk)

`@Convert` ile `tcKimlik`, `email`, `phone` otomatik encrypt. Test: DB'ye gir, raw value base64 ciphertext görmeli.

### Task 8.6.5 — Blind index (HMAC) searchable encryption (45 dk)

`tcKimlikHash` field HMAC-SHA256. `findByTcKimlik` method hash üzerinden query.

### Task 8.6.6 — Tokenization service (45 dk)

CardTokenizationService. `tokenize(pan) → token`, `detokenize(token) → pan`. Role-based access (detokenize sadece payment-processor).

### Task 8.6.7 — TLS 1.3 + cipher suite (30 dk)

Spring Boot HTTPS config. SSLLabs benzeri test (`openssl s_client -connect host:8443 -tls1_3`).

### Task 8.6.8 — mTLS service-to-service (60 dk)

İki Spring Boot service. Client cert mandatory. Wrong cert → 403.

### Task 8.6.9 — Key rotation manual (45 dk)

Old KEK ile encrypt edilmiş 100 record. Yeni KEK ile re-encrypt. Audit log.

### Task 8.6.10 — HashiCorp Vault transit (60 dk)

Vault Docker. `transit` engine. `encrypt/decrypt` Spring VaultTemplate.

---

## Test yazma rehberi

```java
@Test
void aesGcmEncryptDecryptRoundTrip() {
    byte[] key = randomBytes(32);
    byte[] plaintext = "TC: 12345678901".getBytes(UTF_8);
    byte[] aad = "tenant=TR".getBytes(UTF_8);
    
    EncryptedData encrypted = AesGcmEncryption.encrypt(plaintext, key, aad);
    byte[] decrypted = AesGcmEncryption.decrypt(encrypted, key, aad);
    
    assertThat(decrypted).isEqualTo(plaintext);
}

@Test
void shouldFailWithDifferentAad() {
    byte[] key = randomBytes(32);
    byte[] plaintext = "data".getBytes();
    
    EncryptedData encrypted = AesGcmEncryption.encrypt(plaintext, key, "tenant=TR".getBytes());
    
    assertThatThrownBy(() ->
        AesGcmEncryption.decrypt(encrypted, key, "tenant=DE".getBytes()))
        .isInstanceOf(IntegrityException.class);
}

@Test
void shouldFailWithTamperedCiphertext() {
    byte[] key = randomBytes(32);
    EncryptedData encrypted = AesGcmEncryption.encrypt(
        "data".getBytes(), key, null);
    
    encrypted.ciphertext()[0] ^= 0x01;   // tamper
    
    assertThatThrownBy(() -> AesGcmEncryption.decrypt(encrypted, key, null))
        .isInstanceOf(IntegrityException.class);
}

@Test
void shouldUseFreshIvEachEncrypt() {
    byte[] key = randomBytes(32);
    byte[] plaintext = "same data".getBytes();
    
    EncryptedData e1 = AesGcmEncryption.encrypt(plaintext, key, null);
    EncryptedData e2 = AesGcmEncryption.encrypt(plaintext, key, null);
    
    assertThat(e1.iv()).isNotEqualTo(e2.iv());
    assertThat(e1.ciphertext()).isNotEqualTo(e2.ciphertext());
}

@Test
@Transactional
void customerEncryptedColumnsRoundTrip() {
    Customer c = new Customer();
    c.setName("Ahmet");
    c.setTcKimlik("12345678901");
    c.setEmail("ahmet@example.com");
    
    customerRepo.saveAndFlush(c);
    em.clear();
    
    // Raw DB query
    String rawTcKimlik = (String) em.createNativeQuery(
        "SELECT tc_kimlik_encrypted FROM customers WHERE id = :id")
        .setParameter("id", c.getId()).getSingleResult();
    
    assertThat(rawTcKimlik).doesNotContain("12345678901");   // Encrypted
    
    Customer loaded = customerRepo.findById(c.getId()).orElseThrow();
    assertThat(loaded.getTcKimlik()).isEqualTo("12345678901");   // Decrypted
}

@Test
void blindIndexLookup() {
    Customer c = new Customer();
    c.setTcKimlik("12345678901");
    customerRepo.saveAndFlush(c);
    
    Optional<Customer> found = customerService.findByTcKimlik("12345678901");
    
    assertThat(found).isPresent();
    assertThat(found.get().getId()).isEqualTo(c.getId());
}

@Test
void tokenizationRoundTrip() {
    String pan = "4532148803436467";
    
    String token = tokenizationService.tokenize(pan);
    
    assertThat(token).startsWith("tok_");
    assertThat(token).doesNotContain(pan);
    
    String detokenized = tokenizationService.detokenize(token);
    assertThat(detokenized).isEqualTo(pan);
}
```

---

## Claude-verify prompt

```
Encryption implementation'ımı banking-grade kriterlere göre değerlendir:

1. Algoritma:
   - AES-256-GCM (banking standard)?
   - ECB mode YOK?
   - CBC mode → tercih edilmiyor, GCM kullanılıyor mu?
   - SecureRandom (Math.random YOK)?

2. IV:
   - Her encrypt fresh IV?
   - 12-byte (96-bit) GCM standardı?
   - IV ciphertext ile birlikte stored?
   - IV reuse YOK?

3. Envelope encryption:
   - DEK random AES-256?
   - KEK KMS / Vault'ta?
   - DEK in-memory only (Arrays.fill zero out)?
   - Encryption context (AAD) tenant/purpose?

4. KMS integration:
   - AWS KMS / HashiCorp Vault?
   - KMS key permission IAM/policy?
   - Key not exportable?
   - Audit log enable?

5. Column-level:
   - JPA AttributeConverter @Convert?
   - Sensitive PII (TC, email, phone, address) encrypted?
   - Plain field DB'de YOK?

6. Searchable:
   - Blind index (HMAC) lookup?
   - Deterministic encryption pattern leak'i değerlendirildi mi?

7. Tokenization:
   - PAN encrypt yerine tokenization?
   - Token vault ayrı service?
   - Detokenize role-based?

8. TLS:
   - TLS 1.2 min, 1.3 preferred?
   - Weak cipher disabled?
   - HSTS header?
   - Certificate pinning mobile?

9. mTLS:
   - Service-to-service mTLS?
   - Cert rotation automated (service mesh)?

10. Key rotation + crypto-shredding:
    - Yıllık rotation scheduled?
    - Lazy re-encrypt batch?
    - Crypto-shred KVKK silinme hakkı?

11. Anti-pattern:
    - Hardcoded key YOK?
    - Self-implemented crypto YOK?
    - "AES" (default ECB) YOK?
    - IV reuse YOK?
    - PAN encryption yerine tokenization YOK?
    - Weak random YOK?
```

---

## Tamamlama kriterleri

- [ ] AES-GCM utility + tampering test
- [ ] Envelope encryption service (KMS or local KEK)
- [ ] AWS KMS or Vault integration
- [ ] JPA AttributeConverter (encrypted columns)
- [ ] Blind index searchable encryption
- [ ] Tokenization service (role-based)
- [ ] TLS 1.3 + strong cipher config
- [ ] mTLS service-to-service test
- [ ] Key rotation batch
- [ ] Crypto-shredding demo
- [ ] 8+ integration test
- [ ] Tampered ciphertext / wrong AAD reject

---

## Defter notları (10 madde)

1. "AES-256-GCM AEAD properties (encryption + integrity) banking sebebi: ____."
2. "IV reuse catastrophic failure GCM: ____."
3. "Envelope encryption DEK + KEK trade-off (performance, rotation, crypto-shred): ____."
4. "AWS KMS generateDataKey + encryption context: ____."
5. "JPA AttributeConverter column-level encryption transparent: ____."
6. "Blind index (HMAC) deterministic encryption pattern leak trade-off: ____."
7. "Tokenization vs encryption PCI scope reduction: ____."
8. "TLS 1.3 + forward secrecy + HSTS banking baseline: ____."
9. "mTLS service-to-service + service mesh (Istio/Linkerd): ____."
10. "Crypto-shredding KVKK silinme hakkı pattern: ____."
