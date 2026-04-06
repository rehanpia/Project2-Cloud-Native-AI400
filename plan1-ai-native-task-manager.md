# Kubernetes Deployment Plan — AI Native Task Manager

> **Panaversity Cloud Native AI Training | Project 2 — Plan 1**  
> Author: Panaversity Student Rehan Ahmed | Course: Cloud Native AI 400 
> Date: 2026-04-04 | Version: 3.0

---

## 1. Application Overview

The AI Native Task Manager is a cloud-native, multi-service application. The original project defines **four explicit communication rules** that govern how services interact. Every Service definition, NetworkPolicy, and ConfigMap URL in this plan is derived directly from those four rules.
)

| # | Rule | Impact on Architecture |
|---|---|---|
| **1** | UI Interface connects with Backend APIs to manage tasks | UI → Backend: bidirectional HTTP |
| **2** | UI Interface connects **directly** with Todo Agent to manage tasks | UI → Todo Agent: direct HTTP (not only via Backend) |
| **3** | Todo Agent connects with Backend API to manage tasks | Todo Agent → Backend: HTTP |
| **4** | Notification Service connects with **both** UI Interface **and** Backend API | Notification ↔ UI + Notification ↔ Backend |

### 1.2 Component Summary

| Component | Role | Workload Type | Replicas (prod) |
|---|---|---|---|
| **UI Interface** | React/Next.js frontend; calls Backend AND Todo Agent directly | Deployment | 3 |
| **Backend APIs** | REST/GraphQL business logic; receives calls from UI and Todo Agent | Deployment | 4 |
| **Todo Agent** | AI-powered agent; called by UI directly AND calls Backend | Deployment | 2 |
| **Notification Service** | Alerts via push/email/SMS; connects to both UI and Backend | Deployment | 2 |
| **PostgreSQL** | Relational database; accessed by Backend only | StatefulSet | 1 |

---

## 2. Namespace Design

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager-prod
  labels:
    env: production
    app: task-manager
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager-dev
  labels:
    env: development
    app: task-manager
    pod-security.kubernetes.io/enforce: baseline
---
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager-monitoring
  labels:
    purpose: observability
    app: task-manager
```

**Rationale:**
- `task-manager-prod` hosts all live workloads with the `baseline` Pod Security Standard, which prevents privilege escalation and host-path mounts.
- `task-manager-dev` mirrors prod but is fully isolated — a misconfigured dev pod cannot affect production.
- `task-manager-monitoring` hosts Prometheus and Grafana so monitoring RBAC does not bleed into application namespaces.

---

## 3. Pods and Deployments / StatefulSets

### 3.1 UI Interface — Deployment

The UI calls both the Backend API (Rule 1) and the Todo Agent directly (Rule 2). It also receives callbacks from the Notification Service (Rule 4). The ConfigMap reflects all three outbound URLs.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-interface
  namespace: task-manager-prod
  labels:
    app: ui-interface
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ui-interface
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0        # Zero-downtime rolling deploys
  template:
    metadata:
      labels:
        app: ui-interface
        tier: frontend
    spec:
      serviceAccountName: ui-interface-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: ui-interface
          image: panaversity/task-manager-ui:1.0.0
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
                name: ui-config    # Contains URLs for Backend AND Todo Agent (Rules 1 & 2)
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 10
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: next-cache
              mountPath: /app/.next/cache
      volumes:
        - name: tmp
          emptyDir: {}
        - name: next-cache
          emptyDir: {}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ui-interface
```

### 3.2 Backend APIs — Deployment

Receives calls from UI (Rule 1) and Todo Agent (Rule 3). Calls Notification Service (Rule 4) and PostgreSQL.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: task-manager-prod
  labels:
    app: backend-api
    tier: backend
