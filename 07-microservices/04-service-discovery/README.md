# Topic 7.4 — Service Discovery

## Hedef

Microservice'lerin **birbirini dinamik olarak bulma** mekanizmalarını öğrenmek. Client-side discovery (Eureka), server-side (Load Balancer), Kubernetes Service (DNS-based), Consul (HashiCorp). Spring Cloud LoadBalancer ile entegrasyon, zone-aware load balancing, health check, instance evictioning. Banking deployment'larda doğru seçim.

## Süre

Okuma: 1.5 saat • Mini task: 1.5 saat • Test: 30 dk • Toplam: ~3.5 saat

## Önbilgi

- Topic 7.1-7.3 bitti (DDD, decomposition, gateway)
- Phase 11 K8s basics gelecek — burası ön hazırlık
- Container orchestration kavramlarına aşinasın (Docker)

---

## Kavramlar

### 1. Service discovery problemi

**Senaryo:** `transfer-service` `account-service`'i çağırmak istiyor.

Statik konfigürasyon (kötü):

```yaml
account-service:
  url: http://account-service-instance-1:8081
```

K8s'te pod'lar gelir gider, IP'ler değişir. Auto-scaling 1 → 5 replica. Hard-coded URL = kabus.

**Service discovery:** Service'ler **kendilerini register** eder, client'lar **dynamically lookup** yapar.

```
Service registry
  ↑ register (heartbeat)
[account-service-pod-1] (IP: 10.0.0.5)
[account-service-pod-2] (IP: 10.0.0.7)
[account-service-pod-3] (IP: 10.0.0.9)

[transfer-service] → registry.lookup("account-service") → 
  → load balance → 10.0.0.5 veya .7 veya .9
```

### 2. Yaklaşımlar

**Client-side discovery (Eureka):**
- Service'ler registry'e register
- Client registry'den instance list alır
- Client load balancing yapar

**Server-side discovery (AWS ELB, Nginx):**
- Service'ler health check endpoint expose eder
- Load Balancer registry rolü
- Client LB'ye konuşur

**DNS-based (Kubernetes):**
- K8s Service objesi
- DNS resolution → cluster-internal IP
- kube-proxy IP balancing

**Consul (HashiCorp):**
- Service discovery + KV store + service mesh
- Multi-data-center

### 3. Eureka — client-side discovery

Netflix OSS. Java/Spring native.

#### Eureka Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false   # server kendi'yi register etmiyor
    fetch-registry: false
  instance:
    hostname: eureka-server

spring:
  application:
    name: eureka-server
```

UI: `http://localhost:8761`

**Production:** **3+ node cluster** (peer-aware):

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-1:8761/eureka,http://eureka-2:8761/eureka,http://eureka-3:8761/eureka
```

Eureka peer'leri arası replicate. Bir node down → diğerleri devam.

#### Eureka Client (each service)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: account-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true   # K8s'te IP based
    instance-id: ${spring.application.name}:${random.value}
    lease-renewal-interval-in-seconds: 30   # heartbeat
    lease-expiration-duration-in-seconds: 90   # 3 missed = dead
    metadata-map:
      zone: ${ZONE:default}
      version: ${app.version:1.0.0}
      tenant: ${TENANT:default}
```

#### Service registration akışı

```
1. account-service boot → Eureka'ya register
   POST /eureka/apps/ACCOUNT-SERVICE
   {
     "instanceId": "account-service:abc123",
     "hostName": "account-service-pod-1",
     "ipAddr": "10.0.0.5",
     "port": 8081,
     "status": "UP",
     "metadata": {...}
   }

2. Her 30 sn → heartbeat
   PUT /eureka/apps/ACCOUNT-SERVICE/instance-id

3. 90 sn heartbeat yok → instance "DOWN", registry'den çıkar

4. Shutdown hook → deregister
   DELETE /eureka/apps/ACCOUNT-SERVICE/instance-id
```

