# Topic 1.7 — Hata Yönetimi: RFC 7807 ProblemDetail + @ControllerAdvice

## Hedef

Hatalar için **tutarlı, standartlara uyumlu ve information-leak'siz** response'lar üretmek. RFC 7807 ProblemDetail standardını uygulamak. Spring 6'nın `ProblemDetail` API'sini kullanmak. Domain exception'larını HTTP error'a haritalamak. Banking-specific error catalog'u tasarlamak.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~4.5 saat

## Önbilgi

- Topic 1.1-1.6 tamamlandı
- Validation handler'ı çalışıyor (Topic 1.6'nın basit handler'ı)
- Domain exception'larını biliyorsun (`InsufficientFundsException`, `CurrencyMismatchException`)

---

## Kavramlar

### 1. Hatanın anatomisi — kötü vs iyi

**Kötü hata (default Spring Boot):**

```json
{
  "timestamp": "2025-05-12T10:30:00.123+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "trace": "java.lang.NullPointerException...\n  at com.mavibank...",  // ← stacktrace
  "message": "Cannot invoke \"...\"",
  "path": "/v1/transfers"
}
```

Sorunlar:
- **Stacktrace expose** — internal class isimleri, package yapısı sızıyor
- **Generic 500** — istemci ne yapsın bilmiyor
- **Mesaj İngilizce ve teknik** — son kullanıcıya gösterilemez
- **Hata kodu yok** — programmatic karşılama imkânsız
- **Mantıksal kategorisi yok** — retry mi, kullanıcıya soru mu, support mu?

**İyi hata (RFC 7807 banking-uyarlı):**

```json
{
  "type": "https://api.mavibank.com/problems/insufficient-funds",
  "title": "Yetersiz bakiye",
  "status": 422,
  "detail": "Hesabınızdaki bakiye yapmak istediğiniz işlem için yetersiz.",
  "instance": "/v1/transfers/abc-123",
  "code": "ACCOUNT_INSUFFICIENT_FUNDS",
  "accountId": "550e8400-...",
  "availableBalance": "500.00",
  "requestedAmount": "600.00",
  "currency": "TRY",
  "traceId": "abc12345-..."
}
```

İyi tarafları:
- **Standart** (RFC 7807)
- **Stacktrace yok** (sadece traceId loglarda)
- **Kullanıcıya uygun mesaj** (Türkçe, anlaşılır)
- **Programmatic code** (`ACCOUNT_INSUFFICIENT_FUNDS`)
- **Extra structured data** (availableBalance, requestedAmount)
- **traceId** — log'larda karşılığını bulmak için

### 2. RFC 7807 — Problem Details for HTTP APIs

