# LibreChat — Presenter's Speaker Notes

> **File to open**: `diagrams/librechat.drawio`
> Use the page tabs at the bottom to navigate between diagrams.
> This document gives you the exact words to say for each page, in order.

---

## Before You Begin — Setting the Stage

Say this before opening the file:

> "We are going to look at LibreChat — the AI chat interface that users visit at
> `ai.camer.digital`. But rather than just explaining what it looks like, I want to
> show you how it is built, deployed, and connected to the rest of the platform.
>
> LibreChat is not just a standalone web app. It is a carefully integrated piece of
> the platform that talks to Keycloak for authentication, Envoy for AI model access,
> MongoDB for storage, Redis for sessions, Meilisearch for search, and Hetzner S3
> for file uploads. And all of it is deployed through ArgoCD using a family of
> Helm charts that deploy in a specific order.
>
> Let's start from the very top."

Open `librechat.drawio` and go to **Page 1**.

---

## Page 1 — System Overview

### What this diagram shows
LibreChat's place in the platform and every system it connects to.

### Walk through it like this

**Start with the user and LibreChat.**
> "A user opens `ai.camer.digital` in their browser. That hits LibreChat — an open-source
> AI chat application, version 0.8.7, running in 2 pods with autoscaling up to 4.
> The image tag is `v0.8.7` and pull policy is `Always` — every restart pulls fresh."

**Point to Keycloak.**
> "LibreChat does NOT have its own login system. Password login is disabled.
> When a user visits the site, they are immediately redirected to Keycloak for authentication.
> Keycloak is the identity provider. After login, the user comes back to LibreChat
> with an OIDC session. That session is cached in Redis."

**Point to Envoy.**
> "Here is the critical architectural point: LibreChat NEVER calls AI providers directly.
> Every AI request — every `POST /v1/chat/completions` — goes through the Envoy AI Gateway
> using a special key called `CONVERSE_OPENAI_API_KEY`. This is a Lightbridge API key.
> Envoy applies all the same rate limits and budget checks that apply to any other API key user.
> LibreChat is just another caller from Envoy's perspective."

**Walk through data stores.**
> "Four data stores, each with a specific role:
> - MongoDB: LibreChat's primary database — conversations, agents, files metadata. Standalone, no auth.
> - Redis: shared across the platform, TLS-only, used for sessions and real-time features.
> - Meilisearch: full-text search over conversations. Plain HTTP, port 7700.
> - Hetzner S3: image and file uploads. Uses the shared `ssegning-k8s-state` bucket under
>   the `images/` and `files/` prefixes."

**Transition:**
> "Now let's see how this whole system is structured and deployed."

---

## Page 2 — Chart Family

### What this diagram shows
The Helm chart structure: one orchestrator, three leaf apps, and the sync wave ordering.

### Walk through it like this

**Start with the orchestrator.**
> "The deployment is managed by a Helm chart called `librechart` — note the missing 'e',
> that is intentional. This chart lives on the ArgoCD control-plane cluster, not on
> the workload cluster. It renders exactly one Kubernetes resource: an ArgoCD ApplicationSet.
> The ApplicationSet then spawns three child Applications, one for each leaf chart."

**Explain sync waves.**
> "The three children deploy in a specific order using ArgoCD sync waves. Wave minus-one goes first,
> then zero, then one. The order matters:
> 1. Meilisearch must be running before LibreChat starts, because LibreChat tries to connect to
>    it on startup.
> 2. LibreChat and MongoDB must be healthy before the well-known nginx serves its document
>    — because opencode uses LibreChat's API after auth.
> 3. The Agent Seed Job runs AFTER LibreChat is healthy (PostSync). We'll see that in diagram 7."

**Explain the $values pattern.**
> "Each child has two Helm sources. Source 1 is the OCI chart from our GitHub Container Registry.
> Source 2 is the `ai-helm-values` private repository — which provides the production config values.
> This pattern, ADR-0087, separates public chart logic from private environment configuration.
> `ai-helm` is public. `ai-helm-values` is private and holds secrets, model configs, and agent definitions."

