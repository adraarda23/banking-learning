# Topic 11.2 — Docker Compose: Banking Local Stack

## Hedef

Docker Compose ile production-mirror local development environment kurmak. Banking microservices + dependencies (Postgres, Kafka, Keycloak, Vault, Redis, observability stack). Service dependency, healthcheck, network isolation, secrets, volumes, profile-based startup, override files, dev → CI → staging parity.

## Süre

Okuma: 1.5 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 11.1 (Docker for Java) bitti
- Phase 7-9 (microservice, security, observability) bitti

---

## Kavramlar

### 1. Compose v2 — modern

Docker Compose v2 (Go-rewritten, integrated `docker compose` command):

```bash
docker compose up -d
docker compose down
docker compose logs -f service-name
docker compose ps
```

`docker-compose.yml` versioning artık deprecated (top-level `version:` field).

### 2. Banking full stack — compose

```yaml
# docker-compose.yml
name: banking-stack

networks:
  banking-frontend:
    driver: bridge
  banking-backend:
    driver: bridge
    internal: true       # No outbound (security)
  banking-data:
    driver: bridge
    internal: true

volumes:
  postgres-data:
  kafka-data:
  zk-data:
  keycloak-data:
  vault-data:
  redis-data:
  prometheus-data:
  grafana-data:
  loki-data:

x-banking-service: &banking-service-defaults
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "100m"
      max-file: "5"
  deploy:
    resources:
      limits:
        memory: 1G
        cpus: '1.0'
      reservations:
        memory: 512M

services:
  # ─── Data layer ─────────────────────────────────
  
  postgres:
    image: postgres:16-alpine
    container_name: banking-postgres
    environment:
      POSTGRES_DB: banking
      POSTGRES_USER: ${DB_USER:-banking}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    networks:
      - banking-data
    ports:
      - "5432:5432"   # Dev only — production internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 2G
  
  redis:
    image: redis:7-alpine
    container_name: banking-redis
    command:
      - redis-server
      - --requirepass
      - ${REDIS_PASSWORD:-redispass}
      - --maxmemory
      - 512mb
      - --maxmemory-policy
      - allkeys-lru
      - --appendonly
      - "yes"
    volumes:
      - redis-data:/data
    networks:
      - banking-data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.1
    container_name: banking-zk
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zk-data:/var/lib/zookeeper
    networks:
      - banking-data
    healthcheck:
      test: ["CMD-SHELL", "echo ruok | nc localhost 2181 | grep imok"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  kafka:
    image: confluentinc/cp-kafka:7.5.1
    container_name: banking-kafka
    depends_on:
      zookeeper:
        condition: service_healthy
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,EXTERNAL://0.0.0.0:9094
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"   # Banking — explicit topics
      KAFKA_DELETE_TOPIC_ENABLE: "true"
      KAFKA_LOG_RETENTION_HOURS: 168              # 7 days
      KAFKA_NUM_PARTITIONS: 3
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - banking-backend
      - banking-data
    ports:
      - "9094:9094"     # External (dev)
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s
  
  # ─── Security ──────────────────────────────────
  
  keycloak-db:
    image: postgres:16-alpine
    container_name: banking-keycloak-db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD_FILE: /run/secrets/keycloak_db_password
    secrets:
      - keycloak_db_password
    volumes:
      - keycloak-data:/var/lib/postgresql/data
    networks:
      - banking-backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    container_name: banking-keycloak
    depends_on:
      keycloak-db:
        condition: service_healthy
    command: start --optimized
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD_FILE: /run/secrets/keycloak_db_password
      KC_HOSTNAME: auth.mavibank.local
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
      KC_PROXY: edge
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD_FILE: /run/secrets/keycloak_admin_password
    secrets:
      - keycloak_db_password
      - keycloak_admin_password
    volumes:
      - ./keycloak-realm.json:/opt/keycloak/data/import/banking-realm.json:ro
    networks:
      - banking-frontend
      - banking-backend
    ports:
      - "8180:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
  
  vault:
    image: hashicorp/vault:1.15
    container_name: banking-vault
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID_FILE: /run/secrets/vault_token
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    secrets:
      - vault_token
    volumes:
      - vault-data:/vault/file
    networks:
      - banking-backend
    ports:
      - "8200:8200"
    healthcheck:
      test: ["CMD", "vault", "status"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # ─── Observability ──────────────────────────────
  
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: banking-prometheus
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./observability/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    networks:
      - banking-backend
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --enable-feature=exemplar-storage
      - --web.enable-lifecycle
  
  grafana:
    image: grafana/grafana:10.4.0
    container_name: banking-grafana
    depends_on:
      prometheus:
        condition: service_started
    environment:
      GF_SECURITY_ADMIN_PASSWORD_FILE: /run/secrets/grafana_password
      GF_FEATURE_TOGGLES_ENABLE: traceqlEditor,traceToMetrics
    secrets:
      - grafana_password
    volumes:
      - ./observability/grafana/datasources:/etc/grafana/provisioning/datasources:ro
      - ./observability/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - grafana-data:/var/lib/grafana
    networks:
      - banking-frontend
      - banking-backend
    ports:
      - "3000:3000"
  
  jaeger:
    image: jaegertracing/all-in-one:1.55
    container_name: banking-jaeger
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    networks:
      - banking-backend
      - banking-frontend
    ports:
      - "16686:16686"
      - "14268:14268"
  
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.97.0
    container_name: banking-otel
    depends_on:
      jaeger:
        condition: service_started
    volumes:
      - ./observability/otel-collector.yaml:/etc/otelcol-contrib/config.yaml:ro
    networks:
      - banking-backend
    ports:
      - "4317:4317"
      - "4318:4318"
  
  loki:
    image: grafana/loki:2.9.0
    container_name: banking-loki
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./observability/loki-config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/loki
    networks:
      - banking-backend
    ports:
      - "3100:3100"
  
  # ─── Banking services ────────────────────────────
  
  account-service:
    <<: *banking-service-defaults
    image: banking/account-service:${VERSION:-latest}
    container_name: banking-account
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      keycloak:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: banking
      DB_USER_FILE: /run/secrets/db_password
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      KEYCLOAK_ISSUER: http://keycloak:8080/realms/banking
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_SERVICE_NAME: account-service
      JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75 -Duser.timezone=Europe/Istanbul"
    secrets:
      - db_password
    networks:
      - banking-frontend
      - banking-backend
      - banking-data
    ports:
      - "8081:8080"
      - "9091:8081"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/actuator/health/liveness"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
  
  transfer-service:
    <<: *banking-service-defaults
    image: banking/transfer-service:${VERSION:-latest}
    container_name: banking-transfer
    depends_on:
      account-service:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      ACCOUNT_SERVICE_URL: http://account-service:8080
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      REDIS_HOST: redis
      KEYCLOAK_ISSUER: http://keycloak:8080/realms/banking
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_SERVICE_NAME: transfer-service
    secrets:
      - db_password
    networks:
      - banking-frontend
      - banking-backend
      - banking-data
    ports:
      - "8082:8080"
      - "9092:8081"
  
  # ─── Gateway ────────────────────────────────────
  
  gateway:
    <<: *banking-service-defaults
    image: banking/gateway:${VERSION:-latest}
    container_name: banking-gateway
    depends_on:
      account-service:
        condition: service_healthy
      transfer-service:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      KEYCLOAK_ISSUER: http://keycloak:8080/realms/banking
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
    networks:
      - banking-frontend
      - banking-backend
    ports:
      - "8080:8080"

secrets:
  db_password:
    file: ./secrets/db_password.txt
  keycloak_db_password:
    file: ./secrets/keycloak_db_password.txt
  keycloak_admin_password:
    file: ./secrets/keycloak_admin_password.txt
  vault_token:
    file: ./secrets/vault_token.txt
  grafana_password:
    file: ./secrets/grafana_password.txt
```

