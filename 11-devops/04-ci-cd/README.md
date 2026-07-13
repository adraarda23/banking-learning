# Topic 11.4 — CI/CD: GitHub Actions, Jenkins, GitOps

## Hedef

Banking-grade CI/CD pipeline: build, test, quality gate (SonarQube), SAST (FindSecBugs), SCA (Snyk/dependency-check), DAST (ZAP), image build/scan/sign, deploy via GitOps. GitHub Actions (modern) + Jenkins (TR yaygın). Branch protection, code review, secrets management, regulatory approval gate (banking deployment).

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 11.1-11.3 bitti
- Git temel
- Phase 12 (testing) henüz başlamadı — ama unit/integration test kavramı bilinmeli

---

## Kavramlar

### 1. CI/CD overview

```
Developer push → 
   CI: Build + Test + Quality + Security
   ↓
Pull Request → Code Review + Approval
   ↓
Merge → 
   CD: Image Build + Push + Deploy
   ↓
Production (rolling / canary)
```

Banking pratiği:
- **CI:** GitHub Actions, GitLab CI, Jenkins
- **CD (deploy):** ArgoCD / Flux (GitOps)
- **Artifact registry:** Harbor / Nexus / Artifactory
- **SAST:** SonarQube, FindSecBugs, Semgrep
- **SCA:** Snyk, OWASP dependency-check, Renovate
- **DAST:** OWASP ZAP, Burp
- **Image scan:** Trivy, Grype
- **Sign:** Cosign

### 2. GitHub Actions — banking pipeline

