# Phase 11 Mini-Project — Banking Production Deployment Pipeline

## Hedef

Phase 7-10 banking microservice'lerini **production-grade** deploy etmek: Docker image (multi-stage + JIB + scan + sign), K8s manifests (security + observability + autoscale), CI/CD pipeline (build + test + quality + deploy), GitOps (ArgoCD), canary rollout.

## Süre

10-12 gün (günde 3 saat)

## Önbilgi

- Phase 10 mini-project tamam
- Topic 11.1-11.4 bitti

---

## Görev listesi

### 1. Banking image build — all services (1.5 gün)

Phase 10'daki 8 service için:
- account-service
- transfer-service
- card-service
- xborder-service
- loan-service
- fx-service
- compliance-service
- recon-service
- gateway-service
- ledger-service

Her biri için:

**JIB plugin** ile build:
```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <configuration>
        <from>
            <image>eclipse-temurin:21-jre-jammy</image>
        </from>
        <to>
            <image>registry.mavibank.com/banking/${project.artifactId}</image>
            <tags>
                <tag>${project.version}</tag>
                <tag>latest</tag>
            </tags>
        </to>
        <container>
            <user>10001:10001</user>
            <jvmFlags>
                <jvmFlag>-XX:MaxRAMPercentage=75.0</jvmFlag>
                <jvmFlag>-XX:+UseG1GC</jvmFlag>
                <jvmFlag>-XX:MaxGCPauseMillis=200</jvmFlag>
                <jvmFlag>-XX:+HeapDumpOnOutOfMemoryError</jvmFlag>
                <jvmFlag>-XX:HeapDumpPath=/dumps</jvmFlag>
                <jvmFlag>-Duser.timezone=Europe/Istanbul</jvmFlag>
                <jvmFlag>-Dfile.encoding=UTF-8</jvmFlag>
            </jvmFlags>
            <ports>
                <port>8080</port>
                <port>8081</port>
            </ports>
        </container>
    </configuration>
</plugin>
```

Image scan (Trivy) + sign (Cosign) every build.

### 2. K8s manifests — all services (2 gün)

Directory structure (GitOps repo):

```
banking-k8s/
├── base/
│   ├── account-service/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   ├── secretproviderclass.yaml
│   │   └── kustomization.yaml
│   ├── transfer-service/ ...
│   ├── ... 8 more services
│   └── gateway/
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── staging/
│   └── prod/
│       ├── kustomization.yaml
│       ├── patches/
│       └── argocd-rollout.yaml   # Canary config
├── shared/
│   ├── namespaces.yaml
│   ├── rbac.yaml
│   ├── networkpolicy-default-deny.yaml
│   ├── pod-security-standards.yaml
│   ├── ingress.yaml
│   └── cert-manager/
└── infrastructure/
    ├── postgres/
    ├── kafka/
    ├── keycloak/
    ├── vault/
    └── observability/
```

Banking-grade Deployment template (per service):
- 3 replica
- maxUnavailable: 0
- Non-root + readOnlyRootFS + dropAll caps
- Resources requests + limits
- Probes (liveness + readiness + startup)
- NetworkPolicy ingress + egress whitelist
- Vault Secrets CSI
- topologySpreadConstraints zone
- PodAntiAffinity host
- PDB minAvailable 2
- HPA CPU + custom RPS

### 3. Infrastructure — K8s manifests (1.5 gün)

PostgreSQL operator (Zalando Postgres Operator):
```yaml
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: banking-pg
  namespace: banking-data
spec:
  teamId: "banking"
  volume:
    size: 100Gi
    storageClass: fast-ssd
  numberOfInstances: 3
  users:
    banking:
      - superuser
      - createdb
  databases:
    banking: banking
  postgresql:
    version: "16"
    parameters:
      max_connections: "200"
      shared_buffers: "2GB"
      work_mem: "16MB"
  resources:
    requests:
      cpu: 1
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi
```

Kafka Strimzi operator:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: banking-kafka
  namespace: banking-data
spec:
  kafka:
    version: 3.7.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      log.retention.hours: 168
    storage:
      type: persistent-claim
      size: 200Gi
      class: fast-ssd
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
```

Keycloak operator, Vault, observability stack (Prometheus operator, Loki, Tempo, Grafana).

### 4. Service mesh (Istio) — optional (1 gün)

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: banking-istio
spec:
  profile: default
  meshConfig:
    accessLogFile: /dev/stdout
    defaultConfig:
      tracing:
        sampling: 10.0
        zipkin:
          address: zipkin.istio-system:9411
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
```

Sidecar injection:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: banking
  labels:
    istio-injection: enabled
```

mTLS PeerAuthentication:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: banking
spec:
  mtls:
    mode: STRICT   # All service-to-service mTLS
```

