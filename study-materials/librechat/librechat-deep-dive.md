# LibreChat — Everything You Need to Know (General + Our Platform + Labs)

> **Verified on 2026-10-05** against: upstream LibreChat release notes/docs (v0.8.8, config
> schema 1.3.17), `ADORSYS-GIS/ai-helm@main` (`9bc0c3ff`), `ADORSYS-GIS/ai-helm-values@main`
> (`7e91a64`), and read-only `kubectl` against the live `home-remote` (Hetzner) cluster.
> Anything older than that date may have drifted — Part D tells you how to re-verify every claim yourself.
>
> **How this file relates to the others in this folder**
>
> | File | Role |
> |---|---|
> | **this file** | The "know it cold" file: concepts, our integration, gotchas, hands-on labs, self-check quiz |
> | [`guide.md`](guide.md) | Exhaustive reference — every env var, secret, port, template, line by line |
> | [`librechat_presentation.md`](librechat_presentation.md) | Presentation order + per-diagram summary |
> | [`presentation-speaker-notes.md`](presentation-speaker-notes.md) | What to say, page by page, plus anticipated Q&A |
> | [`diagrams/librechat.drawio`](diagrams/librechat.drawio) | The 11 presentation diagrams |
> | [`diagram.drawio`](diagram.drawio) | Two one-page overviews (GitOps structure, runtime integrations) |

---

## Contents

- **Part A — LibreChat in general** (what the product is, independent of us)
  - A1 What it is · A2 Architecture · A3 The two config surfaces · A4 Endpoints · A5 Agents, model specs, subagents ·
    A6 MCP · A7 Authentication · A8 Tools (web search, code interpreter, file search, skills) ·
    A9 Cost, balance, rate limits · A10 Scaling (Redis) · A11 What's new in v0.8.8
- **Part B — LibreChat on our platform** (how we deploy, wire and run it)
  - B1 One-page map · B2 Where everything is defined · B3 Deploy pipeline · B4 Identity & per-user attribution ·
    B5 Models & the gateway · B6 Personas, agents, MCP · B7 Data stores · B8 Secrets & TLS trust ·
    B9 Networking · B10 Neighbours (Code Interpreter, opencode well-known, backup) · B11 Security posture ·
    B12 Live issues observed · B13 Upgrade notes (v0.8.7 → v0.8.8)
- **Part C — Corrections to the earlier study material** (what changed and why it matters)
- **Part D — Practicals** (local labs for the general topic, read-only labs against our infra, change exercises)
- **Part E — Self-check quiz** (with answers)
- **Part F — Sources**

---

# Part A — LibreChat in general

## A1. What it is

LibreChat is an **open-source, self-hosted, multi-provider AI chat application** — a ChatGPT-style
web UI plus a server that talks to many model providers, runs tool-using **agents**, connects to **MCP**
servers, stores conversations, and handles files.

| Fact | Value (2026-10-05) |
|---|---|
| License | MIT |
| Creator / lead | Danny Avila |
| Repo | `github.com/LibreChat-AI/LibreChat` (moved from `danny-avila/LibreChat`; the old URL redirects. Container images are still `ghcr.io/danny-avila/librechat`) |
| Popularity | ~45k GitHub stars |
| Latest release | **v0.8.8** (2026-10-01). Previous: v0.8.7 (2026-06-24) — **what we run** |
| Latest config schema | **1.3.17** (librechat.yaml `version:`). Ours declares `1.3.5` |
| Docs | <https://www.librechat.ai/docs> |

Why organisations pick it: one UI for every model (vendor + self-hosted), SSO, agents with real tools,
MCP, per-user data isolation, and it runs entirely on infrastructure you control.

## A2. Architecture (generic)

```
                ┌────────────────────────── LibreChat ──────────────────────────┐
 Browser ──►    │  client/  (React + Vite SPA, served by the API process)        │
                │  api/     (Node.js / Express server, port 3080)                │
                │  packages/data-provider · packages/api · packages/data-schemas │
                └──┬──────────┬───────────┬──────────┬──────────┬──────────┬────┘
                   │          │           │          │          │          │
                MongoDB   Meilisearch   Redis     File store  RAG API    Code Interpreter
               (required) (optional)  (optional;  (local/s3/  (optional, API (optional)
               all state  conv/message needed for azure/      Python +
                          full-text    >1 replica) firebase)  pgvector)
                   │
           Model providers: OpenAI, Anthropic, Google, Azure, Bedrock, + any OpenAI-compatible "custom" endpoint
           MCP servers:     stdio / SSE / streamable-http
```

| Component | Required? | What it does |
|---|---|---|
| **API server** (Node/Express) | yes | Auth, conversations, streaming to the browser, calls providers, runs agents, MCP client, serves the SPA |
| **MongoDB** (via Mongoose) | yes | Users, conversations, messages, agents, prompts, files metadata, skills, ACLs, tokens/transactions |
| **Meilisearch** | optional (`SEARCH=true`) | Full-text search over conversations/messages; LibreChat syncs Mongo → Meili itself |
| **Redis** | optional, **mandatory for multiple replicas** | Shared cache, sessions, rate-limit counters, stream/resume state, MCP/OAuth state across pods |
| **File storage** (`fileStrategy`) | yes (default `local`) | Uploaded files, generated images, avatars. `s3` returns **presigned URLs** |
| **RAG API** | optional | Separate Python service + Postgres/pgvector; powers `file_search` embeddings |
| **Code Interpreter API** | optional | Sandbox that runs code for the `execute_code` capability (managed by librechat.ai, or self-hosted) |

## A3. The two configuration surfaces — know which knob lives where

| | `.env` (environment variables) | `librechat.yaml` |
|---|---|---|
| Holds | **secrets + infrastructure + auth**: `MONGO_URI`, `JWT_SECRET`, `CREDS_KEY`, `OPENID_*`, `REDIS_URI`, `MEILI_*`, `AWS_*`, provider keys, `ALLOW_*` | **features + behaviour**: `endpoints`, `modelSpecs`, `mcpServers`, `interface`, `fileConfig`, `rateLimits`, `balance`, `webSearch`, `summarization`, `skillSync`, `filteredTools` … |
| Read | at process start | at process start (path from `CONFIG_PATH`, default `/app/librechat.yaml`) |
| Hot reload? | no | **no** — restart required |
| Cross-reference | — | `${ENV_VAR}` placeholders inside yaml (e.g. `apiKey: "${CONVERSE_OPENAI_API_KEY}"`) |

Rule of thumb: *if it is a secret or a connection string → env; if it shapes what users see/do → yaml.*

## A4. Endpoints — how LibreChat reaches models

Built-in endpoint families: `openAI`, `anthropic`, `google`, `azureOpenAI`, `bedrock`, `assistants`, `agents`,
plus **`custom`** — any OpenAI-compatible API (vLLM, Ollama, OpenRouter, our Envoy AI Gateway…).

A custom endpoint (fields from the official docs):

```yaml
endpoints:
  custom:
    - name: "converse"                  # shown in the endpoint menu; also the id other config refers to
      apiKey: "${MY_KEY}"               # env ref, literal, or "user_provided" (user types their own)
      baseURL: "https://gateway/v1"
      models:
        default: ["fallback-model"]     # used if fetch fails / fetch is off
        fetch: true                     # GET {baseURL}/models at startup
      tokenConfig:                      # context window + $/1M-token pricing per model
        "fallback-model": { context: 32768, prompt: 0.1, completion: 0.5, cacheRead: 0.02 }
      titleConvo: true
      titleModel: "cheap-model"
      headers:                          # sent on every request; templated per user/request
        X-User: "{{LIBRECHAT_USER_OPENIDID}}"
      addParams / dropParams / customParams / directEndpoint / iconURL / modelDisplayLabel …
```