[RFC 7807](https://tools.ietf.org/html/rfc7807) — HTTP API'larında hata response standartı.

**Zorunlu/Standart alanlar:**

- `type` — URI, hata tipinin tanımına link (örn. `https://api.mavibank.com/problems/insufficient-funds`)
- `title` — kısa, insan-okur summary (örn. "Insufficient funds")
- `status` — HTTP status code (response status ile aynı)
- `detail` — bu özel hataya özel açıklama (kullanıcıya gösterilebilir)
- `instance` — bu hatanın oluştuğu kaynak URI'si (request path)

**Extension** — istediğin alanı ekleyebilirsin (`availableBalance`, `code`, `traceId`).

`Content-Type: application/problem+json` (önemli — client bunu görüp özel handle edebilir).

### 3. Spring 6 `ProblemDetail` API

Spring 6+ (Spring Boot 3+) `org.springframework.http.ProblemDetail` class'ı RFC 7807 implementasyonu.

```java
ProblemDetail problem = ProblemDetail.forStatusAndDetail(
    HttpStatus.UNPROCESSABLE_ENTITY,
    "Hesabınızdaki bakiye yapmak istediğiniz işlem için yetersiz."
);
problem.setType(URI.create("https://api.mavibank.com/problems/insufficient-funds"));
problem.setTitle("Yetersiz bakiye");
problem.setInstance(URI.create("/v1/transfers/abc-123"));
problem.setProperty("code", "ACCOUNT_INSUFFICIENT_FUNDS");
problem.setProperty("availableBalance", "500.00");
problem.setProperty("requestedAmount", "600.00");
problem.setProperty("currency", "TRY");
```

Otomatik `application/problem+json` content type.

### 4. `@RestControllerAdvice` — global exception handler

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    
    @ExceptionHandler(InsufficientFundsException.class)
    ProblemDetail handle(InsufficientFundsException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY,
            "Hesabınızdaki bakiye yetersiz."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/insufficient-funds"));
        problem.setTitle("Yetersiz bakiye");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "ACCOUNT_INSUFFICIENT_FUNDS");
        problem.setProperty("accountId", ex.getAccountId().value());
        problem.setProperty("availableBalance", ex.getAvailable().amount().toPlainString());
        problem.setProperty("requestedAmount", ex.getRequested().amount().toPlainString());
        problem.setProperty("currency", ex.getRequested().currency().getCurrencyCode());
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handle(MethodArgumentNotValidException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "İstek doğrulama hatalı. Lütfen alanları kontrol edin."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/validation-failed"));
        problem.setTitle("Doğrulama hatası");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "VALIDATION_FAILED");
        
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        problem.setProperty("fieldErrors", fieldErrors);
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
    
    @ExceptionHandler(AccountNotFoundException.class)
    ProblemDetail handle(AccountNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            "Hesap bulunamadı."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/account-not-found"));
        problem.setTitle("Hesap bulunamadı");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "ACCOUNT_NOT_FOUND");
        problem.setProperty("accountId", ex.getAccountId().value());
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
    
    @ExceptionHandler(CurrencyMismatchException.class)
    ProblemDetail handle(CurrencyMismatchException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY,
            "Para birimi uyuşmazlığı."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/currency-mismatch"));
        problem.setTitle("Para birimi uyuşmazlığı");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "CURRENCY_MISMATCH");
        problem.setProperty("expected", ex.getExpected().getCurrencyCode());
        problem.setProperty("actual", ex.getActual().getCurrencyCode());
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
    
    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error", ex);     // ← stacktrace log'a
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "Beklenmedik bir hata oluştu. Lütfen daha sonra tekrar deneyin."   // ← stacktrace YOK
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/internal-error"));
        problem.setTitle("Sunucu hatası");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "INTERNAL_ERROR");
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
    
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
}
```

### 5. Exception hierarchy — banking için tasarım

```
RuntimeException
  └── BankingException                    (banking-domain root)
       ├── AccountException
       │    ├── AccountNotFoundException
       │    ├── AccountClosedException
       │    └── AccountFrozenException
       ├── BalanceException
       │    ├── InsufficientFundsException
       │    └── DailyLimitExceededException
       ├── CurrencyException
       │    └── CurrencyMismatchException
       ├── TransferException
       │    ├── SameAccountTransferException
       │    └── TransferLimitExceededException
       └── ValidationException
            └── InvalidCurrencyException
```

Her exception bir HTTP status'a karşılık geliyor:

| Exception | Status |
|---|---|
| `AccountNotFoundException` | 404 Not Found |
| `AccountClosedException` | 409 Conflict (state conflict) |
| `AccountFrozenException` | 403 Forbidden (yetki sorunu gibi düşün) |
| `InsufficientFundsException` | 422 Unprocessable Entity |
| `CurrencyMismatchException` | 422 |
| `DailyLimitExceededException` | 422 |
| `InvalidCurrencyException` | 400 |

Exception'lar **domain'de** (`banking/.../domain/exception/`) durur, handler `adapter/in/web/`'de.

### 6. `BankingException` — base class

```java
package com.mavibank.banking.common.domain.exception;

public abstract class BankingException extends RuntimeException {
    
    private final String code;
    
    protected BankingException(String code, String message) {
        super(message);
        this.code = code;
    }
    
    protected BankingException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }
    
    public String getCode() {
        return code;
    }
}
```

Concrete exception örnek:

```java
public class InsufficientFundsException extends BankingException {
    
    private final AccountId accountId;
    private final Money available;
    private final Money requested;
    
    public InsufficientFundsException(AccountId accountId, Money available, Money requested) {
        super("ACCOUNT_INSUFFICIENT_FUNDS",
              "Account %s has %s, requested %s".formatted(
                  accountId.value(), available, requested));
        this.accountId = accountId;
        this.available = available;
        this.requested = requested;
    }
    
