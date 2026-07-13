# Topic 11.1 — Docker for Java

## Hedef

Production-grade Java/Spring Boot Docker image build. Multi-stage build, JIB plugin, layer optimization, JRE-only runtime, base image choice (Eclipse Temurin, Distroless, Alpine), non-root user, healthcheck, BellSoft Liberica NIK, security scan, banking image registry strategy.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Linux temel
- Docker konsept (image vs container)
- Spring Boot fat jar yapısı

---

## Kavramlar

### 1. Docker image — anatomy

```
Image = read-only layered filesystem snapshot
Container = running instance + writable layer
```

Java image layers:
```
1. Base OS (Ubuntu, Alpine)
2. JRE/JDK
3. App dependencies (Spring Boot libs)
4. App code (own classes)
5. Config + entrypoint
```

**Layer optimization principle:**
- Frequently changing → top
- Stable → bottom
- Docker cache reuses unchanged layers → fast rebuild

### 2. Naive Dockerfile (anti-pattern)

```dockerfile
FROM openjdk:21-jdk
COPY target/banking-service-1.0.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

**Problems:**
- 700+ MB image (JDK, full Ubuntu)
- Single layer for app (full jar rebuilt on any change)
- Root user (security)
- No healthcheck
- No JVM flags banking
- JDK instead of JRE (build tools shipped)

### 3. Multi-stage build — proper

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /build

# Cache Maven dependencies
COPY pom.xml ./
COPY .mvn/ .mvn/
COPY mvnw ./
RUN ./mvnw dependency:go-offline -B

# Copy + build
COPY src ./src
RUN ./mvnw package -DskipTests -B

# Extract Spring Boot layers
RUN java -Djarmode=layertools -jar target/banking-service-*.jar extract --destination extracted/

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-jammy
LABEL maintainer="banking-platform@mavibank.com"
LABEL version="1.0.0"

# Non-root user
RUN groupadd --gid 1000 banking && \
    useradd --uid 1000 --gid banking --shell /bin/bash --create-home banking
USER banking:banking

WORKDIR /app

# Copy in layer order (least → most volatile)
COPY --from=builder --chown=banking:banking /build/extracted/dependencies/ ./
COPY --from=builder --chown=banking:banking /build/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=banking:banking /build/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=banking:banking /build/extracted/application/ ./

# JVM banking config
ENV JAVA_OPTS="-Xms512m -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
    -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
    -Xlog:gc*:file=/var/log/banking/gc.log:time,uptime,level,tags:filecount=10,filesize=100M \
    -XX:+UseStringDeduplication -XX:+AlwaysPreTouch \
    -Djava.security.egd=file:/dev/./urandom \
    -Duser.timezone=Europe/Istanbul"

EXPOSE 8080 8081

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -fsS http://localhost:8081/actuator/health/liveness || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

**Size:** ~250 MB (was 700+ MB).

### 4. Spring Boot layered jar

Spring Boot 2.3+ supports layered extraction:

```
extracted/
├── dependencies/             (rarely change — libraries)
├── spring-boot-loader/       (rarely change — Spring Boot internal)
├── snapshot-dependencies/    (occasionally — SNAPSHOT versions)
└── application/              (frequently change — your code)
```

Docker cache:
- App code change → only `application/` layer rebuilds (~5 MB)
- Dep change → `dependencies/` rebuild
- Image push: only changed layers

**Build performance:**

Without layers: 250 MB push per commit (~30 sec)
With layers: ~5 MB push per commit (~3 sec)

### 5. JIB plugin — Dockerfile-less

Google JIB plugin builds image without local Docker daemon:

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.0</version>
    <configuration>
        <from>
            <image>eclipse-temurin:21-jre-jammy</image>
        </from>
        <to>
            <image>registry.mavibank.com/banking/${project.artifactId}:${project.version}</image>
            <auth>
                <username>${env.REGISTRY_USER}</username>
                <password>${env.REGISTRY_PASSWORD}</password>
            </auth>
        </to>
        <container>
            <jvmFlags>
                <jvmFlag>-Xms512m</jvmFlag>
                <jvmFlag>-Xmx2g</jvmFlag>
                <jvmFlag>-XX:+UseG1GC</jvmFlag>
                <jvmFlag>-XX:+HeapDumpOnOutOfMemoryError</jvmFlag>
                <jvmFlag>-Djava.security.egd=file:/dev/./urandom</jvmFlag>
                <jvmFlag>-Duser.timezone=Europe/Istanbul</jvmFlag>
            </jvmFlags>
            <ports>
                <port>8080</port>
                <port>8081</port>
            </ports>
            <labels>
                <maintainer>banking-platform@mavibank.com</maintainer>
                <version>${project.version}</version>
            </labels>
            <user>1000:1000</user>
            <environment>
                <TZ>Europe/Istanbul</TZ>
            </environment>
        </container>
        <containerizingMode>packaged</containerizingMode>
    </configuration>
</plugin>
```

