# Topic 9.1 — Structured Logging: Logback + SLF4J + JSON + MDC

## Hedef

Banking sisteminin loglarını **production-grade structured JSON** formatına çevirmek. SLF4J + Logback ile JSON logging, MDC ile request context (traceId, userId, accountId) propagation, log levels strategy, sensitive data redaction, log aggregation (ELK/Loki/Datadog) integration, async appender ile performance, sampling, log retention.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- SLF4J / Logback / Log4j2 farkını duy
- JSON format
- ELK / Loki ne işe yarar bilmek

---

## Kavramlar

### 1. Niye structured logging?

**Plain text log:**
```
2024-05-12 10:30:45.123 INFO  c.b.t.TransferService - Transfer initiated for user 12345 from account 67890 to IBAN TR12... amount 1000
```

Aramak: `grep "user 12345" app.log | grep "Transfer"`. **Yavaş**, fragile (text format değişirse parse bozulur).

**Structured JSON:**
```json
{
  "timestamp": "2024-05-12T10:30:45.123Z",
  "level": "INFO",
  "logger": "com.bank.transfer.TransferService",
  "message": "Transfer initiated",
  "service": "transfer-service",
  "userId": "12345",
  "fromAccount": "67890",
  "toIban": "TR12...",
  "amount": 1000.00,
  "currency": "TRY",
  "traceId": "abc-123",
  "spanId": "xyz-789",
  "tenant": "TR"
}
```

ELK/Loki **field-level query**: `userId:12345 AND service:transfer-service AND amount:>500`.

### 2. SLF4J — abstraction layer

SLF4J = **facade**. Underneath = Logback, Log4j2, JUL.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class TransferService {
    private static final Logger log = LoggerFactory.getLogger(TransferService.class);
    
    public void transfer(...) {
        log.info("Transfer initiated for user {} amount {}", userId, amount);
    }
}
```

**Parameterized message** (`{}`) — **lazy evaluation**. Log level filtered out → string concat çalışmaz.

```java
// ❌
log.debug("Transfer details: " + transfer.toString());   // String concat her zaman
// ✓
log.debug("Transfer details: {}", transfer);             // Lazy, debug off → toString çağrılmaz
```

### 3. Logback configuration — banking JSON

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="appVersion" source="info.app.version" defaultValue="unknown"/>
    
    <!-- JSON console appender (cloud-native) -->
    <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <version/>
                <logLevel/>
                <loggerName/>
                <threadName/>
                <mdc/>
                <pattern>
                    <pattern>{
                        "service": "${appName}",
                        "version": "${appVersion}",
                        "env": "${ENVIRONMENT:-dev}"
                    }</pattern>
                </pattern>
                <message/>
                <stackTrace>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>30</maxDepthPerThrowable>
                        <maxLength>2048</maxLength>
                        <shortenedClassNameLength>30</shortenedClassNameLength>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>
    
    <!-- File rolling for local dev / archival -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${appName}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/${appName}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>500MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    
    <!-- Async wrapper — non-blocking -->
    <appender name="ASYNC_STDOUT" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="JSON_STDOUT"/>
        <queueSize>10000</queueSize>
        <discardingThreshold>0</discardingThreshold>   <!-- Never discard -->
        <includeCallerData>false</includeCallerData>   <!-- Banking: caller data slow -->
        <neverBlock>false</neverBlock>                 <!-- Block if queue full -->
    </appender>
    
    <!-- Profile-specific roots -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="JSON_STDOUT"/>
        </root>
    </springProfile>
    
    <springProfile name="production">
        <root level="INFO">
            <appender-ref ref="ASYNC_STDOUT"/>
        </root>
        <logger name="com.bank" level="INFO"/>
        <logger name="org.springframework" level="WARN"/>
        <logger name="org.hibernate" level="WARN"/>
        <logger name="org.hibernate.SQL" level="WARN"/>
    </springProfile>
</configuration>
```