**Header placeholders** (exact names, from the docs): `{{LIBRECHAT_USER_ID}}`, `{{LIBRECHAT_USER_NAME}}`,
`{{LIBRECHAT_USER_USERNAME}}`, `{{LIBRECHAT_USER_EMAIL}}`, `{{LIBRECHAT_USER_PROVIDER}}`,
`{{LIBRECHAT_USER_ROLE}}`, `{{LIBRECHAT_USER_OPENIDID}}` (the IdP `sub`), `…_GOOGLEID/_GITHUBID/_SAMLID/_LDAPID…`,
`{{LIBRECHAT_USER_EMAILVERIFIED}}`, `{{LIBRECHAT_USER_TENANT_ID}}` (agent requests only), plus request-body
placeholders `{{LIBRECHAT_BODY_CONVERSATIONID}}`, `{{LIBRECHAT_BODY_MESSAGEID}}`, `{{LIBRECHAT_BODY_PARENTMESSAGEID}}`.
These are **the** mechanism for telling a shared upstream *which human* is behind a request — we rely on it (B4).

## A5. Agents, model specs, subagents

- **Agent** = model + instructions + tools (built-in capabilities, MCP tools, actions) + files. Stored as a
  **Mongo document** (`agents` collection), created in the Agent Builder UI or via `POST /api/agents`.
  Every agent has an `author` (a real user `_id`) — LibreChat has **no service accounts**.
  Visibility is an **ACL** (private / shared / public via `PUT /api/permissions/agent/<_id>`).
- **Capabilities** (agents endpoint): `execute_code`, `file_search`, `web_search`, `artifacts`, `ocr`,
  `chain`, `actions`, `tools`, plus **skills** and **subagents**.
- **Subagents**: an agent may delegate to other agents as tools (`subagents: {enabled, agent_ids}`) —
  isolated-context children. Referenced agents must exist first ⇒ creation order matters.
- **Model specs** (`modelSpecs` in yaml) = curated, named presets in the picker. In v0.8.7 a spec pointing at
  a non-agents endpoint becomes an **ephemeral agent** at runtime, so a spec can carry `mcpServers`,
  `webSearch`, `fileSearch`, `executeCode`, `skills` and a `promptPrefix` — *agent-like behaviour with
  no DB object*, fully git-managed. What a spec **cannot** do: subagent delegation (needs real agent ids).
  A spec can also wrap a real DB agent: `preset: { endpoint: agents, agent_id: agent_… }`.
- `interface.modelSelect: false` hides the raw model dropdown; `modelSpecs.enforce: true` removes raw
  access completely; `prioritize` + `default: true` pick the landing persona.

## A6. MCP (Model Context Protocol)

`mcpServers:` in yaml declares servers; transports `stdio`, `sse`, `streamable-http`. Per-server knobs:
`url`, `headers` (same placeholders as A4), `oauth` (per-user OAuth flow: `authorization_url`, `token_url`,
`client_id`, `client_secret`, `redirect_uri`, `scope`), `requiresOAuth`, `serverInstructions` (injected into
the system prompt), `initTimeout`, `chatMenu`, `iconPath`. Per-user OAuth tokens are stored **encrypted
with `CREDS_KEY`/`CREDS_IV`** in Mongo. `interface.mcpServers.{use,create,share,public}` controls whether users
may add their own.

An agent references a whole server with the tool id `sys__all__sys_mcp_<server>` (what our seed script writes).

## A7. Authentication

Strategies: local email/password, social (Google/GitHub/Discord/Apple/Facebook), **OpenID Connect**, SAML, LDAP.

| Variable | Meaning |
|---|---|
| `ALLOW_EMAIL_LOGIN`, `ALLOW_REGISTRATION` | local login / local sign-up |
| `ALLOW_SOCIAL_LOGIN`, `ALLOW_SOCIAL_REGISTRATION` | social+OIDC login / **auto-create the user on first SSO login** |
| `OPENID_ISSUER`, `OPENID_CLIENT_ID/SECRET`, `OPENID_SCOPE`, `OPENID_CALLBACK_URL` | the OIDC client |
| `OPENID_AUTO_REDIRECT` | skip the login page, bounce straight to the IdP |
| `OPENID_REUSE_TOKENS` | use the **IdP's** refresh token as the session; LibreChat refreshes against the IdP (works across replicas) |
| `OPENID_REQUIRED_ROLE` (+ `_TOKEN_KIND`, `_PARAMETER_PATH`) | gate login on a role claim — **only active if `OPENID_REQUIRED_ROLE` is set** |
| `OPENID_AUDIENCE` | audience requested/validated for the tokens |
| `OPENID_USE_END_SESSION_ENDPOINT` | logout also ends the IdP session |
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | LibreChat's **own** access/refresh tokens (HS256). Anyone with `JWT_SECRET` can mint a token for any user `_id` — our seed job does exactly this |
| `CREDS_KEY` (32-byte hex) / `CREDS_IV` (16-byte hex) | AES key/IV encrypting stored credentials (user-provided API keys, MCP OAuth tokens). **Rotating them makes stored creds undecryptable.** v0.8.8 refuses to start if they're invalid |
| `LOGIN_MAX` / `LOGIN_WINDOW` | login attempt rate limit |

## A8. Tools & features you'll be asked about

- **Web search** (`webSearch:`): three stages — *search provider* (Serper or SearXNG) → *scraper* (Firecrawl) →
  *reranker* (Jina or Cohere). All three keys must be present for the pipeline.
- **Code Interpreter** (`execute_code`): LibreChat calls a Code Interpreter API at `LIBRECHAT_CODE_BASEURL`.
  Default is the **managed** librechat.ai service (`LIBRECHAT_CODE_API_KEY`). The open-source
  `clickhouse/code-interpreter` can be **self-hosted**; then LibreChat mints **short-lived Ed25519/RS256 JWTs**
  per request (`CODEAPI_JWT_ENABLED`, private key in env, public key on the service).
- **File search / RAG**: needs the RAG API (`RAG_API_URL`) + embeddings model.
- **Skills** (`SKILL.md` files with `name`/`description` frontmatter): reusable instructions; `skillSync.github`
  mirrors them from a repo on an interval. A broken frontmatter fails that sync (we have a live example — B12).
- **Summarization** (`summarization:`): roll long chats into a running summary when the context hits a ratio.
- **Artifacts** (rendered React/HTML/Mermaid), **Memory**, **Prompts library**, **Presets**, **Bookmarks**, **Sharing**.
- `filteredTools` / `includedTools`: deny/allow-list built-in tools by `pluginKey`.

## A9. Cost, balance, rate limits (LibreChat-side)

- `tokenConfig` (per custom endpoint) gives LibreChat context windows and **$/1M-token prices**; used for
  truncation and for the cost gauge (`interface.contextCost: true`).
- `balance:` turns on **LibreChat's own token credits** per user (startBalance, auto-refill). When disabled,
  LibreChat enforces **no** spend limit itself.
- `rateLimits:` covers LibreChat endpoints (file uploads, imports, TTS/STT, conversation forks) — **not** model spend.
- Model-spend limits can therefore live *in LibreChat* (balance) **or upstream** (a gateway). We do the latter (B5).

