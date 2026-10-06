# LibreChat — Presenter's Speaker Notes

> **File to open**: `diagrams/librechat.drawio` (12 pages).
> This document gives you the words to say for each page, in order. Facts verified 2026-10-05 —
> see [`librechat-deep-dive.md`](librechat-deep-dive.md) for sources and hands-on labs.
> **Tip**: run labs I1 and I6 from the deep dive right before presenting so you can quote live numbers.

---

## Before You Begin — Setting the Stage

> "We are going to look at LibreChat — the AI chat interface our users know as Converse, at `ai.camer.digital`.
> LibreChat is an open-source project, MIT licensed, about 45 thousand GitHub stars. We run version 0.8.7;
> upstream released 0.8.8 four days ago.
>
> But I don't want to show you what it looks like — I want to show you how it's wired: it logs users in through
> Keycloak, sends every model call through our Envoy AI Gateway, stores everything in MongoDB, searches with
> Meilisearch, shares Redis with the platform, puts files in Hetzner object storage, and runs user code in a
> sandbox we host ourselves. All of it deployed by ArgoCD from two git repositories.
>
> Let's start from the top."

Open `librechat.drawio`, **Page 1**.

---

## Page 1 — System Overview

**User and LibreChat.**
> "A user opens `ai.camer.digital`. That's two LibreChat pods behind Traefik, autoscaling between one and four.
> The image is the upstream one — we don't fork LibreChat; everything is configuration."

**Keycloak.**
> "There is no LibreChat password. Email login and registration are off. The browser is sent straight to
> Keycloak, and the first time someone logs in, LibreChat creates their account automatically."

**The gateway — the critical point.**
> "LibreChat never talks to OpenAI, DeepInfra or Google directly. It has exactly one model endpoint, called
> `converse`, pointing at the **internal** plane of our Envoy AI Gateway. It doesn't even hold vendor keys — the
> gateway does. And it doesn't hardcode the model list: at startup it asks the gateway's `/v1/models`."

**Data stores.**
> "MongoDB holds all state — users, conversations, agents. Meilisearch indexes conversations for search. Redis is
> the platform's shared Redis; we need it because we run more than one replica. Files go to Hetzner object storage."

**The extras.**
> "Around that: MCP tool servers — GitHub, Coder, our Lightbridge self-service, docs and design references; a web
> search pipeline; and a self-hosted Code Interpreter in its own namespace. Image generation was switched off in
> September when we lost a GPU node — I'll come back to that."

**Transition:** "So how is this deployed?"

---

## Page 2 — Chart Family

**Orchestrator.**
> "The chart is `librechart` — no 'e', on purpose. It renders a single object: an ArgoCD ApplicationSet. That
> ApplicationSet lives on the ArgoCD cluster, not where the pods run — control objects and workloads are on
> different clusters here (ADR-0017)."

**Two children.**
> "It generates two Applications. Meilisearch first, in wave minus one, so search is up before LibreChat starts.
> Then LibreChat and MongoDB together in wave zero — they share a lifecycle, so they share an Application: a
> LibreChat without its database isn't healthy, it's broken."

**Two sources.**
> "Each child pulls the chart from our OCI registry on a floating version — merge to main is a deploy — and pulls
> its production values from a private repository, `ai-helm-values`. Public repo: how to render. Private repo:
> what is deployed."

**The neighbours — say this explicitly, people get it wrong.**
> "Three things look like LibreChat but are not children of this chart. The Code Interpreter is its own app in its
> own namespace. The opencode well-known used to be the third child; in August it moved to the `ai-models`
> orchestrator because its content is derived from the model catalog. And the MongoDB backup is a separate
> CronJob app."

**Sync policy.**
> "Prune and self-heal are on: manual changes in the cluster are reverted. Git is the only way to change this."

---

## Page 3 — Authentication Flow

> "First visit: `OPENID_AUTO_REDIRECT` sends the browser to Keycloak, realm `camer-digital`, client `converse`.
> After login Keycloak redirects back to `/oauth/openid/callback` with a code; LibreChat exchanges it for tokens.
>
> Two settings matter. `ALLOW_SOCIAL_REGISTRATION=true` means the account is created on first login — that's how
> people get in without an admin. And `OPENID_REUSE_TOKENS=true` means LibreChat keeps Keycloak's refresh token as
> the session and refreshes against Keycloak — which works across both replicas."

**The honest bit about roles.**
> "You'll see variables pointing at a `librechat_roles` claim. They do nothing today: the variable that switches the
> check on, `OPENID_REQUIRED_ROLE`, is commented out with the note 'open beta — everyone can connect'. So any user
> in the realm can use Converse. Turning the gate on is a one-line change plus a Keycloak role assignment."

> "Logout uses Keycloak's end-session endpoint, so logging out of Converse logs you out of SSO."

---

## Page 4 — Config Delivery

