# Topic 12.4 — ArchUnit: Architecture Tests

## Hedef

ArchUnit ile architecture rules **JUnit test olarak** enforce etmek. Hexagonal architecture layers, naming conventions, dependency direction, package structure, banking-specific rules (no direct DB UPDATE for balance, no PAN in DTO, KVKK PII boundary). Banking için "compile-time'da yakalanmayan code review issue"larını automated yakalama.

## Süre

Okuma: 1.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- JUnit 5 (Topic 12.1) bitti
- Hexagonal architecture (Phase 2)
- Banking domain (Phase 10)

---

## Kavramlar

### 1. ArchUnit — niye?

Code review = bireysel + zaman alıcı + insan hatasına açık.

**Architecture rules:**
- "Domain layer'dan infrastructure'a bağımlılık olmaz"
- "Controller'lar Repository'lere doğrudan ulaşmaz, Service üzerinden"
- "Adapter sınıfları `Adapter` ile bitsin"
- "Domain'de Spring annotation kullanılmasın"

Bunlar **kuralları** unutulur, eski code'da gevşer. **ArchUnit = JUnit-style enforcement.**

### 2. Setup

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.2.1</version>
    <scope>test</scope>
</dependency>
```

```java
@AnalyzeClasses(packages = "com.bank", importOptions = ImportOption.DoNotIncludeTests.class)
public class BankingArchitectureTest { ... }
```

### 3. Hexagonal architecture rules

```java
@AnalyzeClasses(packages = "com.bank.transfer", importOptions = DoNotIncludeTests.class)
class TransferHexagonalArchTest {
    
    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..infrastructure..",
                "..adapter..",
                "..persistence..",
                "..rest..",
                "..kafka.."
            );
    
    @ArchTest
    static final ArchRule domainShouldNotDependOnSpring =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "jakarta.servlet..",
                "org.hibernate.."
            );
    
    @ArchTest
    static final ArchRule applicationShouldNotDependOnAdapters =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..adapter.in..",
                "..adapter.out..",
                "..rest..",
                "..persistence.."
            );
    
    @ArchTest
    static final ArchRule layeredArchitecture = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Domain").definedBy("..domain..")
        .layer("Application").definedBy("..application..")
        .layer("Adapter").definedBy("..adapter..")
        .layer("Infrastructure").definedBy("..infrastructure..")
        .whereLayer("Adapter").mayOnlyBeAccessedByLayers("Infrastructure")
        .whereLayer("Application").mayOnlyBeAccessedByLayers("Adapter", "Infrastructure")
        .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Adapter", "Infrastructure");
}
```

### 4. Naming conventions

```java
@ArchTest
static final ArchRule controllersShouldBeNamedXxxController =
    classes().that().resideInAPackage("..adapter.in.rest..")
        .and().areAnnotatedWith(RestController.class)
        .should().haveSimpleNameEndingWith("Controller");

@ArchTest
static final ArchRule repositoriesShouldBeNamedXxxRepository =
    classes().that().resideInAPackage("..adapter.out.persistence..")
        .and().areAnnotatedWith(Repository.class)
        .should().haveSimpleNameEndingWith("Repository");

@ArchTest
static final ArchRule servicesShouldBeNamedXxxService =
    classes().that().resideInAPackage("..application.service..")
        .should().haveSimpleNameEndingWith("Service");

@ArchTest
static final ArchRule portInterfacesShouldEndWithPort =
    classes().that().resideInAPackage("..application.port..")
        .and().areInterfaces()
        .should().haveSimpleNameEndingWith("Port")
        .orShould().haveSimpleNameEndingWith("UseCase");

@ArchTest
static final ArchRule eventsShouldEndWithEvent =
    classes().that().resideInAPackage("..domain.event..")
        .and().haveSimpleNameNotContaining("Test")
        .should().haveSimpleNameEndingWith("Event");
```

### 5. Banking-specific rules

```java
@ArchTest
static final ArchRule noDirectBalanceUpdate =
    noClasses().should().callMethod(JdbcTemplate.class, "update")
        .orShould().callMethod(EntityManager.class, "createNativeQuery")
        .as("All balance changes must go through LedgerService (Topic 10.1)");

@ArchTest
static final ArchRule onlyLedgerServiceCanInsertJournalEntry =
    classes().that().haveSimpleName("JournalEntryRepository")
        .should().onlyBeAccessed().byClassesThat()
            .haveSimpleName("LedgerService")
            .or().haveSimpleName("LedgerEntryListener");