```yaml
# .github/workflows/banking-ci.yml
name: Banking Service CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

env:
  REGISTRY: registry.mavibank.com
  IMAGE_NAME: banking/transfer-service
  JAVA_VERSION: '21'

jobs:
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest-banking   # Dedicated runner (security)
    timeout-minutes: 30
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: banking_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U test"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # SonarQube needs git history
      
      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Cache SonarQube
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
      
      - name: Build
        run: ./mvnw -B clean compile
      
      - name: Unit Tests
        run: ./mvnw -B test
      
      - name: Integration Tests
        run: ./mvnw -B verify -Pintegration
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: banking_test
          DB_USER: test
          DB_PASSWORD: test
      
      - name: SAST — SpotBugs + FindSecBugs
        run: ./mvnw -B com.github.spotbugs:spotbugs-maven-plugin:check
      
      - name: SCA — OWASP Dependency Check
        run: ./mvnw -B org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7
      
      - name: Code Coverage Report
        run: ./mvnw -B jacoco:report
      
      - name: SonarQube Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          ./mvnw -B sonar:sonar \
            -Dsonar.projectKey=banking-transfer-service \
            -Dsonar.qualitygate.wait=true
      
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: target/site/jacoco/

  benchmark:
    name: JMH Benchmark
    runs-on: ubuntu-latest-benchmark   # Dedicated benchmark runner
    needs: build-test
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
      
      - name: Run benchmarks (PR)
        run: |
          ./mvnw -B clean package -DskipTests
          java -jar target/benchmarks.jar -rf json -rff pr-result.json -wi 3 -i 5 -f 2
      
      - name: Get baseline (main)
        run: |
          git fetch origin main
          git checkout origin/main
          ./mvnw -B clean package -DskipTests
          java -jar target/benchmarks.jar -rf json -rff main-result.json -wi 3 -i 5 -f 2
      
      - name: Compare + comment PR
        run: |
          python scripts/compare-benchmark.py main-result.json pr-result.json > diff.md
      
      - uses: actions/github-script@v7
        with:
          script: |
            const diff = require('fs').readFileSync('diff.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '## Benchmark\n\n' + diff
            });

  image-build:
    name: Build & Push Image
    runs-on: ubuntu-latest-banking
    needs: [build-test]
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')
    permissions:
      contents: read
      packages: write
      id-token: write   # For OIDC signing
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Generate tags
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-,format=short
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      - name: Build & push
        id: build
        run: |
          ./mvnw -B compile jib:build \
            -Djib.to.image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7} \
            -Djib.to.tags=${{ steps.meta.outputs.tags }} \
            -Djib.to.auth.username=${{ secrets.REGISTRY_USER }} \
            -Djib.to.auth.password=${{ secrets.REGISTRY_PASSWORD }}
          
          DIGEST=$(docker buildx imagetools inspect ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7} --raw | sha256sum | awk '{print $1}')
          echo "digest=sha256:${DIGEST}" >> $GITHUB_OUTPUT
      
      - name: Generate SBOM
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
          syft ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7} -o spdx-json=sbom.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.json
      
      - name: Scan image — Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'   # Fail on findings
      
      - name: Upload Trivy results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3
      
      - name: Sign image
        env:
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
        run: |
          echo "${{ secrets.COSIGN_KEY }}" > cosign.key
          cosign sign --yes --key cosign.key \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
      
      - name: Attest SBOM
        env:
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
        run: |
          cosign attest --yes --key cosign.key \
            --predicate sbom.json --type spdx \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  deploy-staging:
    name: Deploy Staging
    needs: image-build
    runs-on: ubuntu-latest-banking
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.mavibank.com
    
    steps:
      - uses: actions/checkout@v4
        with:
          repository: mavibank/banking-k8s   # GitOps repo
          token: ${{ secrets.GITOPS_TOKEN }}
      
      - name: Update image tag
        run: |
          cd services/transfer-service/overlays/staging
          sed -i "s|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7}|" kustomization.yaml
      
      - name: Commit + push
        run: |
          git config user.email "ci@mavibank.com"
          git config user.name "Banking CI"
          git add .
          git commit -m "deploy: transfer-service to staging sha-${GITHUB_SHA::7}"
          git push
      
      - name: Wait ArgoCD sync
        run: |
          # ArgoCD CLI / API call to verify sync complete
          argocd app sync banking-transfer-service-staging --grpc-web
          argocd app wait banking-transfer-service-staging --timeout 300

  deploy-production:
    name: Deploy Production
    needs: [image-build, deploy-staging]
    runs-on: ubuntu-latest-banking
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
      url: https://api.mavibank.com
    
    steps:
      - name: Wait for approval
        # Environment "production" has required reviewers (CR Committee)
        run: echo "Production deploy approved"
      
      - uses: actions/checkout@v4
        with:
          repository: mavibank/banking-k8s
          token: ${{ secrets.GITOPS_TOKEN }}
      
      - name: Update image tag (canary first)
        run: |
          cd services/transfer-service/overlays/prod-canary
          sed -i "s|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::7}|" kustomization.yaml
      
      - name: Commit + push
        run: |
          git config user.email "ci@mavibank.com"
          git config user.name "Banking CI"
          git add .
          git commit -m "deploy: transfer-service to prod-canary sha-${GITHUB_SHA::7}"
          git push
      
      - name: Wait canary health (15 min)
        run: |
          # Argo Rollouts metric watch
          # If error rate / latency exceeds threshold → auto rollback
          kubectl argo rollouts get rollout transfer-service -n banking --watch --timeout 15m
      
      - name: Promote canary
        run: |
          kubectl argo rollouts promote transfer-service -n banking
```

### 3. Quality gate — SonarQube banking

```properties
# sonar-project.properties
sonar.projectKey=banking-transfer-service
sonar.projectName=Banking Transfer Service
sonar.sources=src/main
sonar.tests=src/test
sonar.java.binaries=target/classes
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Banking quality gate criteria:
sonar.qualitygate.wait=true
```

**Banking quality gate:**
- New code coverage > 80%
- Code smells: 0 blocker, 0 critical
- Bugs: 0
- Vulnerabilities: 0
- Security hotspots: reviewed
- Cognitive complexity per method < 15
- Duplication < 3%

### 4. Jenkins — TR banking yaygın

Many TR banks legacy Jenkins. Jenkinsfile DSL:

```groovy
// Jenkinsfile
@Library('banking-shared-lib') _

pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  serviceAccountName: jenkins-banking
                  containers:
                    - name: maven
                      image: maven:3.9-eclipse-temurin-21
                      command: [sleep]
                      args: [infinity]
                      resources:
                        limits:
                          memory: 4Gi
                          cpu: 4
                    - name: docker
                      image: docker:24-cli
                      command: [sleep]
                      args: [infinity]
                '''
        }
    }
    
    environment {
        REGISTRY = 'registry.mavibank.com'
        IMAGE_NAME = 'banking/transfer-service'
        SONAR_TOKEN = credentials('sonar-token')
        REGISTRY_CREDS = credentials('registry-creds')
    }
    
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn -B clean compile'
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit') {
                    steps {
                        container('maven') {
                            sh 'mvn -B test'
                        }
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration') {
                    steps {
                        container('maven') {
                            sh 'mvn -B verify -Pintegration'
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                container('maven') {
                    withSonarQubeEnv('banking-sonar') {
                        sh 'mvn -B sonar:sonar'
                    }
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Security') {
            parallel {
                stage('SAST') {
                    steps {
                        container('maven') {
                            sh 'mvn -B com.github.spotbugs:spotbugs-maven-plugin:check'
                        }
                    }
                }
                stage('SCA') {
                    steps {
                        container('maven') {
                            sh 'mvn -B org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7'
                        }
                    }
                }
            }
        }
        
        stage('Image') {
            when { branch 'main' }
            steps {
                container('maven') {
                    sh """
                        mvn -B compile jib:build \\
                          -Djib.to.image=${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} \\
                          -Djib.to.auth.username=${REGISTRY_CREDS_USR} \\
                          -Djib.to.auth.password=${REGISTRY_CREDS_PSW}
                    """
                }
                container('docker') {
                    sh "trivy image --severity CRITICAL,HIGH --exit-code 1 ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Deploy Staging') {
            when { branch 'develop' }
            steps {
                bankingDeploy(env: 'staging', service: 'transfer-service', tag: env.BUILD_NUMBER)
            }
        }
        
        stage('Deploy Production') {
            when { branch 'main' }
            input {
                message 'Deploy to production?'
                submitter 'banking-release-managers'
            }
            steps {
                bankingDeploy(env: 'prod-canary', service: 'transfer-service', tag: env.BUILD_NUMBER)
                
                // Wait + promote
                sleep(time: 15, unit: 'MINUTES')
                input {
                    message 'Promote canary to production?'
                    submitter 'banking-release-managers'
                }
                bankingDeploy(env: 'prod', service: 'transfer-service', tag: env.BUILD_NUMBER)
            }
        }
    }
    
    post {
        failure {
            slackSend(
                channel: '#banking-ci',
                color: 'danger',
                message: "Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}
```

### 5. Banking — branch protection

GitHub branch protection rules (`main`):

```
- Require pull request before merging
  - Required approvers: 2 (banking PR review)
  - Dismiss stale approvals on new push
  - Require code owners review
  - Require approval from CODEOWNERS in changed paths
- Require status checks to pass
  - CI Build & Test
  - Integration Tests
  - SonarQube Quality Gate
  - Trivy Image Scan
  - OWASP Dependency Check
  - JMH Benchmark (no regression > 10%)
- Require conversation resolution
- Require signed commits (banking — supply chain)
- Require linear history (no merge commits)
- Require deployments to succeed before merging (staging)
- Lock branch (no direct push, no force push)
- Restrict who can push (banking release engineers)
```

### 6. Secrets management — CI

**Anti-pattern:**
```yaml
env:
  DB_PASSWORD: hardcoded   # ❌
```

**GitHub Secrets** + OIDC for cloud:

```yaml
- name: AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/banking-ci
    aws-region: eu-central-1
    # No long-lived keys — OIDC short-lived token
```

**Vault integration:**

```yaml
- name: Vault secrets
  uses: hashicorp/vault-action@v3
  with:
    url: https://vault.banking.com
    method: jwt
    role: banking-ci
    secrets: |
      secret/data/banking/ci db_password | DB_PASSWORD ;
      secret/data/banking/ci registry_token | REGISTRY_TOKEN
```

### 7. Banking deployment approval gate

**Production deploy requires:**
- Code review approved
- Quality gate passed
- Image signed + scanned
- Staging test passed
- Manual approval (release manager + ops)
- Change management ticket linked (ITSM)
- Maintenance window aligned
- Rollback plan documented

GitHub Actions environment with required reviewers:

```yaml
deploy-production:
  environment:
    name: production
    url: https://api.mavibank.com
  # → GitHub UI: required reviewers + wait timer
```

### 8. Rollback strategy

**Automatic:**
- Argo Rollouts metric analysis → auto rollback
- Error rate spike, latency spike

**Manual:**
- GitOps revert commit
- ArgoCD UI rollback button
- `kubectl rollout undo deployment/transfer-service`

**Banking pratiği:**
- Auto-rollback canary phase
- Manual approval rollback prod (commit revert)
- Audit trail required

### 9. Banking — CI/CD anti-pattern'leri

**Anti-pattern 1: Shared CI runner**
- Noisy neighbor. Banking dedicated runner.

**Anti-pattern 2: Long-lived credentials in CI**
- AWS access key in GitHub Secret → leak risk. OIDC.

**Anti-pattern 3: kubectl apply from CI**
- Drift from git state. ArgoCD/Flux GitOps.

**Anti-pattern 4: No security scan**
- CVE birikir. Trivy + dependency-check mandatory.

**Anti-pattern 5: No signed images**
- Supply chain attack. Cosign + admission policy.

**Anti-pattern 6: Direct push to main**
- Branch protection PR required. No exception.

**Anti-pattern 7: Skip quality gate**
- "Geç teslim, gate sonra" — quality debt. Block merge.

**Anti-pattern 8: Production deploy auto on merge**
- Banking için manual approval gate.

**Anti-pattern 9: No rollback test**
- Production rollback ilk kez gerçek hata sırasında denenir. Pre-prod test.

**Anti-pattern 10: No change management integration**
- Banking ITSM (Jira / ServiceNow) ticket link mandatory. BDDK regulatory.

---

## Önemli olabilecek araştırma kaynakları

- GitHub Actions documentation
- Jenkins official docs
- ArgoCD / Argo Rollouts docs
- "Continuous Delivery" — Jez Humble
- "The DevOps Handbook" — Gene Kim
- OWASP DevSecOps maturity model
- BDDK IT change management

---

## Mini task'ler

### Task 11.4.1 — GitHub Actions build + test (60 dk)

Workflow: PR + push trigger. Maven build + JUnit. Pass/fail status.

### Task 11.4.2 — Integration test with Postgres service (45 dk)

Services postgres + Testcontainers. PR'da çalıştır.

### Task 11.4.3 — SonarQube integration (45 dk)

Local SonarQube + GitHub Actions. Quality gate `wait=true`. Block merge on fail.

### Task 11.4.4 — SAST + SCA (45 dk)

SpotBugs + FindSecBugs. OWASP dependency-check. Fail on CVSS > 7.

### Task 11.4.5 — JMH regression check (60 dk)

PR vs main baseline. > 10% slower = fail. PR comment with diff.

### Task 11.4.6 — Image build + push + Trivy (60 dk)

JIB build → push registry → Trivy scan → fail on HIGH/CRITICAL.

### Task 11.4.7 — Cosign sign + verify (45 dk)

Sign image on CI. K8s admission policy verify.

### Task 11.4.8 — SBOM + attest (30 dk)

Syft generate. Cosign attest as SPDX.

### Task 11.4.9 — GitOps repo update + ArgoCD sync (60 dk)

CI pushes new image tag to k8s repo. ArgoCD auto-sync. Verify deploy.

### Task 11.4.10 — Production approval gate (45 dk)

GitHub environment "production" + required reviewers. Manual approve.

### Task 11.4.11 — Canary rollout (60 dk)

Argo Rollouts. 10% → 50% → 100% with metric analysis. Auto-rollback on threshold.

### Task 11.4.12 — Slack notification (15 dk)

CI fail → Slack #banking-ci.

---

## Test yazma rehberi

```bash
# Local Act (run GitHub Actions locally)
act -W .github/workflows/banking-ci.yml -j build-test

# Pipeline component test
./mvnw -B clean verify -Pintegration
./mvnw -B sonar:sonar -Dsonar.qualitygate.wait=true
trivy image banking/transfer-service:test --severity HIGH,CRITICAL --exit-code 1
cosign verify --key cosign.pub banking/transfer-service:test
```

