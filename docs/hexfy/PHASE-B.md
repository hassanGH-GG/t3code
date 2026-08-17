# Hexfy Phase B — deferred work

Hexfy is the fork of T3 Code described in `AGENTS.md` extended with a CLI client and a
cross-provider router, built additively (new files, minimal edits to existing files, staying
mergeable with upstream `pingdotgg/t3code`).

This document exists because conversation context does not survive. Read it before resuming any
Phase B item.

## What "Phase B" means

Phase A is the work that can be built and merged with no unresolved unknowns: it depends only on
things already verified in this codebase. Phase B is everything that is blocked on something
unmeasured, unverified, or not yet purchased (a domain, a Cloudflare account, a real Codex binary
to probe against).

**Promotion rule.** An item leaves Phase B only when its unknown has been resolved by a
timeboxed probe that produced either a working demo or a written reason it will not work. A
probe that just confirms "seems plausible" does not promote the item — it needs a concrete
artifact (a demo, a measurement, a rejection with a documented cause).

Every item below states: what it is, why it is deferred (the specific unknown), the probe that
unblocks it, and the design conclusion already reached, where one exists. File:line references
were confirmed against `claude/agent-orchestration-platform-6ogqhh` on 2026-08-17 unless marked
unverified.

---

## 1. Real cross-provider handoff (roadmap items 3.2-3.5)

**What it is.** Moving a thread's active work from one provider (e.g. Claude) to another (e.g.
Codex) without losing the conversation, so a user can route around a rate limit or a capability
gap mid-task.

**Design already decided (do not re-litigate):**

- v1 is a **new sibling thread** on the target provider, in the **same worktree**, seeded with a
  brief delivered as a `role: "system"` message.
- The sibling is linked to its origin by a durable `handoffFrom` field, modeled directly on the
  existing `sourceProposedPlan` reference pattern already in the schema
  (`packages/contracts/src/orchestration.ts:368,842,861,1256` — an optional
  `{ threadId, planId }`-shaped pointer carried on commands/events and projected onto turns via
  `apps/server/src/orchestration/Layers/ProjectionPipeline.ts:1156-1157`). `handoffFrom` should be
  the same shape of thing: `{ threadId, turnId }` pointing at the source, not a blob of copied
  history.
- Explicitly **not** mutating the origin thread's provider binding. That binding is enforced in
  exactly one place — `apps/server/src/orchestration/Layers/ProviderCommandReactor.ts:588` (cross-driver
  rejection: `"Thread '<id>' is bound to driver '<A>' and cannot switch to '<B>'."`) and `:598`
  (same-driver-but-incompatible-resume-state rejection) — and reusing it for handoff means owning
  four guard sites plus turn-derivation heuristics that assume one driver per thread for the life
  of the thread. A sibling thread avoids all of that.
- Parent and child render as **one conversation** in the UI (client-side stitching on
  `handoffFrom`, not a server-side merge).

**Constraints to bake into the design:**

1. **Granularity.** Handoff happens per-turn, at a checkpoint boundary only. Gate it on the
   durable `thread.turn-diff-completed` event — confirmed real and durable
   (`packages/contracts/src/orchestration.ts:1086,1481`; emitted by the decider at
   `apps/server/src/orchestration/decider.ts:1316`; consumed by the projector at
   `apps/server/src/orchestration/projector.ts:647` and by `CheckpointReactor.ts:429,847,922`).
   **Do not gate on `RuntimeReceiptBus`** (`apps/server/src/orchestration/Services/RuntimeReceiptBus.ts`,
   `apps/server/src/orchestration/Layers/RuntimeReceiptBus.ts`) — per the source findings this is a
   production no-op; this claim about its no-op status was not independently re-verified in this
   pass and should be treated as carried over from prior investigation, not re-confirmed.
2. **Export order.** Export the provider-native snapshot (session transcript / resume state)
   **before** stopping the source session. `readThread`-style access needs a live provider
   session, and sessions are reaped after roughly 30 idle minutes (per source findings; the exact
   reap interval was not re-verified against code in this pass — confirm the constant before
   relying on the number).
3. **Framing.** The seeded brief must say truthfully "another agent did this work" — providers
   behave badly when handed a transcript claiming to be their own memory. This is a prompt-design
   constraint, not a schema constraint.
4. **Budget.** Treat the target provider's prompt budget as the real design constraint: verbatim
   recent turns plus a summarized remainder, not the full transcript.