> "LibreChat's behaviour — endpoints, personas, MCP servers, web search, summarization — lives in `librechat.yaml`.
> Ours is about 950 lines in `ai-helm-values`. Five steps get it into the pod."

Walk the five boxes, then:

> "Here's the trap. The file is mounted with `subPath`, and LibreChat only reads it at startup anyway. So changing
> the config does nothing to running pods. The fix is a tiny marker ConfigMap with a `generation` number; its
> checksum is stamped on the pod template. Bump the number in the same commit, and the pods roll. It's at 14 today
> — fourteen config rollouts."

> "And a second trap: if the values file were missing, the chart would render no config at all and LibreChat would
> boot with no models. That's why the rule is 'values repo first'."

---

## Page 5 — Data Stores

**MongoDB.**
> "Mongo 8.2, one pod, a 30-gig volume, and no authentication. The connection string uses the stable per-pod DNS name
> from the headless service — that's the replica-set style address; with one member it's one host. A PDB keeps it
> from being evicted, which also means a node drain blocks until someone handles it. Backups: a CronJob dumps it to
> object storage every night at two."

**Redis.**
> "Redis is shared, owned by the home-os repo. `rediss` with two s's — TLS only, plus a password, plus our internal
> CA. And we target HAProxy, not the plain Redis service: the plain service load-balances across master and replica,
> so half the writes would fail with READONLY. LibreChat can't speak Sentinel, so HAProxy follows the master for us."

**Meilisearch and S3.**
> "Meilisearch: plain HTTP inside the namespace, master-key protected. S3: Hetzner, path-style. One real-world fix:
> LibreChat's presigned URLs expire after two minutes by default — images broke shortly after upload — so we raised
> it to seven days."

---

## Page 6 — Secrets Pipeline

> "Nothing secret is in git. Thirteen ExternalSecrets pull from AWS Secrets Manager through the `ssegning-aws`
> store every hour."

Point at a few:
> "`librechat-config` has the JWT secrets and `CREDS_KEY`/`CREDS_IV` — the AES key that encrypts users' stored
> OAuth tokens. Rotate that carelessly and every stored MCP login becomes unreadable.
> `librechat-main-config` is LibreChat's key for the gateway. `librechat-codeapi-jwt` is the private key LibreChat
> signs Code Interpreter requests with."

> "One caveat: ESO refreshes the Secret every hour, but environment variables are read when the container starts.
> A rotated value only takes effect after a restart."

> "And one secret is not from ESO: `librechat-internal-ca` comes from cert-manager. We only want its `ca.crt` — our
> internal root CA — so Node trusts the gateway's internal TLS and Redis's TLS."

---

## Page 7 — Agent Seed Job

> "Agents in LibreChat are database documents with an author — a real user. There's no service account and no
> YAML schema for them. So how do you manage a fleet of agents in git?"

> "After each sync, ArgoCD runs a PostSync Job on the LibreChat image. It finds the platform user in Mongo — that
> user must have logged in once, or the job fails on purpose — and mints a 30-minute LibreChat token with the JWT
> secret. Then it calls LibreChat's own API: list agents, patch the ones that exist by name, create the rest. Leaves
> first, then the orchestrators, because an orchestrator needs its sub-agents' ids. Then it makes every agent
> publicly visible, and finally it **deletes** platform-owned agents that are no longer in the list."

> "That last step makes it declarative. When image generation was withdrawn, the image agent was pruned
> automatically. The flip side: renaming an agent creates a new id, and anything pointing at the old id breaks."

> "Today's fleet: a Deep Reviewer that delegates to a Security Reviewer and a Test Coverage Reviewer — exposed as a
> persona in the picker — and a `coder` agent that drives Coder workspaces."

---

## Page 8 — opencode Well-Known (a neighbour)

> "Same host, different application. `ai.camer.digital/opencode/.well-known/opencode` is a static JSON that
> bootstraps the opencode CLI. It now belongs to the models orchestrator."

> "`opencode auth login https://ai.camer.digital/opencode` downloads it. Since ADR-0135, login goes through our own
> identity provider, authz-idp at `auth.ai.camer.digital`, with the `opencode-lightbridge` plugin and the device-code
> flow — no local callback port, so it works over SSH and in containers. Developers then call the **external**
> gateway, `api.ai.camer.digital`."

> "Why show it here? Because it's on LibreChat's host and in its namespace, and because a developer's opencode
> usage and their LibreChat usage land on the same per-user account at the gateway — page 11."

---

## Page 9 — Networking

> "Three ingresses on Traefik. The main one: `ai.camer.digital` to LibreChat on 3080, certificate from
> cert-manager with an HTTP-01 challenge. The well-known path on the same host goes to its nginx. And `ai.kivoyo.com`,
> a vanity domain, is redirected by a Traefik middleware — a temporary redirect: 302 for normal page loads, 307 for
> other methods — so the path is kept and nothing gets cached permanently."