@ArchTest
static final ArchRule noPanInDtoExceptCardService =
    noClasses().that().resideInAPackage("..adapter.in.rest.dto..")
        .and().resideOutsideOfPackage("..card.adapter.in.rest.dto..")
        .should().haveSimpleNameContaining("Pan")
        .orShould().haveSimpleNameContaining("CardNumber");

@ArchTest
static final ArchRule sensitiveDataMustUseEncryptedConverter =
    fields().that().areAnnotatedWith(Sensitive.class)
        .should().beAnnotatedWith(Convert.class);

@ArchTest
static final ArchRule moneyMustBeBigDecimal =
    fields().that().haveNameMatching(".*[aA]mount.*|.*[bB]alance.*|.*[fF]ee.*")
        .and().areDeclaredInClassesThat().resideInAPackage("..domain..")
        .should().haveRawType(BigDecimal.class)
        .as("Money fields MUST be BigDecimal (Phase 1)");

@ArchTest
static final ArchRule noDoubleForMoney =
    noFields().that().haveNameMatching(".*[aA]mount.*|.*[bB]alance.*|.*[fF]ee.*")
        .should().haveRawType(double.class)
        .orShould().haveRawType(Double.class)
        .orShould().haveRawType(float.class);

@ArchTest
static final ArchRule restControllersMustReturnResponseEntity =
    methods().that().areAnnotatedWith(GetMapping.class)
        .or().areAnnotatedWith(PostMapping.class)
        .or().areAnnotatedWith(PutMapping.class)
        .or().areAnnotatedWith(DeleteMapping.class)
        .and().areDeclaredInClassesThat().areAnnotatedWith(RestController.class)
        .should().haveRawReturnType(ResponseEntity.class)
        .orShould().haveRawReturnType(ProblemDetail.class);

@ArchTest
static final ArchRule auditLogShouldBeImmutable =
    classes().that().haveSimpleName("AuditEvent")
        .should().beAnnotatedWith(Immutable.class)
        .as("Audit log immutable (Topic 8.7)");

@ArchTest
static final ArchRule transferShouldUseIdempotency =
    methods().that().areAnnotatedWith(PostMapping.class)
        .and().areDeclaredInClassesThat().haveSimpleNameContaining("Transfer")
        .should().haveAnnotationThatMatches(method -> {
            return Arrays.stream(method.getParameterAnnotations())
                .flatMap(Arrays::stream)
                .anyMatch(a -> a.annotationType().getSimpleName().equals("IdempotencyKey"));
        });
```

### 6. KVKK + PCI-DSS rules

```java
@ArchTest
static final ArchRule piiFieldsShouldUseEncryptedConverter =
    fields().that().haveName("tcKimlik")
        .or().haveName("email")
        .or().haveName("phone")
        .and().areDeclaredInClassesThat().areAnnotatedWith(Entity.class)
        .should().beAnnotatedWith(Convert.class);

@ArchTest
static final ArchRule noLogStatementWithSensitive =
    noMethods().should().callMethodWhere(target ->
        target.getOwner().getSimpleName().equals("Logger")
        && Arrays.asList("info", "debug", "warn", "error", "trace").contains(target.getName())
        && callerCodeContains(target, "tcKimlik", "pan", "cvv", "pin"));

@ArchTest
static final ArchRule cardServiceMustUseTokenization =
    fields().that().haveName("pan")
        .and().areDeclaredInClassesThat().resideInAPackage("..card..")
        .should().haveRawType(String.class)
        .andShould().beAnnotatedWith(Tokenized.class);
```

### 7. Package structure rules

```java
@ArchTest
static final ArchRule noCyclicDependencies =
    slices().matching("com.bank.(*)..")
        .should().beFreeOfCycles();

@ArchTest
static final ArchRule serviceLayersShouldNotCallEachOther =
    noClasses().that().resideInAPackage("com.bank.transfer.application..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.bank.account.application..",
            "com.bank.card.application..",
            "com.bank.loan.application.."
        )
        .as("Service-to-service via REST/event only");

@ArchTest
static final ArchRule moduleBoundaryRespect =
    layeredArchitecture()
        .consideringAllDependencies()
        .layer("Transfer").definedBy("com.bank.transfer..")
        .layer("Account").definedBy("com.bank.account..")
        .layer("Card").definedBy("com.bank.card..")
        .layer("Common").definedBy("com.bank.common..")
        .whereLayer("Transfer").mayNotAccessAnyOtherLayer()
        .whereLayer("Account").mayNotAccessAnyOtherLayer()
        .whereLayer("Card").mayNotAccessAnyOtherLayer();
