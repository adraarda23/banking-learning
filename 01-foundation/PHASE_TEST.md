# Faz 1 — PHASE TEST

```admonish question title="Bu test ne işe yarar?"
Faz 2'ye geçmeden önce kendini sına. Bu bir sınav değil, **dürüstlük kontrolü**:
işaretleyemediğin her madde, ileride hangi topic'e geri döneceğini gösterir.
Hepsine "evet" diyebiliyorsan hazırsın.
```

## Pratik test — projeden

- [ ] `core-banking` repo'su GitHub'da
- [ ] `mvn test` lokalde geçiyor, hiçbir test fail değil
- [ ] JaCoCo coverage raporunda line coverage ≥ %75
- [ ] `docker compose up -d` + `mvn spring-boot:run` ile uygulama ayağa kalkıyor
- [ ] Manuel smoke test'in 9 senaryosu da çalıştı (mini-project'in son adımı)
- [ ] OpenAPI UI'da tüm endpoint'ler görünür
- [ ] `/actuator/health` UP
- [ ] Bir tane error response gözlerimle gördüm, `application/problem+json` content type
- [ ] traceId log'da ve response header'ında geçiyor

## Konsept testi — soruları cevapla, defterinde yazılı olsun

### Mimari
- [ ] Hexagonal architecture'ın temel prensibini 3 cümlede anlatabilirim
- [ ] Port ve Adapter arasındaki farkı bir junior dev'e açıklayabilirim
- [ ] Domain class'ta JPA annotation'ı neden istemem? Cevabım hazır.
- [ ] Anemic Domain Model anti-pattern'i tanıyabilirim
- [ ] DDD'nin entity vs value object farkı (örnek `Account` ve `Money` ile)

### Proje ve config
- [ ] Spring Boot `starter-*` paketlerin ne olduğunu biliyorum
- [ ] `@Value` ile `@ConfigurationProperties` farkını ve hangisini ne zaman kullandığımı söyleyebilirim
- [ ] Spring config source precedence'ını (yml < env < cli) tanıyorum
- [ ] Production'da neden `application.yml`'a password yazılmadığını biliyorum
- [ ] 3 profile (dev, test, prod) ayrımı yaptım

### Para işleme
- [ ] `double` ile para işlemi yapmamamın sebebini IEEE 754 perspektifinden açıklayabilirim
- [ ] `new BigDecimal(0.1)`'ın tehlikesi nedir, neden String veya valueOf?
- [ ] `BigDecimal.equals` ve `compareTo` farkını test'le gördüm
- [ ] HALF_EVEN'in 2.5 → 2 ama 3.5 → 4 sonucunu açıklayabilirim
- [ ] JPY (scale 0), TRY (scale 2), BHD (scale 3) farkını handle ediyor
- [ ] Multi-currency conversion'da round-trip kaybının kaynağı

### DB migration
- [ ] Flyway'in çalışma prensibini anlatabilirim
- [ ] Migration dosyası uygulandıktan sonra edit edersem ne olur?
- [ ] Banking'de rollback nasıl yapılır (forward-fix)?
- [ ] Expand-contract pattern (zero-downtime) 3 adımı
- [ ] `ddl-auto: validate` neden `create` veya `update` değil
- [ ] TestContainers H2'den daha iyi çünkü ____

### API design
- [ ] Entity'i HTTP'ye sızdırmamamın 5 sebebi
- [ ] Request ve Response DTO'larını neden ayırıyorum
- [ ] MapStruct compile-time mapper avantajı vs manuel
- [ ] Lombok'u domain class'ında neden kullanmıyorum
- [ ] `record` JPA entity için neden uygun değil
- [ ] Idempotency-Key pattern'i neden gerekli
- [ ] HTTP 422 vs 400 farkı banking context'inde

### Validation
- [ ] Bean Validation 3 katman (syntactic, semantic, domain)
- [ ] `@NotNull` vs `@NotBlank` farkı
- [ ] Custom validator 3 adım (annotation, validator class, kullanım)
- [ ] IBAN mod-97 algoritmasını öğrendim, kodumda çalışıyor
- [ ] T.C. Kimlik No checksum (10. ve 11. hane kuralları)
- [ ] Bean Validation içinde external call neden YOK