Build:
```bash
mvn compile jib:build
# OR build to local docker daemon (without registry)
mvn compile jib:dockerBuild
# OR build to tar
mvn compile jib:buildTar
```

**Advantages:**
- No Dockerfile maintain
- Reproducible builds (same input → same image hash)
- Fast (parallel layer push)
- CI-friendly (no Docker daemon needed)

**Disadvantages:**
- Less flexible (some Dockerfile patterns missing)
- Custom system packages (apt-get) harder

Banking için **JIB ideal** for most service images.

### 6. Base image choices

| Base | Size | Banking |
|---|---|---|
| `eclipse-temurin:21-jre-jammy` (Ubuntu) | ~250 MB | ✓ Standard, glibc, debug tools |
| `eclipse-temurin:21-jre-alpine` | ~80 MB | Small but musl libc (issues with some JNI) |
| `gcr.io/distroless/java21-debian12` | ~150 MB | No shell — security maximum |
| `bellsoft/liberica-runtime-container:jre-21` | ~150 MB | Tuned by BellSoft |
| `azul/zulu-openjdk:21-jre` | ~200 MB | Azul-backed |
| `amazoncorretto:21-alpine-jre` | ~90 MB | AWS, Alpine |

**Banking pratiği:**
- **Eclipse Temurin** — open source, well-tested, secure update cadence
- **Distroless** — high security, no debug shell (banking critical services)
- **BellSoft Liberica** — better startup time (NIK)

**Alpine warnings:**
- musl libc != glibc
- Bouncy Castle FIPS issue
- Some banking native libs incompatible
- Faster startup, smaller

### 7. Distroless — security-first

```dockerfile
FROM eclipse-temurin:21-jdk-jammy AS builder
# ... build steps
RUN java -Djarmode=layertools -jar target/banking-service-*.jar extract

FROM gcr.io/distroless/java21-debian12
COPY --from=builder /build/dependencies/ /app/
COPY --from=builder /build/spring-boot-loader/ /app/
COPY --from=builder /build/snapshot-dependencies/ /app/
COPY --from=builder /build/application/ /app/

USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["java", "-XX:+UseG1GC", "-jar", "/app/spring-boot-loader/org/springframework/boot/loader/launch/JarLauncher.class"]
```

Distroless'in özellikleri:
- No shell (no bash, sh, ash)
- No package manager
- No coreutils (no `ls`, `cat`, etc)
- Sadece Java + glibc + tzdata

**Attack surface minimum.** Banking critical services için tercih.

**Trade-off:** Debug zor (kubectl exec → shell yok). Solution: `kubectl debug` ephemeral container.

### 8. Non-root + security hardening

```dockerfile
# Non-root user (UID > 1000 banking standard)
RUN groupadd --gid 10001 banking && \
    useradd --uid 10001 --gid 10001 --shell /sbin/nologin --no-create-home banking

USER 10001:10001

# Read-only root filesystem (K8s securityContext de set edebilir)
# Writable dirs mount as volume
VOLUME /tmp /var/log/banking /dumps

# Drop capabilities
# (K8s securityContext.capabilities.drop: ["ALL"])
```

**Securit checklist:**
- USER non-root (UID >= 1000)
- No SETUID binaries
- No SSH server
- Minimum packages
- Health endpoint separate port (management)
- No secrets in image
- Image scan in CI

