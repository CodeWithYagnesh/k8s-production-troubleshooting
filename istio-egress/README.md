# Istio Egress Control Strategy

## Why I needed Egress Control

Application pods could call any external domain by default. This is risky because:
- A compromised or malicious dependency inside the cluster could exfiltrate data to an external domain (e.g. database dump sent outbound)
- A third-party/open-source app deployed into the cluster could silently call unknown external endpoints
- Without restriction, there is no visibility or control over outbound traffic

## Goal

Block all egress traffic by default, and allow only specific, explicitly approved external domains — scoped down to the workload/pod level, not just namespace level.

## Egress Architecture Diagram

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

## Istio's 3 Ways to Control Egress

1. **Istio Egress Gateway** — centralized egress traffic exit point
2. **ServiceEntry** — registers allowed external services into the mesh
3. **Sidecar** (optional) — restricts which services/hosts a specific workload's proxy can see

## Step 1 — Block All Egress by Default

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

## Step 2 — Problem: ServiceEntry Is Not Pod-Level

A `ServiceEntry` registers an allowed external domain into the mesh, but by default its visibility applies at the namespace/mesh level — not scoped to a specific workload/pod. This means once a domain is allowed via ServiceEntry, every pod that can see it may reach it, which is too broad for fine-grained control.

## Step 3 — Solution: Sidecar Resource + ServiceEntry

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

## Result

- All egress blocked by default (`REGISTRY_ONLY`)
- Explicit allow-list of external domains via `ServiceEntry`
- Fine-grained, workload-level enforcement via `Sidecar` + `workloadSelector`
- Prevents data exfiltration and unauthorized outbound calls from compromised or untrusted workloads
