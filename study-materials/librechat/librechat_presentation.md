# LibreChat — Complete Presentation Guide

> **File to open**: `diagrams/librechat.drawio`
> Use the page tabs at the bottom to navigate between diagrams.

---

## Presentation Order

| Step | Diagram | Topic |
|------|---------|-------|
| 1 | Page 1 | System Overview — what LibreChat is and how it fits in the platform |
| 2 | Page 2 | Chart Family — orchestrator + 3 leaf apps + sync waves |
| 3 | Page 3 | Authentication Flow — OpenID Connect with Keycloak |
| 4 | Page 4 | Config Delivery — how librechat.yaml gets into the pod |
| 5 | Page 5 | Data Stores — MongoDB, Redis, Meilisearch, S3 |
| 6 | Page 6 | Secrets Pipeline — External Secrets Operator |
| 7 | Page 7 | Agent Seed Job — PostSync mechanics |
| 8 | Page 8 | opencode Well-Known — nginx + JSON + auth plugin |
| 9 | Page 9 | Networking — Ingresses, Services, TLS, vanity redirect |
| 10 | Page 10 | Full Request Flow — browser to AI model response |

---

## Part 1 — What Is LibreChat in This System?

LibreChat (`ghcr.io/danny-avila/librechat:v0.8.7`) is the **primary end-user web interface** for AI interactions on this platform. It is reachable at `https://ai.camer.digital`.

It is NOT a standalone product — in this deployment it is tightly integrated as:
- A client that calls AI models **exclusively through the internal Envoy AI Gateway** (never directly to providers)
- An **OpenID Connect relying party** against the Keycloak realm `camer-digital`
- A **MongoDB application** using Mongoose (no ORM abstraction)
- A **Redis client** for session caching and real-time features

Image pull policy is `Always` — every pod restart pulls the tagged image.

---

## Part 2 — Chart Family

The LibreChat stack is deployed as a **family of 4 Helm charts**:

### Orchestrator: `charts/librechart/`
Renders exactly **one Kubernetes resource**: an ArgoCD `ApplicationSet` named `librechat`. The ApplicationSet lives in the ArgoCD control-plane cluster (`admin@homeos`), NOT on the workload cluster (`home-remote`). It uses a list generator to produce three child Applications.

### Leaf 1: `charts/librechat-search/` (sync-wave -1)
Deploys **Meilisearch** — the full-text search engine. It deploys BEFORE LibreChat so search is ready when LibreChat starts. Port 7700, plain HTTP (no TLS needed in-namespace).

### Leaf 2: `charts/librechat-app/` (sync-wave 0)
The main chart. Deploys **LibreChat + MongoDB**. 742 lines of values — every env var, every secret reference. Also renders all ExternalSecrets and the Agent Seed Job.

### Leaf 3: `charts/librechat-opencode-wellknown/` (sync-wave 1)
A **static nginx pod** that serves one JSON document — the opencode well-known bootstrap document. 2125 lines of values — the entire JSON payload expressed as YAML.

### The `$values` pattern (ADR-0087)
Each child Application has **two Helm sources**:
1. The OCI chart from `oci://ghcr.io/adorsys-gis/charts`
2. `ai-helm-values@main` — a private repo with production-specific values

This separation means chart logic is public in `ai-helm`, but secrets and environment-specific config live in the private `ai-helm-values` repo.

---

## Part 3 — Authentication (OpenID Connect)

LibreChat uses **OIDC exclusively**. Password-based login is disabled:
- `ALLOW_EMAIL_LOGIN=false`
- `ALLOW_REGISTRATION=false`
- `ALLOW_SOCIAL_LOGIN=true`
- `OPENID_AUTO_REDIRECT=true` — immediately redirects unauthenticated users to Keycloak

**OIDC Provider**: Keycloak at `https://auth.verif.fyi/realms/camer-digital`, client `converse`

**Login flow:**
1. User visits `ai.camer.digital` → auto-redirected to Keycloak
2. User authenticates (SSO, federated identity, etc.)
3. Keycloak redirects to `/oauth/openid/callback` with an authorization code
4. LibreChat exchanges the code for an ID token + refresh token
5. Session established — 60-day refresh token (`REFRESH_TOKEN_EXPIRY=5184000000`)

