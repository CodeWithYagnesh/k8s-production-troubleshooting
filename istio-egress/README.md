# 🛡️ Istio Egress Control Strategy

## ❓ Why I needed Egress Control

Application pods could call any external domain by default. This is risky because:
- 🚨 A compromised or malicious dependency inside the cluster could exfiltrate data to an external domain (e.g. database dump sent outbound)
- 🕵️ A third-party/open-source app deployed into the cluster could silently call unknown external endpoints
- 🙈 Without restriction, there is no visibility or control over outbound traffic

## 🎯 Goal

Block all egress traffic by default, and allow only specific, explicitly approved external domains — scoped down to the workload/pod level, not just namespace level.

## 🏗️ Egress Architecture Diagram

The diagram below illustrates how we achieve pod-level egress control. `REGISTRY_ONLY` blocks all unknown outbound traffic mesh-wide. A `ServiceEntry` registers an allowed domain, and a `Sidecar` resource specifically binds that access to `Pod: my-app` only.

```mermaid
flowchart LR
    %% Title Configuration Note
    ConfigNote["<b>1 Egress Control Using : ServiceEntry & Sidecar (for Domains)</b><br/>OutboundTrafficMode: REGISTRY_ONLY"]:::titleNode

    %% External Entity
    OS["Opensense"]:::boxNode

    subgraph Cluster["Kubernetes cluster"]
        direction TB
        
        subgraph Namespace["Namespace"]
            direction TB
            
            %% Service Entry Definition
            SE["<b>ServiceEntry</b><br/><br/>hosts:<br/>- 'yahoo.com'<br/>- 'yagnesh.com'"]:::boxNode

            %% POD 1 Environment
            subgraph Pod1Group[" "]
                direction LR
                SC1["<b>SideCar</b><br/>labelSelector:<br/>  allow-yagnesh: true<br/><br/>egress:<br/>- hosts:<br/>  - '*/yagnesh.com'<br/>  - 'istio-system/*'"]:::boxNode
                
                subgraph Pod1["POD - 1 (Label: allow-yagnesh: true)"]
                    direction LR
                    P1_SC["Sidecar container"]:::boxNode
                    P1_MC["Main Container"]:::boxNode
                end
            end

            %% POD 2 Environment
            subgraph Pod2Group[" "]
                direction LR
                subgraph Pod2["POD - 2 (No restricted Sidecar)"]
                    direction LR
                    P2_SC["Sidecar container"]:::boxNode
                    P2_MC["Main Container"]:::boxNode
                end
            end

            %% POD 3 Environment
            subgraph Pod3Group[" "]
                direction LR
                SC3["<b>SideCar</b><br/>labelSelector:<br/>  allow-github: true<br/><br/>egress:<br/>- hosts:<br/>  - '*/github.com'<br/>  - 'istio-system/*'"]:::boxNode
                
                subgraph Pod3["POD - 3 (Label: allow-github: true)"]
                    direction LR
                    P3_SC["Sidecar container"]:::boxNode
                    P3_MC["Main Container"]:::boxNode
                end
            end
        end
    end

    %% --- Traffic Routing & Logic ---
    
    ConfigNote ~~~ Cluster

    %% Pod 1 Flows
    P1_MC -- "curl yagnesh.com" --> P1_SC
    P1_MC -- "curl yahoo.com" --> P1_SC
    
    P1_SC -->|"yagnesh.com (Allowed)"| SC1
    SC1 -->|"Forwarded"| SE
    
    P1_SC -.->|"yahoo.com (DENIED)"| SC1_Block["Blocked by Sidecar Rules"]:::deniedNode

    %% Pod 2 Flows
    P2_MC -- "curl yagnesh.com" --> P2_SC
    P2_MC -- "curl yahoo.com" --> P2_SC
    P2_MC -- "curl github.com" --> P2_SC
    
    P2_SC -->|"yagnesh.com & yahoo.com"| SE
    
    P2_SC -.->|"github.com (DENIED)"| Reg_Block1["Blocked by REGISTRY_ONLY"]:::deniedNode

    %% Pod 3 Flows
    P3_MC -- "curl github.com" --> P3_SC
    
    P3_SC -->|"github.com (Allowed by SC3)"| SC3
    
    SC3 -.->|"github.com (DENIED)"| Reg_Block2["Blocked: Not in ServiceEntry"]:::deniedNode

    %% Final Outbound
    SE == "yagnesh.com / yahoo.com" ==> OS

    %% --- Styling Definitions ---
    classDef titleNode fill:#fcfcfa,stroke:#ccc,stroke-width:1px,font-size:14px;
    classDef boxNode fill:#ffffff,stroke:#333333,stroke-width:1px;
    classDef deniedNode fill:#ffcccc,stroke:#cc0000,stroke-width:2px,color:#cc0000;
```