    public AccountId getAccountId() { return accountId; }
    public Money getAvailable() { return available; }
    public Money getRequested() { return requested; }
}
```

**Önemli detay:** Exception message **internal log için**. Kullanıcıya gösterilecek mesaj `ProblemDetail.detail`'da, **i18n + lokalize**.

### 7. Error code catalog — banking standartı

Her hata için **stabil, programmatic code**:

```java
public final class ErrorCodes {
    public static final String ACCOUNT_NOT_FOUND = "ACCOUNT_NOT_FOUND";
    public static final String ACCOUNT_CLOSED = "ACCOUNT_CLOSED";
    public static final String ACCOUNT_FROZEN = "ACCOUNT_FROZEN";
    public static final String INSUFFICIENT_FUNDS = "ACCOUNT_INSUFFICIENT_FUNDS";
    public static final String CURRENCY_MISMATCH = "CURRENCY_MISMATCH";
    public static final String DAILY_LIMIT_EXCEEDED = "DAILY_LIMIT_EXCEEDED";
    public static final String SAME_ACCOUNT_TRANSFER = "SAME_ACCOUNT_TRANSFER";
    public static final String VALIDATION_FAILED = "VALIDATION_FAILED";
    public static final String INVALID_CURRENCY = "INVALID_CURRENCY";
    public static final String IDEMPOTENCY_CONFLICT = "IDEMPOTENCY_CONFLICT";
    public static final String INTERNAL_ERROR = "INTERNAL_ERROR";
    
    private ErrorCodes() {}
}
```

Client (mobile/web app) bu code'lara göre özel ekran açar (örn. "Yetersiz bakiye" için para yükleme butonu).

**Code naming kuralı:**
- UPPER_SNAKE_CASE
- Stabil — bir kez yayınlandı mı **asla değiştirme**
- Açıklayıcı, prefix kategori (`ACCOUNT_`, `TRANSFER_`, `CARD_`)

### 8. Information leak — neyi göstermeyeceğin

Banking'de **asla** API response'a sızdırılmayacak şeyler:

- **Stacktrace** (production'da)
- **Internal class/package isimleri**
- **DB schema isimleri** ("`column accounts.balance_amount` doesn't exist")
- **JVM versiyonu, OS bilgisi**
- **Connection string, hostname**
- **Başka kullanıcının bilgileri** (örn. "Account 123 belongs to user X — but you're user Y")
- **Sistemde bir hesabın varlığı/yokluğu** (account enumeration attack)

**Production exception handler stratejisi:** `Exception.class` yakalanır, log'a tam detail, response'a generic mesaj + traceId.

### 9. Information leak güvenlik tuzağı — account enumeration

```java
@ExceptionHandler(AccountNotFoundException.class)
ProblemDetail handle(AccountNotFoundException ex, ...) {
    ProblemDetail p = ProblemDetail.forStatusAndDetail(NOT_FOUND, "Hesap bulunamadı");
    p.setProperty("accountId", ex.getAccountId().value());  // ← TEHLİKE
    return p;
}
```

Saldırgan farklı `accountId`'ler dener, hangileri 404 hangileri 401/403 dönüyor diye bakar. Sonra var olan hesap ID'lerini öğrenir.

**Çözüm:** Her zaman authenticate'i kontrol et, authentication başarısızsa **401 önce dön**. Authenticated user'ın **kendi** hesabını sorgulamasa bile aynı tipte yanıt ver:

```java
// Better
@ExceptionHandler({AccountNotFoundException.class, AccessDeniedException.class})
ProblemDetail handleNotFoundOrForbidden(...) {
    return ProblemDetail.forStatusAndDetail(NOT_FOUND, "İstenilen kaynak bulunamadı.");
    // Erişim yok mu, kaynak yok mu — ayrım yapma
}
```

Phase 8 (Security) topic'inde tam handle edeceğiz.

### 10. Logging vs response — ayrım

Bir exception'ın iki muhatabı var:
- **Client** → API response (kullanıcı dostu)
- **Sysadmin/Developer** → log dosyaları (teknik detay)

```java
@ExceptionHandler(InsufficientFundsException.class)
ProblemDetail handle(InsufficientFundsException ex, HttpServletRequest request) {
    log.warn("Insufficient funds: account={}, available={}, requested={}",
        ex.getAccountId(), ex.getAvailable(), ex.getRequested());
    
    // Response — user-friendly
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(...);
    return problem;
}