### 3. Profile-based startup

Compose `profiles`:

```yaml
services:
  account-service:
    profiles: ["banking", "all"]
  transfer-service:
    profiles: ["banking", "all"]
  prometheus:
    profiles: ["observability", "all"]
  grafana:
    profiles: ["observability", "all"]
  jaeger:
    profiles: ["observability", "all"]
```

```bash
# Banking only
docker compose --profile banking up -d

# Observability only
docker compose --profile observability up -d

# Everything
docker compose --profile all up -d
```

Banking dev flow: bazıları local IDE'de çalıştır + dependency'ler compose'da.

### 4. Override files

```
docker-compose.yml          # Base (production-like)
docker-compose.dev.yml      # Dev overrides (debug, hot reload)
docker-compose.ci.yml       # CI variant
```

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

```yaml
# docker-compose.dev.yml
services:
  account-service:
    environment:
      LOGGING_LEVEL_COM_BANK: DEBUG
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
    volumes:
      - ./account-service/target:/app:rw   # Hot reload
    ports:
      - "5005:5005"                          # Remote debugger
    command:
      - sh
      - -c
      - |
        java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 \
             $JAVA_OPTS -jar /app/spring-boot-loader/...
```

### 5. Healthcheck dependency

`depends_on` with condition:

```yaml
depends_on:
  postgres:
    condition: service_healthy
  kafka:
    condition: service_healthy
```