`pom.xml`:
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

### 4. MDC — Mapped Diagnostic Context

Per-thread (per-request) context. Otomatik JSON'a inject.

```java
import org.slf4j.MDC;

@Component
public class RequestContextFilter implements OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest req, 
                                    HttpServletResponse resp, 
                                    FilterChain chain) {
        try {
            String traceId = req.getHeader("traceparent");
            if (traceId == null) traceId = UUID.randomUUID().toString();
            
            MDC.put("traceId", traceId);
            MDC.put("requestId", UUID.randomUUID().toString());
            MDC.put("requestMethod", req.getMethod());
            MDC.put("requestPath", req.getRequestURI());
            
            if (req.getUserPrincipal() instanceof Authentication auth) {
                MDC.put("userId", auth.getName());
                if (auth instanceof JwtAuthenticationToken jwt) {
                    MDC.put("tenant", jwt.getToken().getClaimAsString("tenant"));
                }
            }
            
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();   // ÇOK ÖNEMLİ — thread pool reuse
        }
    }
}
```

Sonra her log otomatik bu fieldleri taşır:

```java
log.info("Transfer initiated amount={}", amount);
```

JSON:
```json
{
  "message": "Transfer initiated amount=1000",
  "traceId": "...", "requestId": "...", "userId": "...", "tenant": "TR",
  ...
}
```

**CRITICAL: `MDC.clear()` finally block'ta**. Thread pool reuse → eski context yeni request'e leak eder.

### 5. Banking specific MDC fields

```java
public class BankingMdcKeys {
    public static final String TRACE_ID = "traceId";
    public static final String SPAN_ID = "spanId";
    public static final String REQUEST_ID = "requestId";
    public static final String USER_ID = "userId";
    public static final String TENANT = "tenant";
    public static final String BRANCH = "branch";
    public static final String ACCOUNT_ID = "accountId";          // Banking
    public static final String TRANSACTION_ID = "transactionId";  // Banking
    public static final String CHANNEL = "channel";                // mobile/web/atm
    public static final String IP_ADDRESS = "ipAddress";
    public static final String CORRELATION_ID = "correlationId";  // Saga
}
```

```java
public void transfer(TransferRequest req) {
    MDC.put(BankingMdcKeys.TRANSACTION_ID, transactionId.toString());
    MDC.put(BankingMdcKeys.ACCOUNT_ID, req.fromAccount().toString());
    try {
        log.info("Transfer started amount={} to={}", req.amount(), req.toIban());
        // ... business logic ...
        log.info("Transfer completed");
    } finally {
        MDC.remove(BankingMdcKeys.TRANSACTION_ID);
        MDC.remove(BankingMdcKeys.ACCOUNT_ID);
    }
}
```

### 6. Log levels — banking strategy

