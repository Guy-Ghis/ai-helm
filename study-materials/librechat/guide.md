# LibreChat — Complete Operational Guide

> **Scope**: Exactly what exists in `ai-helm` / `ai-helm-values` and what runs in production. No roadmap items.
> Every detail is sourced from the chart files listed at each section.
>
> **Re-verified 2026-10-05** against `ai-helm@main` (`9bc0c3ff`), `ai-helm-values@main` (`7e91a64`) and the live
> `home-remote` cluster. Main changes since the first version: `librechart` has **2** children (the opencode
> well-known moved to `ai-models`, §6/§14), a self-hosted **Code Interpreter** exists (§18), image generation was
> removed (§5.3), the gateway attributes LibreChat traffic **per user** (§17), and the seed job prunes (§13).
> Concepts, labs and quiz: [`librechat-deep-dive.md`](librechat-deep-dive.md).

---

## Table of Contents

1. [What LibreChat Is in This Repo](#1-what-librechat-is-in-this-repo)
2. [The Chart Family — Files and Their Roles](#2-the-chart-family--files-and-their-roles)
3. [Orchestrator: `charts/librechart/`](#3-orchestrator-chartslibrechart)
4. [Leaf 1: `charts/librechat-search/` — Meilisearch](#4-leaf-1-chartslibrechat-search--meilisearch)
5. [Leaf 2: `charts/librechat-app/` — LibreChat + MongoDB](#5-leaf-2-chartslibrechat-app--librechat--mongodb)
6. [Neighbour: `charts/librechat-opencode-wellknown/` — no longer a librechart leaf](#6-neighbour-chartslibrechat-opencode-wellknown--no-longer-a-librechart-leaf)
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
17. [Gateway Integration — Identity, Plans, Rate Limits](#17-gateway-integration--identity-plans-rate-limits)
18. [Self-Hosted Code Interpreter](#18-self-hosted-code-interpreter)
19. [MongoDB Backup](#19-mongodb-backup)
20. [Live State Snapshot (2026-10-05)](#20-live-state-snapshot-2026-10-05)

---

## 1. What LibreChat Is in This Repo

LibreChat (`ghcr.io/danny-avila/librechat`, version `v0.8.7` — set in `global.librechat.version`; upstream's
latest is `v0.8.8`, 2026-10-01, repo now `LibreChat-AI/LibreChat`) is an
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
│   ├── values.yaml                  ← ArgoCD wiring + the 2-child list
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
│   ├── values.yaml                  ← 814 lines: every env var, every secret ref
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
│
│   ── NOT librechart children, but part of the LibreChat picture ──
├── librechat-code-interpreter/      ← flat app in charts/apps, ns librechat-sandbox (ADR-0122, §18)
│   ├── values.yaml                  ← 926 lines: 7-component codeapi stack on bjw-template
│   └── templates/externalsecret.yaml
├── librechat-opencode-wellknown/    ← child of the ai-models orchestrator since ADR-0125 (§14)
│   ├── values.yaml                  ← 2245 lines: the well-known JSON + nginx config
│   └── templates/configmap.yaml     ← nginx default.conf + the JSON content
└── mongodb-backup/                  ← flat app in charts/apps, ns converse-chat (§19)
```

The deployed values for `librechat-app` (the `librechat.yaml` `config:`, the `generation` bump and the agent fleet)
live in **`ai-helm-values`** `environments/prod/values/librechat-app.yaml` (954 lines). There is no values file for
`librechat-search` there — it runs on chart defaults.

---

## 3. Orchestrator: `charts/librechart/`

**Source file**: [`charts/librechart/values.yaml`](../../charts/librechart/values.yaml)

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

### The two children (exact list)

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

  # ⚠️ librechat-opencode-wellknown MOVED to the `ai-models` orchestrator (ADR-0125).
  # Its provider.camer-digital.models block is derived from the model catalog, and
  # only that orchestrator can pass `.Values.models` to a child. Do NOT re-add it here.
```

Each element becomes one ArgoCD `Application`. The ApplicationSet list generator iterates these two
elements and produces one Application per element.

The orchestrator itself is deployed by the `librechat` entry in `charts/apps/values.yaml` with
`controlPlane: true` (so the ApplicationSet lands in `argocd` on the ArgoCD cluster) and `chart: librechart`
(floating from OCI). The `chartName` is the OCI artifact name — the
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

**Source file**: [`charts/librechat-app/values.yaml`](../../charts/librechat-app/values.yaml)

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
| `LIBRECHAT_CODE_BASEURL` | `http://codeapi-api.librechat-sandbox.svc.cluster.local:3112/v1` (self-hosted, §18) |
| `CODEAPI_JWT_ENABLED` | `"true"` — mint a short-lived JWT per Code Interpreter request |
| `CODEAPI_JWT_ALGORITHM` | `EdDSA` |
| `CODEAPI_JWT_KID` | `lc-codeapi-2026-05` |
| `CODEAPI_JWT_PRIVATE_KEY_BASE64` | from Secret `librechat-codeapi-jwt`, key `private_key_base64` |
| `AWS_REGION` | from Secret `librechat-s3-config`, key `s3_region_name` |
| `AWS_ACCESS_KEY_ID` | from Secret `librechat-s3-config`, key `s3_access_key_id` |
| `AWS_SECRET_ACCESS_KEY` | from Secret `librechat-s3-config`, key `s3_secret_access_key` |
| `AWS_BUCKET_NAME` | from Secret `librechat-s3-config`, key `s3_bucket_name` |
| `AWS_ENDPOINT_URL` | `https://nbg1.your-objectstorage.com` |
| `AWS_FORCE_PATH_STYLE` | `"true"` |
| `S3_URL_EXPIRY_SECONDS` | `"604800"` (7 days; LibreChat's default 2 min broke images shortly after upload — `docs/integrations/librechat-s3-presigned-url-expiry.md`) |

**Not set (on purpose)**:
- `OPENID_REQUIRED_ROLE` is **commented out** ("open beta; everyone can connect and test"). Without it the
  `OPENID_REQUIRED_ROLE_TOKEN_KIND` / `_PARAMETER_PATH` settings are inert — **no role gate** on login.
- `IMAGE_GEN_OAI_*` were **removed 2026-09-15** (ADR-0138, no image tier after `hetzner-k8s-gpu-2` was removed).
  Never set the API key without the base URL: the tool falls back to `https://api.openai.com/v1/` and would send our
  internal gateway key to OpenAI. The values repo also lists all five image tools in `filteredTools`.
- `LIBRECHAT_CODE_API_KEY` is still wired but **dormant** (managed librechat.ai sandbox; OSS LibreChat doesn't read it).

**RAG API** (`rag-api` controller): disabled (`enabled: false`). The controller block exists in
`values.yaml` lines 558-616 but renders nothing because `bjw-template` skips disabled controllers.

---

## 6. Neighbour: `charts/librechat-opencode-wellknown/` — no longer a librechart leaf

> ⚠️ **Moved (ADR-0125).** This chart used to be "Leaf 3" of `librechart`. It is now a child of the
> **`ai-models`** orchestrator (`charts/ai-models/templates/applicationset.yaml`), deployed as Application /
> Deployment **`models-opencode-wellknown`** (sync-wave 1) in the same `converse` namespace. Reason: its
> `provider.camer-digital.models` block (per-model reasoning-effort tuning) is *derived from the model catalog*, and
> only the `ai-models` orchestrator can hand a child `.Values.models` — from `librechart` the derivation silently
> produced an empty map. Full details in §14. It shares LibreChat's host, but it is **not part of LibreChat**.

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
| `librechat-code-config` | `code_api_key` | `librechat_code_api_key` (dormant — managed sandbox only) |
| `librechat-codeapi-jwt` | `private_key_base64` | `librechat_codeapi_jwt_private_key_base64` (Ed25519 private key; the public half is consumed by `librechat-code-interpreter`, §18) |

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
| `librechat-app-db` (ClusterIP) | ClusterIP | `27017` | MongoDB pod (exists too, but LibreChat uses the headless name) |

The opencode well-known Service (`models-opencode-wellknown:80 → 8080`) is in the same namespace but belongs to
the `ai-models` orchestrator (§14).

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

This is a fully **permissive** policy — it restricts nothing. Unlike `apps`/`data`/`observability`/`platform`,
the `converse` namespace has **no default-deny baseline** (verified live 2026-10-05: the only policies in
`converse` are this one and three unrelated app policies). Combined with MongoDB's `auth.enabled: false`, network
reachability is the database's only protection — a known hardening gap.

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

**Source**: `db:` block in [`charts/librechat-app/values.yaml`](../../charts/librechat-app/values.yaml#L698-L742)

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

The template at [`charts/librechat-app/templates/_mongo_uri.tpl`](../../charts/librechat-app/templates/_mongo_uri.tpl) builds the URI as:

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

From [`charts/librechat-app/templates/pdb.yaml`](../../charts/librechat-app/templates/pdb.yaml):

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

3. **Render**: [`templates/configmap.yaml`](../../charts/librechat-app/templates/configmap.yaml)
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
         generation: "1"             # chart default; ai-helm-values overrides it — live value "14"
   ```

   When `data.generation` changes, bjw-s recomputes the annotation on the Deployment pod template,
   causing a rolling restart. The annotation key is `checksum/configMaps`, value is a SHA256 of all
   managed ConfigMap data. Without bumping `generation`, a config change in `ai-helm-values` is silently
   not picked up until the next pod restart from any other cause.

---

## 13. The Agent Seed Job — Exact Mechanics

**Source**: [`charts/librechat-app/templates/agent-seed-job.yaml`](../../charts/librechat-app/templates/agent-seed-job.yaml) +
[`charts/librechat-app/files/seed-agents.js`](../../charts/librechat-app/files/seed-agents.js)

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

What the script does (`files/seed-agents.js`, in order):
1. **Find the author**: connect to MongoDB (`MONGO_URI` has no db name ⇒ LibreChat's default db `test`), find the
   user with `PLATFORM_USER_EMAIL`. Missing ⇒ **exit 1** ("log into LibreChat once via SSO first") — fails closed
   rather than authoring under the wrong user.
2. **Mint a token**: `jwt.sign({id: user._id}, JWT_SECRET, {expiresIn: '30m'})` — the payload LibreChat's
   `requireJwtAuth` expects. Every request sends a **browser User-Agent**, because LibreChat's uaParser middleware
   rejects non-browser clients ("Illegal request").
3. **Index existing agents**: `GET /api/agents?limit=200` → map name → `agent_…` id.
4. **Resolve skills**: read the `skills` collection, map names → ObjectIds (a skill not synced yet only *warns*).
5. **Upsert, two phases**: Phase A = agents without `subagentNames` (leaves), Phase B = orchestrators, whose
   `subagentNames` must resolve to ids (else exit 1). Per agent: `mcpServers: [x]` → tool `sys__all__sys_mcp_x`,
   `tools` copied, `skills` → ids + `skills_enabled`, `subagents: {enabled, agent_ids}`; then **PATCH**
   `/api/agents/<id>` if the name exists, else **POST** `/api/agents`.
6. **Make it public**: look up the agent's Mongo `_id` (the ACL API rejects the public `agent_…` id) and
   `PUT /api/permissions/agent/<_id>` with `{updated: [], removed: [], public: true, publicAccessRoleId: "agent_viewer"}`.
7. **Prune**: delete every agent whose `author` is the platform user and whose name is no longer in the fleet.
   Scoped to that author, so user-built agents are never touched. Consequence: **a rename creates a new agent and
   deletes the old one** (new `agent_id` — any modelSpec pointing at the old id breaks). Live proof (ADR-0138
   erratum): `[agent-seed] pruned image-creator (agent_Oo2rOizVp7loelF7HMz9i)`.

The live fleet (ai-helm-values `agentSeed.agents`): **Security Reviewer**, **Test Coverage Reviewer**,
**Deep Reviewer** (subagents: the previous two; surfaced in the picker by the `deep-reviewer` modelSpec via
`agent_id: agent_OTvNLyzJLQt85SZd1xgTe`), and **coder** (`adorsys-coder-pro-internal`, `coder_mcp` + `execute_code`).
The `image-creator` agent is commented out since 2026-09-15.

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

**Owner**: `ai-models` orchestrator (ADR-0125), Application/Deployment `models-opencode-wellknown`, chart
`librechat-opencode-wellknown` (2245-line `values.yaml`), sync-wave 1, ns `converse`. URL:
`https://ai.camer.digital/opencode/.well-known/opencode`. `opencode auth login https://ai.camer.digital/opencode`
fetches it and bootstraps the developer's opencode.

### How it is served
1. `values.yaml` `wellKnown:` = the payload as YAML (`auth` + `config`); the orchestrator passes only the
   catalog-derived `models` map in.
2. `templates/configmap.yaml` renders **two** ConfigMaps: `<release>-nginx-config` (`default.conf`: listen 8080,
   `root /usr/share/nginx/html`, `default_type application/json`, `location = /opencode/.well-known/opencode` with
   `Cache-Control: no-store`, `/healthz`, everything else 404) and `<release>-content` (key `opencode` = the JSON).
3. bjw-template Deployment: `nginxinc/nginx-unprivileged:1.27-alpine`, **2 replicas**, read-only rootfs,
   emptyDirs for `/tmp`, `/var/cache/nginx`, `/var/run`; content mounted as a **directory** (no `subPath`) at
   `/usr/share/nginx/html/opencode/.well-known` — so a content change is served without a pod restart.
4. Service `:80 → 8080`; Ingress host `ai.camer.digital`, path `/opencode/.well-known/opencode` (**Exact**),
   TLS secret `ai.camer.digital-tls` (shared with LibreChat's ingress).

### Authentication (ADR-0135 — no longer Keycloak)
Plugin **`@vymalo/opencode-lightbridge@0.17.0`** replaced `@vymalo/opencode-oauth2` + `@vymalo/opencode-otel`:
- `auth`: `id: camer-digital` (provider id **and** token-cache identity — renaming it logs everyone out),
  `issuer: https://auth.ai.camer.digital` (**authz-idp**), `clientId: opencode-cli` (public client, PKCE),
  scopes `openid profile email offline_access`, `authFlow: device_code` (no local callback port → works over
  SSH/containers/CI).
- `register`: `baseURL https://api.ai.camer.digital/v1`, `syncIntervalMinutes: 60`, `responseApi: false`.
- `gateway.exchange: false` — authz-idp's device-code token is already project-scoped (`project_id`, `account_id`,
  `budget_tier`, `quota_tier`, `model_policy`, `allowed_models`) and its `iss` is what the gateway's
  `lightbridge-apikey` identity trusts; nothing to exchange.
- The `auth.command` stub (`echo plugin-managed`) only satisfies opencode's schema.

Other plugins: `@vymalo/opencode-models-info@0.17.0`, `@vymalo/opencode-ratelimit@0.17.0`,
`@vymalo/opencode-browser@0.17.0`, `opencode-skills-collection@4.0.14`.

### Rate-limit awareness (`@vymalo/opencode-ratelimit`)
`scope: model`, `headerPrefix: x-ratelimit`, tiers: reset ≤ 120 s → **wait** (max 65 s, 3 retries); longer/unknown →
**error** immediately. Timeouts `headerTimeout`/`chunkTimeout` 90 s cover the wait.

### Models, agents, MCP (live, 2026-10-05)
- `provider.camer-digital.models`: 27 entries — per-model **tuning only** (low default `reasoningEffort` + a high
  `thinking` variant); picker membership comes from the gateway (`/v1/models` ∩ `/v1/models/info`,
  `modelsInfoHideUnmatched: true`).
- 28 agents (`assistant` is `default_agent`; e.g. `architect`, `planner`, `reviewer`, `security`, `iac`, `devops`, …).
- MCP: remote (through `api.ai.camer.digital/mcp/<name>`) `brave`, `context7`, `firecrawl`, `refero`, `terraform`;
  local (npx) `confluence`, `drawio`, `git`, `jira`, `memory`, `mermaid` (**the only one enabled by default**),
  `mobile`, `reddit`, `rss`, `sequentialthinking`, `shadcn`, `youtube`.

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
    permanent: false    # temporary: 302 for GET, 307 for HEAD/other methods (verified with curl 2026-10-05)
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

### Automated sync with prune and self-heal

```yaml
automated:
  prune: true       # resources in the cluster but not in the chart are deleted
  selfHeal: true    # manual changes to the cluster are reverted
```

### Server-side apply

`syncOptions: [ServerSideApply=true]` — ArgoCD uses `kubectl apply --server-side` for all resources.
It avoids the 256 KB `last-applied-configuration` annotation limit of client-side apply for large objects
(historically the >100 KB opencode well-known ConfigMap, which now lives under `ai-models`) and gives proper
field ownership.

### Namespace creation

`syncOptions: [CreateNamespace=true]` — ArgoCD creates the `converse` namespace if it does not
exist. The namespace itself is not part of any chart.

### The PostSync hook for agent seeding

ArgoCD runs the `librechat-app-agent-seed` Job **after** the sync is complete and the Application
is Healthy (all pods are ready). The Job uses `BeforeHookCreation` delete policy — the previous Job
is always deleted first, ensuring exactly one Job exists per sync even when multiple syncs fire quickly.

---

## 17. Gateway Integration — Identity, Plans, Rate Limits

**Sources**: ai-helm-values `environments/prod/values/librechat-app.yaml` (`endpoints.custom`),
`security-policies.yaml` (internal AuthConfig), `models.yaml` (`requestRate`, `rateLimitBudgeting`),
`core-gateway.yaml` (`budgetLimiter`); ADR-0021, ADR-0125.

### 17.1 The custom endpoint

```yaml
endpoints:
  custom:
    - name: "converse"
      apiKey: "${CONVERSE_OPENAI_API_KEY}"
      baseURL: "https://core-gateway-internal.envoy-gateway-system.svc.cluster.local/v1"   # api-internal listener, internal-CA TLS
      models:
        default: ["qwen3-4b-local"]   # fallback only
        fetch: true                   # GET /v1/models at startup (ADR-0125) — sees -internal models too
      tokenConfig: { … }              # static context + $/1M prices, hand-copied from models.yaml
      titleConvo: false
      titleModel: "gemma-4"
      modelDisplayLabel: "Converse AI"
      headers:
        X-LibreChat-User:  '{{LIBRECHAT_USER_OPENIDID}}'
        X-LibreChat-Role:  '{{LIBRECHAT_USER_ROLE}}'
        X-LibreChat-Email: '{{LIBRECHAT_USER_EMAIL}}'
        X-LibreChat-Name:  '{{LIBRECHAT_USER_NAME}}'
```

(In the values file the placeholders are written as `'{{ `{{` }}LIBRECHAT_USER_OPENIDID{{ `}}` }}'` because the
chart runs the config through `common.tplvalues.render` — Helm must emit the literal braces.)

### 17.2 What Authorino does with it (internal AuthConfig, host `core-gateway-internal…`)

- **Authentication**: either a Kubernetes SA token (TokenReview, audience `core-gateway-internal`) or a static
  **apiKey** Secret labelled `kuadrant.io/apikey-for=internal-gateway` — LibreChat's is `internal-key-librechat`
  (= `converse_openai_api_key`). ext_authz is attached to **both** listeners (`api-https`, `api-internal`).
- **Descriptors** (CEL, overwriting anything the client sent):
  - `x-account-id` / `x-org-id` = `x-librechat-user` if present, non-empty, and not containing `{{`; else other
    fallbacks (Code-Intelligence repo, Coder workspace SA owner, SA username, apiKey Secret name).
  - `x-billing-plan` = `pro` if `x-librechat-role == "pro"`, else `free` (forwarded user); `internal` for services.
    LibreChat's role values are `USER`/`ADMIN`, so LibreChat users are effectively `free`.
  - `x-oidc-email` / `x-oidc-name` from the forwarded headers (per-user dashboards).
  - Project governance headers (`x-quota-tier`, `x-model-policy`, `x-api-key-id`, …) = constant `""`.
  - Dynamic metadata `budget: {enforced: false, known: false, …}` — the budget-limiter Lua filter skips this plane
    (absent metadata would make it 503).
- **Why the `{{` guard**: 2026-08-01 a paid request arrived with `account_id` literally `{{LIBRECHAT_USER_OPENIDID}}`.

### 17.3 Rate limits that actually apply to a LibreChat user (2026-10-05)

| Mechanism | State | Applies to LibreChat? |
|---|---|---|
| Per-minute burst buckets (`rateLimitBudgeting.plans.*.burst`) | removed 2026-08-01 | no |
| Monthly µ$ cost buckets (`monthlyBudgetUsd`) | deleted 2026-09-05 | no |
| Budget limiter (Lightbridge ledger, 402 `budget_exhausted`) | `enabled: true`, `shadowMode: false` | **no** — internal plane publishes `enforced: false` |
| Per-model RPM (`rpmPerKey`, default `60`) | active, one BackendTrafficPolicy per model, rules `x-api-key-id × model` and `x-account-id × model` | **yes** via the second rule → N requests/min **per user per model** (the first rule's descriptor is empty) |
| LibreChat `balance` | `enabled: false` | — |

---

## 18. Self-Hosted Code Interpreter

**Sources**: `charts/librechat-code-interpreter/`, `charts/apps/values.yaml` (`librechat-code-interpreter` entry +
`global.namespacePodSecurity`), ai-helm-values `deps/librechat-code-interpreter/`, ADR-0122,
`docs/patterns/self-hosted-code-interpreter.md`.

- **What**: the open-source `clickhouse/code-interpreter` ("codeapi") behind LibreChat's `execute_code`. Upstream
  publishes no images → built by the **`ADORSYS-GIS/code-interpreter`** fork, `ghcr.io/adorsys-gis/code-interpreter-*`,
  pinned `sha-d3b0f05`.
- **Deployment**: **flat** app (not a librechart child — the ApplicationSet forces one namespace), chart
  `librechat-code-interpreter` from OCI, namespace **`librechat-sandbox`** with Pod Security **`privileged`**.
  Sync options `CreateNamespace` + `ServerSideApply` only (no `Replace=true`: the PVC and Job have immutable fields).
- **Components** (bjw-template, `fullnameOverride: codeapi`): `api` (:3112), `service-worker`, `sandbox-runner`
  (:2000, NsJail; needs `SYS_ADMIN` + friends because no `/dev/kvm` for the microVM mode), `file-server` (:3000),
  `tool-call-server` (:3033), `egress-gateway` (:3190), `package-init` (Job, populates the packages PVC).
- **Sandbox limits** (sandbox-runner env): no networking (`SANDBOX_DISABLE_NETWORKING`; only the egress gateway
  via a signed manifest), 15 s run / 10 s compile CPU+wall, 100 processes, 64 KB output, 4 concurrent jobs, per-job UIDs.
- **Auth**: LibreChat mints a short-lived JWT per request (`CODEAPI_JWT_ENABLED=true`, `EdDSA`, kid
  `lc-codeapi-2026-05`, private key from `librechat-codeapi-jwt`); the API verifies with the public key from
  `codeapi-secrets` (ESO). Static API keys are rejected outside the service's local mode.
- **Shared infra**: redis-ha (job queue) and the `ssegning-k8s-state` bucket (files). A `CiliumNetworkPolicy`
  (deps overlay) lets only `file-server` reach `*.your-objectstorage.com:443`.
- **Constraints**: `codeapi` PVC is 10Gi RWO on `hcloud-volumes` ⇒ sandbox-runner and service-worker pinned to 1
  replica. Images run as root (no `USER` in the fork's Dockerfiles) — tracked follow-up.

---

## 19. MongoDB Backup

- Flat app `mongodb-backup` (chart floats from OCI), namespace **`converse-chat`**, CronJob `0 2 * * *`, keeps 3
  successful + 3 failed Jobs; an `exporter` init container dumps
  `mongodb://librechat-app-db-0.librechat-app-db-headless.converse.svc.cluster.local:27017`, the main container uploads
  to `https://nbg1.your-objectstorage.com`.
- Needs its **own** `librechat-s3-config` in `converse-chat` (Secrets are namespace-scoped) — provided by the deps
  overlay `environments/prod/deps/mongodb-backup`. Before that existed (until 2026-08-02) every run failed with
  `CreateContainerConfigError`.
- Live: last three runs Complete in ~17–19 s.

---

## 20. Live State Snapshot (2026-10-05)

| Object | State |
|---|---|
| `deploy/librechat-app` | 2/2, image `ghcr.io/danny-avila/librechat:v0.8.7`, pods on worker-1 and worker-3 |
| `sts/librechat-app-db` | 1/1, `mongo:8.2.6`, worker-3 |
| `sts/librechat-search` | 1/1, `getmeili/meilisearch:v1.35.0` |
| `hpa/librechat-app` | 1–4, current 2, CPU 0%/70%, memory 52%/80% |
| PDBs | `librechat-app` (1 allowed disruption), `librechat-app-db-pdb` (0 allowed) |
| Ingresses | `librechat-app`, `librechat-kivoyo-redirect`, `models-opencode-wellknown` |
| ExternalSecrets | 13 × `SecretSynced` |
| `cm/librechat-app-config-rollout` | `generation: "14"` |
| `librechat-sandbox` | 6 Deployments Running + `package-init` Completed, PVC `codeapi` 10Gi Bound |
| `converse-chat` | CronJob `mongodb-backup`, last 3 Jobs Complete |
| Log findings | `[GitHubSkillSync] … skills/governance/SKILL.md contains invalid YAML frontmatter` (hourly — unquoted `description:` containing `: `); `redis client error: Socket closed unexpectedly` (~hourly, auto-reconnects) |

