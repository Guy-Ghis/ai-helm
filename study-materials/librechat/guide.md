# LibreChat — Complete Operational Guide

> **Scope**: Exactly what exists in this repository and what runs in production today. No roadmap items,
> no cross-topic intersections. Every detail is sourced directly from the chart files listed at each section.

---

## Table of Contents

1. [What LibreChat Is in This Repo](#1-what-librechat-is-in-this-repo)
2. [The Chart Family — Files and Their Roles](#2-the-chart-family--files-and-their-roles)
3. [Orchestrator: `charts/librechart/`](#3-orchestrator-chartslibrechart)
4. [Leaf 1: `charts/librechat-search/` — Meilisearch](#4-leaf-1-chartslibrechat-search--meilisearch)
5. [Leaf 2: `charts/librechat-app/` — LibreChat + MongoDB](#5-leaf-2-chartslibrechat-app--librechat--mongodb)
6. [Leaf 3: `charts/librechat-opencode-wellknown/` — nginx + opencode JSON](#6-leaf-3-chartslibrechat-opencode-wellknown--nginx--opencode-json)
7. [Secrets — Every Secret, Where It Lives, What It Contains](#7-secrets--every-secret-where-it-lives-what-it-contains)
8. [Networking — Every Service, Ingress, and Port](#8-networking--every-service-ingress-and-port)
9. [Pod Configuration — Resources, Security, Health Checks](#9-pod-configuration--resources-security-health-checks)
10. [MongoDB — Exact Configuration](#10-mongodb--exact-configuration)
11. [Redis — Exact Configuration](#11-redis--exact-configuration)
12. [Config Delivery — How `librechat.yaml` Gets Into the Pod](#12-config-delivery--how-librechasyaml-gets-into-the-pod)
13. [The Agent Seed Job — Exact Mechanics](#13-the-agent-seed-job--exact-mechanics)
14. [opencode Well-Known — The JSON Document and How It Is Served](#14-opencode-well-known--the-json-document-and-how-it-is-served)
15. [Vanity Domain Redirect](#15-vanity-domain-redirect)
16. [ArgoCD Sync Behaviour](#16-argocd-sync-behaviour)

---

## 1. What LibreChat Is in This Repo

LibreChat (`ghcr.io/danny-avila/librechat`, version `v0.8.7` — set in `global.librechat.version`) is an
open-source self-hosted AI chat application. In this platform it is:

- The primary end-user web interface for AI interactions, reachable at `https://ai.camer.digital`
- A client that calls AI model backends exclusively through the **internal Envoy AI Gateway plane**
- An OpenID Connect relying party against the Keycloak realm `camer-digital`, client name `converse`
- A MongoDB application (no ORM abstraction — it uses Mongoose against MongoDB directly)
- A Redis client for session caching and real-time features

**Image pull policy**: `Always` — every pod restart pulls the tagged image, even if the tag is already present.

---

## 2. The Chart Family — Files and Their Roles

```
charts/
├── librechart/                      ← ORCHESTRATOR
│   ├── Chart.yaml
│   ├── values.yaml                  ← ArgoCD wiring + the 3-child list
│   └── templates/
│       ├── applicationset.yaml      ← The single ApplicationSet CR
│       └── _helpers.tpl             ← ADR-0017 destination guard (hard-fail)
│
├── librechat-search/                ← LEAF 1 (sync-wave -1)
│   ├── Chart.yaml                   ← depends on upstream meilisearch chart
│   └── values.yaml
│
├── librechat-app/                   ← LEAF 2 (sync-wave 0)
│   ├── Chart.yaml                   ← depends on bjw-template 4.6.2 + mongodb 1.7.6
│   ├── values.yaml                  ← 742 lines: every env var, every secret ref
│   ├── files/
│   │   └── seed-agents.js           ← The agent seed script (Node.js)
│   └── templates/
│       ├── _mongo_uri.tpl           ← Builds MongoDB connection string
│       ├── agent-seed-job.yaml      ← PostSync Job + 2 ConfigMaps
│       ├── certificate-internal-ca.yaml ← cert-manager Certificate for the internal CA
│       ├── configmap.yaml           ← Renders the librechat.yaml config
│       ├── externalsecret-app.yaml  ← All app secrets (iterates librechatSecrets.secrets)
│       ├── externalsecret-redis.yaml ← Redis password ExternalSecret
│       ├── externalsecret-s3.yaml   ← S3 credentials ExternalSecret
│       └── pdb.yaml                 ← PodDisruptionBudget for MongoDB
│
└── librechat-opencode-wellknown/    ← LEAF 3 (sync-wave 1)
    ├── Chart.yaml
    ├── values.yaml                  ← 2125 lines: the entire well-known JSON + nginx config
    └── templates/
        └── configmap.yaml           ← Wraps the JSON into a ConfigMap for nginx
```

---

## 3. Orchestrator: `charts/librechart/`

**Source file**: [`charts/librechart/values.yaml`](file:///Users/gisstudent/ai-helm/charts/librechart/values.yaml)

### What the orchestrator does

It renders exactly **one Kubernetes resource**: an `argoproj.io/v1alpha1 ApplicationSet` named `librechat`
(the chart release name). The ApplicationSet lives in the `argocd` namespace on the **admin@homeos**
cluster (the ArgoCD control plane), **not** on `home-remote` where the workloads run.

### ArgoCD wiring (exact values)

| Field | Value |
|---|---|
| `argocd.namespace` | `argocd` |
| `argocd.project` | `ai` |
| `argocd.chartRegistry` | `oci://ghcr.io/adorsys-gis/charts` |
| `argocd.chartVersionRange` | `>=0.0.0` (float — every published version qualifies) |
| `argocd.repoURL` | `https://github.com/adorsys-gis/ai-helm-values` |
| `argocd.targetRevision` | `main` |
| `argocd.env` | `prod` |
| `argocd.destination.name` | `home-remote` |
| `argocd.destination.namespace` | `converse` |
| `argocd.destination.allowInCluster` | `false` |
| `argocd.syncPolicy.automated.prune` | `true` |
| `argocd.syncPolicy.automated.selfHeal` | `true` |
| `argocd.syncPolicy.syncOptions` | `CreateNamespace=true`, `ServerSideApply=true` |

### The three children (exact list)

```yaml
children:
  - name: librechat-search
    chartName: librechat-search
    syncWave: "-1"
    enabled: true

  - name: librechat-app
    chartName: librechat-app
    syncWave: "0"
    enabled: true

  - name: librechat-opencode-wellknown
    chartName: librechat-opencode-wellknown
    syncWave: "1"
    enabled: true
```

Each element becomes one ArgoCD `Application`. The ApplicationSet list generator iterates these three
elements and produces one Application per element. The `chartName` is the OCI artifact name — the
ApplicationSet builds the source URL as `oci://ghcr.io/adorsys-gis/charts/<chartName>`.

Every child Application also has a **second source** that points at `ai-helm-values`
(`https://github.com/adorsys-gis/ai-helm-values @ main`) with `helm.valueFiles` pointing at
`environments/prod/values/<child-name>.yaml`. This is the `$values` pattern (ADR-0087).
`ignoreMissingValueFiles: true` is set on all children, so a missing values file is not fatal.

### The ADR-0017 destination guard

The `_helpers.tpl` in `charts/librechart/` contains a Helm hard-fail guard that aborts rendering
if the `argocd.destination` resolves to the in-cluster API server (`https://kubernetes.default.svc`
or the name `in-cluster`) and `allowInCluster` is not explicitly `true`. This prevents workloads
from accidentally being directed to the ArgoCD control-plane cluster.

---

## 4. Leaf 1: `charts/librechat-search/` — Meilisearch

**Sync-wave**: `-1` (deploys before LibreChat — search must be ready when LibreChat starts).

Meilisearch is a **dependency subchart** — `librechat-search` has no own templates. Its
`Chart.yaml` declares a dependency on the upstream `meilisearch` Helm chart.

Meilisearch exposes its HTTP API on **port 7700**. The Service name follows the chart's release
name: `librechat-search`. LibreChat connects to it via `MEILI_HOST=http://librechat-search:7700`
(plain HTTP, in-namespace, no TLS needed).

The **Meilisearch master key** is not in `charts/librechat-search/values.yaml`; it is in the Secret
`librechat-meili-config` (key `MEILI_MASTER_KEY`), provisioned by the ExternalSecret in
`charts/librechat-app/templates/externalsecret-app.yaml`. LibreChat reads it as `MEILI_MASTER_KEY`.
Meilisearch reads it via its own env config (the meilisearch chart mounts it or sets it as env).

---

## 5. Leaf 2: `charts/librechat-app/` — LibreChat + MongoDB

**Sync-wave**: `0`.

**Source file**: [`charts/librechat-app/values.yaml`](file:///Users/gisstudent/ai-helm/charts/librechat-app/values.yaml)

### 5.1 Chart dependencies

```yaml
# Chart.yaml
dependencies:
  - name: bjw-template          # aliased as `librechat`
    version: '4.6.2'
    alias: librechat

  - name: mongodb                # aliased as `db`
    version: "1.7.6"
    repository: https://repo.helmforge.dev
    alias: db

  - name: common                 # Bitnami helpers (labels, namespace, etc.)
    version: '*'
```

The `bjw-template` subchart (`alias: librechat`) renders the LibreChat Deployment, Service, HPA,
NetworkPolicy, and the `config-rollout` ConfigMap. The `mongodb` subchart (`alias: db`) renders the
MongoDB StatefulSet, headless Service, and PersistentVolumeClaim. The `common` chart provides label
helpers used in the parent-level templates.

### 5.2 The LibreChat Deployment

Rendered entirely by `bjw-template` via `.Values.librechat.controllers.librechat`:

- **Type**: `deployment`
- **Strategy**: `RollingUpdate`
- **Replicas**: `2` (baseline; HPA can scale this)
- **Image**: `ghcr.io/danny-avila/librechat:v0.8.7`, `pullPolicy: Always`
- **podDisruptionBudget**: `minAvailable: 1` (set inline on the controller, not via the standalone `pdb.yaml` which covers MongoDB)

### 5.3 All environment variables — exact values

Every environment variable set on the `librechat` container:

| Variable | Value / Source |
|---|---|
| `PUID` | `"1000"` |
| `PGID` | `"1000"` |
| `TZ` | `"Europe/Berlin"` |
| `NODE_EXTRA_CA_CERTS` | `/etc/internal-ca/ca.crt` (mounted from Secret `librechat-internal-ca`) |
| `ALLOW_EMAIL_LOGIN` | `"false"` |
| `ALLOW_REGISTRATION` | `"false"` |
| `ALLOW_SOCIAL_LOGIN` | `"true"` |
| `ALLOW_SOCIAL_REGISTRATION` | `"true"` |
| `REFRESH_TOKEN_EXPIRY` | `5184000000` (60 days in ms) |
| `OPENID_ISSUER` | `https://auth.verif.fyi/realms/camer-digital` |
| `OPENID_CALLBACK_URL` | `/oauth/openid/callback` |
| `OPENID_SCOPE` | `openid profile email librechat offline_access` |
| `OPENID_REQUIRED_ROLE_TOKEN_KIND` | `access` |
| `OPENID_REQUIRED_ROLE_PARAMETER_PATH` | `librechat_roles` |
| `OPENID_AUTO_REDIRECT` | `"true"` |
| `OPENID_REUSE_TOKENS` | `"true"` |
| `OPENID_AUDIENCE` | `converse` |
| `DEBUG_OPENID_REQUESTS` | `"true"` |
| `OPENID_USE_END_SESSION_ENDPOINT` | `"true"` |
| `LOGIN_MAX` | `"50"` |
| `LOGIN_WINDOW` | `"1"` |
| `DOMAIN_CLIENT` | `https://ai.camer.digital` |
| `DOMAIN_SERVER` | `https://ai.camer.digital` (YAML anchor reuse) |
| `NO_INDEX` | `"true"` |
| `USE_REDIS` | `"true"` |
| `REDIS_KEY_PREFIX` | `"librechat-prod-v2"` |
| `REDIS_URI` | `rediss://redis-ha-haproxy.redis-system.svc.cluster.local:6379` |
| `REDIS_CA` | `/etc/internal-ca/ca.crt` |
| `REDIS_PASSWORD` | from Secret `redis-ha-redis-auth`, key `redis-password` |
| `FORCED_IN_MEMORY_CACHE_NAMESPACES` | `APP_CONFIG,CONFIG_STORE,STARTUP_CONFIG` |
| `SEARCH` | `true` |
| `MEILI_HOST` | `http://librechat-search:7700` |
| `MEILI_MASTER_KEY` | from Secret `librechat-meili-config`, key `MEILI_MASTER_KEY` |
| `MONGO_URI` | built by `_mongo_uri.tpl` (see §10) |
| `CREDS_KEY` | from Secret `librechat-config`, key `creds_key` |
| `CREDS_IV` | from Secret `librechat-config`, key `creds_iv` |
| `JWT_SECRET` | from Secret `librechat-config`, key `jwt_secret` |
| `JWT_REFRESH_SECRET` | from Secret `librechat-config`, key `jwt_refresh_secret` |
| `CONVERSE_OPENAI_API_KEY` | from Secret `librechat-main-config`, key `converse_openai_api_key` |
| `CD_MCP_CLIENT_ID` | from Secret `librechat-mcp-cd-credentials`, key `client_id` |
| `CD_MCP_SECRET_ID` | from Secret `librechat-mcp-cd-credentials`, key `client_secret` |
| `GITHUB_MCP_CLIENT_ID` | from Secret `librechat-mcp-github`, key `client_id` |
| `GITHUB_MCP_SECRET_ID` | from Secret `librechat-mcp-github`, key `client_secret` |
| `CODERS_MCP_CLIENT_ID` | from Secret `librechat-mcp-coder-credentials`, key `client_id` |
| `CODERS_MCP_SECRET` | from Secret `librechat-mcp-coder-credentials`, key `client_secret` |
| `GITHUB_SKILLS_TOKEN` | from Secret `librechat-skills-sync`, key `github_token` |
| `LIBRECHAT_CODE_API_KEY` | from Secret `librechat-code-config`, key `code_api_key` |
| `OPENID_CLIENT_ID` | from Secret `librechat-openid-config`, key `client_id` |
| `OPENID_CLIENT_SECRET` | from Secret `librechat-openid-config`, key `client_secret` |
| `OPENID_SESSION_SECRET` | from Secret `librechat-openid-config`, key `session_secret` |
| `SERPER_API_KEY` | from Secret `librechat-websearch-config`, key `serper_api_key` |
| `FIRECRAWL_API_KEY` | from Secret `librechat-websearch-config`, key `firecrawl_api_key` |
| `JINA_API_KEY` | from Secret `librechat-websearch-config`, key `jina_api_key` |
| `IMAGE_GEN_OAI_API_KEY` | from Secret `librechat-main-config`, key `converse_openai_api_key` |
| `IMAGE_GEN_OAI_BASEURL` | `https://core-gateway-internal.envoy-gateway-system.svc.cluster.local/v1` |
| `IMAGE_GEN_OAI_MODEL` | `gemini-2.5-flash-image` |
| `AWS_REGION` | from Secret `librechat-s3-config`, key `s3_region_name` |
| `AWS_ACCESS_KEY_ID` | from Secret `librechat-s3-config`, key `s3_access_key_id` |
| `AWS_SECRET_ACCESS_KEY` | from Secret `librechat-s3-config`, key `s3_secret_access_key` |
| `AWS_BUCKET_NAME` | from Secret `librechat-s3-config`, key `s3_bucket_name` |
| `AWS_ENDPOINT_URL` | `https://nbg1.your-objectstorage.com` |
| `AWS_FORCE_PATH_STYLE` | `"true"` |

**RAG API** (`rag-api` controller): disabled (`enabled: false`). The controller block exists in
`values.yaml` lines 558-616 but renders nothing because `bjw-template` skips disabled controllers.

---

## 6. Leaf 3: `charts/librechat-opencode-wellknown/` — nginx + opencode JSON

**Sync-wave**: `1`.

This leaf deploys a **static nginx pod** that serves one JSON document at the path
`/opencode/.well-known/opencode` (or equivalently just the `/` since nginx is configured with a
single root). The URL that external clients hit is `https://ai.camer.digital/opencode/.well-known/opencode`.
When a developer runs `opencode auth login https://ai.camer.digital/opencode`, the opencode CLI
fetches this URL and bootstraps the user's full development environment.

**Key sizes**: `values.yaml` is 2125 lines / 110KB. The entire content is the JSON payload expressed
as YAML, plus the nginx configuration. There are no own Kubernetes Deployment fields beyond what
`bjw-template` provides — everything is in `values.yaml`.

### What the JSON document contains (the `wellKnown:` key)

```yaml
wellKnown:
  auth:
    command: ["sh", "-c", "echo plugin-managed"]   # stub — plugin overrides auth
    env: OPENAI_API_KEY

  config:
    $schema: https://opencode.ai/config.json
    default_agent: assistant
    plugin:
      - "@vymalo/opencode-oauth2@0.12.0"
      - "@vymalo/opencode-models-info@0.12.0"
      - "@vymalo/opencode-ratelimit@0.12.0"
      - ["@vymalo/opencode-browser@0.12.0", {port: 4517, groups: [page, control, debug, interactive]}]
      - "opencode-skills-collection@4.0.14"
    provider:
      camer-digital:
        name: "Camer Digital"
        options:
          baseURL: "https://api.ai.camer.digital/v1"
          headerTimeout: 90000   # ms — covers the max 65s burst-wait
          chunkTimeout: 90000
          oauth2:
            issuer: "https://auth.verif.fyi/realms/camer-digital"
            clientId: "opencode-cli"
            scopes: [openid, profile, offline_access]
            authFlow: device_code
            syncIntervalMinutes: 60
            responseApi: false   # disabled 2026-06-13; uses Chat Completions path
          meta:
            modelsInfoUrl: "models/info"
            modelsInfoHideUnmatched: true
            modelsInfoOverwrite: [name]
            rateLimit:
              enabled: true
              scope: model
              headerPrefix: "x-ratelimit"
              tiers:
                - maxResetSeconds: 120
                  action: wait
                  maxWaitMs: 65000
                  maxRetries: 3
                - maxResetSeconds: null
                  action: error
```

**Models with `reasoningEffort: low` default** (plus a `thinking` variant at `high`):
`qwen3-4b-local`, `qwen3-5-4b-local`, `qwen3-8b-local`, `deepseek-v4-flash-0731`, `reviewer`,
`adorsys-planner`, `adorsys-planner-pro`, `kimi-k2.5`, `minimax-m2p5`, `glm-5p2`,
`qwen3p7-plus`, `gemini-3p1-flash-lite`, `glm-4.7-flash`, `minimax-m2.7`, `mimo-v2p5`,
`ornith-1p0-35b`, `adorsys-researcher`, `adorsys-coder`, `adorsys-coder-pro`, `adorsys-reviewer`,
`adorsys-reviewer-pro`, `adorsys-frontend`, `mimo-v2p5-pro`, `minimax-m3`.

**MCP servers**: All configured with `enabled: false` (opt-in). Remote servers:
`brave`, `context7`, `refero`, `firecrawl`, `terraform` — all route through `api.ai.camer.digital/mcp/<name>`
with the `opencode-cli` OAuth client. Local servers (launched via npx by the opencode CLI, not through
the gateway): `memory`, and others.

---

## 7. Secrets — Every Secret, Where It Lives, What It Contains

All secrets are provisioned by **External Secrets Operator** pulling from the `ssegning-aws`
`ClusterSecretStore`. The refresh interval for all is **1 hour** unless noted.

### 7.1 Secrets created by `externalsecret-app.yaml`

This template iterates `librechatSecrets.secrets` and emits one `ExternalSecret` per entry.
Source path: `ai/camer/digital/prod/env` for all of these.

| Secret Name | Keys | Property in SM |
|---|---|---|
| `librechat-main-config` | `converse_openai_api_key` | `converse_openai_api_key` |
| `librechat-config` | `creds_key` | `librechat_config_creds_key` |
| | `creds_iv` | `librechat_config_creds_iv` |
| | `jwt_secret` | `librechat_config_jwt_secret` |
| | `jwt_refresh_secret` | `librechat_config_jwt_refresh_secret` |
| `librechat-openid-config` | `client_id` | `librechat_openid_config_client_id` |
| | `client_secret` | `librechat_openid_config_client_secret` |
| | `session_secret` | `librechat_openid_config_session_secret` |
| `librechat-meili-config` | `MEILI_MASTER_KEY` | `librechat_meili_config_meili_master_key` |
| `librechat-mcp-cd-credentials` | `client_id` | `librechat_mcp_cd_credentials_client_id` |
| | `client_secret` | `librechat_mcp_cd_credentials_client_secret` |
| `librechat-mcp-github` | `client_id` | `librechat_mcp_github_client_id` |
| | `client_secret` | `librechat_mcp_github_client_secret` |
| `librechat-mcp-coder-credentials` | `client_id` | `librechat_mcp_coder_credentials_client_id` |
| | `client_secret` | `librechat_mcp_coder_credentials_client_secret` |
| `librechat-websearch-config` | `serper_api_key` | `librechat_websearch_config_serper_api_key` |
| | `firecrawl_api_key` | `librechat_websearch_config_firecrawl_api_key` |
| | `jina_api_key` | `librechat_websearch_config_jina_api_key` |
| `librechat-skills-sync` | `github_token` | `librechat_skills_sync_github_token` |
| `librechat-code-config` | `code_api_key` | `librechat_code_api_key` |

### 7.2 Secret created by `externalsecret-redis.yaml`

| Secret Name | Key | SM path | SM property |
|---|---|---|---|
| `redis-ha-redis-auth` | `redis-password` | `prod/meta/test-app` | `redis_password` |

This is a **cross-namespace copy**: the shared redis-ha password lives in `redis-system`; each
consumer namespace needs its own `redis-ha-redis-auth` Secret. The ExternalSecret in
`charts/librechat-app` provisions the copy in the `converse` namespace.

### 7.3 Secret created by `externalsecret-s3.yaml`

Target secret: `librechat-s3-config`. Built with a **template** (ESO `target.template`) that merges
two dynamic fetched values with two static literal values:

| Key in Secret | Source |
|---|---|
| `s3_region_name` | static literal `"us-east-1"` (templated from `s3ExternalSecret.region`) |
| `s3_bucket_name` | static literal `"ssegning-k8s-state"` (templated from `s3ExternalSecret.bucket`) |
| `s3_access_key_id` | SM `prod/meta/test-app` property `s3_backup_cnpg_client_id` |
| `s3_secret_access_key` | SM `prod/meta/test-app` property `s3_backup_cnpg_secret` |

The Hetzner Object Storage endpoint (`nbg1.your-objectstorage.com`) is **not** in the Secret — it
is hardcoded as `AWS_ENDPOINT_URL` env var. LibreChat namespaces its uploads under `images/` and
`files/` within the shared `ssegning-k8s-state` bucket.

### 7.4 Secret created by `certificate-internal-ca.yaml`

Name: `librechat-internal-ca`. This is a **cert-manager Certificate** issued from the `self-signed-ca`
ClusterIssuer. The resulting Kubernetes Secret contains the standard cert-manager fields:
`tls.crt`, `tls.key`, and `ca.crt`. Only `ca.crt` is used — it carries the **Home Root CA**
(the "Home SSegning Root CA"), which is also the CA signing redis-ha's server cert and the
Envoy AI Gateway's internal listener cert. The certificate itself is a throwaway leaf with:

- `commonName`: `librechat-internal-ca`
- `dnsNames`: `[librechat-internal-ca]`
- `issuerRef.name`: `self-signed-ca`
- `issuerRef.kind`: `ClusterIssuer`

The Secret is then mounted into the LibreChat pod at `/etc/internal-ca` (read-only). Two env vars
point at `ca.crt` in that mount: `NODE_EXTRA_CA_CERTS` and `REDIS_CA`.

---

## 8. Networking — Every Service, Ingress, and Port

### 8.1 Services

| Service | Type | Port | Target |
|---|---|---|---|
| `librechat-app` (from bjw-s) | ClusterIP | `3080` | LibreChat container port `3080` |
| `librechat-app-db` (headless) | ClusterIP / None | `27017` | MongoDB pod port `27017` |
| `librechat-search` | ClusterIP | `7700` | Meilisearch container port `7700` |
| opencode wellknown service | ClusterIP | `80` | nginx container port `80` |

### 8.2 Main Ingress — `ai.camer.digital`

- **IngressClassName**: `traefik`
- **Host**: `ai.camer.digital`
- **Path**: `/`, `pathType: Prefix`
- **Backend service**: `librechat-app`, port `http` (3080)
- **TLS**: `secretName: ai.camer.digital-tls`, issued by ClusterIssuer `cert-home-cert-http` (ACME HTTP-01)
- **Annotations**:
  - `cert-manager.io/cluster-issuer: cert-home-cert-http`
  - `gethomepage.dev/enabled: "true"`
  - `gethomepage.dev/name: "LibreChat"`
  - `gethomepage.dev/group: "Chat"`
  - `gethomepage.dev/icon: "librechat.png"`

### 8.3 NetworkPolicy

bjw-s renders a `NetworkPolicy` named `librechat-librechat` for the librechat controller. Its rules:

```yaml
policyTypes: [Ingress, Egress]
rules:
  ingress: [{}]   # allow all ingress (no IP/port restrictions)
  egress:  [{}]   # allow all egress
```

This is a permissive policy — it satisfies the Cilium baseline that requires every workload to have
a NetworkPolicy CR, without actually restricting traffic. The enforcement of network segmentation
is handled by the Cilium CNI policy layer, not this specific policy.

### 8.4 HPA

Managed as a `rawResource` (bjw-s `rawResources.hpa`) with `forceRename: '{{ .Release.Name }}'`
to keep the HPA named `librechat-app` instead of getting the bjw-s identifier suffix.

- **apiVersion**: `autoscaling/v2`
- **scaleTargetRef**: `apps/v1 Deployment/librechat-app`
- **minReplicas**: `1`
- **maxReplicas**: `4`
- **Metric 1**: `Resource cpu`, `type: Utilization`, `averageUtilization: 70`
- **Metric 2**: `Resource memory`, `type: Utilization`, `averageUtilization: 80`

---

## 9. Pod Configuration — Resources, Security, Health Checks

### 9.1 LibreChat container resources

```yaml
limits:
  cpu: 1000m
  memory: 2Gi
requests:
  cpu: 500m
  memory: 1Gi
```

### 9.2 LibreChat container security context

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  seccompProfile:
    type: RuntimeDefault
```

`readOnlyRootFilesystem` is **intentionally absent**. LibreChat writes winston logs and upload
staging files to its container filesystem at runtime. Setting this would crashloop the pod.
This is acknowledged as a known Trivy KSV-0014 suppression in `.trivyignore.yaml`.

### 9.3 Pod-level security context

```yaml
defaultPodOptions:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
```

The LibreChat image's `USER` is `node` (uid/gid 1000) from the `node:24-alpine` base. `fsGroup: 1000`
ensures the mounted config ConfigMap and internal CA Secret are group-owned by gid 1000.

### 9.4 Health checks (probes)

All three probe types (liveness, readiness, startup) hit `GET /health` over HTTP on port `http` (3080).

- **Liveness**: `httpGet /health`
- **Readiness**: `httpGet /health`
- **Startup**: `httpGet /health`, `failureThreshold: 30`, `periodSeconds: 5` (allows up to 150 seconds for cold startup)

### 9.5 Volumes and mounts

| Volume Name | Type | Source |
|---|---|---|
| `config` | ConfigMap | `librechat-config` (contains key `librechat` = the YAML config string) |
| `internal-ca` | Secret | `librechat-internal-ca` (cert-manager Certificate) |

**Mount details**:

- `config` → `/app/librechat.yaml`, subPath `librechat`, readOnly — LibreChat reads this on startup
- `internal-ca` → `/etc/internal-ca`, readOnly — `ca.crt` trusted by `NODE_EXTRA_CA_CERTS` and `REDIS_CA`

The `subPath` mount is critical: it mounts only the single key `librechat` from the ConfigMap as
a file, not the whole ConfigMap directory. This means **the file does NOT auto-update** when the
ConfigMap changes — the pod must restart to pick up config changes.

---

## 10. MongoDB — Exact Configuration

**Source**: `db:` block in [`charts/librechat-app/values.yaml`](file:///Users/gisstudent/ai-helm/charts/librechat-app/values.yaml#L698-L742)

| Setting | Value |
|---|---|
| `db.enabled` | `true` |
| `db.auth.enabled` | `false` (no MongoDB authentication) |
| `db.persistence.size` | `30Gi` |
| `db.persistence.storageClass` | `~` (cluster default) |
| `db.architecture` | `standalone` |
| `db.replicaSet.name` | `rs0` |
| `db.replicaSet.members` | `1` |

### MongoDB connection string (rendered by `_mongo_uri.tpl`)

The template at [`charts/librechat-app/templates/_mongo_uri.tpl`](file:///Users/gisstudent/ai-helm/charts/librechat-app/templates/_mongo_uri.tpl) builds the URI as:

```
mongodb://<Release.Name>-db-0.<Release.Name>-db-headless:27017
```

With `Release.Name = librechat-app` and `replicaSet.members = 1`, this renders to:

```
mongodb://librechat-app-db-0.librechat-app-db-headless:27017
```

It connects directly to the **headless Service DNS entry for pod 0** (the StatefulSet pod), not to
the `librechat-app-db` ClusterIP Service. This bypasses the cluster's kube-proxy routing and goes
directly to the pod's IP. With a single-member replica set (`members: 1`), the URI contains exactly
one host.

### MongoDB security context

```yaml
podSecurityContext:
  fsGroup: 999
  runAsNonRoot: true
  runAsUser: 999
  runAsGroup: 999
  seccompProfile:
    type: RuntimeDefault

securityContext:
  runAsNonRoot: true
  runAsUser: 999
  runAsGroup: 999
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true    # ← MongoDB CAN use read-only rootfs
  capabilities:
    drop: [ALL]
  seccompProfile:
    type: RuntimeDefault
```

MongoDB runs as uid/gid 999 (its own `mongodb` user in the image). Because `readOnlyRootFilesystem: true`
is set, the chart must provide a writable `/tmp` for the Unix socket file (`/tmp/mongodb-27017.sock`).
This is done via:

```yaml
extraVolumes:
  - name: tmp
    emptyDir: {}
extraVolumeMounts:
  - name: tmp
    mountPath: /tmp
```

### PodDisruptionBudget for MongoDB

From [`charts/librechat-app/templates/pdb.yaml`](file:///Users/gisstudent/ai-helm/charts/librechat-app/templates/pdb.yaml):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: librechat-app-db-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: mongodb
      app.kubernetes.io/instance: librechat-app
      app.kubernetes.io/name: db
```

---

## 11. Redis — Exact Configuration

LibreChat connects to the **shared** `redis-ha` cluster deployed by `home-os`. It does **not** own
a Redis instance.

| Setting | Value |
|---|---|
| `USE_REDIS` | `"true"` |
| `REDIS_KEY_PREFIX` | `"librechat-prod-v2"` |
| `REDIS_URI` | `rediss://redis-ha-haproxy.redis-system.svc.cluster.local:6379` |
| `REDIS_CA` | `/etc/internal-ca/ca.crt` |
| `REDIS_PASSWORD` | from Secret `redis-ha-redis-auth`, key `redis-password` |
| `FORCED_IN_MEMORY_CACHE_NAMESPACES` | `APP_CONFIG,CONFIG_STORE,STARTUP_CONFIG` |

**Critical connection details**:

- The scheme is `rediss://` (double-s = TLS). Redis-HA is TLS-only; a plaintext `redis://` connection
  is reset by the server.
- The connection targets `redis-ha-haproxy` (the HAProxy master-router Service in namespace `redis-system`),
  NOT `redis-ha-redis` (the round-robin Service). The round-robin Service routes to both master and
  replicas — hitting a replica with a write causes `READONLY You can't write against a read only replica`.
  HAProxy health-checks both pods over TLS and routes all connections to whichever is `role:master`,
  following Sentinel failover automatically.
- `FORCED_IN_MEMORY_CACHE_NAMESPACES` forces three namespaces to use the local in-process cache
  instead of Redis: `APP_CONFIG`, `CONFIG_STORE`, `STARTUP_CONFIG`. This prevents these frequently-read
  configs from round-tripping to Redis on every request.

---

## 12. Config Delivery — How `librechat.yaml` Gets Into the Pod

### The full chain

1. **Source**: The `config:` YAML object lives in `ai-helm-values` at
   `environments/prod/values/librechat-app.yaml` (private repo, not visible here).

2. **Injection**: The `librechart` ApplicationSet adds `ai-helm-values@main` as a second Helm source
   for the `librechat-app` child Application, with `helm.valueFiles` set to
   `environments/prod/values/librechat-app.yaml`. This merges the `config:` key into the chart's
   values at render time.

3. **Render**: [`templates/configmap.yaml`](file:///Users/gisstudent/ai-helm/charts/librechat-app/templates/configmap.yaml)
   checks `{{ with .Values.config }}` — if the key is non-empty, it renders:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: librechat-config         # from global.configmap.name
     namespace: <release-namespace>
   data:
     librechat: |
       <the config: object, YAML-serialized and indented>
   ```

   If `config:` is absent or empty (e.g. the values file is missing), the `{{ with }}` guard renders
   nothing — an empty ConfigMap is not emitted. This fails closed: LibreChat starts with no model
   endpoints rather than stale ones.

4. **Mount**: bjw-s persistence mounts the ConfigMap into the pod:

   ```yaml
   persistence:
     config:
       type: configMap
       name: librechat-config
       advancedMounts:
         librechat:
           librechat:
             - path: /app/librechat.yaml
               subPath: librechat    # single-key mount — NOT a directory
               readOnly: true
   ```

5. **Rollout trigger**: Because `subPath` mounts do not auto-update when the ConfigMap changes,
   a marker ConfigMap `librechat-app-config-rollout` is used:

   ```yaml
   configMaps:
     config-rollout:
       suffix: config-rollout        # → name: librechat-app-config-rollout
       includeChecksumInControllers:
         - librechat                 # → bjw-s stamps checksum/configMaps annotation
       data:
         generation: "1"             # bump this in the same commit as any config change
   ```

   When `data.generation` changes, bjw-s recomputes the annotation on the Deployment pod template,
   causing a rolling restart. The annotation key is `checksum/configMaps`, value is a SHA256 of all
   managed ConfigMap data. Without bumping `generation`, a config change in `ai-helm-values` is silently
   not picked up until the next pod restart from any other cause.

---

## 13. The Agent Seed Job — Exact Mechanics

**Source**: [`charts/librechat-app/templates/agent-seed-job.yaml`](file:///Users/gisstudent/ai-helm/charts/librechat-app/templates/agent-seed-job.yaml) +
[`charts/librechat-app/files/seed-agents.js`](file:///Users/gisstudent/ai-helm/charts/librechat-app/files/seed-agents.js)

Guarded by `agentSeed.enabled` (default `false`; actual fleet enabled in `ai-helm-values`).

### What it renders when enabled

Three Kubernetes resources:

1. **ConfigMap** `librechat-app-agent-seed-script` — contains the content of `files/seed-agents.js`
   under key `seed-agents.js`.

2. **ConfigMap** `librechat-app-agent-seed-fleet` — contains the JSON of `agentSeed.agents` array
   under key `fleet.json`.

3. **Job** `librechat-app-agent-seed` — runs `node /seed/seed-agents.js` in the LibreChat container
   image (same image as the app, defaults to `ghcr.io/danny-avila/librechat:v0.8.7`).

### Job annotations

```yaml
argocd.argoproj.io/hook: PostSync
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

`PostSync` means it runs after the LibreChat Application sync is complete and the Application is
Healthy. `BeforeHookCreation` means the previous Job is deleted before a new one is created —
every sync re-runs the seed (it is idempotent).

### The seed script's operation

Environment variables the Job passes to the container:

| Variable | Source |
|---|---|
| `NODE_PATH` | `/app/node_modules` (uses LibreChat's own installed modules) |
| `MONGO_URI` | built directly in the template (same formula as `_mongo_uri.tpl`) |
| `JWT_SECRET` | from Secret `librechat-config`, key `jwt_secret` |
| `LIBRECHAT_URL` | defaults to `http://librechat-app.<namespace>.svc.cluster.local:3080` |
| `PLATFORM_USER_EMAIL` | `agentSeed.platformUserEmail` (required, e.g. `platform@ai.camer.digital`) |
| `FLEET_PATH` | `/fleet/fleet.json` |

The script is two-phase:
1. It connects to MongoDB directly to find the platform user by email and generate a JWT for that user.
2. It uses LibreChat's own REST API (`LIBRECHAT_URL`) to GET each agent by name and then PATCH
   (update) or POST (create) it. This is idempotent — running twice produces the same result.

Volumes:
- `seed-script` → `/seed` (mounts the `agent-seed-script` ConfigMap)
- `seed-fleet` → `/fleet` (mounts the `agent-seed-fleet` ConfigMap)

**Checksum**: The pod template carries:
```yaml
annotations:
  checksum/seed: <sha256 of seed-agents.js + fleet JSON>
```
This causes the Job pod spec to change (and the hook to fire fresh) when either the script or the
fleet definition changes.

**Job spec**: `backoffLimit: 3`, `ttlSecondsAfterFinished: 600`, `restartPolicy: Never`.

---

## 14. opencode Well-Known — The JSON Document and How It Is Served

The nginx pod at Leaf 3 serves the well-known document. Here is the exact mechanism:

1. `values.yaml` contains the `wellKnown:` key — a YAML object representing the full JSON payload.
2. `templates/configmap.yaml` renders this into a ConfigMap where `data.content` is the JSON string
   (serialized via Helm's `toJson` or `toYaml` pipeline).
3. nginx is configured (via its own config in the chart) to serve the content of this ConfigMap key
   at the well-known path.
4. The nginx pod runs with 2 replicas for availability.

### OAuth2 plugin wiring (the key to how auth works in opencode)

The `@vymalo/opencode-oauth2@0.12.0` plugin manages authentication completely:

- The `auth.command` stub in the JSON (`["sh", "-c", "echo plugin-managed"]`) satisfies opencode's
  schema check but is never actually executed — the plugin's `chat.headers` hook overrides the
  `Authorization` header on every request before it is sent.
- The plugin uses the **device_code** OAuth2 flow against Keycloak (`authFlow: device_code`). This
  is deliberate: `authorization_code` flow requires binding a local callback port, which fails in
  headless or remote development environments. Device code requires only browser access.
- The Keycloak client is `opencode-cli`. Scopes: `openid profile offline_access`.
- Token synchronization interval: 60 minutes (`syncIntervalMinutes: 60`).
- The plugin caches the access token and refreshes it via the `offline_access`/refresh token.

### Rate limit awareness (`@vymalo/opencode-ratelimit@0.12.0`)

When a request gets a 429 response, the plugin reads the `x-ratelimit-reset` header from the
Envoy AI Gateway's response. The `tiers` configuration determines what to do:

- If reset ≤ 120 seconds: it is a per-minute burst bucket reset. The plugin **waits** up to 65 seconds
  (`maxWaitMs: 65000`) and retries up to 3 times (`maxRetries: 3`).
- If reset > 120 seconds (or null): it is a monthly budget reset. The plugin **errors immediately**
  (`action: error`) so the 429 surfaces to the user right away instead of freezing the session.

`scope: model` keys rate-limit state per-model (keyed on `x-ai-eg-model`), matching the per-model
`BackendTrafficPolicy` that the Envoy AI Gateway enforces.

---

## 15. Vanity Domain Redirect

Two `rawResources` in `charts/librechat-app/values.yaml` (lines 166-210):

### Traefik Middleware: `kivoyo-redirect`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: kivoyo-redirect
spec:
  redirectRegex:
    regex: "^https?://ai\\.kivoyo\\.com/(.*)"
    replacement: "https://ai.camer.digital/${1}"
    permanent: false    # HTTP 302 (not 301) — temporary redirect
```

The `forceRename: kivoyo-redirect` override ensures the name is exactly `kivoyo-redirect` regardless
of how many rawResources are in the chart (bjw-s would otherwise prefix with the release name and
append the identifier).

### Ingress: `librechat-kivoyo-redirect`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: librechat-kivoyo-redirect
  annotations:
    cert-manager.io/cluster-issuer: cert-home-cert-http
    traefik.ingress.kubernetes.io/router.middlewares: '<namespace>-kivoyo-redirect@kubernetescrd'
spec:
  ingressClassName: traefik
  tls:
    - hosts: [ai.kivoyo.com]
      secretName: ai.kivoyo.com-tls
  rules:
    - host: ai.kivoyo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: librechat-app
                port:
                  number: 3080
```

The backend Service (`librechat-app:3080`) is never reached in practice — Traefik fires the
`redirectRegex` middleware before the request gets to the backend. The backend spec exists only
because the Kubernetes Ingress spec requires one. The `traefik.ingress.kubernetes.io/router.middlewares`
annotation references the Middleware CR using the full Traefik notation:
`<namespace>-kivoyo-redirect@kubernetescrd`.

TLS for `ai.kivoyo.com` is issued by `cert-home-cert-http` (ACME HTTP-01). ACME HTTP-01 challenges
come in as exact paths `/.well-known/acme-challenge/<token>` — Traefik router priority places these
higher than the host-only redirect, so cert-manager can complete the challenge and issue the cert.

---

## 16. ArgoCD Sync Behaviour

### Sync wave ordering

| Wave | Application | What happens |
|---|---|---|
| `-1` | `librechat-search` | Meilisearch deploys first |
| `0` | `librechat-app` | LibreChat + MongoDB deploy |
| `1` | `librechat-opencode-wellknown` | nginx deploys last |

### Automated sync with prune and self-heal

```yaml
automated:
  prune: true       # resources in the cluster but not in the chart are deleted
  selfHeal: true    # manual changes to the cluster are reverted
```

### Server-side apply

`syncOptions: [ServerSideApply=true]` — ArgoCD uses `kubectl apply --server-side` for all resources.
This is required for large resources (e.g. the opencode well-known ConfigMap which is over 100KB)
and for proper management of CRDs with complex merge strategies.

### Namespace creation

`syncOptions: [CreateNamespace=true]` — ArgoCD creates the `converse` namespace if it does not
exist. The namespace itself is not part of any chart.

### The PostSync hook for agent seeding

ArgoCD runs the `librechat-app-agent-seed` Job **after** the sync is complete and the Application
is Healthy (all pods are ready). The Job uses `BeforeHookCreation` delete policy — the previous Job
is always deleted first, ensuring exactly one Job exists per sync even when multiple syncs fire quickly.