@ExceptionHandler(Exception.class)
ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
    log.error("Unexpected error for request {}", request.getRequestURI(), ex);  // stacktrace
    
    // Response — generic, no leak
    return ProblemDetail.forStatusAndDetail(INTERNAL_SERVER_ERROR, "Beklenmedik bir hata.");
}
```

### 11. Trace ID propagation

Production'da bir kullanıcı "hata aldım, bu kod ne?" dediğinde:

1. Kullanıcı `traceId` veya `code` paylaşır
2. Sysadmin log'larda `traceId`'yi arar
3. Tam stacktrace bulur, sorunu çözer

`traceId` her request'in başında üretilir, log'larda MDC'de tutulur, response'a da yazılır.

**Filter ile traceId üretmek:**

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class TraceIdFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String traceId = ((HttpServletRequest) req).getHeader("X-Trace-Id");
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString();
        }
        MDC.put("traceId", traceId);
        ((HttpServletResponse) res).setHeader("X-Trace-Id", traceId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("traceId");
        }
    }
}
```

Logback config'ine MDC ekle:
```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%X{traceId:-no-trace}] %-5level [%thread] %logger{36} - %msg%n"
```

Phase 9'da (Observability) full distributed tracing OpenTelemetry ile yapacağız. Phase 1'de basit traceId yeter.

### 12. Spring Boot 3 — ResponseEntityExceptionHandler

Spring Boot 3+ built-in Spring exception'ları (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, vb.) için `ResponseEntityExceptionHandler` extend etmek daha temiz:

```java
@RestControllerAdvice
class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    // Override built-in handlers if needed
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
        MethodArgumentNotValidException ex,
        HttpHeaders headers,
        HttpStatusCode status,
        WebRequest request) {
        
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "İstek doğrulama hatalı."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/validation-failed"));
        problem.setTitle("Doğrulama hatası");
        problem.setProperty("code", "VALIDATION_FAILED");
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(e ->
            errors.put(e.getField(), e.getDefaultMessage())
        );
        problem.setProperty("fieldErrors", errors);
        problem.setProperty("traceId", MDC.get("traceId"));
        
        return ResponseEntity.badRequest()
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }
    
    // Domain exception'ları için ayrı @ExceptionHandler'lar
    @ExceptionHandler(InsufficientFundsException.class)
    ResponseEntity<ProblemDetail> handle(InsufficientFundsException ex, HttpServletRequest req) {
        // ...
    }
}
```

### 13. Error response example collection — kataloğun

`docs/error-catalog.md` dosyası tut. Her error code için:

```markdown
## ACCOUNT_INSUFFICIENT_FUNDS
**Status:** 422 Unprocessable Entity
**Description:** Hesapta yapılmak istenen işlem için yeterli bakiye yok.
**HTTP Response Example:**
```json
{
  "type": "https://api.mavibank.com/problems/insufficient-funds",
  "title": "Yetersiz bakiye",
  "status": 422,
  "detail": "Hesabınızdaki bakiye yapmak istediğiniz işlem için yetersiz.",
  "code": "ACCOUNT_INSUFFICIENT_FUNDS",
  "accountId": "550e8400-...",
  "availableBalance": "500.00",
  "requestedAmount": "600.00",
  "currency": "TRY",
  "traceId": "abc12345"
}
```

**Client handling:** Show "Insufficient funds" screen with "Top up" button.
**Retry:** Not automatic — user action required.
```

Banking ekiplerinde error catalog **publicly documented** ve client/partner ekipleri buradan referans alır.

### 14. Logging levels — banking pratiği

Hangi exception hangi log level'da?

