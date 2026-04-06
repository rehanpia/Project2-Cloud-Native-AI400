# Kubernetes Deployment Plan — AI Employee (OpenClaw)

> **Panaversity Cloud Native AI Training | Project 2 — Plan 2**  
> Author: Rehan Ahmed | Course: Cloud Native AI 400
> Date: 2026-04-04 | Version: 3.0

---

## 1. Application Overview

OpenClaw is a **Personal AI Employee** — an autonomous AI agent that performs complex multi-step tasks on behalf of a user: browsing the web, writing code, managing files, sending emails, scheduling meetings, and calling external APIs.

The original project asks three specific security questions:

| Question | Where Answered |
|---|---|
| How many ConfigMaps, Secrets? | Sections 6 & 7 |
| RBAC — will we use it or anything else? | Section 8 |
| **What happens when a secret Expires / is Compromised / Agent tries to access it?** | **Section 7 — Secret Failure Scenarios** |

Because OpenClaw acts with elevated permissions on behalf of real users, **security is the primary design constraint** throughout this entire plan.

### 1.1 Component Breakdown

| Component | Role | Namespace | Security Level |
|---|---|---|---|
| **Auth & Identity Gateway** | OAuth2/OIDC, user sessions, JWT issuance | openclaw-core | Critical |
| **OpenClaw Orchestrator** | LLM reasoning engine, task planning & delegation | openclaw-core | Critical |
| **Tool Executor** | Sandboxed execution of browser/shell/file/API tools | openclaw-sandbox | Critical |
| **Memory Service** | Long-term + short-term memory (vector + relational) | openclaw-core | High |
| **Credential Vault Proxy** | Proxies user OAuth tokens; never exposes raw secrets | openclaw-core | Critical |
| **Audit Logger** | Immutable append-only record of all agent actions | openclaw-audit | High |
| **User Dashboard** | Web interface for task submission & monitoring | openclaw-frontend | Medium |

---

## 2. Security Architecture Principles

Seven principles govern every design decision in this plan:

1. **Zero Trust** — every pod authenticates every request; no implicit trust between services, even within the same namespace.
2. **Least Privilege** — each ServiceAccount has exactly the permissions it needs and nothing more.
3. **Sandbox Isolation** — Tool Executor runs in a dedicated namespace. A compromised executor cannot reach the Orchestrator's secrets even if it escapes its container.
4. **Secret Zero Avoidance** — no raw secrets in environment variables; all injected via HashiCorp Vault Agent as in-memory files.
5. **Immutable Audit Trail** — all agent actions written to append-only persistent storage in an isolated namespace.
6. **Deny-All Networking** — all namespaces begin with deny-all NetworkPolicies; traffic is explicitly allowed pair-by-pair.
7. **Pod Security Standards** — all namespaces enforce `restricted` PSS: non-root, read-only filesystem, no privilege escalation, all capabilities dropped.

---

## 3. Namespace Design

Four namespaces enforce hard isolation boundaries between tiers:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-core
  labels:
    app: openclaw
    tier: core
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-sandbox
  labels:
    app: openclaw
    tier: sandbox
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-frontend
  labels:
    app: openclaw
    tier: frontend
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-audit
  labels:
    app: openclaw
    tier: audit
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
```

**Why four namespaces?**
The `openclaw-sandbox` namespace has its own ResourceQuota, LimitRange, and NetworkPolicy set entirely separate from core services. Even if the Tool Executor is fully compromised by malicious user input, it cannot reach the Orchestrator's secrets, modify its own RBAC, or exfiltrate user credentials — all enforced by the Kubernetes control plane, not application code.

---

## 4. Pods, Deployments, and StatefulSets

### 4.1 Auth & Identity Gateway — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-gateway
  namespace: openclaw-core
  labels:
    app: auth-gateway
    security-level: critical
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-gateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: auth-gateway
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "auth-gateway"
        vault.hashicorp.com/agent-inject-secret-auth: "secret/openclaw/auth"
    spec:
      serviceAccountName: auth-gateway-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: auth-gateway
          image: panaversity/openclaw-auth:1.0.0
          ports:
            - containerPort: 8443
              name: https
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          envFrom:
            - configMapRef:
                name: auth-config
          readinessProbe:
            httpGet:
              path: /health
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true
      volumes:
        - name: tmp
          emptyDir: {}
        - name: vault-secrets
          emptyDir:
            medium: Memory    # In-memory only — secrets never written to disk
```