VirtualService routing (canary):
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: transfer-service
  namespace: banking
spec:
  hosts:
    - transfer-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: transfer-service
            subset: canary
    - route:
        - destination:
            host: transfer-service
            subset: stable
          weight: 90
        - destination:
            host: transfer-service
            subset: canary
          weight: 10
```

### 5. GitHub Actions CI/CD pipeline (1.5 gün)

Workflow per service (matrix):

```yaml
name: Banking Services CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.changes.outputs.services }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            account-service:
              - 'services/account-service/**'
            transfer-service:
              - 'services/transfer-service/**'
            # ... all services
  
  ci:
    needs: detect-changes
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    runs-on: banking-runner
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build & test
        working-directory: services/${{ matrix.service }}
        run: ./mvnw -B clean verify
      
      - name: SAST
        working-directory: services/${{ matrix.service }}
        run: ./mvnw -B com.github.spotbugs:spotbugs-maven-plugin:check
      
      - name: SCA
        working-directory: services/${{ matrix.service }}
        run: ./mvnw -B org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7
      
      - name: SonarQube
        working-directory: services/${{ matrix.service }}
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./mvnw -B sonar:sonar -Dsonar.qualitygate.wait=true
      
      - name: Build image (JIB)
        if: github.event_name == 'push'
        working-directory: services/${{ matrix.service }}
        run: |
          ./mvnw -B compile jib:build \
            -Djib.to.image=${{ env.REGISTRY }}/banking/${{ matrix.service }}:sha-${GITHUB_SHA::7} \
            -Djib.to.tags=sha-${GITHUB_SHA::7},${{ github.ref_name }}
      
      - name: Scan + sign
        if: github.event_name == 'push'
        run: |
          trivy image --severity HIGH,CRITICAL --exit-code 1 \
            ${{ env.REGISTRY }}/banking/${{ matrix.service }}:sha-${GITHUB_SHA::7}
          syft ${{ env.REGISTRY }}/banking/${{ matrix.service }}:sha-${GITHUB_SHA::7} \
            -o spdx-json=sbom.json
          cosign sign --yes --key cosign.key \
            ${{ env.REGISTRY }}/banking/${{ matrix.service }}:sha-${GITHUB_SHA::7}
  
  deploy-staging:
    needs: ci
    if: github.ref == 'refs/heads/develop'
    runs-on: banking-runner
    environment: staging
    steps:
      - uses: actions/checkout@v4
        with:
          repository: mavibank/banking-k8s
          token: ${{ secrets.GITOPS_TOKEN }}
      - name: Update tags + push
        run: |
          for service in ${{ join(needs.detect-changes.outputs.services, ' ') }}; do
            cd overlays/staging/$service
            sed -i "s|newTag: .*|newTag: sha-${GITHUB_SHA::7}|" kustomization.yaml
            cd -
          done
          git add -A
          git commit -m "deploy(staging): sha-${GITHUB_SHA::7}"
          git push
      - name: Wait ArgoCD
        run: argocd app wait banking-staging --timeout 600
  
  deploy-prod:
    needs: ci
    if: github.ref == 'refs/heads/main'
    runs-on: banking-runner
    environment: production   # Required reviewers
    steps:
      # Similar, with canary rollout via Argo Rollouts
```

### 6. ArgoCD applications (0.5 gün)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: banking-services
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - service: account-service
                - service: transfer-service
                - service: card-service
                - service: xborder-service
                - service: loan-service
                - service: fx-service
                - service: compliance-service
                - service: recon-service
                - service: ledger-service
                - service: gateway-service
          - list:
              elements:
                - env: dev
                  cluster: banking-dev
                - env: staging
                  cluster: banking-staging
                - env: prod
                  cluster: banking-prod
                  syncPolicy:
                    manual: true   # Prod manual approval
  
  template:
    metadata:
      name: '{{service}}-{{env}}'
    spec:
      project: banking
      source:
        repoURL: https://github.com/mavibank/banking-k8s
        path: overlays/{{env}}/{{service}}
        targetRevision: main
      destination:
        name: '{{cluster}}'
        namespace: banking
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        retry:
          limit: 5
```

### 7. Canary rollout (Argo Rollouts) (1 gün)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: transfer-service
  namespace: banking
