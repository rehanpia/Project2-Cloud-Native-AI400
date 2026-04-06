# K8 Planning Skill

> **Panaversity Cloud Native AI Training | Project 2 — Reusable Agent Skill**  
> Author: Rehan Ahmed  
> Version: 2.0.0 | Date: 2026-04-04

---

## Skill Metadata

```yaml
name: k8-planning-skill
version: "2.0.0"
description: >
  A reusable agent skill that generates comprehensive, production-grade
  Kubernetes deployment plans for any application. Given a list of services
  and context, it produces Deployments/StatefulSets, Services, ConfigMaps,
  Secrets, Namespaces, RBAC, NetworkPolicies, HPAs, and PodDisruptionBudgets.
author: Panaversity Cloud Native AI Student
tags:
  - kubernetes
  - cloud-native
  - devops
  - planning
  - infrastructure
  - security
trigger_keywords:
  - "kubernetes deployment plan"
  - "k8s plan"
  - "k8 architecture"
  - "deploy to kubernetes"
  - "kubernetes infrastructure"
  - "design k8s for"
  - "plan kubernetes"
input_required:
  - application_name
  - services_list
  - environment
input_optional:
  - security_level
  - cloud_provider
  - persistence_required
  - external_traffic_required
  - team_size
output_format: markdown
```

---

## Skill Description

The **K8 Planning Skill** is a structured reasoning skill for AI agents. When an agent receives a request to plan a Kubernetes deployment, this skill provides the step-by-step reasoning framework, decision trees, validated YAML templates, and output format to produce a complete, production-grade plan.

This skill generates **infrastructure planning documents** — not deployable code. A DevOps engineer or an execution agent fills in placeholder values and applies the manifests to a real cluster.

**What this skill covers:**
- Workload type selection (Deployment vs StatefulSet)
- Namespace design and Pod Security Standards
- Resource sizing tiers (XS → XL)
- Security controls by level (standard / high / critical)
- Service type selection (ClusterIP / NodePort / LoadBalancer / Headless)
- ConfigMap and Secret design
- RBAC: one ServiceAccount per workload, minimum permissions
- NetworkPolicies with correct AND/OR logic for cross-namespace rules
- HorizontalPodAutoscalers and PodDisruptionBudgets

---

## When to Invoke This Skill

Invoke this skill when you receive any of these request patterns:

- "Plan a Kubernetes deployment for [application]"
- "Design K8s infrastructure for [list of services]"
- "How should I deploy [app] to Kubernetes?"
- "Create a K8 architecture for [project]"
- "Generate a Kubernetes manifest plan for [services]"
- "What deployments, services, and RBAC do I need for [app]?"

---

## Input Schema

```json
{
  "application_name": "string — e.g. 'E-Commerce Platform'",
  "services": [
    {
      "name": "string — e.g. 'payment-service'",
      "type": "stateless | stateful | agent | frontend | database | messaging",
      "description": "string — brief role description",
      "needs_external_traffic": "boolean",
      "needs_persistence": "boolean",
      "security_level": "standard | high | critical"
    }
  ],
  "environment": "dev | staging | prod",
  "cloud_provider": "aws | gcp | azure | on-prem | generic",
  "additional_constraints": "string — optional free-text (e.g. 'must use mTLS', 'no internet egress')"
}
```

---

## Agent Reasoning Framework

The agent MUST work through these five phases sequentially. Do not skip phases.

### Phase 1 — Service Classification

For each service, apply these rules:

```
WORKLOAD TYPE:
  service needs persistent storage (DB, vector store, files)?  → StatefulSet
  service is a message broker (Kafka, RabbitMQ)?               → StatefulSet
  service is stateless and independently scalable?             → Deployment

SERVICE TYPE:
  end-users on the internet need to reach this?    → LoadBalancer (or Ingress + ClusterIP)
  only pods in same namespace need to reach this?  → ClusterIP
  pods in OTHER namespaces need to reach this?     → ClusterIP + cross-namespace NetworkPolicy
  StatefulSet needs stable pod identity / DNS?     → also create a Headless service

SECURITY BASELINE for ALL services:
  - Dedicated ServiceAccount (never use 'default')
  - Role with only the exact permissions needed
  - requests AND limits set for CPU and memory
  - readinessProbe AND livenessProbe defined
  - Image pinned to exact version tag (never :latest in prod)
```

