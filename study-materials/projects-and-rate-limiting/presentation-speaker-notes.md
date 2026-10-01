# Projects & Rate Limiting — Presenter's Speaker Notes

> **File to open**: `diagrams/projects-and-rate-limiting.drawio`  
> **Tip**: In Draw.io, use the page tabs at the bottom to navigate between diagrams.  
> This document gives you the exact words to say for each page, in order.

---

## Before You Begin — Setting the Stage (No diagram yet)

Say this to frame the whole topic before opening the file:

> "This system has two separate but connected concerns. The first is **Projects** — a way for
> teams and organisations to group their API keys, control which AI models they can use, and
> manage a shared budget. The second is **Rate Limiting** — the set of mechanisms the gateway
> uses to actually *enforce* those budgets and usage limits on every single request.
>
> These two things talk to each other only through HTTP headers. Lightbridge, the auth service,
> decides the rules. Envoy, the gateway, enforces them. They never share a database.
> Let me show you how that works, starting from the ground up."

Then open the `.drawio` file and go to **Page 1**.

---

## Page 1 — Data Model

### What this diagram shows
The core database schema that defines what a Project *is* in this system.

### Walk through it like this

**Start with ACCOUNTS.**
> "Everything starts with an Account. An Account is essentially one billing identity — it maps
> directly to a Keycloak user. The `id` column is literally the JWT `sub` claim — the unique
> identifier Keycloak puts in every token."

**Move to PROJECTS.**
> "A user can own multiple Projects. A Project is the core grouping unit. Look at the fields:
>
> - `billing_plan` — this is the plan the whole project runs on: free, pro, enterprise, etc.
> - `model_policy` — this is the key one: `allow_all`, `allowlist`, or `deny_all`. This controls
>   which AI models every API key in this project can call.
> - `allowed_models` — a JSON list of model IDs. Only relevant when `model_policy = allowlist`.
> - `project_quota` — a pooled spending ceiling for the whole project. This field exists in the
>   database but is **not yet enforced at the gateway**. That is future work.
> - `status` — a project can be `active` or `suspended`. Suspension is stored but not yet enforced
>   at the gateway either."

**Move to PROJECT_MEMBERS.**
> "A project has a roster of member accounts. Each member has a role — `lead` or `member`.
> There is also a `quota_tier` field which would give different members different spending limits.
> Again, this field exists in the DB but is not yet enforced. The Helm chart has the template
> for it, but the list is empty today."

**Move to API_KEYS.**
> "Finally, a project issues API keys. Each key is linked to exactly one project and one
> owner account. The actual secret is never stored — only its SHA-256 hash. The `billing_plan`
> is copied from the project at the moment the key is minted. If the project's plan changes
> later, existing keys keep their old plan until they are rotated."

**Point to the arrows.**
> "The relationships: one Account owns many Projects. A Project has many Members and many API Keys.
> An Account can be a member of many Projects — and its API keys are billed against its account."

**Pause for questions, then say:**
> "Now let's see what you actually *do* with a project."

---

## Page 2 — Project Lifecycle

### What this diagram shows
The management operations — what an admin does to set up a project, in sequence.

### Walk through it like this

**Step 1 — Create.**
> "You call the Lightbridge API with a `POST /projects`. The API writes a row to PostgreSQL.
> It returns a project ID. That is the entire creation step. There is no Envoy involvement here,
> no Helm chart change. This is purely a database operation."

**Step 2 — Restrict models.**
> "Now the admin wants to limit developers to only `gpt-4o`. They send a `PATCH` with
> `model_policy: allowlist` and `allowed_models: ['gpt-4o']`. The API updates the projects row.
> From this moment on, any request made with a key from this project will be checked against
> this list at the gateway — in real time, on every single request."

**Step 3 — Add a member.**
> "Adding a team member is just inserting a row into `project_members`. The member account
> can now use the project's API keys. Roles are `lead` or `member`. Leads will eventually
> have elevated permissions within the project."

**Step 4 — Mint a key.**
> "The final step is minting an API key. The API generates a random secret, stores only its
> hash, and returns the plaintext secret exactly once. If you lose it, you cannot recover it —
> you revoke it and mint a new one. The key inherits the project's `billing_plan` at mint time."

**Transition:**
> "So that is the management plane. All of this is self-service — users can do it themselves
> through the API or through the AI assistant via the MCP server. Now let's look at how a
> restriction set in Step 2 actually becomes a blocked request at the gateway."