**Why deferred.** Two things are genuinely unmeasured: how much context actually survives a
handoff (does the target provider produce materially worse output after a synthesized seed vs. a
native resume?), and what a cold re-seed truly costs in tokens (full uncached prompt at whatever
size the brief ends up). Neither can be answered without building the seeding path and running it
against real provider sessions.

**Mitigation already designed, to preserve in the implementation.** Store resume cursors
**per (thread, provider)** rather than one-per-thread. Today a resume cursor is one-per-session
and a session is one-per-thread by construction
(`apps/server/src/provider/Layers/ProviderService.ts` — one active session per thread; exact line
range for the constructor logic was not re-verified in this pass, carried over from prior
investigation as `:372-430`). If Hexfy instead keys resume cursors by `(thread, provider)`,
returning to a provider re-enters **its own warm session** instead of re-seeding cold — so
A/B alternation between two providers only replays the material added while the thread was away
on the other provider, not the whole transcript every time. This is additive persistence (new
column/table), not a change to existing resume behavior.

**Probe.** Build the sibling-thread + brief-seed path for one provider pair (see 3.6/3.7 below)
and run a real multi-turn task through at least one handoff. Measure: tokens consumed by the
seed, and a qualitative judgment (does the target provider pick up the task correctly). Timebox:
folds into the 3.6 probe below, since there is no cheaper way to measure this than trying it.

---

## 2. Codex `thread/inject_items` probe (roadmap item 3.6)

**What it is.** The Codex app-server protocol schema defines an RPC method
`thread/inject_items`, present in the generated schema at
`packages/effect-codex-app-server/src/_generated/schema.gen.ts:32377,32840` and in the method
tables at `packages/effect-codex-app-server/src/_generated/meta.gen.ts:27,219,314,514,609`.

**Confirmed: zero callers anywhere outside the generated schema files.** A repo-wide search for
`inject_items` / `injectItems` under `apps/` returns nothing, and under `packages/` it only
appears in the two generated files above (`schema.gen.ts`, `meta.gen.ts`). No adapter, driver, or
test in this repo calls it.

**Why this matters.** If this method accepts synthesized conversation history, Codex gains a
**native seeding path** distinct from the generic sibling-thread-plus-system-message approach in
item 1, and the fidelity ceiling for Codex-targeted handoffs rises considerably — potentially a
real resume rather than a cold reseed.

**Why deferred.** The method is schema-only. Nothing in this codebase has ever called it, so
whether it works, what it accepts, and what it costs are all unknown until it is called against a
real Codex binary.

**Probe. Timebox: one day.** Call `thread/inject_items` against a real Codex app-server process
and answer:

- Which item shapes are accepted (does it take the same item union Codex emits over
  `codex.app-server.notification`, or a narrower injection-specific shape)?
- Do reasoning items require encrypted content to be accepted (Codex's own reasoning items carry
  provider-encrypted payloads in normal operation — does injection need the same encryption, and
  if so, can Hexfy produce it)?
- Do injected reasoning/content items survive **cross-account** injection (item produced under
  account A, injected into a session under account B) or is injection scoped to the originating
  account/session?
- Which Codex CLI/app-server releases actually expose this method (it may be newer than the
  pinned version, or gated behind a flag)?