```java
@Test
@Tag("ci-validate")
void shouldNotIntroduceRegressionInTransferLatency() throws Exception {
    Options opt = new OptionsBuilder()
        .include("transferBenchmark")
        .warmupIterations(3)
        .measurementIterations(5)
        .forks(1)
        .build();
    
    RunResult r = new Runner(opt).run().iterator().next();
    double score = r.getPrimaryResult().getScore();
    
    // Read main baseline from artifact
    double baseline = readBaseline();
    
    assertThat(score)
        .as("Transfer latency regression")
        .isLessThan(baseline * 1.10);   // < 10% regression
}
```

---

## Claude-verify prompt

```
CI/CD pipeline'ımı banking-grade kriterlere göre değerlendir:

1. CI build:
   - Java 21 + Maven + cache?
   - Unit test + integration test (services or Testcontainers)?
   - Code coverage Jacoco?
   - Surefire/Failsafe reports?

2. Quality gate:
   - SonarQube integration?
   - Quality gate wait + fail?
   - New code coverage > 80%?
   - 0 blocker / critical?

3. Security:
   - SAST (SpotBugs + FindSecBugs)?
   - SCA (OWASP dependency-check / Snyk)?
   - DAST (ZAP) post-deploy?
   - Fail on CVSS > 7?

4. Image:
   - JIB / Docker build reproducible?
   - Multi-tag (sha + version)?
   - Trivy scan fail HIGH/CRITICAL?
   - SBOM generate (Syft)?
   - Cosign sign?
   - Cosign attest SBOM?

5. Performance:
   - JMH benchmark PR vs main?
   - Regression > 10% fail?
   - PR comment with diff?

6. Deployment:
   - GitOps (ArgoCD/Flux)?
   - CI pushes to k8s repo (no kubectl apply)?
   - ArgoCD auto-sync?
   - Canary (Argo Rollouts) with metric analysis?
   - Auto-rollback?

7. Approval gates:
   - PR review 2+ approver?
   - CODEOWNERS enforced?
   - Branch protection?
   - Production manual approval (GitHub environment)?
   - Change management ticket link?

8. Secrets:
   - GitHub Secrets / Vault?
   - OIDC short-lived?
   - No long-lived AWS keys?

9. Runner:
   - Dedicated banking runner (no shared)?
   - Benchmark runner (stable env)?

10. Notifications:
    - Slack on fail?
    - Email release announcement?
    - ITSM ticket update?

11. Anti-pattern:
    - Shared CI runner YOK?
    - Long-lived credentials YOK?
    - kubectl apply from CI YOK?
    - No security scan YOK?
    - No image sign YOK?
    - Direct push main YOK?
    - Auto prod deploy YOK?
    - No change management YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] GitHub Actions CI: build + test + Sonar + SAST + SCA
- [ ] JMH benchmark regression PR check
- [ ] Image: JIB + Trivy + SBOM + Cosign
- [ ] GitOps update (k8s repo)
- [ ] ArgoCD auto-sync
- [ ] Canary (Argo Rollouts)
- [ ] Production manual approval
- [ ] Branch protection (signed commits, 2 reviewer, status check)
- [ ] Slack notifications
- [ ] Jenkins alternative pipeline
- [ ] Rollback test (commit revert + Argo auto)
- [ ] Banking ITSM integration documentation

---

## Defter notları (10 madde)

1. "CI/CD pipeline 6 stage (build, test, quality, security, image, deploy) banking flow: ____."
2. "Quality gate SonarQube wait + block merge banking standard: ____."
3. "SAST + SCA + DAST + image scan defense-in-depth: ____."
4. "JMH benchmark PR vs main + regression % gate: ____."
5. "Image build JIB + Trivy + SBOM (Syft) + Cosign sign banking supply chain: ____."
6. "GitOps ArgoCD declarative + CI no kubectl apply + drift detect: ____."
7. "Canary Argo Rollouts + metric analysis + auto-rollback: ____."
8. "Branch protection (2 reviewer + CODEOWNERS + signed commits) banking: ____."
9. "Production approval gate + change management ticket banking BDDK: ____."
10. "Secrets OIDC short-lived + Vault dynamic banking pattern: ____."