spec:
  replicas: 5
  selector:
    matchLabels:
      app: transfer-service
  template:
    # ... Deployment template
  
  strategy:
    canary:
      canaryService: transfer-service-canary
      stableService: transfer-service-stable
      trafficRouting:
        istio:
          virtualService:
            name: transfer-service
          destinationRule:
            name: transfer-service
            canarySubsetName: canary
            stableSubsetName: stable
      
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
              - templateName: latency-p99
            args:
              - name: service-name
                value: transfer-service-canary
        - setWeight: 25
        - pause: { duration: 10m }
        - analysis: { ... }
        - setWeight: 50
        - pause: { duration: 15m }
        - analysis: { ... }
        - setWeight: 100

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.observability:9090
          query: |
            sum(rate(http_server_requests_seconds_count{service="{{args.service-name}}",status!~"5.."}[5m]))
            / sum(rate(http_server_requests_seconds_count{service="{{args.service-name}}"}[5m]))

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-p99
spec:
  args:
    - name: service-name
  metrics:
    - name: p99
      interval: 1m
      successCondition: result[0] <= 1.0   # 1 second
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.observability:9090
          query: |
            histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="{{args.service-name}}"}[5m])) by (le))
```

### 8. Production-like senaryolar (1 gün)

#### Senaryo 1: Successful deploy
1. PR opened
2. CI: build + test + SonarQube pass → image build + scan + sign
3. Merge to develop
4. Deploy staging
5. Smoke test pass
6. PR to main → manual approval
7. Canary 10% → metric pass → 25% → 50% → 100%
8. Production stable

#### Senaryo 2: Auto-rollback canary
1. PR introduces regression (latency spike)
2. CI passes (regression below CI threshold)
3. Canary 10% deploy
4. Argo Rollouts analysis: latency p99 > 1s → fail
5. Auto-rollback to stable
6. Slack alert + PR comment

#### Senaryo 3: Security gate fail
1. PR adds vulnerable dependency
2. OWASP dependency-check → CVE 9.8 detected
3. Build fail
4. PR blocked

#### Senaryo 4: Production hotfix
1. Critical bug in prod
2. Hotfix branch from main
3. Expedited PR (1 approver if hotfix label)
4. Quick canary 100% (skipped intermediate steps)
5. Monitor

#### Senaryo 5: GitOps drift detect
1. Someone kubectl apply manually (anti-pattern)
2. ArgoCD detect drift
3. Self-heal: revert to git state
4. Slack alert: "manual change detected, reverted"

#### Senaryo 6: Disaster recovery
1. Region down
2. Cluster lost
3. ArgoCD bootstrap secondary cluster from git
4. Database failover (replica → primary)
5. Service restored
6. RTO < 1 hour, RPO < 5 minutes

### 9. Defter notları (15 item)

1. "JIB build banking 10 service multi-tag + Cosign sign: ____."
2. "K8s manifests Kustomize base + overlays (dev/staging/prod): ____."
3. "Banking Deployment template (3 replica + security + probe + resource + topology): ____."
4. "NetworkPolicy default-deny + namespace + egress whitelist banking PCI: ____."
5. "Vault Secrets CSI + SecretProviderClass dynamic credential: ____."
6. "PostgreSQL Zalando operator + replica + storage class: ____."
7. "Kafka Strimzi operator + 3 replica + min-ISR 2: ____."
8. "Istio service mesh + mTLS STRICT + VirtualService canary routing: ____."
9. "GitHub Actions matrix CI per service + path filter: ____."
10. "Quality gate (SonarQube + SAST + SCA + image scan) defense-in-depth: ____."
11. "ArgoCD ApplicationSet multi-env + manual prod sync: ____."
12. "Argo Rollouts canary + Prometheus metric analysis + auto-rollback: ____."
13. "6 senaryo (deploy / rollback / security gate / hotfix / drift / DR) banking: ____."
14. "Banking IT compliance (BDDK change management + signed image + audit trail): ____."
15. "DevOps maturity banking (manual → CI/CD → GitOps → progressive delivery): ____."

---

## Tamamlama kriterleri

- [ ] 10 banking service Docker image (JIB + scan + sign)
- [ ] K8s manifests Kustomize (base + 3 overlay)
- [ ] Banking Deployment template (security + observability + autoscale)
- [ ] NetworkPolicy default-deny + service-specific allow
- [ ] Vault Secrets CSI integration
- [ ] PostgreSQL + Kafka operator
- [ ] Istio service mesh (optional)
- [ ] GitHub Actions CI/CD matrix
- [ ] ArgoCD ApplicationSet 10 service x 3 env
- [ ] Argo Rollouts canary + auto-rollback
- [ ] 6 senaryo reproduce + verify
- [ ] 15 defter notu

---

## Önemli not

Phase 11 = **production deployment maturity**. Banking için bu phase:

- BDDK IT regulations uyumlu deploy
- Zero-downtime deploy
- Audit trail (git + signed images)
- DR capable
- Compliance gate enforced

TR bankalarında **DevOps / Platform Engineer** roller bu skills'i istiyor. Senior backend engineer "deploy nasıl yapılıyor" değil "deploy strategy + pipeline design" perspective'inden bakabilmeli.

Phase 12 (Testing) ile **quality assurance** dimension'unu kapatacağız → complete banking engineer profile.
