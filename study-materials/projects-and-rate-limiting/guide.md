# Projects & Rate Limiting — Complete Operational Guide

> **Scope**: Exactly what exists in this repository and what runs in production today. This covers
> the Lightbridge orchestration stack (the auth/API-key service whose source of truth for project
> membership lives in the external `lightbridge-authz` repo) and the gateway rate-limit system that
> enforces billing plans via the `ai-models` orchestrator and `ai-model` per-model leaf charts.

---

## Table of Contents

1. [The Two Distinct Systems](#1-the-two-distinct-systems)
2. [Lightbridge Stack — How It Is Deployed](#2-lightbridge-stack--how-it-is-deployed)
3. [Lightbridge-secrets Leaf — Every Secret](#3-lightbridge-secrets-leaf--every-secret)
4. [Lightbridge-db Leaf — The Database](#4-lightbridge-db-leaf--the-database)
5. [Lightbridge-app Leaf — The Running Service](#5-lightbridge-app-leaf--the-running-service)
6. [The `ai-models` Orchestrator — Rate Limit Configuration](#6-the-ai-models-orchestrator--rate-limit-configuration)
7. [The `ai-model` Leaf — BackendTrafficPolicy](#7-the-ai-model-leaf--backendtrafficpolicy)
8. [The Five Billing Plans — Exact Current Values](#8-the-five-billing-plans--exact-current-values)
9. [How a Request Gets Its Billing Plan](#9-how-a-request-gets-its-billing-plan)
10. [The Shared Budget — Current Active Behaviour](#10-the-shared-budget--current-active-behaviour)
11. [The Billing Period Marker and Counter Rotation](#11-the-billing-period-marker-and-counter-rotation)
12. [The `tiers` and `projectEnvelope` Fields — Current State](#12-the-tiers-and-projectenvelope-fields--current-state)
13. [The Append-Only List Contract — Why and How It Works](#13-the-append-only-list-contract--why-and-how-it-works)
14. [Redis Key Structure — What Counters Actually Look Like](#14-redis-key-structure--what-counters-actually-look-like)
15. [ArgoCD Sync Behaviour for These Charts](#15-argocd-sync-behaviour-for-these-charts)

---

## 1. The Two Distinct Systems

This guide covers two separate but related components of the platform:

**System A — Lightbridge** (`charts/lightbridge/`): An **App-of-Apps orchestrator** that deploys the
`lightbridge-authz` service — a Rust API that manages API key minting, project membership, and
the Keycloak token-exchange integration. The source of truth for project/member data is the PostgreSQL
database (CloudNativePG) that this stack owns.

**System B — Rate Limiting** (`charts/ai-models/` + `charts/ai-model/`): The gateway-level enforcement
of spending plans. The `ai-models` orchestrator passes plan/tier configuration down to each `ai-model`
leaf chart, which renders a Kubernetes `BackendTrafficPolicy` CR telling the Envoy AI Gateway how
many requests or how much money each user is allowed to spend per model per month.

These two systems interact only through **HTTP headers**: Lightbridge (via Keycloak's AuthConfig)
stamps headers (`x-billing-plan`, `x-account-id`) on each request passing through the Envoy AI
Gateway. The gateway then consults its BackendTrafficPolicy rate-limit rules keyed on those headers.

---

## 2. Lightbridge Stack — How It Is Deployed

**Source file**: [`charts/lightbridge/values.yaml`](file:///Users/gisstudent/ai-helm/charts/lightbridge/values.yaml)

The orchestrator chart `charts/lightbridge/` renders **three static ArgoCD Application CRs**
(not an ApplicationSet) in the `argocd` namespace on the `admin@homeos` cluster. The children
are fixed and heterogeneous, hence static Application CRs rather than an ApplicationSet.

### Shared ArgoCD wiring

| Field | Value |
|---|---|
| `argocd.project` | `ai` |
| `argocd.chartRegistry` | `oci://ghcr.io/adorsys-gis/charts` |
| `argocd.chartVersionRange` | `>=0.0.0` (float for secrets + db leaves) |
| `argocd.repoURL` | `https://github.com/adorsys-gis/ai-helm-values` |
| `argocd.targetRevision` | `main` |
| `argocd.destination.name` | `home-remote` |
| `argocd.destination.namespace` | `converse` |
| Automated prune | `true` |
| Automated selfHeal | `true` |
| syncOptions | `CreateNamespace=true`, `ServerSideApply=true` |

### The three child Applications

| Child app name | ArgoCD name | Sync-wave | Chart version |
|---|---|---|---|
| `lightbridge-secrets` | `lightbridge-secrets` | `0` | floated `>=0.0.0` from OCI |
| `lightbridge-db` | `lightbridge-db` | `1` | floated `>=0.0.0` from OCI |
| `lightbridge-app` | `lightbridge-app` | `2` | **pinned** `0.8.1` |

`lightbridge-app` is **not floated**. It points at `lightbridge-authz-stack` chart version `0.8.1`
from `oci://ghcr.io/adorsys-gis/charts/lightbridge-authz-stack`. This is deliberate: a new
`lightbridge-authz` chart release does not auto-deploy. The workload container image floats
separately via `argocd-image-updater`.

The `lightbridge-app` child also has a **second source** for its ingress TLS Certificates via a
kustomize overlay at `environments/prod/deps/lightbridge-backend` in `ai-helm-values`.

`lightbridge-db` has an `ignoreDifferences` rule targeting `postgresql.cnpg.io/Cluster` and the
`manager` fieldManager — the CNPG operator injects defaults (affinity, bootstrap, probes, etc.)
that the chart never declares; without ignoring these fields the Application would permanently show
OutOfSync.

---

## 3. Lightbridge-secrets Leaf — Every Secret

**Source file**: [`charts/lightbridge-secrets/values.yaml`](file:///Users/gisstudent/ai-helm/charts/lightbridge-secrets/values.yaml)

All ExternalSecrets pull from the `ssegning-aws` `ClusterSecretStore`. Refresh interval: `1h`.

| Kubernetes Secret name | Keys | SM path | Property |
|---|---|---|---|
| `lightbridge-api-client` | `secret-id` | `ai/camer/digital/prod/env` | `lightbridge_api_client_secret_id` |
| `opa-secret` | `OPA_PASSWORD` | `ai/camer/digital/prod/env` | `lightbridge_opa_password` |
| `lightbridge-cnpg-s3` | `s3_region_name` | literal `us-east-1` (not from SM) | — |
| | `s3_access_key_id` | `prod/meta/test-app` | `s3_backup_cnpg_client_id` |
| | `s3_secret_access_key` | `prod/meta/test-app` | `s3_backup_cnpg_secret` |
| `redis-ha-redis-auth` | `redis-password` | `prod/meta/test-app` | `redis_password` |

**Purpose of each secret**:

- `lightbridge-api-client` (`secret-id`): The Keycloak `lightbridge-api-key` client's secret.
  Used by the Lightbridge API pod for the OAuth2 token-exchange flow that mints scoped API-key tokens.
  The pod reads it as env var `API_KEY_CLIENT_SECRET` via `secretKeyRef: {name: lightbridge-api-client, key: secret-id}`.

- `opa-secret` (`OPA_PASSWORD`): The OPA server's basic-auth password. Read by the API and MCP pods
  (their configs carry the OPA listener block) AND by the OPA component itself. The Keycloak
  token-exchange SPI presents `authorino:<this password>` as Basic auth to OPA for context resolution —
  this must match what the Keycloak SPI is configured with.

- `lightbridge-cnpg-s3`: CNPG barman backup credentials for Hetzner Object Storage. The
  `s3_region_name: us-east-1` literal is stored as a secret key (not a config) because the
  barman-cloud plugin's defaulting webhook injects an `s3Credentials.region` ref pointing at this
  secret — it reads the key at backup time. Without this key, every ScheduledBackup fails.

- `redis-ha-redis-auth` (`redis-password`): Redis AUTH password for the Lightbridge API pod's
  Redis-backed rate limiting. This is the **same** platform secret (`prod/meta/test-app /
  redis_password`) that LibreChat and the Envoy rate-limiter use. Each consuming namespace needs its
  own copy; this ExternalSecret provisions the `converse`-namespace copy.

---

## 4. Lightbridge-db Leaf — The Database

**Source file**: [`charts/lightbridge-db/values.yaml`](file:///Users/gisstudent/ai-helm/charts/lightbridge-db/values.yaml)

This leaf chart renders **raw YAML strings** under a `resources:` list — the template iterates them
and emits them verbatim. There is no Helm subchart for CNPG; the Cluster CR is hand-authored YAML
in values.yaml.

### CNPG Cluster: `lightbridge-main-db`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: lightbridge-main-db
  namespace: converse
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:18.4-system-trixie
  storage:
    size: "5Gi"
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters:
        barmanObjectName: lightbridge-main-db
        serverName: lightbridge-main-db
  postgresql:
    parameters:
      archive_timeout: "30min"
  resources:
    limits:   {cpu: 300m,  memory: 1Gi}
    requests: {cpu: 200m,  memory: 500Mi}
```

### Declarative roles on the Cluster

The `managed.roles` block causes CNPG to create and maintain these PostgreSQL login roles automatically:

| Role name | DB it owns | Password secret |
|---|---|---|
| `repoauth` | `repoauth` | `repo-auth-db-role` |
| `codeintel` | `codeintel` | `lightbridge-codeintel-db-role` |
| `grafana_ro` | (none — read-only member of `pg_read_all_data`) | `lightbridge-grafana-ro-db-role` |
| `coder` | (coder's own db, set in coder-secrets) | `coder-db-role` |
| `lakefs` | (mlops) | `lakefs-db-role` |
| `mlflow` | (mlops) | `mlflow-db-role` |

All password secrets are provisioned by External Secrets Operator (not by this chart — they are
ESO targets elsewhere). CNPG reads the secret and sets the role's password accordingly.

The `grafana_ro` role has `inRoles: [pg_read_all_data]` — a PG14+ built-in role granting SELECT
on all tables without per-table grants.

### barman-cloud ObjectStore: `lightbridge-main-db`

```yaml
spec:
  retentionPolicy: "7d"
  configuration:
    destinationPath: s3://ssegning-k8s-state/lightbridge-main-db
    endpointURL: https://nbg1.your-objectstorage.com
    data:
      jobs: 1           # serialised upload — prevents Hetzner S3 rate limiting
      compression: gzip
    s3Credentials:
      accessKeyId:     {name: lightbridge-cnpg-s3, key: s3_access_key_id}
      secretAccessKey: {name: lightbridge-cnpg-s3, key: s3_secret_access_key}
```

`retentionPolicy: "7d"` (reduced from 180d). A 180-day catalog accumulated enough objects that
Hetzner Object Storage's `ListObjectsV2` calls hit rate limits (`SlowDown` / `GatewayTimeout`),
causing every retention prune step to fail. 7 days keeps the catalog small.

`data.jobs: 1` serialises the multipart upload. Parallel `UploadPart` calls were also hitting the
per-request rate limit, failing after 4 retries. Serial upload avoids this.

### ScheduledBackup

```yaml
schedule: "0 0 */6 * * *"   # every 6 hours: 00:00, 06:00, 12:00, 18:00
immediate: true
method: plugin
pluginConfiguration:
  name: barman-cloud.cloudnative-pg.io
```

`immediate: true` triggers a backup immediately on creation of the ScheduledBackup object (not just
on the next scheduled time).

### PodMonitor

Scrapes the CNPG per-instance metrics exporter on port `9187` (name `metrics`). This feeds Grafana
Alloy → the LGTM observability stack.

---

## 5. Lightbridge-app Leaf — The Running Service

The `lightbridge-app` Application pulls the **upstream** `lightbridge-authz-stack` Helm chart at
version `0.8.1` from `oci://ghcr.io/adorsys-gis/charts/lightbridge-authz-stack`. This chart is
published by the `lightbridge-authz` repository's CI — its content is not in `ai-helm`.

What this chart deploys (from the `lightbridge-authz` repo, published as the umbrella chart):
- **api** component: The Rust HTTP API server — handles API key CRUD, project membership queries,
  the OAuth2 token-exchange with Keycloak, and serves the `x-billing-plan` / `x-account-id` claim
  resolution endpoints that Authorino calls.
- **mcp** component: The MCP server exposing Lightbridge self-service tools (project management,
  API key management) over the MCP protocol for use by LibreChat agents.

The `lightbridge` App-of-Apps values.yaml (lines 80-101 in `charts/lightbridge/values.yaml`)
pass the workload values to this upstream chart: api + mcp components only (opa and usage were
removed). The actual workload configuration (image tags, resource limits, replica counts, ingress
hostnames) lives in `ai-helm-values environments/prod/values/lightbridge-app.yaml`.

The **container image** is not pinned in the chart — argocd-image-updater manages image tags
(annotations on the `lightbridge-app` Application CR, managed in `ai-helm-values`).

---

## 6. The `ai-models` Orchestrator — Rate Limit Configuration

**Source file**: [`charts/ai-models/values.yaml`](file:///Users/gisstudent/ai-helm/charts/ai-models/values.yaml)

The `ai-models` chart renders one ArgoCD ApplicationSet. That ApplicationSet generates one child
Application per enabled model (each pointing at `charts/ai-model/`, the per-model leaf chart) plus
one Application for the shared backends chart. Rate-limit configuration (plans, sharedBudget,
tiers, projectEnvelope) is defined in `ai-models/values.yaml` and **propagated into each child
Application's `valuesObject`** — so every per-model `BackendTrafficPolicy` inherits the same plan list.

### ArgoCD wiring

| Field | Value |
|---|---|
| `argocd.namespace` | `argocd` |
| `argocd.project` | `ai` |
| `argocd.chartRegistry` | `oci://ghcr.io/adorsys-gis/charts` |
| `argocd.chartVersionRange` | `>=0.0.0` |
| `argocd.destination.name` | `home-remote` |
| `argocd.destination.namespace` | `converse` |

### `sharedBudget.enabled: true` — currently active

```yaml
sharedBudget:
  enabled: true    # live since 2026-07-08
```

When `true`, every per-model `ai-model` child sets `mergeType: StrategicMerge` on its BTP and
drops the per-model monthly-budget rule (family 3) entirely. The monthly budget instead lives on
the gateway-wide BTP in `charts/core-gateway` with `shared: true` (one shared Redis counter per
user/plan across all models). The per-model BTPs retain only per-model burst rules and timeouts.

### `rateLimitBudgeting` — the top-level key

```yaml
rateLimitBudgeting:
  plans:     [ ...ordered list... ]
  tiers:     []
  projectEnvelope: {}
```

This key and all three sub-keys are passed verbatim into every `ai-model` child's values via the
ApplicationSet's `valuesObject`. The `ai-model` template then reads `.Values.plans`,
`.Values.tiers`, `.Values.projectEnvelope` to render BTP rules.

---

## 7. The `ai-model` Leaf — BackendTrafficPolicy

**Source file**: [`charts/ai-model/templates/backendtrafficpolicy.yaml`](file:///Users/gisstudent/ai-helm/charts/ai-model/templates/backendtrafficpolicy.yaml)

This is a 401-line Helm template. It renders a single `gateway.envoyproxy.io/v1alpha1 BackendTrafficPolicy`
CR named `<modelName>` (e.g. `gpt-4o`, `claude-sonnet-4-5`, etc.). The BTP targets the
`HTTPRoute` of the same name.

### Always-rendered fields (independent of rate-limit rules)

```yaml
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: <modelName>
  mergeType: StrategicMerge    # only when sharedBudget.enabled = true
  retry:
    numRetries: 0              # only when sharedBudget.enabled = true
  timeout:
    http:
      requestTimeout: "600s"   # default; overridable per model via .Values.timeout
```

`numRetries: 0` neutralises the retry the gateway-wide BTP (merged via StrategicMerge) brings in.
LLM calls are never retried — a retry would amplify load on stressed backends and risk double-charging
a partial generation.

### The `$hasRules` pre-computation guard

Before rendering the `rateLimit:` block, the template iterates all plans and tiers to determine if
**any** rule will render. If no rule renders (because all `burst:` blocks are commented out AND
`sharedBudget.enabled` is true AND `tiers` is empty AND `projectEnvelope` is empty), the entire
`rateLimit:` block is **omitted**. This is critical because the BTP CRD requires `rateLimit.global.rules`
to be a non-empty list — emitting an empty list fails the API server validation.

### Rule families (ordered, index-stable)

The template renders up to 5 families in this fixed order:

| Family | Index range | Header keys | Condition to render |
|---|---|---|---|
| 1 (burst req/min) | `rule/0` .. | `x-account-id` (Distinct) + `x-billing-plan` (Exact) + `x-ai-eg-model` (Exact) | `plan.burst.requestsPerMin` set |
| 2 (burst tok/min) | follows 1 | same + cost from `llm_total_token` metadata | `plan.burst.tokensPerMin` set AND not image model |
| 3 (monthly budget) | follows 2 | `x-account-id` (Distinct) + `x-billing-plan` (Exact) + `x-billing-period` (Distinct) | `plan.monthlyBudgetUsd` set AND NOT `sharedBudget.enabled` |
| 4 (tier budget) | follows 3 | `x-project-id` (Distinct) + `x-account-id` (Distinct) + `x-quota-tier` (Exact) + `x-billing-period` (Distinct) | `tier` entries exist in `tiers` |
| 5 (project envelope) | last, single | `x-project-id` (Distinct) + `x-project-quota` (Exact) + `x-billing-period` (Distinct) | `projectEnvelope.monthlyBudgetUsd` set |

**Currently rendered families**:

Since `sharedBudget.enabled: true` AND all `burst:` blocks are commented out AND `tiers: []` AND
`projectEnvelope: {}`, **no rule family renders today**. The `rateLimit:` block is entirely omitted
from every per-model BTP. Per-model BTP today is: targetRefs + mergeType + retry + timeout only.

The rate-limit enforcement happens on the **gateway-wide BTP** in `charts/core-gateway`.

---

## 8. The Five Billing Plans — Exact Current Values

**Source**: [`charts/ai-models/values.yaml`](file:///Users/gisstudent/ai-helm/charts/ai-models/values.yaml) lines 110-157

These are an **ordered list** (not a map). Their index position is preserved exactly as shown here.

```yaml
plans:
  # index 0
  - id: enterprise
    monthlyBudgetUsd: 1000
    # burst: COMMENTED OUT

  # index 1
  - id: free
    monthlyBudgetUsd: 50
    # burst: COMMENTED OUT

  # index 2
  - id: internal
    # no monthlyBudgetUsd (uncapped)
    # burst: COMMENTED OUT

  # index 3
  - id: pro
    monthlyBudgetUsd: 200
    # burst: COMMENTED OUT

  # index 4
  - id: service
    # no monthlyBudgetUsd (uncapped)
    # burst: COMMENTED OUT
```

| Plan id | index | monthlyBudgetUsd | Burst limit | Purpose |
|---|---|---|---|---|
| `enterprise` | 0 | $1,000 | none | Customer enterprise tier (top self-serve) |
| `free` | 1 | $50 | none | Default for authenticated users |
| `internal` | 2 | uncapped | none | In-cluster services (LibreChat, cron jobs, k8s SAs) |
| `pro` | 3 | $200 | none | Pro self-serve tier |
| `service` | 4 | uncapped | none | Remote service accounts (GitHub runners, etc.) |

**`monthlyBudgetUsd` is dormant today**: Because `sharedBudget.enabled: true`, the per-model BTP
drops the monthly-budget rule (family 3). The live `monthlyBudgetUsd` caps are in
`charts/core-gateway` under `backendTrafficPolicy.monthlyBudget.plans`. Both sets of numbers must
be kept in sync so a rollback to per-model budgets preserves the same caps.

**All burst blocks are commented out**: Per-minute burst limits (`requestsPerMin`, `tokensPerMin`)
were removed across all plans on 2026-08-01. They are commented (not deleted) so re-enabling is
one uncomment. The index ordering of plans cannot change regardless of burst state.

---

## 9. How a Request Gets Its Billing Plan

A request arriving at the Envoy AI Gateway goes through the following header-stamping before the
BTP rules evaluate it:

### External plane (`api.ai.camer.digital`)

1. The client presents a JWT (issued by Lightbridge-authz via Keycloak token-exchange).
2. The Authorino `AuthConfig` (in `charts/core-gateway` / `ai-helm-values`) validates the JWT.
3. Authorino reads the JWT claim `billing_plan` and stamps it as the `x-billing-plan` request header.
4. Authorino reads the JWT claim `sub` and stamps it as the `x-account-id` header.
5. The Envoy AI Gateway reads `x-billing-plan` to select which plan's BTP rules apply.

The `billing_plan` claim is set by Keycloak's `lightbridge-api-key` client protocol mapper when the
API key JWT is minted. The claim value is one of: `enterprise`, `free`, `pro`, `service`. If absent,
Authorino defaults to `free`.

### Internal plane (`core-gateway-internal...svc.cluster.local`)

1. The client (e.g. LibreChat) presents a static API key or a Kubernetes service account token.
2. The internal AuthConfig stamps `x-billing-plan: internal` for clients on the internal plane.
3. Internal-plane callers are uncapped (`internal` plan has no `monthlyBudgetUsd` and no burst).

### The `x-billing-period` header (calendar month marker)

A Lua `EnvoyExtensionPolicy` in `charts/core-gateway` stamps `x-billing-period` on every request
with the current calendar month in `YYYY-MM` format (e.g. `"2026-09"`). This header is included in
the Redis counter key for monthly-budget rules. When the month changes, the counter key changes
(because `"2026-09"` → `"2026-10"`), effectively rotating the budget on the 1st of each calendar
month without relying on the rate-limit `unit` field for rotation.

---

## 10. The Shared Budget — Current Active Behaviour

The shared budget (`sharedBudget.enabled: true`) is the **only active rate-limit mechanism** today.
It lives in `charts/core-gateway` (not in `charts/ai-model`). Here is what it actually does:

- One `BackendTrafficPolicy` CR at the gateway level (not per-model) carries a `rateLimit` block
  with `shared: true`.
- The counter is shared across all models — a user spending $50 on any combination of models counts
  against the same $50 bucket.
- The key: `x-account-id` (Distinct) + `x-billing-plan` (Exact) + `x-billing-period` (Distinct).
- Each per-model BTP merges with this gateway BTP via `mergeType: StrategicMerge`. The `StrategicMerge`
  combines the rule lists from both policies — the gateway BTP's shared-budget rule + the per-model
  BTP's per-model timeout, in one effective policy.

The `core-gateway` chart's `backendTrafficPolicy.monthlyBudget.plans` (not shown in this repo
section, lives in `ai-helm-values`) holds the live USD caps (e.g. free=$50, pro=$200, enterprise=$1000).

---

## 11. The Billing Period Marker and Counter Rotation

**`unit: Year` — deliberate, not a mistake**

Every monthly-budget rule in the BTP uses `unit: Year`:

```yaml
limit:
  requests: <limit-in-micro-usd>
  unit: Year    # ← NOT Month
```

This is intentional and critical. The Lyft ratelimit service appends its own epoch to every Redis
key: `floor(now / unit_seconds) * unit_seconds`. For `unit: Month` (2,592,000 s = 30 days, not a
calendar month), this epoch rotates every 30 days starting from Unix epoch zero — which can fall
mid-month (e.g. 2026-08-05). A rotation on 2026-08-05 would grant every user a fresh budget
mid-month, silently giving them double their allocation.

`unit: Year` (31,536,000 s) is the longest unit the CRD schema permits. Its epoch rotates annually
(around 2026-12-18, then ~2027-12-18, etc.) — a known, documented residual rotation. The actual
monthly rotation is entirely controlled by `x-billing-period` changing from `"2026-09"` to
`"2026-10"` on the 1st of each month. `unit` is just a TTL: it controls when Redis garbage-collects
the counter, not when it resets.

---

## 12. The `tiers` and `projectEnvelope` Fields — Current State

**Source**: [`charts/ai-models/values.yaml`](file:///Users/gisstudent/ai-helm/charts/ai-models/values.yaml) lines 159-171

```yaml
tiers: []
projectEnvelope: {}
```

**`tiers: []`** — an empty list. No tier rules render in any BTP. The field shape and the template
logic that would render tier rules exist in `charts/ai-model/templates/backendtrafficpolicy.yaml`
(lines 268-366), but with an empty list the `{{- range $i, $tier := $tiers }}` loop produces nothing.

**`projectEnvelope: {}`** — an empty map. No project envelope rule renders. The `{{- with $projectEnvelope.monthlyBudgetUsd }}` guard produces nothing.

Both fields follow the same `id` + ordered-list contract as `plans`. They are defined in
`ai-models/values.yaml` and propagated to every `ai-model` child via the ApplicationSet's `valuesObject`.

---

## 13. The Append-Only List Contract — Why and How It Works

This is the single most operationally important constraint in the rate-limit system.

### The Lyft ratelimit Redis key format

The Lyft ratelimit service (used by the Envoy AI Gateway) builds a Redis key from the **position**
of a rule in the BTP's `rateLimit.global.rules` list, not from any named identifier. The key
format is:

```
<domain>_rule/0_x-account-id_<account-value>_x-billing-plan_<plan-value>_...
<domain>_rule/1_x-account-id_<account-value>_x-billing-plan_<plan-value>_...
```

`rule/0`, `rule/1`, etc. are the zero-based indices of rules as they appear in the rendered YAML.

### What happens when the list is reordered

If you insert a new plan at index 0 (e.g. alphabetically before `enterprise`), then:

- The old `enterprise` rule (previously `rule/0`) becomes `rule/1` in the new render.
- The new `enterprise` rule key `_rule/1_...enterprise_...` does not exist in Redis yet — it starts at zero.
- Every `enterprise` user's accumulated spend counter is effectively orphaned at its old key (`_rule/0_...`),
  and a fresh counter starts — they get their full budget again, mid-month.

This happened live on 2026-07-16 on `charts/core-gateway` when `enterprise` was added mid-alphabetically.

### How the append-only contract is enforced

1. **The list shape**: `plans`, `tiers`, and `projectEnvelope` are YAML lists (`- id: ...`), not maps.
   Helm sorts map keys alphabetically on every render — a map would reorder on any insertion.

2. **The id guard**: Every `range` over `plans`/`tiers` in the BTP template calls:
   ```
   {{- if not $plan.id }}{{ fail "ai-model plans[N]: `id` is required..." }}{{ end }}
   ```
   This prevents silent rendering of a malformed entry.

3. **The ordering comment**: Every entry has an `# index N` comment above it in `values.yaml`,
   and the file comment explicitly says `APPEND ONLY. NEVER reorder, insert, or remove`.

4. **Tiers rendered after plans**: The `tiers` loop runs strictly after the `plans` loop. This means
   inserting a new tier never shifts a plan's index, and inserting a new plan never shifts a tier's index.

### The rule for commenting vs. deleting

When a burst rule is removed, the `burst:` block is **commented out**, not deleted. The list entry
remains with just `id:` to hold its index. Deleting the entry would renumber all subsequent entries.
Under `sharedBudget.enabled` this was safe to do for burst rules because the per-model BTP emits
no rules at all (the whole `rateLimit:` block is omitted), but the comment-only approach was chosen
for clarity and safety.

---

## 14. Redis Key Structure — What Counters Actually Look Like

When the monthly-budget rule (family 3) was active (before `sharedBudget.enabled`), a counter for a
`free` plan user looked like this in Redis:

```
<domain>_rule/1_x-account-id_<user-sub>_x-billing-plan_free_x-billing-period_2026-09_<epoch>
```

Where:
- `rule/1` = the `free` plan is at index 1 in the `plans` list
- `x-account-id_<user-sub>` = the `Distinct` header makes each value its own bucket
- `x-billing-plan_free` = the `Exact` header matches only `free`
- `x-billing-period_2026-09` = the `Distinct` Lua-stamped calendar month marker
- `<epoch>` = Lyft's own `floor(now/unit_seconds)*unit_seconds` appended automatically

Today (with `sharedBudget.enabled`), the active counter is on the **gateway-wide BTP** in
`core-gateway` with a `shared: true` flag. The key structure is similar but the domain prefix
changes (gateway BTP domain vs. per-model BTP domain) and there is one counter per user/plan
regardless of which model was called.

---

## 15. ArgoCD Sync Behaviour for These Charts

### `charts/lightbridge/` sync behaviour

All three child Applications target `home-remote/converse`. Sync waves:

- Wave `0`: `lightbridge-secrets` — ExternalSecrets must exist before the DB starts archiving or
  the app reads its client secret.
- Wave `1`: `lightbridge-db` — CloudNativePG Cluster + ObjectStore + ScheduledBackup. The DB must
  be ready before the app connects to it.
- Wave `2`: `lightbridge-app` — the Lightbridge service itself.

The `lightbridge-app` Application is the only one with a pinned chart version (`0.8.1`). It has
a second source (kustomize overlay for ingress TLS Certificates from `ai-helm-values`).

### `charts/ai-models/` sync behaviour

The `ai-models` orchestrator generates child Applications via ApplicationSet. Each model gets its
own Application. All run with `automated.prune: true` and `automated.selfHeal: true`. There is no
sync-wave ordering between model Applications — they all converge in parallel.

The critical point: when a `plans` entry is modified (or the `sharedBudget.enabled` flag changes),
ArgoCD will reconcile every per-model Application in the next sync. Depending on whether rules are
added or removed, this may create or delete BTP rules. With the current state (no rules rendering
in any per-model BTP), changes to `plans` values are render-neutral — they produce no diff because
the `rateLimit:` block is omitted entirely.
