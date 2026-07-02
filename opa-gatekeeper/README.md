# 🛡️ OPA Gatekeeper — Kubernetes Policy Enforcement

## ❓ Why OPA Gatekeeper?

By default, Kubernetes allows any pod to deploy from any container registry, with any configuration, missing any label. This creates serious security and operational risks:

- A developer deploys a container from an untrusted public registry — malware risk
- Pods deploy without required labels — breaks monitoring and alerting
- Deployments run with 1 replica — no high availability
- No enforcement of `imagePullPolicy: Always` — stale images run silently

OPA Gatekeeper solves this by acting as an **admission controller** — it intercepts every `kubectl apply` and enforces your policies before Kubernetes accepts the resource.

---

## 🧠 How It Works

```
kubectl apply pod.yaml
        │
        ▼
Kubernetes API Server
        │
        │ webhook call
        ▼
   Gatekeeper
        │
   ┌────┴────────────────────┐
   │  ConstraintTemplate      │  ← Rego policy (the blueprint)
   │  Constraint              │  ← Parameters + match rules (the instance)
   └────┬────────────────────┘
        │
   ALLOW / DENY
        │
        ▼
  Resource saved / rejected
```

**Two CRDs work together:**
- `ConstraintTemplate` — defines the Rego policy logic (blueprint)
- `Constraint` — instantiates the policy with parameters and match rules

```
ConstraintTemplate  ──creates──▶  Custom CRD (e.g. K8sRequiredLabels)
                                        │
                                   Constraint (ns-must-have-gk)
                                        │
                                Enforced on matching resources
```

---

## 🚦 Enforcement Actions

| Action | Behavior |
|---|---|
| `deny` | Blocks the resource — admission rejected |
| `dryrun` | Allows the resource but records violations — safe for rollout |
| `warn` | Allows the resource but shows a warning to the user |

> **Best practice:** Always use `dryrun` first when rolling out new constraints to avoid breaking a running cluster.

---

## 📦 Installation

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace

kubectl get pods -n gatekeeper-system
```

---

## ✅ Validation Policies

### Policy 1 — Required Labels on Namespace

Every namespace must have a specific label. If missing → denied.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
```

---

### Policy 2 — Allowed Image Registries

Pods must use images only from approved registries. Any other registry → denied.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not strings.any_prefix_match(container.image, input.parameters.allowedImages)
          msg := sprintf("container <%v> has an invalid image <%v>", [container.name, container.image])
        }
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not strings.any_prefix_match(container.image, input.parameters.allowedImages)
          msg := sprintf("initContainer <%v> has an invalid image <%v>", [container.name, container.image])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-secure-registry
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "default"
  parameters:
    allowedImages:
      - "secure-registry.example.com/"
      - "gcr.io/my-project/"
```

---

### Policy 3 — Deny Specific Registry (dryrun)

Flag pods using a specific registry — useful for auditing before enforcing.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: denyrepo
spec:
  crd:
    spec:
      names:
        kind: DenyRepo
      validation:
        openAPIV3Schema:
          type: object
          properties:
            deny:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package denyrepo
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          strings.any_prefix_match(container.image, input.parameters.deny)
          msg := sprintf("container <%v> uses denied registry in image <%v>", [container.name, container.image])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: DenyRepo
metadata:
  name: deny-public-dockerhub
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "test"
  parameters:
    deny:
      - "docker.io/"
```

---

### Policy 4 — Required Pod Labels per Namespace