---

## Page 3 — Model Policy Enforcement

### What this diagram shows
The real-time enforcement path: how a database setting becomes a HTTP 403 Forbidden.

### Walk through it like this

**Set the scene.**
> "A developer just joined the project. They have an API key. They try to call `deepseek-r1`.
> But the project admin set `model_policy: allowlist` with only `gpt-4o` allowed.
> Watch what happens."

**Phase 1 — Identity Resolution.**
> "The request arrives at Envoy. Envoy immediately pauses it and hands the API key to Authorino
> — the external authorisation service. Authorino does a token exchange with Keycloak.
> Keycloak talks to the Lightbridge API. Lightbridge reads the project row from PostgreSQL and
> returns `model_policy: allowlist, allowed_models: ['gpt-4o']`.
>
> Authorino takes that and stamps it as headers onto the *internal* representation of the request:
>   - `x-model-policy: allowlist`
>   - `x-allowed-models: gpt-4o`
>   - `x-ai-eg-model: deepseek-r1` — this one is set by the AI Gateway ext_proc which reads
>     the model field from the request body"

**Phase 2 — Lua Enforcement.**
> "Now the Lua filter runs. It's a short script called `model-policy.lua`. It reads these three
> headers and makes one decision:
>   - Policy is `allowlist`
>   - Allowed list is `['gpt-4o']`
>   - Requested model is `deepseek-r1`
>   - `deepseek-r1` is NOT in `['gpt-4o']` → DENY
>
> Envoy returns `403 Forbidden`. The AI backend is never even contacted. The whole thing
> happens in a few milliseconds on every request."

**Key point to emphasise:**
> "Notice that Envoy does not call Lightbridge on every request. It only calls Authorino —
> which in turn calls Lightbridge — once at authentication time. The result gets stamped as
> headers. After that, the Lua filter makes the decision purely from those headers.
> This is why the system is fast."

**Transition:**
> "But model policy is only one dimension of control. Let's zoom out and see how an API key
> becomes ALL the headers and metadata that Envoy uses to make decisions."

---

## Page 4 — Identity Resolution

### What this diagram shows
The full picture of what Authorino stamps on every authenticated request.

### Walk through it like this

> "When a user sends a request with an API key, by the time that request reaches the Lua
> filters and the rate-limit checks, it has a lot of extra information attached to it.
> Here is exactly what gets added and where it comes from."

**Walk left to right.**
> "The user sends a request. Envoy intercepts it. Authorino validates the key. Keycloak does
> the token exchange — calling the Lightbridge API — and the Lightbridge API queries PostgreSQL
> for the project row.
>
> All of that happens in a single authentication step. Then Authorino stamps these headers:
>
>   `x-billing-plan: pro` — so Envoy knows which plan's rate limits to apply  
>   `x-account-id: proj-abc` — so rate-limit counters are keyed per project  
>   `x-model-policy: allowlist` — for the model policy Lua filter  
>   `x-allowed-models: gpt-4o, claude-3-5-sonnet` — the allowlist itself  
>   `x-billing-period: 2026-10` — **this one is special**: it is NOT set by Authorino.
>   A separate Lua filter runs first and stamps the current calendar month. This is what makes
>   monthly budget counters reset on the 1st of each month."

**Then point to the dynamic metadata box.**
> "And separately — not as headers, but as internal Envoy metadata that only Envoy can see —
> Authorino also stamps the account's live balance from the Lightbridge ledger:
>
>   `budget.remaining_micros: 8457487` — that is about $8.46  
>   `budget.decision: allow`  
>   `budget.known: true` — meaning the ledger was reachable and returned a real number  
>   `budget.enforced: true` — meaning this caller is subject to budget enforcement"

> "This is exactly what the Budget Limiter Lua filter reads. It does not call anything.
> It just reads this number. If it's positive, allow. If it's zero or negative, 402."

**Transition:**
> "Now you have the full picture of what information is available. Let's see the two
> enforcement mechanisms that use this information."

---

## Page 5 — Two Guardians

### What this diagram shows
The high-level architecture of the two independent enforcement systems.

### Walk through it like this

> "There are exactly two active mechanisms enforcing limits in production today.
> They are completely independent — one does not know the other exists.
> Think of them as two different people checking your ticket at two different doors."

