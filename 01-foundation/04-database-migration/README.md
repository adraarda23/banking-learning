# Topic 1.4 — Database Migration: Flyway

```admonish info title="Bu bölümde"
- Schema değişikliklerini versiyonlu SQL dosyalarıyla kod gibi yönetmeyi öğreneceksin
- Flyway'in çalışma prensibini, dosya adlandırma kurallarını ve migration türlerini (Versioned / Repeatable / Undo) kavrayacaksın
- Banking-grade migration kurallarını göreceksin: checksum disiplini, expand-contract pattern, forward-only rollback
- PostgreSQL'i Docker ile lokal kurup TestContainers ile gerçek DB üzerinde migration testleri yazacaksın
- Flyway ile Liquibase'i karşılaştırıp hangi durumda hangisinin seçildiğini anlayacaksın
```

## Hedef

Bir banking projesinde **schema değişiklikleri**ni kod gibi yönetmek. Flyway'i derinlemesine anlayıp banking-grade migration kuralları öğrenmek. PostgreSQL'i Docker ile lokal kurmak, TestContainers ile test'leri gerçek DB üzerinde çalıştırmak.

## Süre

Okuma: 1.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 1.1-1.3 tamamlandı
- SQL temel (CREATE TABLE, INSERT, ALTER) biliyorsun
- Docker temel kurulu (`docker --version`)

---

## Kavramlar

### 1. Neden migration tool gerekli?

Bir banking schema'sı 10 yılda yüzlerce kez değişir: yeni kolon, yeni tablo, index, tip değişikliği, constraint. Soru şu: bu değişiklikleri kim, hangi sırayla, hangi ortamda uyguladı?

Kötü senaryoyu bilirsin: DBA elle production'da SQL çalıştırır, dev ile prod arasında schema farkı oluşur, "hangi script'i çalıştırdık?" karmaşası başlar. Yeni gelen developer'a "lokali kur" demek 4 saatlik çile demektir.

İyi yol ise değişiklikleri kod gibi yönetmek:

- Schema değişiklikleri **versiyonlu SQL dosyaları** olarak repo'da durur
- Uygulama başlarken veya pipeline'da otomatik uygulanır
- Hangi versiyona kadar gelindiği **DB'deki bir tabloda** yazılıdır
- Yeni developer: `mvn spring-boot:run` → DB otomatik kurulur

Bu disiplini **migration tool**'lar sağlar. En yaygın ikisi: **Flyway** (basit, dosya-bazlı, SQL odaklı — TR bankalarında yaygın) ve **Liquibase** (XML/YAML desteği, daha esnek ama karmaşık). Bu projede Flyway kullanıyoruz; Liquibase'i sondaki karşılaştırmada tanıyacaksın.

### 2. Flyway temelleri

#### Çalışma prensibi

Flyway'in mantığı basit: dosyalara bak, DB'deki kayıtla karşılaştır, eksikleri sırayla uygula.

1. `src/main/resources/db/migration/` klasörü taranır
2. DB'de `flyway_schema_history` tablosu otomatik yaratılır
3. Henüz uygulanmamış migration'lar **sırayla** çalıştırılır, her biri history'e yazılır
4. Aynı dosya **bir kez** uygulanır — sonradan değiştirmek = checksum hatası

```mermaid
flowchart LR
    A["Uygulama başlar"] --> B["Migration klasörü taranır"]
    B --> C["History tablosu okunur"]
    C --> D{"Yeni migration var mı?"}
    D -->|"Evet"| E["Sırayla uygula, history'e yaz"]
    E --> D
    D -->|"Hayır"| F["Checksum doğrula"]
    F --> G["Uygulama devam eder"]
```

#### Dosya adlandırma

```
V1__create_accounts_table.sql
V2__add_balance_column.sql
V3__create_journal_entries_table.sql
V4.1__add_index_on_account_owner.sql
V20251215.1__add_iban_column.sql
```

Format şöyle çözülür: `V` prefix + version number (dot ile ayrılabilir: `1`, `1.1`) + **iki underscore** (`__`) + description + `.sql`. En sık yapılan hata tek underscore koymak — Flyway dosyayı görmezden gelir.

Sıralama string değil **numeric** yapılır: `V1.10 > V1.2` çünkü 10 > 2.

