# LiteLLM Proxy Helm Chart

Helm chart that deploys the [LiteLLM proxy](https://docs.litellm.ai/) — an OpenAI-compatible API gateway that manages keys, enforces budgets, and routes requests to upstream LLM models.

## Architecture

```
Student app ──→ LiteLLM Proxy (port 4000) ──→ vLLM (Socrates)
                  │
                  ├── PostgreSQL (credentials + spend tracking)
                  └── JupyterHub extension (key lifecycle)
```

This is a **meta-chart** that wraps two subcharts:
- **`litellm-helm`** — official LiteLLM proxy chart (from `oci://ghcr.io/berriai/litellm-helm`)
- **`postgresql`** — version-controlled PostgreSQL (image pin, unlike Bitnami's `latest` tag)

The official chart's bundled Bitnami PostgreSQL/Redis dependencies have been removed to avoid image drift.

## Installation

### Option 1: GitHub Pages (recommended)

```bash
helm repo add dive4dec https://dive4dec.github.io/litellm
helm repo update
helm upgrade --install litellm dive4dec/litellm \
  -f your-values.yaml \
  -n litellm-proxy --create-namespace --timeout 5m
```

### Option 2: Git submodule

```bash
git submodule add git@github.com:dive4dec/litellm.git litellm
git submodule update --init --recursive
helm upgrade --install litellm litellm \
  -f your-values.yaml \
  -n litellm-proxy --create-namespace --timeout 5m
```

### Option 3: OCI registry

```bash
helm package litellm --destination ./charts
helm push ./litellm-0.2.0.tgz oci://ghcr.io/your-org/litellm
helm upgrade --install litellm oci://ghcr.io/your-org/litellm/litellm \
  -n litellm-proxy --create-namespace --timeout 5m
```

## Values Reference

### Master Key

The proxy uses a **master key** for admin API operations (key generation, deletion, user management). By default, the chart auto-generates a random key (`sk-` + 18 random chars) on install.

**⚠️ Critical: If the JupyterHub extension manages keys, its `LITELLM_MASTER_KEY` must match the proxy's master key.**

To use a fixed master key:

```yaml
litellm-helm:
  masterkey: "sk-you...-key"
```

Then set the same value in your JupyterHub hub values:

```yaml
hub:
    LITELLM_MASTER_KEY: "sk-you...-key"
```

To extract the current proxy master key from the cluster:

```bash
kubectl -n <litellm-namespace> get secret litellm-masterkey \
  -o go-template='{{index .data "masterkey"}}' | base64 -d
```

### Model Configuration

```yaml
litellm-helm:
  proxy_config:
    model_list:
      - model_name: Socrates
        litellm_params:
          model: openai/Socrates
          api_base: http://vllm.ai-agent.svc.cluster.local:8000/v1
          api_key: <from-vllm-secret>
          max_parallel_requests: 64
          input_cost_per_token: 0.000005
          output_cost_per_token: 0.00001
```

### Database

```yaml
litellm-helm:
  db:
    deployStandalone: false
    useExisting: true
    endpoint: proxy-postgresql
    url: "postgresql://litellm:***@proxy-postgresql/litellm"

postgresql:
  fullnameOverride: "proxy-postgresql"
  credentials:
    database: litellm
    username: litellm
    password: "***"
```

### Key Budget Limits

| Payload Field | Description |
|---|---|
| `max_budget` | Max spend in USD per `budget_duration` window |
| `budget_duration` | Rolling window: `"30s"`, `"15m"`, `"24h"`, `"30d"` |
| `tpm_limit` | Tokens per minute limit |
| `rpm_limit` | Requests per minute limit |
| `duration` | Key lifetime before auto-deactivation |

The proxy enforces `max_budget` as a **rolling window** over `budget_duration`.

### Autoscaling

```yaml
litellm-helm:
  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilizationPercentage: 70
```

## Endpoints

- **API**: OpenAI-compatible endpoint on port 4000
- **Admin**: `http://litellm.<namespace>.svc.cluster.local:4000` (in-cluster)
- **Health**: `GET /health/liveliness`, `GET /health/readyz`

## Upstream

Based on `oci://ghcr.io/berriai/litellm-helm`. Full upstream docs: https://github.com/BerriAI/litellm/blob/main/helm/litellm/README.md