## 🚥 Istio's 3 Ways to Control Egress

1. 🚪 **Istio Egress Gateway** — centralized egress traffic exit point
2. 📝 **ServiceEntry** — registers allowed external services into the mesh
3. 🏎️ **Sidecar** (optional) — restricts which services/hosts a specific workload's proxy can see

## 🛑 Step 1 — Block All Egress by Default

By default, Istio's `outboundTrafficPolicy` mode is `ALLOW_ANY` (any external domain reachable). Change it to `REGISTRY_ONLY` to block everything not explicitly registered.

**Option A — via IstioOperator**
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

**Option B — via istio ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

At this point, all outbound traffic is blocked unless registered via a `ServiceEntry`.

## ⚠️ Step 2 — Problem: ServiceEntry Is Not Pod-Level

A `ServiceEntry` registers an allowed external domain into the mesh, but by default its visibility applies at the namespace/mesh level — not scoped to a specific workload/pod. This means once a domain is allowed via ServiceEntry, every pod that can see it may reach it, which is too broad for fine-grained control.

## 💡 Step 3 — Solution: Sidecar Resource + ServiceEntry

To restrict egress access to specific domains **per workload**, combine:
- `ServiceEntry` — to register the allowed external domain
- `Sidecar` resource with `workloadSelector` — to scope which hosts that specific workload's proxy is allowed to see

**ServiceEntry — register the allowed external domain**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: allow-external-api
  namespace: app-namespace
spec:
  hosts:
    - api.example.com
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
```

**Sidecar — restrict visibility to specific workload (pod-level)**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: restricted-egress
  namespace: app-namespace
spec:
  workloadSelector:
    labels:
      app: my-app
  egress:
    - hosts:
        - "./*"
        - "istio-system/*"
        - "app-namespace/api.example.com"
```

This ensures only pods matching `app: my-app` can reach `api.example.com`, while all other workloads in the namespace remain restricted to in-mesh traffic only.

## ✅ Result

- 🧱 All egress blocked by default (`REGISTRY_ONLY`)
- 📋 Explicit allow-list of external domains via `ServiceEntry`
- 🎯 Fine-grained, workload-level enforcement via `Sidecar` + `workloadSelector`
- 🔒 Prevents data exfiltration and unauthorized outbound calls from compromised or untrusted workloads

## 🚀 Advanced Egress Concepts

Based on Istio's architecture, here are the detailed roles of different resources for more advanced egress management:

### 1. Egress Gateways vs Sidecar-Only
While the Sidecar + ServiceEntry approach is great for pod-level restrictions, an **Egress Gateway** (`Gateway` resource) provides a centralized, dedicated exit node for all outbound traffic. 
- **Use Cases**: Useful when you need a static, predictable outgoing IP address (for external firewall whitelisting), or when you want to enforce centralized auditing and observability.
- **Routing**: You use a `VirtualService` to direct egress traffic from the application sidecars to the Egress Gateway, and from there to the external `ServiceEntry`.

### 2. VirtualService in Egress
A `VirtualService` dictates *how* requests to external services are routed. It is essential when directing egress traffic through an Egress Gateway, allowing you to intercept traffic at the pod level, forward it to the gateway proxy, and optionally inject faults, retries, or timeouts.