```admonish tip title="İpucu"
**Banking pratiği:** Timestamp-based versioning (`V20251215_120000__add_iban.sql`) ekip içinde aynı versiyon çakışmasını engeller. Küçük ekipte sequential (`V1`, `V2`, ...) yeterli.
```

#### Migration türleri

| Tür | Prefix | Davranış | Ne için |
|---|---|---|---|
| Versioned | `V` | Bir kez uygulanır, checksum kontrolü var | Tablo, kolon, index — asıl iş |
| Repeatable | `R` | İçerik değişince yeniden uygulanır | View, stored procedure, function |
| Undo | `U` | Flyway Pro/Teams (paralı), open source'ta YOK | Banking'de zaten kullanmayız |

Repeatable örneği: `R__create_balance_summary_view.sql` — bu dosyayı düzenleyebilirsin, checksum değişince Flyway yeniden uygular.

Undo yoksa rollback nasıl olur? Forward migration ile düzeltme — Kural 6'da anlatacağım.

### 3. İlk migration — `accounts` tablosu

Teoriyi gördün, şimdi gerçek bir banking tablosu yazalım. `src/main/resources/db/migration/V1__create_accounts_table.sql`:

```sql
CREATE TABLE accounts (
    id              UUID PRIMARY KEY,
    owner_id        UUID NOT NULL,
    currency        CHAR(3) NOT NULL,
    balance_amount  NUMERIC(19, 4) NOT NULL DEFAULT 0.00,
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    opened_at       TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    closed_at       TIMESTAMP WITH TIME ZONE,
    version         BIGINT NOT NULL DEFAULT 0,
    
    CONSTRAINT chk_account_status CHECK (status IN ('ACTIVE', 'FROZEN', 'CLOSED')),
    CONSTRAINT chk_balance_currency_match CHECK (currency ~ '^[A-Z]{3}$')
);

CREATE INDEX idx_accounts_owner_id ON accounts(owner_id);
CREATE INDEX idx_accounts_status ON accounts(status) WHERE status != 'CLOSED';
```

Her satırın bir sebebi var:

- **`UUID` primary key** — distributed sistem için iyi (Phase 7'de microservice'e bölünce). Sequential `BIGSERIAL` de yaygın, banka tercihine bağlı.
- **`NUMERIC(19, 4)`** — `BigDecimal` karşılığı. 4 decimal ileride mikro-faiz için, 19 basamak ~10^15 TRY'ye yeter.
- **`CHAR(3)`** — currency ISO kodu sabit 3 karakter.
- **`status` CHECK constraint** — domain enum'una karşılık DB-level guarantee.
- **`version` BIGINT** — optimistic locking için (Phase 2'de detay).
- **`TIMESTAMP WITH TIME ZONE`** — banka multi-region olabilir. Bare `TIMESTAMP` yerine **her zaman** `TIMESTAMPTZ`.
- **Partial index** (`WHERE status != 'CLOSED'`) — kapalı hesaplara index ayırmak gereksiz; küçük index hızlı index demektir.

### 4. Banking migration kuralları (zor öğrenilenler)

Bu 7 kuralın her biri production'da yaşanmış bir acıdan doğdu. Tek tek gidelim.

#### Kural 1: Uygulanmış migration'ı edit etme

```admonish warning title="Dikkat"
Bir migration uygulandıktan sonra dosyayı değiştirme. Flyway checksum tutar — dosya içeriği değişirse validate aşamasında hata alırsın.
```

`V2__add_status_column.sql` uygulandı, checksum kaydedildi. Sonradan içeriği değiştirirsen:

```
ERROR: Validate failed: Migration checksum mismatch for migration version 2
```

**Doğru yol:** Yeni migration ekle (`V3__alter_status_column.sql`).

#### Kural 2: Defensive yaz

Flyway her migration'ı bir kez çalıştırır, yani `CREATE TABLE accounts` normalde ikinci kez çalışmaz. Yine de defensive yazmak iyi alışkanlıktır:

```sql
CREATE TABLE IF NOT EXISTS accounts (...);
```

```admonish tip title="İpucu"
Defensive yazım (`IF NOT EXISTS` gibi) acil onarımda script'i elle çalıştırman gerektiğinde hayatını kurtarır.
```

#### Kural 3: Migration atomic olmayabilir

DDL statement'leri her DB'de transaction içinde çalışmaz. Yani multi-statement bir migration **yarıda kalabilir**:

- **PostgreSQL:** DDL transaction içinde çalışır, rollback mümkün. Flyway tüm migration'ı tek transaction'da çalıştırır (default).
- **Auto-commit DB'ler (Oracle 11g, MySQL 5.x):** Her DDL implicit commit. Yarıda kalma → manuel temizlik.

Banking'de Oracle yaygın olduğundan alışkanlık edin: migration'ları **küçük parçalara böl**, her biri 1-2 statement.

#### Kural 4: Migration yarışı

İki app instance aynı anda başlarsa ikisi de migration uygulamaya çalışır. Flyway DB-level lock kullanır: ikinci bekler, sonra yapılacak iş kalmadığını görür. Genelde sorunsuz.

Ama **çok büyük migration** (örn. 1M satırlık tabloya kolon ekleme + populate) sırasında diğer app'ler timeout alabilir. Banking'de bu yüzden migration **ayrı bir job'da** çalıştırılır, app'ler "schema hazır" varsayar:

```yaml
spring:
  flyway:
    enabled: false   # app'te kapat
```

CI/CD pipeline'da explicit Flyway job çalıştırılır.

#### Kural 5: Backward-compatible migration (expand-contract)

Banking'de **zero-downtime deployment** standarttır: deploy sırasında eski ve yeni kod versiyonu aynı anda çalışır. Migration ikisiyle de uyumlu olmalı.

**Yanlış:**
```sql
ALTER TABLE accounts RENAME COLUMN balance_amount TO balance;
```

Eski versiyon hâlâ `balance_amount` arıyor — anında patlar.

**Doğru — 3 adımlı expand-contract, 3 ayrı release:**

1. **Expand:** `ALTER TABLE accounts ADD COLUMN balance NUMERIC(19,4); UPDATE accounts SET balance = balance_amount;` — uygulama iki kolona da yazar
2. **Migrate:** Yeni versiyon `balance` kolonunu okuyacak şekilde deploy edilir
3. **Contract:** Eski versiyon retire olunca `ALTER TABLE accounts DROP COLUMN balance_amount;`

```mermaid
flowchart LR
    A["Release 1: Expand<br/>yeni kolonu ekle"] --> B["Release 2: Migrate<br/>kod yeni kolonu okur"]
    B --> C["Release 3: Contract<br/>eski kolonu sil"]
```

#### Kural 6: Rollback = forward-fix

Banking'de migration rollback demek **geri almak değil, ileri düzelten yeni migration yazmak** demektir:

```
V5__add_fee_column.sql              ← üretime gitti
V6__revert_fee_column.sql           ← V5'i geri al (DROP COLUMN fee)
```

Sebep: migration history tutarlı kalır, audit trail temiz olur ve "önceki versiyona dön" gerçek dünyada zaten problemlidir.

#### Kural 7: Schema migration ≠ data migration

```admonish warning title="Dikkat"
Büyük data migration (örn. 10M satırı yeniden hesapla) Flyway'de **çalıştırma**: saatlerce sürer ve app boot'u bloklar, atomic değildir, hata durumunda parçalı kalır.
```

Doğru iş bölümü: schema değişikliğini Flyway'le yap (`ADD COLUMN`), data populate'i ayrı bir **batch job**'a ver (Phase 5'te Spring Batch ile). Job idempotent ve restartable olmalı.

### 5. PostgreSQL Docker ile local setup

Migration'ları denemek için gerçek bir PostgreSQL lazım. `~/projects/core-banking/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: banking-postgres
    environment:
      POSTGRES_DB: banking_dev
      POSTGRES_USER: banking_dev
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U banking_dev -d banking_dev"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
docker compose up -d        # başlat
docker compose ps           # durum
docker compose logs -f      # log akışı
docker compose down         # durdur
docker compose down -v      # durdur + volume sil (sıfırdan başla)
```

`application-dev.yml`:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/banking_dev
    username: banking_dev
    password: dev_password
```

### 6. `flyway_schema_history` tablosu

Flyway'in hafızası bu tablo. İlk migration sonrası bakarsan:

```sql
SELECT * FROM flyway_schema_history;
```

| installed_rank | version | description | type | script | checksum | installed_by | installed_on | execution_time | success |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 1 | create accounts table | SQL | V1__create_accounts_table.sql | -1234567890 | banking_dev | 2025-... | 45 | true |