## A10. Scaling — why Redis matters

One replica works with in-memory state. **More than one replica requires Redis** (`USE_REDIS=true`, `REDIS_URI`):
shared cache, sessions, rate-limit stores, stream resumption, MCP connection/OAuth state.
`REDIS_KEY_PREFIX` namespaces keys on a shared Redis. `FORCED_IN_MEMORY_CACHE_NAMESPACES` keeps chosen cache
namespaces per-process (cheap, but each pod holds its own copy — fine for config that is identical on every pod).
`rediss://` = TLS; `REDIS_CA` trusts a private CA. LibreChat **cannot speak Redis Sentinel** natively — put a
master-router (HAProxy) in front of a Sentinel setup.

## A11. What's new in v0.8.8 (2026-10-01) — so you can answer "should we upgrade?"

Highlights from the release notes: agent control (interrupt, steer, queue, approve tool calls); unified Agent
Builder with same-run skill authoring and opt-in **Programmatic Tool Calling**; **background tasks**; Scheduled
Chats (beta, off by default); experimental **code workspaces**; beta **Agents API** (Chat Completions / Open
Responses compatible); experimental Agent Plugins (bundle skills + MCP); execution visibility (activity, tool
timing, context usage, Langfuse trace viewer); new models (GPT-6/6.1, Claude Opus/Sonnet 5.5, Gemini 3.8 Flash);
**tenant-scoped custom endpoints**; hardened MCP OAuth; and *"Reject Invalid AES Credentials Before Startup"*.
Config schema moved through 1.3.15 → 1.3.17.

---

# Part B — LibreChat on our platform

## B1. One-page map

```
                       Keycloak  auth.verif.fyi/realms/camer-digital  (client: converse)
                           ▲  OIDC (auto-redirect, token reuse)
                           │
User ──HTTPS──► Traefik ──► librechat-app (Deployment, 2 pods, HPA 1–4)  ns: converse   [home-remote / Hetzner k3s]
 ai.camer.digital            │  image ghcr.io/danny-avila/librechat:v0.8.7
 ai.kivoyo.com ─302─┘        │
                             ├─ MongoDB 8.2.6   librechat-app-db-0 (StatefulSet, 30Gi, no auth)   ← nightly backup (converse-chat)
                             ├─ Meilisearch v1.35.0  librechat-search:7700 (separate Application, wave -1)
                             ├─ redis-ha (home-os, ns redis-system) via HAProxy, rediss:// + password + internal CA
                             ├─ Hetzner Object Storage  nbg1 / bucket ssegning-k8s-state  (fileStrategy: s3)
                             ├─ Code Interpreter  codeapi-api.librechat-sandbox:3112  (Ed25519 JWT per request)
                             ├─ MCP: GitHub (Copilot MCP), Lightbridge self-service, Coder, Code Intelligence,
                             │       terraform/context7/refero (via api.ai.camer.digital/mcp/*), draw.io
                             ├─ Web search: Serper → Firecrawl → Jina
                             └─ ALL model calls ──► Envoy AI Gateway, INTERNAL plane
                                   https://core-gateway-internal.envoy-gateway-system.svc.cluster.local/v1
                                   Authorino: static apiKey (CONVERSE_OPENAI_API_KEY) + X-LibreChat-User → per-user descriptors
                                   → per-model RPM (Lyft ratelimit) → model backends (DeepInfra, Fireworks, Google, self-hosted GPU)
```

The ArgoCD control objects live on a **different cluster** (`admin@homeos`, ns `argocd`); workloads run on `home-remote`.

## B2. Where everything is defined (memorise this table)

| Concern | Repo | Path |
|---|---|---|
| Orchestrator (ApplicationSet, 2 children) | ai-helm | `charts/librechart/` |
| LibreChat + Mongo chart (env, secrets, ingress, HPA, seed job) | ai-helm | `charts/librechat-app/` |
| Meilisearch chart | ai-helm | `charts/librechat-search/` |
| Code Interpreter chart (flat app, own ns) | ai-helm | `charts/librechat-code-interpreter/` + entry in `charts/apps/values.yaml` |
| opencode well-known chart (now a child of **ai-models**) | ai-helm | `charts/librechat-opencode-wellknown/`, wired in `charts/ai-models/templates/applicationset.yaml` |
| The `librechat.yaml` content, `config-rollout` generation, agent fleet | **ai-helm-values** (private) | `environments/prod/values/librechat-app.yaml` |
| How the gateway treats LibreChat (identity, plan) | **ai-helm-values** | `environments/prod/values/security-policies.yaml` (internal AuthConfig) |
| Model catalog, per-model RPM, prices | **ai-helm-values** | `environments/prod/values/models.yaml` |
| Code Interpreter egress policy | **ai-helm-values** | `environments/prod/deps/librechat-code-interpreter/` |
| Mongo backup | ai-helm `charts/mongodb-backup` + ai-helm-values `values/mongodb-backup.yaml`, `deps/mongodb-backup/` |
| Shared skills synced into LibreChat | ai-helm | `skills/` |
| Redis, cert-manager issuers, Traefik, ESO | home-os / external | — (consumed only) |
| Root Application `ai-apps-v2` | home-os | `charts/cd` |

Decisions to cite: ADR-0014 (split), ADR-0017 (destinations), ADR-0021 (internal plane + per-user attribution),
ADR-0053 (vanity redirect), ADR-0055/0056 (OCI + values repo), ADR-0086/0088 (agent fleet + seed),
ADR-0087 (config to values repo), ADR-0122 (self-hosted code interpreter), ADR-0125 (models from `/v1/models`,
well-known moved), ADR-0138 (image generation withdrawn).

## B3. Deploy pipeline — from a commit to a running pod

1. **Chart change** merged to `ai-helm` main → `publish-charts-oci` publishes `oci://ghcr.io/adorsys-gis/charts/<chart>`.
2. **Root** `ai-apps-v2` (tracks `main`) renders `charts/apps` → the `librechat` Application (`controlPlane: true`)
   deploys the `librechart` chart **into `argocd` on the ArgoCD cluster**.
3. `librechart` renders **one ApplicationSet** (`librechat`), List generator, **two** elements:
   `librechat-search` (wave −1), `librechat-app` (wave 0). Each child Application has **two sources**:
   - Source A: `oci://ghcr.io/adorsys-gis/charts/<child>` @ `>=0.0.0` (floats to newest)
   - Source B: `ai-helm-values@main` as `ref: values`; valueFiles
     `$values/environments/prod/values/<child>.yaml`, `ignoreMissingValueFiles: true`
   - destination `home-remote` / ns `converse`; `prune` + `selfHeal`; `CreateNamespace`, `ServerSideApply`.
4. **Config change** (edit `config:` in `ai-helm-values`) → re-render → ConfigMap `librechat-config` changes.
   Because the file is mounted with **`subPath`**, the pod never sees it **unless** you also bump
   `librechat.configMaps.config-rollout.data.generation` (live: **"14"**) in the same commit → checksum
   annotation changes → rolling restart.
5. **PostSync**: Job `librechat-app-agent-seed` reconciles the agent fleet (B6).

⚠️ *Values-repo-first*: if `librechat-app.yaml` were missing, `ignoreMissingValueFiles` would render the chart
with **no `config:`**, the `{{ with .Values.config }}` guard would emit **no ConfigMap**, and LibreChat would start
with no endpoints — a broken chat, not a failed sync.

