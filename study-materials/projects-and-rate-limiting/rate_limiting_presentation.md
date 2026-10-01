# Projects & Rate Limiting — Complete Presentation Guide

> Import `.puml` diagrams into Draw.io via:
> **Arrange → Insert → Advanced → PlantUML**

---

## Presentation Order

Follow this order when explaining to someone:

| Step | Diagram | Topic |
|------|---------|-------|
| 1 | `01-data-model.puml` | What a Project IS — the data model |
| 2 | `02-project-lifecycle.puml` | Creating a project, adding members, minting keys |
| 3 | `03-model-policy-enforcement.puml` | How model restrictions are enforced at the gateway |
| 4 | `04-identity-resolution.puml` | How an API key becomes headers on a request |
| 5 | `05-two-guardians.puml` | The two enforcement mechanisms (speed + money) |
| 6 | `06-rpm-per-key.puml` | The Speed Cop: how rpmPerKey + Redis works |
| 7 | `07-budget-limiter.puml` | The Bank: how the Lua budget limiter works |
| 8 | `08-append-only-trap.puml` | Why the plans list is append-only |
| 9 | `09-three-layers.puml` | Plans / Tiers / Project Envelope (AND logic) |
| 10 | `10-full-request-gauntlet.puml` | The complete life of one AI request |

---

## Part 1 — Projects

### What is the GOAL of Projects?

Without projects, every API key is its own isolated identity with its own plan. That does not reflect how real teams work. The goal of Projects is:

- **Group usage across a team**: 10 developers in one company all draw from the same project budget, not 10 separate budgets.
- **Isolate environments**: A company can have a "Production" project and a "Dev" project with different model allowlists and separate budgets.
- **Centralize model policy**: The company admin decides that their developers may only call `gpt-4o`. All API keys under that project automatically inherit that restriction.
- **Enable billing per project**: In the future, invoices map to projects, not to individual keys.

---

### What IS a Project? (Data Model)

A **Project** is a row in the `PROJECTS` table in the Lightbridge PostgreSQL database. It is the central grouping entity. Everything else hangs off it.

**PROJECTS table**
```
id               — unique ID (PK)
account_id       — which Account (= user) owns this project
name             — human-readable label
billing_identity — "who pays" (UNIQUE, links to budget ledger)
billing_plan     — enterprise | free | pro | service | internal
model_policy     — allow_all | allowlist | deny_all
allowed_models   — jsonb list (only used when model_policy = allowlist)
project_quota    — pooled spending ceiling (NOT YET enforced)
default_limits   — jsonb, default {}
is_default       — bool (each Account has one default project)
status           — active | suspended
```

**PROJECT_MEMBERS table**
```
project_id  — FK→projects.id  (PK)
account_id  — FK→accounts.id  (PK)
role        — lead | member
quota_tier  — per-member ceiling (NOT YET enforced)
```

**API_KEYS table**
```
id               — PK
project_id       — FK→projects.id
owner_account_id — FK→accounts.id
key_hash         — SHA-256 of the secret (secret never stored)
billing_plan     — copied from the project at mint time
status           — active | revoked
expires_at       — optional TTL
```

Key relationships:
- One Account **owns** many Projects
- One Project has a **roster** of member Accounts (PROJECT_MEMBERS)
- One Project **issues** many API Keys
- Budget grants can be **scoped** to a Project

---

### Which Binary Manages Projects?

The **`lightbridge-authz` service** (a Rust HTTP API) is the sole owner of project data. It is deployed via the `lightbridge-app` Helm chart in the `converse` namespace.

It is the only thing that reads and writes the PROJECTS, PROJECT_MEMBERS, and API_KEYS tables.

Envoy **never** touches this database. Envoy only reads headers that Authorino stamps at request time based on what Lightbridge returns.

The service is also exposed via an **MCP server** (the `mcp` component), which means developers can manage projects directly from LibreChat or opencode.

---

### What Can We Do With Projects Today?