| Level | Banking use | Production |
|---|---|---|
| **TRACE** | Step-by-step debug | OFF |
| **DEBUG** | Detailed flow info | OFF (selective ON for incident) |
| **INFO** | Business events (transfer initiated, login) | ON |
| **WARN** | Recoverable issues (retry, fallback triggered) | ON |
| **ERROR** | Failed operations, exceptions | ON, alert |
| **FATAL** | System-wide critical (Logback yok, ERROR'ı kullan) | ON, paging |

**Banking pratiği:**
- Production root level: INFO
- Hibernate SQL: WARN (verbose)
- Spring framework: WARN (verbose)
- Banking application: INFO
- Selective DEBUG: profile-based, runtime config (actuator/loggers)

**Dynamic log level via Actuator:**

```bash
curl -X POST http://localhost:8080/actuator/loggers/com.bank.transfer \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

Production incident'te debug açıp, sonra geri INFO'ya.

### 7. Banking log content — what to log

**HER ZAMAN log:**
- Login (success/fail) + IP + user-agent
- Logout
- Password change, MFA enroll/disable
- Transfer initiated/completed/failed
- Card block/unblock
- Limit changes
- Admin actions
- Authorization failures (403)
- Validation failures (400)
- External API call (DB query log değil — Hibernate burda DEBUG)
- Saga state transitions

**LOG'a YAZILMAZ (KVKK + güvenlik):**
- Şifre (plain veya hashed)
- TC kimlik no (mask → `***-TC-***`)
- Card PAN (mask → `4532-****-****-6467`)
- Card CVV (asla yok)
- Card PIN (asla yok)
- Authorization header (mask → `Bearer ***`)
- Session token
- Cookie value
- Account balance (loglamaktan kaçın, gerekirse mask)

### 8. Sensitive data masking

```java
@Component
public class SensitiveDataMasker {
    
    private static final Pattern TC_KIMLIK = Pattern.compile("\\b\\d{11}\\b");
    private static final Pattern PAN = Pattern.compile("\\b\\d{13,19}\\b");
    private static final Pattern BEARER = Pattern.compile("(?i)Bearer\\s+[\\w.-]+");
    private static final Pattern PASSWORD_QS = Pattern.compile("(?i)(password|pwd|pin)=([^&\\s,]+)");
    private static final Pattern EMAIL = Pattern.compile("\\b[\\w.-]+@[\\w.-]+\\.\\w+\\b");
    
    public String mask(String input) {
        if (input == null) return null;
        String result = input;
        result = TC_KIMLIK.matcher(result).replaceAll(m -> 
            "***-" + m.group().substring(7) + "-***");
        result = PAN.matcher(result).replaceAll(m -> {
            String pan = m.group();
            return pan.substring(0, 4) + "-****-****-" + pan.substring(pan.length() - 4);
        });
        result = BEARER.matcher(result).replaceAll("Bearer ***");
        result = PASSWORD_QS.matcher(result).replaceAll("$1=***");
        result = EMAIL.matcher(result).replaceAll(m -> {
            String email = m.group();
            int at = email.indexOf('@');
            return email.charAt(0) + "***" + email.substring(at);
        });
        return result;
    }
}
```

#### Logback masking pattern layout

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <jsonGeneratorDecorator class="com.bank.logging.MaskingJsonGeneratorDecorator"/>
</encoder>
```

```java
public class MaskingJsonGeneratorDecorator implements JsonGeneratorDecorator {
    
    @Override
    public JsonGenerator decorate(JsonGenerator generator) {
        return new MaskingJsonGenerator(generator);
    }
}

public class MaskingJsonGenerator extends JsonGeneratorDelegate {
    
    private static final SensitiveDataMasker MASKER = new SensitiveDataMasker();
    
    public MaskingJsonGenerator(JsonGenerator delegate) {
        super(delegate, false);
    }
    
    @Override
    public void writeString(String text) throws IOException {
        super.writeString(MASKER.mask(text));
    }
}
```

Tüm string field'lar masked yazılır.

### 9. StructuredArguments — explicit field

```java
import static net.logstash.logback.argument.StructuredArguments.*;

log.info("Transfer completed", 
    kv("transferId", transferId),
    kv("amount", amount),
    kv("currency", "TRY"),
    kv("durationMs", elapsedMs));
```

JSON:
```json
{
  "message": "Transfer completed",
  "transferId": "...",
  "amount": 1000.00,
  "currency": "TRY",
  "durationMs": 145
}
```

Daha clean ve queryable (her field separate).

### 10. Markers — log filtering

```java
private static final Marker AUDIT = MarkerFactory.getMarker("AUDIT");
private static final Marker SECURITY = MarkerFactory.getMarker("SECURITY");

log.info(AUDIT, "Transfer audit", kv("transferId", id));
log.warn(SECURITY, "Failed login attempt", kv("username", username), kv("ip", ip));
```

Logback config: marker'a göre filter (audit ayrı dosyaya, security SIEM'e).

### 11. Async appender — performance