## B4. Identity & per-user attribution — the most important integration point

There are **two separate identities** on every chat request:

1. **User → LibreChat**: OIDC against Keycloak (`converse` client). `ALLOW_SOCIAL_REGISTRATION=true` ⇒ the first
   SSO login auto-creates the LibreChat user. `OPENID_REUSE_TOKENS=true` ⇒ the Keycloak refresh token is the session.
   `OPENID_REQUIRED_ROLE` is **commented out** ("open beta") ⇒ **any** realm user can log in today; the
   `librechat_roles` path/kind variables are set but inert.
2. **LibreChat → gateway**: LibreChat authenticates **as itself** with one static key,
   `CONVERSE_OPENAI_API_KEY` (Authorino Secret `internal-key-librechat`, label `kuadrant.io/apikey-for=internal-gateway`)
   on the **internal plane** — **and forwards the human** in templated headers:

```yaml
headers:
  X-LibreChat-User:  '{{LIBRECHAT_USER_OPENIDID}}'   # Keycloak sub
  X-LibreChat-Role:  '{{LIBRECHAT_USER_ROLE}}'
  X-LibreChat-Email: '{{LIBRECHAT_USER_EMAIL}}'
  X-LibreChat-Name:  '{{LIBRECHAT_USER_NAME}}'
```

The internal AuthConfig (CEL) then **overwrites** the rate-limit descriptors:

| Descriptor | Value when `X-LibreChat-User` is present (and not a raw `{{…}}` placeholder) |
|---|---|
| `x-account-id` | the user's Keycloak `sub` — **same key as their opencode usage** ⇒ one person, one budget |
| `x-org-id` | the same `sub` |
| `x-billing-plan` | `pro` if `X-LibreChat-Role == "pro"`, else `free` |
| `x-oidc-email` / `x-oidc-name` | from the forwarded email/name (per-user dashboards show names, not `missing:*`) |
| `x-api-key-id`, project headers | constant `""` (LibreChat has no Lightbridge project concept) |
| budget metadata | `enforced: false` — the budget-limiter Lua filter does **not** apply to the internal plane |

Why the `{{` guard exists: on 2026-08-01 a real request arrived with `x-account-id` literally
`{{LIBRECHAT_USER_OPENIDID}}` — LibreChat occasionally fails to substitute the placeholder; without the guard,
unrelated users would pool into one bucket. Note `LIBRECHAT_USER_ROLE` is LibreChat's role (`USER`/`ADMIN`),
so in practice LibreChat users land on `free`.

**Trust model**: forwarding identity in a header is only safe because (a) the internal listener is ClusterIP-only
and accepts only authenticated first-party services, and (b) Authorino *overwrites* the descriptors — a client can't
pick its own plan.

## B5. Models & the gateway

- **Model list**: `models.fetch: true` — at startup LibreChat calls `GET /v1/models` on the internal gateway, which
  returns **all** models including `-internal` ones (ADR-0125). `default: [qwen3-4b-local]` is only the fallback.
  The public plane (`api.ai.camer.digital/v1/models`) returned 24 models today, without `-internal` ids.
- **tokenConfig** is still **static** (context + prices copied by hand from `models.yaml`) — a new model appears
  in the picker automatically but without context/cost until someone adds its entry.
- **Rate limiting today** (models.yaml, 2026-09-05 onwards): the monthly µ$ cost buckets were **deleted**; the
  per-minute burst buckets were removed 2026-08-01. What remains on each model's own `BackendTrafficPolicy` is a
  **per-model requests-per-minute** limit (`rpmPerKey`, default 60) with two rules — `x-api-key-id × model` and
  `x-account-id × model`. For LibreChat the first is empty (no descriptor), so the **second** applies: N req/min
  **per user per model**. Money limits are the Lightbridge **budget limiter** (402 `budget_exhausted`) — which the
  internal plane explicitly opts out of. LibreChat's own `balance` is **disabled**.
- **Image generation is OFF** (ADR-0138, 2026-09-15): the GPU node `hetzner-k8s-gpu-2` was removed, the one card
  went to the text model, `IMAGE_GEN_OAI_*` env vars were deleted, the `image-creator` spec/agent withdrawn, and
  `filteredTools` blocks all five image tools. Don't leave `IMAGE_GEN_OAI_API_KEY` set without `IMAGE_GEN_OAI_BASEURL`
  — the tool falls back to `api.openai.com` and would leak our gateway key.

## B6. Personas, agents, MCP — what users actually see

**Model specs** (picker, git-managed, `interface.modelSelect: false`):

| Spec | Model | Extras |
|---|---|---|
| **Converse** (default) | deepseek-v4-flash-0731 | web search, file search, code execution |
| Repo Concierge | deepseek-v4-flash-0731 | GitHub issues/projects/PRs/repos MCPs |
| Reviewer | deepseek-v4-flash-0731 | `code-review` skill, GitHub PRs/repos |
| Deep Reviewer | → DB agent `agent_OTvNLyzJLQt85SZd1xgTe` | delegates to Security + Test Coverage reviewers |
| Self-Service | deepseek-v4-flash-0731 | `lightbridge_self_service` MCP (accounts, projects, API keys) |
| Researcher | deepseek-v4-flash-0731 | web + files + context7 |
| Designer | deepseek-v4-flash-0731 | Refero MCP |

**Seeded DB agents** (`agentSeed`, author `platform@ai.camer.digital`): Security Reviewer, Test Coverage Reviewer,
Deep Reviewer (orchestrator), `coder` (model `adorsys-coder-pro-internal`, Coder MCP + execute_code).

**Seed job mechanics** (`files/seed-agents.js`): find the platform user in Mongo → mint a 30-minute LibreChat JWT
with `JWT_SECRET` (browser User-Agent, because LibreChat rejects non-browser UAs) → `GET /api/agents` → for each
agent **PATCH** (exists by *name*) or **POST** → grant **public view** via `PUT /api/permissions/agent/<_id>` →
leaves first, orchestrators second (subagent ids must exist) → resolve `skills` names to ids (warn, not fail, if
not synced yet) → **prune** platform-authored agents no longer listed. Consequence: a **rename** = new agent id +
old one deleted (any modelSpec `agent_id` pointing at it breaks).

**MCP servers**: GitHub ×5 (direct to `api.githubcopilot.com`, per-user GitHub OAuth — note the very broad scope
list), `lightbridge_self_service` (`mcp.ai.camer.digital`), `coder_mcp` (Coder OAuth, `coder:all`),
`terraform_mcp` / `context7_mcp` / `refero_mcp` (through the gateway's `/mcp/*` with Keycloak OAuth, client
`self-service-mcp-api`), `lightbridge_code_intelligence`, `drawio` (public, no auth; returns XML, no inline render).
Brave/firecrawl MCPs are intentionally omitted (the web-search pipeline covers them). Users may add their own MCP
servers (`interface.mcpServers.create: true`) but not share them.

Other live config: summarization with `glm-4.7-flash` at 70% context; skill sync from `ai-helm/skills` every 60 min;
`fileStrategy: s3` with `S3_URL_EXPIRY_SECONDS=604800` (7 days — default 2 min broke images); agents
`recursionLimit 75/max 100`; titles via `gemma-4` (but `titleConvo: false` on the endpoint).

## B7. Data stores