```admonish warning title="Dikkat"
Bu tabloyu **elle değiştirme** — sadece okumak için. Production'da locked permissions ile korunması gerekir.
```

### 7. Spring Boot entegrasyonu

`pom.xml`'da `flyway-core` ve `postgresql` driver varsa Spring Boot gerisini kendisi bağlar: DataSource hazır olunca Flyway'i çalıştırır, migration'lar biter, **sonra** Hibernate başlar ve `ddl-auto: validate` ile schema'yı doğrular.

```mermaid
sequenceDiagram
    participant App as Spring Boot
    participant FW as Flyway
    participant DB as PostgreSQL
    App->>FW: DataSource hazir, migrate baslat
    FW->>DB: flyway_schema_history oku
    FW->>DB: Yeni migration dosyalarini uygula
    DB-->>FW: Basarili, history guncellendi
    FW-->>App: Migration tamam
    App->>DB: Hibernate ddl-auto validate
    DB-->>App: Schema uyumlu, app hazir
```

**Önemli config:**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false      # production'da false bırak
    validate-on-migrate: true       # checksum kontrol
    out-of-order: false             # numara atlanamaz
    placeholder-replacement: false  # ${} yer tutucu kullanmıyoruz (security)
    fail-on-missing-locations: true # locations bulunamazsa fail
```

```admonish warning title="Dikkat"
`spring.jpa.hibernate.ddl-auto: validate` ile Hibernate **şemayı oluşturmaz**, sadece entity ile DB schema uyumunu doğrular. Schema'yı **Flyway** yönetir. `create`, `update`, `create-drop` değerleri **banking'de yasak**.
```

### 8. Migration testing

Migration da koddur, kod test edilir. Üç strateji var:

1. **TestContainers ile gerçek PostgreSQL** — her test'te taze container, migration'lar otomatik uygulanır, assertion yazarsın
2. **Shared TestContainer** — daha hızlı ama isolation azalır
3. **H2 in-memory** — hızlı ama PostgreSQL dialect'ini tam taklit etmez → banking için **uygun değil**

Banking projesinde standart: **TestContainers + PostgreSQL**.

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
class MigrationIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void accountsTableShouldExist() {
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.tables WHERE table_name = 'accounts'",
            Integer.class
        );
        assertThat(count).isEqualTo(1);
    }
    
    @Test
    void accountStatusCheckShouldRejectInvalidValue() {
        assertThatThrownBy(() ->
            jdbcTemplate.update(
                "INSERT INTO accounts (id, owner_id, currency, status) VALUES (?, ?, ?, ?)",
                UUID.randomUUID(), UUID.randomUUID(), "TRY", "INVALID"
            )
        ).isInstanceOfAny(DataIntegrityViolationException.class);
    }
}
```

Dikkat et: schema'yı `information_schema` üzerinden sorguluyoruz — tablo var mı, kolon tipi ne, constraint çalışıyor mu. `@ServiceConnection` (Spring Boot 3.1+) container'ın connection bilgilerini otomatik DataSource'a bağlar; manuel `@DynamicPropertySource` gerekmez.

### 9. Banking schema — bu bölümde yazacakların

Üç tablo yazacaksın. `accounts`'u gördün; diğer ikisi double-entry bookkeeping'in temeli:

- **`journal_entries`** — bir mali olayı temsil eder (transfer, faiz, ücret): id, transaction_id (idempotency key), description, occurred_at
- **`journal_lines`** — entry'nin alt kalemleri (debit/credit): id, journal_entry_id, account_id, direction, amount, currency

Double-entry kuralı: bir `journal_entry`'nin tüm `journal_lines`'larında toplam DEBIT = toplam CREDIT.

```sql
CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL UNIQUE,    -- idempotency key
    description     VARCHAR(500),
    occurred_at     TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by      VARCHAR(100) NOT NULL
);

CREATE TABLE journal_lines (
    id                  UUID PRIMARY KEY,
    journal_entry_id    UUID NOT NULL REFERENCES journal_entries(id),
    account_id          UUID NOT NULL REFERENCES accounts(id),
    direction           VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
    amount              NUMERIC(19, 4) NOT NULL CHECK (amount > 0),
    currency            CHAR(3) NOT NULL
);

CREATE INDEX idx_journal_lines_account ON journal_lines(account_id);
CREATE INDEX idx_journal_lines_entry ON journal_lines(journal_entry_id);
```

