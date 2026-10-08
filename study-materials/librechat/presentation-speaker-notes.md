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

### What this diagram shows
How LibreChat's main settings file, `librechat.yaml`, travels from Git into the running pod — and why changing it
needs one extra step.

### Set the scene (before pointing at any box)

> "LibreChat has two kinds of settings. Secrets and connection details — passwords, database addresses, the Keycloak
> client — are environment variables; we saw those on the previous page. Everything about *behaviour* — which models
> users see, the personas in the picker, the MCP tools, web search, summarization — lives in one file called
> `librechat.yaml`. LibreChat reads it from `/app/librechat.yaml`.
>
> That file is not baked into the Docker image. We use the upstream image unchanged, and inject our file at deploy
> time. Ours is about 950 lines. Let's follow it from Git to the pod."

### Walk the four numbered boxes, top to bottom

**Box 1 — ai-helm-values.**
> "It starts in our private repository, `ai-helm-values`, in `environments/prod/values/librechat-app.yaml`, under a
> key called `config:`. This is the only place you edit it. The public `ai-helm` repo has the chart — *how* to
> deploy — but deliberately no copy of the config — *what* is deployed. That split is ADR-0087."

**Box 2 — ArgoCD.**
> "When ArgoCD syncs `librechat-app`, it pulls two things: the chart from our OCI registry, and this values file from
> `ai-helm-values`. Helm merges them, so `config:` becomes an ordinary chart value."

**Box 3 — ConfigMap `librechat-config`.**
> "The chart has a small template that takes `config:` and writes it, as YAML text, into a ConfigMap called
> `librechat-config`, under a single key named `librechat`. A ConfigMap is just Kubernetes' way of storing a file."

**Box 4 — LibreChat pod.**
> "The pod mounts that one key as the file `/app/librechat.yaml`, read-only. When LibreChat starts, it reads the file
> once, and that's the configuration it runs with."

### The trap — point at the "Rollout ConfigMap" box

> "Now the important part. Suppose I change a persona in `ai-helm-values` and merge. ArgoCD syncs, the ConfigMap is
> updated — and nothing changes for users. Why?
>
> Two reasons. First, LibreChat reads the file **only at startup**; a running pod never looks at it again. Second,
> the file is mounted with `subPath` — a single-file mount — and Kubernetes never refreshes `subPath` files anyway.
> So the new config sits in the ConfigMap, but the pods keep running the old one until they happen to restart.
>
> The fix is this second, tiny ConfigMap, `librechat-app-config-rollout`. It holds one number: `generation`.
> The chart computes a checksum of it and stamps it on the pod template. Change the number, the checksum changes,
> the pod template changes — and Kubernetes does a rolling restart: new pods come up with the new file, old ones go
> away, no downtime. It's at 14 today."

### Point at the green "To change the config" box

> "So the rule for anyone editing the config is three steps:
> one, edit `config:` in `ai-helm-values`;
> two, in the **same commit**, bump `generation` — today from 14 to 15;
> three, merge. ArgoCD syncs and the pods roll.
> Forget step two and your change is silently ignored — the ConfigMap looks right, so it's a confusing one to debug."

### Point at the red warning box

> "One more trap, the opposite way. ArgoCD is told to ignore a missing values file. If someone deleted or renamed
> `librechat-app.yaml`, the chart would render with no `config:` at all, produce no ConfigMap, and LibreChat would
> start with no models. ArgoCD would report success — a broken chat, not a failed deploy. That's why the rule is
> *values repo first*: the file must exist on `ai-helm-values` before any chart change that depends on it."

### Close with the note at the bottom

> "In one sentence: the config travels Git → ArgoCD → ConfigMap → pod, but the pod only notices when it restarts,
> and the `generation` number is how we make it restart on purpose."

### Likely questions

**Q: Why not make LibreChat reload the file automatically?**
> LibreChat has no hot-reload for `librechat.yaml`; even without `subPath`, a restart would be needed. The
> `generation` bump gives a controlled rolling restart instead.