Synchronous logging = her log write = I/O block. High-throughput banking için **AsyncAppender** şart.

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="JSON_STDOUT"/>
    <queueSize>10000</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <includeCallerData>false</includeCallerData>
    <neverBlock>false</neverBlock>
</appender>
```

**Trade-off:**
- `queueSize` küçük → fast but log drop in burst
- `discardingThreshold=0` → never drop
- `neverBlock=false` → app waits if queue full (preserve logs, slow app)
- `neverBlock=true` → drop logs to keep app fast (banking için tehlikeli)

Banking: queueSize 10000, never drop, block if full.

**Log4j2 alternative:** `AsyncLogger` LMAX Disruptor backed (faster). Banking high-throughput için consideration.

### 12. Log aggregation — ELK / Loki / Datadog

#### ELK (Elasticsearch + Logstash + Kibana)
- Heavy (Elasticsearch storage)
- Powerful search + analytics
- Banking yaygın

#### Loki (Grafana)
- Index labels only (cheap storage)
- LogQL benzer to PromQL
- Cloud-native, K8s native

#### Datadog Logs / Splunk
- Managed, premium
- Banking large org

#### Setup — stdout → Fluent Bit → Elasticsearch (K8s)

```
[app container] → stdout (JSON)
    ↓
[Fluent Bit daemonset] (parse + enrich + forward)
    ↓
[Elasticsearch cluster]
    ↓
[Kibana UI]
```

Spring Boot **stdout JSON**, infrastructure handles rest. K8s-native pattern.

### 13. Banking — log retention

| Log type | Retention |
|---|---|
| Application info/debug | 30-90 gün |
| Application error | 1-2 yıl |
| Audit log | **5-10 yıl** (BDDK requirement) |
| Security event | 5-10 yıl |
| Access log (load balancer) | 6-12 ay |

**Tiered storage:**
- Hot: 7 gün (Elasticsearch, fast query)
- Warm: 30 gün (Elasticsearch warm tier)
- Cold: 1 yıl (S3 Glacier, cheap)
- Archive: 10 yıl (S3 Deep Archive, regulatory)

### 14. Log query examples (Kibana KQL)

```
service:transfer-service AND level:ERROR
```

```
userId:12345 AND action:LOGIN AND result:FAIL
```

```
service:account-service AND durationMs:>1000
```

```
traceId:"abc-123"
```

Banking incident: traceId üzerinden 4 service log'unu birleştir.

### 15. Banking anti-pattern'leri

**Anti-pattern 1: System.out.println / printStackTrace**

```java
System.out.println("Transfer: " + transfer);   // ❌
e.printStackTrace();                            // ❌
```

Log framework'e gitmez, format yok, no MDC, no aggregation.

**Anti-pattern 2: String concat instead of parameterized**

```java
log.info("User " + user + " transfer " + amount);   // ❌
log.info("User {} transfer {}", user, amount);        // ✓
```

Lazy evaluation + cleaner.

**Anti-pattern 3: PII in log**

TC, PAN, password log'da → KVKK + PCI-DSS ihlal. Masking şart.

**Anti-pattern 4: MDC.clear() unutmak**

Thread pool reuse → eski request context yeni log'a sızar. Try-finally.

**Anti-pattern 5: Plain text log production'da**

ELK/Loki effective değil. JSON şart.

**Anti-pattern 6: Excessive log volume**

Her DB query INFO ile log → storage exhaust + signal-to-noise loss. DEBUG/TRACE'a düşür.

**Anti-pattern 7: Sync appender high-throughput'ta**

I/O block → latency. AsyncAppender.

**Anti-pattern 8: Stack trace silinmiş**

```java
log.error("Error happened");   // ❌
log.error("Error happened", e); // ✓
```

Exception object pass et — stack trace gider.

**Anti-pattern 9: Log seviyesi production'da DEBUG**

DEBUG verbose. Production storage burst + performance hit.

**Anti-pattern 10: Log içine business logic gömmek**

```java
log.info("Sending email to {}", emailService.send(...));   // ❌ side effect in log
```

Log statement side-effect free olmalı.

---

## Önemli olabilecek araştırma kaynakları

- Logback documentation
- SLF4J user manual
- Logstash Logback Encoder
- ELK / Loki / Datadog Logs docs
- OWASP Logging Cheat Sheet
- BDDK log retention requirements
- KVKK log masking guidance

---

## Mini task'ler

### Task 9.1.1 — JSON Logback setup (30 dk)

`logstash-logback-encoder` dependency. `logback-spring.xml` JSON. Spring profile'a göre level.

### Task 9.1.2 — MDC RequestContextFilter (45 dk)

`OncePerRequestFilter` ile traceId, requestId, userId, tenant MDC'ye. Test: log JSON'da fieldler.

### Task 9.1.3 — Banking-specific MDC enrichment (30 dk)

Service method'larda transactionId, accountId MDC'ye. Try-finally `MDC.remove()`.

### Task 9.1.4 — Sensitive data masking (60 dk)

`MaskingJsonGeneratorDecorator` TC, PAN, Bearer, password regex mask. Test: log dosyasında bu pattern'ler bulunmamalı.

### Task 9.1.5 — StructuredArguments + Markers (30 dk)

Audit + Security marker'lar. `kv()` ile field'leri explicit.

### Task 9.1.6 — Async appender + performance (45 dk)

Sync vs async benchmark (JMH veya simple loop). High-volume log throughput karşılaştır.

### Task 9.1.7 — Actuator dynamic log level (15 dk)

`POST /actuator/loggers/com.bank.transfer` ile runtime level değişimi.

### Task 9.1.8 — Loki / ELK local stack (45 dk)

Docker compose Loki + Grafana (or ELK). App stdout JSON → Loki query LogQL: `{service="transfer-service"} |~ "transfer initiated"`.

---

## Test yazma rehberi

```java
@SpringBootTest
class StructuredLoggingTest {
    
