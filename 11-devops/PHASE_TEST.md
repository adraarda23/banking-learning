# Faz 11 — PHASE TEST

Faz 12'ye geçmeden önce kendini sına.

## Pratik test

- [ ] 10 banking service Docker image (JIB + multi-stage + layered jar + non-root)
- [ ] Trivy image scan + SBOM (Syft) + Cosign sign all images
- [ ] Docker Compose full stack (data + security + observability + services)
- [ ] K8s manifests Kustomize (base + dev/staging/prod overlays)
- [ ] Banking Deployment template (3 replica + 3 probe + security context + topology)
- [ ] NetworkPolicy default-deny + service-specific allow
- [ ] Vault Secrets CSI / External Secrets
- [ ] PostgreSQL operator (Zalando) + Kafka operator (Strimzi)
- [ ] HPA (CPU + custom RPS) + PDB minAvailable
- [ ] Ingress + TLS + cert-manager + security headers
- [ ] GitHub Actions CI/CD pipeline (per service matrix)
- [ ] SonarQube quality gate + SAST (SpotBugs) + SCA (OWASP) + DAST (ZAP) in CI
- [ ] ArgoCD ApplicationSet (multi-env)
- [ ] Argo Rollouts canary (10% → 50% → 100% + auto-rollback)
- [ ] Branch protection (2 reviewer + signed commits + CODEOWNERS)
- [ ] Production manual approval gate
- [ ] 6 production-like senaryo (deploy / rollback / security gate / hotfix / drift / DR)

## Konsept testi

### Docker for Java
- [ ] Multi-stage build + Spring Boot layered jar cache
- [ ] JIB plugin Dockerfile-less + reproducible
- [ ] Base image (Temurin / Distroless / Alpine) banking trade-off
- [ ] Non-root USER 10001 + read-only root FS + drop capabilities
- [ ] Banking JVM flags (G1GC + heap dump + JFR + Istanbul TZ + UTF-8)
- [ ] Healthcheck Actuator separate port
- [ ] Trivy scan + SBOM + Cosign sign supply chain
- [ ] MaxRAMPercentage K8s memory limit consistency
- [ ] Image registry private + version tag + digest
- [ ] Anti-pattern (root, latest, JDK, secrets, single layer, no scan)

### Docker Compose
- [ ] Compose v2 + docker compose command modern
- [ ] Healthcheck + depends_on service_healthy + start_period
- [ ] Multiple network (frontend/backend/data) + internal:true
- [ ] File-based secrets + chmod + Spring Boot config import
- [ ] Profile-based partial startup banking workflow
- [ ] Override files (base + dev + CI)
- [ ] Init script seed banking demo data
- [ ] Resource limits memory + CPU K8s-realistic
- [ ] Compose dev/CI vs production K8s ayrımı
- [ ] Smoke test end-to-end script

### Kubernetes Basics
- [ ] Core (Pod, Deployment, Service, Ingress, ConfigMap, Secret) banking
- [ ] Deployment 3 probe (liveness/readiness/startup) banking startup window
- [ ] Security context (runAsNonRoot + readOnlyRootFS + drop ALL caps)
- [ ] Resources requests + limits scheduler placement
- [ ] NetworkPolicy ingress + egress whitelist banking PCI
- [ ] Service ClusterIP internal + Ingress TLS + security headers
- [ ] Vault Secrets CSI external secrets banking
- [ ] HPA CPU + custom RPS + PDB minAvailable
- [ ] Rolling update vs Blue-Green vs Canary banking strategy
- [ ] GitOps ArgoCD declarative + audit + no manual kubectl

### CI/CD
- [ ] Pipeline 6 stage (build + test + quality + security + image + deploy)
- [ ] SonarQube quality gate wait + block merge
- [ ] SAST + SCA + DAST + image scan defense-in-depth
- [ ] JMH benchmark PR vs main + regression % gate
- [ ] Image JIB + Trivy + SBOM + Cosign sign
- [ ] GitOps ArgoCD declarative + CI no kubectl + drift detect
- [ ] Canary Argo Rollouts + metric analysis + auto-rollback
- [ ] Branch protection (2 reviewer + CODEOWNERS + signed commits)
- [ ] Production approval gate + change management ticket
- [ ] Secrets OIDC short-lived + Vault dynamic

## Banking compliance check

- [ ] BDDK Bilgi Sistemleri Tebliği: data residency, DR, audit, pentest
- [ ] BDDK change management: ITSM ticket linked, approval gate
- [ ] PCI-DSS network segmentation (NetworkPolicy + data tier isolation)
- [ ] KVKK data residency (TR cluster)
- [ ] Signed images + SBOM (supply chain regulatory)
- [ ] Audit trail (git log + ArgoCD history + cluster audit log)
- [ ] Backup / DR (Velero / managed K8s)
- [ ] Penetration test annual (DAST in CI baseline)

## Soft skills

- [ ] 15 mini-project defter notu yazılmış
- [ ] GitOps philosophy internalize (git = single source of truth)
- [ ] Progressive delivery (canary + auto-rollback) practice
- [ ] DR plan + RTO/RPO targets banking realistic
- [ ] BDDK + KVKK + PCI-DSS compliance deploy mapping

## Kaç gün?

Tahmin: 25-30 gün (günde 3 saat). Phase 11 = **production deployment maturity** banking için. Bu phase senior backend → DevOps overlap zone.

---

Hepsine "evet" → **Faz 12'ye geç → 12-testing/**

---

## Bonus — banking platform engineer işareti

Phase 11 bittikten sonra şu cümleleri söyleyebiliyorsan:

- "Banking microservice'i sıfırdan production-grade containerize edip K8s'e deploy edebilirim (JIB + multi-stage + Kustomize + Helm)."
- "GitOps ile declarative deployment + drift detect + auto-heal kurabilirim."
- "Argo Rollouts canary + Prometheus metric analysis + auto-rollback design ederim."
- "K8s security hardening (non-root + readOnlyFS + drop caps + NetworkPolicy + PSP/PSS) uygulayabilirim."
- "BDDK + KVKK + PCI-DSS compliance K8s deploy stratejisi ile birlikte düşünebilirim."
- "Supply chain integrity (Cosign sign + SBOM + Trivy scan + admission policy) kurabilirim."
- "DR plan + multi-region + RTO/RPO targets banking için tasarlayabilirim."
- "Service mesh (Istio) mTLS + traffic splitting banking için entegre edebilirim."
- "Vault Secrets CSI + dynamic credentials + audit + rotation."
- "JMH regression CI gate + benchmark dedicated runner banking pattern."

TR bankalarında **mid-senior backend + platform engineer** dual hat'i için rekabetçi profil. Banking için **DevSecOps maturity** seniorluğun göstergelerinden.

---

## Banking için Phase 11'in yeri

Phase 11 = **deployment operability**. TR bankaları için:

- **Cloud-native migration:** Legacy → K8s + GitOps geçişi
- **BDDK denetim:** Change management + audit + DR mandatory
- **Zero-downtime:** Bankacılık SLA 99.95% upgrade gerektirir
- **Supply chain:** Image signing + SBOM regulatory trend
- **Compliance gate:** Production deploy ITSM + approval + audit

Mid+ engineer **deployment'a ait** bilgi olmadan kariyer ilerletemiyor. Phase 11 + Phase 12 (Testing) ile **complete profile**.

Senior banking engineer için Phase 11 = "kod yazıp gönderilmesi" vs "production'a güvenle gidişi" arasındaki fark. Bu fark — değer farkı.