> "Network policy: LibreChat's policy allows everything, and the `converse` namespace has no default-deny baseline.
> That's a known gap — Mongo has no password, so the network is its only protection."

---

## Page 10 — Full Request Flow

> "A user sends a message. Traefik terminates TLS, LibreChat checks its own session — no Keycloak round-trip per
> request. It builds an OpenAI-style request to the internal gateway with its one API key — and four extra headers
> carrying who the user is.
>
> The gateway runs Authorino. It accepts the key, then rewrites the rate-limit identity: the account becomes the
> user's Keycloak id, the plan becomes free or pro. The budget limiter is explicitly switched off for this internal
> plane. Then the per-model limit: by default 60 requests per minute **per user, per model**. Then the model
> backend, and the answer streams back."

> "So: one key to authenticate, but per-user limits and per-user cost dashboards. LibreChat's own balance system
> is turned off — the gateway is the single place that counts."

---

## Page 11 — Per-User Identity

> "This is the subtle part, so one picture just for it. Two identities on every request: the human, known to
> LibreChat through Keycloak, and LibreChat itself, known to the gateway through its key. The bridge is LibreChat's
> header templating — `{{LIBRECHAT_USER_OPENIDID}}` becomes the Keycloak subject."

> "Why is it safe to trust a header? Because only authenticated first-party services can reach the internal listener,
> and Authorino overwrites the rate-limit headers — a client can't choose its own plan."

> "And a defensive detail: in August a request arrived with the placeholder text literally unreplaced. Without a
> guard, every such request would have pooled into one fake account. The gateway now ignores any value containing
> `{{`."

> "Because the account key is the same Keycloak id opencode uses, a person's chat and coding usage share one account."

---

## Page 12 — Self-Hosted Code Interpreter

> "When a persona runs code, LibreChat calls our own Code Interpreter — the open-source clickhouse service, built
> from our fork because upstream publishes no images."

> "Auth: no static key. LibreChat signs a short-lived Ed25519 token per request; the service verifies it with the
> public half."

> "Isolation: user code runs in NsJail, which needs SYS_ADMIN-level capabilities because our nodes don't offer KVM
> for the safer microVM mode. That requires privileged Pod Security — so it lives in its own namespace,
> `librechat-sandbox`, and the elevation never touches LibreChat or Mongo. Sandboxes have no network; the only
> outbound hole is the file server to object storage."

> "Limits: one sandbox runner, because the packages volume is ReadWriteOnce."

---

## Anticipated Questions and Answers

**Q: Do all LibreChat users share one quota?**
> No. They share one *authentication* key, but the gateway rate-limits per user via `X-LibreChat-User`
> (`x-account-id` = Keycloak sub). LibreChat's own balance feature is disabled.

**Q: Can anyone in the company log in?**
> Anyone in the `camer-digital` realm — the role gate (`OPENID_REQUIRED_ROLE`) is off during the open beta.

**Q: What happens if MongoDB goes down?**
> LibreChat stops working — all state is there. Single member, no failover; recovery is restart or restore from the
> nightly 02:00 dump. The PDB prevents voluntary eviction.

**Q: Why does MongoDB have no password?**
> Historical simplicity; its protection is network reachability — and today that's weak (allow-all policy, no
> namespace baseline). It's a real hardening item: either Mongo auth or a NetworkPolicy allowing only LibreChat, the
> seed job and the backup job.

**Q: How do I add a model?**
> Add it to the gateway catalog (`ai-helm-values` `models.yaml`). LibreChat picks it up from `/v1/models` on the next
> restart. Add a `tokenConfig` entry if you want context/cost shown.

**Q: How do I add a persona?**
> Add a `modelSpecs.list` entry in `ai-helm-values` `librechat-app.yaml` and bump `generation` in the same commit.

**Q: Why no image generation?**
> ADR-0138: losing a GPU node left one card, kept for the text model everyone uses. The env vars were deleted and
> `filteredTools` blocks all image tools, so nobody can re-enable it by accident (a `GOOGLE_KEY` would otherwise
> silently turn on Gemini image generation).

**Q: If the seed job runs on every sync, does it overwrite user agents?**
> No — it only touches agents authored by the platform user. It does overwrite and prune *platform* agents; that's
> intended.

**Q: Are we on the latest version?**
> No — v0.8.7; v0.8.8 came out on 2026-10-01. Main upgrade check: v0.8.8 refuses to start with invalid `CREDS_KEY`/`CREDS_IV`.

**Q: Is anything broken right now?**
> Skill sync has been failing hourly because `skills/governance/SKILL.md` has invalid YAML frontmatter (an unquoted
> description containing `: `). Chat is unaffected; that skill just isn't refreshed.

**Q: Why `NO_INDEX=true`?**
> Sends `X-Robots-Tag: noindex` so search engines don't index the app.

---

*End of presenter's notes.*