### Phase 2 — Namespace Design

```
IF environment = prod AND total services >= 4:
  → Create separate namespaces by tier (frontend, core/backend, data, monitoring)

IF any service has security_level = critical:
  → Give it a dedicated namespace
  → Apply pod-security.kubernetes.io/enforce: restricted

IF any service is an AI agent, tool runner, or sandbox executor:
  → Isolate in its own namespace
  → automountServiceAccountToken: false
  → Deny-all NetworkPolicy on that namespace

ALWAYS create:
  → {app-name}-monitoring namespace (for Prometheus/Grafana)
  → Apply deny-all NetworkPolicy as the first policy in every namespace
```

### Phase 3 — Resource Sizing

Select the tier that fits each service's workload profile:

| Tier | CPU Request | CPU Limit | Mem Request | Mem Limit | Best For |
|---|---|---|---|---|---|
| XS | 100m | 250m | 128Mi | 256Mi | Simple utilities, init containers |
| S | 250m | 500m | 256Mi | 512Mi | Lightweight frontends, notification services |
| M | 500m | 1000m | 512Mi | 1Gi | Standard REST APIs, background workers |
| L | 1000m | 2000m | 1Gi | 2Gi | Databases, agents, message consumers |
| XL | 2000m | 4000m | 4Gi | 8Gi | LLM orchestrators, ML inference services |

**Sizing rules:**
- Always set both `requests` AND `limits`. Omitting limits gives pods the Burstable QoS class and makes them eviction candidates under memory pressure.
- For prod, minimum replicas: frontend=3, backend=4, agents=2, stateful=1.
- Add `topologySpreadConstraints` with `maxSkew: 1` for any Deployment with 3+ replicas.
- Add `startupProbe` for any service that takes > 10 seconds to start (Next.js, JVM apps, model loading).

### Phase 4 — Security Controls by Level

```
STANDARD security level — apply to all services:
  ✓ Dedicated ServiceAccount
  ✓ Role: only specific resourceNames, only 'get' verb for secrets
  ✓ runAsNonRoot: true
  ✓ fsGroup: set (required for volume file ownership)
  ✓ seccompProfile: RuntimeDefault
  ✓ Resource limits defined
  ✓ Image pinned to exact version

HIGH security level — add to standard:
  ✓ readOnlyRootFilesystem: true
  ✓ allowPrivilegeEscalation: false
  ✓ capabilities.drop: ["ALL"]
  ✓ Deny-all NetworkPolicy + explicit allow per pair
  ✓ Secret rotation strategy documented

CRITICAL security level — add to high:
  ✓ automountServiceAccountToken: false (where K8s API not needed)
  ✓ Dedicated namespace with pod-security.kubernetes.io/enforce: restricted
  ✓ Secrets via Vault sidecar (in-memory volume), NOT env vars
  ✓ mTLS via Istio PeerAuthentication: STRICT
  ✓ Istio AuthorizationPolicy: named principals only
  ✓ All actions logged to append-only audit store
  ✓ PodDisruptionBudget defined
```

### Phase 5 — Communication Mapping

Build a full matrix before writing NetworkPolicies:

```
For every service pair (A calls B):
  1. Identify the exact port B listens on
  2. Is this intra-namespace or cross-namespace?
  3. Does it need mTLS? (critical level = yes)
  4. Which auth method? (JWT / mTLS cert / none)

CRITICAL NetworkPolicy rule (corrected from common mistake):
  Cross-namespace AND logic: BOTH namespaceSelector AND podSelector
  must be in the SAME list item under 'from:'.
  Placing them as separate list items = OR logic (allows ANY pod from
  that namespace OR any pod with that label, anywhere) — a security hole.

  CORRECT (AND logic — requires BOTH conditions):
    ingress:
      - from:
          - namespaceSelector:      ← same list item
              matchLabels:
                tier: core
            podSelector:            ← same list item = AND
              matchLabels:
                app: orchestrator

  WRONG (OR logic — insecure):
    ingress:
      - from:
          - namespaceSelector:      ← separate list item
              matchLabels:
                tier: core
          - podSelector:            ← separate item = OR (INSECURE)
              matchLabels:
                app: orchestrator
```

