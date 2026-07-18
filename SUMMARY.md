# Summary

[Banking Backend Öğrenme Yolu — Giriş](./README.md)

# Faz 1 — Foundation

- [Faz Özeti](./01-foundation/README.md)
  - [Topic 1.1 — Mimari: Hexagonal, Ports & Adapters, Layered](./01-foundation/01-architecture/README.md)
  - [Topic 1.2 — Proje Setup: Spring Boot 3 + Maven + Profiles](./01-foundation/02-project-setup/README.md)
  - [Topic 1.3 — Para İşleme: BigDecimal, RoundingMode, Currency](./01-foundation/03-money-handling/README.md)
  - [Topic 1.4 — Database Migration: Flyway](./01-foundation/04-database-migration/README.md)
  - [Topic 1.5 — API Design: DTO, MapStruct, Lombok kararı](./01-foundation/05-api-design/README.md)
  - [Topic 1.6 — Validation: Bean Validation, custom validators](./01-foundation/06-validation/README.md)
  - [Topic 1.7 — Hata Yönetimi: RFC 7807 ProblemDetail + @ControllerAdvice](./01-foundation/07-error-handling/README.md)
  - [Mini Proje: `core-banking` v0.1](./01-foundation/mini-project/README.md)
  - [Faz Testi](./01-foundation/PHASE_TEST.md)

# Faz 2 — JPA & Transactions

- [Faz Özeti](./02-jpa-transactions/README.md)
  - [Topic 2.1 — JPA Fundamentals & Entity Lifecycle](./02-jpa-transactions/01-jpa-fundamentals/README.md)
  - [Topic 2.2 — Spring Data JPA & Repositories](./02-jpa-transactions/02-spring-data-jpa/README.md)
  - [Topic 2.3 — Transaction Management](./02-jpa-transactions/03-transactions/README.md)
  - [Topic 2.4 — Locking: Optimistic & Pessimistic](./02-jpa-transactions/04-locking/README.md)
  - [Topic 2.5 — N+1 Query Problem](./02-jpa-transactions/05-n-plus-one/README.md)
  - [Topic 2.6 — HikariCP & Connection Pool Tuning](./02-jpa-transactions/06-connection-pool/README.md)
  - [Topic 2.7 — Hibernate Performance Tuning](./02-jpa-transactions/07-hibernate-performance/README.md)
  - [Mini Proje: `core-banking` v0.2 (Concurrency & Performance Hardening)](./02-jpa-transactions/mini-project/README.md)
  - [Faz Testi](./02-jpa-transactions/PHASE_TEST.md)

# Faz 3 — Concurrency & JVM

- [Faz Özeti](./03-concurrency/README.md)
  - [Topic 3.1 — Java Memory Model (JMM) & Happens-Before](./03-concurrency/01-jmm-memory-model/README.md)
  - [Topic 3.2 — Synchronization Primitives: synchronized, volatile, Atomics, CAS](./03-concurrency/02-synchronization-primitives/README.md)
  - [Topic 3.3 — Lock Family: ReentrantLock, ReadWriteLock, StampedLock, Condition, Deadlock](./03-concurrency/03-locks/README.md)
  - [Topic 3.4 — Executor Framework: ThreadPoolExecutor, Queues, ForkJoinPool](./03-concurrency/04-executor-framework/README.md)
  - [Topic 3.5 — CompletableFuture & Async Composition](./03-concurrency/05-completable-future/README.md)
  - [Topic 3.6 — Concurrent Collections](./03-concurrency/06-concurrent-collections/README.md)
  - [Topic 3.7 — Virtual Threads (Project Loom)](./03-concurrency/07-virtual-threads-loom/README.md)
  - [Topic 3.8 — Structured Concurrency](./03-concurrency/08-structured-concurrency/README.md)
  - [Topic 3.9 — JVM Internals: Memory, GC, JIT](./03-concurrency/09-jvm-internals/README.md)
  - [Topic 3.10 — Performance Tools (JFR, async-profiler, MAT, jstack)](./03-concurrency/10-performance-tools/README.md)
  - [Topic 3.11 — JMH (Java Microbenchmark Harness)](./03-concurrency/11-jmh-benchmarking/README.md)
  - [Mini Proje: Concurrency Hardening](./03-concurrency/mini-project/README.md)
  - [Faz Testi](./03-concurrency/PHASE_TEST.md)

# Faz 4 — SQL & Oracle

- [Faz Özeti](./04-sql-oracle/README.md)
  - [Topic 4.1 — Index Internals](./04-sql-oracle/01-index-internals/README.md)
  - [Topic 4.2 — Execution Plan & Query Tuning](./04-sql-oracle/02-execution-plan-tuning/README.md)
  - [Topic 4.3 — Window Functions & Advanced SQL](./04-sql-oracle/03-window-functions-advanced-sql/README.md)
  - [Topic 4.4 — PL/SQL Programming](./04-sql-oracle/04-plsql/README.md)
  - [Topic 4.5 — Oracle-Specific Features](./04-sql-oracle/05-oracle-specific/README.md)
  - [Topic 4.6 — DB Concurrency & Locking](./04-sql-oracle/06-db-concurrency-locking/README.md)
  - [Mini Proje: `core-banking` Oracle Migration & PL/SQL](./04-sql-oracle/mini-project/README.md)
  - [Faz Testi](./04-sql-oracle/PHASE_TEST.md)