| Store | Ours? | Facts |
|---|---|---|
| MongoDB 8.2.6 | yes (`librechat-app`, helmforge `mongodb@1.7.6` as `db`) | standalone, 1-member rs `rs0`, **auth disabled**, 30Gi on cluster default SC (`hcloud-volumes`), runs as uid 999 with read-only rootfs + `/tmp` emptyDir; URI `mongodb://librechat-app-db-0.librechat-app-db-headless:27017` (no db name ⇒ default db `test`); PDB minAvailable 1 ⇒ **0 allowed disruptions** (blocks node drains — by design) |
| Meilisearch v1.35.0 | yes (`librechat-search`, chart `meilisearch@0.25.1`) | `MEILI_ENV=production`, master key from `librechat-meili-config` (ESO), PVC, plain HTTP 7700 |
| redis-ha | **no** (home-os) | via `redis-ha-haproxy.redis-system:6379`, TLS-only, password, prefix `librechat-prod-v2`, `APP_CONFIG,CONFIG_STORE,STARTUP_CONFIG` forced in-memory |
| Hetzner Object Storage | shared bucket | `ssegning-k8s-state`, path-style, creds = the platform `s3_backup_cnpg_*` material |
| Backups | `mongodb-backup` CronJob in ns `converse-chat` | daily `0 2 * * *`, ~18 s, last three runs Complete; dumps to S3 |

## B8. Secrets & TLS trust

13 ExternalSecrets in `converse` (all `SecretSynced`, ClusterSecretStore `ssegning-aws`, refresh 1h):
`librechat-main-config`, `-config`, `-openid-config`, `-meili-config`, `-mcp-cd-credentials`, `-mcp-github`,
`-mcp-coder-credentials`, `-websearch-config`, `-skills-sync`, `-code-config` (dormant), **`-codeapi-jwt`**
(Ed25519 private key for Code Interpreter JWTs), `-s3-config` (templated: static region/bucket + fetched keys),
`redis-ha-redis-auth` (copy of the platform redis password). App secrets come from key
`ai/camer/digital/prod/env`; platform ones (redis, S3) from `prod/meta/test-app`.

⚠️ ESO refreshes the **Secret** hourly, but env vars from `secretKeyRef` are read **at container start** — a rotated
secret reaches LibreChat only after a pod restart.

**Internal CA**: `librechat-internal-ca` is a throwaway cert-manager Certificate from ClusterIssuer `self-signed-ca`;
only its `ca.crt` (the "Home SSegning Root CA") matters. Mounted at `/etc/internal-ca`, used by
`NODE_EXTRA_CA_CERTS` (gateway internal listener TLS) and `REDIS_CA` (redis TLS).

## B9. Networking

| Object | Detail |
|---|---|
| Ingress `librechat-app` | `ai.camer.digital` `/` → `librechat-app:3080`, Traefik, TLS `ai.camer.digital-tls` via `cert-home-cert-http` (ACME HTTP-01) |
| Ingress `librechat-kivoyo-redirect` | `ai.kivoyo.com` + Middleware `kivoyo-redirect` (redirectRegex, non-permanent): **302 on GET**, **307 on HEAD/other methods** (verified with curl), path+query kept |
| Ingress `models-opencode-wellknown` | same host, path `/opencode/.well-known/opencode` (Exact), shares the TLS secret |
| Services | `librechat-app:3080`, `librechat-app-db` + `-headless:27017`, `librechat-search:7700` |
| HPA | 1–4 replicas, CPU 70% / memory 80% (live: 2 replicas, memory ~52%) |
| PDBs | `librechat-app` minAvailable 1; `librechat-app-db-pdb` minAvailable 1 |
| NetworkPolicy | `librechat-app` = **allow-all** ingress+egress; `converse` has **no** default-deny baseline (unlike `apps`/`data`/`observability`/`platform`) |

## B10. Neighbours you must be able to explain

- **Code Interpreter** (`librechat-code-interpreter`, ADR-0122): flat app in its **own namespace
  `librechat-sandbox`** with **privileged** Pod Security (NsJail needs `SYS_ADMIN` etc.; no `/dev/kvm` for the safer
  microVM mode). Seven components: `api` (3112), `service-worker`, `sandbox-runner` (2000), `file-server` (3000),
  `tool-call-server` (3033), `egress-gateway` (3190), `package-init` Job. Images from the `ADORSYS-GIS/code-interpreter`
  fork (upstream publishes none), pinned `sha-d3b0f05`. Auth = LibreChat-minted Ed25519 JWT (kid `lc-codeapi-2026-05`).
  Shares redis-ha and the S3 bucket; a CiliumNetworkPolicy lets only file-server reach `*.your-objectstorage.com`.
  RWO packages PVC ⇒ sandbox-runner pinned to 1 replica. Sync options intentionally exclude `Replace=true`.
- **opencode well-known** (`models-opencode-wellknown`): moved from `librechart` to the **`ai-models`** orchestrator
  (ADR-0125) because its per-model tuning is derived from the model catalog. Serves `{auth, config}`; plugins
  `@vymalo/opencode-lightbridge@0.17.0` (device-code login against **`authz-idp` at `https://auth.ai.camer.digital`**,
  client `opencode-cli` — ADR-0135, *not* Keycloak any more), `-models-info`, `-ratelimit`, `-browser`,
  `opencode-skills-collection`. It shares the host but is **not part of LibreChat**.
- **mongodb-backup**: separate flat app in `converse-chat`; its own copy of `librechat-s3-config` via a deps overlay
  (Secrets are namespace-scoped — the missing copy broke every backup until 2026-08-02).

## B11. Security posture — honest assessment

Strengths: SSO-only (no local accounts), all secrets via ESO, TLS to Redis and the gateway with CA verification,
hardened pod/container security contexts, LibreChat never holds vendor API keys (the gateway does), per-user
attribution at the gateway, image-gen tool explicitly blocked, Code Interpreter isolated in its own namespace.

Weaknesses to be ready to discuss:
- MongoDB has **no authentication** and LibreChat's NetworkPolicy is **allow-all**, with no namespace baseline —
  any pod that can route to `converse` can read every conversation.
- `OPENID_REQUIRED_ROLE` is off — every realm user gets in.
- `DEBUG_OPENID_REQUESTS=true` in production (verbose auth logs).
- GitHub MCP OAuth requests a very broad scope set (incl. `delete_repo`, `admin:org`).
- Code Interpreter containers run as root (fork images have no `USER`); mitigated by caps drop/seccomp elsewhere.
- Single Mongo member: no HA; recovery = restore from the nightly dump.
- `JWT_SECRET` holders can impersonate any LibreChat user (that is how the seed job works).

## B12. Live issues observed on 2026-10-05 (great discussion material)