spec:
  replicas: 4
  selector:
    matchLabels:
      app: backend-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: backend-api
        tier: backend
    spec:
      serviceAccountName: backend-api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend-api
          image: panaversity/task-manager-api:1.0.0
          ports:
            - containerPort: 8000
              name: http
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
                name: backend-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: backend-secrets
                  key: jwt-secret
          readinessProbe:
            httpGet:
              path: /api/health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8000
            initialDelaySeconds: 45
            periodSeconds: 20
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /api/health
              port: 8000
            failureThreshold: 12
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: backend-api
```

### 3.3 Todo Agent — Deployment

Called directly by UI (Rule 2) AND calls Backend API (Rule 3). Both connections are modelled explicitly in Services and NetworkPolicies.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-agent
  namespace: task-manager-prod
  labels:
    app: todo-agent
    tier: agent
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-agent
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: todo-agent
        tier: agent
    spec:
      serviceAccountName: todo-agent-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1002
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: todo-agent
          image: panaversity/todo-agent:1.0.0
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
              cpu: "1000m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          envFrom:
            - configMapRef:
                name: agent-config    # Contains Backend API URL (Rule 3)
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: agent-secrets
                  key: openai-api-key
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: agent-secrets
                  key: anthropic-api-key
          readinessProbe:
            httpGet:
              path: /agent/health
              port: 9000
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /agent/health
              port: 9000
            initialDelaySeconds: 60
            periodSeconds: 20
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /agent/health
              port: 9000
            failureThreshold: 15
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

### 3.4 Notification Service — Deployment

Connects to **both UI Interface and Backend API** (Rule 4). This is bidirectional — it receives triggers from Backend and can push events back to UI.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: task-manager-prod
  labels:
    app: notification-service
    tier: messaging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: notification-service
        tier: messaging
    spec:
      serviceAccountName: notification-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1003
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: notification-service
          image: panaversity/notification-service:1.0.0
          ports:
            - containerPort: 8080
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
                name: notification-config    # Contains URLs for UI and Backend (Rule 4)
          env:
            - name: SMTP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: notification-secrets
                  key: smtp-password
            - name: TWILIO_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: notification-secrets
                  key: twilio-auth-token
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

### 3.5 PostgreSQL — StatefulSet

Accessed only by Backend API. Uses `pg_isready` exec probe — not HTTP — to correctly verify database connectivity.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: task-manager-prod
  labels:
    app: postgres
    tier: database
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      serviceAccountName: postgres-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
              name: postgres
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1000m"
              memory: "2Gi"
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: backend-config
                  key: db-name
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: backend-config
                  key: db-user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: postgres-password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "$(POSTGRES_USER)", "-d", "$(POSTGRES_DB)"]
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "$(POSTGRES_USER)", "-d", "$(POSTGRES_DB)"]
            initialDelaySeconds: 45
            periodSeconds: 20
            failureThreshold: 3
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "gp3"
        resources:
          requests:
            storage: 20Gi
```

---

## 4. Services

### 4.1 Service Types — Mapped to Communication Rules

| Service | Type | Port | Callers | Rule |
|---|---|---|---|---|
| ui-interface-svc | **LoadBalancer** | 80/443 | Internet users | External access |
| backend-api-svc | **ClusterIP** | 8000 | UI, Todo Agent | Rules 1, 3 |
| todo-agent-svc | **ClusterIP** | 9000 | UI Interface | Rule 2 |
| notification-svc | **ClusterIP** | 8080 | UI, Backend | Rule 4 |
| postgres-svc | **ClusterIP** | 5432 | Backend only | Internal |
| postgres-headless | **Headless** | 5432 | StatefulSet DNS | Internal |