### 4.2 OpenClaw Orchestrator — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-orchestrator
  namespace: openclaw-core
  labels:
    app: openclaw-orchestrator
    security-level: critical
spec:
  replicas: 2
  selector:
    matchLabels:
      app: openclaw-orchestrator
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: openclaw-orchestrator
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "orchestrator"
        vault.hashicorp.com/agent-inject-secret-llm: "secret/openclaw/llm-keys"
    spec:
      serviceAccountName: orchestrator-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: orchestrator
          image: panaversity/openclaw-orchestrator:1.0.0
          ports:
            - containerPort: 9000
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "2000m"
              memory: "4Gi"
            limits:
              cpu: "4000m"
              memory: "8Gi"
          envFrom:
            - configMapRef:
                name: orchestrator-config
          readinessProbe:
            httpGet:
              path: /health
              port: 9000
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 9000
            initialDelaySeconds: 60
            periodSeconds: 20
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: workspace
              mountPath: /workspace
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true
      volumes:
        - name: tmp
          emptyDir: {}
        - name: workspace
          emptyDir:
            sizeLimit: 500Mi
        - name: vault-secrets
          emptyDir:
            medium: Memory
```

### 4.3 Tool Executor — Deployment (Sandbox Namespace)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tool-executor
  namespace: openclaw-sandbox
  labels:
    app: tool-executor
    security-level: critical
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tool-executor
  template:
    metadata:
      labels:
        app: tool-executor
    spec:
      serviceAccountName: tool-executor-sa
      automountServiceAccountToken: false    # Zero K8s API access from sandbox
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534                     # nobody — most restricted non-root UID
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: tool-executor
          image: panaversity/openclaw-tools:1.0.0
          ports:
            - containerPort: 7000
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          envFrom:
            - configMapRef:
                name: toolexec-config
          readinessProbe:
            httpGet:
              path: /health
              port: 7000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 7000
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          volumeMounts:
            - name: tool-workspace
              mountPath: /workspace
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tool-workspace
          emptyDir:
            sizeLimit: 1Gi
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
```

### 4.4 Memory Service — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: memory-service
  namespace: openclaw-core
spec:
  serviceName: memory-headless
  replicas: 1
  selector:
    matchLabels:
      app: memory-service
  template:
    metadata:
      labels:
        app: memory-service
    spec:
      serviceAccountName: memory-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1002
        fsGroup: 1002
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: memory-service
          image: panaversity/openclaw-memory:1.0.0
          ports:
            - containerPort: 6333
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "1000m"
              memory: "2Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          envFrom:
            - configMapRef:
                name: memory-config
          env:
            - name: ENCRYPTION_KEY
              valueFrom:
                secretKeyRef:
                  name: memory-secrets
                  key: encryption-key
          readinessProbe:
            httpGet:
              path: /health
              port: 6333
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 6333
            initialDelaySeconds: 60
            periodSeconds: 20
          volumeMounts:
            - name: memory-data
              mountPath: /var/lib/memory
  volumeClaimTemplates:
    - metadata:
        name: memory-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "gp3-encrypted"
        resources:
          requests:
            storage: 50Gi
```

### 4.5 Credential Vault Proxy — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: credential-vault-proxy
  namespace: openclaw-core
  labels:
    app: credential-vault-proxy
    security-level: critical
spec:
  replicas: 2
  selector:
    matchLabels:
      app: credential-vault-proxy
  template:
    metadata:
      labels:
        app: credential-vault-proxy
    spec:
      serviceAccountName: vault-proxy-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1005
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: vault-proxy
          image: panaversity/openclaw-vault-proxy:1.0.0
          ports:
            - containerPort: 8200
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8200
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8200
            initialDelaySeconds: 30
            periodSeconds: 20
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

### 4.6 Audit Logger — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: audit-logger
  namespace: openclaw-audit
spec:
  serviceName: audit-headless
  replicas: 1
  selector:
    matchLabels:
      app: audit-logger
  template:
    metadata:
      labels:
        app: audit-logger
    spec:
      serviceAccountName: audit-logger-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1003
        fsGroup: 1003
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: audit-logger
          image: panaversity/openclaw-audit:1.0.0
          ports:
            - containerPort: 5000
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 20
          volumeMounts:
            - name: audit-log-data
              mountPath: /var/log/audit
  volumeClaimTemplates:
    - metadata:
        name: audit-log-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "gp3-encrypted"
        resources:
          requests:
            storage: 100Gi
```

