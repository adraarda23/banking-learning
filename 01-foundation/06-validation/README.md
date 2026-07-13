# Topic 1.6 — Validation: Bean Validation, custom validators

## Hedef

Input validasyonunu **kod-temiz** ve **deklaratif** bir biçimde uygulamak. Jakarta Bean Validation'ın derin özelliklerini öğrenmek. Banking-specific validator'lar (IBAN, TC Kimlik No, IBAN-currency uyumu) yazmak. Validation group'larıyla aynı DTO'yu farklı senaryolarda farklı kurallarla kontrol etmek.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~4.5 saat

## Önbilgi

- Topic 1.1-1.5 tamamlandı
- DTO ayrımı yapıldı
- `@Valid` ne anlama geldiğini sezgisel olarak biliyorsun

---

## Kavramlar

### 1. Neden validation katmanlı (input'a güvenmeme)

Banking'de "input'a güvenmek" güvenlik açığıdır. Sebepler:
- Client-side validation bypass edilebilir
- API direkt çağrılır (curl, Postman)
- Internal sistemler bile bug üretebilir
- Audit trail için sebep loglu validation gerekir

Validation 3 katmanda yapılır:

| Katman | Sorumluluğu | Örnek |
|---|---|---|
| **Syntactic** (input parser) | Tip, format, length | "amount sayı mı?", "currency 3 karakter mi?" |
| **Semantic** (DTO validation) | Business kuralı dışında değer kuralları | "amount > 0 mı?", "currency ISO listesinde mi?" |
| **Domain** (aggregate) | İş kuralları, state-bağımlı | "hesabın bakiyesi yeterli mi?", "hesap kapalı mı?" |

İlk iki katman **Bean Validation** ile, üçüncü katman **domain code** ile. Karıştırma.

### 2. Jakarta Bean Validation — temel

Standart: JSR 380 (Jakarta Bean Validation 3.0). Implementation: **Hibernate Validator** (Spring Boot default).

**Standard constraint'ler:**