```yaml
# UI Interface — LoadBalancer (internet-facing)
apiVersion: v1
kind: Service
metadata:
  name: ui-interface-svc
  namespace: task-manager-prod
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: ui-interface
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: https
      port: 443
      targetPort: 3000
---
# Backend API — ClusterIP (called by UI Rule 1, and Todo Agent Rule 3)
apiVersion: v1
kind: Service
metadata:
  name: backend-api-svc
  namespace: task-manager-prod
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
    - name: http
      port: 8000
      targetPort: 8000
---
# Todo Agent — ClusterIP (called directly by UI, Rule 2)
apiVersion: v1
kind: Service
metadata:
  name: todo-agent-svc
  namespace: task-manager-prod
spec:
  type: ClusterIP
  selector:
    app: todo-agent
  ports:
    - name: http
      port: 9000
      targetPort: 9000
---
# Notification Service — ClusterIP (called by both UI and Backend, Rule 4)
apiVersion: v1
kind: Service
metadata:
  name: notification-svc
  namespace: task-manager-prod
spec:
  type: ClusterIP
  selector:
    app: notification-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
# PostgreSQL — ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: task-manager-prod
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
---
# PostgreSQL — Headless (for StatefulSet stable DNS)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: task-manager-prod
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

---

## 5. ConfigMaps

ConfigMaps embed all internal DNS service URLs. The URLs are **not secrets** — they are internal cluster addresses. Each ConfigMap reflects exactly the communication rules the service participates in.

```yaml
# UI Interface — needs Backend URL (Rule 1) AND Todo Agent URL (Rule 2)
apiVersion: v1
kind: ConfigMap
metadata:
  name: ui-config
  namespace: task-manager-prod
data:
  NODE_ENV: "production"
  NEXT_PUBLIC_APP_NAME: "AI Task Manager"
  API_TIMEOUT_MS: "30000"
  # Rule 1: UI → Backend
  BACKEND_API_URL: "http://backend-api-svc.task-manager-prod.svc.cluster.local:8000"
  # Rule 2: UI → Todo Agent (direct connection)
  TODO_AGENT_URL: "http://todo-agent-svc.task-manager-prod.svc.cluster.local:9000"
  # Rule 4: Notification can push events back to UI
  NOTIFICATION_URL: "http://notification-svc.task-manager-prod.svc.cluster.local:8080"
---
# Backend API — needs Notification URL (Rule 4) and DB connection
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: task-manager-prod
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  db-name: "taskmanager"
  db-user: "taskapp"
  DB_HOST: "postgres-svc.task-manager-prod.svc.cluster.local"
  DB_PORT: "5432"
  DB_MAX_CONNECTIONS: "100"
  # Rule 4: Backend → Notification
  NOTIFICATION_URL: "http://notification-svc.task-manager-prod.svc.cluster.local:8080"
---
# Todo Agent — needs Backend URL (Rule 3)
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-config
  namespace: task-manager-prod
data:
  AGENT_MODEL: "claude-sonnet-4-20250514"
  MAX_TOKENS: "4096"
  AGENT_TIMEOUT_SECONDS: "120"
  MAX_RETRIES: "3"
  # Rule 3: Todo Agent → Backend API
  BACKEND_API_URL: "http://backend-api-svc.task-manager-prod.svc.cluster.local:8000"
---
# Notification Service — needs URLs for both UI and Backend (Rule 4)
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-config
  namespace: task-manager-prod
data:
  SMTP_HOST: "smtp.gmail.com"
  SMTP_PORT: "587"
  SMTP_SECURE: "true"
  SENDER_EMAIL: "noreply@taskmanager.ai"
  MAX_RETRY_ATTEMPTS: "3"
  # Rule 4: Notification → UI Interface (push-back events)
  UI_CALLBACK_URL: "http://ui-interface-svc.task-manager-prod.svc.cluster.local:3000"
  # Rule 4: Notification → Backend API
  BACKEND_API_URL: "http://backend-api-svc.task-manager-prod.svc.cluster.local:8000"