#### Client lookup

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

@Autowired WebClient.Builder webClientBuilder;

public Mono<Account> getAccount(UUID id) {
    return webClientBuilder.baseUrl("http://account-service").build()
        .get().uri("/accounts/{id}", id)
        .retrieve()
        .bodyToMono(Account.class);
}
```

`@LoadBalanced` + `lb://` veya service name → Spring Cloud LoadBalancer Eureka registry'den instance list alıp dağıtır.

### 4. Kubernetes Service — DNS-based

K8s'te **service discovery built-in**. Eureka'ya gerek YOK (genelde).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: account-service
  namespace: banking
spec:
  selector:
    app: account-service
  ports:
    - port: 8081
      targetPort: 8081
  type: ClusterIP
```

K8s DNS otomatik:
- `account-service.banking.svc.cluster.local` (FQDN)
- `account-service.banking` (kısa)
- `account-service` (aynı namespace'te)

Client kod:

```java
webClient.baseUrl("http://account-service:8081")
    .get().uri("/accounts/123")
    .retrieve()...
```

Resolve cluster IP'ye (kube-proxy load balance to pods). **Eureka yok, library yok, sadece DNS.**

#### Service types

```yaml
type: ClusterIP    # internal only (default)
type: NodePort     # external via node IP + high port
type: LoadBalancer # cloud provider LB
type: ExternalName # DNS alias
```

Banking: Internal service-to-service `ClusterIP`. External gateway için `LoadBalancer`.

### 5. Spring Cloud LoadBalancer

`@LoadBalanced` ile WebClient/RestTemplate service name'le çağrı:

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder()
        .filter(new RetryFilter())
        .filter(new TracingFilter());
}

@Bean
public AccountServiceClient accountServiceClient(WebClient.Builder builder) {
    return new AccountServiceClient(builder.baseUrl("http://account-service").build());
}
```

Eureka **veya** K8s discovery client — Spring Boot autodetect.

#### Load balancing strategies

```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}
```

Built-in:
- `RoundRobinLoadBalancer` (default)
- `RandomLoadBalancer`
- Custom (weighted, sticky, vb.)

### 6. Zone-aware load balancing

Multi-AZ deployment'larda **aynı zone'daki** instance'ı tercih et — cross-AZ latency azalt.

```yaml
spring:
  cloud:
    loadbalancer:
      zone: ${ZONE:default}   # env'den
```

Service registration metadata:

```yaml
eureka:
  instance:
    metadata-map:
      zone: ${ZONE:default}
```

LoadBalancer: caller zone == instance zone → tercih. Yoksa fallback (cross-zone).

**Banking pratiği:** AWS multi-AZ veya GCP multi-region. Cross-AZ latency 1-2ms tasarrufu.

### 7. Health check entegrasyonu

Service discovery health check ile registry'i temizler.

#### Spring Boot Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true   # liveness, readiness ayrı
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

`/actuator/health` UP olmadığında instance registry'den çıkarılır.

#### Custom health indicator

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final JdbcTemplate jdbc;
    
    @Override
    public Health health() {
        try {
            jdbc.queryForObject("SELECT 1 FROM dual", Integer.class);
            return Health.up().withDetail("database", "reachable").build();
        } catch (Exception e) {
            return Health.down(e).withDetail("database", "unreachable").build();
        }
    }
}
```

Banking — custom health indicators:
- Database connectivity
- Kafka producer health
- External service availability (KKB, TCMB)
- Disk space

### 8. K8s probes (Phase 11 preview)

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  failureThreshold: 3
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  failureThreshold: 1
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  failureThreshold: 30
  periodSeconds: 10
```

**Banking pratiği:**
- **Liveness fail:** Pod restart
- **Readiness fail:** Traffic'ten çıkar (service endpoints'ten remove)
- **Startup:** Slow start (Spring Boot 30-60 sn) için ekstra grace period

### 9. Consul