**Q: Why not compute the checksum from the config itself, so the bump is automatic?**
> The config ConfigMap is rendered by our own template, outside the bjw-s library that adds the checksum annotation;
> the library can only hash ConfigMaps it manages. The small marker ConfigMap is the workaround — at the cost of
> remembering to bump it.

**Q: How do I check which config is live?**
> `kubectl -n converse get cm librechat-config -o jsonpath='{.data.librechat}'` shows what's in the ConfigMap;
> compare with the pods' start time (`kubectl -n converse get pods -l app.kubernetes.io/name=librechat-app`). If
> the ConfigMap changed after the pods started, the pods are running an older config.

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
> usage and their LibreChat usage land on the same per-user account at the gateway — page 10."

---

## Page 9 — Code Interpreter

### What this diagram shows
What happens when a persona runs code: where the code actually executes, how it is isolated, and why it lives in
its own namespace.

### Set the scene

> "Some personas — Converse, for example — can *run* code: Python for a calculation, a chart, a CSV. That code is
> written by an AI on behalf of a user, so we treat it as untrusted. It does not run inside LibreChat. It runs in a
> separate service, the Code Interpreter, which we host ourselves in the namespace `librechat-sandbox`. Before
> August, LibreChat used a paid external service for this; now the code and the data never leave our cluster."

### Walk the boxes, left to right

**LibreChat (run code) → "code + token" → API.**
> "When the AI decides to run code, LibreChat sends it to the Code Interpreter's API. There is no fixed password:
> for every single request LibreChat signs a short-lived token with a private key — that's the yellow box. The API
> checks the signature with the matching public key. A stolen token is useless after a few minutes."

**API → Worker.**
> "The API accepts the job and puts it in a queue — the Redis box on the right. The worker takes jobs off the
> queue one by one and prepares them: which packages, how much time, what the code is allowed to do."

**Worker → Sandbox runner (the red box).**
> "This is where the code actually runs. For every execution the sandbox runner builds a fresh, throwaway jail with
> a tool called NsJail: its own processes, its own files, a temporary user ID, and **no network**. The code gets
> about fifteen seconds, then the jail is destroyed. The next run — even from the same user — gets a brand-new jail.
> Up to four jobs run at the same time; the rest wait in the queue."

**The three helpers on the right.**
> "The **file server** moves files in and out — the CSV you uploaded, the chart the code produced — and stores them
> in S3. **Tool calls** lets running code call tools in a controlled way. The **egress gateway** is the only way
> anything can leave a jail, and only if the job was explicitly allowed. **Package setup** is a one-time job that
> pre-installs Python and other language packages, so jails don't download anything."

### The security point — the note at the bottom

> "Building those jails needs Linux privileges that normal pods don't get, so this namespace is allowed
> 'privileged' pods. That's exactly why it's not inside `converse`: the extra privileges stay here, far from
> LibreChat and MongoDB. And it's deployed as its own ArgoCD app, not under librechart, because librechart puts
> all its children in one namespace."

### Close

> "So: LibreChat signs a request, the API queues it, and the code runs once in a disposable, network-less jail —
> on our own cluster."

### Likely questions

**Q: Do users share a sandbox?**
> They share the sandbox-runner *pod*, but every execution gets its own jail that is destroyed afterwards — two
> users' code never runs in the same jail.

**Q: Why not a virtual machine per run? Isn't that safer?**
> Yes — the service supports tiny VMs, but they need KVM, which our Hetzner cloud nodes don't offer. NsJail shares
> the host kernel, which is the trade-off we accepted.

**Q: Can the code reach the internet or our internal services?**
> No — jails have networking disabled; the only exit is the egress gateway, and only for explicitly granted jobs.

---

## Page 10 — Per-User Identity

> "Before we trace a full request, one subtle idea you need for it. There are two identities on every request: the human, known to
> LibreChat through Keycloak, and LibreChat itself, known to the gateway through its key. The bridge is LibreChat's
> header templating — `{{LIBRECHAT_USER_OPENIDID}}` becomes the Keycloak subject."