1. **Skill sync failing hourly**: `[GitHubSkillSync] Source "ai-helm-skills" failed: skills/governance/SKILL.md
   contains invalid YAML frontmatter: bad indentation of a mapping entry (2:113)`. Root cause: the unquoted
   `description:` contains `repository: Epic` — a second `: ` inside a plain scalar. Fix = quote the value (or use `>-`).
   Introduced by `7b1bb1f1` (#828). Lab D-I8 walks through it.
2. `redis client error: Socket closed unexpectedly` roughly hourly — reconnects automatically; the docs flag
   HAProxy/idle-timeout noise as usually benign (see `docs/architecture/06-networking-tls.md`). Worth watching, not alarming.
3. Placeholder leak (`{{LIBRECHAT_USER_OPENIDID}}` arriving literally) — mitigated by the gateway guard, root cause untraced.

## B13. Upgrade notes — v0.8.7 → v0.8.8

- Change `global.librechat.version` in `charts/librechat-app/values.yaml` (also the seed-job image default).
- Validate `CREDS_KEY`/`CREDS_IV` format first — v0.8.8 refuses to start on invalid AES credentials.
- Review config 1.3.15–1.3.17 changes (tenant-scoped custom endpoints, MCP OAuth hardening, web-search credential
  pairing validation, Redis stream coalescing defaults) and bump `config.version` deliberately.
- Re-test: placeholder headers (B4), the seed script's API calls (`/api/agents`, `/api/permissions`), MCP OAuth
  logins, Code Interpreter JWT auth.
- Bump `generation` if `config:` changes in the same rollout.

---

# Part C — Corrections to the earlier study material

| Earlier claim | Reality (2026-10-05) | Source |
|---|---|---|
| librechart has **3** children incl. opencode-wellknown | **2** children; well-known moved to `ai-models` (app `models-opencode-wellknown`) | `charts/librechart/values.yaml`, ADR-0125 |
| No Code Interpreter in the family | Self-hosted Code Interpreter in `librechat-sandbox` (flat app) | ADR-0122 |
| Users without `librechat_roles` get 401 | `OPENID_REQUIRED_ROLE` is commented out — no role gate | chart env |
| All users share one key; per-user quotas are in `librechat.yaml` | One **auth** key, but `X-LibreChat-User` gives **per-user** gateway descriptors; LibreChat `balance` is disabled | ADR-0021, security-policies.yaml |
| Request passes model-policy (403) + budget-limiter (402) + Lightbridge key validation | Internal plane: static apiKey, constant-empty project headers, `budget.enforced=false`; per-model RPM per user remains | security-policies.yaml, models.yaml |
| "Keycloak session check via Redis" on every request | LibreChat validates its own session; with token reuse it refreshes against Keycloak only when needed | upstream docs |
| `IMAGE_GEN_OAI_*` → gemini-2.5-flash-image | Removed (ADR-0138); `filteredTools` blocks all image tools | chart + values |
| Code Interpreter = managed librechat.ai | Self-hosted; `LIBRECHAT_CODE_API_KEY` is dormant | ADR-0122 |
| opencode logs in via Keycloak `opencode-cli` with `@vymalo/opencode-oauth2@0.12.0` | `@vymalo/opencode-lightbridge@0.17.0` against `authz-idp` (`auth.ai.camer.digital`) | ADR-0135 |
| Agent seed = idempotent upsert only | Also grants public view, resolves skills, and **prunes** removed agents | `seed-agents.js`, ADR-0138 erratum |
| Kivoyo redirect is a 302 | 302 for GET, 307 for HEAD/other methods | curl, live |
| `librechat-app` NetworkPolicy satisfies a Cilium baseline in `converse` | `converse` has no baseline; the policy is allow-all | live `kubectl` |
| LibreChat model list is static | `models.fetch: true` from the gateway's `/v1/models` | ADR-0125 |
| `librechat-app` values 742 lines, well-known 2125 lines | 814 and 2245 lines | upstream main |

---

# Part D — Practicals

> **Safety rules for anything touching our cluster**
> - Read-only verbs only: `get`, `describe`, `logs`, `port-forward` to *read*. No `edit`, `delete`, `rollout`,
>   `scale`, `exec` that changes anything. ArgoCD selfHeal reverts manual edits anyway — and they can still cause an outage first.
> - **Never print Secret values** and never dump conversation documents — that is real user data.
> - Changes go through git (fork → PR), never `kubectl apply`.
> - Our context is `hetzner-prod` (= `home-remote`). ArgoCD objects live on `admin@homeos`, which you may not have.

## D-G. General labs (your laptop, Docker)

### G1 — Run LibreChat locally (30 min)
```bash
git clone https://github.com/LibreChat-AI/LibreChat.git && cd LibreChat
git checkout v0.8.7                       # same version as prod; try v0.8.8 later (G9)
cp .env.example .env
docker compose up -d                      # api + mongodb + meilisearch + rag_api + vectordb
docker compose ps
open http://localhost:3080                # register the first user (becomes ADMIN)
```
**Observe**: which containers start and why (map them to A2). `docker compose logs api | head -50`.
**Question**: which variables in `.env` would you never put in `librechat.yaml`, and why?

### G2 — Add a custom endpoint (free, local model)
Run Ollama (`ollama run qwen2.5:0.5b`) and create `librechat.yaml` (mount it via `docker-compose.override.yml`):
```yaml
version: 1.3.5
endpoints:
  custom:
    - name: "local"
      apiKey: "ollama"
      baseURL: "http://host.docker.internal:11434/v1"
      models: { default: ["qwen2.5:0.5b"], fetch: true }
      titleConvo: true
      titleModel: "current_model"
```
```yaml
# docker-compose.override.yml
services:
  api:
    volumes:
      - ./librechat.yaml:/app/librechat.yaml
```
`docker compose up -d` → pick *local*. **Then edit the yaml without restarting**: nothing changes (A3).
Restart → it does. That is exactly why we need the `generation` bump in prod.

### G3 — See header placeholders with your own eyes (the key to B4)
```bash
docker run -d --name echo -p 8081:8080 mendhak/http-https-echo:31
```
Add a second custom endpoint `baseURL: "http://host.docker.internal:8081/v1"` with
```yaml
headers:
  X-LibreChat-User: "{{LIBRECHAT_USER_ID}}"
  X-LibreChat-Email: "{{LIBRECHAT_USER_EMAIL}}"
  X-LibreChat-Role: "{{LIBRECHAT_USER_ROLE}}"
```
Send a chat (it will error — the echo server isn't a model), then `docker logs echo | grep -i x-librechat`.
**You just reproduced how our gateway learns who the human is.** Bonus: compare with `{{LIBRECHAT_USER_OPENIDID}}`
after G6 (empty for local accounts — why?).

### G4 — Model specs, the picker, and an ephemeral agent
Add `modelSpecs` with two personas (one with `webSearch: false`, one with a `promptPrefix`), set
`interface.modelSelect: false`, restart. Then flip `modelSpecs.enforce: true` and note what disappears.

### G5 — Agents & Mongo
Create an agent in the UI. Then inspect it:
```bash
docker compose exec mongodb mongosh LibreChat --quiet --eval 'db.agents.find({}, {id:1,name:1,author:1,tools:1}).toArray()'
docker compose exec mongodb mongosh LibreChat --quiet --eval 'db.getCollectionNames()'
```
**Find**: `author` (an ObjectId), `id` (`agent_…`) vs `_id`. Relate to why our seed resolves `_id` before calling
`/api/permissions` (B6). (Locally the DB is named `LibreChat`; in prod the URI has no db name ⇒ `test`.)

### G6 — OIDC with a local Keycloak
```bash
docker run -d --name kc -p 8082:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:26.0 start-dev
```
Create realm `lab`, confidential client `librechat` (redirect `http://localhost:3080/oauth/openid/callback`), a user.
Set in `.env`: `OPENID_ISSUER=http://host.docker.internal:8082/realms/lab`, `OPENID_CLIENT_ID`, `OPENID_CLIENT_SECRET`,
`OPENID_SESSION_SECRET=<random>`, `OPENID_SCOPE="openid profile email"`, `OPENID_CALLBACK_URL=/oauth/openid/callback`,
`ALLOW_SOCIAL_LOGIN=true`, `ALLOW_SOCIAL_REGISTRATION=true`. Restart, log in.
Then: (a) `OPENID_AUTO_REDIRECT=true`; (b) add a client role, set `OPENID_REQUIRED_ROLE=<role>`,
`OPENID_REQUIRED_ROLE_TOKEN_KIND=access`, `OPENID_REQUIRED_ROLE_PARAMETER_PATH=resource_access.librechat.roles` —
log in without the role (denied), grant it (allowed). (c) Set `ALLOW_SOCIAL_REGISTRATION=false` and log in with a
brand-new Keycloak user — what happens? Now you can explain our prod settings from experience.

### G7 — MCP
Add the public draw.io MCP (same as prod):
```yaml
mcpServers:
  drawio:
    type: streamable-http
    url: "https://mcp.draw.io/mcp"
    serverInstructions: "Produce draw.io diagrams."
```
Use it from an agent; note the result is XML text, not a rendered diagram.

### G8 — Redis & multi-replica
Add a `redis:7` service, set `USE_REDIS=true`, `REDIS_URI=redis://redis:6379`, `REDIS_KEY_PREFIX=lab`, then
`docker compose exec redis redis-cli --scan --pattern 'lab*' | head`. Discuss: what breaks with 2 api replicas and no Redis?

### G9 — Upgrade rehearsal
`git checkout v0.8.8`, `docker compose pull && docker compose up -d`. Read the startup logs: config-version
warnings, AES credential validation. Write down what you'd need to change in our chart (B13).

## D-I. Infra labs (read-only, against `hetzner-prod`)

### I1 — Map the live deployment
```bash
kubectl -n converse get deploy,sts,svc,ingress,hpa,pdb -l 'app.kubernetes.io/instance in (librechat-app,librechat-search,models-opencode-wellknown)'
kubectl -n converse get pods -o wide | grep -E 'librechat|wellknown'
kubectl -n librechat-sandbox get pods,svc,pvc
kubectl -n converse-chat get cronjob,jobs
kubectl get ns converse librechat-sandbox --show-labels    # spot the privileged PSS label
```
**Explain** every object to yourself using B7–B10. Which node does Mongo run on? What happens to chat if that node dies?

### I2 — Render the charts yourself (no cluster needed)
```bash
cd ai-helm   # upstream checkout, on main
helm dep build charts/librechart && helm template librechat charts/librechart | less       # 1 ApplicationSet, 2 elements
helm dep build charts/librechat-app
helm template librechat-app charts/librechat-app -n converse \
  -f ../ai-helm-values/environments/prod/values/librechat-app.yaml > /tmp/lc.yaml
grep -n 'kind:' /tmp/lc.yaml | sort | uniq -c | sort -rn | head
grep -n 'checksum/configMaps' /tmp/lc.yaml
```
Now render **without** the values file and compare: the `librechat-config` ConfigMap disappears and the seed Job too
— the "values-repo-first" hazard, seen with your own eyes.

### I3 — The generation trap
```bash
kubectl -n converse get cm librechat-app-config-rollout -o jsonpath='{.data.generation}'; echo
grep -n 'generation:' ../ai-helm-values/environments/prod/values/librechat-app.yaml
kubectl -n converse get deploy librechat-app -o jsonpath='{.spec.template.metadata.annotations}'; echo
```
**Explain**: what would happen if someone changed `config:` but not `generation`? How would you detect it?
(Hint: compare `kubectl -n converse get cm librechat-config -o jsonpath='{.data.librechat}'` — the new content —
with the pod start time.)

### I4 — Verify non-secret runtime settings
```bash
kubectl -n converse get deploy librechat-app -o json \
 | jq -r '.spec.template.spec.containers[0].env[] | select(.value!=null) | "\(.name)=\(.value)"' \
 | grep -E 'OPENID|REDIS_URI|MEILI_HOST|DOMAIN|S3_URL|CODE'
```
Confirm: no `OPENID_REQUIRED_ROLE`, no `IMAGE_GEN_*`. (This reads the Deployment spec, not the running process — no exec.)

### I5 — Secrets pipeline without reading secrets
```bash
kubectl -n converse get externalsecret
kubectl -n converse describe externalsecret librechat-s3-config | sed -n '/Spec:/,/Status:/p'   # see the template
kubectl -n converse get secret librechat-config -o json | jq '.data | keys'                      # KEY NAMES ONLY
kubectl -n converse get certificate librechat-internal-ca
```

### I6 — Public surface checks
```bash
curl -s https://ai.camer.digital/health                                   # 200 OK
curl -s https://ai.camer.digital/api/config | jq '{openidLoginEnabled, openidAutoRedirect, emailLoginEnabled, registrationEnabled}'
curl -sI https://ai.kivoyo.com/c/new | grep -iE '^HTTP|^location'           # 307 (HEAD)
curl -s -o /dev/null -w '%{http_code} -> %{redirect_url}\n' 'https://ai.kivoyo.com/c/new?x=1'   # 302 (GET)
curl -s https://ai.camer.digital/opencode/.well-known/opencode | jq '.config.plugin | map(if type=="array" then .[0] else . end)'
curl -s https://api.ai.camer.digital/v1/models | jq '.data | length'       # public list (no -internal)
```
**Question**: `/api/config` is what the SPA reads before login — what does it reveal, and is that a problem?

### I7 — Meilisearch & Mongo health (read-only)
```bash
kubectl -n converse port-forward svc/librechat-search 7700:7700 &
curl -s localhost:7700/health; curl -s localhost:7700/version     # /version needs the key → 401 shows auth works
kill %1
kubectl -n converse port-forward svc/librechat-app-db 27017:27017 &
mongosh --quiet mongodb://localhost:27017/test --eval 'db.getCollectionNames(); db.stats().dataSize'
kill %1
```
⚠️ Only metadata. Don't `find()` users or messages. Note how little stood between you and the data (B11).

### I8 — Triage a real failure: the skill sync
```bash
kubectl -n converse logs deploy/librechat-app --since=2h | grep -i skillsync
```
Reproduce locally:
```bash
python3 -c "import yaml,sys; t=open('skills/governance/SKILL.md').read().split('---')[1]; yaml.safe_load(t)"
```
You'll get the same "mapping values are not allowed" error. **Exercise**: on a branch of your fork, quote the
`description:` value, re-run the parser, and draft a PR (`fix(skills): quote governance skill description`) —
don't merge from a study branch. This is a genuine upstream bug; tell the maintainer.

### I9 — Agent seed after a sync
The Job lives 10 min after finishing (`ttlSecondsAfterFinished: 600`). Right after a librechat-app sync:
```bash
kubectl -n converse get jobs | grep agent-seed
kubectl -n converse logs job/librechat-app-agent-seed
```
Expect `author=platform@…`, `patched …`, maybe `pruned …`, `done: N agents`. **Question**: what if the
platform user never logged in? (the Job fails closed — find the line in `seed-agents.js`.)

### I10 — Per-user attribution in observability
In Grafana (`observe.camer.digital`), open the per-user / user-directory dashboards, or in Explore → Loki:
```logql
{service_name="envoy-ai-gateway"} | json | azp="" | user_id!="" | line_format "{{.user_id}} {{.model}}"
```
(Label names follow the Alloy pipeline in ADR-0046 — if the query returns nothing, copy the query from the
per-user dashboard panel instead.) Find LibreChat traffic (no `azp`, a Keycloak-UUID `user_id`). Then: how would `{{LIBRECHAT_USER_OPENIDID}}` look if
it leaked? (B12-3). See `docs/patterns/per-user-observability.md`.

### I11 — Code Interpreter isolation
```bash
kubectl -n librechat-sandbox get deploy codeapi-sandbox-runner -o json \
 | jq '.spec.template.spec.containers[0].securityContext'
kubectl -n librechat-sandbox get networkpolicy
kubectl -n librechat-sandbox get ciliumnetworkpolicy
```
**Explain**: why its own namespace, why `privileged` PSS, why a CNP only for file-server, why 1 replica.

### I12 — Backup
```bash
kubectl -n converse-chat get cronjob mongodb-backup -o yaml | grep -E 'schedule|image:'
kubectl -n converse-chat logs job/$(kubectl -n converse-chat get jobs -o name | tail -1 | cut -d/ -f2) --all-containers | tail -20
```
**Question**: what's the RPO (data you could lose) with a single Mongo member and a 02:00 daily dump?

## D-X. Change exercises (git only — draft, render, never push to prod from here)

1. **Add a persona**: in a scratch copy of `ai-helm-values/environments/prod/values/librechat-app.yaml`, add a
   `modelSpecs.list` entry; bump `generation`; render with I2; diff the ConfigMap. Write the PR description you'd submit.
2. **Add a model's tokenConfig**: pick a model from `curl …/v1/models` that has no `tokenConfig` entry; copy its
   context/prices from `models.yaml`. Explain what users gain.
3. **Turn on the role gate** (paper exercise): what Keycloak change, what env change, what rollout order, what
   happens to users already logged in?
4. **Draft the v0.8.8 upgrade PR**: list every file and the test plan (B13).
5. **Lock down Mongo** (design exercise): write the NetworkPolicy that allows only `librechat-app`, the seed job
   and the backup CronJob (cross-namespace!) to reach `:27017`. What would break if you forgot the backup?

---

# Part E — Self-check quiz (answers below)

1. Name the two Applications the `librechat` ApplicationSet generates and their waves.
2. Where does the `librechat.yaml` content live, and how does it reach the pod?
3. Why must `generation` be bumped, and what is its live value?
4. What authenticates LibreChat to the gateway, and what identifies the human?
5. Which descriptor makes a LibreChat user's budget shared with their opencode usage?
6. Why does the AuthConfig check `.contains("{{")`?
7. Is there a role gate on LibreChat login today?
8. Which plan does a normal LibreChat user get at the gateway, and why?
9. Why is Mongo's URI pointing at `…-db-0.…-db-headless` instead of the ClusterIP Service?
10. Why `redis-ha-haproxy` and not `redis-ha-redis`?
11. What two env vars use `/etc/internal-ca/ca.crt`?
12. What happens to a seeded agent you remove from `agentSeed.agents`? And if you rename it?
13. Why is the Code Interpreter in its own namespace?
14. Where is the opencode well-known deployed now, and why did it move?
15. Why was image generation withdrawn, and what stops a user re-enabling it via a Gemini key?
16. What does `models.fetch: true` change, and what does it *not* solve?
17. Why did presigned S3 URLs need `S3_URL_EXPIRY_SECONDS=604800`?
18. A Secret was rotated in AWS an hour ago. Does LibreChat use the new value?
19. What's broken in production right now regarding skills, and how would you fix it?
20. What's the first thing that blocks a v0.8.8 upgrade if misconfigured?

<details><summary>Answers</summary>

1. `librechat-search` (−1), `librechat-app` (0).
2. `ai-helm-values` `environments/prod/values/librechat-app.yaml` `config:` → ApplicationSet `$values` valueFile →
   `templates/configmap.yaml` → ConfigMap `librechat-config` key `librechat` → `subPath` mount `/app/librechat.yaml`.
3. `subPath` mounts don't update and LibreChat reads config only at start; bumping changes the checksum annotation → rollout. Live: "14".
4. Static apiKey `CONVERSE_OPENAI_API_KEY` (internal plane); the human via `X-LibreChat-User` (+ Role/Email/Name).
5. `x-account-id` = Keycloak `sub`.
6. LibreChat sometimes sends the raw placeholder; without the guard all such requests would pool into one account.
7. No — `OPENID_REQUIRED_ROLE` is commented out.
8. `free` — `X-LibreChat-Role` is LibreChat's own role (USER/ADMIN), never "pro".
9. `_mongo_uri.tpl` builds a replica-set-style seed list — one stable per-pod DNS name per member
   (`<release>-db-<i>.<release>-db-headless`). With `members: 1` that's a single host. Per-pod names are what a
   replica-set client expects; scaling members up would extend the list automatically.
10. The round-robin Service hits replicas ~50% of the time → `READONLY` errors; HAProxy routes to the current master.
11. `NODE_EXTRA_CA_CERTS`, `REDIS_CA`.
12. Pruned (deleted) on the next seed. A rename creates a new agent (new id) and prunes the old one.
13. Its sandbox needs `SYS_ADMIN`-class capabilities → `privileged` PSS; isolating it keeps that elevation away from LibreChat/Mongo.
14. Under the `ai-models` orchestrator (`models-opencode-wellknown`); only that orchestrator can pass the model catalog it derives from.
15. GPU-2 removed → one card → text model kept (ADR-0138); `filteredTools` blocks `gemini_image_gen` (and the other four).
16. The picker list comes from the gateway's `/v1/models`; `tokenConfig` (context/prices) is still static.
17. LibreChat's default presigned TTL is 2 minutes → images/avatars broke soon after upload.
18. The Secret updates within 1h, but env vars are read at container start — only after a pod restart.
19. Skill sync fails on invalid YAML in `skills/governance/SKILL.md` (unquoted `description` with `: `); quote it.
20. Invalid `CREDS_KEY`/`CREDS_IV` — v0.8.8 rejects them at startup.
</details>

---

# Part F — Sources

Upstream: <https://www.librechat.ai/docs> · custom endpoints
<https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/custom_endpoint> · token reuse
<https://www.librechat.ai/docs/configuration/authentication/OAuth2-OIDC/token-reuse> · v0.8.8 changelog
<https://www.librechat.ai/changelog/v0.8.8> · config 1.3.17 <https://www.librechat.ai/changelog/config_v1.3.17> ·
repo <https://github.com/LibreChat-AI/LibreChat>.

Ours (ai-helm): `charts/librechart/`, `charts/librechat-app/` (`values.yaml`, `templates/`, `files/seed-agents.js`),
`charts/librechat-search/`, `charts/librechat-code-interpreter/`, `charts/librechat-opencode-wellknown/`,
`charts/apps/values.yaml`, `charts/ai-models/templates/applicationset.yaml`, ADRs 0014, 0017, 0021, 0053, 0055,
0056, 0086, 0087, 0088, 0122, 0125, 0135, 0138, `docs/integrations/librechat-*.md`,
`docs/patterns/self-hosted-code-interpreter.md`, `docs/patterns/per-user-observability.md`.

Ours (ai-helm-values): `environments/prod/values/{librechat-app,security-policies,models,core-gateway,mongodb-backup}.yaml`,
`environments/prod/deps/{librechat-code-interpreter,mongodb-backup}/`.

Live (read-only `kubectl`, context `hetzner-prod`, 2026-10-05): namespaces `converse`, `librechat-sandbox`, `converse-chat`.