---

## Output Template

Every plan generated by this skill MUST use this section structure:

```markdown
# Kubernetes Deployment Plan — {APPLICATION_NAME}

> Panaversity Cloud Native AI Training | Version: X.Y | Date: YYYY-MM-DD

## 1. Application Overview
[Table: Component | Role | Workload Type | Namespace | Replicas]

## 2. Namespace Design
[Namespace YAML with PSS labels + rationale for each namespace]

## 3. Pods and Deployments / StatefulSets
[One subsection per workload — full YAML with comments explaining key decisions]

## 4. Services
[Summary table: Service | Type | Port | Exposed To | Rationale]
[Full Service YAML for each]

## 5. ConfigMaps
[All ConfigMaps. Non-sensitive values only. URLs, feature flags, timeouts.]

## 6. Secrets Management
[Secret YAML stubs with annotations]
[Table: Layer | Tool | Policy — showing full rotation strategy]

## 7. RBAC — Roles and RoleBindings
[ServiceAccount + Role + RoleBinding per workload]
[Human access roles at the end]

## 8. Inter-Service Communication
[ASCII flow diagram]
[DNS address table: From | To | Address | Port | Auth method]
[mTLS policy YAML if applicable]
[NetworkPolicy YAML — deny-all first, then explicit allows]

## 9. HorizontalPodAutoscalers
[HPA for all stateless Deployments in prod]

## 10. PodDisruptionBudgets
[PDB for all critical and multi-replica workloads]

## 11. Resource Summary Table
[Component | Type | Replicas | CPU Req | CPU Limit | Mem Req | Mem Limit]

## 12. Architecture Diagram
[ASCII box diagram showing namespace boundaries and traffic flow]
```

---

## Reusable YAML Snippets Library

### Snippet 1 — Namespace with Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {APP_NAME}-{TIER}
  labels:
    app: {APP_NAME}
    tier: {TIER}
    # Choose one: restricted (no root, no priv esc) | baseline | privileged
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Snippet 2 — Deployment Template (Hardened)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {SERVICE_NAME}
  namespace: {NAMESPACE}
  labels:
    app: {SERVICE_NAME}
    tier: {TIER}
spec:
  replicas: {REPLICAS}
  selector:
    matchLabels:
      app: {SERVICE_NAME}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: {SERVICE_NAME}
    spec:
      serviceAccountName: {SERVICE_NAME}-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: {UID}
        fsGroup: {FSGROUP}         # Required for volume file ownership
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: {SERVICE_NAME}
          image: {REGISTRY}/{IMAGE}:{VERSION}   # Never use :latest in prod
          ports:
            - containerPort: {PORT}
              name: http
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "{CPU_REQUEST}"
              memory: "{MEM_REQUEST}"
            limits:
              cpu: "{CPU_LIMIT}"
              memory: "{MEM_LIMIT}"
          envFrom:
            - configMapRef:
                name: {SERVICE_NAME}-config
          env:
            - name: {SECRET_ENV_NAME}
              valueFrom:
                secretKeyRef:
                  name: {SECRET_NAME}
                  key: {SECRET_KEY}
          readinessProbe:
            httpGet:
              path: /health
              port: {PORT}
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: {PORT}
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          startupProbe:              # Include for slow-starting services (JVM, Next.js, model loading)
            httpGet:
              path: /health
              port: {PORT}
            failureThreshold: 12    # 12 * 5s = 60s max startup time
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      topologySpreadConstraints:    # Include for Deployments with 3+ replicas
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: {SERVICE_NAME}
```

### Snippet 3 — StatefulSet Template

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {SERVICE_NAME}
  namespace: {NAMESPACE}
  labels:
    app: {SERVICE_NAME}
spec:
  serviceName: {SERVICE_NAME}-headless
  replicas: {REPLICAS}
  selector:
    matchLabels:
      app: {SERVICE_NAME}
  template:
    metadata:
      labels:
        app: {SERVICE_NAME}
    spec:
      serviceAccountName: {SERVICE_NAME}-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: {UID}
        fsGroup: {UID}             # Same as runAsUser for DB containers
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: {SERVICE_NAME}
          image: {IMAGE}:{VERSION}
          ports:
            - containerPort: {PORT}
              name: {PROTOCOL}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "{CPU_REQUEST}"
              memory: "{MEM_REQUEST}"
            limits:
              cpu: "{CPU_LIMIT}"
              memory: "{MEM_LIMIT}"
          readinessProbe:
            exec:
              command: {READINESS_COMMAND}   # Use exec for DB probes, not httpGet
            initialDelaySeconds: 15
            periodSeconds: 10
          livenessProbe:
            exec:
              command: {LIVENESS_COMMAND}
            initialDelaySeconds: 45
            periodSeconds: 20
          volumeMounts:
            - name: {DATA_VOLUME}
              mountPath: {DATA_PATH}
  volumeClaimTemplates:
    - metadata:
        name: {DATA_VOLUME}
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "{STORAGE_CLASS}"   # Use encrypted class in prod
        resources:
          requests:
            storage: {STORAGE_SIZE}
```

