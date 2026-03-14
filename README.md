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
  |  chat-llama3-8b-v1, embed-bge-small-v1
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

### manifests/routing/
| File | Purpose |
|---|---|
| `tenant-a-routes.yaml` | Initial HTTPRoute for chat and embeddings |
| `tenant-a-chat-canary.yaml` | Weighted canary split for model version rollout |
| `tenant-a-chat-router.yaml` | Router service pattern for complex fallback |
| `tenant-a-route-timeouts.yaml` | Explicit timeouts for chat vs embeddings |

### manifests/reliability/
| File | Purpose |
|---|---|
| `chat-pdb.yaml` | PodDisruptionBudget for model serving |

### manifests/observability/
| File | Purpose |
|---|---|
| `prometheus-rules.yaml` | PrometheusRule recording rules (RPS, error rate, p95 latency) |
| `gateway-log-example.json` | Structured log contract with tenant, model, token fields |

### manifests/security/
| File | Purpose |
|---|---|
| `model-networkpolicy.yaml` | NetworkPolicy: only gateway namespace can reach models |
| `tenant-a-quota.yaml` | ResourceQuota for tenant namespace fairness |
| `audit-log-example.json` | Audit log contract for access decisions |

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
3. Update `gateway.yaml` with your real GatewayClass name and TLS certificate
4. Update hostnames and backend service names to match your environment
5. Apply the base manifests:

```bash
kubectl apply -f manifests/base/
```

6. Apply routing:

```bash
kubectl apply -f manifests/routing/tenant-a-routes.yaml
```

7. Test with the provided payloads:

```bash
export GATEWAY_ADDR=http://YOUR_GATEWAY_ADDRESS

curl -N \
  -H "Host: tenant-a.ai.example.com" \
  -H "Content-Type: application/json" \
  -X POST \
  --data @test/chat-stream.json \
  ${GATEWAY_ADDR}/v1/chat/completions
```

## Adapt for Your Cluster

- **GatewayClass**: Replace `your-gateway-class` in `gateway.yaml` with your controller's class name
- **Hostnames**: Replace `*.ai.example.com` with your real domain
- **Model services**: Replace the example service names and ports with your actual model endpoints
- **Tenants**: Copy the `tenant-a` pattern for additional tenants, adding new namespaces, ReferenceGrants, and HTTPRoutes
- **Timeouts**: Adjust timeout values in `tenant-a-route-timeouts.yaml` based on your model latency characteristics
- **Observability**: Adapt `prometheus-rules.yaml` metric names to match your gateway's exported metrics

## Related

- [Tutorial: Stand Up an AI Gateway on Kubernetes](https://inkbytestudio.com/tutorials/ai-gateway-kubernetes-inference-workloads)
- [Blog: Kubernetes AI Workloads and Networking](https://inkbytestudio.com/blog/kubernetes-ai-workloads-networking)

## License

MIT