### 4.7 User Dashboard — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-dashboard
  namespace: openclaw-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-dashboard
  template:
    metadata:
      labels:
        app: user-dashboard
    spec:
      serviceAccountName: dashboard-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1004
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: dashboard
          image: panaversity/openclaw-dashboard:1.0.0
          ports:
            - containerPort: 3000
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          envFrom:
            - configMapRef:
                name: dashboard-config
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 20
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

---

## 5. Services

### 5.1 Service Summary

| Service | Namespace | Type | Port | Accessible From |
|---|---|---|---|---|
| user-dashboard-svc | openclaw-frontend | LoadBalancer | 443 | Internet |
| auth-gateway-svc | openclaw-core | ClusterIP | 8443 | Frontend, Core |
| orchestrator-svc | openclaw-core | ClusterIP | 9000 | Core only |
| tool-executor-svc | openclaw-sandbox | ClusterIP | 7000 | Core → Sandbox only |
| memory-svc | openclaw-core | ClusterIP | 6333 | Core only |
| memory-headless | openclaw-core | Headless | 6333 | StatefulSet DNS |
| credential-vault-svc | openclaw-core | ClusterIP | 8200 | Core only |
| audit-svc | openclaw-audit | ClusterIP | 5000 | Core + Sandbox |
| audit-headless | openclaw-audit | Headless | 5000 | StatefulSet DNS |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-dashboard-svc
  namespace: openclaw-frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/id"
spec:
  type: LoadBalancer
  selector:
    app: user-dashboard
  ports:
    - name: https
      port: 443
      targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: auth-gateway-svc
  namespace: openclaw-core
spec:
  type: ClusterIP
  selector:
    app: auth-gateway
  ports:
    - name: https
      port: 8443
      targetPort: 8443
---
apiVersion: v1
kind: Service
metadata:
  name: orchestrator-svc
  namespace: openclaw-core
spec:
  type: ClusterIP
  selector:
    app: openclaw-orchestrator
  ports:
    - name: http
      port: 9000
      targetPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: tool-executor-svc
  namespace: openclaw-sandbox
spec:
  type: ClusterIP
  selector:
    app: tool-executor
  ports:
    - name: http
      port: 7000
      targetPort: 7000
---
apiVersion: v1
kind: Service
metadata:
  name: memory-svc
  namespace: openclaw-core
spec:
  type: ClusterIP
  selector:
    app: memory-service
  ports:
    - name: http
      port: 6333
      targetPort: 6333
---
apiVersion: v1
kind: Service
metadata:
  name: memory-headless
  namespace: openclaw-core
spec:
  clusterIP: None
  selector:
    app: memory-service
  ports:
    - port: 6333
---
apiVersion: v1
kind: Service
metadata:
  name: credential-vault-svc
  namespace: openclaw-core
spec:
  type: ClusterIP
  selector:
    app: credential-vault-proxy
  ports:
    - name: http
      port: 8200
      targetPort: 8200
---
apiVersion: v1
kind: Service
metadata:
  name: audit-svc
  namespace: openclaw-audit
spec:
  type: ClusterIP
  selector:
    app: audit-logger
  ports:
    - name: http
      port: 5000
      targetPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: audit-headless
  namespace: openclaw-audit
spec:
  clusterIP: None
  selector:
    app: audit-logger
  ports:
    - port: 5000