# Faz 5 — Spring Batch: Banking EOD ve Toplu İşleme

- [Faz Özeti](./05-batch/README.md)
  - [Topic 5.1 — Spring Batch Mimarisi: Job, Step, JobRepository](./05-batch/01-batch-architecture/README.md)
  - [Topic 5.2 — Chunk-Oriented Processing](./05-batch/02-chunk-oriented/README.md)
  - [Topic 5.3 — Skip, Retry, Restart](./05-batch/03-skip-retry-restart/README.md)
  - [Topic 5.4 — Listeners & Flow Control](./05-batch/04-listeners-flow/README.md)
  - [Topic 5.5 — Partitioning & Parallel Execution](./05-batch/05-partitioning-parallel/README.md)
  - [Topic 5.6 — Scheduling: @Scheduled, Quartz, ShedLock, K8s CronJob](./05-batch/06-scheduling/README.md)
  - [Mini Proje: Banking EOD Job Suite](./05-batch/mini-project/README.md)
  - [Faz Testi](./05-batch/PHASE_TEST.md)

# Faz 6 — Messaging & Events (Kafka odaklı)

- [Faz Özeti](./06-messaging/README.md)
  - [Topic 6.1 — Kafka Mimarisi: Broker, Topic, Partition, Replica](./06-messaging/01-kafka-architecture/README.md)
  - [Topic 6.2 — Kafka Producer Design](./06-messaging/02-producer/README.md)
  - [Topic 6.3 — Kafka Consumer Design](./06-messaging/03-consumer/README.md)
  - [Topic 6.4 — Spring Kafka Integration Deep](./06-messaging/04-spring-kafka/README.md)
  - [Topic 6.5 — Kafka Streams: Real-Time Stream Processing](./06-messaging/05-kafka-streams/README.md)
  - [Topic 6.6 — Outbox Pattern & CDC](./06-messaging/06-outbox-cdc/README.md)
  - [Topic 6.7 — Saga Pattern](./06-messaging/07-saga/README.md)
  - [Mini Proje: Event-Driven Banking Platform](./06-messaging/mini-project/README.md)
  - [Faz Testi](./06-messaging/PHASE_TEST.md)

# Faz 7 — Microservices: Monolitten Servis-Bazlı Mimariye Geçiş

- [Faz Özeti](./07-microservices/README.md)
  - [Topic 7.1 — DDD Strategic Design: Bounded Context, Context Map, Aggregate](./07-microservices/01-ddd-strategic/README.md)
  - [Topic 7.2 — Service Decomposition](./07-microservices/02-service-decomposition/README.md)
  - [Topic 7.3 — API Gateway: Spring Cloud Gateway](./07-microservices/03-api-gateway/README.md)
  - [Topic 7.4 — Service Discovery](./07-microservices/04-service-discovery/README.md)
  - [Topic 7.5 — Resilience4j: Circuit Breaker, Retry, Bulkhead, RateLimiter, TimeLimiter, Fallback](./07-microservices/05-resilience4j/README.md)
  - [Topic 7.6 — Distributed Tracing: OpenTelemetry + Jaeger](./07-microservices/06-distributed-tracing/README.md)
  - [Topic 7.7 — Distributed Locks & Transactions](./07-microservices/07-distributed-locks-transactions/README.md)
  - [Mini Proje: Microservices Split & Resilience](./07-microservices/mini-project/README.md)
  - [Faz Testi](./07-microservices/PHASE_TEST.md)

# Faz 8 — Security

- [Faz Özeti](./08-security/README.md)
  - [Topic 8.1 — Spring Security 6 Architecture: Filter Chain, Method Security](./08-security/01-spring-security-architecture/README.md)
  - [Topic 8.2 — Authentication: UserDetails, Password Hashing, Policy, MFA](./08-security/02-authentication/README.md)
  - [Topic 8.3 — JWT: JWS, JWE, Refresh Token Rotation](./08-security/03-jwt/README.md)
  - [Topic 8.4 — OAuth2 & OIDC: Authorization Framework](./08-security/04-oauth2-oidc/README.md)
  - [Topic 8.5 — Keycloak: Production-grade IdP for Banking](./08-security/05-keycloak/README.md)
  - [Topic 8.6 — Encryption: At Rest, In Transit, Envelope, KMS](./08-security/06-encryption/README.md)
  - [Topic 8.7 — OWASP Top 10 (2021) — Banking Perspective](./08-security/07-owasp-top10/README.md)
  - [Mini Proje: Banking Security Hardening (End-to-End)](./08-security/mini-project/README.md)
  - [Faz Testi](./08-security/PHASE_TEST.md)