```

---

## 6. Secrets Management

### 6.1 Secret Definitions

> **Production Rule:** All `<base64-encoded>` values are placeholders. Use External Secrets Operator syncing from AWS Secrets Manager or HashiCorp Vault. Never commit real values to Git.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: task-manager-prod
  annotations:
    secret-expiry: "2027-01-01"
    rotation-policy: "90-days"
    managed-by: "external-secrets-operator"
type: Opaque
data:
  postgres-password: <base64-encoded>
  database-url: <base64-encoded>
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
  namespace: task-manager-prod
  annotations:
    secret-expiry: "2026-10-01"
    rotation-policy: "90-days"
type: Opaque
data:
  jwt-secret: <base64-encoded>
---
apiVersion: v1
kind: Secret
metadata:
  name: agent-secrets
  namespace: task-manager-prod
  annotations:
    secret-expiry: "2026-07-01"
    rotation-policy: "90-days"
type: Opaque
data:
  openai-api-key: <base64-encoded>
  anthropic-api-key: <base64-encoded>
---
apiVersion: v1
kind: Secret
metadata:
  name: notification-secrets
  namespace: task-manager-prod
  annotations:
    secret-expiry: "2027-01-01"
    rotation-policy: "180-days"
type: Opaque
data:
  smtp-password: <base64-encoded>
  twilio-auth-token: <base64-encoded>
```

### 6.2 Secret Rotation Strategy

| Layer | Tool | Policy |
|---|---|---|
| **Source of truth** | AWS Secrets Manager / HashiCorp Vault | Master store; auto-rotation at source |
| **K8s sync** | External Secrets Operator | Polls TTL; auto-updates K8s Secret within 1 min |
| **Expiry alerting** | Weekly CronJob | Reads `secret-expiry` annotation; Slack alert 14 days before |
| **Immutability** | `immutable: true` on long-lived secrets | Prevents accidental in-cluster edits |
| **Audit trail** | Kubernetes Audit Logs → SIEM | All Secret `get`/`list` events recorded |

---

## 7. RBAC — Roles and RoleBindings

Every workload gets a dedicated `ServiceAccount`. Roles grant access only to the specific ConfigMap and Secret each service requires by name.

### 7.1 UI Interface

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ui-interface-sa
  namespace: task-manager-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ui-interface-role
  namespace: task-manager-prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["ui-config"]
    verbs: ["get"]
  # No secret access — UI reads no secrets directly
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ui-interface-rolebinding
  namespace: task-manager-prod
subjects:
  - kind: ServiceAccount
    name: ui-interface-sa
    namespace: task-manager-prod