**Point to sync settings.**
> "`prune=true` and `selfHeal=true` — if someone manually changes something in the cluster,
> ArgoCD reverts it. If a resource exists in the cluster but not in the chart, it is deleted.
> This makes ArgoCD the single source of truth."

**Transition:**
> "Now let's see how authentication actually works."

---

## Page 3 — Authentication Flow

### What this diagram shows
The OIDC login flow from user visit to session established.

### Walk through it like this

**Set the scene.**
> "A user visits `ai.camer.digital` for the first time. They are not authenticated.
> LibreChat has `OPENID_AUTO_REDIRECT=true`, so instead of showing a login page,
> it immediately redirects the browser to Keycloak."

**Walk through the flow.**
> "The user logs in on Keycloak — using SSO, a federated identity, or however their
> organisation has configured Keycloak. Keycloak redirects back to LibreChat at
> `/oauth/openid/callback` with an authorization code.
>
> LibreChat exchanges that code for an ID token and a refresh token. The refresh token
> is valid for 60 days — `REFRESH_TOKEN_EXPIRY=5184000000` milliseconds. Users rarely
> need to log in again."

**Explain the role check.**
> "LibreChat also checks that the user has the required Keycloak role. The role is read
> from the `librechat_roles` claim in the access token. Users without this role are blocked.
> This is how the platform restricts access — Keycloak controls who has the role."

**Key restrictions.**
> "Just to be explicit: `ALLOW_EMAIL_LOGIN=false` and `ALLOW_REGISTRATION=false`.
> There is no way to create a LibreChat account by email. There is no self-registration.
> The only way in is through Keycloak OIDC. This is deliberate — it keeps identity
> management in one place."

**Transition:**
> "Once you're logged in, LibreChat loads its configuration from a file. Let's see how that
> file gets into the pod."

---

## Page 4 — Config Delivery

### What this diagram shows
The 5-step chain that gets `librechat.yaml` into the running pod.

### Walk through it like this

> "LibreChat reads its model endpoints, agent configs, and feature flags from a file at
> `/app/librechat.yaml`. This file does not come from the Docker image — it is injected
> at runtime. Here is how."

**Walk through each step.**
> "Step 1: The config lives in the private `ai-helm-values` repository.
> Step 2: When ArgoCD syncs, it merges that file as a second Helm values source.
> Step 3: Helm renders it into a ConfigMap called `librechat-config` with a single key `librechat`.
> Step 4: The ConfigMap is mounted into the pod using `subPath: librechat`. This means
>         only that single key is mounted as the file `/app/librechat.yaml`, not the entire
>         ConfigMap directory.
> Step 5: Rollout. This is the tricky part."

**Explain the rollout trigger.**
> "Here is the problem with `subPath` mounts: they do NOT hot-reload when the ConfigMap changes.
> If you change the config in `ai-helm-values`, ArgoCD syncs, Helm re-renders the ConfigMap —
> but the running pod never sees the new file. It keeps using the old one.
>
> The solution is a second ConfigMap called `librechat-app-config-rollout` with a `generation`
> field. When you change the config, you ALSO bump this field in the same commit.
> The bjw-s chart stamps the SHA256 of this ConfigMap as an annotation on the Deployment's
> pod template. When the annotation changes, Kubernetes triggers a rolling restart.
> Without bumping `generation`, your config change is silently ignored until the next restart."

**Transition:**
> "Now let's look at the data stores in detail."

---

## Page 5 — Data Stores

### What this diagram shows
MongoDB, Redis, Meilisearch, and S3 — how each is configured and why.

### Walk through it like this

**MongoDB.**
> "MongoDB is the only data store that LibreChat owns. It runs as a single-pod StatefulSet
> in the `converse` namespace. No authentication — `auth.enabled: false`. 30 gigabytes of
> persistent storage.
>
> The connection string connects directly to the pod via the headless service DNS entry:
> `mongodb://librechat-app-db-0.librechat-app-db-headless:27017`. It bypasses the ClusterIP
> service deliberately — with only one member in the replica set, the headless DNS goes
> straight to the pod."

