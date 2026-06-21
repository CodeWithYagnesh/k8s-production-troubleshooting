
# Coraza WAF Implementation — Istio Gateway

<p align="center">
  <img src="image.svg" alt="Coraza WAF Architecture Diagram">
</p>

## Why I needed Coraza WAF

Cluster applications were exposed to attackers. Attack traffic included:
- Malicious requests targeting application endpoints
- Attempts to access/exfiltrate data

Needed a layer to inspect and restrict incoming requests **before** they reach the application — enforced centrally at the Istio Gateway, not per-service. A single enforcement point at the gateway means every application behind it is protected without modifying app code or adding a WAF to each service individually.

## Goal

Block malicious requests at the WAF layer without breaking legitimate traffic (avoid false positives blocking real users).

## Architecture Overview

Coraza runs as a **WasmPlugin** — a WebAssembly module loaded directly into the Envoy proxy's filter chain on the `istio-ingressgateway` pod. Every request passes through this filter chain before it is routed onward via `VirtualService` to the backend `Service`/Pod.

Because it lives inside Envoy at the gateway, it inspects:
- Request headers, query params, and body (`SecRequestBodyAccess`)
- Response headers and body (`SecResponseBodyAccess`)

...before Istio's routing rules (`VirtualService`) even get evaluated for a blocked request. This means a malicious request never reaches the application pod at all.

## Strategy — 4 Stages

1. Run Coraza in **detection-only** mode
2. Collect logs → identify false positives → build suppression rules
3. Apply suppression rules + fix request-side issues
4. Enable rule engine (blocking mode)

This staged rollout exists for one reason: turning on blocking (`SecRuleEngine On`) immediately, without first observing real traffic, risks blocking legitimate users on day one. OWASP CRS rules are generic and not tuned to any specific application, so false positives are expected until tuned.

## Stage Details

### Stage 1 — Detection Only
- Deployed Coraza WAF as a WasmPlugin on the Istio Ingress Gateway
- Set `SecRuleEngine DetectionOnly`
- Let it run against real traffic without blocking anything
- Purpose: learn what kind of requests actually hit the cluster (legit + malicious mixed)

```bash
kubectl apply -f wasmplugin.yaml
kubectl get wasmplugin -n istio-system
```

In this mode, Coraza evaluates every rule and logs a match, but lets the request continue to the application regardless of the outcome.

### Stage 2 — Build Suppression Rules
- Analyzed logs from the detection-only phase
- Found legitimate traffic matching OWASP CRS rules (false positives) — common culprits in CRS are rules around request body anomaly scoring, SQL/XSS pattern heuristics that overlap with normal JSON payloads, and strict content-type checks
- Used `SecRuleUpdateActionById` for targeted per-rule suppression (changes the action of a specific rule ID instead of removing it entirely)
- Used `SecRuleRemoveById` when a rule needed to be fully disabled rather than just reduced in severity
- Used `per_authority_directives` for domain-specific rule exceptions, since different applications behind the same gateway can have very different false-positive profiles

### Stage 3 — Apply + Fix
- Applied suppression rules to the WasmPlugin config
- Where possible, fixed the request itself instead of just suppressing the rule — suppression should be the exception, not the default response to every false positive, since over-suppressing weakens the WAF's actual protection
- Re-monitored traffic to confirm false positives were resolved before moving to the next stage

### Stage 4 — Enable Blocking
- Once false-positive-free, set `SecRuleEngine On`
- WAF now actively blocks malicious requests at the gateway, before they reach any app
- Blocked requests receive a `403` response directly from the gateway pod — the application never sees the request

## Logging / Observability

- Coraza logs shipped to Elasticsearch / OpenSearch
- Built Kibana dashboards for WAF security analysis and attack pattern visibility
- Useful checks while tuning:

```bash
kubectl logs -n istio-system deploy/istio-ingressgateway -c istio-proxy --tail=100 -f
```

Dashboards should track: rule hit frequency by rule ID, blocked vs detected counts, top source IPs, and top targeted domains — this is what turns raw logs into something you can actually act on for further tuning.

## Result

Cluster secured against external threats at the gateway layer, with minimal false positives and centralized visibility into attack traffic.

## Configuration Example

Example `WasmPlugin` configuration (`wasmplugin.yaml`) showing global rules as well as domain-specific exceptions:

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: coraza-waf
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  url: oci://ghcr.io/corazawaf/coraza-proxy-wasm:latest
  phase: AUTHN
  pluginConfig:
    directives_map:
      default:
        - "SecRuleEngine On"
        - "SecRequestBodyAccess On"
        - "SecResponseBodyAccess On"
        - "Include @owasp_crs/*.conf"

      app1.example.com:
        - "SecRuleEngine On"
        - "SecRequestBodyAccess On"
        - "Include @owasp_crs/*.conf"
        - "SecRuleUpdateActionById 949110 \"deny,status:403\""
        - "SecRuleRemoveById 920420"

      app2.example.com:
        - "SecRuleEngine DetectionOnly"
        - "SecRequestBodyAccess On"
        - "Include @owasp_crs/*.conf"

    default_directives: "default"
    per_authority_directives:
      app1.example.com: "app1.example.com"
      app2.example.com: "app2.example.com"
```

## Rollout Flow

```
[Detection Only] → [Collect Logs] → [Build Suppression Rules] → [Apply + Fix FPs] → [Monitor] → [Enable Blocking]
```

## Lessons Learned

- Always start in `DetectionOnly` — never go straight to blocking on a new domain
- Tune per-domain (`per_authority_directives`), not globally — one app's false positive is another app's real attack signature
- Prefer fixing the request format over suppressing a rule wherever the false positive comes from the app's own request shape, not from CRS being overly strict
- Centralized logging (Elasticsearch/Kibana) is what makes Stage 2 actually feasible at scale — without it, finding false positives in raw pod logs is impractical