**Left side — Speed Cop.**
> "The first is the Speed Cop. It is powered by the Lyft ratelimit service and Redis.
> It cares about one thing only: how many requests per minute is this caller sending to
> this specific model?
>
> If they exceed the limit, Envoy returns `429 Too Many Requests`.
> The limit resets automatically every 60 seconds."

**Right side — The Bank.**
> "The second is the Bank. It is powered by a Lua script inside Envoy that reads metadata
> set by Authorino. It cares about one thing: does this account have any balance left in the
> Lightbridge ledger?
>
> If the balance is zero or negative, Envoy returns `402 Payment Required`.
> This only clears when someone tops up the account — there is no automatic reset."

**Point to outcomes.**
> "Both checks happen on every request. The speed cop runs last. But a request only reaches
> the speed cop if the bank already allowed it. A request can fail at either gate
> independently."

**Pause and say:**
> "Now let me show you each of these in detail."

---

## Page 6 — RPM Per Key (Speed Cop)

### What this diagram shows
The exact mechanics of how the Lyft ratelimit service and Redis enforce requests-per-minute.

### Walk through it like this

> "When a request arrives, Envoy has a sidecar called the Lyft ratelimit service.
> On every request, Envoy asks it one question: 'Is this caller over their limit?'"

**Walk through the flow.**
> "Lyft builds a Redis key using:
>   - The rule's *position* in the BackendTrafficPolicy list (we'll come back to this)
>   - The API key ID
>   - The model name
>
> So the key looks like: `rpm-rule/0/key-abc123/deepseek-r1`
>
> Lyft increments this counter in Redis by 1. Redis returns the new count.
> If the count is under the limit (say 60 req/min) → Lyft says allow.
> If the count hits 60 → Lyft says deny → 429."

**Explain the two rules.**
> "There are always *two* rules per model, not one. Rule 0 is keyed on the API key ID.
> Rule 1 is keyed on the account ID. Why two? Because not all callers have an API key ID.
> Internal services like LibreChat authenticate differently. Rule 1 catches those.
> Both rules have the same limit. The first one to fire wins."

**Explain the counter reset.**
> "The Redis counter has a 60-second TTL. It expires automatically. No one needs to reset it.
> If you hit 60 requests in one minute, you just need to wait for the window to slide."

**Explain rpmPerKey: 0.**
> "Setting `rpmPerKey: 0` on a model completely opts out of this mechanism — no rules
> are rendered for that model in the BackendTrafficPolicy. Zero = disabled."

**Transition:**
> "Now the money enforcement."

---

## Page 7 — Budget Limiter (The Bank)

### What this diagram shows
How the Lua script makes real-time 402 decisions using balance metadata from Lightbridge.

### Walk through it like this

> "The budget limiter is a Lua script running *inside* Envoy. It does not call any external
> service. It reads metadata that was already stamped on the request during authentication.
> The script makes one comparison."

**Walk through the decision table.**
> "The happy path: Authorino stamped `budget.remaining_micros = 8457487`.
> The script reads that: 8,457,487 > 0 → allow. Done. The request continues."

**Walk through the deny path.**
> "If the account has run out of money, `remaining_micros` will be zero or negative.
> The script returns a `402 Payment Required` with a JSON body that tells the user:
>   - How much they have left (which is 0 or negative)
>   - When their budget will next reset
>   - A URL where they can top up their balance"

**Walk through the 503 paths.**
> "There are two cases where the script returns 503 instead of 402. These are important:
>   - If `budget.known = false` — the ledger was unreachable or timed out. We do not know
>     the balance. We could not in good conscience allow OR charge. We return 503 with the
>     message 'budget unavailable'. It is OUR outage, not the user's spent budget.
>   - If a model request arrives with no budget metadata at all — that means the authentication
>     chain is misconfigured. Rather than silently letting it through unmetered, we return 503.
>     This is fail-closed design."

**Explain shadow mode.**
> "Before going live with this enforcement, the team ran it in shadow mode for a day.
> In shadow mode, the script runs and makes the exact same decisions, but instead of actually
> blocking anyone, it just logs what it *would* have done. This let the team measure:
> 'If we had been enforcing, how many real users would have been wrongly blocked?'
> Once confident the false-positive rate was acceptable, they flipped `shadowMode: false`
> and it went live. Rollback is just setting `shadowMode: true` again — ArgoCD picks it up
> in one sync, no code change needed."

