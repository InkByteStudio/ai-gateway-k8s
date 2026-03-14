# ai-gateway-k8s

Reference Kubernetes Gateway API manifests for standing up an AI inference gateway.

This is a reference set of Kubernetes manifests. Apply, adapt, and extend for your cluster. Every file comes from the companion tutorial and is organized for easy adaptation.

## Architecture

```
Clients
  |
  v
Gateway (ai-gateway namespace)
  |  HTTPS listener, tenant hostname matching
  v
HTTPRoutes (tenant namespaces)
  |  Path-based routing, canary splits, timeouts
  v
Model Services (ai-models namespace)
  |  chat-llama3-8b-v1, chat-llama3-8b-v2, embed-bge-small-v1
  v
Inference Pods
```

## What's Included

### manifests/base/
| File | Purpose |
|---|---|
| `namespaces.yaml` | Gateway, model-serving, and tenant namespace definitions |
| `gateway.yaml` | Shared Gateway resource with HTTPS listener |
| `model-services.yaml` | Model-serving Service definitions |
| `referencegrant.yaml` | Cross-namespace ReferenceGrant for tenant routes |
| `tls-certificate.yaml` | cert-manager Certificate (or manual TLS secret stub) |

### manifests/routing/
| File | Purpose |
|---|---|
| `tenant-a-routes.yaml` | Initial HTTPRoute for chat and embeddings |
| `tenant-a-chat-canary.yaml` | Weighted canary split for model version rollout |
| `tenant-a-chat-router.yaml` | Router service pattern for complex fallback |
| `tenant-a-route-timeouts.yaml` | Explicit timeouts for chat vs embeddings |
| `llm-router-service.yaml` | Service stub for the router-pattern backend |

> **Note:** The routing manifests are **mutually exclusive patterns** for the same
> `tenant-a.ai.example.com` hostname and `/v1/chat/completions` path. Apply **one**
> at a time to avoid route conflicts:
> - `tenant-a-routes.yaml` — baseline routing (start here)
> - `tenant-a-route-timeouts.yaml` — adds explicit timeouts (replaces baseline)
> - `tenant-a-chat-canary.yaml` — weighted canary split (requires `chat-llama3-8b-v2` service)
> - `tenant-a-chat-router.yaml` — router-service pattern (requires `llm-router` service)

### manifests/reliability/
| File | Purpose |
|---|---|
| `chat-pdb.yaml` | PodDisruptionBudget for chat-llama3-8b-v1 |
| `chat-v2-pdb.yaml` | PodDisruptionBudget for chat-llama3-8b-v2 |

### manifests/observability/
| File | Purpose |
|---|---|
| `prometheus-rules.yaml` | PrometheusRule recording rules (RPS, error rate, p95 latency) |
| `service-monitor.yaml` | ServiceMonitor for Prometheus Operator scraping |
| `gateway-log-example.json` | Structured log contract with tenant, model, token fields |

### manifests/security/
| File | Purpose |
|---|---|
| `model-networkpolicy.yaml` | NetworkPolicy: only gateway namespace can reach models |
| `tenant-a-quota.yaml` | ResourceQuota for tenant namespace compute limits |
| `rate-limit-example.yaml` | Gateway-layer rate limiting for per-tenant request fairness |
| `audit-log-example.json` | Audit log contract for access decisions |

> **Fairness note:** `tenant-a-quota.yaml` limits compute resources within the tenant
> namespace but does not control request throughput to shared model backends in `ai-models`.
> For gateway-layer rate limiting, see `rate-limit-example.yaml`.

### templates/
| File | Purpose |
|---|---|
| `traffic-profile.yaml` | AI traffic classification template |
| `ai-gateway-responsibilities.yaml` | Gateway scope and responsibility document |

### test/
| File | Purpose |
|---|---|
| `embeddings.json` | Non-streaming test payload |
| `chat-stream.json` | Streaming chat test payload |

## Prerequisites