**Outcome that promotes this item:** a working demo of `thread/inject_items` successfully seeding
a Codex session with synthesized history, OR a written note of exactly which question above
killed it (e.g. "requires encryption Hexfy cannot produce" or "method rejects cross-account
input").

---

## 3. Remaining provider pairs (roadmap item 3.7)

**What it is.** Handoff support for every pair beyond the first proven one — e.g. Claude→Cursor,
Codex→Grok, OpenCode→Claude, and so on across all five drivers
(`apps/server/src/provider/builtInDrivers.ts` — Codex, Claude, Cursor, Grok, OpenCode).

**Why deferred.** This is explicitly undesignable before the first pair (item 1/2 above) is
proven. Each target provider needs its own seeding decision: does it have a native injection path
like Codex's `thread/inject_items` (unproven, see item 2), or does it only get the generic
sibling-thread-plus-system-message path? The three "dark" harnesses (Cursor, Grok, OpenCode) have
not been individually investigated for a native seeding mechanism analogous to
`thread/inject_items` — that investigation is itself part of this deferred item, not yet done.

**Probe.** For each remaining provider, repeat the item-2-style investigation: does its adapter or
underlying protocol expose anything resembling history injection? (`ClaudeAdapter.ts`,
`CursorAdapter`/driver, `GrokAdapter`/driver, `OpenCodeAdapter`/driver — adapter files under
`apps/server/src/provider/Layers/`.) Timebox per provider: half a day of protocol/schema reading
before defaulting to the generic sibling-thread path.

---

## 4. Budgets for Grok, Cursor, and OpenCode (roadmap item 2.4)

**Verified (this codebase, current state).** Of the two normalized signals that exist in the
provider-runtime event vocabulary:

| adapter         | `thread.token-usage.updated`                                      | `account.rate-limits.updated` |
| --------------- | ----------------------------------------------------------------- | ----------------------------- |
| ClaudeAdapter   | yes (`ClaudeAdapter.ts:3507` handles SDK `rate_limit_event`)      | yes                           |
| CodexAdapter    | yes (`CodexAdapter.ts:1395` handles `account/rateLimits/updated`) | yes                           |
| CursorAdapter   | no                                                                | no                            |
| GrokAdapter     | no                                                                | no                            |
| OpenCodeAdapter | no                                                                | no                            |

Confirmed independently in this pass: `packages/effect-acp` (the ACP-based transport shared by
some of these adapters) has no usage-related fields; `UsageProviderKind` — the type gating which
providers `apps/server/src/usage/UsageService.ts` will even attempt to price —
is literally `Schema.Literals(["claude", "codex"])` at `packages/contracts/src/usage.ts:26`. Grok,
Cursor, and OpenCode are not just unimplemented, they are outside the type today.

**Why deferred.** These three providers must therefore get their budgets from Hexfy's own token
counting against a user-declared ceiling, since the providers themselves report nothing. Whether
Hexfy's own tokenizer's counts track closely enough to each vendor's real metering to be useful
for routing decisions is unmeasured.

**Design conclusion already reached: the three-tier model.**

| tier | mechanism                                                      | applies to    |
| ---- | -------------------------------------------------------------- | ------------- |
| 1    | provider-reported usage/limits                                 | Claude, Codex |
| 2    | Hexfy-metered token count against a user-declared ceiling      | all five      |
| 3    | reactive circuit breaker (route away after a live limit error) | all five      |

The tier an estimate came from **must travel with the number** — a UI or router that cannot tell
tier-1 (measured) from tier-2 (estimated) from tier-3 (unknown until it fails) will render
estimates as if they were measurements, which is dishonest and will mislead routing decisions.

**Design conclusion already reached: the ranking unit.** Rank/route on **fraction of that
provider's own current window consumed** (a 0..1 number) plus the window's reset instant — never
a summed cross-provider "points" pool. A router needs to pick a winner among heterogeneous
budgets (Claude's `utilization` 0..1, Codex's `usedPercent` per window), not compute a total; a
pooled sum is only meaningful if the underlying units are commensurable, and they are not (see
`docs/hexfy/RISKS.md`, risk 4).

**Probe.** Instrument Hexfy's own token counter against real Cursor/Grok/OpenCode sessions and
compare its running count to whatever ceiling the vendor documents for that plan tier (public
plan-limit pages, not an API — none of the three expose one). Timebox: this is inherently
per-provider and open-ended; treat each provider as its own probe, half a day each, and accept
"no public ceiling to compare against, tier 2 stays best-effort" as a valid, promoting outcome for
a given provider if that is what's found.

---

## 5. `/handoff` at the prompt (roadmap item 4.8)

**What it is.** A CLI/prompt-level command letting a user explicitly say "hand this off to
Codex" mid-conversation.

**Why deferred.** Purely blocked on handoff (items 1-3) existing at all. No independent unknown
of its own — this is UI/UX surfacing work on top of a mechanism that does not yet exist.

**Probe.** None needed beyond items 1-3 landing. Promotes automatically once a working handoff
path exists for at least one provider pair.

---

## 6. What an unattended daemon forgets (roadmap item 4.12)

**What it is.** Whether Hexfy can run as a headless/detached daemon (no foreground GUI client
attached) and still have fresh provider state.

**Verified mechanism.** `apps/server/src/background/BackgroundPolicy.ts:58` sets
`DEFAULT_LEASE_TTL_MS = 45_000` (45 seconds; max lease `MAX_LEASE_TTL_MS = 120_000` at line 59).
This lease is consumed by, among others, all five provider driver files
(`apps/server/src/provider/Drivers/{ClaudeDriver,CodexDriver,CursorDriver,GrokDriver,OpenCodeDriver}.ts`)
and `apps/server/src/vcs/VcsStatusBroadcaster.ts`. The general shape — background/opportunistic
work gated on `BackgroundPolicy.shouldRunScopeWork` / `shouldRunOpportunisticWork`, which in turn
depend on a live client activity lease — is confirmed in `BackgroundPolicy.ts`. The specific claim
that provider snapshots, model catalogs, and vcs/git status **all** stop refreshing once no
foreground client's lease is active was not exhaustively traced consumer-by-consumer in this
pass; the driver files and `VcsStatusBroadcaster.ts` import `BackgroundPolicy`, which is strong
circumstantial evidence, but the model-catalog-refresh code path specifically was not opened and
read.

**Why deferred.** If the above holds fully, "detached daemon" and "fresh numbers" cannot
currently both be true — exactly the state a background router (items 8-9 below) needs to be
false, since routing on stale rate-limit data defeats the purpose.

**Probe.** Start the server, let 90+ seconds pass with no client holding a lease (no web/desktop/
mobile client attached, so the ~45s TTL definitely expires), then check whether provider
snapshots, model catalog data, and vcs status actually stop updating. This is a runnable
experiment, not a code-reading exercise — confirm empirically before designing around it.

**Decision to make once the probe lands:**

- **Option A:** give the CLI itself a client-kind that holds a lease (a contract addition — some
  concept of "background CLI client" that counts toward `hasDemand`/lease accounting in
  `BackgroundPolicy`).
- **Option B:** let the specific pollers a background router needs (rate limits, at minimum) run
  ungated by the lease system, accepting the resource-usage tradeoff `BackgroundPolicy` exists to
  prevent.

---

## 7. Automatic routing (roadmap items 6.2-6.4)

**What it is.** The router picking a provider automatically — first by budget (route away from an
exhausted provider), later by task-type/capability (route research vs. implement vs. review to
whichever provider is "best" at it).

**Why deferred.** Only trustworthy once (a) budgets are proven accurate enough to route on (item 4) and (b) handoff is real enough that a routing decision can actually be executed without losing
the thread (items 1-3). Routing on unmeasured budgets or routing into a handoff that degrades
output quality would make the product worse than manual switching.

**Design conclusions already reached:**

- **Ordering: budget routing before task-type routing.** Budget is measurable (a number against a
  ceiling); "provider X is better at research" is a preference/heuristic with no ground truth in
  this codebase. Ship the measurable one first.
- **Record every routing decision as an event.** `ModelSelection` — the actual routing key,
  `{ instanceId, model, options? }` — is defined at `packages/contracts/src/orchestration.ts:83`
  and currently comes from the client (a human clicking a model picker). A Hexfy router is a
  function `(intent, budgets, capabilities, pins, history) -> ModelSelection`. Recording that
  decision as a new orchestration event (additive member of the union at
  `packages/contracts/src/orchestration.ts:1343` onward, ~30 event types today) gives a readable,
  arguable decision log for free — "why did it switch me to Codex" has an answer.

**Precedent that automatic provider-driven switching already exists, one level down:** Codex
itself emits `model.rerouted` (`{ fromModel, toModel, reason }`), sourced from the app-server
`model/rerouted` notification and handled at `apps/server/src/provider/Layers/CodexAdapter.ts:1324`.
Claude does not emit an equivalent. So a provider already tells T3 "I moved you from model A to
model B because `<reason>`" — Hexfy's contribution is doing that same kind of move **across**
providers, not inventing the concept.

**Probe.** No new probe beyond items 1-4 landing; this item's promotion is entirely downstream of
theirs.

---

## 8. Hosted access (roadmap items 7.1-7.2)

**What it is.** Two genuinely distinct products, not one feature with two flags:

- **Flavour A — hosted CONNECTION.** Agents keep running on the user's own machine; the cloud
  component only brokers reachability (so a phone can reach a laptop that's asleep on wifi, push
  notifications, etc.).
- **Flavour B — hosted COMPUTE.** A rented machine runs the vendor CLIs themselves, which requires
  the user's Claude/Codex/etc. subscription credentials to live on rented infrastructure Hexfy
  does not own.

**Verified: flavour A's infrastructure already exists in-repo.** `infra/relay/` is a real
Cloudflare Worker project: `infra/relay/src/worker.ts` as the entry point, `infra/relay/alchemy.run.ts`
for infra-as-code, `infra/relay/migrations/` for Drizzle migrations, DPoP-based auth under
`infra/relay/src/auth/` (confirmed present: `auth/` subdirectory exists alongside `agentActivity/`,
`db.ts`, `dbConfig.ts`, `deploymentConfig.ts`, `environments/`, `http/`, `observability.ts`,
`persistence/`, `queues.ts`, `zone.ts`; the specific filenames `DpopProofs.ts` and
`RelayTokens.ts` inside `auth/` were not individually re-opened in this pass, carried over from
prior investigation), plus APNs push and mobile registrations per the directory layout
(`agentActivity/`).

**Verified: cloud is gated entirely behind build-time values**, so building without them yields a
fully local product. `apps/server/src/cloud/publicConfig.ts:14` declares
`__T3CODE_BUILD_RELAY_URL__: string | undefined`; when unset the relay URL resolves to empty
string (`:53-55`). `hasCloudPublicConfig` at `:211` gates whether the `connect` CLI subcommand is
even offered in `apps/server/src/bin.ts`.

**Why flavour B is a materially different decision, not deferred for lack of code but for lack of
a decided posture:** running vendor CLIs on infrastructure Hexfy operates means the user's
subscription credentials sit on that infrastructure. This is the same posture Anthropic itself
uses for Claude Code on the web — but Anthropic owns both the compute and the subscription
product in that case. Hexfy would be putting a third party's subscription credentials on a fourth
party's (Hexfy's) rented infrastructure, a different trust and terms-of-service shape. See
`docs/hexfy/RISKS.md` risk 1 for the Anthropic-terms analysis this rests on.