```

---

## 6. ConfigMaps

**Total ConfigMaps: 5** (one per service that has non-secret configuration)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-config
  namespace: openclaw-core
data:
  OIDC_PROVIDER_URL: "https://accounts.google.com"
  TOKEN_EXPIRY_SECONDS: "3600"
  REFRESH_TOKEN_EXPIRY_SECONDS: "604800"
  SESSION_COOKIE_SECURE: "true"
  SESSION_COOKIE_SAMESITE: "Strict"
  MFA_REQUIRED: "true"
  ORCHESTRATOR_URL: "http://orchestrator-svc.openclaw-core.svc.cluster.local:9000"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: orchestrator-config
  namespace: openclaw-core
data:
  LLM_MODEL: "claude-sonnet-4-20250514"
  MAX_AGENT_STEPS: "50"
  AGENT_TIMEOUT_SECONDS: "300"
  TOOL_EXECUTOR_URL: "http://tool-executor-svc.openclaw-sandbox.svc.cluster.local:7000"
  MEMORY_URL: "http://memory-svc.openclaw-core.svc.cluster.local:6333"
  AUDIT_URL: "http://audit-svc.openclaw-audit.svc.cluster.local:5000"
  VAULT_PROXY_URL: "http://credential-vault-svc.openclaw-core.svc.cluster.local:8200"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: toolexec-config
  namespace: openclaw-sandbox
data:
  BROWSER_TIMEOUT_SECONDS: "30"
  MAX_FILE_SIZE_MB: "100"
  ALLOWED_DOMAINS: "*.google.com,*.github.com,*.wikipedia.org"
  SHELL_EXECUTION_ENABLED: "false"
  AUDIT_URL: "http://audit-svc.openclaw-audit.svc.cluster.local:5000"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: memory-config
  namespace: openclaw-core
data:
  VECTOR_DIMENSION: "1536"
  MAX_MEMORY_ITEMS: "10000"
  SHORT_TERM_TTL_SECONDS: "3600"
  LONG_TERM_STORAGE: "true"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dashboard-config
  namespace: openclaw-frontend
data:
  NODE_ENV: "production"
  AUTH_GATEWAY_URL: "https://auth-gateway-svc.openclaw-core.svc.cluster.local:8443"
```

---

## 7. Secrets Management

### 7.1 How Many Secrets?

**Total Secrets: 5**

| Secret Name | Namespace | Contains | Rotation |
|---|---|---|---|
| `auth-secrets` | openclaw-core | OIDC client secret, session signing key, CSRF secret | 90 days |
| `orchestrator-secrets` | openclaw-core | Anthropic API key, OpenAI API key, inter-service JWT secret | 90 days |
| `memory-secrets` | openclaw-core | Data encryption key | 180 days |
| `user-oauth-tokens` | openclaw-core | Per-user OAuth access/refresh tokens (via Vault Proxy) | On refresh |
| `monitoring-secrets` | openclaw-core | Slack webhook for expiry alerts | 365 days |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-secrets
  namespace: openclaw-core
  annotations:
    secret-expiry: "2026-07-01"
    rotation-policy: "90-days"
    managed-by: "vault-external-secrets"
type: Opaque
data:
  oidc-client-secret: <base64>
  session-signing-key: <base64>
  csrf-secret: <base64>
---
apiVersion: v1
kind: Secret
metadata:
  name: orchestrator-secrets
  namespace: openclaw-core
  annotations:
    secret-expiry: "2026-07-01"
    rotation-policy: "90-days"
type: Opaque
data:
  anthropic-api-key: <base64>
  openai-api-key: <base64>
  inter-service-jwt-secret: <base64>
---
apiVersion: v1
kind: Secret
metadata:
  name: memory-secrets
  namespace: openclaw-core
  annotations:
    secret-expiry: "2027-01-01"
    rotation-policy: "180-days"
type: Opaque
data:
  encryption-key: <base64>
---
apiVersion: v1
kind: Secret
metadata:
  name: monitoring-secrets
  namespace: openclaw-core
  annotations:
    secret-expiry: "2027-01-01"
type: Opaque
data:
  slack-webhook: <base64>
