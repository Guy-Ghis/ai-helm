# LibreChat — Complete Presentation Guide

> **File to open**: `diagrams/librechat.drawio` (12 pages — use the page tabs at the bottom).
> **Verified**: 2026-10-05 against `ai-helm@main`, `ai-helm-values@main` and the live `home-remote` cluster.
> Deeper material: [`librechat-deep-dive.md`](librechat-deep-dive.md) (concepts + labs + quiz) ·
> [`guide.md`](guide.md) (line-by-line reference) · [`presentation-speaker-notes.md`](presentation-speaker-notes.md) (script).

---

## Presentation Order

| Step | Diagram | Topic |
|------|---------|-------|
| 1 | Page 1 | System Overview — what LibreChat is and everything it connects to |
| 2 | Page 2 | Chart Family — orchestrator + 2 leaf apps, and the 3 neighbours deployed elsewhere |
| 3 | Page 3 | Authentication Flow — OpenID Connect with Keycloak |
| 4 | Page 4 | Config Delivery — how `librechat.yaml` gets into the pod |
| 5 | Page 5 | Data Stores — MongoDB, Redis, Meilisearch, S3, backup |
| 6 | Page 6 | Secrets Pipeline — External Secrets Operator + the internal CA |
| 7 | Page 7 | Agent Seed Job — PostSync, two-phase upsert, public grant, prune |
| 8 | Page 8 | opencode Well-Known — a neighbour on the same host (now under `ai-models`) |
| 9 | Page 9 | Code Interpreter — where user code runs, in a throwaway jail in `librechat-sandbox` |
| 10 | Page 10 | Per-User Identity — how one shared key still gives per-user limits |
| 11 | Page 11 | Full Request Flow — browser to model response, everything together |
| 12 | Page 12 | GitHub MCP tools — connect GitHub once, then let the AI act on GitHub as you |

---

## Part 1 — What Is LibreChat in This System?

LibreChat is an MIT-licensed, self-hosted, multi-provider AI chat app (repo now `LibreChat-AI/LibreChat`).
We run **`ghcr.io/danny-avila/librechat:v0.8.7`** (upstream latest: **v0.8.8**, released 2026-10-01) at
`https://ai.camer.digital`, product name "Converse".

In our platform it is:
- a client that calls models **only through the internal Envoy AI Gateway** — it holds no vendor API key;
- an **OIDC relying party** of Keycloak realm `camer-digital` (client `converse`);
- a **MongoDB** application (all state), with **Meilisearch** for search, the shared **redis-ha** for multi-replica
  state, and **Hetzner Object Storage** for files;
- a host for curated **personas** (model specs), **seeded agents**, and **MCP** tools (GitHub, Coder, Lightbridge, …);
- the consumer of a **self-hosted Code Interpreter** for `execute_code`.

Live: 2 replicas (HPA 1–4), Mongo 8.2.6, Meilisearch v1.35.0, all 13 ExternalSecrets synced.

---

## Part 2 — Chart Family

`charts/librechart` (orchestrator) renders **one ApplicationSet** named `librechat` in ns `argocd` on the ArgoCD
cluster (`admin@homeos`). Its List generator emits **two** child Applications onto `home-remote` / ns `converse`:

| Child | Wave | Deploys |
|---|---|---|
| `librechat-search` | −1 | Meilisearch (chart `meilisearch@0.25.1`) |
| `librechat-app` | 0 | LibreChat (bjw-template 4.6.2) + MongoDB (helmforge `mongodb@1.7.6`), ExternalSecrets, internal-CA cert, HPA, NetworkPolicy, Kivoyo redirect, Agent Seed Job |

Each child has **two sources** (ADR-0055/0087): the OCI chart `oci://ghcr.io/adorsys-gis/charts/<child>` floating on
`>=0.0.0`, and `ai-helm-values@main` as `$values` providing `environments/prod/values/<child>.yaml`
(`ignoreMissingValueFiles: true`). Sync: `prune`, `selfHeal`, `CreateNamespace`, `ServerSideApply`.

**Three neighbours that are NOT librechart children** (frequent confusion):
- `librechat-code-interpreter` — flat app in `charts/apps`, own namespace `librechat-sandbox` (ADR-0122).
- opencode well-known — moved to the `ai-models` orchestrator as `models-opencode-wellknown` (ADR-0125).
- `mongodb-backup` — flat app in ns `converse-chat`, nightly dump of LibreChat's Mongo.

---

## Part 3 — Authentication (OpenID Connect)