**Key contrast with the old system:**
> "The old system used Redis cost buckets — a Lyft ratelimit rule that counted micro-USD
> instead of requests. The problem: if a user got a refill, their Redis counter was still
> at its limit. The refill was in the ledger but not in Redis. A user on 2026-09-05 had
> $8.46 in their ledger after a refill but still got 429 on every request because their
> Redis bucket said 'exhausted'. An engineer had to manually delete the Redis key.
> The Lua approach fixed this completely — it reads from the live ledger, not from a Redis counter."

**Transition:**
> "Now I want to show you something that is critically important for anyone who ever touches
> the rate-limit configuration — the append-only list contract."

---

## Page 8 — Append-Only Trap

### What this diagram shows
Why inserting a new plan in the middle of the list silently gives users free budget resets.

### Walk through it like this

> "The Lyft ratelimit service has a specific quirk: it builds Redis counter keys using the
> *position* of a rule in the list — its index — not its name. This seems like an implementation
> detail, but it has a major operational consequence."

**Show the BEFORE panel.**
> "Here is the current list. `free` is at index 1. A free user has been spending this month.
> Their Redis counter is stored at the key `rule/1_x-account-id_ABC_x-billing-plan_free_...`.
> They have spent $12 of their $15 budget."

**Show the AFTER panel.**
> "Now someone adds a new plan called `basic` and inserts it at the front of the list — which
> seems alphabetically sensible. Look what happens: everything shifts down by one. `enterprise`
> moves from index 0 to index 1. `free` moves from index 1 to index 2.
>
> On the next request from that free user, Lyft looks up `rule/2` — because `free` is now
> at index 2. But `rule/2` does not exist in Redis yet. Redis has never seen this key.
> Lyft starts a fresh counter at zero. The user's $12 of spending is gone from the system's
> perspective. They get a fresh $15 — mid-month, for free. The old `rule/1` key just sits in
> Redis until it expires on its TTL."

**Emphasise this happened for real.**
> "This is not a theoretical risk. It happened live on 2026-07-16 on the core-gateway chart
> when `enterprise` was added mid-alphabetically."

**Explain the rules.**
> "The fix is simple: treat the list as append-only. You can only add new plans at the *end*.
> You can never insert, delete, or reorder.
>
> When a burst rule was removed (all of them were removed on 2026-08-01), the entry was NOT
> deleted from the list. The body was commented out. The `id:` stub remains to hold the index
> position. Every entry in the list is a reserved slot for that plan's index, forever."

**Pre-empt the obvious question.**
> "You might think: why not use a YAML map, keyed by name? Like `plans: { free: {...}, pro: {...} }`?
> The problem is that Helm sorts map keys alphabetically on every render. So inserting `basic`
> would cause exactly the same index shift — but silently, automatically, invisibly. The list
> format at least makes the ordering explicit and auditable."

**Transition:**
> "Now let's look at the three conceptual layers of rate limiting that the Helm chart is
> designed to support — even though most of them are empty today."

---

## Page 9 — Three Layers (AND Logic)

### What this diagram shows
The three families of rate-limit rules (Plans, Tiers, Envelope) and how they compose.

### Walk through it like this

> "The BackendTrafficPolicy template in the Helm chart supports up to five rule families.
> Three of them correspond to different 'layers' of control that operate with AND logic —
> if any one bucket is exhausted, the request is denied."

**Walk through each layer.**

> "**Layer 1 — User Plan**: This is a per-user, per-plan monthly spending cap. A free user
> can spend up to $50 per month across all models. An enterprise user up to $1,000.
> This was previously enforced via Redis cost buckets, but those were deleted on 2026-09-05
> because they could not be cleared by a ledger refill. Today, the Lua Budget Limiter
> handles money enforcement instead."

> "**Layer 2 — Member Tier**: This would give different members of the same project different
> spending allowances. A senior developer might get $200/month from the project pool, while
> a junior gets $50. The `quota_tier` field on PROJECT_MEMBERS holds this. The Helm template
> has the logic to render these rules. But `tiers: []` today — nobody has configured any tiers."

> "**Layer 3 — Project Envelope**: This would cap the total spend for an entire project,
> regardless of how many members it has or what plan they are on. If a project is granted
> $500/month total, that is shared across all its API keys. The `project_quota` field in
> the PROJECTS table holds this. Again, the Helm template supports it. But
> `projectEnvelope: {}` today — not configured."