**Banking detayı:** `journal_lines.amount` her zaman pozitif; yönü `direction` belirler. Bu bir konvansiyon — accountant'lar böyle düşünür.

### 10. Repeatable migration örneği

View gibi tanımlar zamanla evrilir; her değişiklik için yeni `V` dosyası açmak yerine `R__` kullanırsın. `R__refresh_balance_summary_view.sql`:

```sql
DROP VIEW IF EXISTS account_balance_summary;

CREATE VIEW account_balance_summary AS
SELECT 
    a.id AS account_id,
    a.currency,
    COALESCE(SUM(
        CASE 
            WHEN jl.direction = 'CREDIT' THEN jl.amount
            WHEN jl.direction = 'DEBIT' THEN -jl.amount
        END
    ), 0) AS calculated_balance,
    a.balance_amount AS stored_balance
FROM accounts a
LEFT JOIN journal_lines jl ON jl.account_id = a.id
GROUP BY a.id, a.currency, a.balance_amount;
```

View'i değiştirmek istediğinde aynı dosyayı edit edip Flyway'i çalıştırırsın — otomatik yeniden yaratılır.

### 11. Liquibase — kısa karşılaştırma

Flyway'in ana rakibini de tanı. Liquibase değişiklikleri SQL yerine soyut bir formatta tutar:

```xml
<changeSet id="1" author="ahmet">
    <createTable tableName="accounts">
        <column name="id" type="uuid">
            <constraints primaryKey="true"/>
        </column>
        ...
    </createTable>
</changeSet>
```

| | Flyway | Liquibase |
|---|---|---|
| Format | Düz SQL | XML/YAML/JSON + SQL |
| DB-agnostic | Hayır | Evet — aynı changeset birden fazla DB'de |
| Built-in rollback | Yok (open source) | Var (`rollback` tag) |
| Öğrenme eğrisi | Düşük — SQL bilen herkes | Yüksek — format disiplini ister |
| Ekstra | — | Change preview, dry-run |

```mermaid
flowchart TD
    A["Migration tool seçimi"] --> B{"SQL odaklı ve basit mi olsun?"}
    B -->|"Evet"| C["Flyway"]
    B -->|"Hayır"| D{"Çoklu DB veya built-in rollback lazım mı?"}
    D -->|"Evet"| E["Liquibase"]
    D -->|"Hayır"| C
```

**TR banking gerçeği:** Flyway daha yaygın; bazı eski enterprise sistemler Liquibase. İkisini de bil, projende Flyway kullan.

### 12. SQL versioning kuralları (banking)

**Naming convention:**

| Nesne | Kural | Örnek |
|---|---|---|
| Tablo | `snake_case` plural | `accounts`, `journal_entries` |
| Kolon | `snake_case` | `owner_id`, `balance_amount` |
| Foreign key kolon | `<referenced_table_singular>_id` | `account_id` |
| Index | `idx_<table>_<col>` | `idx_accounts_owner_id` |
| Constraint | `chk_<table>_<rule>` | `chk_account_status` |
| Trigger | `trg_<table>_<event>` | `trg_accounts_update_audit` |