**Recommendation on record:** build flavour A, defer flavour B as its own separate decision
requiring explicit sign-off (not something to slide into during flavour A's implementation).

**What blocks flavour A specifically:** not code — a domain, an auth tenant (Clerk, per the
existing `__T3CODE_BUILD_CLERK_PUBLISHABLE_KEY__` / `__T3CODE_BUILD_CLERK_CLI_OAUTH_CLIENT_ID__`
build vars also declared in `publicConfig.ts`), and a Cloudflare account to deploy Hexfy's own
copy of `infra/relay/` to. These are purchases/signups, not engineering unknowns — hence Phase B,
not Phase A, purely because they have not been acquired yet.

**Probe.** Acquire the domain, Clerk tenant, and Cloudflare account; deploy `infra/relay/` under
Hexfy's own identity; set the build vars; confirm `connect` works end to end for one real device
pairing. This is closer to a checklist than a research probe — the code path is already proven by
upstream running it in production.

---

## 9. Multi-account for the three dark harnesses

**What it is.** T3 Code's multi-account support today is driver-specific: Claude keys continuation
identity on its config directory (accounts are isolated per config dir); Codex uses a
shared-home-plus-shadow-home scheme (`apps/server/src/provider/Drivers/CodexHomeLayout.ts:55,64` —
`continuationKey: `codex:home:${sharedHomePath}``, so two accounts sharing a `CODEX_HOME` are
continuation-compatible). Cursor, Grok, and OpenCode have **no config-directory concept in their
drivers at all** — there is nothing analogous to Claude's config-dir isolation or Codex's
shared-home scheme to key multi-account behavior off of for these three.

**Why deferred.** Difficulty is unknown per adapter. Each of the three dark harnesses would need
its own investigation into how (or whether) its underlying CLI supports running multiple
authenticated identities side by side at all, before any T3-side multi-account mechanism could be
designed for it.

**Probe.** Per provider (Cursor, Grok, OpenCode): does the vendor CLI support multiple
authenticated profiles/accounts at all (env var override, config flag, separate home directory)?
If yes, what does T3's driver need to key continuation identity on, analogous to Claude's config
dir or Codex's shared home? Timebox: half a day per provider, protocol/CLI-flag investigation
only — no code changes.

---

## Summary table

| item                                   | blocked on                                           | probe timebox                           |
| -------------------------------------- | ---------------------------------------------------- | --------------------------------------- |
| Real handoff (3.2-3.5)                 | context-survival + cold-reseed cost, both unmeasured | folds into 3.6 probe                    |
| Codex `inject_items` (3.6)             | never called against a real binary                   | 1 day                                   |
| Remaining provider pairs (3.7)         | first pair unproven                                  | 0.5 day per provider, after 3.6         |
| Budgets for Grok/Cursor/OpenCode (2.4) | tokenizer accuracy vs. vendor metering unmeasured    | 0.5 day per provider                    |
| `/handoff` prompt (4.8)                | handoff not existing                                 | none — auto-promotes with 3.2-3.5       |
| Unattended daemon (4.12)               | lease-gated refresh unconfirmed empirically          | run server, wait 90s, observe           |
| Automatic routing (6.2-6.4)            | budgets + handoff both unproven                      | none — auto-promotes with 2.4 + 3.2-3.5 |
| Hosted access (7.1-7.2)                | domain/tenant/Cloudflare account not purchased       | checklist, not research                 |
| Multi-account, dark harnesses          | per-adapter difficulty unknown                       | 0.5 day per provider                    |