```

### 8. Spring Boot specific

```java
@ArchTest
static final ArchRule controllersShouldNotUseRepositoriesDirect =
    noClasses().that().areAnnotatedWith(RestController.class)
        .should().dependOnClassesThat().areAnnotatedWith(Repository.class)
        .as("Controller → Service → Repository");

@ArchTest
static final ArchRule transactionalOnServicesNotRepositories =
    classes().that().areAnnotatedWith(Transactional.class)
        .should().beAnnotatedWith(Service.class)
        .orShould().resideInAPackage("..application.service..");

@ArchTest
static final ArchRule noFieldInjection =
    noFields().should().beAnnotatedWith(Autowired.class)
        .as("Constructor injection only (immutable + testable)");

@ArchTest
static final ArchRule preAuthorizeOnSensitiveEndpoints =
    methods().that().areAnnotatedWith(PostMapping.class)
        .or().areAnnotatedWith(DeleteMapping.class)
        .and().areDeclaredInClassesThat().areAnnotatedWith(RestController.class)
        .and().areDeclaredInClassesThat().haveSimpleNameNotContaining("Public")
        .should().beAnnotatedWith(PreAuthorize.class)
        .as("All sensitive endpoints have explicit authorization");
```

### 9. Custom predicate + condition

```java
@ArchTest
static final ArchRule customPredicate = 
    methods().that(new DescribedPredicate<JavaMethod>("are public banking endpoints") {
        @Override
        public boolean test(JavaMethod method) {
            return method.getOwner().isAnnotatedWith(RestController.class)
                && method.getModifiers().contains(JavaModifier.PUBLIC)
                && !method.getOwner().getSimpleName().contains("Public");
        }
    })
    .should(new ArchCondition<JavaMethod>("be auditable") {
        @Override
        public void check(JavaMethod method, ConditionEvents events) {
            boolean hasAudit = method.isAnnotatedWith(Audited.class);
            if (!hasAudit) {
                events.add(SimpleConditionEvent.violated(method,
                    method.getFullName() + " is not @Audited"));
            }
        }
    });
```

### 10. Banking — ArchUnit anti-pattern'leri

**Anti-pattern 1: Tests in src/main rule check**
```java
@AnalyzeClasses(packages = "com.bank")   // includes tests
```
`DoNotIncludeTests.class` import option.

**Anti-pattern 2: Hardcoded class names**
```java
.haveSimpleName("AccountService")   // breaks on rename
```
Use predicates / annotations.

**Anti-pattern 3: Rule disabled with @Ignore**
Violation found → temporarily ignore. Comment with TODO + ticket reference.

**Anti-pattern 4: All-or-nothing rules**
Migrating legacy code → rule too strict, all tests fail. Phased rollout:
```java
.allowEmptyShould(true)
.ignoreDependencyByExclusion("com.bank.legacy..")
```

**Anti-pattern 5: No banking-domain rules**
Generic hexagonal rules OK. Banking-specific (no double for money, audit immutable, PII converter) **more valuable**.

**Anti-pattern 6: Architecture test out of CI**
ArchUnit test must be in CI. Otherwise drift.

**Anti-pattern 7: Test failure not actionable**
Rule failure → "rule X violated" generic message. `.as("...")` clear explanation + fix direction.

**Anti-pattern 8: Reflection rule check**
Slow. ArchUnit static bytecode analysis.

**Anti-pattern 9: Rule writes business logic**
Rule = static structural. "Method X must call Y" — questionable. Architecture not behavior.

**Anti-pattern 10: One mega test class**
Categorize: Hexagonal, Naming, Banking-Domain, Security, KVKK separate test classes.

---

## Önemli olabilecek araştırma kaynakları

- ArchUnit user guide
- "Building Evolutionary Architectures" — Neal Ford
- Hexagonal architecture references
- Banking domain DDD references

---

## Mini task'ler

### Task 12.4.1 — Setup + first rule (30 dk)

ArchUnit dependency. Package structure rule first.

### Task 12.4.2 — Hexagonal layer rules (60 dk)

Domain isolation, application not depend adapters.

### Task 12.4.3 — Naming conventions (45 dk)

Controller/Repository/Service/Port suffix.

### Task 12.4.4 — Money BigDecimal rule (45 dk)

Fields amount/balance/fee → BigDecimal only.

### Task 12.4.5 — PII converter rule (45 dk)

tcKimlik/email/phone → @Convert annotated.

### Task 12.4.6 — No direct balance update (60 dk)

Only LedgerService accesses JournalEntryRepository.

### Task 12.4.7 — No PAN in DTO except card (45 dk)

PII boundary rule.

### Task 12.4.8 — Cyclic dependencies (30 dk)

Slices().beFreeOfCycles().

### Task 12.4.9 — Custom predicate + condition (60 dk)

Sensitive endpoint @Audited check.

### Task 12.4.10 — CI integration + actionable failures (30 dk)

ArchUnit tests in CI. Failure message clear.

---

## Test yazma rehberi

```java
@AnalyzeClasses(
    packages = "com.bank",
    importOptions = {ImportOption.DoNotIncludeTests.class, ImportOption.DoNotIncludeJars.class}
)
class BankingArchitectureTest {
    