- `ALLOW_EMAIL_LOGIN=false`, `ALLOW_REGISTRATION=false` — no local accounts.
- `ALLOW_SOCIAL_LOGIN=true`, `ALLOW_SOCIAL_REGISTRATION=true` — the first SSO login auto-creates the user.
- `OPENID_AUTO_REDIRECT=true` — straight to Keycloak (`https://auth.verif.fyi/realms/camer-digital`, client `converse`).
- `OPENID_SCOPE=openid profile email librechat offline_access`, `OPENID_AUDIENCE=converse`.
- `OPENID_REUSE_TOKENS=true` — Keycloak's refresh token becomes the session; LibreChat refreshes against Keycloak.
- `REFRESH_TOKEN_EXPIRY=5184000000` (60 days), `OPENID_USE_END_SESSION_ENDPOINT=true` (logout ends the IdP session).
- **Role gate is OFF**: `OPENID_REQUIRED_ROLE` is commented out ("open beta"). The `librechat_roles` path/kind
  variables are configured but have no effect — **any realm user can log in**.

---

## Part 4 — Config Delivery

1. **Source**: `config:` in `ai-helm-values/environments/prod/values/librechat-app.yaml` (954 lines).
2. **Injection**: the ApplicationSet adds it as a `$values` valueFile.
3. **Render**: `templates/configmap.yaml` → ConfigMap `librechat-config`, single key `librechat` (only if `config:` exists).
4. **Mount**: `subPath: librechat` → `/app/librechat.yaml` (read-only, read once at startup).
5. **Rollout**: marker ConfigMap `librechat-app-config-rollout` → `checksum/configMaps` pod annotation.
   **Bump `librechat.configMaps.config-rollout.data.generation` (live: "14") in the same commit** as any config change.

Highlights of the live config: model list fetched from the gateway (`models.fetch: true`), 7 personas with the raw
model picker hidden, summarization on `glm-4.7-flash`, skill sync from `ai-helm/skills`, `fileStrategy: s3`,
LibreChat `balance` **disabled**, `filteredTools` blocking all five image tools.

---

## Part 5 — Data Stores

| Store | Key facts |
|---|---|
| **MongoDB 8.2.6** | owned by `librechat-app`; standalone, rs `rs0` with 1 member; **no auth**; 30Gi; URI `mongodb://librechat-app-db-0.librechat-app-db-headless:27017`; PDB minAvailable 1; nightly backup at 02:00 (`converse-chat/mongodb-backup`) |
| **redis-ha** (shared, home-os) | `rediss://redis-ha-haproxy.redis-system.svc.cluster.local:6379` — TLS + password + internal CA; HAProxy, not `redis-ha-redis` (READONLY trap); prefix `librechat-prod-v2`; `APP_CONFIG,CONFIG_STORE,STARTUP_CONFIG` forced in-memory |
| **Meilisearch v1.35.0** | `http://librechat-search:7700`, master key from `librechat-meili-config`, wave −1 |
| **Hetzner Object Storage** | `nbg1.your-objectstorage.com`, bucket `ssegning-k8s-state`, path-style, presigned URL TTL raised to 7 days |

---

## Part 6 — Secrets Pipeline

ESO + ClusterSecretStore `ssegning-aws`, refresh 1h. App secrets from `ai/camer/digital/prod/env`, platform secrets
(redis password, S3 keys) from `prod/meta/test-app`. 13 ExternalSecrets live:

| Secret | Purpose |
|---|---|
| `librechat-config` | `CREDS_KEY/IV` (AES for stored creds), `JWT_SECRET`, `JWT_REFRESH_SECRET` |
| `librechat-openid-config` | Keycloak client id/secret + session secret |
| `librechat-main-config` | `CONVERSE_OPENAI_API_KEY` — LibreChat's key for the internal gateway |
| `librechat-meili-config` | Meilisearch master key |
| `librechat-mcp-cd-credentials` / `-mcp-github` / `-mcp-coder-credentials` | MCP OAuth clients (Keycloak self-service, GitHub, Coder) |
| `librechat-websearch-config` | Serper, Firecrawl, Jina |
| `librechat-skills-sync` | GitHub token for skill sync |
| `librechat-codeapi-jwt` | Ed25519 **private** key for Code Interpreter JWTs |
| `librechat-code-config` | dormant managed-Code-Interpreter key |
| `librechat-s3-config` | S3 creds (templated with static region/bucket) |
| `redis-ha-redis-auth` | copy of the redis password into `converse` |

`librechat-internal-ca` is from **cert-manager** (`self-signed-ca`), not ESO: its `ca.crt` is trusted via
`NODE_EXTRA_CA_CERTS` (gateway TLS) and `REDIS_CA` (redis TLS). Env vars bind at pod start — a rotated secret needs a restart.

---

## Part 7 — Agent Seed Job

`agentSeed.enabled: true` (in ai-helm-values) → PostSync Job `librechat-app-agent-seed` on the LibreChat image,
`BeforeHookCreation`, `backoffLimit 3`, TTL 600 s, re-runs when script/fleet checksum changes.

1. Find `platform@ai.camer.digital` in Mongo (must have logged in once — else **fail closed**); mint a 30-min LibreChat
   JWT with `JWT_SECRET`.