HashiCorp Consul — service discovery + KV store + service mesh.

```yaml
spring:
  cloud:
    consul:
      host: consul-server
      port: 8500
      discovery:
        service-name: account-service
        health-check-path: /actuator/health
        health-check-interval: 10s
        tags:
          - zone=${ZONE}
          - version=${app.version}
```

**Avantajlar:**
- Multi-data-center
- KV store (config + service discovery birleşik)
- Service mesh (Connect)

**Banking pratiği:** Multi-region setup için ideal. K8s ile birlikte (K8s service mesh).

### 10. Service mesh (Istio, Linkerd) — modern alternatif

Service mesh: Service-to-service communication abstraction layer.

```
Service A → [sidecar proxy] → [sidecar proxy] → Service B
            (mTLS, retry, CB, tracing)
```

Faydalar:
- mTLS otomatik
- Retry, circuit breaker proxy'de (kod'da değil)
- Traffic shaping (canary, blue-green)
- Observability built-in

Banking modern banking ekipleri Istio'ya yatırıyor. Yine de Spring Cloud yaygın.

### 11. Banking karar matrisi

| Senaryo | Önerilen |
|---|---|
| K8s deployment | K8s Service (DNS) |
| K8s + multi-region | Consul veya Istio |
| Legacy bare metal | Eureka |
| Java-only ecosystem | Eureka veya K8s |
| Polyglot (Go, Python, Java) | Consul, K8s, Istio |
| Modern startup | Istio + K8s |

**Banking ortalama:** K8s + Spring Cloud LoadBalancer (Eureka değil) en yaygın.

### 12. Banking örnek — full setup

#### Maven module structure

```
core-banking/
├── eureka-server/             (optional, K8s'te yok)
├── api-gateway/
├── account-service/
├── transfer-service/
├── fraud-service/
└── notification-service/
```

#### account-service `application.yml`

```yaml
spring:
  application:
    name: account-service

# K8s-native — Eureka gerek yok
# Veya alternatif Eureka client:
# eureka:
#   client:
#     service-url:
#       defaultZone: http://eureka:8761/eureka/

server:
  port: 8081

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

#### transfer-service WebClient setup

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClient() {
        return WebClient.builder()
            .defaultHeaders(headers -> {
                headers.set("X-Source", "transfer-service");
            });
    }
    
    @Bean
    public WebClient accountServiceClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://account-service")   // service name, K8s DNS resolve
            .build();
    }
}

@Service
public class AccountServiceClient {
    
    private final WebClient accountServiceClient;
    
    public Mono<AccountDto> getAccount(UUID id, String userToken) {
        return accountServiceClient.get()
            .uri("/accounts/{id}", id)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + userToken)
            .retrieve()
            .bodyToMono(AccountDto.class);
    }
}
```

### 13. Service-to-service authentication

İki yaklaşım:

#### A) User token propagation

`transfer-service` user'ın JWT token'ını `account-service`'e geçer:

```java
public Mono<AccountDto> debit(UUID accountId, BigDecimal amount, String userToken) {
    return accountServiceClient.post()
        .uri("/accounts/{id}/debit", accountId)
        .header(HttpHeaders.AUTHORIZATION, "Bearer " + userToken)
        .bodyValue(new DebitRequest(amount))
        .retrieve()
        .bodyToMono(AccountDto.class);
}
```

Backend service user yetkisinde işlem yapar. **Authorization preserved.**

#### B) Client credentials (service-to-service trust)

```java
@Service
public class FraudServiceClient {
    
    private final WebClient fraudServiceClient;
    private final OAuth2ClientService oauth2Service;
    
    public Mono<FraudScore> scoreTransfer(TransferEvent transfer) {
        return oauth2Service.getServiceToken()
            .flatMap(token -> fraudServiceClient.post()
                .uri("/score")
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .bodyValue(transfer)
                .retrieve()
                .bodyToMono(FraudScore.class));
    }
}
```

