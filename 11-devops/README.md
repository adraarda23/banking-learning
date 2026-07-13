# Faz 11 — DevOps: Docker, Kubernetes, CI/CD

## Faz hedefi

Java backend developer'sın ama 2026 Türkiye bankacılığında **DevOps bilmeyen junior** mid-level rolüne geçemez. Sebep: TR bankalarının büyük çoğunluğu (Akbank, Garanti BBVA, İş Bankası, Yapı Kredi, Ziraat, Halkbank, QNB, Denizbank, Vakıfbank vb.) artık on-prem **OpenShift / Tanzu Kubernetes** ya da **private cloud (TurkTelekom, Vodafone, KOBI Cloud) Kubernetes** kullanıyor. Eski PaaS (Weblogic, JBoss EAP, IBM WAS) yerini Spring Boot fat JAR + container'lı dağıtıma bırakıyor. Açık iş ilanlarında "Docker, Kubernetes, CI/CD pipeline tecrübesi" artık nice-to-have değil **must**.

Bu fazın sonunda şunları yapabileceksin:

- Bir Spring Boot uygulamasını **production-grade multi-stage Dockerfile** ile container'a koymak (JRE-only, layer cache, distroless seçenekleri, JIB alternatifi)
- `docker compose` ile **tam lokal stack**'i (PostgreSQL + Kafka + Keycloak + Prometheus + Grafana + Jaeger + Loki) tek komutla ayağa kaldırmak
- **Kubernetes** temel objelerini (Pod, Deployment, Service, Ingress, ConfigMap, Secret, StatefulSet) anlayıp manifest yazmak
- **Liveness / readiness / startup probe**'larını doğru yapılandırmak (banking'de en sık production incident sebebi yanlış probe konfigürasyonu)
- **Resource requests/limits** ve **HPA (Horizontal Pod Autoscaler)** ile uygulamayı autoscale etmek
- **Helm chart** veya **Kustomize** ile environment-bazlı (dev/test/uat/prod) konfigürasyon yönetmek
- **GitHub Actions** ile PR build + test + SonarQube + image push + deploy pipeline'ı kurmak
- **Jenkins** ile aynı pipeline'ı yapmak (TR bankaları hâlâ çok kullanıyor)
- **Blue-green / canary deployment** stratejilerini tanımak ve banking risk perspektifinden değerlendirmek

## Faz süresi

- Okuma + topic kavramları: ~12 saat
- Mini task'ler: ~16 saat
- Mini-project: 2-3 gün
- **Toplam: ~1 hafta yoğun, 2 hafta normal tempo**

## Önbilgi

- Faz 1-10 tamamlandı: `core-banking` projesi çalışıyor, JPA + Flyway + REST + Kafka + observability ile entegre
- Linux temel komutları (`ls`, `cd`, `ps`, `grep`, `tail`, `find`, environment variable export)
- Bash / shell script okuma seviyesi
- Git kullanımı (branch, merge, rebase, tag)
- HTTP, port, TCP/IP temelleri

## Faz topic'leri

| # | Topic | Süre | Çıktı |
|---|-------|------|-------|
| 1 | [Docker for Java](./01-docker-for-java/index.md) | ~4-5 saat | Multi-stage Dockerfile, JIB, CDS, distroless |
| 2 | [Docker Compose](./02-docker-compose/index.md) | ~3-4 saat | Tam lokal banking stack `compose.yml` |
| 3 | [Kubernetes Basics](./03-kubernetes-basics/index.md) | ~6-8 saat | Pod/Deployment/Service/Ingress/CM/Secret/HPA manifests |
| 4 | [CI/CD](./04-ci-cd/index.md) | ~5-6 saat | GitHub Actions + Jenkinsfile + SonarQube + deploy |
| MP | [Mini-Project](./mini-project/index.md) | 2-3 gün | E2E containerized + K8s deploy + CI/CD pipeline |

## Faz çıktısı

Mini-project sonunda elinde olacak:

- `core-banking/Dockerfile` (multi-stage, JRE-only final image, ~180 MB)
- `core-banking/.dockerignore`
- `core-banking/compose.yml` (banking stack: app + Postgres + Kafka + Zookeeper + Keycloak + Prometheus + Grafana + Jaeger + Loki + Tempo)
- `core-banking/k8s/` (Deployment, Service, Ingress, ConfigMap, Secret, HPA YAML'leri — Kustomize ile dev/prod overlay)
- `core-banking/.github/workflows/ci.yml` (PR build + test + JaCoCo + SonarQube)
- `core-banking/.github/workflows/cd.yml` (main push → image build + push to GHCR + deploy to minikube)
- `core-banking/Jenkinsfile` (alternatif, declarative pipeline)
- Çalışan minikube cluster'da deploy edilmiş `core-banking` (curl ile erişebiliyorsun)

## Banking domain gerçekleri — bu fazda ne anlatacağız

### TR bankalarında DevOps gerçek durumu (2026 itibarıyla)

**On-prem Kubernetes / OpenShift hakim:** Banking regülasyonu (BDDK, KVKK) ve veri lokalitesi sebebiyle çoğu TR bankası **public cloud'a tam göçemiyor**. Bunun yerine:
- **OpenShift on-prem** (Akbank, Garanti, Yapı Kredi, İş Bankası — Red Hat lisanslı, klasik OCP 4.x)
- **VMware Tanzu (TKGI / TKG)** — eski PCF/PAS göçü yapan bankalar
- **Vanilla Kubernetes + Rancher** — daha yeni kuran küçük bankalar, fintech'ler
- **Private cloud Kubernetes** (Turkcell PaaS, Vodafone Cloud, BulutTürk) — ortaya çıkıyor ama ağırlık on-prem'de

**Public cloud kullanımı kısıtlı ama var:** Sadece **non-customer-data** workload'ları (CRM, marketing, analytics) Azure/AWS'e gidebiliyor. Customer/finansal veri **on-prem zorunlu** çoğunlukla.

**CI/CD araç dağılımı:**
- **Jenkins**: Hâlâ TR bankalarının ~%60'ında ana CI/CD aracı. Eski ama çalışıyor, plugin ekosistemi geniş, Groovy DSL ile pipeline yazılıyor. Junior'ın **mutlaka** görmesi gereken.
- **GitLab CI**: %25 civarı. Self-hosted GitLab kullanan bankalarda yaygın.
- **GitHub Actions**: Public cloud / SaaS friendly bankalarda yeni kullanılmaya başlandı, ama on-prem GitHub Enterprise Server + Actions runner setup'ı gerektiriyor.
- **Azure DevOps Pipelines**: Microsoft yatırımı yapan bankalar (Halkbank, bazı katılım bankaları).
- **Argo CD / Flux** (GitOps): Modernleşen bankalarda yeni yeni geliyor.

**Image registry:**
- **Harbor** (open-source) — TR bankalarında en yaygın self-hosted registry. RBAC, scanning (Trivy/Clair), replication ile.
- **Nexus Repository Manager** — Maven + Docker tek aracadda, eski bankalarda hâkim.
- **JFrog Artifactory** — premium, büyük bankalarda var.
- **Docker Hub / GHCR**: production'da YOK (regülasyon), sadece eğitim ve POC için.

**Deployment stratejileri TR banking:**
- **Maintenance window** (gece 02:00-06:00): hâlâ çok yaygın, "downtime kabul edilir" mentalitesi
- **Rolling update**: Kubernetes default, ama probe'lar yanlış ayarlanırsa 5xx fırlatır
- **Blue-green**: Critical servisler (core banking, ATM gateway) için kullanılıyor
- **Canary**: Daha yeni bankalarda (Akbank Lab, Garanti Digital Assets, Enpara) görülmeye başladı

**Regülasyon kısıtları DevOps'a yansıması:**
- **BDDK BS-MD-007** veri lokalitesi: image registry, log, backup TR'de olmalı
- **PCI-DSS**: Card data işleyen servislerin pipeline'ında SAST + DAST + image scan zorunlu
- **KVKK**: Production data dev/test'e kopyalanamaz, anonymization gerekli
- **SOX**: Production değişikliklerinin audit log'u, approver imzası, change ticket'ı (ServiceNow / Jira) olmalı

### Junior'ın yaptığı en büyük 5 DevOps hatası

1. **Dockerfile'da Maven build runtime'da yapmak (single-stage)**: Image 800 MB+ olur. Multi-stage ile 180 MB.
2. **Probe'ları aynı yapmak (liveness == readiness == startup)**: Slow-starting app kubernetes tarafından "öldürülür" sonsuza dek restart loop'a girer.
3. **Resource limits hiç vermemek**: Pod node'un tüm RAM'ini yer, **diğer pod'ları OOMKill ettirir**, multi-tenant cluster'da yangın çıkarır.
4. **Secret'ı ConfigMap'e koymak**: DB password environment variable olarak ConfigMap'te → audit fail, **incident** olur.
5. **Pipeline'da `mvn deploy` yapmak ama test atlamak**: "Pipeline yeşil ama production crashed" senaryosu.

Bu 5 hatanın hepsini fazın sonunda **yapmıyor olacaksın**.

## Faz sonrası

Faz 11'i bitirdiğinde:

1. [`PHASE_TEST.md`](./PHASE_TEST.md)'i çöz, ≥%80 ile geçtiğine emin ol
2. CV'ne ekle: "Docker (multi-stage), Kubernetes (Deployment, Service, ConfigMap, HPA, Helm), CI/CD (GitHub Actions, Jenkins), GitOps awareness"
3. GitHub'da projenin `Dockerfile`, `k8s/`, `.github/workflows/` klasörlerinin görünür olduğundan emin ol — recruiter ilk bunlara bakıyor
4. Faz 12 (Testing) → integration test, contract test, performance test, mutation testing

---

> **Not:** Bu faz, "Java developer mıyım yoksa SRE/DevOps engineer mı olacağım?" sorusunun cevabı **değil**. Java developer'sın. Ama DevOps'u **konuşabilen, kendi pipeline'ını kurabilen, production incident'ı debug edebilen** bir Java developer'sın. TR'de mid-level role bunu artık zorunlu kılıyor.
