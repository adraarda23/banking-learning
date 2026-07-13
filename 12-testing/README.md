# Phase 12 — Testing Mastery (FINAL PHASE)

## Hedef

Bir banking backend developer'ın **test disiplini**ni master seviyeye çıkarmak. Test yazmak değil, **doğru test pyramid'ini kurmak**, **test'leri belgelenmiş davranış** olarak kullanmak ve **CI pipeline'ında test'lerin gate olarak çalışmasını** sağlamak.

Bu son faz. Bittiğinde mid-level banking backend developer için gerekli tüm test araçlarını biliyor olacaksın.

## Süre

Toplam: ~3-4 hafta (günde 2-3 saat çalışma). Her topic ortalama 1.5-2 gün.

## Önbilgi

- Faz 1-11 tamamlandı
- `core-banking` projesi çalışıyor, JaCoCo coverage zaten %75+ civarı
- TestContainers temel kullanımı biliyor (Faz 1, Topic 1.4'te öğrendin)
- Spring Boot test slice'ları (`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`) biliyor
- Mockito temel (`when`, `verify`) biliyor

---

## Topic listesi

1. **[01-junit5-advanced/](01-junit5-advanced/README.md)** — JUnit 5 Jupiter Platform/Engine/API mimarisi, parametrik test'lerin tüm kaynakları, dynamic test'ler, `@Nested`, `@DisplayName`, lifecycle, conditional execution
2. **[02-mockito-deep/](02-mockito-deep/README.md)** — Mockito proxy-based çalışma şekli, `@Mock` vs `@Spy` vs `@InjectMocks`, BDDMockito, ArgumentCaptor, MockedStatic, MockedConstruction
3. **[03-testcontainers/](03-testcontainers/README.md)** — TestContainers concept, PostgreSQL/Kafka/Keycloak/Redis container'ları, container reuse, `@ServiceConnection`, wait strategies, network setup
4. **[04-archunit/](04-archunit/README.md)** — Mimariyi JUnit test'i olarak yazmak, layered/sliced architecture rule'ları, hexagonal enforcement
5. **[05-contract-testing/](05-contract-testing/README.md)** — Consumer-driven contract testing, Spring Cloud Contract, Pact framework, broker (Pactflow)
6. **[06-mutation-testing/](06-mutation-testing/README.md)** — PIT framework, mutator types, survival rate, test kalitesini ölçme, CI threshold gate
7. **[07-performance-testing-ci/](07-performance-testing-ci/README.md)** — Performance test'lerin CI'a entegrasyonu, regresyon tespiti, baseline karşılaştırma, performance gate
8. **[mini-project/](mini-project/README.md)** — Final tamamlama: tüm test araçlarını `core-banking`'e entegre etmek

---

## Test Pyramid — neden bu kadar önemli?

```
                    /\
                   /  \
                  / E2E\          ← çok az (10-20 tane), yavaş, kırılgan
                 /------\
                /        \
               /Integration\       ← orta (50-200 tane), gerçek bağımlılıklarla
              /------------\
             /              \
            /     Unit       \    ← çok (1000+ tane), hızlı, izole
           /------------------\
```

**Banking realitesi:**

| Test türü | Hız | Sayı | Bağımlılık | Amaç |
|-----------|-----|------|------------|------|
| Unit | ms | 1000+ | yok | İş kuralı doğru mu |
| Integration | sn | 50-200 | TestContainers | Adapter'lar çalışıyor mu |
| Contract | sn | 20-50 | mock'lanmış downstream | Service'ler birbirleriyle uyumlu mu |
| E2E | dk | 10-20 | tüm stack | Critical path çalışıyor mu |
| Performance | dk-saat | 5-10 senaryo | tüm stack | SLA tutuyor mu |

**TR bank pratiği:**
- Lokal `mvn test` < 30sn (sadece unit) → developer her commit'te çalıştırır
- CI'da `mvn verify` 5-10dk → her PR'de
- Nightly E2E + Performance suite 30-60dk → master branch'inde gece
- Production'da sentetik monitoring → 24/7

Yanlış piramit (anti-pattern: "ice cream cone"):
- 80% E2E, 20% unit
- Çok yavaş, çok kırılgan
- Bir testin neden patladığı belirsiz
- Developer test'leri "skip" etmeye başlar

Doğru piramit:
- 70-80% unit (hızlı, deterministik)
- 15-20% integration (gerçek adapter'lar)
- 5-10% E2E (smoke + critical path)
- + Contract test'ler (microservice boundary için)
- + Mutation test (test kalitesi metriği)
- + Performance test (regresyon koruma)

---

## Test as documentation

Bu fazın felsefesi: **test, kodun nasıl kullanılacağının canlı dokümantasyonudur**.

Bir junior `TransferService`'i nasıl kullanacağını öğrenmek istiyorsa, JavaDoc okumak yerine `TransferServiceTest`'i okumalı. Test'ler:

1. Public API kullanımını gösterir
2. Edge case'leri ortaya koyar
3. İş kurallarını formalize eder
4. Regression koruması sağlar
5. Refactoring güvenliği verir

Bu yüzden:
- Test isimleri **açıklayıcı cümleler** olmalı (`shouldRejectTransfer_WhenSourceAccountIsClosed`)
- `@DisplayName` ile Türkçe / İngilizce açıklama
- Given/When/Then yapısı (BDD) tercih
- Magic number'lar yerine **anlamlı sabitler** (`var amount = Money.of("100.00", TRY)`)
- Setup kodu `@BeforeEach` veya Test Data Builder'da

---

## Test discipline — banking gerçekleri

### "Flaky test" sorunu

Banking'de zaman zaman geçen, zaman zaman geçmeyen test bir **felakettir**. Çünkü:
- Developer test'leri ciddiye almamaya başlar
- "Yeniden çalıştır geçer" kültürü oluşur
- Gerçek bug'lar "flaky" diye atlanır

**Flaky test sebepleri:**
- `Thread.sleep()` kullanımı (timing'e bağlı)
- `LocalDateTime.now()` vs sabit bir clock
- Random veri (sabit seed kullan)
- Test order'a bağımlılık (`@TestMethodOrder` ile düzelt)
- Shared state (`@DirtiesContext` veya container reset)
- External service çağrısı (mock veya TestContainers)

### "Test smell"'ler

| Smell | Açıklama | Çözüm |
|-------|----------|-------|
| Slow test | Tek test > 1sn | Mock kullan veya integration'a taşı |
| Brittle test | Refactor'da kırılır | Implementation detay test ediliyor |
| Mystery guest | Test dışında shared state | `@BeforeEach` ile kur |
| Test code duplication | Kopyala-yapıştır | Builder, helper |
| Conditional test logic | `if/else` test'in içinde | İki ayrı test yaz |
| Eager test | Tek metotta 10 assertion | Bir konsept = bir test |

### Coverage hedefleri

| Metric | Target | Tool |
|--------|--------|------|
| Line coverage | ≥ %80 | JaCoCo |
| Branch coverage | ≥ %70 | JaCoCo |
| Mutation coverage | ≥ %70 | PIT |
| Critical path E2E | %100 | Cucumber/REST-Assured |

**Önemli not:** Coverage hedef değil, **alttan limit**. %95 coverage olan kötü test suite olabilir. %75 coverage olan iyi test suite olabilir. Mutation testing gerçek kaliteyi gösterir.

---

## Bu fazda öğreneceğin araçlar

| Araç | Amaç | Lisans |
|------|------|--------|
| JUnit 5 (Jupiter) | Test framework | EPL 2.0 (free) |
| AssertJ | Fluent assertion | Apache 2.0 |
| Mockito | Mock framework | MIT |
| TestContainers | Real services in tests | MIT |
| ArchUnit | Architecture as test | Apache 2.0 |
| Spring Cloud Contract | Contract testing | Apache 2.0 |
| Pact (Pact-JVM) | Consumer-driven contract | MIT |
| PIT (PITest) | Mutation testing | Apache 2.0 |
| Gatling | Performance testing | Apache 2.0 |
| JaCoCo | Code coverage | EPL 2.0 |
| WireMock | HTTP mock | Apache 2.0 |
| REST-Assured | API E2E test | Apache 2.0 |

Hepsi açık kaynak. TR banka stack'inde 8-10'u standart.

---

## Tamamlama kriterleri (faz bazlı)

- [ ] 7 topic + mini-project tamamlandı
- [ ] `core-banking` projesinde JaCoCo line coverage ≥ %80
- [ ] PIT mutation score ≥ %70
- [ ] ArchUnit rule'ları geçiyor (hexagonal architecture enforced)
- [ ] Contract test çalışıyor (account-service ↔ fraud-service)
- [ ] CI pipeline'ında performance gate çalışıyor (p99 < 200ms)
- [ ] PHASE_TEST.md'deki tüm sorulara cevap verebiliyorsun

Bu fazı bitirdiğinde **12 fazlık banking learning track'i** tamamlanmış olacak. CV'ne `core-banking` projeni koyabilir, TR mid-level banking backend mülakatlarına girebilirsin.

---

## Defter notları

Faz boyunca defterine şu soruların cevaplarını yaz:

1. "Test piramidinin 3 katmanını ve her birinin amacını anlatabilir miyim?"
2. "JUnit 4'ten JUnit 5'e geçişin avantajları neler?"
3. "Mockito'nun `@Mock` ve `@Spy` farkını ne zaman kullanırım?"
4. "TestContainers H2'den hangi durumlarda daha üstündür?"
5. "ArchUnit ile hangi mimari kuralları enforce ederim?"
6. "Consumer-driven contract testing'in microservice mimarisindeki rolü nedir?"
7. "Mutation testing'in line coverage'a kıyasla avantajı nedir?"
8. "CI'da performance regression gate nasıl çalışır?"

---

İyi şanslar. Bu son faz — disiplini en yüksek tutman gereken faz. Banking'de test kalitesi, kod kalitesinin **birinci ölçütü**dür.