    @ArchTest
    static final ArchRule domainIsIsolated = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
            .resideInAnyPackage("..infrastructure..", "..adapter..", "..rest..", "..persistence..", 
                              "org.springframework..", "jakarta.persistence..")
        .as("Domain layer must be infrastructure-agnostic");
    
    @ArchTest
    static final ArchRule moneyIsBigDecimal = fields()
        .that().haveNameMatching(".*[aA]mount.*|.*[bB]alance.*|.*[fF]ee.*")
        .and().areDeclaredInClassesThat().resideInAPackage("..domain..")
        .should().haveRawType(BigDecimal.class)
        .as("Banking money fields MUST be BigDecimal");
    
    @ArchTest
    static final ArchRule noDirectBalanceMutation = noClasses()
        .that().resideOutsideOfPackage("..ledger..")
        .should().callMethodWhere(target -> 
            target.getOwner().getName().endsWith("JournalEntryRepository")
            && target.getName().startsWith("save"))
        .as("Only ledger module may persist journal entries (audit + invariant safety)");
    
    @ArchTest
    static final ArchRule piiFieldsEncrypted = fields()
        .that().haveName("tcKimlik").or().haveName("phone").or().haveName("email")
        .and().areDeclaredInClassesThat().areAnnotatedWith(Entity.class)
        .should().beAnnotatedWith(Convert.class)
        .as("PII fields MUST use AttributeConverter (KVKK)");
}
```

---

## Claude-verify prompt

```
ArchUnit rules'larımı banking-grade kriterlere göre değerlendir:

1. Hexagonal:
   - Domain isolation (no infrastructure dep)?
   - Application no adapter dep?
   - Layered architecture rule?

2. Naming:
   - Controller/Repository/Service/Port suffix?
   - Event/Command/Query naming?

3. Banking domain:
   - Money fields BigDecimal only?
   - No double for money?
   - PII fields @Convert?
   - Ledger access only via LedgerService?
   - PAN restricted to card module?

4. Security:
   - No field injection (@Autowired)?
   - Sensitive endpoints @PreAuthorize?
   - Audit immutable?

5. Spring:
   - @Transactional on services?
   - Controllers don't use Repository directly?

6. Package structure:
   - No cyclic dependencies?
   - Module boundary respect?
   - Service-to-service via REST/event only?

7. Custom rules:
   - Banking-domain custom predicate?
   - Actionable failure messages?

8. Test categorization:
   - Hexagonal / Naming / Domain / Security / KVKK separate?

9. CI integration:
   - ArchUnit in mvn test?
   - Fail PR on violation?

10. Anti-pattern:
    - DoNotIncludeTests yok YOK?
    - Hardcoded class names YOK?
    - All-or-nothing strict legacy YOK?
    - Behavioral rules (not structural) YOK?
    - One mega test class YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Hexagonal layer rules
- [ ] Naming convention rules
- [ ] Banking domain rules (money, PII, ledger access)
- [ ] Security rules (@PreAuthorize, audit)
- [ ] Spring-specific rules
- [ ] Cyclic dependency check
- [ ] Module boundary
- [ ] 1 custom predicate + condition
- [ ] CI integrated
- [ ] 15+ rule total

---

## Defter notları (10 madde)

1. "ArchUnit code review automation + architecture rule enforcement banking: ____."
2. "Hexagonal layer rules (domain isolation + dependency direction): ____."
3. "Naming convention rules (Controller/Repository/Service/Port suffix): ____."
4. "Banking money BigDecimal rule (no double anywhere): ____."
5. "PII @Convert rule + ledger access restriction banking: ____."
6. "Cyclic dependency check + module boundary respect: ____."
7. "Custom predicate + condition banking-domain specific: ____."
8. "Test categorization (Hexagonal/Naming/Domain/Security/KVKK): ____."
9. "Actionable failure messages `.as()` clear fix direction: ____."
10. "CI integration + PR block on architecture violation: ____."