**Redis.**
> "Redis is different — LibreChat does NOT own it. It uses the platform's shared Redis HA
> cluster deployed by the `home-os` infrastructure.
>
> The URI is `rediss://` — double-s means TLS. This is not a typo. The Redis HA cluster
> only accepts TLS connections. A plaintext `redis://` connection gets reset immediately.
>
> The target is `redis-ha-haproxy`, NOT `redis-ha-redis`. This is critical. `redis-ha-redis`
> is the round-robin service — it routes to both the master and the replicas. If LibreChat
> sends a write to a replica, Redis returns `READONLY You can't write against a read only replica`.
> HAProxy health-checks both pods and always routes to whichever is the current master.
>
> Three namespaces are forced to use in-memory cache instead of Redis: `APP_CONFIG`,
> `CONFIG_STORE`, and `STARTUP_CONFIG`. These are read on almost every request, so Redis
> round-trips would add unnecessary latency."

**Meilisearch.**
> "Meilisearch is straightforward — a search engine on port 7700. LibreChat uses it for
> full-text search over conversations. Plain HTTP inside the namespace. Protected by a
> master key in a secret. It deploys one wave before LibreChat so it is ready when
> LibreChat starts."

**S3.**
> "LibreChat uploads user images and files to Hetzner Object Storage. It uses the AWS SDK
> with `AWS_FORCE_PATH_STYLE=true` — required because Hetzner uses path-style addressing,
> not the AWS virtual-hosted style. The files go into a shared bucket under `images/`
> and `files/` prefixes."

**Transition:**
> "All of these connections require credentials. Let's see how those credentials are managed."

---

## Page 6 — Secrets Pipeline

### What this diagram shows
How secrets flow from AWS Secrets Manager into the running pods.

### Walk through it like this

> "Every secret in this system comes from AWS Secrets Manager via the External Secrets Operator.
> ESO is a Kubernetes controller that watches ExternalSecret custom resources and reconciles
> them into real Kubernetes Secrets. The refresh interval is one hour — secrets rotate in the
> pod without a restart."

**Walk through the secrets.**
> "The `librechat-config` secret holds the JWT signing keys and credential encryption keys.
> LibreChat uses these to sign user JWTs and encrypt stored OAuth credentials.
>
> `librechat-openid-config` holds the Keycloak OIDC client credentials — the client ID and
> secret for the `converse` Keycloak client.
>
> `librechat-main-config` holds the most important secret: `CONVERSE_OPENAI_API_KEY`.
> This is the Lightbridge API key that LibreChat uses for all AI model calls through Envoy.
>
> The `redis-ha-redis-auth` secret is a cross-namespace copy. The Redis password lives in
> the `redis-system` namespace. LibreChat needs it in the `converse` namespace. ESO creates
> the copy."

**Point to the cert-manager secret.**
> "One secret is different: `librechat-internal-ca`. This is NOT from ESO — it is created by
> cert-manager. It carries the Home Root CA certificate. Two things need it:
> `NODE_EXTRA_CA_CERTS` — so Node.js trusts the internal CA when calling Envoy over TLS,
> and `REDIS_CA` — so the Redis TLS connection is verified correctly."

**Transition:**
> "Now let's talk about the Agent Seed Job — one of the most interesting operational mechanisms."

---

## Page 7 — Agent Seed Job

### What this diagram shows
How AI agents are automatically seeded into LibreChat after every deployment.

### Walk through it like this

**Explain the problem it solves.**
> "LibreChat stores AI agent definitions in MongoDB. These agents are the pre-configured
> AI assistants that users see when they open LibreChat — the Coder, the Researcher,
> the Planner, etc. The challenge: how do you keep these agents up to date when the
> platform team updates their configurations?"

**Walk through the mechanism.**
> "After every ArgoCD sync, when LibreChat is confirmed Healthy, ArgoCD fires a PostSync Job.
> This is a Kubernetes Job with a special annotation: `argocd.argoproj.io/hook: PostSync`.
>
> The Job runs the LibreChat Docker image itself — same image as the main app — and executes
> a Node.js script that does two things:
>
> Phase 1: It connects directly to MongoDB to find a special platform user by email
> and generates a JWT for that user. This is the identity used to make the API calls.
>
> Phase 2: It calls LibreChat's own REST API for each agent defined in the fleet JSON file.
> For agents that already exist, it does a PATCH to update them. For new agents, it does a POST.
>
> It is idempotent — running it twice produces exactly the same state."