### 9. .dockerignore

```
# .dockerignore
.git
.gitignore
target/
!target/banking-service-*.jar    # If building outside JIB
*.md
.idea/
.vscode/
*.log
node_modules/
.env*
docker-compose*.yml
Dockerfile*
.dockerignore
.github/
docs/
```

Build context minimum → faster build + smaller cache.

### 10. Banking JVM flags — production

```bash
JAVA_OPTS="
  -Xms2g -Xmx2g                                       # Same heap (avoid resize pause)
  -XX:+UseG1GC                                         # Modern collector
  -XX:MaxGCPauseMillis=200                             # Target pause
  -XX:InitiatingHeapOccupancyPercent=45                # Concurrent start
  -XX:+ParallelRefProcEnabled
  -XX:+AlwaysPreTouch                                  # Touch heap on startup
  -XX:+UseStringDeduplication                          # Reduce String memory
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/dumps/heap-%p-%t.hprof
  -Xlog:gc*:file=/var/log/banking/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
  -XX:NativeMemoryTracking=summary
  -XX:+StartFlightRecording=duration=0s,filename=/var/log/banking/flight.jfr,settings=default,maxage=1h
  -XX:+UnlockDiagnosticVMOptions
  -XX:+DebugNonSafepoints
  -Dfile.encoding=UTF-8
  -Duser.timezone=Europe/Istanbul
  -Djava.security.egd=file:/dev/./urandom
  -Dspring.profiles.active=production
  -Dotel.service.name=transfer-service
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317
"
```

For **ultra-low-latency banking** (FAST):
```bash
-XX:+UseZGC -XX:+ZGenerational
```

### 11. Healthcheck — banking

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -fsS http://localhost:8081/actuator/health/liveness || exit 1
```

Spring Boot Actuator separates:
- `/actuator/health/liveness` — process alive?
- `/actuator/health/readiness` — ready to accept traffic?

**K8s:**
- livenessProbe → liveness endpoint
- readinessProbe → readiness endpoint
- startupProbe → liveness (with longer initial delay)

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: management
  initialDelaySeconds: 60
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: management
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: management
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 30   # 5 min startup window banking için
```

### 12. Image scan — security

**Trivy** — open source, popular:

```bash
trivy image registry.mavibank.com/banking/transfer-service:1.0.0
```

Output:
```
banking/transfer-service:1.0.0 (debian 12.5)
  Total: 23 (UNKNOWN: 0, LOW: 15, MEDIUM: 5, HIGH: 2, CRITICAL: 1)

CVE-2024-XXXX  CRITICAL  openssl 3.0.11 → 3.0.13
CVE-2024-YYYY  HIGH      curl 8.5.0 → 8.6.0
...
```

CI integration:
```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: registry.mavibank.com/banking/transfer-service:${{ github.sha }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'   # Fail build on CRITICAL/HIGH
```

**Grype** alternative:
```bash
grype banking/transfer-service:1.0.0
```

**Snyk** commercial alternative — well-maintained DB.

### 13. SBOM — Software Bill of Materials

```bash
syft banking/transfer-service:1.0.0 -o spdx-json=sbom.json
# or
trivy sbom banking/transfer-service:1.0.0 --output sbom.json --format spdx-json
```

Banking için **regulatory** (supply chain transparency). PCI-DSS v4 requirement.

### 14. Image signing — Cosign

```bash
# Generate keypair
cosign generate-key-pair

# Sign image (CI on successful build)
cosign sign --key cosign.key registry.mavibank.com/banking/transfer-service:1.0.0

# Verify (K8s admission control)
cosign verify --key cosign.pub registry.mavibank.com/banking/transfer-service:1.0.0
```

K8s admission controller (Sigstore policy controller) → only signed images allowed.

Banking supply chain integrity için kritik.

### 15. Banking — Docker anti-pattern'leri

**Anti-pattern 1: Root user**
```dockerfile
USER root   # ❌
```
Banking için **non-root mandatory**.

**Anti-pattern 2: latest tag**
```dockerfile
FROM openjdk:latest   # ❌
```
Reproducibility yok. Specific version (`21-jre-jammy`).