Service kendi adına konuşur. User yetki gerektirmeyen internal işlem.

### 14. mTLS (mutual TLS) service-to-service

Banking için **şart**:

K8s Service mesh (Istio) ile otomatik. Veya Spring Boot manual:

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:account-service-keystore.p12
    key-store-password: ${KS_PWD}
    key-store-type: PKCS12
    client-auth: need   # mTLS
    trust-store: classpath:banking-truststore.p12
    trust-store-password: ${TS_PWD}
```

Phase 8 (Security) detay.

### 15. Banking anti-pattern'leri

**Anti-pattern 1: Hard-coded URLs production'da**

```yaml
account-service.url: http://10.0.0.5:8081   # ❌
```

K8s pod restart → IP değişir → 5xx errors.

**Anti-pattern 2: Discovery server SPOF**

Eureka tek node → fail = registry yok = service discovery down.

**Banking:** Eureka cluster (3+ node, peer-aware). Veya K8s native (highly available).

**Anti-pattern 3: Health check'siz registration**

Dead instance'lar registry'de kalır, traffic gider, fail.

**Banking:** Spring Boot Actuator health endpoint + Eureka/K8s integration zorunlu.

**Anti-pattern 4: Eureka K8s'te + K8s Service**

İkili discovery — kafa karışıklığı, lookup chaos. **Sadece birini seç.**

K8s'te: K8s Service. Bare metal: Eureka.

**Anti-pattern 5: Service name'i version'da değiştirme**

```
v1: account-service
v2: account-service-v2   # ❌ Eureka registry'e ayrı entry
```

Client zorunlu coupling. **Versioning URL path'inde** (`/v1/`, `/v2/`).

**Anti-pattern 6: Service-to-service public endpoint**

```
transfer-service → external API gateway → account-service
```

Public gateway round-trip + auth check. **Direct internal call** (cluster network).

---

## Önemli olabilecek araştırma kaynakları

- Spring Cloud Netflix Eureka documentation
- Spring Cloud LoadBalancer reference
- Spring Cloud Kubernetes documentation
- Consul official documentation
- Istio service discovery docs
- "Microservices Patterns" — service discovery chapter

---

## Mini task'ler

### Task 7.4.1 — Eureka server setup (30 dk)

`eureka-server` Maven module. `@EnableEurekaServer`. Port 8761. UI'da bir şey görmek.

### Task 7.4.2 — 4 servisi Eureka'ya register et (45 dk)

account, transfer, fraud, notification Eureka client config'i. Hepsi UP olarak Eureka UI'da görünmeli.

### Task 7.4.3 — Gateway'den `lb://account-service` çağrı (30 dk)

API Gateway route'unda `uri: lb://account-service`. Service name resolve, traffic backend'e ulaşıyor mu?

Backend instance'ı kapat, başka instance açtır → otomatik failover.

### Task 7.4.4 — K8s Service alternatif (45 dk)

minikube veya kind ile K8s cluster. Account service Deployment + Service yaml. `kubectl exec` ile başka pod'dan `curl http://account-service:8081/actuator/health` çalışsın.

Spring Cloud Kubernetes dependency ekle. Discovery K8s-based.

### Task 7.4.5 — Health check + auto-eviction (30 dk)

Custom `DatabaseHealthIndicator`. DB'yi durdur → health DOWN → Eureka instance'ı eject etmeli.

Test: 90 saniyede registry'den düşer.

### Task 7.4.6 — Zone-aware load balancing (30 dk)

2 zone simulation:
- account-service instance zone=tr
- account-service instance zone=eu

Transfer service zone=tr config. Calls → tr instance'ına gitmeli (preference).

---

## Test yazma rehberi

