# Gatekeeper & OPA Metrics Exporter Installation

## Install Using Helm

Add the Gatekeeper Helm repository and install Gatekeeper in the `gatekeeper-system` namespace:

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace
```

## Verify Installation

Check that the Gatekeeper pods are running:

```bash
kubectl get pods -n gatekeeper-system
```

## Install OPA Metrics ExporterYa

Apply the OPA metrics exporter manifest:

```bash
kubectl apply -f monitoring/opa-metrices-scorecard.yaml
```

> See: [opa-metrices-scorecard.yaml](monitoring/opa-metrices-scorecard.yaml)