roleRef:
  kind: Role
  name: ui-interface-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.2 Backend API

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-api-sa
  namespace: task-manager-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-api-role
  namespace: task-manager-prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["backend-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-secrets", "backend-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-api-rolebinding
  namespace: task-manager-prod
subjects:
  - kind: ServiceAccount
    name: backend-api-sa
    namespace: task-manager-prod
roleRef:
  kind: Role
  name: backend-api-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.3 Todo Agent

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: todo-agent-sa
  namespace: task-manager-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: todo-agent-role
  namespace: task-manager-prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["agent-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["agent-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: todo-agent-rolebinding
  namespace: task-manager-prod
subjects:
  - kind: ServiceAccount
    name: todo-agent-sa
    namespace: task-manager-prod
roleRef:
  kind: Role
  name: todo-agent-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.4 Notification Service

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-sa
  namespace: task-manager-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: notification-role
  namespace: task-manager-prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["notification-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["notification-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: notification-rolebinding
  namespace: task-manager-prod
subjects:
  - kind: ServiceAccount
    name: notification-sa
    namespace: task-manager-prod
roleRef:
  kind: Role
  name: notification-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.5 PostgreSQL

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: postgres-sa
  namespace: task-manager-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: postgres-role
  namespace: task-manager-prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["backend-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: postgres-rolebinding
  namespace: task-manager-prod
subjects:
  - kind: ServiceAccount
    name: postgres-sa
    namespace: task-manager-prod
roleRef:
  kind: Role
  name: postgres-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.6 Developer Read-Only (Human Access)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: task-manager-developer
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "events", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch"]
  # Secrets deliberately excluded — developers cannot read secrets in prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-readonly
  namespace: task-manager-prod
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: task-manager-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## 8. Inter-Service Communication

### 8.1 Communication Flow — Matching All 4 Rules

```
                        INTERNET
                           │
                    HTTPS :443 / HTTP :80
                           │
                    ┌──────▼───────┐
                    │ LoadBalancer │
                    └──────┬───────┘
                           │ :3000
                    ┌──────▼───────────────┐
                    │    UI Interface      │ ◄─────────────────┐
                    │      (3 pods)        │                   │
                    └──┬───────────────┬───┘                   │
                       │               │                       │
              Rule 1   │ :8000         │ :9000  Rule 2         │
                       │               │                       │
              ┌────────▼────┐   ┌──────▼───────┐              │
              │ Backend API │   │  Todo Agent  │              │
              │  (4 pods)   │◄──│   (2 pods)   │              │ Rule 4
              └──┬───┬───┬──┘   └──────────────┘              │ (push-back)
           Rule 3│   │   │          Rule 3 ↑                  │
          :8080  │ :5432│           (agent→backend)            │
                 │   │   │                              ┌──────┴──────────┐
          ┌──────▼┐  │   │                              │ Notification    │
          │ Notif.│  │   │          ◄───────────────────│   Service       │
          │  Svc  │  │   │            Rule 4 (bidirect) │   (2 pods)      │
          └───────┘  │   │                              └─────────────────┘
                     │   │
              ┌──────▼┐  └── :5432 ──► [PostgreSQL StatefulSet]
              │ SMTP/ │
              │Twilio │
              └───────┘
```

### 8.2 DNS Address Table

| From | To | DNS Address | Port | Rule |
|---|---|---|---|---|
| UI Interface | Backend API | `backend-api-svc.task-manager-prod.svc.cluster.local` | 8000 | Rule 1 |
| UI Interface | Todo Agent | `todo-agent-svc.task-manager-prod.svc.cluster.local` | 9000 | Rule 2 |
| UI Interface | Notification | `notification-svc.task-manager-prod.svc.cluster.local` | 8080 | Rule 4 |
| Todo Agent | Backend API | `backend-api-svc.task-manager-prod.svc.cluster.local` | 8000 | Rule 3 |
| Backend API | Notification | `notification-svc.task-manager-prod.svc.cluster.local` | 8080 | Rule 4 |
| Notification | UI Interface | `ui-interface-svc.task-manager-prod.svc.cluster.local` | 3000 | Rule 4 |
| Backend API | PostgreSQL | `postgres-svc.task-manager-prod.svc.cluster.local` | 5432 | Internal |

### 8.3 NetworkPolicies — One Policy Per Communication Rule

```yaml
# ─── BASELINE: Deny everything in namespace ─────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: task-manager-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# ─── RULE 1: UI Interface ↔ Backend API ─────────────────────────────────────
# UI can call Backend (Rule 1); Backend can respond.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule1-ui-to-backend
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: backend-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ui-interface
      ports:
        - port: 8000

---
# ─── RULE 2: UI Interface → Todo Agent (direct connection) ──────────────────
# UI calls Todo Agent directly — NOT only through Backend.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule2-ui-to-todo-agent
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: todo-agent
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ui-interface
      ports:
        - port: 9000

---
# ─── RULE 3: Todo Agent → Backend API ───────────────────────────────────────
# Todo Agent calls Backend to create/update tasks.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule3-todo-agent-to-backend
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: backend-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: todo-agent
      ports:
        - port: 8000

---
# ─── RULE 4a: Backend API → Notification Service ────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule4a-backend-to-notification
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: notification-service
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend-api
      ports:
        - port: 8080

---
# ─── RULE 4b: UI Interface → Notification Service ───────────────────────────
# UI can also call Notification directly (Rule 4 is bidirectional).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule4b-ui-to-notification
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: notification-service
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ui-interface
      ports:
        - port: 8080

---
# ─── RULE 4c: Notification Service → UI Interface (push-back) ───────────────
# Notification pushes real-time events back to UI (Rule 4 bidirectionality).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule4c-notification-to-ui
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: ui-interface
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: notification-service
      ports:
        - port: 3000

---
# ─── DATABASE: Backend → PostgreSQL ─────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend-api
      ports:
        - port: 5432

---
# ─── EGRESS: Todo Agent → External LLM APIs ─────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-agent-external-egress
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: todo-agent
  egress:
    - ports:
        - port: 443
          protocol: TCP
        - port: 53
          protocol: UDP     # DNS resolution for external API calls

---
# ─── EGRESS: Notification → External (SMTP / Twilio) ───────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-notification-external-egress
  namespace: task-manager-prod
spec:
  podSelector:
    matchLabels:
      app: notification-service
  egress:
    - ports:
        - port: 587
          protocol: TCP
        - port: 443
          protocol: TCP
        - port: 53
          protocol: UDP
```

---

## 9. HorizontalPodAutoscalers

```yaml
# Backend API HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: task-manager-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 4
  maxReplicas: 12
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
---
# Todo Agent HPA — lower threshold because LLM calls are bursty
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-agent-hpa
  namespace: task-manager-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-agent
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
---
# UI Interface HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ui-interface-hpa
  namespace: task-manager-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ui-interface
  minReplicas: 3
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 10. PodDisruptionBudgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-api-pdb
  namespace: task-manager-prod
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: backend-api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ui-interface-pdb
  namespace: task-manager-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: ui-interface
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: todo-agent-pdb
  namespace: task-manager-prod
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: todo-agent
```

---

## 11. Resource Summary

| Component | Type | Replicas | CPU Request | CPU Limit | Mem Request | Mem Limit |
|---|---|---|---|---|---|---|
| ui-interface | Deployment | 3 | 250m | 500m | 256Mi | 512Mi |
| backend-api | Deployment | 4 | 500m | 1000m | 512Mi | 1Gi |
| todo-agent | Deployment | 2 | 1000m | 2000m | 1Gi | 2Gi |
| notification-service | Deployment | 2 | 250m | 500m | 256Mi | 512Mi |
| postgres | StatefulSet | 1 | 500m | 1000m | 1Gi | 2Gi |
| **TOTAL** | | **12** | **2500m** | **5000m** | **3Gi** | **6Gi** |

---

## 12. Final Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Namespace: task-manager-prod                           │
│                                                                          │
│  [Internet] ──LB:80/443──► ┌─────────────────────┐                      │
│                             │   UI Interface x3   │◄── Rule 4c (Notif)  │
│                             └──┬──────────────┬───┘                      │
│                    Rule 1 :8000│     Rule 2   │:9000                     │
│                                │              │                          │
│                    ┌───────────▼──┐   ┌───────▼──────┐                  │
│                    │ Backend API  │◄──│  Todo Agent  │ Rule 3           │
│                    │   x4, HPA   │   │   x2, HPA    │──► [LLM APIs]    │
│                    └──┬───┬───┬───┘   └──────────────┘                  │
│                       │   │   │                                          │
│               Rule 4a │ :5432│  :8080                                   │
│                       │   │   │                                          │
│          ┌────────────▼┐  │   └──────────────► ┌──────────────────┐    │
│          │  Notif. Svc │  │       Rule 4a       │  notification    │    │
│          │   x2 pods   │  │                     │  (same service)  │    │
│          └─────────────┘  │                     └──────────────────┘    │
│          Rule 4b: UI also─┘                                              │
│          calls Notification                                              │
│                            └──:5432──► [PostgreSQL StatefulSet]          │
└──────────────────────────────────────────────────────────────────────────┘

Rules verified: ✅ Rule 1 (UI↔Backend)  ✅ Rule 2 (UI→Agent direct)
                ✅ Rule 3 (Agent→Backend)  ✅ Rule 4 (Notif↔UI+Backend)
```

---

*Plan 1 Complete — See plan2-ai-employee-openclaw.md for the OpenClaw deployment.*