Pods in specific namespaces must have required labels. Warns if missing.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: nspodreqired
spec:
  crd:
    spec:
      names:
        kind: NsPodRequired
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
            namespaces:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package nspodreqired
        violation[{"msg": msg}] {
          ns := input.review.object.metadata.namespace
          input.parameters.namespaces[_] == ns
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("pods in namespace <%v> must have labels: %v", [ns, missing])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NsPodRequired
metadata:
  name: ns-pod-req
spec:
  enforcementAction: warn
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["owner"]
    namespaces: ["test"]
```

---

### Policy 5 — Unique Label Values Across Pods

No two pods in the same namespace can share the same label value. Requires SyncSet to access existing pods.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: uniquelabelspod
spec:
  crd:
    spec:
      names:
        kind: UniqueLabelsPod
      validation:
        openAPIV3Schema:
          type: object
          properties:
            label:
              type: string
            namespaces:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package uniquelabelspod
        violation[{"msg": msg}] {
          ns := input.review.object.metadata.namespace
          input.parameters.namespaces[_] == ns
          label_key := input.parameters.label
          input_label_value := input.review.object.metadata.labels[label_key]
          other_pod := data.inventory.namespace[ns]["v1"]["Pod"][pod]
          other_pod.metadata.name != input.review.object.metadata.name
          other_pod.metadata.labels[label_key] == input_label_value
          msg := sprintf("Label '%v' with value '%v' must be unique in namespace '%v'", [label_key, input_label_value, ns])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: UniqueLabelsPod
metadata:
  name: unique-owner
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    label: "owner"
    namespaces: ["test"]
```

**SyncSet (required for data.inventory):**
```yaml
apiVersion: syncset.gatekeeper.sh/v1alpha1
kind: SyncSet
metadata:
  name: syncset-pods
spec:
  gvks:
    - group: ""
      version: "v1"
      kind: "Pod"
    - group: ""
      version: "v1"
      kind: "Namespace"
```

---

### Policy 6 — Limit Deployment Replicas

Deployments cannot exceed a maximum replica count.

**ConstraintTemplate:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: limitedreplica
spec:
  crd:
    spec:
      names:
        kind: LimitedReplica
      validation:
        openAPIV3Schema:
          type: object
          properties:
            max:
              type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package limitedreplica
        violation[{"msg": msg}] {
          input.review.object.spec.replicas > input.parameters.max
          msg := sprintf("Deployment replicas <%v> exceeds maximum allowed <%v>", [input.review.object.spec.replicas, input.parameters.max])
        }
```

**Constraint:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: LimitedReplica
metadata:
  name: limit-five
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - test
  parameters:
    max: 5
```

---

## 🔄 Mutation Policies

Mutation happens **before** validation — Gatekeeper automatically modifies resources before they are saved.

### Mutation 1 — Auto-add Label to All Pods

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: add-label-owner
spec:
  match:
    scope: Namespaced
    kinds:
    - apiGroups: ["*"]
      kinds: ["Pod"]
  location: "metadata.labels.owner"
  parameters:
    assign:
      value: "platform-team"
```

---

### Mutation 2 — Force imagePullPolicy Always

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: add-ipp-always
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
    - apiGroups: ["*"]
      kinds: ["Pod"]
    namespaces: ["default"]
  location: "spec.containers[name: *].imagePullPolicy"
  parameters:
    assign:
      value: Always
```

---

### Mutation 3 — Auto-add Port to Services

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: ModifySet
metadata:
  name: set-svc-port-80
spec:
  applyTo:
    - groups: [""]
      kinds: ["Service"]
      versions: ["v1"]
  location: "spec.ports"
  parameters:
    operation: merge
    values:
      fromList:
        - port: 80
          targetPort: 80
          name: default
```

---

### Mutation 4 — Rewrite Image Registry

```yaml
apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: AssignImage
metadata:
  name: assign-container-image
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  location: "spec.containers[name:*].image"
  parameters:
    assignDomain: "secure-registry.example.com"
    assignPath: "my-project"
  match:
    source: "All"
    scope: Namespaced
    kinds:
    - apiGroups: ["*"]
      kinds: ["Pod"]
```

---

## 📊 Monitoring

### OPA Scorecard Exporter

Exposes Gatekeeper constraint violation metrics to Prometheus.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opa-exporter
  namespace: opa-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opa-exporter
  template:
    metadata:
      labels:
        app: opa-exporter
    spec:
      containers:
      - name: opa-exporter
        image: mcelep/opa_scorecard_exporter:v0.0.3
        ports:
        - containerPort: 9141
          name: metrics
```

### PodMonitor for Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: gatekeeper-metric
  namespace: gatekeeper-system
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      gatekeeper.sh/system: "yes"
  podMetricsEndpoints:
    - portNumber: 8888
      path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: opa-exporter
  namespace: opa-exporter
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: opa-exporter
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
```

---

## 🚨 Emergency Recovery

If Gatekeeper blocks the cluster from functioning:

```bash
kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io \
  gatekeeper-validating-webhook-configuration
```

Redeploy the webhook config to re-enable Gatekeeper after fixing the issue.

---

## 📋 Policy Summary

| Policy | Kind | Action | What it enforces |
|---|---|---|---|
| Required Labels | K8sRequiredLabels | deny | Namespaces must have `gatekeeper` label |
| Allowed Registries | K8sAllowedRepos | deny | Pods must use approved image registries |
| Deny Registry | DenyRepo | dryrun | Flag pods using public registries |
| Pod Labels per NS | NsPodRequired | warn | Pods in `test` must have `owner` label |
| Unique Labels | UniqueLabelsPod | deny | `owner` label must be unique per namespace |
| Replica Limit | LimitedReplica | deny | Max 5 replicas per Deployment in `test` |
| Auto-label Pods | AssignMetadata | mutate | Add `owner: platform-team` to all pods |
| imagePullPolicy | Assign | mutate | Force `Always` in `default` namespace |
| Service Port | ModifySet | mutate | Auto-add port 80 to all Services |
| Image Registry | AssignImage | mutate | Rewrite image to secure registry |

---

## 📚 References

- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/)
- [Rego Language](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gatekeeper Mutation](https://open-policy-agent.github.io/gatekeeper/website/docs/mutation/)
- [Gatekeeper SyncSet](https://open-policy-agent.github.io/gatekeeper/website/docs/syncset/)
- [Gatekeeper Audit](https://open-policy-agent.github.io/gatekeeper/website/docs/audit/)
- [ExpansionTemplate](https://open-policy-agent.github.io/gatekeeper/website/docs/expansion/)