### Snippet 4 — Service Templates (All Types)

```yaml
# LoadBalancer — for internet-facing services
apiVersion: v1
kind: Service
metadata:
  name: {SERVICE_NAME}-svc
  namespace: {NAMESPACE}
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: {SERVICE_NAME}
  ports:
    - name: https
      port: 443
      targetPort: {CONTAINER_PORT}
---
# ClusterIP — for internal-only services (default, most secure)
apiVersion: v1
kind: Service
metadata:
  name: {SERVICE_NAME}-svc
  namespace: {NAMESPACE}
spec:
  type: ClusterIP
  selector:
    app: {SERVICE_NAME}
  ports:
    - name: http
      port: {PORT}
      targetPort: {PORT}
---
# Headless — required alongside StatefulSets for stable pod DNS
apiVersion: v1
kind: Service
metadata:
  name: {SERVICE_NAME}-headless
  namespace: {NAMESPACE}
spec:
  clusterIP: None           # This is what makes it headless
  selector:
    app: {SERVICE_NAME}
  ports:
    - name: http
      port: {PORT}
      targetPort: {PORT}
```

### Snippet 5 — RBAC Template (Minimum Permissions)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {SERVICE_NAME}-sa
  namespace: {NAMESPACE}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {SERVICE_NAME}-role
  namespace: {NAMESPACE}
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["{CONFIGMAP_NAME}"]   # Named resource — tightest possible scope
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["{SECRET_NAME}"]
    verbs: ["get"]                        # Only 'get' — never 'list' or 'watch' for secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {SERVICE_NAME}-rolebinding
  namespace: {NAMESPACE}
subjects:
  - kind: ServiceAccount
    name: {SERVICE_NAME}-sa
    namespace: {NAMESPACE}
roleRef:
  kind: Role
  name: {SERVICE_NAME}-role
  apiGroup: rbac.authorization.k8s.io
```

### Snippet 6 — NetworkPolicy Templates (with AND/OR explanation)

```yaml
# Step 1: Always start with deny-all in every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: {NAMESPACE}
spec:
  podSelector: {}           # Selects ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