**Show the current state.**
> "So in production right now, NONE of these three layers are active. The rateLimit block
> in every per-model BackendTrafficPolicy is completely empty — it does not render at all.
> The only active rate limit is `rpmPerKey` (Family 6) which we saw on Page 6.
> And the only money enforcement is the Lua Budget Limiter on Page 7.
>
> These layers are the *architecture for the future*. The infrastructure to enforce them
> is already built. Someone just needs to populate `tiers` and `projectEnvelope` in the
> values file."

**Transition:**
> "Finally, let's put everything together and watch a single request travel through the
> entire system from start to finish."

---

## Page 10 — Full Request Gauntlet

### What this diagram shows
The complete, end-to-end journey of one AI model request through every gate.

### Walk through it like this

> "This is the sequence a single request goes through every time. This is the full gauntlet.
> Let's trace it."

**Step by step from top to bottom.**

> "**1. Request arrives.** A user sends `POST /v1/chat/completions` with their API key and
> a model request body like `{ 'model': 'gpt-4o', 'messages': [...] }`.
> The AI Gateway ext_proc reads the model field from the body and stamps `x-ai-eg-model: gpt-4o`."

> "**2. Authorino.** The request pauses. Authorino validates the API key, calls Lightbridge,
> and gets back the project's billing plan, model policy, allowed models, and the account's
> current budget balance. All of this is stamped as headers and internal dynamic metadata.
> The calendar month is also stamped by the first Lua filter (`x-billing-period: 2026-10`)."

> "**3. Model Policy Lua (filter order 13).** The Lua script checks: is the requested model
> in the project's allowed list? If not → **403 Forbidden**. End of request."

> "**4. Budget Limiter Lua (filter order 14).** The Lua script reads `budget.remaining_micros`
> from the metadata. If it is zero or negative → **402 Payment Required**. End of request.
> If the ledger is unreachable → **503 Service Unavailable**."

> "**5. Lyft Ratelimit (rpmPerKey).** If the request has survived to this point, Envoy asks
> the Lyft ratelimit service: has this caller exceeded their requests-per-minute for this model?
> Lyft increments the Redis counter. If over the limit → **429 Too Many Requests**. End of request."

> "**6. AI Model Backend.** If the request has passed all five checkpoints: it is forwarded
> to the AI provider (OpenAI, Anthropic, self-hosted vLLM, etc.). The response streams back.
> The model's per-request timeout is 600 seconds (10 minutes). There are zero retries —
> LLM calls are not retried because a retry would amplify load on a stressed backend and risk
> double-charging a partial generation."

**Final summary statement.**
> "So the full picture: Lightbridge manages the data. Authorino translates that data into
> request-time headers and metadata. The Lua filters enforce model policy and budget.
> The Lyft ratelimit service enforces speed. None of these services know about each other —
> they communicate only through well-defined headers and metadata on the request.
> That is the architecture."

---

## Anticipated Questions and Answers

**Q: What happens if Lightbridge is down?**
> Authorino would fail to authenticate the request. Envoy would return 503. No requests get
> through — this is the fail-closed stance. Lightbridge going down is a full service outage.

**Q: Can a user in a project bypass model restrictions by minting their own key?**
> No. Keys are minted through the Lightbridge API which always links them to a project.
> The project's `model_policy` and `allowed_models` are read from the database at auth time,
> not from the key itself. Even if someone guesses a key, it belongs to a project and inherits
> that project's restrictions.

**Q: What is the difference between a 402 from the budget limiter vs a 429 from rpmPerKey?**
> 429 means you are sending too fast — wait 60 seconds and try again, it will work.
> 402 means your account has no money left — the only fix is a top-up. The refill URL is
> included in the 402 response body.

**Q: Why does the Lua script return 503 when the ledger is down, not 402?**
> Because 402 means "you have no budget". 503 means "we cannot tell you". These are different
> situations with different runbooks. A 503 should trigger an alert for the platform team.
> A 402 should trigger a notification to the user to top up.

**Q: What is shadow mode?**
> A dry-run mode for the Budget Limiter. The script makes all the same decisions but never
> actually blocks anything. It logs what it *would* have done. This is how you safely test
> enforcement logic on production traffic before going live. One value change turns it on or off.

**Q: Is there a way to give a specific project more budget without changing the billing plan?**
> Yes, through budget grants in the Lightbridge ledger. A grant is a one-time addition to an
> account's balance. This is how top-ups work. The `BUDGET_GRANTS` table tracks these.
> The `BUDGET_AUGMENTATION_REQUESTS` table tracks requests for augmentation that may need
> manual review for large amounts.

---

*End of presenter's notes.*