**Migration content kuralları:**
- Bir migration **bir konuya** odaklansın ve **küçük** olsun (forward-fix'i kolaylaştırır)
- Migration'da **production data'yı UPDATE etme** (idempotent olmayabilir, audit kaybı)
- Dosya başına comment yaz: ne yaptığını anlat

---

## Önemli olabilecek araştırma kaynakları

- Flyway documentation (flywaydb.org/documentation/)
- "Refactoring Databases" Scott Ambler (DB değişiklik pattern'leri)
- PostgreSQL official docs (postgresql.org/docs/)
- TestContainers documentation (testcontainers.com)
- "Database Reliability Engineering" Laine Campbell
- Liquibase documentation (karşılaştırma için)
- "Zero downtime deployments with Flyway" blog yazıları

---

## Mini task'ler

### Task 1.4.1 — PostgreSQL Docker setup (15 dk)

`core-banking/docker-compose.yml` yaz (yukarıda örnek). Çalıştır:
```bash
docker compose up -d
docker compose ps        # postgres healthy mi?
docker compose logs postgres | head
```

PostgreSQL container'ına gir, DB'i incele:
```bash
docker exec -it banking-postgres psql -U banking_dev -d banking_dev
\dt           # tabloları listele (boş)
\q
```

### Task 1.4.2 — V1: accounts table migration (30 dk)

`src/main/resources/db/migration/V1__create_accounts_table.sql` yaz.

Spec:
- UUID primary key
- `owner_id` UUID NOT NULL
- `currency` CHAR(3) NOT NULL
- `balance_amount` NUMERIC(19, 4) NOT NULL DEFAULT 0.0000
- `status` VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' with CHECK ('ACTIVE', 'FROZEN', 'CLOSED')
- `opened_at` TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
- `closed_at` TIMESTAMPTZ (nullable)
- `version` BIGINT NOT NULL DEFAULT 0

Index: `owner_id` üzerinde (partial: status != 'CLOSED').

App'i çalıştır (`mvn spring-boot:run`), migration'ın otomatik uygulandığını doğrula:

```bash
docker exec -it banking-postgres psql -U banking_dev -d banking_dev
\dt                       # accounts ve flyway_schema_history görünmeli
\d accounts               # schema'yı incele
SELECT * FROM flyway_schema_history;
```

### Task 1.4.3 — V2: journal tables migration (30 dk)

`V2__create_journal_tables.sql`:
- `journal_entries` tablosu (id, transaction_id UNIQUE, description, occurred_at, created_by)
- `journal_lines` tablosu (id, journal_entry_id FK, account_id FK, direction CHECK, amount > 0, currency)
- Index'ler

App'i restart et, V2'nin uygulandığını doğrula.

### Task 1.4.4 — V3: extra ALTER migration (15 dk)

V1'de unutulan bir şeyi V3 ile ekle. Örnek: account adına `customer_reference` kolonu (banka müşterisinin dış sistemden referans no).

`V3__add_customer_reference_to_accounts.sql`:
```sql
ALTER TABLE accounts ADD COLUMN customer_reference VARCHAR(50);
CREATE INDEX idx_accounts_customer_reference ON accounts(customer_reference) 
    WHERE customer_reference IS NOT NULL;
```

App'i restart et.

### Task 1.4.5 — Migration'ı edit et ve sonuçları gör (10 dk)

V1 dosyasını **kasten** edit et. Örneğin bir comment ekle:
```sql
-- yeni comment satırı
CREATE TABLE accounts (...);
```

App'i restart et. Aldığın hatayı **defterine yaz** (`FlywayValidateException` veya `checksum mismatch`).

Sonra dosyayı eski haline getir, restart et — düzelmeli.

```admonish tip title="İpucu"
**Bu deneyimi geçirmek lazım.** Production'da bu hatayı bir kere yedikten sonra migration'a saygın artar.
```

### Task 1.4.6 — Repeatable migration: balance summary view (30 dk)

`R__create_balance_summary_view.sql` yaz (yukarıda örnek var). Bir test query'si yaz:

```sql
SELECT * FROM account_balance_summary;
```

Boş tablo → boş sonuç. Manuel bir kaç account insert et:

```sql
INSERT INTO accounts (id, owner_id, currency, balance_amount, status)
VALUES (gen_random_uuid(), gen_random_uuid(), 'TRY', 1000.00, 'ACTIVE');
```

Query'i tekrar çalıştır, view'ün sonucunu gör.

Sonra view'i değiştir (örn. bir kolon ekle), restart et, otomatik yeniden yaratıldığını doğrula.

### Task 1.4.7 — Forward-only rollback örneği (15 dk)

`V3__add_customer_reference_to_accounts.sql`'ı silmek istediğini varsay (yanlış karar verdik). V4 yaz:

`V4__drop_customer_reference_from_accounts.sql`:
```sql
DROP INDEX IF EXISTS idx_accounts_customer_reference;
ALTER TABLE accounts DROP COLUMN customer_reference;
```

Bu yaklaşım banking pratiği — geri alma yerine ileri düzeltme.

---

## Test yazma rehberi

### Test 1.4.1 — TestContainers entegrasyonu

`pom.xml`'da var (Topic 1.2'de eklendi):
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

`src/test/java/com/mavibank/banking/migration/MigrationIntegrationTest.java`:

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
class MigrationIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void accountsTableExists() {
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.tables WHERE table_name = 'accounts'",
            Integer.class
        );
        assertThat(count).isEqualTo(1);
    }
    
    @Test
    void accountsHasCorrectColumns() {
        List<String> columns = jdbcTemplate.queryForList(
            "SELECT column_name FROM information_schema.columns WHERE table_name = 'accounts' ORDER BY ordinal_position",
            String.class
        );
        assertThat(columns).contains("id", "owner_id", "currency", "balance_amount",
                                     "status", "opened_at", "closed_at", "version");
    }
    
    @Test
    void balanceAmountIsNumericNineteenFour() {
        Map<String, Object> column = jdbcTemplate.queryForMap(
            """
            SELECT data_type, numeric_precision, numeric_scale 
            FROM information_schema.columns 
            WHERE table_name = 'accounts' AND column_name = 'balance_amount'
            """
        );
        assertThat(column).containsEntry("data_type", "numeric");
        assertThat(column).containsEntry("numeric_precision", 19);
        assertThat(column).containsEntry("numeric_scale", 4);
    }
    
    @Test
    void statusCheckRejectsInvalidValue() {
        assertThatThrownBy(() ->
            jdbcTemplate.update(
                "INSERT INTO accounts (id, owner_id, currency, status) VALUES (?::uuid, ?::uuid, ?, ?)",
                UUID.randomUUID().toString(), UUID.randomUUID().toString(), "TRY", "INVALID_STATUS"
            )
        ).isInstanceOf(DataIntegrityViolationException.class);
    }
    
    @Test
    void journalLineDirectionCheckRejectsInvalidValue() {
        // önce bir account ve journal_entry insert et, sonra invalid direction
        // ...
    }
    
    @Test
    void journalLineAmountMustBePositive() {
        // amount = 0 veya negative INSERT'i fail etsin
    }
    
    @Test
    void foreignKeyConstraintWorks() {
        // var olmayan account_id ile journal_line insert et, fail etsin
        UUID fakeAccountId = UUID.randomUUID();
        UUID journalEntryId = insertJournalEntry();
        
        assertThatThrownBy(() ->
            jdbcTemplate.update(
                "INSERT INTO journal_lines (id, journal_entry_id, account_id, direction, amount, currency) " +
                "VALUES (?, ?, ?, ?, ?, ?)",
                UUID.randomUUID(), journalEntryId, fakeAccountId, "DEBIT", new BigDecimal("100.00"), "TRY"
            )
        ).isInstanceOf(DataIntegrityViolationException.class);
    }
    
    private UUID insertJournalEntry() {
        // helper method
    }
    
    @Test
    void flywaySchemaHistoryHasAllMigrations() {
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM flyway_schema_history WHERE success = true",
            Integer.class
        );
        assertThat(count).isGreaterThanOrEqualTo(3);  // V1, V2, V3 at least
    }
}
```

### Test 1.4.2 — `application-test.yml` config

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true

logging:
  level:
    org.flywaydb: INFO
    org.testcontainers: INFO
```