**Explain BeforeHookCreation.**
> "The delete policy is `BeforeHookCreation`. This means before the new Job is created,
> the previous Job is deleted. This prevents orphaned Job objects piling up in the namespace.
> Every sync leaves exactly one Job."

**Why not an Init Container?**
> "You might wonder: why not just use an Init Container? Init Containers run before
> the main pod starts. But the seed script calls LibreChat's own API — LibreChat needs
> to be running and healthy first. PostSync guarantees this."

**Transition:**
> "Now let's look at a completely different capability: how developers bootstrap their
> opencode CLI from this platform."

---

## Page 8 — opencode Well-Known

### What this diagram shows
The nginx static server, the well-known JSON document, and the OAuth2 plugin flow.

### Walk through it like this

**Explain what this is.**
> "A developer wants to use the opencode AI coding assistant pointed at our platform.
> They run: `opencode auth login https://ai.camer.digital/opencode`.
>
> The opencode CLI fetches `https://ai.camer.digital/opencode/.well-known/opencode`.
> This is a JSON document served by a static nginx pod — Leaf 3 of our chart family.
> The entire document is 2125 lines of YAML in the values file, rendered into a ConfigMap
> and served by nginx."

**Walk through what the document contains.**
> "The document bootstraps the CLI with everything it needs:
>
> - The AI provider baseURL: `https://api.ai.camer.digital/v1` — the public Envoy gateway.
> - The auth config — but this is a stub. The real auth is handled by a plugin.
> - The list of plugins to install, including `@vymalo/opencode-oauth2` and
>   `@vymalo/opencode-ratelimit`.
> - The full list of available AI models.
> - MCP server configurations (all disabled by default — opt-in per developer)."

**Walk through the auth plugin flow.**
> "The oauth2 plugin intercepts every AI request and injects an `Authorization: Bearer` header.
> It uses the device_code flow against Keycloak — Keycloak client `opencode-cli`.
>
> Why device_code? Because `authorization_code` flow needs a local callback port.
> In SSH sessions, containers, and CI/CD environments, there is no browser to receive a
> redirect on localhost. Device code flow just prints a URL — the developer opens it in any
> browser on any machine."

**Explain the rate limit plugin.**
> "The `opencode-ratelimit` plugin handles 429 responses gracefully. It has two tiers:
> - If the `x-ratelimit-reset` header says reset is in ≤120 seconds: this is an RPM bucket.
>   The plugin waits up to 65 seconds and retries up to 3 times automatically.
> - If reset is >120 seconds: this is a monthly budget. The plugin errors immediately
>   so the user sees the message instead of the CLI freezing."

**Transition:**
> "Now let's look at how traffic gets routed in and out of this whole system."

---

## Page 9 — Networking

### What this diagram shows
All ingresses, services, TLS configuration, and the vanity domain redirect.

### Walk through it like this

**Main ingress.**
> "The main entry point is `ai.camer.digital`. The Traefik IngressController terminates
> TLS using a certificate issued by cert-manager with ACME HTTP-01 challenge.
> The certificate is stored in the Secret `ai.camer.digital-tls`.
> All traffic on `/` goes to `librechat-app:3080`."

**Vanity domain redirect.**
> "There is also `ai.kivoyo.com` — a vanity domain. Any request to `ai.kivoyo.com`
> is redirected to `ai.camer.digital` via a Traefik Middleware that does a regex replacement.
> It is a 302 — temporary redirect, not permanent — so browsers do not cache it aggressively.
>
> The Ingress for kivoyo.com has a backend pointing to `librechat-app:3080`, but that backend
> is never reached in practice. The Traefik middleware intercepts and redirects before
> the request gets to the backend. The backend spec is there only because Kubernetes Ingress
> requires one."

**Services.**
> "Internally, four Kubernetes Services expose the components. The MongoDB service is headless
> — `ClusterIP: None` — which is what allows the pod-direct DNS lookup via `db-0.db-headless`."