### 3. ServiceEntry Limitations & External Authorization
- **Pod-Label Routing**: A `ServiceEntry` natively applies globally or per-namespace, but **cannot** restrict access based on the requesting pod's labels. This is why the `Sidecar` resource is strictly required to lock down access at the pod level.
- **HTTPS & External Authorization**: If you route HTTPS traffic through an Egress Gateway and try to enforce L7 external authorization (like OPA or WAF), it **will not work out-of-the-box**. The gateway only sees encrypted TCP traffic (SNI). To enforce path-based HTTP policies on outbound HTTPS traffic, you must perform **TLS Origination** at the egress gateway so it can inspect the decrypted payload.

## 📊 Detailed Egress Diagrams

```markdown
### 1) Default passthrough — ALLOW_ANY
```
```mermaid
flowchart LR
    subgraph K8S["Kubernetes Cluster"]
        subgraph NS["Namespace"]
            subgraph POD["Pod"]
                SC["Sidecar"]
                APP["Application"]
                SC --> APP
            end
        end
    end
    APP -->|"allowed, no restriction"| EXT["partner-a.example.com"]
```

```
### 2A) Cluster-level ServiceEntry — REGISTRY_ONLY
```
```mermaid
flowchart LR
    SE["ServiceEntry (cluster-wide)<br/>hosts:<br/>partner-a.example.com<br/>partner-b.example.com"]
    subgraph K8S["Kubernetes Cluster · REGISTRY_ONLY"]
        subgraph NS["Namespace"]
            subgraph POD["Pod"]
                SC["Sidecar"]
                APP["Application"]
            end
        end
    end
    SE -.registers.-> K8S
    APP -->|"allowed"| D1["partner-a.example.com"]
    APP -->|"allowed"| D2["partner-b.example.com"]
    APP -->|"blocked"| D3["partner-c.example.com"]
```

```
### 2B) Namespace-level ServiceEntry — REGISTRY_ONLY
```
```mermaid
flowchart LR
    subgraph K8S["Kubernetes Cluster · REGISTRY_ONLY"]
        subgraph NS["Namespace"]
            SE["ServiceEntry (namespace-scoped)<br/>hosts:<br/>partner-a.example.com<br/>partner-b.example.com"]
            subgraph POD["Pod"]
                SC["Sidecar"]
                APP["Application"]
            end
        end
    end
    APP -->|"allowed"| D1["partner-a.example.com"]
    APP -->|"allowed"| D2["partner-b.example.com"]
    APP -->|"blocked"| D3["partner-c.example.com"]
```

```
### 2C) ServiceEntry + Sidecar resource — AND logic
```
```mermaid
flowchart LR
    SE["ServiceEntry<br/>hosts:<br/>partner-a.example.com<br/>partner-c.example.com"]
    SIDECAR["Sidecar egress hosts:<br/>*/partner-a.example.com<br/>*/partner-b.example.com<br/>istio-system/*"]
    subgraph K8S["Kubernetes Cluster · REGISTRY_ONLY"]
        subgraph NS["Namespace"]
            subgraph POD["Pod"]
                SC["Sidecar container"]
                APP["Application"]
            end
        end
    end
    SE -.-> K8S
    SIDECAR -.-> POD
    APP -->|"allowed, in both lists"| D1["partner-a.example.com"]
    APP -->|"blocked, sidecar has it, SE doesn't"| D2["partner-b.example.com"]
    APP -->|"blocked, SE has it, sidecar doesn't"| D3["partner-c.example.com"]
```

```
### 2D) Internal & external domains — Sidecar with pod selector
```
```mermaid
flowchart LR
    SE["ServiceEntry<br/>hosts: payment-gateway.example.com,<br/>static-assets.example.com,<br/>internal-api.example.com,<br/>analytics.example.com,<br/>registry.example.com<br/>port: 443 TLS"]
    SIDECAR["Sidecar (with pod selector)<br/>hosts: static-assets.example.com,<br/>*.cluster.local"]
    POD["Pod<br/>reaches: static-assets.example.com,<br/>kubernetes.default.svc"]
    SE --> SIDECAR --> POD
```

```
### 2E) Internal domains only — default Sidecar (no pod selector)
```
```mermaid
flowchart LR
    SIDECAR["Sidecar (without pod selector)<br/>default sidecar<br/>hosts: *.cluster.local"]
    POD["Pod<br/>reaches: kubernetes.default.svc"]
    SIDECAR --> POD
```