**Role check**: LibreChat reads `librechat_roles` from the access token (`OPENID_REQUIRED_ROLE_PARAMETER_PATH`) to gate access. Users without the required role get a 401.

---

## Part 4 — Config Delivery

The `librechat.yaml` config file reaches the pod through a 5-step chain:

1. **Source**: `config:` key in `ai-helm-values/environments/prod/values/librechat-app.yaml`
2. **Injection**: ArgoCD ApplicationSet merges this file as Helm source 2 at render time
3. **Render**: `templates/configmap.yaml` serializes the `config:` object into a ConfigMap with a single key `librechat`
4. **Mount**: `subPath: librechat` mounts only that single key as `/app/librechat.yaml`
5. **Rollout**: A separate `config-rollout` ConfigMap with a `generation` field triggers rolling restarts

**Critical detail**: `subPath` mounts do NOT auto-update when the ConfigMap changes. You MUST bump the `generation` field in the same commit as any config change to trigger a rolling restart. Without this, LibreChat continues running on stale config.

---

## Part 5 — Data Stores

### MongoDB
- **Owned by**: `librechat-app` chart
- **Architecture**: standalone (single pod, single replica set member `rs0`)
- **Auth**: disabled (`auth.enabled: false`) — no MongoDB password
- **Storage**: 30Gi PVC on the cluster default StorageClass
- **Connection string**: `mongodb://librechat-app-db-0.librechat-app-db-headless:27017`
  - Connects to the **pod directly** via headless service DNS — NOT through the ClusterIP service
- **PDB**: `minAvailable: 1` — MongoDB pod cannot be evicted if it would leave zero pods

### Redis HA
- **Shared** — LibreChat does NOT own this Redis instance
- **URI**: `rediss://redis-ha-haproxy.redis-system.svc.cluster.local:6379` (double-s = TLS)
- **Target**: HAProxy master-router — NOT `redis-ha-redis` (round-robin would cause READONLY errors on replica hits)
- **Key prefix**: `librechat-prod-v2`
- **Forced in-memory**: `APP_CONFIG`, `CONFIG_STORE`, `STARTUP_CONFIG` — these 3 namespaces skip Redis to avoid round-trips on every request

### Meilisearch
- Plain HTTP, port 7700
- Master key from Secret `librechat-meili-config`
- LibreChat connects via `MEILI_HOST=http://librechat-search:7700`

### S3 (Hetzner Object Storage)
- Endpoint: `nbg1.your-objectstorage.com`
- Bucket: `ssegning-k8s-state` (shared with other platform services)
- Prefixes used by LibreChat: `images/` and `files/`
- `AWS_FORCE_PATH_STYLE=true` — required for Hetzner's S3-compatible API

---

## Part 6 — Secrets Pipeline

All secrets come from **AWS Secrets Manager** via the **External Secrets Operator** (ESO).

- **ClusterSecretStore**: `ssegning-aws`
- **SM path**: `ai/camer/digital/prod/env`
- **Refresh interval**: 1 hour

The `externalsecret-app.yaml` template iterates a list and emits one ExternalSecret per entry, each becoming a Kubernetes Secret in the `converse` namespace.

Key secrets and their purpose:
| Secret | Purpose |
|--------|---------|
| `librechat-config` | JWT signing keys, credential encryption keys |
| `librechat-openid-config` | Keycloak OIDC client credentials |
| `librechat-main-config` | The `CONVERSE_OPENAI_API_KEY` (the Lightbridge API key) |
| `librechat-meili-config` | Meilisearch master key |
| `redis-ha-redis-auth` | Redis password (cross-namespace copy from `redis-system`) |
| `librechat-s3-config` | Hetzner S3 credentials |
| `librechat-mcp-*` | OAuth2 credentials for MCP server integrations |
| `librechat-websearch-config` | Serper, Firecrawl, Jina API keys |

The `librechat-internal-ca` Secret is different — it is created by **cert-manager** (not ESO). It carries the Home Root CA certificate used to trust the Redis HA TLS certificate and the Envoy AI Gateway's internal listener.

---

## Part 7 — Agent Seed Job

