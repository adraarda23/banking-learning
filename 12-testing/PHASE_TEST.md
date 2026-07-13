# Faz 12 — PHASE TEST (FINAL)

12 phase roadmap'i tamamlamadan önce kendini sına. Bu son sınav — TR banka **mid+/senior backend engineer** profili kontrol.

## Pratik test

- [ ] JUnit 5 advanced (parametric matrix + dynamic + @Nested + custom assertions) all 10 service
- [ ] Mockito STRICT_STUBS + @Captor banking + 5 scenario per service
- [ ] Testcontainers full stack (Postgres + Kafka + Keycloak + Redis + LocalStack + Vault)
- [ ] LedgerInvariantExtension @AfterEach trial balance check
- [ ] ArchUnit 50+ rule (hexagonal + naming + banking domain + security)
- [ ] Spring Cloud Contract + Pact Broker + can-i-deploy gate
- [ ] Avro Schema Registry BACKWARD compatibility banking events
- [ ] PIT mutation testing — banking critical class ≥ 85%
- [ ] JMH benchmark PR vs main 10% regression gate
- [ ] k6 SLO-driven CI (smoke per PR, load nightly, soak weekly)
- [ ] 8 banking-specific kasten kırma senaryo end-to-end test
- [ ] Banking quality Grafana dashboard
- [ ] Quality gates pipeline (unit + integration + archunit + contract + mutation + benchmark + smoke)

## Konsept testi

### JUnit 5 Advanced
- [ ] JUnit 5 architecture (Platform + Jupiter + Vintage)
- [ ] @ParameterizedTest 4 source banking matrix
- [ ] @TestFactory dynamic banking
- [ ] @Nested + @DisplayName Turkish readability
- [ ] AssertJ fluent + custom domain assertions
- [ ] PER_CLASS vs PER_METHOD lifecycle
- [ ] Custom extensions banking (LedgerInvariant, MdcCleanup)
- [ ] Parallel execution + @Execution SAME_THREAD
- [ ] @Tag + Maven Surefire/Failsafe profile
- [ ] Anti-pattern (order dep, sleep, magic numbers, shared static)