    @Autowired ObjectMapper mapper;
    
    @Test
    void shouldEmitJsonLog(CapturedOutput output) {
        log.info("Test message {}", "value");
        
        // Output JSON line
        String json = output.getOut().lines()
            .filter(l -> l.contains("Test message"))
            .findFirst().orElseThrow();
        
        Map<String, Object> parsed = mapper.readValue(json, Map.class);
        assertThat(parsed).containsEntry("level", "INFO");
        assertThat(parsed).containsEntry("message", "Test message value");
        assertThat(parsed).containsKey("@timestamp");
    }
    
    @Test
    void shouldIncludeMdcContext(CapturedOutput output) {
        MDC.put("userId", "user-123");
        MDC.put("traceId", "trace-abc");
        try {
            log.info("Action");
        } finally {
            MDC.clear();
        }
        
        Map<String, Object> json = parseLastJson(output);
        assertThat(json).containsEntry("userId", "user-123");
        assertThat(json).containsEntry("traceId", "trace-abc");
    }
    
    @Test
    void shouldMaskTcKimlik(CapturedOutput output) {
        log.info("Customer TC: 12345678901");
        
        String out = output.getOut();
        assertThat(out).doesNotContain("12345678901");
        assertThat(out).contains("***-");
    }
    
    @Test
    void shouldMaskPan(CapturedOutput output) {
        log.info("Card processed: 4532148803436467");
        
        assertThat(output.getOut()).doesNotContain("4532148803436467");
        assertThat(output.getOut()).contains("4532-****");
    }
    
    @Test
    void shouldMaskBearerToken(CapturedOutput output) {
        log.info("Authorization: Bearer eyJhbGciOi...");
        
        assertThat(output.getOut()).doesNotContain("eyJhbGciOi");
        assertThat(output.getOut()).contains("Bearer ***");
    }
    