**HPA.**
> "LibreChat has a HorizontalPodAutoscaler: minimum 1, maximum 4 replicas.
> It scales on CPU >70% or memory >80%. The PodDisruptionBudget ensures at least
> 1 LibreChat pod is always available during node drains or upgrades."

**Transition:**
> "Finally, let's put everything together and trace a real request from the browser to the
> AI model and back."

---

## Page 10 — Full Request Flow

### What this diagram shows
The complete end-to-end journey of a user's AI request.

### Walk through it like this

**Walk top to bottom.**

> "A user types a message in their browser. The browser sends HTTPS to Traefik.
> Traefik terminates TLS and routes to `librechat-app:3080`.
>
> LibreChat receives the request. It validates the user's session — checking Redis
> for the cached session token. If valid, it constructs an AI model request.
>
> LibreChat sends `POST /v1/chat/completions` to the Envoy AI Gateway internal endpoint —
> `core-gateway-internal.envoy-gateway-system.svc.cluster.local`. It uses the
> `CONVERSE_OPENAI_API_KEY` — a single shared Lightbridge API key for all LibreChat traffic.
>
> Envoy hands the request to Authorino for authentication. Authorino validates the key
> against Lightbridge, stamps billing headers, and stamps budget metadata. All the same
> headers we saw in the Projects & Rate Limiting topic.
>
> Then the three Lua filters run in order: billing-period stamps the month, model-policy
> checks if the model is allowed, budget-limiter checks the balance.
>
> Then Lyft Ratelimit checks the RPM. If all checks pass, the request goes to the AI model
> backend. The response streams back through Envoy, through LibreChat, to the browser."

**Make the critical point.**
> "Here is the key thing to remember: ALL LibreChat users share ONE API key.
> The `CONVERSE_OPENAI_API_KEY` is not per-user. From Envoy's perspective, LibreChat
> is one billing account. Individual user quotas within LibreChat are enforced by
> LibreChat's own configuration in `librechat.yaml` — not by Envoy's rate limits.
>
> This is different from how opencode or Kilo Code work, where each developer has
> their own API key and their own individual Envoy rate-limit counters."

---

## Anticipated Questions and Answers

**Q: Why does LibreChat use only one API key for all users?**
> LibreChat's architecture does not support per-user API keys to downstream providers.
> It authenticates users via OIDC, but all model calls use a single configured API key.
> Per-user rate limiting is done inside LibreChat via its `librechat.yaml` config, not at the gateway.

**Q: What happens if MongoDB goes down?**
> LibreChat cannot function — it uses MongoDB for all persistence. There is a PDB
> (`minAvailable: 1`) to prevent accidental eviction during maintenance. MongoDB does not
> have replication in this setup (single member), so there is no automatic failover.

**Q: Why is Redis authentication disabled for MongoDB but required for Redis?**
> MongoDB is in the same namespace, same cluster, only accessible via ClusterIP. The threat model
> considers it safe without auth. Redis is a shared service used by multiple platform components,
> so it uses authentication. The security posture is different because the blast radius of a MongoDB
> compromise is limited to LibreChat, while a Redis compromise would affect the whole platform.

**Q: What is the REDIS_KEY_PREFIX for?**
> Multiple applications could share the same Redis cluster and accidentally overwrite each other's keys
> if they use the same key names. The prefix `librechat-prod-v2` namespaces all LibreChat's Redis keys.
> The `v2` suffix was introduced when the key schema changed — prefixing with v2 ensures no stale
> v1 keys interfere.

**Q: If the Agent Seed Job runs on every sync, won't it overwrite customizations users made?**
> Yes, intentionally. Agents defined in the fleet JSON are "platform-managed" agents.
> Users can create their own personal agents in LibreChat — those are stored in MongoDB and
> are not touched by the seed job. The seed job only upserts agents by name. If the platform
> team changes an agent's system prompt in `ai-helm-values`, the next sync overwrites it in
> LibreChat — this is the desired behavior.

**Q: Why is `NO_INDEX=true` set?**
> This sets the HTTP header `X-Robots-Tag: noindex, nofollow` on all LibreChat responses.
> It tells search engine crawlers not to index the site. This prevents the internal AI chat
> interface from appearing in Google search results.

---

*End of presenter's notes.*