Compose waits until dependency healthy before starting dependent.

**Banking critical:** Account service'in Postgres olmadan başlaması yanlış. `service_started` (default) yetmez, `service_healthy` gerekir.

### 6. Secrets management

```bash
# secrets dir
mkdir -p secrets
echo "MyDbPass123!" > secrets/db_password.txt
chmod 600 secrets/db_password.txt
```

Compose mounts `/run/secrets/db_password`. App reads file (Spring Boot `SPRING_CONFIG_IMPORT=file:/run/secrets/db_password`).

**Production:** Vault / K8s Secret. Compose secret pattern dev'de mirror.

### 7. Network isolation

```yaml
networks:
  banking-frontend:    # Gateway + load balancer accessible
  banking-backend:     # Service-to-service
    internal: true     # No outbound
  banking-data:        # DB, Redis, Kafka
    internal: true
```

`internal: true` = no internet egress. Data tier breach contains.

Banking data tier: DB + Redis + Kafka isolated network.

### 8. Resource limits

```yaml
deploy:
  resources:
    limits:
      memory: 2G
      cpus: '2.0'
    reservations:
      memory: 1G
      cpus: '0.5'
```

K8s requests/limits parallel. Local'da test edebilirsin.

### 9. Init scripts + seed data

```yaml
postgres:
  volumes:
    - ./init-db.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    - ./seed-data.sql:/docker-entrypoint-initdb.d/02-seed.sql:ro
```

Postgres entrypoint runs `*.sql` files in `/docker-entrypoint-initdb.d/`.

Banking: seed COA, demo customers, sample transactions.

### 10. Banking — compose anti-pattern'leri

**Anti-pattern 1: Hardcoded secrets in compose**

```yaml
environment:
  DB_PASSWORD: secret123   # ❌
```

File-based secrets.

**Anti-pattern 2: All services on single network**

Data tier needs isolation. Multiple networks.

**Anti-pattern 3: depends_on without healthcheck**

Service starts before dependency ready → connection errors. `service_healthy`.

**Anti-pattern 4: latest tag**

Reproducibility yok. Specific version.

**Anti-pattern 5: No resource limits**

Single service OOM kills host. Limits per service.

**Anti-pattern 6: Compose for production**

Compose dev/CI ideal. Production = K8s (Topic 11.3).

**Anti-pattern 7: Volume mount source code in production-like compose**

Hot reload dev only. Production-mirror compose no source mount.

**Anti-pattern 8: No init script idempotency**

Re-run = duplicate data. SQL `INSERT ... ON CONFLICT DO NOTHING`.

**Anti-pattern 9: Mixed profile + default services**

Profile-less service always start. Banking için: dev local IDE → only dependency profile.

**Anti-pattern 10: No logging driver config**

Disk fills with logs. `max-size: "100m" max-file: "5"`.

---

## Önemli olabilecek araştırma kaynakları

- Docker Compose v2 reference
- Compose spec
- "Compose-spec" YAML reference

---

## Mini task'ler

### Task 11.2.1 — Base compose stack (60 dk)

Postgres + Kafka + Keycloak + Redis. Healthchecks. Network separation.

### Task 11.2.2 — Add observability (45 dk)

Prometheus + Grafana + Jaeger + OTel + Loki.

### Task 11.2.3 — Banking services (60 dk)

3+ service from Phase 10. JIB-built image. Depends on dependencies.

### Task 11.2.4 — Profile-based startup (30 dk)

`banking`, `observability`, `data`, `all` profiles. Test partial up.

### Task 11.2.5 — Override file dev (30 dk)

`docker-compose.dev.yml` with debug + remote debugger + hot reload.

### Task 11.2.6 — File-based secrets (30 dk)

`secrets/` dir + chmod 600. App reads.

### Task 11.2.7 — Network isolation (30 dk)

3 networks (frontend, backend, data). `internal: true` data tier.

### Task 11.2.8 — Init script seed data (45 dk)

COA seed + 100 demo customers + sample transactions.

### Task 11.2.9 — Resource limits per service (15 dk)

Memory + CPU. Test under load.