### Mockito Deep
- [ ] Test doubles (Mock / Stub / Spy / Fake) banking ayrımı
- [ ] STRICT_STUBS production-grade
- [ ] when/thenReturn/thenThrow/thenAnswer stubbing
- [ ] ArgumentCaptor banking journal + event payload
- [ ] ArgumentMatchers + all-or-nothing rule
- [ ] verify (times/never/InOrder) banking saga compensation
- [ ] @Spy partial mock + doReturn safe
- [ ] BDDMockito given/willReturn/then/should
- [ ] MockedStatic legacy + Clock injection preferred
- [ ] Anti-pattern (over-mock, mock value, mock what you don't own)

### Testcontainers
- [ ] vs H2/Mock production parity banking
- [ ] Singleton + withReuse + @ServiceConnection performance
- [ ] Per-test state clean + ledger invariant @AfterEach
- [ ] Keycloak realm import + token + secured endpoint
- [ ] LocalStack KMS + Vault transit banking encryption
- [ ] @ServiceConnection Spring Boot 3.1+ auto-config
- [ ] Wait.forHttp health-ready vs forListeningPort
- [ ] CI Testcontainers reuse off + cache
- [ ] Network isolation multi-container
- [ ] Anti-pattern (per-test container, hardcoded port, no init script)

### ArchUnit
- [ ] Code review automation + architecture enforcement
- [ ] Hexagonal layer rules (domain isolation)
- [ ] Naming convention (Controller/Repository/Service/Port)
- [ ] Banking money BigDecimal (no double)
- [ ] PII @Convert + ledger access restriction
- [ ] Cyclic dependency check + module boundary
- [ ] Custom predicate + condition banking-domain
- [ ] Test categorization (Hexagonal/Naming/Domain/Security/KVKK)
- [ ] Actionable failure messages `.as()`
- [ ] CI integration + PR block on violation

### Contract Testing
- [ ] CDC vs end-to-end microservice trade-off
- [ ] Spring Cloud Contract vs Pact banking pick
- [ ] Stub publish + consumer @AutoConfigureStubRunner
- [ ] Pact Broker can-i-deploy CI gate
- [ ] Kafka event contract message stub
- [ ] Avro Schema Registry BACKWARD compat
- [ ] Banking strict matcher (money decimal, ISO date, enum)
- [ ] OpenAPI diff breaking change detect
- [ ] Provider @State independent isolation
- [ ] Contract = API design doc auto-generated

### Mutation Testing
- [ ] Coverage vs Mutation score execution vs verification
- [ ] PIT mutator types DEFAULT vs STRONGER + BIG_DECIMAL banking
- [ ] Banking critical class hedef (Ledger/Interest/Fraud/IBAN/Encryption)
- [ ] %90+ critical + %60+ DTO trade-off
- [ ] Incremental + git diff PR-only
- [ ] HALF_EVEN vs HALF_UP rounding mutation detect
- [ ] ACT/360 vs ACT/365 day count mutation
- [ ] CI gate threshold + PR comment
- [ ] Equivalent mutants false positive review
- [ ] Mutation = test quality metric (not replace integration/e2e)

### Performance Testing CI
- [ ] Pipeline placement (PR smoke + nightly load + pre-release suite)
- [ ] JMH PR vs main regression gate + PR comment
- [ ] k6 SLO-matching thresholds + PR preview
- [ ] Nightly load + staging refresh + baseline compare
- [ ] Banking realistic load (endpoint mix + think time + Pareto amount)
- [ ] Baseline long-term store + Grafana trend
- [ ] Dedicated CI runner (CPU pin + turbo off)
- [ ] Pre-release full suite gate
- [ ] Slack on regression
- [ ] Anti-pattern (shared runner + absolute threshold + no baseline)

## Banking quality matrix

| Dimension | Target | Banking critical |
|---|---|---|
| Unit test coverage | 80%+ | Ledger/Interest/Fraud 95% |
| Mutation score | 85% overall | Critical 90%+ |
| Integration tests | 25+ per service | Ledger invariant @AfterEach |
| Contract tests | All service pairs | Avro events too |
| ArchUnit rules | 50+ | Banking domain 30+ |
| JMH benchmarks | 20+ | BigDecimal + Lock + Cache |
| Load test SLO | p99 < 1s | Transfer service |
| Error rate SLO | < 0.5% | All endpoints |

## Banking compliance check

- [ ] BDDK denetim: test coverage + mutation audit-able
- [ ] KVKK: PII rule (ArchUnit) + crypto-shred test (Testcontainers)
- [ ] PCI-DSS: PAN tokenization (mutation test rounding/encryption)
- [ ] MASAK: fraud rule mutation score 90%+
- [ ] Audit trail: hash chain test + tamper detect
- [ ] Banking ledger invariant: extension every integration test
- [ ] Idempotency: under-load test (Testcontainers + concurrency)
- [ ] Sanctions screening: real-time integration test
- [ ] Saga compensation: InOrder verify (Mockito) + integration (Testcontainers)
- [ ] Encryption: tampering test + KMS roundtrip

## Soft skills

- [ ] 20 mini-project defter notu yazılmış
- [ ] Test pyramid balance internalize (unit > integration > e2e)
- [ ] Quality gate strategy design edebilirim
- [ ] Banking compliance test mapping yapabilirim
- [ ] Team convention (naming + architecture) ArchUnit ile enforce
- [ ] Performance regression detection pipeline design

## Kaç gün?

Tahmin: 35-40 gün (günde 3 saat). Phase 12 = **quality assurance maturity** banking için. Bu phase'i bitiren engineer **testing strategy** sahibi.

---

Hepsine "evet" → **12 phase complete! TR banka senior interview'a hazır.**

---

## Bonus — banking senior engineer işareti

Phase 12 bittikten sonra şu cümleleri söyleyebiliyorsan:

- "Banking microservice test strategy (unit + integration + contract + mutation + perf) tasarlayıp implement edebilirim."
- "JUnit 5 + AssertJ + custom domain assertions ile banking acceptance test'leri yazabilirim."
- "Mockito STRICT_STUBS + @Captor + InOrder ile banking saga + compensation verify."
- "Testcontainers full stack + ledger invariant extension banking integration test."
- "ArchUnit 50+ rule + banking domain custom (money + PII + ledger access)."
- "Spring Cloud Contract + Pact Broker + can-i-deploy gate microservice safety."
- "Avro Schema Registry BACKWARD compat banking event evolution."
- "PIT mutation testing banking critical class %90+ + git diff PR-only."
- "JMH benchmark CI + 10% regression gate + PR comment automation."
- "k6 SLO-driven load + nightly + soak + pre-release suite banking."
- "Banking quality dashboard + KVKK + BDDK + PCI-DSS compliance test mapping."

TR bankalarında **senior backend engineer** profili tamamlanmış. Banking için **complete competency**:

```
Technical: Java/Spring/Kafka/K8s ✓
Domain: TR banking + accounting + regulatory ✓
Operational: observability + DevOps + testing ✓
Soft: communication + leadership + mentoring (next)
```

---

## 12 phase complete!

```
Phase 1:  Foundation ✓
Phase 2:  Hexagonal + DDD ✓
Phase 3:  JPA + Postgres ✓
Phase 4:  SQL + Oracle ✓
Phase 5:  Spring Batch ✓
Phase 6:  Kafka + Messaging ✓
Phase 7:  Microservices ✓
Phase 8:  Security ✓
Phase 9:  Observability ✓
Phase 10: Banking Domain ✓
Phase 11: DevOps ✓
Phase 12: Testing ✓
```

**TR banka mid+/senior backend engineer interview'a hazırsın.**

---

## Sırada ne?

### Yakın vade
- Mock interview pratiği (banking-specific case study)
- Real banking case writeup
- Bir TR bank için CV update (Phase 10 mini-project highlight)
- LinkedIn profile + portfolio site

### Orta vade
- Open-source contribution
  - Spring projects
  - jPOS (ISO 8583)
  - Prowide (SWIFT)
  - Banking-related libs
- Conference talk submission
  - Devnot
  - JavaTurkey
  - Yazılım Tarafı
- Blog post serisi (banking domain + tech)

### Uzun vade
- Senior banking architect role
- Banking platform lead
- BDDK/TBB regulatory contribution
- Banking tech book author / mentor

---

## Phase 12'nin yeri

Phase 12 = **quality bar setter**. Senior engineer kalitesi:

- Junior: writes tests
- Mid: writes good tests
- **Senior: designs test strategy + sets quality bar + enforces convention**

Phase 12 ile **TR bank senior backend engineer** profili tamamlandı.

Banking için **complete profile**:
- Technical mastery (Phase 1-7)
- Security + compliance (Phase 8)
- Operational excellence (Phase 9, 11)
- Domain expertise (Phase 10)
- Quality assurance (Phase 12)

TR banka sektöründe **mid+/senior rolü** için competitive edge sağlandı.

İyi çalışmalar — başarıyla yol al.