**Anti-pattern 3: JDK in runtime**
```dockerfile
FROM eclipse-temurin:21-jdk   # ❌ runtime için
```
JRE yeterli. JDK build tools shipped → attack surface.

**Anti-pattern 4: Secrets in image**
```dockerfile
ENV DB_PASSWORD=secret   # ❌
```
Image inspect = leak. K8s Secret veya Vault.

**Anti-pattern 5: Single layer fat jar**
```dockerfile
COPY target/app.jar /app.jar   # ❌
```
Code change → full jar rebuild. Layered jar.

**Anti-pattern 6: No healthcheck**
Docker swarm / K8s auto-restart için healthcheck şart.

**Anti-pattern 7: Build tools in runtime**
```dockerfile
RUN apt-get install -y maven git curl   # ❌
```
Multi-stage build kullan.

**Anti-pattern 8: Verbose logs by default in image**
```dockerfile
ENV LOG_LEVEL=DEBUG   # ❌
```
Production INFO. Env override at deploy.

**Anti-pattern 9: No image scan**
CVE'ler birikir. Trivy scan CI'da mandatory.

**Anti-pattern 10: Hardcoded JVM heap**
```dockerfile
ENV JAVA_OPTS="-Xmx2g"
```
K8s memory limits ile inconsistent. Use `MaxRAMPercentage`:
```bash
-XX:MaxRAMPercentage=75.0
```

---

## Önemli olabilecek araştırma kaynakları

- Spring Boot Docker reference
- Google JIB documentation
- "Docker for Java Developers" — Arun Gupta
- Eclipse Temurin images
- Distroless project (Google)
- Trivy / Grype / Snyk docs
- Cosign + Sigstore
- "The Twelve-Factor App"

---

## Mini task'ler

### Task 11.1.1 — Multi-stage Dockerfile (45 dk)

Layered jar extract. Builder stage + runtime stage. Test: image build, run, healthcheck.

### Task 11.1.2 — JIB Maven plugin (45 dk)

`mvn jib:dockerBuild`. Compare image size vs Dockerfile approach.

### Task 11.1.3 — Distroless variant (30 dk)

Same app distroless base. Compare size + attack surface.

### Task 11.1.4 — Non-root + UID 10001 (15 dk)

USER directive + chown layer. Run: confirm `id` shows 10001.

### Task 11.1.5 — Healthcheck Actuator (30 dk)

`/actuator/health/liveness` separated. `--start-period=60s`. Test: image healthy after 60s.

### Task 11.1.6 — JVM banking flags (30 dk)

JAVA_OPTS env. GC log mount. Heap dump path.

### Task 11.1.7 — Image scan Trivy (30 dk)

Local scan. Fix one HIGH CVE (base image update).

### Task 11.1.8 — SBOM generate (15 dk)

Syft veya Trivy SBOM JSON output.

### Task 11.1.9 — Cosign sign + verify (30 dk)

Keypair, sign, verify roundtrip.

### Task 11.1.10 — Memory ratio JVM (15 dk)

`-XX:MaxRAMPercentage=75.0`. Test K8s memory limit ile consistent.

---

## Test yazma rehberi

```bash
# build + scan
docker build -t banking/transfer-service:test .
trivy image --severity HIGH,CRITICAL --exit-code 1 banking/transfer-service:test

# size check
size=$(docker image inspect banking/transfer-service:test --format='{{.Size}}')
test $size -lt 300000000   # < 300 MB

# user check
docker run --rm banking/transfer-service:test id
# uid=10001(banking) gid=10001(banking)

# healthcheck
docker run -d --name btest banking/transfer-service:test
sleep 60
status=$(docker inspect --format='{{.State.Health.Status}}' btest)
test "$status" = "healthy"
docker rm -f btest

# layer count
docker history banking/transfer-service:test | wc -l
# Expected: ~10 layers (not single)
```

JUnit Testcontainers test:

```java
@Testcontainers
class DockerImageTest {
    
    @Container
    static GenericContainer<?> banking = new GenericContainer<>(
        DockerImageName.parse("banking/transfer-service:test"))
        .withExposedPorts(8080, 8081)
        .waitingFor(Wait.forHttp("/actuator/health/liveness")
            .forPort(8081)
            .withStartupTimeout(Duration.ofMinutes(2)));
    
    @Test
    void shouldRunAsNonRoot() throws Exception {
        Container.ExecResult result = banking.execInContainer("id", "-u");
        assertThat(result.getStdout().trim()).isEqualTo("10001");
    }
    
    @Test
    void shouldHaveCorrectTimezone() throws Exception {
        Container.ExecResult result = banking.execInContainer("cat", "/etc/timezone");
        assertThat(result.getStdout()).contains("Europe/Istanbul");
    }
    
    @Test
    void healthEndpointShouldBeUp() throws Exception {
        String url = "http://" + banking.getHost() + ":" + banking.getMappedPort(8081) 
            + "/actuator/health";
        // HTTP GET → expect "UP"
    }
}
```

---

## Claude-verify prompt

```
Docker image'imi banking-grade kriterlere göre değerlendir:

1. Build approach:
   - Multi-stage build veya JIB?
   - Layered jar extraction (Spring Boot)?
   - Reproducible (locked versions)?

2. Base image:
   - Specific version (no latest)?
   - JRE (not JDK) runtime?
   - Eclipse Temurin / Distroless / BellSoft chosen rationale?

3. Size:
   - < 300 MB target?
   - Build context minimum (.dockerignore)?

4. Security:
   - Non-root USER (UID >= 1000)?
   - No secrets in image?
   - Read-only root FS support?
   - Drop capabilities (K8s)?
   - Trivy scan in CI fail HIGH/CRITICAL?
   - SBOM generated?
   - Cosign signed?

5. JVM:
   - G1GC + MaxGCPauseMillis?
   - HeapDumpOnOOM + path?
   - GC log + rotation?
   - JFR continuous?
   - MaxRAMPercentage (K8s memory limit aware)?
   - Time zone Europe/Istanbul?
   - file.encoding UTF-8?

6. Healthcheck:
   - HEALTHCHECK directive (Dockerfile)?
   - K8s liveness + readiness + startup probes?
   - Separate management port (8081)?

7. Layer efficiency:
   - Dependencies layer (rare change)?
   - Application layer (frequent change)?
   - Cache reuse fast rebuild?

8. Observability:
   - Otel agent / env vars?
   - Log volume mount?
   - Heap dump volume mount?

9. CI integration:
   - Build trigger on PR?
   - Tag with git SHA + version?
   - Push to private registry?
   - Sign + scan in pipeline?

10. Anti-pattern:
    - Root USER YOK?
    - latest tag YOK?
    - JDK runtime YOK?
    - Secrets in image YOK?
    - Single fat jar layer YOK?
    - No healthcheck YOK?
    - No scan YOK?
    - Hardcoded heap YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Multi-stage Dockerfile + layered jar
- [ ] JIB Maven plugin variant
- [ ] Distroless variant
- [ ] Non-root USER 10001
- [ ] Healthcheck Actuator separate port
- [ ] Banking JVM flags
- [ ] Trivy scan + 0 CRITICAL
- [ ] SBOM (SPDX) generated
- [ ] Cosign sign + verify
- [ ] MaxRAMPercentage K8s-aware
- [ ] 5+ Testcontainers test

---

## Defter notları (10 madde)

1. "Multi-stage build builder + runtime separation + size reduction: ____."
2. "Spring Boot layered jar (dependencies/loader/snapshot/application) cache optimization: ____."
3. "JIB plugin Dockerfile-less + reproducible builds banking CI: ____."
4. "Base image (Eclipse Temurin / Distroless / Alpine) banking selection: ____."
5. "Non-root USER UID 10001 + read-only root + drop capabilities: ____."
6. "Banking JVM flags (G1GC + heap dump + JFR + Istanbul TZ + UTF-8): ____."
7. "Healthcheck liveness/readiness/startup K8s probe pattern: ____."
8. "Trivy scan + SBOM + Cosign sign banking supply chain integrity: ____."
9. "MaxRAMPercentage K8s memory limit consistency: ____."
10. "Anti-pattern (root user, latest tag, secrets in image, single layer): ____."