### Task 11.2.10 — End-to-end smoke test (45 dk)

`compose up -d` + wait healthy + smoke test (login, transfer) + `compose down`.

---

## Test yazma rehberi

```bash
# Smoke test script
#!/bin/bash
set -euo pipefail

echo "Starting stack..."
docker compose up -d

echo "Waiting for healthy..."
for service in postgres kafka keycloak account-service transfer-service gateway; do
  echo "  Waiting for $service..."
  timeout 180 sh -c "until [ \"\$(docker inspect --format='{{.State.Health.Status}}' banking-$service)\" = 'healthy' ]; do sleep 5; done"
done

echo "Running smoke tests..."
# Login
TOKEN=$(curl -sf -X POST http://localhost:8180/realms/banking/protocol/openid-connect/token \
  -d "client_id=banking-web" \
  -d "username=ahmet" \
  -d "password=test" \
  -d "grant_type=password" | jq -r '.access_token')

# Get accounts
curl -fsS -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/v1/accounts/me > /dev/null

# Initiate transfer
curl -fsS -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: smoke-$(date +%s)" \
  -d '{"toIban":"TR320010009999987654321098","amount":"100","currency":"TRY"}' \
  http://localhost:8080/v1/transfers/havale > /dev/null

echo "All smoke tests passed."

echo "Cleanup..."
docker compose down -v
```

JUnit Testcontainers compose:

```java
@Testcontainers
class BankingStackIntegrationTest {
    
    @Container
    static DockerComposeContainer<?> stack = new DockerComposeContainer<>(
        new File("docker-compose.yml"),
        new File("docker-compose.test.yml"))
        .withExposedService("gateway", 8080, Wait.forHttp("/actuator/health").forStatusCode(200))
        .withExposedService("keycloak", 8080)
        .withLocalCompose(true);
    
    @Test
    void transferEndpointShouldBeAccessible() {
        Integer port = stack.getServicePort("gateway", 8080);
        String url = "http://localhost:" + port + "/actuator/health";
        // HTTP GET → 200
    }
}
```

---

## Claude-verify prompt

```
Compose setup'ımı banking-grade kriterlere göre değerlendir:

1. Stack content:
   - Postgres + Kafka + Keycloak + Redis + Vault?
   - Prometheus + Grafana + Jaeger + Loki + OTel?
   - 3+ banking microservice?
   - Gateway?

2. Health checks:
   - Every service healthcheck?
   - depends_on with service_healthy?
   - start_period banking için 60s+?

3. Network:
   - frontend / backend / data ayrı?
   - data internal:true?
   - Banking PCI scope segmented?

4. Secrets:
   - File-based secrets/?
   - chmod 600?
   - No env var plain password?

5. Volumes:
   - Persistent data volume?
   - Init script seed data?
   - GC log + heap dump mount?

6. Profiles:
   - banking / observability / data?
   - Partial startup possible?

7. Override files:
   - dev.yml debug + hot reload?
   - test.yml CI variant?

8. Resources:
   - Memory + CPU limits per service?
   - K8s-realistic budget?

9. Logging:
   - max-size + max-file?
   - JSON driver?

10. Anti-pattern:
    - Hardcoded secrets YOK?
    - Single network YOK?
    - depends_on plain (no condition) YOK?
    - latest tag YOK?
    - No resource limit YOK?
    - Production compose YOK (K8s should be)?
    - Source code mount production-mirror YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Full stack compose (data + security + observability + banking)
- [ ] Healthcheck + depends_on healthy
- [ ] 3 network (frontend/backend/data)
- [ ] File-based secrets
- [ ] Init script seed
- [ ] Profile-based startup (3+ profile)
- [ ] dev.yml override
- [ ] Resource limits
- [ ] End-to-end smoke test script
- [ ] Banking realistic environment local

---

## Defter notları (10 madde)

1. "Compose v2 + docker compose command modern usage: ____."
2. "Healthcheck + depends_on service_healthy + start_period banking: ____."
3. "Multiple network (frontend/backend/data) + internal:true data tier: ____."
4. "File-based secrets + chmod + Spring Boot config-import file: ____."
5. "Profile-based partial startup banking dev workflow: ____."
6. "Override file (base + dev + CI) layered config: ____."
7. "Init script seed banking COA + demo customer: ____."
8. "Resource limits memory + CPU K8s-realistic budget: ____."
9. "Compose dev/CI ideal + production K8s ayrımı: ____."
10. "Smoke test end-to-end (compose up → login → transfer → down): ____."