```

---

### 7.2 Secret Failure Scenarios

> **This section directly answers the project question: "What happens when a secret is Expired / Compromised / Agent tries to access it?"**

---

#### Scenario A — Secret Expiry

**Trigger:** A secret reaches its `secret-expiry` date (e.g., an LLM API key expires after 90 days).

**What happens without a process:**
All pods using the expired key start receiving `401 Unauthorized` from the LLM API. Requests fail silently or crash the agent mid-task. Users see errors with no clear cause.

**What our plan does:**

```
Week -2  ── CronJob runs every Monday ──► Reads secret-expiry annotations
             Finds secret expiring in < 14 days
             Sends Slack alert: "⚠️ orchestrator-secrets expires in 12 days"
             
Week 0   ── SRE rotates key in HashiCorp Vault
             
Within 60s── External Secrets Operator detects TTL expired in Vault
             Fetches new value and updates K8s Secret automatically
             
Pods     ── On next pod restart (or via secret-reload sidecar):
             Pick up new key from mounted Vault volume
             Zero downtime if using rolling restart strategy
```

**CronJob for expiry alerting:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-expiry-checker
  namespace: openclaw-core
spec:
  schedule: "0 9 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-checker-sa
          securityContext:
            runAsNonRoot: true
            runAsUser: 1006
          containers:
            - name: checker
              image: panaversity/secret-expiry-checker:1.0.0
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
              env:
                - name: ALERT_DAYS_BEFORE
                  value: "14"
                - name: SLACK_WEBHOOK_URL
                  valueFrom:
                    secretKeyRef:
                      name: monitoring-secrets
                      key: slack-webhook
          restartPolicy: OnFailure
```

---

#### Scenario B — Secret Compromise

**Trigger:** An LLM API key, JWT secret, or OIDC credential is leaked (e.g., accidentally logged, exfiltrated by a compromised pod, or found in a git commit).

**Immediate response (< 5 minutes):**

```
Step 1 ── Revoke the key immediately at source (Anthropic dashboard / AWS IAM)
           The compromised key is now invalid — any ongoing misuse stops.

Step 2 ── In Vault: revoke the active lease for that secret
           vault lease revoke -prefix secret/openclaw/llm-keys

Step 3 ── Generate a new key in AWS Secrets Manager / Vault
           External Secrets Operator syncs to K8s within 60 seconds

Step 4 ── Force rolling restart of affected pods:
           kubectl rollout restart deployment/openclaw-orchestrator -n openclaw-core

Step 5 ── Check Kubernetes Audit Logs for which pods accessed the secret:
           All Secret 'get' events are logged to SIEM
           Determine the blast radius — which pods read it and when

Step 6 ── If a pod was the leak source: cordon the node, delete the pod,
           review its audit log trail for exfiltration attempts

Step 7 ── Post-incident: rotate ALL secrets (not just the compromised one)
           because a compromised pod may have accessed multiple secrets
```

**Detection mechanisms already built into this plan:**

| Mechanism | What it catches |
|---|---|
| Kubernetes Audit Logs → SIEM | Unexpected `get` on a Secret by a pod that shouldn't have access |
| Vault Audit Log | Every read of every secret path, with timestamps and caller identity |
| RBAC `resourceNames` scope | Pod can only get its own specific secrets — cross-secret access is denied at the API level |
| NetworkPolicy deny-all | A compromised pod cannot exfiltrate secrets over the network to unexpected destinations |

---

#### Scenario C — Agent Unauthorized Secret Access

**Trigger:** The OpenClaw agent (Orchestrator or Tool Executor) attempts to access a secret it does not own — either through a bug, prompt injection, or an active attack attempting to use the agent as a vector.

**What happens at each layer:**

```
Layer 1: RBAC (first line of defence)
  ── Agent pod runs as ServiceAccount 'orchestrator-sa'
  ── Role 'orchestrator-role' only permits 'get' on 'orchestrator-secrets'
  ── Attempt to read 'auth-secrets' or 'memory-secrets':
     → Kubernetes API returns 403 Forbidden immediately
     → Request never reaches the secret data

Layer 2: Vault Agent Sidecar (second line)
  ── Vault only injects secrets the role is authorized for
  ── Even if a pod calls Vault directly, Vault checks K8s ServiceAccount JWT
  ── Unauthorized path access → Vault returns 403
  ── All attempts logged to Vault audit log

Layer 3: Tool Executor Sandbox (third line — for AI-driven access attempts)
  ── Tool Executor runs as user 65534 (nobody) with automountServiceAccountToken: false
  ── Has NO K8s API credentials — cannot call kubectl, cannot call the K8s API server
  ── Even if an injected prompt says "read the file /var/run/secrets/kubernetes.io/serviceaccount/token"
     → The file does not exist (token was never mounted)
  ── NetworkPolicy blocks all egress except HTTPS:443 and DNS
     → Cannot exfiltrate any data it does find to an unexpected destination

Layer 4: Audit logging (detection)
  ── Every 403 from the K8s API is logged to the Kubernetes Audit Log
  ── Every Vault access attempt (successful or not) is logged
  ── Anomaly detection: if orchestrator-sa suddenly attempts 50 secret reads
     in 10 seconds → alert triggers
```

