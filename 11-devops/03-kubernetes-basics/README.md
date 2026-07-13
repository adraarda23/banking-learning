# Topic 11.3 — Kubernetes Basics for Banking

## Hedef

K8s core objects banking-grade derinlikte: Pod, Deployment, StatefulSet, Service, Ingress, ConfigMap, Secret, ServiceAccount, RBAC, NetworkPolicy, PVC, HPA, PDB. Banking deployment patterns: rolling, blue-green, canary, namespace strategy, multi-region, GitOps (ArgoCD/Flux), service mesh (Istio/Linkerd), pod security standards, BDDK uyumlu deploy stratejisi.

## Süre

Okuma: 3 saat • Mini task: 4 saat • Test: 1 saat • Toplam: ~8 saat

## Önbilgi

- Topic 11.1-11.2 bitti
- Docker container kavramı
- YAML

---

## Kavramlar

### 1. K8s mental model

```
Cluster
├── Control plane (master)
│   ├── kube-apiserver
│   ├── etcd (state store)
│   ├── scheduler
│   └── controller-manager
└── Worker nodes (compute)
    ├── kubelet (node agent)
    ├── kube-proxy (network)
    └── container runtime (containerd)

Resources (declarative):
- Pod: 1+ container, ephemeral
- Deployment: replica + rollout
- Service: stable network endpoint
- ConfigMap / Secret: config
- Ingress: HTTP routing
- PVC: persistent storage
- Job / CronJob: batch
```

### 2. Pod — fundamental unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: account-service
  labels:
    app: account-service
spec:
  containers:
    - name: app
      image: banking/account-service:1.0.0
      ports:
        - name: http
          containerPort: 8080
        - name: management
          containerPort: 8081
```

**Banking pratiği:** Pod **direct yaratma** yapma — Deployment kullan.

### 3. Deployment — banking standard

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: account-service
  namespace: banking
  labels:
    app: account-service
    tier: backend
    domain: banking
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # Banking — no downtime
  selector:
    matchLabels:
      app: account-service
  template:
    metadata:
      labels:
        app: account-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: account-service
      
      # Security context — pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      
      # Topology — distribute across zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: account-service
      
      containers:
        - name: app
          image: registry.mavibank.com/banking/account-service:1.0.0
          imagePullPolicy: IfNotPresent
          
          ports:
            - name: http
              containerPort: 8080
            - name: management
              containerPort: 8081
          
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -Duser.timezone=Europe/Istanbul"
            - name: KEYCLOAK_ISSUER
              valueFrom:
                configMapKeyRef:
                  name: banking-config
                  key: keycloak.issuer
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: banking-config
                  key: db.host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: banking-secrets
                  key: db.password
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://otel-collector.observability:4317
            - name: OTEL_SERVICE_NAME
              value: account-service
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          
          resources:
            requests:
              memory: 1Gi
              cpu: 500m
            limits:
              memory: 2Gi
              cpu: 2000m
          
          # Container security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
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
            periodSeconds: 10
            failureThreshold: 30   # 5 min startup window
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: gc-log
              mountPath: /var/log/banking
            - name: dumps
              mountPath: /dumps
            - name: secrets-vault
              mountPath: /secrets
              readOnly: true
      
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: gc-log
          emptyDir:
            sizeLimit: 500Mi
        - name: dumps
          emptyDir:
            sizeLimit: 4Gi   # Heap dump space
        - name: secrets-vault
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: banking-vault
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: account-service
                topologyKey: kubernetes.io/hostname
      
      terminationGracePeriodSeconds: 60
```

**Banking değerlendirme:**

- `maxUnavailable: 0` — zero downtime mandatory
- `runAsNonRoot: true` + `readOnlyRootFilesystem` — security baseline
- `drop: ALL` capabilities — minimum privilege
- `topologySpreadConstraints` — fault tolerance (zone distribute)
- `podAntiAffinity` — same-node multiple pod yok
- Resources requests + limits both set
- 3 probe types (liveness, readiness, startup)
- `terminationGracePeriodSeconds: 60` — graceful shutdown

### 4. Service — stable endpoint

```yaml
apiVersion: v1
kind: Service
metadata:
  name: account-service
  namespace: banking
spec:
  type: ClusterIP   # Internal only
  selector:
    app: account-service
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: management
      port: 8081
      targetPort: management

---
# Headless service (for direct pod access — banking nadir)
apiVersion: v1
kind: Service
metadata:
  name: account-service-headless
spec:
  clusterIP: None
  selector:
    app: account-service
```