2. `GET /api/agents`; for each fleet entry **PATCH** (by name) or **POST**; leaves first, then orchestrators
   (subagent ids must exist); map `mcpServers` → `sys__all__sys_mcp_<server>`, `skills` → skill ids.
3. Grant **public view** (`PUT /api/permissions/agent/<_id>`, `agent_viewer`).
4. **Prune** platform-authored agents no longer in the fleet (never touches user agents).

Fleet today: Security Reviewer, Test Coverage Reviewer, **Deep Reviewer** (delegates to both; exposed as a persona),
`coder`. A rename = new id + old deleted.

Why PostSync, not an init container: the script calls LibreChat's own REST API, so LibreChat must be up.

---

## Part 8 — opencode Well-Known (a neighbour)

`https://ai.camer.digital/opencode/.well-known/opencode` — served by nginx Deployment `models-opencode-wellknown`
(2 replicas, chart `librechat-opencode-wellknown`), **now a child of the `ai-models` orchestrator** (ADR-0125) because
its per-model tuning is derived from the model catalog. Same host as LibreChat, but **not part of LibreChat**.

- `opencode auth login https://ai.camer.digital/opencode` fetches `{auth, config}`.
- Auth: plugin **`@vymalo/opencode-lightbridge@0.17.0`**, **device_code** flow against **authz-idp**
  (`https://auth.ai.camer.digital`, client `opencode-cli`, ADR-0135) — no longer Keycloak.
- Provider `camer-digital` → `https://api.ai.camer.digital/v1` (external plane), plus `-models-info`, `-ratelimit`
  (wait ≤120 s resets, error on longer), `-browser`, `opencode-skills-collection`.
- Why device_code: no local callback port needed (SSH, containers, CI).

---

## Part 9 — Code Interpreter

`execute_code` → `LIBRECHAT_CODE_BASEURL=http://codeapi-api.librechat-sandbox.svc.cluster.local:3112/v1`.
LibreChat mints a short-lived **Ed25519 JWT** per request (`CODEAPI_JWT_*`, kid `lc-codeapi-2026-05`); the API verifies it
with the public key. Components: api, service-worker, sandbox-runner (NsJail — needs `SYS_ADMIN`), file-server,
tool-call-server, egress-gateway, package-init. Own namespace with **privileged** Pod Security so the elevation
doesn't touch LibreChat/Mongo; reuses redis-ha + the S3 bucket; Cilium policy lets only file-server reach object
storage; RWO PVC ⇒ 1 sandbox-runner replica. Images from the `ADORSYS-GIS/code-interpreter` fork, pinned by SHA.

---

---

## Part 10 — Per-User Identity

Two identities per request: *user → LibreChat* (OIDC) and *LibreChat → gateway* (static apiKey). The bridge is
LibreChat's header templating (`{{LIBRECHAT_USER_OPENIDID}}` = Keycloak `sub`). Authorino trusts it only on the
internal plane, **overwrites** descriptors (client can't choose its plan), and ignores values that still contain
`{{` (a real placeholder leak seen 2026-08-01). `X-LibreChat-Role` is LibreChat's role (USER/ADMIN), so users map to
`free`. Email/name feed the per-user Grafana boards.

---

## Part 11 — Full Request Flow

```
Browser ─HTTPS─► Traefik ─► librechat-app:3080
   LibreChat: validates its session (JWT cookie; refreshes via Keycloak when needed — token reuse)
   builds POST /v1/chat/completions to the "converse" custom endpoint:
     Authorization: Bearer ${CONVERSE_OPENAI_API_KEY}
     X-LibreChat-User/-Role/-Email/-Name: {{LIBRECHAT_USER_*}}
 ─TLS (internal CA)─► core-gateway-internal.envoy-gateway-system.svc.cluster.local:443  (api-internal listener)
   Authorino (internal AuthConfig): apiKey OK → x-account-id = user's Keycloak sub, x-billing-plan = free|pro,
                                    x-oidc-email/name, project headers = "", budget.enforced = false
   Budget-limiter Lua: skipped for this plane (enforced=false)
   Lyft ratelimit: per-model RPM per x-account-id (default 60/min)  → 429 if exceeded
   AIGatewayRoute → model backend (DeepInfra / Fireworks / Google / self-hosted GPU)
 ◄─ streamed response ─ back through LibreChat to the browser (SSE)
```

**Key insight**: LibreChat authenticates with **one** key, but the gateway **attributes and rate-limits per user**,
using the same account key as that person's opencode usage. LibreChat's own `balance` is off.

---

## What changed since the first version of this deck

Two leaves instead of three (well-known moved), Code Interpreter added, role gate is off, per-user attribution
replaces "one shared billing account", no Lua model-policy/budget checks on LibreChat's path, image generation
withdrawn, opencode now logs in via authz-idp. Full table: [`librechat-deep-dive.md` Part C](librechat-deep-dive.md#part-c--corrections-to-the-earlier-study-material).