| Action | Available? |
|--------|-----------|
| Create / delete a project | ✅ Yes |
| Add / remove a member | ✅ Yes |
| Set model policy (allow_all, allowlist, deny_all) | ✅ Yes |
| Set allowed model list | ✅ Yes |
| Mint / revoke an API key | ✅ Yes |
| Set billing plan on a project | ✅ Yes |
| Per-project spend ceiling (project_quota) | ⚠️ In DB schema — NOT YET enforced |
| Per-member spend ceiling (quota_tier) | ⚠️ In DB schema — NOT YET enforced |
| Per-project rate limits (tiers, envelope) | ⚠️ Helm template exists — tiers: [] today |

---

### What Functionalities Are Missing (But Would Be Cool)?

1. **Per-project spend ceiling (`project_quota`)**: The field exists in the DB (`project_quota — pooled ceiling, not yet enforced`). Would cap total spend for a whole project across all its API keys.

2. **Per-member spend ceiling (`quota_tier`)**: Exists on PROJECT_MEMBERS (`quota_tier — per-member ceiling, not yet enforced`). Would let a project lead give different members different spending allowances. The `tiers:` Helm list is already structured to render these rules.

3. **Project-level rate limits (`projectEnvelope`)**: The Helm template renders a project-envelope rule when `projectEnvelope.monthlyBudgetUsd` is set. Today `projectEnvelope: {}` so nothing renders.

4. **Project suspension enforcement**: The `status: suspended` field exists on PROJECTS, but there is no gateway rule that reads it and blocks traffic for suspended projects.

5. **Role-based model access within a project**: Today `allowed_models` is a flat list for the whole project. There is no way to say "members can use `gpt-4o` but only leads can use `claude-opus-4`".

---

## Part 2 — Rate Limiting

### The Two Guardians

Two active mechanisms in production today:

| Mechanism | What it limits | Response |
|-----------|---------------|---------|
| `rpmPerKey` (Lyft ratelimit + Redis) | Speed: requests per minute per model | 429 Too Many Requests |
| Dynamic Budget Limiter (Lua + Lightbridge) | Money: micro-USD balance | 402 Payment Required |

These are completely independent. One can fire without the other.

---

### How rpmPerKey Works (The Speed Cop)

On every request Envoy asks the Lyft ratelimit service whether the caller is over their limit. Lyft builds a Redis key from the **position (index)** of the rule in the BackendTrafficPolicy list, the caller's API key ID, and the model name. Redis holds a counter that increments by 1 per request and expires after 60 seconds.

Two rules per model:
- `rpm-rule/0` keyed on `x-api-key-id` + model
- `rpm-rule/1` keyed on `x-account-id` + model

If the counter hits the limit → 429.

---

### How the Budget Limiter Works (The Bank)

Authorino reads the caller's live balance from the Lightbridge ledger and stamps it as Envoy internal dynamic metadata. The Lua script reads that one number:

- `remaining > 0` → allow
- `remaining ≤ 0` → 402 budget_exhausted

Shadow mode (`shadowMode: true`): script runs, decides, logs, but never blocks. Used before going live to observe false positives safely.

---

### The Append-Only List Contract

Lyft ratelimit keys Redis counters by the **position** (index) of a rule in the list, not by its name. Inserting a new plan mid-list shifts every subsequent plan's index, orphaning their live Redis counters and giving users a free budget reset.

This happened live on 2026-07-16.

Rules:
- Never insert mid-list
- Never delete an entry (comment body, keep the `id:` stub)
- Only append at the end

Why not use a YAML map? Helm sorts map keys alphabetically on every render, producing the same shift problem invisibly.

---

### The Three Layers (AND Logic)

| Family | Bucket | Active today? |
|--------|--------|---------------|
| User Plan (Family 3) | Monthly USD cap per billing plan | ❌ Replaced by Lua Budget Limiter |
| Member Tier (Family 4) | Per-member cap within a project | ❌ tiers: [] empty |
| Project Envelope (Family 5) | Total cap for the whole project | ❌ projectEnvelope: {} empty |
| rpmPerKey (Family 6) | Requests per minute per model | ✅ ONLY ACTIVE RULE TODAY |

---

## Diagrams Reference

All `.puml` files are in:
`study-materials/projects-and-rate-limiting/diagrams/`

Import each one into Draw.io via **Arrange → Insert → Advanced → PlantUML**.