### Error handling
- [ ] RFC 7807'nin 5 standart alanı
- [ ] Banking'de stacktrace response'a neden yazılmaz
- [ ] Account enumeration attack ve koruması
- [ ] `application/problem+json` content type'ın amacı
- [ ] traceId nereden gelir, nereye gider (request → MDC → log → response header)
- [ ] Error code naming convention (UPPER_SNAKE_CASE, kategori prefix, stabil)

## Banking domain anlama

- [ ] Double-entry accounting prensibini anladım: her transfer = 1 journal_entry + 2 journal_lines (1 DEBIT + 1 CREDIT)
- [ ] Account aggregate root'unun sorumluluğu: kendi state'ini tutarlı tutmak
- [ ] Optimistic locking için neden `version` kolonu kullanıyoruz (Phase 2'de detay)
- [ ] Domain'de `@Transactional` annotation'ı yok — application service'de
- [ ] TR para birimleri (TRY, TL, TRL) tarihini biliyorum
- [ ] Idempotency banking'de neden kritik

## Test alışkanlığı

- [ ] AssertJ kullanmaya alıştım (Hamcrest yerine)
- [ ] `@ParameterizedTest` ile birden fazla input test ediyorum
- [ ] Test Data Builder (Object Mother) pattern'i uyguladım
- [ ] `@WebMvcTest` ile controller layer ayrı test ettim
- [ ] `@SpringBootTest` + TestContainers ile gerçek DB integration test'i
- [ ] Test naming `shouldXxx_WhenYyy` veya `methodName_scenario_expected` formatında
- [ ] Bir test yazmadan önce **neyi test edeceğimi** anlıyorum (TDD'ye yatkın)

## Soft skills

- [ ] Notion/Obsidian'da öğrenme defterim aktif, her topic için yazılmış notlar var
- [ ] AI'ya kod yazdırmak yerine soru sordurma diline alıştım
- [ ] Stack trace okuyup hata yerini bulabilirim (en az 3 vaka geçirdim)
- [ ] Bir konuda takıldığımda önce Stack Overflow / official docs okudum, AI'ya en son sordum
- [ ] Git commit mesajlarım anlamlı (conventional commits stiline benziyor)

## Bu fazda kaç gün harcadım?

Tahmin: ~14 gün (günde 2-3 saat). Daha az veya çok olabilir. Önemli olan **kalite**, hız değil.

Kendine sor: "Eğer bir mülakatta bana Phase 1 boyunca öğrendiğim X konuyu derinden sorsalar, **rahatça** anlatabilir miyim?"

Cevabın "evet" ise **Faz 2'ye geç** → `../02-jpa-transactions/`

Cevabın "hayır" veya emin değilim ise:
- Defter notlarını gözden geçir
- İlgili topic'in kavramlar bölümünü tekrar oku
- Bir mini-task'i daha denemekte fayda olabilir
- Acele etme — temel iyi atılırsa Faz 2 hızlı geçer

---

## Bonus — junior'dan mid-junior'a geçtiğinin işareti

Faz 1'i bitirdiğinde, şu cümleleri **kendinden ödün vermeden** söyleyebiliyorsan, junior seviyenin üst sınırındasın:

- "Production-grade Spring Boot uygulaması yazabilirim."
- "BigDecimal money precision konusunda kararlar verip uygulayabilirim."
- "Hexagonal architecture'ı pratik bir projede uygulayabilirim."
- "TestContainers ile integration test yazabilirim."
- "RFC 7807 standardına uyumlu API error handling kurabilirim."

Bunlar mid-level developer'ın bilmesi gereken şeyler. Senin Phase 1'i bitirdiğinde **kabul edilebilir junior+** seviyedesin. Phase 5 sonunda mid-junior, Phase 9 sonunda mid level.

---

```admonish success title="Sonraki durak: Faz 2"
İyi şanslar. Phase 2 daha derin ve teknik — JPA internals, transaction propagation,
locking, N+1, connection pool tuning. Faz 1 sağlam ise Faz 2 üzerine inşa olur.
→ [Faz 2 — JPA & Transactions](../02-jpa-transactions/index.md)
```