| Exception | Log level |
|---|---|
| Domain validation hatası (`InsufficientFundsException`) | WARN |
| 4xx (client hatası) | INFO veya DEBUG |
| 500 unexpected | ERROR + stacktrace |
| Security violation (`AccessDeniedException`) | WARN veya ERROR (security team monitoring) |
| Recoverable infra issue (DB connection timeout, retried) | WARN |
| Unrecoverable | ERROR |

**Kural:** ERROR sadece operator'ın bakması gereken durumlar için. 422 her log'a ERROR yazarsan log'lar yararsızlaşır.

### 15. Exception throwing kuralları — banking

**Banking exception throwing 5 kuralı:**

1. **Domain-specific exception fırlat**, generic `RuntimeException` değil
2. **Context bilgisi ekle** (accountId, amount) — handler kullanabilsin
3. **Unchecked exception** kullan (Java checked exception'lar boilerplate yaratır, dilbilimsel olarak da banking domain'inde işe yaramaz)
4. **Exception'a ToString implement etme** sıkı tut — sensitive bilgi sızıyor mu kontrol
5. **Stack trace'i log'a yaz ama response'a yazma**

---

## Önemli olabilecek araştırma kaynakları

- RFC 7807 — "Problem Details for HTTP APIs" (full text)
- Spring `ProblemDetail` Javadoc
- Spring `ResponseEntityExceptionHandler` source code (Spring 6'da incelenmeli)
- "Designing Web APIs" — error handling chapter
- OWASP "Improper Error Handling" guide
- "Information Disclosure" OWASP A01:2021 öncesi
- Stripe API documentation — error model (banking reference)
- Anthropic / OpenAI API error model (modern API tasarım örneği)
- SLF4J MDC documentation
- Spring Boot 3 Migration Guide — `ProblemDetail` adoption section

---

## Mini task'ler

### Task 1.7.1 — Domain exception'larını yaz (30 dk)

`banking/common/domain/exception/BankingException.java` (abstract base).

`banking/account/domain/exception/`:
- `AccountNotFoundException(AccountId)`
- `AccountClosedException(AccountId)`
- `AccountFrozenException(AccountId)`
- `InsufficientFundsException(AccountId, Money available, Money requested)`

`banking/common/domain/exception/`:
- `CurrencyMismatchException(Currency expected, Currency actual)`

`banking/transfer/domain/exception/`:
- `SameAccountTransferException(AccountId)`

Hepsi `BankingException` extend etsin, `code` field'ı set etsin.

### Task 1.7.2 — `ErrorCodes` catalog class'ı (10 dk)

`banking/common/error/ErrorCodes.java` yaz (yukarıda örnek). Tüm error code'ları tek yerde topla.

### Task 1.7.3 — `GlobalExceptionHandler` yaz (60 dk)

`banking/common/adapter/in/web/GlobalExceptionHandler.java`:

- `extends ResponseEntityExceptionHandler`
- `@RestControllerAdvice`
- `@ExceptionHandler` her domain exception için
- Override `handleMethodArgumentNotValid` (validation 400)
- `@ExceptionHandler(Exception.class)` catch-all → 500, stacktrace log, generic response

Her handler:
- `ProblemDetail` döndür
- `type`, `title`, `detail`, `instance` set
- `code` property
- `traceId` property (MDC'den)
- Domain-specific property'ler (accountId, available, requested)

### Task 1.7.4 — `TraceIdFilter` (20 dk)

`banking/common/adapter/in/web/filter/TraceIdFilter.java` yaz. `@Component @Order(HIGHEST_PRECEDENCE)` ile en başta çalışsın.

Logback pattern'ini güncelle:
```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%X{traceId:-no-trace}] %-5level [%thread] %logger{36} - %msg%n"
```

Bir endpoint çağır, response'da `X-Trace-Id` header'ını gör. Log'da aynı traceId'nin geçtiğini doğrula.

### Task 1.7.5 — Tamamlama: tüm exception'ları map et (30 dk)

Bu listede her birini handle et:

- `InsufficientFundsException` → 422 + tüm detay
- `AccountNotFoundException` → 404 + accountId
- `AccountClosedException` → 409 (state conflict)
- `AccountFrozenException` → 403
- `CurrencyMismatchException` → 422
- `SameAccountTransferException` → 422
- `InvalidCurrencyException` → 400
- `HttpMessageNotReadableException` (JSON parse hatası — Spring built-in) → 400
- `MethodArgumentTypeMismatchException` (UUID parse fail) → 400
- `NoHandlerFoundException` (404 endpoint) → 404
- `Exception.class` (catch-all) → 500

Her biri için **ayrı `@ExceptionHandler` method** yaz.

### Task 1.7.6 — Error catalog dökümanı (15 dk)

`docs/error-catalog.md` yaz. Yukarıdaki örnek formatı kullan. **Her ErrorCode için** bir bölüm. Banking'in real-life dokümantasyonu bu — alışkanlık kazan.

### Task 1.7.7 — Manuel test (15 dk)

Curl ile error scenarios test et:

```bash
# 1. Validation error (currency invalid)
curl -X POST http://localhost:8080/v1/accounts \
  -H "Content-Type: application/json" \
  -d '{"currency": "tr"}'

# 2. Not found
curl http://localhost:8080/v1/accounts/00000000-0000-0000-0000-000000000000

# 3. Bad JSON
curl -X POST http://localhost:8080/v1/accounts \
  -H "Content-Type: application/json" \
  -d 'not even json'

# 4. UUID parse fail
curl http://localhost:8080/v1/accounts/not-a-uuid
```

Her response için:
- Content-Type `application/problem+json` mı?
- `type`, `title`, `status`, `detail`, `instance` doğru mu?
- `code` ve `traceId` var mı?
- 500 hatasında stacktrace **YOK** mu (yalnızca log'da)?

Sonuçları **defterine** yapıştır.

---

## Test yazma rehberi

### Test 1.7.1 — Exception handler integration (`@WebMvcTest`)

```java
@WebMvcTest(controllers = {AccountController.class, GlobalExceptionHandler.class})
class GlobalExceptionHandlerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private OpenAccountUseCase openAccountUseCase;
    
    @MockBean
    private GetAccountUseCase getAccountUseCase;
    
    @MockBean
    private AccountWebMapper mapper;
    
    @Test
    void insufficientFundsShouldReturn422() throws Exception {
        UUID accountId = UUID.randomUUID();
        when(getAccountUseCase.execute(any())).thenThrow(
            new InsufficientFundsException(
                new AccountId(accountId),
                Money.of("500.00", "TRY"),
                Money.of("600.00", "TRY")
            )
        );
        
        mockMvc.perform(get("/v1/accounts/" + accountId))
            .andExpect(status().isUnprocessableEntity())
            .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
            .andExpect(jsonPath("$.type").value("https://api.mavibank.com/problems/insufficient-funds"))
            .andExpect(jsonPath("$.title").value("Yetersiz bakiye"))
            .andExpect(jsonPath("$.status").value(422))
            .andExpect(jsonPath("$.code").value("ACCOUNT_INSUFFICIENT_FUNDS"))
            .andExpect(jsonPath("$.accountId").value(accountId.toString()))
            .andExpect(jsonPath("$.availableBalance").value("500.00"))
            .andExpect(jsonPath("$.requestedAmount").value("600.00"))
            .andExpect(jsonPath("$.currency").value("TRY"))
            .andExpect(jsonPath("$.traceId").exists());
    }
    
    @Test
    void accountNotFoundShouldReturn404() throws Exception {
        UUID accountId = UUID.randomUUID();
        when(getAccountUseCase.execute(any())).thenThrow(
            new AccountNotFoundException(new AccountId(accountId))
        );
        
        mockMvc.perform(get("/v1/accounts/" + accountId))
            .andExpect(status().isNotFound())
            .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
            .andExpect(jsonPath("$.code").value("ACCOUNT_NOT_FOUND"));
    }
    
    @Test
    void validationErrorShouldReturn400() throws Exception {
        mockMvc.perform(post("/v1/accounts")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isBadRequest())
            .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
            .andExpect(jsonPath("$.code").value("VALIDATION_FAILED"))
            .andExpect(jsonPath("$.fieldErrors").exists());
    }
    
    @Test
    void invalidJsonShouldReturn400() throws Exception {
        mockMvc.perform(post("/v1/accounts")
                .contentType(MediaType.APPLICATION_JSON)
                .content("not json"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.title").exists());
    }
    
    @Test
    void invalidUuidShouldReturn400() throws Exception {
        mockMvc.perform(get("/v1/accounts/not-a-uuid"))
            .andExpect(status().isBadRequest());
    }
    
    @Test
    void unexpectedExceptionShouldReturn500WithoutStacktrace() throws Exception {
        when(getAccountUseCase.execute(any())).thenThrow(
            new RuntimeException("Unexpected DB error with sensitive info")
        );
        
        UUID id = UUID.randomUUID();
        mockMvc.perform(get("/v1/accounts/" + id))
            .andExpect(status().isInternalServerError())
            .andExpect(jsonPath("$.code").value("INTERNAL_ERROR"))
            .andExpect(jsonPath("$.detail").doesNotExist()
                .when(s -> s.contains("Unexpected DB error")))   // Sensitive bilgi LEAK OLMAMALI
            .andExpect(jsonPath("$.detail").value("Beklenmedik bir hata oluştu. Lütfen daha sonra tekrar deneyin."));
    }
    
    @Test
    void responseShouldIncludeTraceId() throws Exception {
        UUID id = UUID.randomUUID();
        when(getAccountUseCase.execute(any())).thenThrow(
            new AccountNotFoundException(new AccountId(id))
        );
        
        mockMvc.perform(get("/v1/accounts/" + id))
            .andExpect(header().exists("X-Trace-Id"))
            .andExpect(jsonPath("$.traceId").exists());
    }
}
```

### Test 1.7.2 — `TraceIdFilterTest`

```java
class TraceIdFilterTest {
    
    @Test
    void shouldGenerateNewTraceIdWhenNotProvided() throws Exception {
        TraceIdFilter filter = new TraceIdFilter();
        var req = new MockHttpServletRequest();
        var res = new MockHttpServletResponse();
        var chain = new MockFilterChain();
        
        filter.doFilter(req, res, chain);
        
        String traceId = res.getHeader("X-Trace-Id");
        assertThat(traceId).isNotNull();
        assertThat(UUID.fromString(traceId)).isNotNull();   // valid UUID
    }
    
    @Test
    void shouldPreserveProvidedTraceId() throws Exception {
        TraceIdFilter filter = new TraceIdFilter();
        var req = new MockHttpServletRequest();
        req.addHeader("X-Trace-Id", "incoming-trace-id");
        var res = new MockHttpServletResponse();
        var chain = new MockFilterChain();
        
        filter.doFilter(req, res, chain);
        
        assertThat(res.getHeader("X-Trace-Id")).isEqualTo("incoming-trace-id");
    }
    
    @Test
    void shouldCleanUpMdcAfterRequest() throws Exception {
        TraceIdFilter filter = new TraceIdFilter();
        var req = new MockHttpServletRequest();
        var res = new MockHttpServletResponse();
        var chain = new MockFilterChain();
        
        filter.doFilter(req, res, chain);
        
        assertThat(MDC.get("traceId")).isNull();
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki hata yönetimi kodumu banking-grade kriterlere göre değerlendir. Sadece 
eksik veya yanlışları işaretle, kod yazma:

1. Exception hierarchy:
   - `BankingException` base abstract class var mı?
   - Her domain exception code field'ı ile yapılandırılmış mı?
   - Generic `RuntimeException` direkt fırlatılan yer var mı? (Olmamalı)
   - Exception'lar domain paketinde mi?
   - Context bilgisi (accountId, amount, vb.) exception field'larına eklenmiş mi?

2. ProblemDetail / RFC 7807:
   - `ProblemDetail` API'si Spring 6'dan kullanılmış mı (custom error response class DEĞİL)?
   - `type` field'ı stable URL mu (api.mavibank.com/problems/...)?
   - `title` insan-okuyabilir mi?
   - `detail` kullanıcıya gösterilebilir mi (teknik jargon, internal info YOK)?
   - `instance` request path'i içeriyor mu?
   - `code` property'si her response'ta var mı?
   - `traceId` property'si var mı?

3. Status code'lar:
   - InsufficientFunds → 422 mi?
   - CurrencyMismatch → 422 mi?
   - AccountNotFound → 404 mü?
   - AccountClosed → 409 mu?
   - Validation error → 400 mü?
   - Unexpected → 500 mü?

4. Information leak:
   - 500 response'unda stacktrace VAR MI? (Olmamalı)
   - 500 response'unda exception message VAR MI? (Olmamalı — generic mesaj)
   - DB schema/internal class isimleri response'ta sızıyor mu? (Olmamalı)
   - Account enumeration koruması var mı (404 vs 403 ayrımı yok)?

5. Logging:
   - 500 hatasında log'a stacktrace yazılıyor mu (`log.error("...", ex)`)?
   - Domain validation hatalarında WARN seviyesi mi (ERROR DEĞİL)?
   - traceId log pattern'ine MDC ile eklenmiş mi?

6. Trace ID:
   - `TraceIdFilter` `@Order(HIGHEST_PRECEDENCE)` ile en başta mı?
   - Incoming `X-Trace-Id` header varsa korunuyor mu?
   - Yoksa yeni UUID üretiliyor mu?
   - Response header'a `X-Trace-Id` yazılıyor mu?
   - MDC `traceId` request sonunda temizleniyor mu (memory leak)?

7. Error catalog:
   - `ErrorCodes` class'ı public static final string'lerle tüm code'ları topluyor mu?
   - Code naming UPPER_SNAKE_CASE mi?
   - Kategorize prefix var mı (`ACCOUNT_`, `TRANSFER_`)?
   - `docs/error-catalog.md` dokümantasyonu yazılmış mı?

8. Test:
   - Her exception için handler test edilmiş mi?
   - 500 response'ta stacktrace olmadığını DOĞRULAYAN test var mı?
   - `Content-Type: application/problem+json` test edilmiş mi?
   - `traceId` response'ta varlığı test edilmiş mi?
   - Validation 400 detayı (fieldErrors) test edilmiş mi?
   - Bad JSON, invalid UUID gibi Spring built-in exception'lar handle ediliyor mu?

9. Banking-specific:
   - Bütün domain exception'ları ele alınmış mı (handler eksik kalmamış)?
   - Generic `Exception.class` catch-all en son mu (specific'ler önce)?
   - `BindingResult` field error'ları structured olarak property'ye eklenmiş mi?

Her madde için PASS / FAIL / EKSIK işaretle, sebep açıkla. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] `BankingException` abstract base ve concrete exception'lar yazıldı
- [ ] `ErrorCodes` catalog class'ı
- [ ] `GlobalExceptionHandler` tüm domain exception'larını ProblemDetail ile handle ediyor
- [ ] Spring built-in exception'lar (`MethodArgumentNotValid`, `HttpMessageNotReadable`, `MethodArgumentTypeMismatch`) handle ediliyor
- [ ] `Exception.class` catch-all 500 + generic mesaj, stacktrace log'a, NO LEAK
- [ ] `TraceIdFilter` çalışıyor, log pattern'inde traceId görünüyor
- [ ] Response header `X-Trace-Id` set ediliyor
- [ ] `docs/error-catalog.md` her error code için bölüm
- [ ] Curl ile manuel test yapıldı, response'lar **defterimde**
- [ ] `@WebMvcTest` ile handler integration test yazıldı (8+ test)
- [ ] 500 hatası test edildi — stacktrace response'ta YOK
- [ ] Content-Type `application/problem+json` her error response'ta

---

## Defter notları

1. "RFC 7807'nin 5 standart alanı: ____."
2. "Banking'de stacktrace response'a neden yazılmaz: ____."
3. "Account enumeration attack nedir, nasıl korunulur: ____."
4. "ProblemDetail'da kullanılan `application/problem+json` content type'ın amacı: ____."
5. "`traceId` neden gerekli, nereden gelir, nereye gider: ____."
6. "Error code naming convention ve neden değiştirilemez: ____."
7. "Banking exception'larında context bilgisi (accountId, amount) neden field olarak tutulur: ____."
8. "`Exception.class` catch-all'ın sıralama önemi (specific önce mi catch-all önce mi): ____."
9. "Log level kararı: 422 hatası neden ERROR değil de WARN/INFO: ____."
10. "ProblemDetail extension property'leri ne için kullanılır: ____."