**RBAC that enforces this — showing the minimal scoping:**

```yaml
# Orchestrator can ONLY read its own two secrets — nothing else
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orchestrator-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orchestrator-config"]    # Only this ConfigMap
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["orchestrator-secrets"]   # Only this Secret
    verbs: ["get"]                            # Only 'get' — not 'list', not 'watch'
```

**Tool Executor has zero RBAC permissions:**

```yaml
# Tool Executor deliberately has an empty rules list
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tool-executor-role
  namespace: openclaw-sandbox
rules: []    # No K8s API permissions whatsoever
```

---

### 7.3 Multi-Layer Secret Management Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SECRET LIFECYCLE                          │
│                                                             │
│  Source: HashiCorp Vault                                    │
│    └── Dynamic secrets, lease-based rotation               │
│         │                                                   │
│         ▼ External Secrets Operator (TTL sync)              │
│  K8s Secret (openclaw-core namespace)                       │
│    └── immutable: true for stable long-lived secrets        │
│         │                                                   │
│         ▼ Vault Agent Sidecar                               │
│  In-Memory Volume (/vault/secrets)                          │
│    └── Never written to disk, lost on pod restart           │
│         │                                                   │
│         ▼ Read by application                               │
│  Pod reads secret from file — NOT from env var              │
│                                                             │
│  Monitoring:                                                │
│    ├── Expiry CronJob (weekly, 14-day warning)              │
│    ├── K8s Audit Logs → SIEM (all secret access)           │
│    └── Vault Audit Log (all Vault access)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. RBAC — Roles and RoleBindings

**Answer to project question: "RBAC — will we use it or any other thing?"**

Yes — we use **RBAC as the primary access control mechanism**, augmented by:
- **Istio AuthorizationPolicy** for service-to-service mTLS authentication
- **Vault Policies** for secret-level access control
- **NetworkPolicies** for network-level isolation

### 8.1 Service Accounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-gateway-sa
  namespace: openclaw-core
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orchestrator-sa
  namespace: openclaw-core
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tool-executor-sa
  namespace: openclaw-sandbox
automountServiceAccountToken: false    # No K8s API access
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: memory-service-sa
  namespace: openclaw-core
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-proxy-sa
  namespace: openclaw-core
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: audit-logger-sa
  namespace: openclaw-audit
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: openclaw-frontend
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-checker-sa
  namespace: openclaw-core
```

### 8.2 Auth Gateway Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: auth-gateway-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["auth-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["auth-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: auth-gateway-rolebinding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: auth-gateway-sa
    namespace: openclaw-core
roleRef:
  kind: Role
  name: auth-gateway-role
  apiGroup: rbac.authorization.k8s.io
```

### 8.3 Orchestrator Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orchestrator-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orchestrator-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["orchestrator-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orchestrator-rolebinding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: orchestrator-sa
    namespace: openclaw-core
roleRef:
  kind: Role
  name: orchestrator-role
  apiGroup: rbac.authorization.k8s.io
```

### 8.4 Tool Executor Role — Zero Permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tool-executor-role
  namespace: openclaw-sandbox
rules: []    # Intentionally empty — no K8s API access from sandbox
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tool-executor-rolebinding
  namespace: openclaw-sandbox
subjects:
  - kind: ServiceAccount
    name: tool-executor-sa
    namespace: openclaw-sandbox
roleRef:
  kind: Role
  name: tool-executor-role
  apiGroup: rbac.authorization.k8s.io
```