When `agentSeed.enabled: true`, every ArgoCD sync triggers a **PostSync Job** that creates or updates AI agents in the LibreChat database.

**Hook annotations**:
- `argocd.argoproj.io/hook: PostSync` — runs after Application is Healthy
- `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation` — old Job deleted before new one runs

**Two-phase operation**:
1. Connect directly to MongoDB, find the platform user by email, generate a JWT
2. Use LibreChat's REST API to GET each agent by name, then PATCH (update) or POST (create)

The script is **idempotent** — running it twice produces the same result. This means every sync safely re-runs it.

Why PostSync and not an Init Container? Because the seed script calls LibreChat's own API. LibreChat must be running and healthy first. PostSync guarantees this.

---

## Part 8 — opencode Well-Known

The well-known endpoint at `https://ai.camer.digital/opencode/.well-known/opencode` is served by a static **nginx pod** (Leaf 3).

When a developer runs `opencode auth login https://ai.camer.digital/opencode`, the CLI fetches this URL and bootstraps the user's entire development environment: OAuth2 auth, AI provider config, plugins, MCP servers, and model list.

**Key sections of the JSON**:
- `auth.command`: a stub (`echo plugin-managed`) — the oauth2 plugin overrides auth completely
- `provider.camer-digital.baseURL`: `https://api.ai.camer.digital/v1` — the public AI Gateway endpoint
- `oauth2.authFlow: device_code` — works in headless/remote environments (no local port needed)
- `oauth2.clientId: opencode-cli` — a separate Keycloak client from `converse`
- `rateLimit.tiers`: automatic retry on 429 (wait ≤120s), immediate error on 429 (>120s = monthly budget)

**Why device_code flow?** `authorization_code` requires a local callback port. In SSH sessions, containers, and CI environments, there is no browser to receive the redirect. Device code flow lets the user open any browser on any machine.

---

## Part 9 — Networking

### Main Ingress (`ai.camer.digital`)
- IngressClassName: `traefik`
- TLS: issued by `cert-home-cert-http` (ACME HTTP-01), secret `ai.camer.digital-tls`
- Backend: `librechat-app:3080`
- Homepage widget annotations: shows up in the Gethomepage dashboard

### Vanity Domain Redirect (`ai.kivoyo.com` → `ai.camer.digital`)
Two resources working together:
1. **Traefik Middleware** `kivoyo-redirect`: regex redirect (302, not 301) from `ai.kivoyo.com` to `ai.camer.digital`
2. **Ingress** `librechat-kivoyo-redirect`: terminates TLS for `ai.kivoyo.com`, fires the middleware

The backend Service (`librechat-app:3080`) is never actually reached — Traefik intercepts and redirects before forwarding. The backend spec is required by the Kubernetes Ingress schema.

### Services
| Service | Type | Port | Target |
|---------|------|------|--------|
| `librechat-app` | ClusterIP | 3080 | LibreChat container |
| `librechat-app-db` | Headless | 27017 | MongoDB pod |
| `librechat-search` | ClusterIP | 7700 | Meilisearch |
| opencode wellknown | ClusterIP | 80 | nginx |

### HPA
- Min: 1 replica, Max: 4 replicas
- Scale on: CPU >70% or Memory >80%

---

## Part 10 — Full Request Flow

```
User Browser
  → Traefik (HTTPS termination)
    → librechat-app:3080 (LibreChat)
      → Keycloak (session validation via Redis)
        → LibreChat constructs AI request using CONVERSE_OPENAI_API_KEY
          → Envoy AI Gateway (internal, core-gateway-internal.svc:443)
            → Authorino (ext_authz: validates key, stamps billing headers)
              → Lua: billing-period (stamp x-billing-period)
              → Lua: model-policy (403 if model denied)
              → Lua: budget-limiter (402 if balance ≤ 0)
              → Lyft Ratelimit (429 if rpm exceeded)
                → AI Model Backend (streams response)
                  → back through LibreChat to user browser
```

**Key insight**: All LibreChat users share the **same** `CONVERSE_OPENAI_API_KEY`. Individual user quotas within LibreChat are enforced by LibreChat's own configuration (`librechat.yaml`), not by the Envoy gateway's per-key rate limits.