---

## Claude-verify prompt

```
Aşağıdaki Flyway migration setup'umu banking-grade kriterlere göre değerlendir. 
Sadece sorunları belirt, kod yazma:

1. Dosya naming:
   - Migration dosyaları `V{version}__{description}.sql` formatında mı?
   - Versiyon numaraları çakışmıyor mu?
   - Description anlamlı, snake_case veya boşluk-ile-olabilir mi?
   - Repeatable migration'lar `R__` prefix'i kullanıyor mu?

2. SQL kalitesi:
   - Tablo/kolon naming snake_case mi?
   - Plural tablo, singular kolon mu?
   - Primary key her tabloda var mı?
   - Foreign key constraint'ler var mı?
   - CHECK constraint'ler enum-benzeri kolonlarda var mı (status, direction)?
   - TIMESTAMP WITH TIME ZONE kullanılmış mı (bare TIMESTAMP DEĞİL)?
   - Para kolonu NUMERIC(19, 4) veya benzer kesin tip mi (FLOAT/DOUBLE PRECISION DEĞİL)?
   - Index'ler eklenmiş mi (özellikle foreign key kolonları için)?

3. Banking-specific:
   - `accounts` tablosu version kolonu var mı (optimistic locking için)?
   - `journal_entries` tablosunda `transaction_id` UNIQUE constraint'i var mı (idempotency için)?
   - `journal_lines.amount` CHECK > 0 mı (negatif amount = direction'la ifade)?
   - `journal_lines.direction` CHECK ('DEBIT', 'CREDIT') var mı?

4. Configuration:
   - `spring.jpa.hibernate.ddl-auto` `validate` veya `none` mu (create/update DEĞİL)?
   - Flyway `validate-on-migrate: true` mu?
   - Production'da `baseline-on-migrate: false` mu?

5. Test:
   - TestContainers ile PostgreSQL test yapıyor musun (H2 DEĞİL)?
   - `@ServiceConnection` veya `@DynamicPropertySource` ile bağlanmış mı?
   - Tablo varlığı, kolon tipi, constraint'ler test edilmiş mi?
   - Constraint violation testleri var mı (status invalid, amount negative, foreign key)?
   - flyway_schema_history kontrolü var mı?

6. Anti-pattern:
   - Migration dosyası uygulandıktan sonra edit edilmiş mi? (Olmamalı, V3 yapılmalı)
   - Schema migration'da büyük data UPDATE var mı? (Olmamalı)
   - Production data'yı modify eden migration var mı (mass UPDATE)? (Olmamalı)
   - Migration rollback için `V*` yerine Flyway Pro'nun U* özelliği denenmiş mi (open source'ta yok)?

Her madde için PASS / FAIL / EKSIK işaretle. Kod yazma. Sadece neyi düzeltmem 
gerektiğini söyle.
```