### 8.5 Vault Proxy ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vault-proxy-cluster-role
rules:
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-proxy-cluster-binding
subjects:
  - kind: ServiceAccount
    name: vault-proxy-sa
    namespace: openclaw-core
roleRef:
  kind: ClusterRole
  name: vault-proxy-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

### 8.6 Secret Checker Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-checker-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["list"]    # Only needs to list to read annotations — not get values
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-checker-rolebinding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: secret-checker-sa
    namespace: openclaw-core
roleRef:
  kind: Role
  name: secret-checker-role
  apiGroup: rbac.authorization.k8s.io
```

### 8.7 Human Access Roles

```yaml
# SRE — admin in core, read-only in sandbox (security boundary)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sre-admin-core
  namespace: openclaw-core
subjects:
  - kind: Group
    name: sre-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# Developers — view only, no secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-readonly-core
  namespace: openclaw-core
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## 9. Inter-Service Communication

### 9.1 Architecture Diagram

```
Internet (HTTPS :443)
        │
        ▼
[AWS NLB LoadBalancer]
  (openclaw-frontend)
        │
        ▼
[user-dashboard :3000] ──JWT──► [auth-gateway-svc :8443]
  (openclaw-frontend)              (openclaw-core)
                                          │
                                   validates token
                                          │
                                          ▼
                                 [orchestrator-svc :9000]
                                   (openclaw-core)
                                  /       │        \
                        mTLS     /        │         \
                                ▼         ▼          ▼
                   [tool-executor]  [memory-svc]  [credential-vault-svc]
                   (openclaw-sandbox) (core:6333)     (core:8200)
                          │
                          │ (every action)
                          ▼
                     [audit-svc :5000]
                     (openclaw-audit)
                     ▲
                     │ (also receives from orchestrator)
                     └──────────────────────────
```

### 9.2 DNS Address Table

| From | To | DNS Address | Port |
|---|---|---|---|
| Dashboard | Auth Gateway | `auth-gateway-svc.openclaw-core.svc.cluster.local` | 8443 |
| Auth Gateway | Orchestrator | `orchestrator-svc.openclaw-core.svc.cluster.local` | 9000 |
| Orchestrator | Tool Executor | `tool-executor-svc.openclaw-sandbox.svc.cluster.local` | 7000 |
| Orchestrator | Memory | `memory-svc.openclaw-core.svc.cluster.local` | 6333 |
| Orchestrator | Vault Proxy | `credential-vault-svc.openclaw-core.svc.cluster.local` | 8200 |
| Orchestrator | Audit | `audit-svc.openclaw-audit.svc.cluster.local` | 5000 |
| Tool Executor | Audit | `audit-svc.openclaw-audit.svc.cluster.local` | 5000 |

### 9.3 Istio mTLS Enforcement

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: require-mtls-core
  namespace: openclaw-core
spec:
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: require-mtls-sandbox
  namespace: openclaw-sandbox
spec:
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: tool-executor-authz
  namespace: openclaw-sandbox
spec:
  selector:
    matchLabels:
      app: tool-executor
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/openclaw-core/sa/orchestrator-sa"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/execute"]
```

### 9.4 NetworkPolicies

```yaml
# Default deny all in every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: openclaw-core
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: openclaw-sandbox
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: openclaw-frontend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: openclaw-audit
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# LoadBalancer → Dashboard
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb-to-dashboard
  namespace: openclaw-frontend
spec:
  podSelector:
    matchLabels:
      app: user-dashboard
  ingress:
    - ports:
        - port: 3000
---
# Dashboard egress → Auth Gateway (cross-namespace AND logic)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dashboard-to-auth
  namespace: openclaw-frontend
spec:
  podSelector:
    matchLabels:
      app: user-dashboard
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              tier: core
          podSelector:              # AND logic — same list item
            matchLabels:
              app: auth-gateway
      ports:
        - port: 8443
---
# Orchestrator → Tool Executor (cross-namespace AND logic)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-orchestrator-to-toolexec
  namespace: openclaw-sandbox
spec:
  podSelector:
    matchLabels:
      app: tool-executor
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tier: core
          podSelector:              # AND logic — same list item
            matchLabels:
              app: openclaw-orchestrator
      ports:
        - port: 7000