| Annotation | Anlamı |
|---|---|
| `@NotNull` | null olmamalı |
| `@NotBlank` | null veya boş string olmamalı (whitespace trim'leyerek) |
| `@NotEmpty` | null veya empty collection/string olmamalı (trim YAPMAZ) |
| `@Size(min=, max=)` | string/collection length |
| `@Min(value)`, `@Max(value)` | sayı için (long bazlı) |
| `@DecimalMin`, `@DecimalMax` | BigDecimal için |
| `@Positive`, `@PositiveOrZero` | sayı > 0 veya >= 0 |
| `@Negative`, `@NegativeOrZero` | sayı < 0 veya <= 0 |
| `@Digits(integer=, fraction=)` | sayı basamak kontrolü |
| `@Email` | email format (basit regex) |
| `@Pattern(regexp=)` | regex |
| `@Past`, `@PastOrPresent` | tarih geçmişte |
| `@Future`, `@FutureOrPresent` | tarih gelecekte |
| `@AssertTrue`, `@AssertFalse` | boolean |
| `@Valid` | nested object'i de validate et |

### 3. Banking örnekleri — built-in constraint'lerle

```java
public record OpenAccountRequest(
    @NotNull(message = "Owner ID is required")
    UUID ownerId,
    
    @NotBlank(message = "Currency is required")
    @Size(min = 3, max = 3, message = "Currency must be 3 characters")
    @Pattern(regexp = "^[A-Z]{3}$", message = "Currency must be uppercase ISO 4217")
    String currency,
    
    @DecimalMin(value = "0.00", message = "Opening balance cannot be negative")
    @Digits(integer = 19, fraction = 4, message = "Invalid amount format")
    BigDecimal openingBalance
) {}
```

```java
public record TransferRequest(
    @NotNull UUID fromAccountId,
    @NotNull UUID toAccountId,
    
    @NotNull
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    @DecimalMax(value = "999999999.99", message = "Amount exceeds maximum")
    @Digits(integer = 19, fraction = 4)
    BigDecimal amount,
    
    @NotBlank
    @Pattern(regexp = "^[A-Z]{3}$")
    String currency,
    
    @Size(max = 500, message = "Description too long")
    String description
) {}
```

### 4. `@Valid` etkinleştirme

Spring Controller method parameter'ında:

```java
@PostMapping
public AccountResponse openAccount(
    @Valid @RequestBody OpenAccountRequest request
) { ... }
```

`@Valid` olmadan validation annotation'ları **çalışmaz**. Bu junior tuzağı.

### 5. Validation hata handling

Validation başarısız olunca Spring `MethodArgumentNotValidException` fırlatır. Default response:

```json
{
  "timestamp": "...",
  "status": 400,
  "error": "Bad Request",
  "trace": "...",  // stacktrace expose oluyor → tehlike
  "path": "/v1/accounts"
}
```

**Banking için yeterli değil.** Topic 1.7'de RFC 7807 ProblemDetail ile düzgün handle edeceğiz.

Şimdilik basit `@ControllerAdvice`:

```java
@RestControllerAdvice
class ValidationExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handle(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.badRequest().body(Map.of(
            "errors", errors
        ));
    }
}
```

### 6. Custom validation — `@IbanFormat`

Banking'de IBAN (Uluslararası Banka Hesap Numarası) doğrulamak çok yaygın. Custom annotation yaz:

#### Adım 1: Annotation tanımı

```java
package com.mavibank.banking.common.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IbanFormatValidator.class)
public @interface IbanFormat {
    String message() default "Invalid IBAN format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

#### Adım 2: Validator implementation

```java
package com.mavibank.banking.common.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.math.BigInteger;

public class IbanFormatValidator implements ConstraintValidator<IbanFormat, String> {
    
    @Override
    public boolean isValid(String iban, ConstraintValidatorContext context) {
        if (iban == null) return true;  // null kontrolü @NotNull işi
        
        String normalized = iban.replaceAll("\\s+", "").toUpperCase();
        if (normalized.length() < 15 || normalized.length() > 34) return false;
        if (!normalized.matches("^[A-Z]{2}\\d{2}[A-Z0-9]+$")) return false;
        
        // Move first 4 chars to end
        String rearranged = normalized.substring(4) + normalized.substring(0, 4);
        
        // Replace letters with numbers (A=10, B=11, ...)
        StringBuilder numericForm = new StringBuilder();
        for (char c : rearranged.toCharArray()) {
            if (Character.isDigit(c)) {
                numericForm.append(c);
            } else {
                numericForm.append(c - 'A' + 10);
            }
        }
        
        // Validate mod-97 == 1
        try {
            BigInteger value = new BigInteger(numericForm.toString());
            return value.mod(BigInteger.valueOf(97)).intValue() == 1;
        } catch (NumberFormatException e) {
            return false;
        }
    }
}
```

#### Adım 3: Kullanım

```java
public record InternationalTransferRequest(
    @NotBlank @IbanFormat String fromIban,
    @NotBlank @IbanFormat String toIban,
    @NotNull @DecimalMin("0.01") BigDecimal amount,
    @NotBlank @Pattern(regexp = "^[A-Z]{3}$") String currency
) {}
```

### 7. Custom validation — `@TcKimlikNo` (TR'ye özgü)

T.C. Kimlik Numarası 11 haneli, son 2 hane checksum.

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TcKimlikNoValidator.class)
public @interface TcKimlikNo {
    String message() default "Geçersiz T.C. Kimlik Numarası";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class TcKimlikNoValidator implements ConstraintValidator<TcKimlikNo, String> {
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;
        if (value.length() != 11) return false;
        if (!value.matches("\\d{11}")) return false;
        if (value.charAt(0) == '0') return false;
        
        int[] digits = new int[11];
        for (int i = 0; i < 11; i++) {
            digits[i] = Character.getNumericValue(value.charAt(i));
        }
        
        // 10. hane checksum: (1+3+5+7+9. haneler toplamı * 7 - 2+4+6+8. haneler toplamı) mod 10
        int oddSum = digits[0] + digits[2] + digits[4] + digits[6] + digits[8];
        int evenSum = digits[1] + digits[3] + digits[5] + digits[7];
        int digit10 = ((oddSum * 7) - evenSum) % 10;
        if (digit10 < 0) digit10 += 10;
        if (digit10 != digits[9]) return false;
        
        // 11. hane checksum: ilk 10 hane toplamı mod 10
        int totalSum = 0;
        for (int i = 0; i < 10; i++) totalSum += digits[i];
        if (totalSum % 10 != digits[10]) return false;
        
        return true;
    }
}
```

### 8. Cross-field validation — `@AssertTrue` ile

İki field'ı **birlikte** doğrulamak gerekir bazen. Örnek: TransferRequest'te `fromAccountId != toAccountId`.

**Yöntem 1: `@AssertTrue` method on DTO**

```java
public record TransferRequest(
    @NotNull UUID fromAccountId,
    @NotNull UUID toAccountId,
    @NotNull @DecimalMin("0.01") BigDecimal amount,
    @NotBlank @Pattern(regexp = "^[A-Z]{3}$") String currency
) {
    @AssertTrue(message = "From and to accounts must be different")
    @JsonIgnore
    public boolean isDifferentAccounts() {
        return fromAccountId != null && !fromAccountId.equals(toAccountId);
    }
}
```

**Dikkat:** `@JsonIgnore` koy yoksa Jackson `isDifferentAccounts` getter'ını serialize edip JSON'a yazar.

**Yöntem 2: Class-level custom annotation (daha temiz büyük projede)**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DifferentAccountsValidator.class)
public @interface DifferentAccounts {
    String message() default "Source and destination accounts must differ";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class DifferentAccountsValidator 
    implements ConstraintValidator<DifferentAccounts, TransferRequest> {
    
    @Override
    public boolean isValid(TransferRequest req, ConstraintValidatorContext ctx) {
        if (req == null) return true;
        return req.fromAccountId() == null || 
               !req.fromAccountId().equals(req.toAccountId());
    }
}

@DifferentAccounts
public record TransferRequest(...) {}
```

### 9. Validation group'ları — aynı DTO, farklı senaryolar

Bazen aynı DTO **farklı senaryolarda farklı kurallarla** validate edilmeli:

- "Hesap aç" → ownerId zorunlu, accountId yok
- "Hesap güncelle" → accountId zorunlu, ownerId yok

Tek DTO + group'lar:

```java
public interface OnCreate {}
public interface OnUpdate {}

public record AccountRequest(
    @Null(groups = OnCreate.class)
    @NotNull(groups = OnUpdate.class)
    UUID id,
    
    @NotNull(groups = OnCreate.class)
    UUID ownerId,
    
    @NotBlank(groups = OnCreate.class)
    @Pattern(regexp = "^[A-Z]{3}$", groups = OnCreate.class)
    String currency
) {}
```

Controller:

```java
@PostMapping
public AccountResponse create(
    @Validated(OnCreate.class) @RequestBody AccountRequest request
) { ... }

@PutMapping("/{id}")
public AccountResponse update(
    @PathVariable UUID id,
    @Validated(OnUpdate.class) @RequestBody AccountRequest request
) { ... }
```

`@Validated` (Spring) vs `@Valid` (standard) — `@Validated` group destekler.

**Banking pratiği:** Tek DTO + group genelde **karışıklığa** sebep olur. Daha temiz: her senaryo için ayrı DTO. Group'ları biliyor ol ama **kullanırken iki kere düşün.**

### 10. Constraint composition — birleşik annotation

Bir validation kombinasyonu birden fazla yerde tekrarlanıyorsa, kendi composite annotation'ını yaz:

```java
@NotBlank
@Size(min = 3, max = 3)
@Pattern(regexp = "^[A-Z]{3}$")
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {})
public @interface IsoCurrencyCode {
    String message() default "Invalid ISO currency code";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

Kullanım:
```java
public record OpenAccountRequest(
    @NotNull UUID ownerId,
    @IsoCurrencyCode String currency
) {}
```

Üç annotation tek annotation'a düştü. Aynı kombinasyonu 10 DTO'da kullanıyorsan değer.

### 11. Nested validation — `@Valid` ile cascade

DTO'da başka DTO varsa, nested validation:

```java
public record AddressDto(
    @NotBlank String street,
    @NotBlank String city,
    @Pattern(regexp = "^\\d{5}$") String postalCode
) {}

public record CustomerRequest(
    @NotBlank String name,
    @TcKimlikNo String tcKimlikNo,
    @Valid AddressDto address          // ← @Valid cascade
) {}
```

`@Valid` olmadan `address`'in field'ları validate edilmez.

### 12. Programmatic validation — Validator bean'i

Controller dışında validation gerekirse:

```java
@Service
class TransferService {
    
    private final Validator validator;
    
    TransferService(Validator validator) {
        this.validator = validator;
    }
    
    public void someInternalMethod(TransferRequest req) {
        Set<ConstraintViolation<TransferRequest>> violations = validator.validate(req);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
}
```

Banking'de **iç akışta** input gelirse (örn. Kafka consumer'dan), elle validate etmek standart.