```
### 2F) External IPs
```
```mermaid
flowchart LR
    SE1["ServiceEntry<br/>hosts: mail.smtps.external<br/>address: 198.51.100.10<br/>port: 993 TCP"]
    SE2["ServiceEntry<br/>hosts: data.tcp.external<br/>address: 203.0.113.5, 203.0.113.6<br/>port: 465 TCP"]
    SIDECAR["Sidecar (with pod selector)<br/>hosts: mail.smtps.external:993,<br/>data.tcp.external:465,<br/>*.cluster.local"]
    POD["Pod<br/>reaches:<br/>203.0.113.5:465<br/>203.0.113.6:465<br/>198.51.100.10:993"]
    SE1 --> SIDECAR
    SE2 --> SIDECAR
    SIDECAR --> POD
```

```
### 2G) Full strategy — multiple ServiceEntries + Sidecars + Pods
```
```mermaid
flowchart TB
    subgraph SES["ServiceEntries"]
        SE1["mail.smtps.external<br/>address: 198.51.100.10<br/>port: 993 TCP"]
        SE2["data.tcp.external<br/>address: 203.0.113.5, 203.0.113.6<br/>port: 465 TCP"]
        SE3["se-domains-https-443<br/>payment-gateway.example.com<br/>static-assets.example.com<br/>internal-api.example.com<br/>analytics.example.com<br/>registry.example.com<br/>port: 443 TLS"]
    end

    subgraph SCS["Sidecars"]
        SC0["default sidecar<br/>no pod selector<br/>hosts: *.cluster.local"]
        SC1["sc-sales<br/>hosts: mail.smtps.external:993,<br/>data.tcp.external:465,<br/>static-assets.example.com:443,<br/>*.cluster.local"]
        SC2["sc-print-service<br/>hosts: static-assets.example.com:443,<br/>*.cluster.local"]
    end

    subgraph PODS["Pods"]
        P0["Pod: any<br/>kubernetes.default.svc.cluster.local"]
        P1["Pod: sales-instance-a<br/>203.0.113.5:465, 203.0.113.6:465<br/>198.51.100.10:993<br/>payment-gateway.example.com<br/>static-assets.example.com<br/>kubernetes.default.svc.cluster.local"]
        P2["Pod: sales-instance-b<br/>same access as sales-instance-a"]
        P3["Pod: print-service<br/>static-assets.example.com<br/>kubernetes.default.svc.cluster.local"]
    end

    SE1 --> SC1
    SE2 --> SC1
    SE3 --> SC1
    SE3 --> SC2
    SC0 --> P0
    SC1 --> P1
    SC1 --> P2
    SC2 --> P3
```

```
### 3) Bypass Envoy entirely — excludeOutboundIPRanges (not used in production)
```
```mermaid
flowchart LR
    ANNOT["annotation:<br/>traffic.sidecar.istio.io/excludeOutboundIPRanges:<br/>'203.0.113.99/32'"]
    subgraph K8S["Kubernetes Cluster"]
        subgraph NS["Namespace"]
            subgraph POD["Pod"]
                SC["Sidecar container<br/>(bypassed for this IP)"]
                APP["Application"]
            end
        end
    end
    ANNOT -.-> POD
    APP -->|"direct, no Envoy"| IP["203.0.113.99<br/>curl partner-b.example.com"]
```

IPs use RFC 5737 documentation ranges (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) — safe placeholders, never routable on the real internet.

## 📚 References

- [StackOverflow: Does Istio ServiceEntry support routing to different hosts by requesting pod label?](https://stackoverflow.com/questions/69634855/does-istio-serviceentry-support-routing-to-different-hosts-by-requesting-pod-lab)
- [StackOverflow: Istio egress gateway with external authorization cannot enforce policy on HTTPS](https://stackoverflow.com/questions/79769417/istio-egress-gateway-with-external-authorization-cannot-enforce-policy-on-https)
- [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway)
- [EgressGateway](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)
- [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/)
- [Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar)
- [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [Traffic Egress Through Virtualservice 4th point](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#egress-gateway-for-http-traffic)