# Faz 9 — Observability & Performance

- [Faz Özeti](./09-observability/README.md)
  - [Topic 9.1 — Structured Logging: Logback + SLF4J + JSON + MDC](./09-observability/01-structured-logging/README.md)
  - [Topic 9.2 — Metrics: Micrometer + Prometheus + Grafana](./09-observability/02-metrics/README.md)
  - [Topic 9.3 — Distributed Tracing (Deep): OpenTelemetry + Exemplars](./09-observability/03-distributed-tracing/README.md)
  - [Topic 9.4 — Profiling: JFR + async-profiler + Flame Graphs](./09-observability/04-profiling/README.md)
  - [Topic 9.5 — Heap & Thread Dump Analysis](./09-observability/05-heap-thread-dumps/README.md)
  - [Topic 9.6 — JMH Deep & CI Integration](./09-observability/06-jmh-deep/README.md)
  - [Topic 9.7 — Load Testing: Gatling, k6, JMeter](./09-observability/07-load-testing/README.md)
  - [Mini Proje: Banking Observability Stack (End-to-End)](./09-observability/mini-project/README.md)
  - [Faz Testi](./09-observability/PHASE_TEST.md)

# Faz 10 — Domain Mastery (Banking Domain Bilgisi)

- [Faz Özeti](./10-domain/README.md)
  - [Topic 10.1 — Double-Entry Accounting (Banking Deep)](./10-domain/01-double-entry-accounting/README.md)
  - [Topic 10.2 — ISO 8583: Card Transaction Messaging](./10-domain/02-iso-8583/README.md)
  - [Topic 10.3 — ISO 20022 & SWIFT MT](./10-domain/03-iso-20022-swift/README.md)
  - [Topic 10.4 — TR Payment Systems: EFT, FAST, BKM, Troy, TCMB](./10-domain/04-tr-payment-systems/README.md)
  - [Topic 10.5 — FX & Interest Calculations](./10-domain/05-fx-interest/README.md)
  - [Topic 10.6 — Regulatory: BDDK, MASAK, KVKK, PCI-DSS, KKB, GDPR](./10-domain/06-regulatory/README.md)
  - [Topic 10.7 — Reconciliation & Settlement](./10-domain/07-reconciliation-settlement/README.md)
  - [Topic 10.8 — Lending & Credit: Kredi Yaşam Döngüsü](./10-domain/08-lending-credit/README.md)
  - [Topic 10.9 — Card Products: Ekstre, Taksit, Chargeback, Revolving](./10-domain/09-card-products/README.md)
  - [Topic 10.10 — Participation (Faizsiz) Banking: Katılım Bankacılığı](./10-domain/10-participation-banking/README.md)
  - [Mini Proje: TR Banking Domain Integration](./10-domain/mini-project/README.md)
  - [Faz Testi](./10-domain/PHASE_TEST.md)

# Faz 11 — DevOps: Docker, Kubernetes, CI/CD

- [Faz Özeti](./11-devops/README.md)
  - [Topic 11.1 — Docker for Java](./11-devops/01-docker-for-java/README.md)
  - [Topic 11.2 — Docker Compose: Banking Local Stack](./11-devops/02-docker-compose/README.md)
  - [Topic 11.3 — Kubernetes Basics for Banking](./11-devops/03-kubernetes-basics/README.md)
  - [Topic 11.4 — CI/CD: GitHub Actions, Jenkins, GitOps](./11-devops/04-ci-cd/README.md)
  - [Mini Proje: Banking Production Deployment Pipeline](./11-devops/mini-project/README.md)
  - [Faz Testi](./11-devops/PHASE_TEST.md)

# Phase 12 — Testing Mastery (FINAL PHASE)

- [Faz Özeti: Phase 12 — Testing Mastery (FINAL PHASE)](./12-testing/README.md)
  - [Topic 12.1 — JUnit 5 Advanced](./12-testing/01-junit5-advanced/README.md)
  - [Topic 12.2 — Mockito Deep](./12-testing/02-mockito-deep/README.md)
  - [Topic 12.3 — Testcontainers](./12-testing/03-testcontainers/README.md)
  - [Topic 12.4 — ArchUnit: Architecture Tests](./12-testing/04-archunit/README.md)
  - [Topic 12.5 — Contract Testing: Spring Cloud Contract, Pact](./12-testing/05-contract-testing/README.md)
  - [Topic 12.6 — Mutation Testing: PIT](./12-testing/06-mutation-testing/README.md)
  - [Topic 12.7 — Performance Testing in CI](./12-testing/07-performance-testing-ci/README.md)
  - [Mini Proje: Banking Testing Excellence](./12-testing/mini-project/README.md)
  - [Faz Testi](./12-testing/PHASE_TEST.md)