**Service types:**
- `ClusterIP` — internal (banking standard)
- `NodePort` — exposes on node (dev/test)
- `LoadBalancer` — cloud provider LB
- `ExternalName` — DNS alias

Banking: external traffic via **Ingress**, not LoadBalancer.

### 5. Ingress — HTTP routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-api
  namespace: banking
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "5m"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains; preload";
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.mavibank.com
      secretName: api-mavibank-tls
  rules:
    - host: api.mavibank.com
      http:
        paths:
          - path: /v1/accounts
            pathType: Prefix
            backend:
              service:
                name: account-service
                port:
                  number: 8080
          - path: /v1/transfers
            pathType: Prefix
            backend:
              service:
                name: transfer-service
                port:
                  number: 8080
```

Banking ingress: TLS + security headers + rate limit + body size + WAF (annotation).

### 6. ConfigMap + Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: banking-config
  namespace: banking
data:
  keycloak.issuer: "https://auth.mavibank.com/realms/banking"
  db.host: "postgres.banking-data.svc.cluster.local"
  db.port: "5432"
  db.name: "banking"
  kafka.bootstrap: "kafka.banking-data:9092"
  redis.host: "redis.banking-data"
  application.yaml: |
    spring:
      datasource:
        url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
        username: ${DB_USER}
        hikari:
          maximum-pool-size: 20

---
apiVersion: v1
kind: Secret
metadata:
  name: banking-secrets
  namespace: banking
type: Opaque
stringData:
  db.password: ${DB_PASSWORD}
  jwt.secret: ${JWT_SECRET}
```

**Banking pratiği:** Secret K8s'de etcd'de plaintext stored (encrypted at rest config gerek). **Vault Secrets Operator** veya **External Secrets** preferred:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: banking-vault
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.banking-platform"
    roleName: "banking-account-service"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/banking/account-service"
        secretKey: "db_password"
```

Pod mounts → Vault dynamically issues credentials.

### 7. NetworkPolicy — banking PCI segmentation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: account-service-policy
  namespace: banking
spec:
  podSelector:
    matchLabels:
      app: account-service
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # From gateway only
    - from:
        - namespaceSelector:
            matchLabels:
              name: banking-gateway
          podSelector:
            matchLabels:
              app: gateway
      ports:
        - port: 8080
    # Prometheus scrape
    - from:
        - namespaceSelector:
            matchLabels:
              name: observability
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - port: 8081
  
  egress:
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
    # Postgres
    - to:
        - namespaceSelector:
            matchLabels:
              name: banking-data
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    # Kafka
    - to:
        - namespaceSelector:
            matchLabels:
              name: banking-data
          podSelector:
            matchLabels:
              app: kafka
      ports:
        - port: 9092
    # OTel collector
    - to:
        - namespaceSelector:
            matchLabels:
              name: observability
          podSelector:
            matchLabels:
              app: otel-collector
      ports:
        - port: 4317
```

**Banking PCI scope:** NetworkPolicy isolates card service. Sadece authorized service'ler erişebilir.