---

## Tamamlama kriterleri

- [ ] Docker ile PostgreSQL local çalışıyor
- [ ] V1: `accounts` table migration yazılı, uygulanmış
- [ ] V2: `journal_entries` ve `journal_lines` tables migration yazılı
- [ ] V3: bir ALTER migration eklenmiş
- [ ] R: balance summary view repeatable migration
- [ ] V4: forward-fix migration (V3'ü geri alan) yazılmış, kavram anlaşılmış
- [ ] Migration'ı edit edip checksum hatası alındı (bilinçli deneyim)
- [ ] TestContainers ile migration integration test yazılı
- [ ] Tüm test'ler `mvn test` ile geçiyor
- [ ] `flyway_schema_history` tablosunu inceleyip anlamış
- [ ] "Migration rollback nasıl yapılır?" sorusuna "forward-fix" cevabı verebilirim
- [ ] H2 yerine TestContainers PostgreSQL kullanma sebebini biliyorum

---

## Defter notları

1. "Flyway'in çalışma prensibi: ____."
2. "Migration dosyası adlandırma kuralı: ____."
3. "Versioned (V) ve Repeatable (R) migration farkı: ____."
4. "Migration uygulandıktan sonra edit edersem ne olur: ____."
5. "Banking'de schema rollback nasıl yapılır (forward-fix nedir): ____."
6. "Expand-contract pattern (zero-downtime migration) 3 adımı: ____."
7. "`spring.jpa.hibernate.ddl-auto: validate` neden gerekli, neden create değil: ____."
8. "TestContainers H2'den daha iyi çünkü ____."
9. "TIMESTAMP WITH TIME ZONE ve bare TIMESTAMP farkı: ____."
10. "`flyway_schema_history` tablosunu manuel düzenlemek nasıl olur, neden tehlikelidir: ____."

```admonish success title="Bölüm Özeti"
- Flyway, migration'ları `V{version}__{description}.sql` dosyaları olarak sırayla uygular; uygulanma durumu `flyway_schema_history` tablosunda tutulur
- Uygulanmış bir migration asla edit edilmez — checksum mismatch hatası alırsın; düzeltme her zaman yeni bir migration ile yapılır
- Rollback = forward-fix: geri alma yerine ileri düzeltme migration'ı yazılır, audit trail temiz kalır
- Zero-downtime için expand-contract pattern: önce ekle (expand), sonra kodu geçir (migrate), en son eskiyi sil (contract) — 3 ayrı release
- Schema'yı Flyway yönetir; Hibernate `ddl-auto: validate` ile sadece doğrular — `create`/`update` banking'de yasak
- Migration testleri H2 ile değil, TestContainers + gerçek PostgreSQL ile yazılır
```