- Kubernetes cluster (1.28+)
- A Gateway API controller installed (Envoy Gateway, Istio, Cilium, etc.)
- At least one model-serving endpoint deployed
- `kubectl` CLI

## Quick Start

1. Clone this repository
2. Review `templates/traffic-profile.yaml` and `templates/ai-gateway-responsibilities.yaml`
3. Set up TLS: install [cert-manager](https://cert-manager.io/docs/installation/) and apply `manifests/base/tls-certificate.yaml`, or create a manual TLS secret (see comments in that file)
4. Update `gateway.yaml` with your real GatewayClass name
5. Update hostnames and backend service names to match your environment
6. Apply the base manifests:

```bash
kubectl apply -f manifests/base/
```

7. Apply **one** routing pattern (they are mutually exclusive):

```bash
# Option A: baseline routing
kubectl apply -f manifests/routing/tenant-a-routes.yaml

# Option B: routing with explicit timeouts
# kubectl apply -f manifests/routing/tenant-a-route-timeouts.yaml

# Option C: canary split (requires chat-llama3-8b-v2 service)
# kubectl apply -f manifests/routing/tenant-a-chat-canary.yaml

# Option D: router-service pattern (requires llm-router service)
# kubectl apply -f manifests/routing/tenant-a-chat-router.yaml
```

8. Test with the provided payloads:

```bash
export GATEWAY_ADDR=https://YOUR_GATEWAY_ADDRESS

# Use -k if your TLS certificate is self-signed
curl -k -N \
  -H "Host: tenant-a.ai.example.com" \
  -H "Content-Type: application/json" \
  -X POST \
  --data @test/chat-stream.json \
  ${GATEWAY_ADDR}/v1/chat/completions
```

## Using Kustomize

Apply all manifests at once:

```bash
kubectl apply -k manifests/
```

Or build and inspect before applying:

```bash
kustomize build manifests/ | kubectl apply -f -
```

The default kustomization uses baseline routing (`tenant-a-routes.yaml`). To use a different routing pattern, edit `manifests/routing/kustomization.yaml` and swap the resource file.

## Observability Setup

The observability manifests assume [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) is installed.

1. **ServiceMonitor** (`manifests/observability/service-monitor.yaml`): Update `selector.matchLabels` and `endpoints[].port` to match your gateway controller's Service and metrics port. Common mappings:
   - Envoy Gateway: label `app.kubernetes.io/name: envoy`, port `http-metrics`
   - Istio: label `app: istiod`, port `http-monitoring`
   - Cilium: label `k8s-app: cilium-envoy`, port `envoy-metrics`

2. **PrometheusRule** (`manifests/observability/prometheus-rules.yaml`): The recording rules reference metric names (`ai_gateway_requests_total`, `ai_gateway_request_duration_ms_bucket`) that must be mapped to your gateway's actual exported metrics.

3. **Prometheus configuration**: Ensure your Prometheus instance's `spec.ruleSelector` and `spec.serviceMonitorSelector` match the labels on these resources (or use a nil selector to pick up all).

## Adapt for Your Cluster

- **GatewayClass**: Replace `your-gateway-class` in `gateway.yaml` with your controller's class name
- **Hostnames**: Replace `*.ai.example.com` with your real domain
- **Model services**: Replace the example service names and ports with your actual model endpoints
- **Tenants**: Copy the `tenant-a` pattern for additional tenants, adding new namespaces, ReferenceGrants, and HTTPRoutes
- **Timeouts**: Adjust timeout values in `tenant-a-route-timeouts.yaml` based on your model latency characteristics
- **Observability**: Adapt `prometheus-rules.yaml` metric names to match your gateway's exported metrics

## Related

- [Tutorial: Stand Up an AI Gateway on Kubernetes](https://igotasite4that.com/tutorials/ai-gateway-kubernetes-inference-workloads)
- [Blog: Kubernetes AI Workloads and Networking](https://igotasite4that.com/blog/kubernetes-ai-workloads-networking)

## License

MIT