### 8. RBAC — pod permissions

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: account-service
  namespace: banking

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: account-service-role
  namespace: banking
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    resourceNames: ["banking-config", "banking-secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: account-service-binding
  namespace: banking
subjects:
  - kind: ServiceAccount
    name: account-service
    namespace: banking
roleRef:
  kind: Role
  name: account-service-role
  apiGroup: rbac.authorization.k8s.io
```

**Banking:** Minimum privilege. Default `default` SA YOK production'da.

### 9. HPA — Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: account-service
  namespace: banking
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: account-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_server_requests_seconds_count_rate
        target:
          type: AverageValue
          averageValue: "100"   # 100 req/s per pod
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 30
```

Banking için:
- Scale-up fast (peak load)
- Scale-down slow (stability)
- Custom metrics (RPS, queue depth) recommended

### 10. PDB — Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: account-service
  namespace: banking
spec:
  minAvailable: 2     # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: account-service
```

Node drain (maintenance) sırasında min 2 pod available. Banking zero-downtime için kritik.

### 11. Namespace strategy — banking

```
namespaces:
  banking            # Banking microservices
  banking-data       # Postgres, Kafka, Redis (data tier — PCI scope)
  banking-gateway    # API Gateway
  banking-auth       # Keycloak, IdP
  observability      # Prometheus, Grafana, Loki, Jaeger, OTel
  vault              # HashiCorp Vault
  ingress-nginx      # Ingress controller
  cert-manager       # TLS automation
  argocd             # GitOps
  istio-system       # Service mesh
```

Per-environment cluster yaygın banking için (dev, staging, prod separate clusters).

### 12. Deployment strategies

#### Rolling Update (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

3 replica → 4 → 3 (new) → 4 → 3 (new) → ... step by step.

Banking standard.

#### Blue-Green
```
Blue: current 3 replica
Green: new version 3 replica deploy
Test green
Switch service selector blue → green
Old blue stay rollback için
```

Banking için: high-stake deploys (regulatory release).

#### Canary (Argo Rollouts / Flagger)
```
Stable: 90% traffic
Canary: 10% traffic (new version)
Watch metrics
If good → promote to 50% → 100%
If bad → automatic rollback
```

Banking için: feature flags + canary, kademeli rollout.

### 13. GitOps — ArgoCD / Flux

**Declarative deployment:**

```yaml
# argo-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: banking-account-service
  namespace: argocd
spec:
  project: banking
  source:
    repoURL: https://github.com/mavibank/banking-k8s
    path: services/account-service/overlays/prod
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: banking
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Banking GitOps:**
- Git = source of truth
- Manual kubectl apply YOK
- Argo Sync detect drift + remediate
- PR review + approval flow
- Audit (git log)

### 14. Banking — K8s anti-pattern'leri

**Anti-pattern 1: latest tag in image**
- Same as Docker (Topic 11.1).

**Anti-pattern 2: default ServiceAccount**
- Permission'lar default SA'a düşer. Per-deployment SA.

**Anti-pattern 3: No resource requests**
- Scheduler hata yapar. Always set requests + limits.

**Anti-pattern 4: Privileged container**
```yaml
securityContext:
  privileged: true   # ❌
```
Banking için **ASLA**.

**Anti-pattern 5: HostPath volume**
- Node compromise. emptyDir veya PVC.

**Anti-pattern 6: No PodDisruptionBudget**
- Node drain → cluster-wide outage.

**Anti-pattern 7: HostNetwork**
- Pod sees host network. Banking için yasak.

**Anti-pattern 8: Cluster admin role binding**
- Service account cluster-admin → catastrophic. Minimal Role.

**Anti-pattern 9: No NetworkPolicy**
- Default = all pods can talk all. Banking PCI ihlali.

**Anti-pattern 10: K8s as production without backup**
- etcd corruption → cluster gone. Velero veya managed K8s.

---

## Önemli olabilecek araştırma kaynakları

- Kubernetes documentation
- "Kubernetes Patterns" — Bilgin Ibryam
- "Production Kubernetes" — Josh Rosso
- CNCF Cloud Native Trail Map
- Pod Security Standards (NIST)
- BDDK IT regulations
- ArgoCD docs

---

## Mini task'ler

### Task 11.3.1 — Local K8s (kind / minikube) (30 dk)

`kind create cluster --name banking`. Verify `kubectl get nodes`.

### Task 11.3.2 — Namespace + RBAC (30 dk)

`banking`, `banking-data`, `observability` namespaces. ServiceAccount + Role + RoleBinding.

### Task 11.3.3 — Deployment account-service (60 dk)

Banking-grade Deployment YAML (above). Apply. Verify pod healthy.

### Task 11.3.4 — Service + Ingress (45 dk)

ClusterIP service. Nginx Ingress + TLS (self-signed dev). Verify external access.

### Task 11.3.5 — ConfigMap + Secret (30 dk)

Banking config + secrets. Pod env injection.

### Task 11.3.6 — NetworkPolicy (60 dk)

Account-service → Postgres only. Block Internet. Test with debug pod.

### Task 11.3.7 — HPA (45 dk)

CPU + custom metric (Prometheus adapter). Load test → scale up.

### Task 11.3.8 — PDB (15 dk)

minAvailable=2. Drain node simulate.

### Task 11.3.9 — Rolling update (30 dk)

Image version bump. Apply. Watch rollout. Zero-downtime verify.

### Task 11.3.10 — ArgoCD setup (60 dk)

ArgoCD install. Application CR. Auto-sync.

### Task 11.3.11 — Vault Secrets CSI (60 dk)

Vault dev mode. SecretProviderClass. Pod mounts secret.

### Task 11.3.12 — Helm chart packaging (60 dk)

`helm create banking-service`. values.yaml. `helm install`.

---

## Test yazma rehberi

```bash
# Smoke test K8s manifests
#!/bin/bash
kubectl apply -k k8s/overlays/dev

# Wait deployments ready
kubectl wait --for=condition=available --timeout=300s deployment --all -n banking

# Verify replicas
[[ $(kubectl get deploy account-service -n banking -o jsonpath='{.status.readyReplicas}') -ge 2 ]]

# Verify NetworkPolicy
kubectl run debug --rm -it --image=curlimages/curl -n default -- \
  curl -m 5 http://account-service.banking.svc/actuator/health
# Expected: timeout (NetworkPolicy blocks)

# Verify from authorized pod
kubectl run debug --rm -it --image=curlimages/curl -n banking-gateway --labels="app=gateway" -- \
  curl -m 5 http://account-service.banking.svc/actuator/health
# Expected: 200

# Verify non-root
USER=$(kubectl exec -n banking deploy/account-service -- id -u)
[[ "$USER" = "10001" ]]

# Verify read-only FS
kubectl exec -n banking deploy/account-service -- touch /test 2>&1 | grep "Read-only"
```

Helm test:
```bash
helm install --dry-run --debug banking-account ./helm/banking-service
helm template ./helm/banking-service | kubeval -
```

---

## Claude-verify prompt

```
K8s manifests'ımı banking-grade kriterlere göre değerlendir:

1. Deployment:
   - replicas >= 3?
   - rollingUpdate maxUnavailable: 0?
   - 3 probe (liveness, readiness, startup)?
   - terminationGracePeriodSeconds 30+?

2. Security context:
   - runAsNonRoot?
   - readOnlyRootFilesystem?
   - drop: ALL capabilities?
   - allowPrivilegeEscalation: false?
   - seccompProfile RuntimeDefault?

3. Resources:
   - requests + limits both?
   - K8s-realistic numbers?

4. Topology:
   - podAntiAffinity (different node)?
   - topologySpreadConstraints (different zone)?

5. ServiceAccount + RBAC:
   - Per-deployment SA (not default)?
   - Minimal Role permission?
   - No cluster-admin binding?

6. ConfigMap + Secret:
   - Vault Secrets CSI for prod?
   - No plain Secret password in git?
   - External secrets pattern?

7. NetworkPolicy:
   - Ingress whitelist?
   - Egress whitelist (DNS, DB, Kafka, OTel)?
   - Banking PCI segment?

8. Ingress:
   - TLS cert-manager?
   - Security headers (HSTS, XFO, XCT)?
   - Rate limit?
   - WAF integration?

9. HPA + PDB:
   - HPA min/max replica + CPU + custom metric?
   - PDB minAvailable 2+?
   - Scale-up fast / scale-down slow?

10. GitOps:
    - ArgoCD/Flux declarative?
    - Git as source of truth?
    - Kustomize / Helm overlay?

11. Anti-pattern:
    - latest tag YOK?
    - default SA YOK?
    - privileged YOK?
    - hostPath YOK?
    - hostNetwork YOK?
    - No NetworkPolicy YOK?
    - No PDB YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Local K8s cluster (kind)
- [ ] Banking-grade Deployment (3 replica + probes + security context + resources)
- [ ] ServiceAccount + RBAC minimal
- [ ] ConfigMap + Secret (Vault CSI)
- [ ] NetworkPolicy (ingress + egress whitelist)
- [ ] Ingress + TLS + security headers + rate limit
- [ ] HPA + custom metric
- [ ] PDB minAvailable
- [ ] Rolling update zero-downtime test
- [ ] ArgoCD GitOps
- [ ] Helm chart packaging
- [ ] 8+ smoke test

---

## Defter notları (10 madde)

1. "K8s core (Pod, Deployment, Service, Ingress, ConfigMap, Secret) banking standard: ____."
2. "Deployment 3 probe (liveness, readiness, startup) banking startup window: ____."
3. "Security context (runAsNonRoot + readOnlyRootFS + drop ALL caps): ____."
4. "Resources requests + limits + scheduler placement banking realistic: ____."
5. "NetworkPolicy ingress + egress whitelist banking PCI segmentation: ____."
6. "Service ClusterIP internal + Ingress + TLS + security headers: ____."
7. "Vault Secrets CSI external secrets banking pattern: ____."
8. "HPA CPU + custom RPS + PDB minAvailable banking resilience: ____."
9. "Rolling update vs Blue-Green vs Canary banking deploy strategy: ____."
10. "GitOps ArgoCD declarative + audit + manual kubectl apply YOK: ____."