    @Test
    void mdcClearOnRequestEnd() throws Exception {
        mockMvc.perform(get("/v1/accounts")).andExpect(status().isOk());
        
        // After request done, MDC should be empty (thread reused)
        assertThat(MDC.getCopyOfContextMap()).isNullOrEmpty();
    }
    
    @Test
    void shouldChangeLogLevelViaActuator() throws Exception {
        mockMvc.perform(post("/actuator/loggers/com.bank")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"configuredLevel\": \"DEBUG\"}"))
            .andExpect(status().isNoContent());
        
        mockMvc.perform(get("/actuator/loggers/com.bank"))
            .andExpect(jsonPath("$.configuredLevel").value("DEBUG"));
    }
}
```

---

## Claude-verify prompt

```
Structured logging implementation'ımı banking-grade kriterlere göre değerlendir:

1. JSON format:
   - Logstash encoder veya benzer JSON encoder?
   - @timestamp, level, logger, thread, message standard fields?
   - service, version, env enrichment?

2. MDC context:
   - traceId, requestId, userId, tenant standard?
   - Banking-specific (transactionId, accountId, channel)?
   - OncePerRequestFilter setup?
   - MDC.clear() finally block?
   - Thread pool leak yok mu?

3. Log levels:
   - Production root: INFO?
   - Hibernate SQL: WARN?
   - Spring framework: WARN?
   - Dynamic level via Actuator?

4. Masking:
   - TC kimlik regex mask?
   - PAN regex mask (first 4 + last 4)?
   - Bearer token mask?
   - Password/PIN query string mask?
   - Email partial mask?

5. Banking content rules:
   - Login (success/fail) logged?
   - Transfer state transitions?
   - Authorization failures?
   - Plain password/CVV/PIN ASLA log'da?

6. Performance:
   - AsyncAppender?
   - includeCallerData false?
   - queueSize 5000-10000?
   - neverBlock false (banking — preserve logs)?

7. Markers:
   - AUDIT marker → separate appender / SIEM?
   - SECURITY marker?

8. StructuredArguments:
   - kv() explicit fields?
   - Or pattern-based?

9. Aggregation:
   - stdout JSON (cloud-native)?
   - Fluent Bit / Fluentd / Filebeat?
   - ELK / Loki / Datadog?

10. Retention:
    - Application log 30-90 gün?
    - Audit log 5-10 yıl?
    - Tiered storage (hot/warm/cold)?

11. Anti-pattern:
    - System.out.println YOK?
    - String concat YOK?
    - PII unmasked YOK?
    - MDC.clear() unutulmuş YOK?
    - Stack trace silinmiş YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Logback JSON encoder + Spring profile
- [ ] RequestContextFilter MDC enrichment
- [ ] Banking-specific MDC keys
- [ ] Sensitive data masking (TC, PAN, Bearer, password)
- [ ] StructuredArguments kv() usage
- [ ] Async appender config
- [ ] Markers (AUDIT, SECURITY)
- [ ] Dynamic log level via Actuator
- [ ] Loki / ELK local stack + LogQL query
- [ ] 8+ integration test (masking, MDC, async)

---

## Defter notları (10 madde)

1. "Structured JSON vs plain text logging banking trade-off: ____."
2. "SLF4J parameterized logging lazy evaluation: ____."
3. "MDC per-thread context + RequestContextFilter pattern: ____."
4. "Banking-specific MDC keys (transactionId, accountId, channel): ____."
5. "Sensitive data masking (TC, PAN, Bearer) regex: ____."
6. "Log levels banking strategy (production INFO, dynamic DEBUG): ____."
7. "StructuredArguments kv() explicit field vs pattern message: ____."
8. "AsyncAppender queue + discardingThreshold + neverBlock trade-off: ____."
9. "Log aggregation (ELK / Loki / Datadog) + Fluent Bit K8s: ____."
10. "Banking log retention (audit 5-10 yıl) + tiered storage: ____."
