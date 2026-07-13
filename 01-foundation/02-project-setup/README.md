# Topic 1.2 — Proje Setup: Spring Boot 3 + Maven + Profiles

## Hedef

Boş bir klasörden çalışan bir Spring Boot 3 uygulamasına kadar gelmek. Sadece "çalıştır" değil, **production-grade konfigürasyon** ile: profil ayrımı, type-safe config binding, secrets'ı yml'den uzakta tutma.

## Süre

Okuma: 1 saat • Mini task: 2 saat • Test: 30 dk • Toplam: ~3.5 saat

## Önbilgi

- Topic 1.1 tamamlandı
- Java 21 kurulu (Maven projende Java 21 hedefleyeceğiz — virtual thread için)
- Maven kurulu (`mvn -version`)

---

## Kavramlar

### 1. Spring Boot felsefesi

Spring Boot 3 (2022-2024+), Spring Framework 6 üzerine kurulu. Üç temel taahhüt:

1. **Opinionated defaults:** "Çoğu durum için doğru olanı varsayılan yap." Logging, JSON, embedded server hepsi sıfır config çalışır.
2. **Auto-configuration:** Classpath'te `spring-boot-starter-data-jpa` görüyorsa, otomatik olarak Hibernate kur, DataSource bekle, EntityManager bean'i hazırla.
3. **Production-ready:** Actuator ile health/metrics, externalized config, profile management — production'a hazır.

**Spring Boot ≠ Spring Framework.** Spring Framework (Core, MVC, Data, Security) altyapı. Spring Boot ise bu altyapının üstüne **starter dependency'leri + auto-config + opinionated default** ekler.

### 2. Spring Initializr vs manuel Maven

İki yol var:

**A. Spring Initializr (önerilen):** [start.spring.io](https://start.spring.io)
- Web UI'da bağımlılıkları tıklayarak seçersin
- Hazır `pom.xml` + boş main class indirir
- Hızlı, hatasız başlama

**B. Manuel:** Boş klasör, `pom.xml`'i elinle yaz
- Tüm versiyon ve plugin'leri kendin bilmen lazım
- Genelde junior tuzağı: yanlış versiyon, eksik plugin, çalışmayan setup

Bu projede **Initializr ile başla.** Aldıktan sonra `pom.xml`'i inceleyerek "Initializr ne kurdu" sorusunu kendin cevapla.

### 3. `pom.xml` anatomisi

Spring Boot uygulamasının `pom.xml`'i:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <!-- Spring Boot parent — versiyon yönetimi buradan gelir -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>  <!-- Bu projedeki Spring Boot versiyonu -->
        <relativePath/>
    </parent>

    <groupId>com.mavibank</groupId>
    <artifactId>core-banking</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <name>core-banking</name>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Web (Tomcat + Spring MVC) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Validation (Hibernate Validator) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- JPA + Hibernate -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- PostgreSQL driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Flyway -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- Actuator (health, metrics) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- TestContainers (Topic 1.4'te kullanılacak) -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <version>1.20.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>1.20.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Anahtar şeyler:**

- **`<parent>`:** Spring Boot'un parent POM'una bağlanmak `dependencyManagement` getirir. Her Spring Boot dependency için versiyon yazmana gerek yok — parent yönetir. Bu çok değerli (versiyon uyumsuzluğu tehlikeleri kalkar).

- **`spring-boot-starter-*`:** "Starter" = bir grup ilgili dependency. `starter-web` = Spring MVC + Tomcat + Jackson + validation altyapısı. Tek satır dependency, arkada 30 jar gelir.

- **`spring-boot-maven-plugin`:** `mvn spring-boot:run` ve fat JAR build için.

### 4. Maven temel komutlar (banking'de günlük kullanım)

```bash
mvn clean              # target/ klasörünü siler
mvn compile            # sadece compile
mvn test               # test'leri çalıştırır (unit + integration)
mvn package            # JAR oluşturur
mvn install            # local Maven repo'na koyar
mvn spring-boot:run    # uygulamayı başlatır

mvn dependency:tree    # bağımlılık ağacını gösterir (versiyon çakışmalarını bulmak için)
mvn versions:display-dependency-updates  # güncel olmayan dep'leri listeler

mvn -DskipTests package  # test'siz build
mvn test -Dtest=AccountTest  # sadece bir test class
```

**Önemli:** TR bankalarında çoğu yerde Maven kullanılır. Gradle bazı modern startup'larda. Maven'a hakim ol.

### 5. Single module vs Multi-module

**Single module:** Tek `pom.xml`, tüm kod `src/main/java/` altında. Package'larla organize.

**Multi-module:** Parent `pom.xml` + alt modüller (her birinin kendi `pom.xml`'i).

```
core-banking/
├── pom.xml                                  (parent)
├── banking-domain/
│   ├── pom.xml
│   └── src/main/java/
├── banking-application/
│   ├── pom.xml
│   └── src/main/java/
└── banking-adapter-web/
    ├── pom.xml
    └── src/main/java/
```

**Phase 1'de:** Single module + iyi package yapısı. Sebep:
- Multi-module Maven karmaşıklığı junior için ekstra yük
- Package-private görünürlüğü `domain` package'ında kullanırsan zaten ayrışma sağlanır
- Phase 7'de microservice'e ayırırken multi-module'a geçeriz

### 6. `application.yml` — externalized configuration

Spring Boot, `src/main/resources/application.properties` veya `application.yml` dosyasından konfigürasyon okur. YAML daha okunabilir, onu kullanırız.

**Temel `application.yml`:**

```yaml
spring:
  application:
    name: core-banking
  
  datasource:
    url: jdbc:postgresql://localhost:5432/banking
    username: ${DB_USERNAME:banking_app}
    password: ${DB_PASSWORD:}
    hikari:
      maximum-pool-size: 10
      connection-timeout: 5000
      leak-detection-threshold: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate   # Flyway yönetir, Hibernate dokunmasın
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 50
        order_inserts: true
  
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized

logging:
  level:
    root: INFO
    com.mavibank: DEBUG
    org.hibernate.SQL: DEBUG    # sadece development'ta
```

**Önemli noktalar:**

- **`${DB_PASSWORD:}` syntax'ı:** Environment variable `DB_PASSWORD` set edilmemişse default olarak boş string. Production'da DB password yml'de yazmayacaksın — environment variable veya secret manager'dan gelecek.
- **`spring.jpa.hibernate.ddl-auto: validate`:** Hibernate schema'ya dokunmasın, sadece doğrulasın. Schema migration Flyway'in işi.
- **`management.endpoints.web.exposure.include`:** Actuator'da hangi endpoint'leri expose edeceksin. Production'da `health,info,prometheus` kafidir.

### 7. Profile sistemi (dev / test / prod)

Banking projelerinde **en az 3 ortam** var:

- **dev:** developer'ın laptop'u — H2 in-memory veya lokal PostgreSQL, verbose log
- **test:** CI'da çalışan integration test'ler — TestContainers ile geçici DB
- **prod:** production — gerçek DB, encrypted password, sıkı log

Spring Boot bunları **profile** sistemi ile yönetir.

**Dosya hiyerarşisi:**

```
src/main/resources/
├── application.yml              ← ortak tüm profile'lerde geçerli
├── application-dev.yml          ← `dev` profile aktifken eklenir
├── application-test.yml         ← `test` profile aktifken eklenir
└── application-prod.yml         ← `prod` profile aktifken eklenir
```

**Override mekaniği:** Üstte `application.yml`, üzerine profile-specific dosya. Aynı key varsa profile-specific kazanır.

**Örnek `application-dev.yml`:**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/banking_dev
    username: banking_dev
    password: dev_password    # local dev için OK, production'da ASLA

logging:
  level:
    com.mavibank: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # parametre değerlerini de logla
```

**Örnek `application-prod.yml`:**

```yaml
spring:
  datasource:
    url: ${DB_URL}              # env var'dan gelir
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}    # secret manager'dan
  jpa:
    show-sql: false

logging:
  level:
    root: WARN
    com.mavibank: INFO
```

**Profile aktifleştirme yolları:**

```bash
# 1. Komut satırı
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# 2. Environment variable
SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run

# 3. application.yml'da default
spring:
  profiles:
    active: dev

# 4. Java -jar
java -jar -Dspring.profiles.active=prod core-banking.jar
```

**Birden fazla profile:**
```
SPRING_PROFILES_ACTIVE=prod,monitoring,debug
```
Sırasıyla uygulanır, sondaki kazanır.

### 8. Configuration source precedence (kim kazanır)

Spring Boot config kaynaklarının önceliği (sondaki en güçlü):

1. `application.yml`
2. `application-{profile}.yml`
3. OS environment variables (`DB_PASSWORD=...`)
4. JVM system properties (`-Dspring.datasource.url=...`)
5. Command-line arguments (`--spring.datasource.url=...`)

**Banking pratik kuralı:**

- Sabit defaults → `application.yml`
- Profile-specific override → `application-{profile}.yml`
- Secret (password, API key) → environment variable, **asla yml'de değil**
- Acil prod override → komut satırı argümanı

### 9. `@Value` vs `@ConfigurationProperties`

İki yolla konfigürasyonu kod'a bağlarsın.

**`@Value` — basit ama kötü scale eder:**

```java
@Service
public class TransferService {
    @Value("${banking.transfer.max-amount:10000}")
    private BigDecimal maxAmount;
    
    @Value("${banking.transfer.fee.percentage}")
    private BigDecimal feePercentage;
}
```

Sorunlar:
- Field başına annotation
- Validation yok
- Refactor'da string key kaybolur
- Type-safe değil (typo'da runtime hatası)
- Test etmek zor (her field için ayrı `@TestPropertySource`)

**`@ConfigurationProperties` — type-safe, banking standardı:**

```java
@ConfigurationProperties(prefix = "banking.transfer")
@Validated
public record TransferProperties(
    @NotNull @DecimalMin("1.00") BigDecimal maxAmount,
    @NotNull @DecimalMin("0.00") @DecimalMax("100.00") BigDecimal feePercentage,
    @NotNull Duration timeoutDuration,
    Map<String, BigDecimal> currencyLimits
) {}
```

Karşılığı `application.yml`:

```yaml
banking:
  transfer:
    max-amount: 50000.00
    fee-percentage: 0.5
    timeout-duration: PT30S
    currency-limits:
      TRY: 100000.00
      USD: 10000.00
      EUR: 10000.00
```

**Aktivasyon (Spring Boot 3):**

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class CoreBankingApplication {
    public static void main(String[] args) {
        SpringApplication.run(CoreBankingApplication.class, args);
    }
}
```

Veya per-class:
```java
@EnableConfigurationProperties(TransferProperties.class)
```

**Kullanım:**

```java
@Service
class TransferService {
    private final TransferProperties props;
    
    TransferService(TransferProperties props) {
        this.props = props;
    }
    
    void transfer(...) {
        if (amount.isGreaterThan(props.maxAmount())) {
            throw new ExceedsTransferLimitException(...);
        }
    }
}
```

**Banking kural:** Tüm business rules için `@ConfigurationProperties` kullan. `@Value` sadece çok basit (ve genelde Spring'in kendi prop'ları için) durumlarda.

### 10. Property naming — kebab-case ↔ camelCase

YAML'da `kebab-case`, Java'da `camelCase`. Spring Boot otomatik mapleyemi yapar (**relaxed binding**):

```yaml
banking:
  transfer:
    max-amount: 1000      # YAML: kebab-case
```

```java
record TransferProperties(BigDecimal maxAmount) {}  // Java: camelCase
```

Eşleşir. Banking ekibinde tutarlılık için YAML'da hep kebab-case yaz.

### 11. Profile activation testi

Test'lerde profile aktifleştirme:

```java
@SpringBootTest
@ActiveProfiles("test")
class TransferServiceIntegrationTest {
    // ...
}
```

`application-test.yml`:

```yaml
spring:
  datasource:
    url: jdbc:tc:postgresql:16:///banking_test   # TestContainers JDBC URL

logging:
  level:
    org.hibernate.SQL: DEBUG
```

### 12. Secret management — banking gerçeği

Banking'de credential yönetimi:

**Asla:**
- `application.yml`'de plain password
- Git'e commit edilmiş secret
- Slack/email üzerinden paylaşılan password

**Üretim:**
- HashiCorp Vault
- AWS Secrets Manager
- Kubernetes Secrets (dikkat: base64, encrypted değil)
- Sealed Secrets

**Local dev için ara çözüm:**
- `.env` dosyası (`.gitignore`'da)
- `application-local.yml` (Git'e dahil değil)

Spring Boot 3.2+ `spring.config.import` ile Vault entegrasyonu çok temiz:

```yaml
spring:
  config:
    import: "vault://secret/banking"
```

Bunu Phase 8 (Security) topic'inde detaylı yapacağız. Şimdilik environment variable kullanmak yeterli.

### 13. Spring Boot DevTools — development hayat kurtarıcı

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

Faydaları:
- Classpath değişikliğinde otomatik restart
- LiveReload (browser auto-refresh)
- H2 console gibi development-only auto-config
- `spring.devtools.restart.enabled=true`

**Production'da otomatik kapanır** — `scope=runtime` + Spring Boot kontrolü.

### 14. Component scanning ve auto-configuration mekaniği

`@SpringBootApplication` aslında 3 annotation'ın birleşimi:

```java
@SpringBootConfiguration  // ≈ @Configuration
@EnableAutoConfiguration  // classpath bazlı auto-config
@ComponentScan            // bu class'ın bulunduğu paket + altları taranır
public @interface SpringBootApplication { }
```

**Component scan kuralı:** `@SpringBootApplication` annotation'lı class'ın **paketi ve alt paketleri** taranır.

Yani main class'ını `com.mavibank.banking` paketine koyarsan:
- `com.mavibank.banking.account.adapter.in.web.*` ✓ taranır
- `com.mavibank.banking.transfer.adapter.out.persistence.*` ✓ taranır
- `com.mavibank.other.*` ✗ taranmaz

**Sonuç:** Main class'ını **en üst paketin altına** koy.

---

## Önemli olabilecek araştırma kaynakları

- Spring Boot 3 reference doc (docs.spring.io)
- "Spring Boot in Action" (eski versiyon ama temel mantık aynı)
- Baeldung Spring Boot tutorial serisi
- "Spring Boot 3 and Spring Framework 6" Dan Vega (YouTube)
- TestContainers official documentation
- HikariCP GitHub README (banka tarafında kritik)
- Reflectoring.io — production-grade Spring Boot
- "12-Factor App" (12factor.net) — config yönetimi prensipleri

---

## Mini task'ler

Tüm task'leri `~/projects/core-banking/` içinde yap. Topic 1.1'deki domain class'ları kalsın, üzerine Spring Boot ekliyoruz.

### Task 1.2.1 — Spring Initializr ile proje oluştur (15 dk)

[start.spring.io](https://start.spring.io)'a git:
- Project: Maven
- Language: Java
- Spring Boot: 3.3.x veya 3.4.x (en güncel stabil)
- Group: `com.mavibank`
- Artifact: `core-banking`
- Name: `core-banking`
- Description: `Banking backend learning project`
- Package name: `com.mavibank.banking`
- Packaging: Jar
- Java: 21

Dependencies:
- Spring Web
- Spring Data JPA
- Validation
- Spring Boot Actuator
- PostgreSQL Driver
- Flyway Migration
- Spring Boot DevTools
- Lombok (opsiyonel — Topic 1.5'te tartışacağız)

Indir, zip'i `~/projects/core-banking/` içine aç. **Önemli:** Topic 1.1'de yazdığın `Account`, `Money`, vb. class'larını Initializr'ın oluşturduğu `src/main/java/com/mavibank/banking/` altına taşı (paket yapın korunsun).

`mvn spring-boot:run` ile boş bir Spring Boot ayağa kalkmalı.

### Task 1.2.2 — `pom.xml`'i inceleyerek not al (20 dk)

`pom.xml`'i aç. Şunları kendine sor ve **defterine cevap yaz**:

1. `<parent>` neden var, neyi sağlıyor?
2. `spring-boot-starter-web` dependency'sinin transitive olarak getirdiği ana kütüphaneler neler? (`mvn dependency:tree | head -50`)
3. `spring-boot-maven-plugin`'in `repackage` goal'u ne yapar?
4. `spring-boot-starter-test` hangi test kütüphanelerini getirir?
5. Tomcat versiyonu kaç? Bunu nereden tespit ettin?

### Task 1.2.3 — `application.yml` ile profile yapısı kur (30 dk)

`src/main/resources/` altında:
- `application.yml` — ortak ayarlar
- `application-dev.yml` — local PostgreSQL
- `application-test.yml` — TestContainers
- `application-prod.yml` — env vars

`application.yml`'da default profile'i `dev` yap:

```yaml
spring:
  profiles:
    active: dev
```

Test:
```bash
mvn spring-boot:run                              # dev profile (default)
SPRING_PROFILES_ACTIVE=prod mvn spring-boot:run  # prod profile (DB env var yoksa hata)
```

**Anti-pattern uyarısı:** `application.yml`'a default profile yazmak — production'da unutursun. Bunu Phase 1'de pedagojik amaçla yapıyoruz, gerçek üretimde **profile aktivasyonu deployment'tan gelmeli** (env var, K8s ConfigMap).

### Task 1.2.4 — `@ConfigurationProperties` class yaz (30 dk)

`banking/account/config/AccountProperties.java`:

```java
@ConfigurationProperties(prefix = "banking.account")
@Validated
public record AccountProperties(
    @NotNull @DecimalMin("0.00") BigDecimal minOpeningBalance,
    @NotEmpty Set<String> supportedCurrencies,
    @NotNull Duration accountInactivityThreshold
) {}
```

`application.yml`:

```yaml
banking:
  account:
    min-opening-balance: 0.00
    supported-currencies:
      - TRY
      - USD
      - EUR
    account-inactivity-threshold: P365D    # ISO-8601 duration
```

`CoreBankingApplication`'a `@ConfigurationPropertiesScan` ekle. Bir test class'tan inject ederek değerlerin doğru bind olduğunu doğrula.

### Task 1.2.5 — Health check ve actuator (15 dk)

`application.yml`'a:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: always   # dev için OK, prod'da `when_authorized`
```

Uygulamayı çalıştır, `curl http://localhost:8080/actuator/health` ile UP cevabını al.
`http://localhost:8080/actuator/env` ile property kaynaklarını incele.

### Task 1.2.6 — Logging configuration (15 dk)

`application.yml` log seviyelerini ayarla:

```yaml
logging:
  level:
    root: INFO
    com.mavibank: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG     # dev'de SQL'i gör
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"
```

Logger inject et:
```java
@Service
class AccountService {
    private static final Logger log = LoggerFactory.getLogger(AccountService.class);
    
    void doSomething() {
        log.debug("Doing something for account {}", accountId);
    }
}
```

`DEBUG` ve `INFO` log'larının görünür olduğunu doğrula.

### Task 1.2.7 — `.gitignore` ve `.env` (10 dk)

Project root'a `.gitignore`:

```
# Maven
target/
*.iml
.idea/
.vscode/

# Spring Boot
HELP.md
application-local.yml
.env

# OS
.DS_Store
Thumbs.db
```

`application-local.yml` (gitignored — secret'larınla aynı yere):
```yaml
spring:
  datasource:
    password: my_local_dev_password
```

**Önemli:** Bunu commit ediyor musun? Hayır. **`.env`/`*-local.*` asla git'e gitmesin.**

---

## Test yazma rehberi

### Test 1.2.1 — `@SpringBootTest` smoke test

`src/test/java/com/mavibank/banking/CoreBankingApplicationTests.java`:

```java
@SpringBootTest
@ActiveProfiles("test")
class CoreBankingApplicationTests {

    @Test
    void contextLoads() {
        // Sadece Spring context'in yüklendiğini doğrular
        // Boş test olsa da değerli — config hatalarını yakalar
    }
}
```

`application-test.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test    # TestContainers'ı Topic 1.4'te kullanacağız
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
  flyway:
    enabled: false
```

H2 dependency'sini test scope'ta ekle:

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

### Test 1.2.2 — `@ConfigurationProperties` binding test

```java
@SpringBootTest
@ActiveProfiles("test")
class AccountPropertiesTest {
    
    @Autowired
    private AccountProperties props;
    
    @Test
    void shouldBindSupportedCurrenciesFromYml() {
        assertThat(props.supportedCurrencies())
            .containsExactlyInAnyOrder("TRY", "USD", "EUR");
    }
    
    @Test
    void minOpeningBalanceShouldNotBeNegative() {
        assertThat(props.minOpeningBalance())
            .isGreaterThanOrEqualTo(BigDecimal.ZERO);
    }
}
```

### Test 1.2.3 — Validation hata fırlatması

`application-test.yml`'da `min-opening-balance: -1.00` koy (yanlış config), uygulamanın **boot olmamasını** bekle. Bunu test class olarak değil, manuel `mvn spring-boot:run` ile dene. Validation hatası ile fail olmalı.

Otomatik test için:
```java
@Test
void shouldFailWhenMinOpeningBalanceIsNegative() {
    ConfigurableApplicationContext ctx = null;
    try {
        ctx = new SpringApplicationBuilder(CoreBankingApplication.class)
            .properties("banking.account.min-opening-balance=-1.00",
                       "banking.account.supported-currencies=TRY",
                       "banking.account.account-inactivity-threshold=P365D")
            .web(WebApplicationType.NONE)
            .run();
        fail("Should have failed validation");
    } catch (Exception e) {
        assertThat(e).hasCauseInstanceOf(ConstraintViolationException.class);
    } finally {
        if (ctx != null) ctx.close();
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki Spring Boot 3 + Maven projemin yapısını ve konfigürasyonunu değerlendir. 
Sadece eksiklikleri ve hataları işaretle, kod yazma.

1. pom.xml:
   - Spring Boot 3.3.x veya 3.4.x parent kullanılmış mı?
   - Java 21 hedeflenmiş mi?
   - Gereksiz dependency var mı?
   - Versiyon yönetimi parent POM'a bırakılmış mı (manuel <version> yok)?

2. Proje yapısı:
   - Main class en üst paket altında mı (component scan doğru çalışsın)?
   - Topic 1.1'deki domain class'ları doğru paket yapısında mı?
   - test/main paketleri eşleşiyor mu?

3. application.yml:
   - Profile dosyaları (dev, test, prod) ayrılmış mı?
   - Hassas bilgi (password) yml'de YAZILMIŞ MI? Yazılmışsa fail.
   - `spring.jpa.hibernate.ddl-auto: validate` veya `none` mu (create/update OLMAMALI)?
   - Actuator endpoint exposure kontrolünde mi (her şey expose edilmemiş)?
   - Datasource bilgileri env variable referansları ile mi geliyor?

4. @ConfigurationProperties:
   - `record` veya immutable class olarak yazılmış mı?
   - `@Validated` annotation'ı var mı?
   - Field validation (`@NotNull`, `@DecimalMin`, vb.) eklenmiş mi?
   - `@ConfigurationPropertiesScan` veya `@EnableConfigurationProperties` ile aktive edilmiş mi?

5. .gitignore:
   - `target/`, `*.iml`, `.idea/` var mı?
   - `application-local.yml` ve `.env` var mı?

6. Test:
   - `contextLoads()` smoke test var mı?
   - `@ActiveProfiles("test")` kullanılmış mı?
   - ConfigurationProperties binding testi var mı?

Her madde için PASS / FAIL / EKSIK işaretle ve nedenini söyle. Kod düzeltmeleri 
yapma, sadece açıklama yap.
```

---

## Tamamlama kriterleri

- [ ] `mvn spring-boot:run` ile uygulama ayağa kalkıyor
- [ ] 3 profile dosyası ayrı, ortak ayarlar `application.yml`'da
- [ ] `application-prod.yml`'da hiçbir secret YAZILI değil, hepsi `${ENV_VAR}` referansı
- [ ] `.gitignore`'da `application-local.yml` ve `.env` var
- [ ] `@ConfigurationProperties` ile en az 1 typed config class
- [ ] `mvn test` geçiyor (en az `contextLoads` testi)
- [ ] `/actuator/health` UP dönüyor
- [ ] Logger ile DEBUG ve INFO log'ları ayırt edebiliyorum
- [ ] `mvn dependency:tree`'yi okuyup nereden ne geldiğini anlayabiliyorum

---

## Defter notları

1. "Spring Boot'un üç temel taahhüdü: ____, ____, ____."
2. "`@Value` yerine `@ConfigurationProperties` tercih etmemin sebebi ____, ____, ____."
3. "Configuration source precedence: en güçlüden zayıfa ____."
4. "Banking projesinde secret'ları neden yml'de tutmam? ____."
5. "Bir profile aktifleştirmenin 4 yolu: ____, ____, ____, ____."
6. "Spring Boot'un component scan'i hangi paketleri tarar? ____."