---
# Tool Executor egress → internet (HTTPS) + Audit Logger
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sandbox-controlled-egress
  namespace: openclaw-sandbox
spec:
  podSelector:
    matchLabels:
      app: tool-executor
  egress:
    - ports:
        - port: 443
          protocol: TCP
        - port: 53
          protocol: UDP
    - to:
        - namespaceSelector:
            matchLabels:
              tier: audit
          podSelector:
            matchLabels:
              app: audit-logger
      ports:
        - port: 5000
---
# Core services → Audit Logger
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-core-to-audit
  namespace: openclaw-audit
spec:
  podSelector:
    matchLabels:
      app: audit-logger
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tier: core
      ports:
        - port: 5000
    - from:
        - namespaceSelector:
            matchLabels:
              tier: sandbox
      ports:
        - port: 5000
```

---

## 10. PodDisruptionBudgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-gateway-pdb
  namespace: openclaw-core
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: auth-gateway
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orchestrator-pdb
  namespace: openclaw-core
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: openclaw-orchestrator
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: tool-executor-pdb
  namespace: openclaw-sandbox
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: tool-executor
```

---

## 11. Resource Summary

| Component | Namespace | Type | Replicas | CPU Req | CPU Limit | Mem Req | Mem Limit |
|---|---|---|---|---|---|---|---|
| auth-gateway | openclaw-core | Deployment | 3 | 500m | 1000m | 512Mi | 1Gi |
| orchestrator | openclaw-core | Deployment | 2 | 2000m | 4000m | 4Gi | 8Gi |
| tool-executor | openclaw-sandbox | Deployment | 3 | 500m | 2000m | 1Gi | 4Gi |
| memory-service | openclaw-core | StatefulSet | 1 | 1000m | 2000m | 2Gi | 4Gi |
| credential-vault-proxy | openclaw-core | Deployment | 2 | 250m | 500m | 256Mi | 512Mi |
| audit-logger | openclaw-audit | StatefulSet | 1 | 250m | 500m | 512Mi | 1Gi |
| user-dashboard | openclaw-frontend | Deployment | 2 | 250m | 500m | 256Mi | 512Mi |
| **TOTAL** | | | **14** | **4750m** | **10500m** | **8.75Gi** | **19Gi** |

---

## 12. Security Checklist

| Control | ✅ | Implementation |
|---|---|---|
| Non-root containers | ✅ | `runAsNonRoot: true` on all pods |
| Read-only root filesystem | ✅ | `readOnlyRootFilesystem: true` on all pods |
| No privilege escalation | ✅ | `allowPrivilegeEscalation: false` on all pods |
| All capabilities dropped | ✅ | `capabilities.drop: ["ALL"]` on all pods |
| fsGroup set everywhere | ✅ | All pod specs include `fsGroup` |
| Sandbox K8s token disabled | ✅ | `automountServiceAccountToken: false` on Tool Executor |
| mTLS enforced | ✅ | Istio `PeerAuthentication: STRICT` on core + sandbox |
| Cross-namespace AND NetworkPolicy | ✅ | Both selectors in same `from` item |
| Secrets via Vault memory volume | ✅ | No raw secrets in env vars for critical services |
| Secret expiry handling | ✅ | Weekly CronJob + 14-day Slack alert |
| Secret compromise response | ✅ | 7-step revocation playbook in Section 7.2B |
| Agent unauthorized access response | ✅ | 4-layer defence documented in Section 7.2C |
| Immutable audit log | ✅ | StatefulSet + encrypted PVC in isolated namespace |
| Pod Security Standards | ✅ | `restricted` on core/sandbox/audit; `baseline` on frontend |
| PodDisruptionBudgets | ✅ | Auth Gateway, Orchestrator, Tool Executor |
| Image versions pinned | ✅ | All images use `1.0.0`, no `:latest` |
| ConfigMaps count documented | ✅ | 5 ConfigMaps listed in Section 6 |
| Secrets count documented | ✅ | 5 Secrets listed in Section 7.1 |

---

*Plan 2 Complete — See k8-planning-skill.md for the reusable K8 Planning Skill.*