### Test 7.4.1 — Service registration

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class EurekaIntegrationTest {
    
    @Autowired EurekaClient eurekaClient;
    
    @Test
    void shouldRegisterToEureka() {
        Application application = eurekaClient.getApplication("ACCOUNT-SERVICE");
        
        assertThat(application).isNotNull();
        assertThat(application.getInstances()).isNotEmpty();
        
        InstanceInfo instance = application.getInstances().get(0);
        assertThat(instance.getStatus()).isEqualTo(InstanceStatus.UP);
    }
}
```

### Test 7.4.2 — Load balanced WebClient

```java
@SpringBootTest
class LoadBalancedClientTest {
    
    @Autowired @LoadBalanced WebClient.Builder webClientBuilder;
    
    @MockBean LoadBalancerClient loadBalancerClient;
    
    @Test
    void shouldResolveServiceNameToInstance() {
        ServiceInstance instance = new DefaultServiceInstance(
            "account-service:abc", "account-service",
            "10.0.0.5", 8081, false);
        when(loadBalancerClient.choose("account-service")).thenReturn(instance);
        
        WebClient client = webClientBuilder.baseUrl("http://account-service").build();
        
        // Mock external HTTP — assert URL was resolved to 10.0.0.5:8081
        Mono<String> result = client.get().uri("/test")
            .retrieve()
            .bodyToMono(String.class);
        
        StepVerifier.create(result).expectNext("...").verifyComplete();
    }
}
```

---

## Claude-verify prompt

```
Service discovery setup'ımı banking-grade kriterlere göre değerlendir:

1. Discovery choice:
   - K8s'te K8s Service + DNS kullanıyor mu (Eureka değil)?
   - Eureka cluster (3+ node, peer-aware)?
   - Bare metal'de Eureka uygun seçim?

2. Service registration:
   - Spring application.name explicit?
   - Health check endpoint expose edilmiş?
   - prefer-ip-address K8s için?
   - Metadata (zone, version, tenant)?

3. Client-side load balancing:
   - @LoadBalanced WebClient.Builder?
   - http://service-name URL (lb://)?
   - Strategy (round-robin, random) bilinçli?

4. Health check:
   - Actuator /health expose?
   - Custom health indicator (DB, Kafka)?
   - K8s liveness vs readiness ayrı?

5. Zone-aware:
   - Multi-AZ deployment'larda zone preference?
   - Metadata.zone set?

6. Service authentication:
   - User token propagation (authorization preserved)?
   - Client credentials (service-to-service trust)?
   - mTLS ya da OAuth2?

7. Anti-pattern:
   - Hard-coded URLs?
   - Discovery server SPOF (tek node)?
   - Health check'siz registration?
   - İkili discovery (Eureka + K8s)?

8. Test:
   - Service registration test?
   - Load balancer resolution test?
   - Failover test (instance down)?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Discovery seçimi gerekçeli (K8s vs Eureka vs Consul)
- [ ] 4 servis registered (UI'da görünür)
- [ ] Gateway lb:// service-name routing
- [ ] Health check + auto-eviction çalışıyor
- [ ] Zone-aware load balancing config
- [ ] Service-to-service auth (token propagation)
- [ ] mTLS veya internal token pattern
- [ ] K8s Service alternative denedim (minikube)
- [ ] Failover test (instance crash → auto redirect)

---

## Defter notları (10 madde)

1. "Client-side vs server-side discovery + DNS-based farkı: ____."
2. "Eureka register + heartbeat (30s) + eviction (90s) lifecycle: ____."
3. "K8s Service DNS otomatik discovery (FQDN format): ____."
4. "Spring Cloud LoadBalancer + @LoadBalanced WebClient: ____."
5. "Zone-aware load balancing AWS multi-AZ banking için: ____."
6. "Health check Spring Boot Actuator + custom indicator: ____."
7. "Service-to-service auth (user token vs client credentials): ____."
8. "Eureka SPOF mitigation 3+ peer cluster: ____."
9. "Banking için karar (K8s native vs Eureka vs Consul): ____."
10. "Service mesh (Istio) ile sidecar pattern banking adoption: ____."