---
# Step 2: Intra-namespace allow (simple — no namespace selector needed)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-{SOURCE}-to-{TARGET}
  namespace: {NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: {TARGET_SERVICE}
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: {SOURCE_SERVICE}
      ports:
        - port: {PORT}
---
# Step 3: Cross-namespace allow — AND logic (BOTH conditions must match)
# CRITICAL: namespaceSelector and podSelector in the SAME list item = AND
# Putting them in SEPARATE list items = OR (insecure — allows any pod in ns OR any pod with that label)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-{SOURCE_NS}-{SOURCE_SVC}-to-{TARGET_SVC}
  namespace: {TARGET_NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: {TARGET_SERVICE}
  ingress:
    - from:
        - namespaceSelector:          # AND condition 1: must be from this namespace
            matchLabels:
              tier: {SOURCE_TIER}
          podSelector:                # AND condition 2: must be this pod (SAME list item!)
            matchLabels:
              app: {SOURCE_SERVICE}
      ports:
        - port: {PORT}
---
# Step 4: Controlled external egress (HTTPS only)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-{SERVICE}-external-egress
  namespace: {NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: {SERVICE_NAME}
  egress:
    - ports:
        - port: 443
          protocol: TCP
        - port: 53                    # DNS resolution (required for external calls)
          protocol: UDP
```

### Snippet 7 — HPA Template

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {SERVICE_NAME}-hpa
  namespace: {NAMESPACE}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {SERVICE_NAME}
  minReplicas: {MIN}
  maxReplicas: {MAX}
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
      stabilizationWindowSeconds: 300    # 5-min cooldown prevents flapping
    scaleUp:
      stabilizationWindowSeconds: 60
```

### Snippet 8 — PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {SERVICE_NAME}-pdb
  namespace: {NAMESPACE}
spec:
  # Use minAvailable OR maxUnavailable — not both
  minAvailable: {N}          # Keep at least N pods alive during voluntary disruptions
  # maxUnavailable: 1        # Alternative: allow at most 1 to be down
  selector:
    matchLabels:
      app: {SERVICE_NAME}
```

### Snippet 9 — ResourceQuota per Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {NAMESPACE}-quota
  namespace: {NAMESPACE}
spec:
  hard:
    pods: "{MAX_PODS}"
    requests.cpu: "{TOTAL_CPU_REQUEST}"
    requests.memory: "{TOTAL_MEM_REQUEST}"
    limits.cpu: "{TOTAL_CPU_LIMIT}"
    limits.memory: "{TOTAL_MEM_LIMIT}"
    secrets: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

### Snippet 10 — Secret Expiry CronJob (Corrected Syntax)

```yaml
# Common mistake: 'name: KEY: "value"' is invalid YAML.
# Correct: 'name' and 'value' are separate fields.
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-expiry-checker
  namespace: {NAMESPACE}
spec:
  schedule: "0 9 * * 1"         # Every Monday 09:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-checker-sa
          securityContext:
            runAsNonRoot: true
            runAsUser: {UID}
          containers:
            - name: checker
              image: {CHECKER_IMAGE}:{VERSION}
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
              env:
                - name: ALERT_DAYS_BEFORE    # Correct: separate 'name' and 'value' fields
                  value: "14"
                - name: SLACK_WEBHOOK_URL
                  valueFrom:
                    secretKeyRef:
                      name: monitoring-secrets
                      key: slack-webhook
          restartPolicy: OnFailure
```

### Snippet 11 — Istio mTLS + AuthorizationPolicy

```yaml
# Enforce mTLS for all traffic in a namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: require-mtls
  namespace: {NAMESPACE}
spec:
  mtls:
    mode: STRICT
---
# Restrict which service account can call which service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: {TARGET_SERVICE}-authz
  namespace: {TARGET_NAMESPACE}
spec:
  selector:
    matchLabels:
      app: {TARGET_SERVICE}
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/{SOURCE_NAMESPACE}/sa/{SOURCE_SERVICE_ACCOUNT}"
      to:
        - operation:
            methods: ["{HTTP_METHOD}"]
            paths: ["{PATH}"]
```

---

## Decision Reference Cards

### Card 1 — Which Workload Type?

```
Service needs persistent data (database, vector store, files, queue state)?
  YES → StatefulSet + Headless Service + volumeClaimTemplate
  NO  → Deployment

Service is a message broker (Kafka, RabbitMQ, NATS)?
  YES → StatefulSet

Service is stateless, independently scalable, no local data?
  YES → Deployment + HPA
```

### Card 2 — Which Service Type?

```
Internet users need to reach this?
  YES → LoadBalancer (simple setup) OR Ingress + ClusterIP (advanced: SSL offload, path routing)
  NO  →

Only pods in same namespace call this?
  YES → ClusterIP (short DNS: service-name:port)

Pods in OTHER namespaces call this?
  YES → ClusterIP + cross-namespace NetworkPolicy (AND logic — see Snippet 6)

Is this a StatefulSet?
  YES → Also create a Headless Service (clusterIP: None) for stable per-pod DNS
        e.g., postgres-0.postgres-headless.namespace.svc.cluster.local
```

### Card 3 — Security Controls by Profile

```
All workloads (baseline):
  → Dedicated ServiceAccount
  → Named-resource RBAC (not wildcard)
  → CPU + memory requests AND limits
  → Probes: readiness + liveness (+ startup for slow services)
  → Image pinned to exact version (not :latest)
  → runAsNonRoot: true + fsGroup set

Add for HIGH:
  → readOnlyRootFilesystem: true
  → allowPrivilegeEscalation: false
  → capabilities.drop: ["ALL"]
  → Deny-all + explicit NetworkPolicies
  → Secret rotation documented + expiry annotations

Add for CRITICAL:
  → automountServiceAccountToken: false
  → Dedicated restricted namespace
  → Vault sidecar injection (secrets as memory volumes, not env vars)
  → Istio mTLS: STRICT + AuthorizationPolicy per service
  → Audit logging for all actions
  → PodDisruptionBudget
  → topologySpreadConstraints (multi-node HA)
```

### Card 4 — Cross-Namespace NetworkPolicy AND vs OR

```
SCENARIO: Orchestrator in 'core' namespace calls Tool Executor in 'sandbox' namespace.

GOAL: Only the orchestrator pod from the core namespace should be allowed.

WRONG (OR logic — dangerous):
  from:
    - namespaceSelector:        ← matches ANY pod from 'core' namespace
        matchLabels:
          tier: core
    - podSelector:              ← OR any pod with this label, from ANY namespace
        matchLabels:
          app: orchestrator

CORRECT (AND logic — secure):
  from:
    - namespaceSelector:        ← must be from 'core' namespace
        matchLabels:
          tier: core
      podSelector:              ← AND must have this label (same list item)
        matchLabels:
          app: orchestrator
```

---

## Evaluation Criteria (Pre-Submission Self-Check)

Before outputting the plan, verify each item:

**Completeness:**
- [ ] Every component has a Deployment or StatefulSet
- [ ] Every component has a corresponding Service
- [ ] Every Service type is appropriate (LB for external, ClusterIP for internal)
- [ ] All StatefulSets have a Headless Service

**Security:**
- [ ] Every pod has a dedicated ServiceAccount (not `default`)
- [ ] Every ServiceAccount has a Role with named resources (not wildcards)
- [ ] No secrets exposed as plain env vars in critical workloads (use Vault volumes)
- [ ] Every Secret has `secret-expiry` annotation and rotation policy
- [ ] All namespaces have a `default-deny-all` NetworkPolicy
- [ ] Cross-namespace NetworkPolicies use AND logic (same list item)
- [ ] All pods: `runAsNonRoot: true`, `fsGroup` set, `seccompProfile: RuntimeDefault`

**Reliability:**
- [ ] All containers have both `readinessProbe` and `livenessProbe`
- [ ] Slow-starting services have `startupProbe`
- [ ] All production Deployments have HPA defined
- [ ] Critical workloads have PodDisruptionBudget
- [ ] Multi-replica Deployments have `topologySpreadConstraints`
- [ ] Images use pinned version tags (no `:latest`)

**YAML correctness:**
- [ ] CronJob env vars use separate `name:` and `value:` keys (not `name: KEY: value`)
- [ ] Cross-namespace NetworkPolicies: `namespaceSelector` + `podSelector` in same `from` item
- [ ] `fsGroup` is set wherever volumes are mounted
- [ ] StatefulSet `serviceName` matches the Headless Service name

---

## Example Invocations

### Example 1 — Simple Blog Platform

**Input:**
```json
{
  "application_name": "My Blog",
  "services": [
    {"name": "frontend", "type": "frontend", "needs_external_traffic": true, "security_level": "standard"},
    {"name": "api", "type": "stateless", "needs_external_traffic": false, "security_level": "standard"},
    {"name": "postgres", "type": "database", "needs_persistence": true, "security_level": "standard"}
  ],
  "environment": "prod",
  "cloud_provider": "aws"
}
```

**Agent Reasoning:**
- frontend → Deployment (S tier: 250m/512Mi), LoadBalancer Service (user-facing), 3 replicas
- api → Deployment (M tier: 500m/1Gi), ClusterIP Service, 3 replicas, HPA(3-8)
- postgres → StatefulSet (L tier: 1000m/2Gi), ClusterIP + Headless, 1 replica, 20Gi PVC
- 2 namespaces: `blog-prod`, `blog-monitoring`
- Security: standard — runAsNonRoot, limits, RBAC, deny-all NetworkPolicies
- No mTLS (standard level), no Vault sidecar

### Example 2 — AI Agent Platform

**Input:**
```json
{
  "application_name": "My AI Assistant",
  "services": [
    {"name": "dashboard", "type": "frontend", "needs_external_traffic": true, "security_level": "standard"},
    {"name": "orchestrator", "type": "agent", "needs_external_traffic": false, "security_level": "high"},
    {"name": "tool-runner", "type": "agent", "needs_external_traffic": false, "security_level": "critical"},
    {"name": "memory", "type": "database", "needs_persistence": true, "security_level": "high"},
    {"name": "auth-service", "type": "stateless", "needs_external_traffic": false, "security_level": "critical"}
  ],
  "environment": "prod",
  "cloud_provider": "aws"
}
```

**Agent Reasoning:**
- dashboard → Deployment (S), LoadBalancer, `app-frontend` namespace
- auth-service → Deployment (M), ClusterIP, `app-core` namespace, critical controls, Vault sidecar
- orchestrator → Deployment (XL: 2000m/4Gi), ClusterIP, `app-core` namespace, high controls
- tool-runner → Deployment (L), ClusterIP, **`app-sandbox` namespace**, critical controls, `automountServiceAccountToken: false`
- memory → StatefulSet (L), ClusterIP + Headless, `app-core` namespace, 50Gi encrypted PVC
- 4 namespaces: `app-frontend`, `app-core`, `app-sandbox`, `app-audit`
- mTLS enforced: core ↔ sandbox (Istio PeerAuthentication: STRICT)
- AuthorizationPolicy: only orchestrator SA can call tool-runner
- Cross-namespace NetworkPolicies use AND logic throughout
- Audit logger in `app-audit` namespace receives writes from both core and sandbox

---

## Skill Limitations

- This skill generates **plans**, not deployable manifests. All `{PLACEHOLDER}` values must be replaced before applying to a cluster.
- Cloud-provider-specific annotations (AWS NLB, GCP NEG, Azure AGIC) are examples; verify current syntax with your provider's documentation.
- `storageClassName` values (`gp3`, `gp3-encrypted`, `standard-rwo`) vary by cluster. Verify with `kubectl get storageclass`.
- Vault and External Secrets Operator require separate cluster-level installation not covered by this skill.
- Istio mTLS snippets require Istio to be installed on the cluster. Linkerd can be used as an alternative with different CRD names.
- This skill does not generate CI/CD pipeline configuration, Helm charts, or Kustomize overlays.

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-04-04 | Initial release |
| 2.0.0 | 2026-04-04 | Fixed cross-namespace NetworkPolicy AND/OR bug; fixed CronJob YAML syntax; added fsGroup to all pod templates; added startupProbe, topologySpreadConstraints, PodDisruptionBudget, ResourceQuota, and Istio snippets; added Card 4 (AND vs OR); expanded self-check; 11 snippets total |

---

*K8 Planning Skill v2.0.0 — End of Document*  
*Designed for Panaversity Cloud Native AI Training Program*