> "Why is it safe to trust a header? Because only authenticated first-party services can reach the internal listener,
> and Authorino overwrites the rate-limit headers — a client can't choose its own plan."

> "And a defensive detail: in August a request arrived with the placeholder text literally unreplaced. Without a
> guard, every such request would have pooled into one fake account. The gateway now ignores any value containing
> `{{`."

> "Because the account key is the same Keycloak id opencode uses, a person's chat and coding usage share one account.
>
> Keep this picture in mind — on the next page we follow one real chat message end to end, and you'll see exactly
> where this happens."

---

## Page 11 — Full Request Flow

> "Now everything together. A user sends a message. Traefik terminates TLS, LibreChat checks its own session — no Keycloak round-trip per
> request. It builds an OpenAI-style request to the internal gateway with its one API key — and four extra headers
> carrying who the user is.
>
> The gateway runs Authorino — the identity step from the previous page. It accepts the key, then rewrites the rate-limit identity: the account becomes the
> user's Keycloak id, the plan becomes free or pro. The budget limiter is explicitly switched off for this internal
> plane. Then the per-model limit: by default 60 requests per minute **per user, per model**. Then the model
> backend, and the answer streams back."

> "So: one key to authenticate, but per-user limits and per-user cost dashboards. LibreChat's own balance system
> is turned off — the gateway is the single place that counts."

---

## Page 12 — GitHub MCP tools

### What this diagram shows
How a persona like Repo Concierge can read and act on GitHub for a user — first the one-time connect (green band),
then a normal chat that uses a GitHub tool (blue band).

### Band A — connect GitHub (once per user, per GitHub server)
> "We plug in GitHub's official MCP server — five slices of it: issues, projects, pull requests, repos and docs
> search. Each acts on GitHub as **the user**, so each user must connect their own GitHub account first.
>
> The user opens a GitHub persona; LibreChat sees there's no GitHub token yet and shows a 'connect' link (steps 1–2).
> The user logs in on github.com and approves the access (3). GitHub sends the browser back to LibreChat with a
> one-time code (4–5). LibreChat swaps that code, plus our GitHub app's client secret, for the user's GitHub token
> (6–7). It encrypts the token and stores it in MongoDB under that user (8). Then it connects to GitHub's MCP server
> with that token and asks which tools exist (9–10)."

### Band B — use it
> "Now the user asks 'list my open PRs' (11). LibreChat sends the message **and the list of GitHub tools** to the
> model through our AI gateway (12). The model doesn't call GitHub itself — it answers 'please run
> list_pull_requests' (13). LibreChat runs that tool against GitHub's MCP server with the user's token (14) and gets
> the result back (15). It hands the result to the model, which writes the answer (16), streamed to the user (17)."

### Points to stress (the two boxes at the bottom)
> "Three facts: the tools can only do what the user could do on github.com; each of the five servers is connected
> separately; and the GitHub traffic goes straight from LibreChat to GitHub — only the model calls go through our
> gateway.
>
> Two cautions: write tools are switched on — we don't use GitHub's read-only mode — so the personas are told to ask
> before commenting, labelling or merging. And the access requested is very broad, including deleting repos and
> organisation admin — a hardening item."

### Likely questions
**Q: Where do the GitHub client id and secret come from?** From Secret `librechat-mcp-github` (ESO), injected as
`GITHUB_MCP_CLIENT_ID` / `GITHUB_MCP_SECRET_ID` and referenced in `librechat.yaml`.

**Q: Can another user use my GitHub token?** No — tokens are stored per user and each user gets their own MCP connection.

**Q: Where is this configured?** `ai-helm-values` `librechat-app.yaml` → `config.mcpServers.github_*`
(URL `https://api.githubcopilot.com/mcp/x/<toolset>`, callback `https://ai.camer.digital/api/mcp/<server>/oauth/callback`).

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