### 13. Method-level validation — `@Validated` class-level

```java
@Service
@Validated
class AccountService {
    
    public Account openAccount(
        @NotNull OwnerId ownerId,
        @NotNull Currency currency,
        @DecimalMin("0.00") BigDecimal openingBalance
    ) {
        // ...
    }
}
```

Method parameter validation. `ConstraintViolationException` fırlatır (Spring `@Valid`'in tetiklediği `MethodArgumentNotValidException` ile farklı).

Banking pratiği: **Service layer'da method-level validation tartışmalı.** Çoğu ekip "domain code kendi precondition'ını kontrol eder" der, Bean Validation sadece DTO katmanında.

### 14. Validation message i18n

`messages.properties`:
```
account.balance.insufficient=Yetersiz bakiye: hesapta {0} var, istenilen {1}
account.currency.invalid=Geçersiz para birimi
```

DTO:
```java
@NotBlank(message = "{account.currency.required}")
@Pattern(regexp = "^[A-Z]{3}$", message = "{account.currency.invalid}")
String currency
```

`MessageSource` ve `LocalValidatorFactoryBean` setup'ı:

```java
@Configuration
class ValidationConfig {
    
    @Bean
    MessageSource messageSource() {
        ReloadableResourceBundleMessageSource ms = new ReloadableResourceBundleMessageSource();
        ms.setBasename("classpath:messages");
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }
    
    @Bean
    LocalValidatorFactoryBean validator(MessageSource messageSource) {
        LocalValidatorFactoryBean bean = new LocalValidatorFactoryBean();
        bean.setValidationMessageSource(messageSource);
        return bean;
    }
}
```

TR bankalarında **Türkçe + İngilizce** mesaj desteği yaygın.

### 15. Validation anti-pattern'ler

**Anti-pattern 1: Business logic'i validation'a koymak**

```java
// ❌ Kötü
@AssertTrue
public boolean isWithinTransferLimit() {
    // External service çağrısı yapma!
    return limitService.getDailyLimit(fromAccountId).compareTo(amount) >= 0;
}
```

Bean Validation **stateless** olmalı. External call yapma. State'e dayalı kurallar domain logic'inde.

**Anti-pattern 2: Validation'ı tek satıra yıkmak**

```java
// ❌ Kötü
@Pattern(regexp = "^.{1,500}$")
String description;
```

`@Size(max = 500)` daha okunabilir.

**Anti-pattern 3: `@Email` yetersizliği**

`@Email` annotation basit regex, RFC 5322 tam uygulamaz. Banking'de gerçek email validation için `commons-validator` ya da DNS lookup.

**Anti-pattern 4: Validation'ı atlayarak güvenmek**

```java
// ❌ Tehlikeli
public void process(String userInput) {
    // güveniyorum hiç temizleme
    sql.execute("SELECT * FROM accounts WHERE name = '" + userInput + "'");
}
```

Banking'de **her input** validate + sanitize edilmeli, prepared statement, escape, vb.

---

## Önemli olabilecek araştırma kaynakları

- Jakarta Bean Validation 3.0 spec
- Hibernate Validator documentation
- Spring Boot validation reference
- "Spring Boot Validation" Baeldung serisi
- Apache Commons Validator (IBAN, ISIN, ISBN validators)
- IBAN format spec (ECBS)
- T.C. Kimlik No algoritması (NVI / İçişleri Bakanlığı)
- "Hexagonal Architecture" perspectifinden validation ne nerede olur tartışması

---

## Mini task'ler

### Task 1.6.1 — DTO'lara validation ekle (15 dk)

Topic 1.5'te yazdığın DTO'lara validation annotation'ları ekle:

`OpenAccountRequest`:
- `ownerId`: `@NotNull`
- `currency`: `@NotBlank @Pattern(regexp = "^[A-Z]{3}$")`

`TransferRequest`:
- `fromAccountId`, `toAccountId`: `@NotNull`
- `amount`: `@NotNull @DecimalMin("0.01") @Digits(integer=19, fraction=4)`
- `currency`: `@NotBlank @Pattern(regexp = "^[A-Z]{3}$")`
- `description`: `@Size(max=500)`

### Task 1.6.2 — `@IsoCurrencyCode` composite annotation (30 dk)

`banking/common/validation/IsoCurrencyCode.java` yaz. `@NotBlank + @Size(3,3) + @Pattern("^[A-Z]{3}$")` birleştir. DTO'larında kullanmaya başla.

### Task 1.6.3 — `@IbanFormat` custom validator (60 dk)

Yukarıdaki implementation'ı yaz. Test'leri (sonraki bölümde) yaz. **Algoritmayı kendi başına anlamadan kopyalama** — referans bir IBAN bul, kâğıt üzerinde mod-97'yi hesapla, neden o şekilde çalıştığını **defterine** yaz.

### Task 1.6.4 — `@TcKimlikNo` validator (30 dk)

Yukarıdaki algoritmayı yaz. Test'lerini yaz. Bir geçerli ve geçersiz TC No bul, test et.

### Task 1.6.5 — `@DifferentAccounts` cross-field validator (30 dk)

`TransferRequest`'e ekle. `fromAccountId.equals(toAccountId)` durumunda fail.

Class-level annotation olarak yaz (yukarıdaki Yöntem 2).

### Task 1.6.6 — Validation `@ControllerAdvice` (15 dk)

Geçici basit handler yaz (Topic 1.7'de ProblemDetail ile replace edeceğiz):

```java
@RestControllerAdvice
class ValidationExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.badRequest().body(Map.of(
            "status", 400,
            "errors", fieldErrors
        ));
    }
}
```

### Task 1.6.7 — Manuel validation deneyimi (15 dk)

Bir test endpoint'e `curl` ile invalid request gönder:

```bash
curl -X POST http://localhost:8080/v1/accounts \
  -H "Content-Type: application/json" \
  -d '{"currency": "tr"}'
```

Beklenen: 400 response, validation hatası. Aldığın JSON'u **defterine** yapıştır, neyi yanlış bulduğunu yorumla (currency lowercase, ownerId yok).

---

## Test yazma rehberi

### Test 1.6.1 — `IbanFormatValidatorTest`

```java
class IbanFormatValidatorTest {
    
    private final IbanFormatValidator validator = new IbanFormatValidator();
    
    @ParameterizedTest
    @ValueSource(strings = {
        "TR320010009999901234567890",   // gerçek-format TR IBAN
        "DE89370400440532013000",       // Alman
        "GB29NWBK60161331926819",       // İngiltere
        "FR1420041010050500013M02606"   // Fransa
    })
    void shouldAcceptValidIbans(String iban) {
        assertThat(validator.isValid(iban, null)).isTrue();
    }
    
    @ParameterizedTest
    @ValueSource(strings = {
        "TR320010009999901234567891",   // checksum yanlış
        "TR3200100099999",              // çok kısa
        "tr320010009999901234567890",   // lowercase
        "TR32 0010 0099 9990 1234 5678 90",  // boşluklu - normalize edip kabul edebilir, karara göre
        "TR-32-..."                     // ayraçlı
    })
    void shouldRejectInvalidIbans(String iban) {
        // Bazıları normalization sonrası kabul edilebilir - test'in kararını yansıtsın
        boolean valid = validator.isValid(iban, null);
        // Eğer normalize ediyorsan: 4. test kabul edilebilir
        // Eğer etmiyorsan: hepsi reject
    }
    
    @Test
    void nullShouldBeAllowed() {
        // @NotNull ayrı kontrol — null'a izin ver
        assertThat(validator.isValid(null, null)).isTrue();
    }
}
```

### Test 1.6.2 — `TcKimlikNoValidatorTest`

```java
class TcKimlikNoValidatorTest {
    
    private final TcKimlikNoValidator validator = new TcKimlikNoValidator();
    
    @ParameterizedTest
    @ValueSource(strings = {
        "10000000146",  // sample valid (algoritma çıkışı)
        // Gerçek test verisi internet'te bulabilirsin — algorithm consistent
    })
    void shouldAcceptValidTcNos(String tcNo) {
        assertThat(validator.isValid(tcNo, null)).isTrue();
    }
    
    @ParameterizedTest
    @ValueSource(strings = {
        "00000000000",     // 0 ile başlar
        "1234567890",      // 10 hane
        "123456789012",    // 12 hane
        "abcdefghijk",     // harf
        "12345678901"      // checksum hatalı (büyük ihtimal)
    })
    void shouldRejectInvalidTcNos(String tcNo) {
        assertThat(validator.isValid(tcNo, null)).isFalse();
    }
    
    @Test
    void nullShouldBeAllowed() {
        assertThat(validator.isValid(null, null)).isTrue();
    }
}
```

### Test 1.6.3 — DTO-level validation (`@SpringBootTest` veya bağımsız `Validator`)

```java
class OpenAccountRequestValidationTest {
    
    private static Validator validator;
    
    @BeforeAll
    static void setUpValidator() {
        try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
            validator = factory.getValidator();
        }
    }
    
    @Test
    void validRequestShouldPass() {
        var req = new OpenAccountRequest(UUID.randomUUID(), "TRY");
        Set<ConstraintViolation<OpenAccountRequest>> violations = validator.validate(req);
        assertThat(violations).isEmpty();
    }
    
    @Test
    void missingOwnerIdShouldFail() {
        var req = new OpenAccountRequest(null, "TRY");
        var violations = validator.validate(req);
        assertThat(violations).hasSize(1);
        assertThat(violations.iterator().next().getPropertyPath().toString()).isEqualTo("ownerId");
    }
    
    @Test
    void lowercaseCurrencyShouldFail() {
        var req = new OpenAccountRequest(UUID.randomUUID(), "try");
        var violations = validator.validate(req);
        assertThat(violations).isNotEmpty();
    }
    
    @Test
    void shortCurrencyShouldFail() {
        var req = new OpenAccountRequest(UUID.randomUUID(), "TR");
        var violations = validator.validate(req);
        assertThat(violations).isNotEmpty();
    }
}
```

### Test 1.6.4 — Controller validation integration (`@WebMvcTest`)

Topic 1.5'te yazdığın `AccountControllerTest`'i genişlet:

```java
@Test
void shouldReturn400WithFieldErrorWhenCurrencyInvalid() throws Exception {
    mockMvc.perform(post("/v1/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {
                  "ownerId": "%s",
                  "currency": "tr"
                }
                """.formatted(UUID.randomUUID())))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors.currency").exists());
}
```

### Test 1.6.5 — Cross-field validation

```java
@Test
void transferRequestShouldRejectSameAccountIds() throws Exception {
    UUID sameId = UUID.randomUUID();
    
    mockMvc.perform(post("/v1/transfers")
            .header("Idempotency-Key", UUID.randomUUID().toString())
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {
                  "fromAccountId": "%s",
                  "toAccountId": "%s",
                  "amount": "100.00",
                  "currency": "TRY"
                }
                """.formatted(sameId, sameId)))
        .andExpect(status().isBadRequest());
}
```

---

## Claude-verify prompt

```
Aşağıdaki Bean Validation kodumu banking-grade kriterlere göre değerlendir. 
Sadece eksik veya yanlışları işaretle, kod yazma:

1. DTO validation kapsama:
   - Tüm request DTO'larda field'lara uygun validation annotation'ı var mı?
   - `@NotNull` ile `@NotBlank` doğru ayırt edilmiş mi (string için NotBlank)?
   - BigDecimal field'lar için `@DecimalMin`, `@Digits` kullanılmış mı (Min/Max DEĞİL)?
   - Currency code için `@Pattern("^[A-Z]{3}$")` veya composite annotation var mı?

2. Custom validator: IBAN
   - `@IbanFormat` annotation tanımı doğru mu (target, retention, validatedBy)?
   - `IbanFormatValidator` mod-97 algoritmasını implement ediyor mu?
   - Null kabul ediyor mu (`@NotNull` ayrı sorumluluktur)?
   - Length kontrolü 15-34 arasında mı?

3. Custom validator: TC Kimlik No
   - 11 hane kontrolü var mı?
   - İlk hane 0 olmaması kontrol ediliyor mu?
   - 10. ve 11. hane checksum algoritması uygulanmış mı?
   - Test edilmiş mi (gerçek geçerli + geçersiz)?

4. Composite annotation (`@IsoCurrencyCode`):
   - Annotation üstüne `@NotBlank`, `@Size`, `@Pattern` doğru kombine edilmiş mi?
   - Validator class belirtilmemiş (`@Constraint(validatedBy = {})`) mi (composite olduğu için)?

5. Cross-field validation:
   - `TransferRequest`'te `fromAccountId == toAccountId` kontrolü yapılıyor mu?
   - `@AssertTrue` ile yapılıyorsa `@JsonIgnore` eklenmiş mi (getter serialize edilmesin)?
   - Veya class-level custom annotation tercih edilmiş mi?

6. Controller setup:
   - Controller method'larında `@Valid` annotation'ı var mı?
   - `@RequestBody` doğru kullanılmış mı?
   - Validation exception handler (`@RestControllerAdvice`) yazılmış mı?
   - Hata response'unda field bazlı detay var mı?

7. Anti-pattern kontrolü:
   - Validation içinde external service çağrısı var mı? (Olmamalı — Bean Validation stateless)
   - Business kuralı validation'a yazılmış mı (örn. "yeterli bakiye var mı")? (Olmamalı — domain logic'e ait)
   - `@Email` ile gerçek email doğrulaması bekleniyor mu? (Yetersiz, banking için commons-validator)

8. Test coverage:
   - Custom validator'lar için unit test var mı?
   - Geçerli ve geçersiz örnekler `@ParameterizedTest` ile test edilmiş mi?
   - DTO-level validation Validator factory ile test edilmiş mi?
   - Controller test'inde validation 400 response'u test edilmiş mi?
   - Null input için validator'ın true döndürdüğü test edilmiş mi?

9. Banking-specific:
   - Para field'larında `@Digits(integer=19, fraction=4)` veya benzer kesinlik kontrolü var mı?
   - Description gibi uzun text field'lar için `@Size(max=...)` var mı?
   - Currency code regex (`^[A-Z]{3}$`) ile zorlanmış mı?

Her madde için PASS / FAIL / EKSIK işaretle. Kod yazmadan, sadece neyi düzeltmem 
gerektiğini söyle.
```

---

## Tamamlama kriterleri

- [ ] Tüm request DTO'larda field-level validation annotation'ları
- [ ] `@IsoCurrencyCode` composite annotation yazıldı ve kullanılıyor
- [ ] `@IbanFormat` validator yazıldı, mod-97 algoritması anlaşıldı (defterimde yazılı)
- [ ] `@TcKimlikNo` validator yazıldı
- [ ] `TransferRequest`'te cross-field validation (different accounts)
- [ ] Validation `@ControllerAdvice` ile 400 response veriyor
- [ ] Curl ile manuel test yapıp 400 + field errors aldım
- [ ] Custom validator'lar için `@ParameterizedTest`'lerle test yazıldı
- [ ] Controller validation'ı `@WebMvcTest` ile test edildi
- [ ] Bean Validation içinde external call YOK
- [ ] Business rule validation'a sızdırılmamış (domain'de durur)

---

## Defter notları

1. "3 validation katmanı (syntactic, semantic, domain) farkları: ____."
2. "`@NotNull` ile `@NotBlank` farkı: ____."
3. "Custom validator yazmak için 3 adım: ____."
4. "IBAN mod-97 algoritması neden çalışır (kısaca): ____."
5. "T.C. Kimlik No checksum'ı neden son 2 hane: ____."
6. "Bean Validation içinde external call neden yapılmaz: ____."
7. "`@AssertTrue` ile `@JsonIgnore` neden birlikte: ____."
8. "Composite annotation ne işe yarar (örnek senaryo): ____."
9. "`@Valid` ile `@Validated` farkı: ____."
10. "Banking'de neden client-side validation'a güvenilmez: ____."
