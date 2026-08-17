# Hexfy risk register

Standing risks for the Hexfy fork (T3 Code + CLI client + cross-provider router, built
additively — see `AGENTS.md` and `docs/hexfy/PHASE-B.md`). Each entry states severity, the
evidence behind it, and the mitigation (or the honest absence of one).

Evidence is labeled by source:

- **repo** — confirmed directly against `claude/agent-orchestration-platform-6ogqhh` in this pass,
  with file:line citations.
- **web** — confirmed by web search in this pass, with source links; current as of 2026-08-17 and
  subject to change.
- **carried over** — asserted in prior investigation (this fork's own research), not
  independently re-verified in this pass. Treated as a caveat, not a fact, per the rule that
  caveats must never be silently upgraded.

---

## 1. Anthropic's terms are a hard wall for multiplexed subscription auth — compliance is UNVERIFIED

**Severity: high.** If Anthropic's terms prohibit the usage pattern Hexfy depends on, the
flagship feature (routing across providers using the user's own subscriptions) is void
regardless of how well it's engineered.

**Evidence.**

- **carried over, explicitly flagged unverified in the source investigation:** "ToS: multiplexing
  subscription-authenticated CLIs / multiple accounts — UNVERIFIED, delegated to the prior-art
  agent. Do not guess." This has not been resolved by an authoritative source (Anthropic's own
  terms of service or usage policy) in this pass or, as far as this document's inputs show, in any
  prior pass either.
- **web:** a related but distinct question — OpenAI/Codex multi-account tolerance — surfaces only
  a community forum thread, not a stated policy: "Can I use 2 ChatGPT account in codex
  simultaneously?" on the OpenAI Developer Community forum
  (community.openai.com/t/can-i-use-2-chatgpt-account-in-codex-simultaneously/1379038), where the
  answer is "OpenAI's terms don't explicitly state that users cannot have multiple accounts...
  this absence of explicit prohibition doesn't mean it's encouraged." This is evidence about a
  different vendor (OpenAI, not Anthropic) and a related but different question (multiple
  accounts on one vendor, not routing a single thread's work across multiple vendors' CLIs). It is
  not evidence about Anthropic's position, and it is explicitly _not_ policy — see risk 4 for
  where this specific forum-reply-vs-policy gap matters most directly.

**Why the design is believed to survive it (the design's own answer — not a compliance
determination).** Hexfy does not extract, store, or repurpose any provider's OAuth/session
credentials outside that provider's own CLI. Each provider's CLI runs as its own subprocess under
its own normal authentication (Claude keys on its config directory; Codex uses a
shared-home-plus-shadow-home scheme — `apps/server/src/provider/Drivers/CodexHomeLayout.ts:55,64`,
repo-verified), the same way a human running that CLI directly would authenticate. Hexfy
orchestrates _when_ to invoke which CLI and _how much conversation_ to hand it; it does not call
any vendor's API directly with credentials pulled out of that flow, and it does not pool one
vendor's quota into another's. Multi-account operation is not a novel idea T3 is introducing —
upstream already documents it as a supported pattern: `docs/user/providers-codex.md:10` lists
"switch to another account when one account hits limits" as a reason to configure multiple
accounts (repo-verified), and `docs/user/providers-claude.md:96` discusses cross-config-dir
switching within a thread ("Usually, no.") (repo-verified) — i.e., the _shape_ of running multiple
authenticated CLI identities already exists in the product Hexfy forks from; Hexfy's contribution
is doing it automatically instead of by hand. This is a defensible design position, not a legal
conclusion — it has not been checked against Anthropic's actual terms.

**Residual caution beyond the credential story: continuous background running.** Even if the
credential-handling pattern above is sound, Hexfy's own roadmap includes an unattended daemon
mode (`docs/hexfy/PHASE-B.md`, item 6 — "what an unattended daemon forgets") where the CLI issues
work with no human present to trigger each turn. Continuous, automated invocation is a materially
different usage pattern from a human interactively running the CLI turn by turn, and has not been
evaluated separately from the multi-account question above. Caution here should not be resolved
by analogy to the multi-account answer; it is its own open question.

**Mitigation.** Get an authoritative answer (terms of service text, or direct vendor contact)
before building anything that depends on simultaneous multi-account or multi-provider
subscription use at scale. Until then, treat this as a hold on shipping the pooling/routing
feature broadly, not on building it in a lab/opt-in capacity.

---

## 2. "Without losing context" is an approximation, not a guarantee

**Severity: medium-high** for user trust — a handoff that silently drops information is worse
than one that visibly starts fresh, because the user won't know what was lost.

**Evidence (repo, verified this pass).**

- No provider harness accepts a foreign transcript as native resume state. Claude's resume is
  opaque provider-owned state re-entering its own session file by UUID; Codex's resume is its own
  thread id; there is no cross-provider injection API in general use today (the one candidate,
  Codex's `thread/inject_items`, has zero callers anywhere in this repo — see
  `docs/hexfy/PHASE-B.md` item 2). So any handoff necessarily goes through a synthesized brief, not
  a faithful transcript replay.
- The durable activity payload that would need to be faithfully replayed is untyped.
  `OrchestrationThreadActivity` at `packages/contracts/src/orchestration.ts:341-350`:

  ```ts
  export const OrchestrationThreadActivity = Schema.Struct({
    id: EventId,
    tone: OrchestrationThreadActivityTone, // "info" | "tool" | "approval" | "error"
    kind: TrimmedNonEmptyString, // free-form string, not a closed union
    summary: TrimmedNonEmptyString,
    payload: Schema.Unknown, // untyped
    turnId: Schema.NullOr(TurnId),
    sequence: Schema.optional(NonNegativeInt),
    createdAt: IsoDateTime,
  });
  ```

  `kind` is a free-form string (stable values are assigned by convention at the producer, e.g.
  `"approval.requested"`, but the schema does not enforce a closed set) and `payload` is
  `Schema.Unknown`. Consumers duck-type it back out defensively (per prior investigation, in
  `ActivityPayloadProjection.ts`; not re-opened in this pass).

**Consequence.** Because the durable record itself does not carry a typed payload, there is no
way to mechanically verify that a synthesized handoff brief faithfully represents what happened in
a turn — "faithful" can only be judged by eye, turn by turn, until the payloads are typed. Summary-
grade handoff (a human-readable digest) is feasible today; faithful, verifiable handoff is not.

**Mitigation.** Type the payload per `kind` (a discriminated union keyed on the existing `kind`
strings) before treating handoff output as anything more than best-effort. This is additive
schema work — no existing producer needs to change its behavior, only the contract needs to grow a
union.

---

## 3. Switching to save quota can cost quota

**Severity: medium.** This risk can invert the feature's own value proposition if unmeasured.

**Evidence (repo, verified this pass, mechanism only — cost is NOT measured).** Every handoff
seeds a brand-new session on the target provider (see `docs/hexfy/PHASE-B.md` item 1). A brand-new
session has no cached context, so the seed brief is submitted as a full, uncached prompt at
whatever token cost that provider charges for an uncached prompt of that size. If a router
switches providers frequently to chase available budget, each switch pays this cold-start cost —
potentially enough to erase the budget savings the switch was meant to capture.

**Not yet quantified.** No measurement exists in this codebase of (a) typical seed-brief size in
tokens, (b) uncached-vs-cached price differential per provider, or (c) the breakeven switch
frequency below which pooling still nets positive. This is explicitly listed as unmeasured in
prior investigation and remains so.

**Mitigation already designed (see `docs/hexfy/PHASE-B.md` item 1).** Store resume cursors
per `(thread, provider)` rather than per-thread, so returning to a previously-used provider
re-enters its own warm session instead of re-seeding cold — this bounds the cost to "pay once per
provider per thread," not "pay every time you switch." This mitigates but does not eliminate the
risk; the first visit to each provider, and any router that thrashes between many providers, still
pays full cold-start cost. Quantifying this is a stated Phase B probe, not yet run.

---

## 4. Budgets are honest for two providers and estimated for three; the "tolerance" story for one of them is a forum reply, not policy

**Severity: medium.** A router that treats an estimate as a measurement will make worse decisions
than one that has no number at all, because it will act confident.

**Evidence (repo, verified this pass).**

| adapter         | `thread.token-usage.updated`  | `account.rate-limits.updated` |
| --------------- | ----------------------------- | ----------------------------- |
| ClaudeAdapter   | yes (`ClaudeAdapter.ts:3507`) | yes                           |
| CodexAdapter    | yes (`CodexAdapter.ts:1395`)  | yes                           |
| CursorAdapter   | no                            | no                            |
| GrokAdapter     | no                            | no                            |
| OpenCodeAdapter | no                            | no                            |

`UsageProviderKind` — the type gating which providers `apps/server/src/usage/UsageService.ts`
prices at all — is `Schema.Literals(["claude", "codex"])` at `packages/contracts/src/usage.ts:26`.
Three of five providers are outside even the retrospective-usage type, let alone live budgeting.

**The forum-reply-vs-policy gap (web, verified this pass, about OpenAI specifically — separate
from the Anthropic question in risk 1).** The only evidence found for OpenAI's stance on running
multiple accounts through Codex CLI is a single community forum thread
(community.openai.com/t/can-i-use-2-chatgpt-account-in-codex-simultaneously/1379038) describing
practitioners doing it, not a documented, citable OpenAI policy permitting or restricting it. This
matters for risk 1's broader ToS question and is called out separately here because it specifically
affects whether Codex's tier-1 budget signal (item in the three-tier model,
`docs/hexfy/PHASE-B.md` item 4) can be relied upon at multi-account scale, versus single-account
where it is unambiguous.

**Mitigation (design already recorded — see `docs/hexfy/PHASE-B.md` item 4).** The three-tier
model: tier 1 (provider-reported: Claude, Codex), tier 2 (Hexfy-metered against a user-declared
ceiling: all five), tier 3 (reactive circuit breaker: all five). **The tier must travel with the
number** so a router or UI never renders an estimate as if it were a measurement. Rank on fraction-
of-current-window-consumed (0..1) plus reset instant, never a summed cross-provider points total —
the units are not commensurable across providers.

---

## 5. The build depends on a toolchain that is not in the repo

**Severity: medium.** This is a contributor-onboarding and CI-reproducibility risk, not a runtime
risk, but it affects every future contributor including automated agents working on this fork.

**Evidence (repo, verified this pass).**

- `README.md:99` instructs `curl -fsSL https://vite.plus | bash` to install the `vp` CLI — the
  tool every other script in this repo assumes is already on `PATH` (`vp i`, `vp run dev`,
  `vp test run`, etc., per `AGENTS.md` and root `package.json` scripts). This is a remote script
  piped directly into a shell, executed before any of the repo's own dependency management
  (`pnpm`, lockfiles) can run — there is no vendored copy of `vp` in this repository and no way to
  obtain it through `pnpm install` alone.
  - Note: `package.json` and several sub-packages _do_ declare a `"vite-plus": "catalog:"` entry
    (resolving to version `0.2.2` in `pnpm-workspace.yaml`, itself aliased to
    `npm:@voidzero-dev/vite-plus-core@0.2.2`). That is a workspace-internal dependency used by
    `@effect/vitest` tooling, not the global `vp` command-line binary the setup scripts invoke —
    the binary still requires the curl-pipe-to-shell bootstrap in `README.md:99` before any of
    those package-level entries become reachable.
- The root `package.json` `prepare` script is
  `"node scripts/clean-tsgo-backups.mjs && effect-tsgo patch && vp config --no-agent"` — it runs
  `effect-tsgo patch` unconditionally on install. `pnpm-workspace.yaml` pins
  `effect: 4.0.0-beta.103` — a beta release, not a stable one — and the `prepare` step patches it
  as part of onboarding every contributor, every time dependencies are installed.

**Mitigation.** None currently recorded beyond what upstream already does (the repo's own
`AGENTS.md` treats `vp i` as a given, pointing to the worktree setup script — `t3.json`'s "Setup
Worktree" script runs `vp i` — as the mechanism that's supposed to have already handled this).
Hexfy inherits this as-is under the additive/stay-mergeable posture; changing it would be a
divergence from upstream, which the fork's own posture (decision 1 in the source investigation)
argues against absent a strong reason.

---

## 6. Four peer surfaces multiply every feature

**Severity: medium, ongoing** (not a one-time risk — recurs on every feature Hexfy adds).

**Evidence (repo — this is the project's own stated guidance, not an external claim).**
`AGENTS.md`, "Hit every surface" section: "The most common defect in this repo is a change that
works on the path you tested and is missing everywhere else," listing entry points, clients (web,
desktop, mobile), providers (five adapters), contracts, reverse states, connection modes, and docs
as the dimensions that must each be independently decided for a change to be complete.

**Why this specifically threatens Hexfy.** Every Hexfy feature in `docs/hexfy/PHASE-B.md` —
handoff, budgets, routing, hosted access — is exactly the shape of feature this warning is about:
provider-shaped (needs a decision per one of five adapters, even if the decision is "not supported
here"), and reachable from multiple clients (a CLI-first product still has to decide what web,
desktop, and mobile do when a Hexfy-specific feature fires, even if the answer for some of them is
"nothing yet").

**Mitigation.** No mitigation beyond discipline: treat the "Hit every surface" checklist in
`AGENTS.md` as a literal per-feature checklist for every Phase A/B item, not a one-time read.

---

## 7. Scale and market context

**Severity: low-medium** — a carrying-cost risk, not a blocking one, given the additive posture.

**Evidence.**

- **carried over (LOC table from prior investigation, not recounted in this pass):**
  approximately 690,000 lines of TypeScript/TSX across the monorepo (apps/server ~230k,
  apps/web ~198k, apps/mobile ~83k, apps/desktop ~42k, plus supporting packages), against an
  upstream that continues to move.
- **web, verified this pass:** several comparable coding-agent orchestration projects have shut
  down or been archived within the last year as of 2026-08-17:
  - Terragon shut down in January 2026.
  - Crystal (multi-session Codex/Claude Code orchestrator) was deprecated in February 2026,
    succeeded by a project called Nimbalyst; its GitHub repo (`stravu/crystal`) now carries a
    "Crystal is now Nimbalyst" notice.
  - Vibe Kanban's operating company (Bloop) announced a shutdown on April 10, 2026; the project
    itself continues as community-maintained open source on a fully local architecture.
  - Roo Code (VS Code extension, ~24k GitHub stars) had its extension shut down May 15, 2026, with
    the GitHub repository (`RooCodeInc/Roo-Code`) remaining as a public Apache-2.0 archive and
    ZooCode/Cline listed as migration paths.

  These are current as of this search and could change; treat star counts and exact dates as
  approximate.

**What this risk is, precisely.** Adjacent products in this space have not, in general, survived
as commercial entities over the past year even when their code lived on. A 690k-LOC fork tracking
a live upstream is a real ongoing cost regardless of Hexfy's own commercial fate.

**Mitigation.** The additive, stay-mergeable posture (decision 1 in the source investigation: new
files over edits, `git merge upstream/main` kept available per the Phase 0 git-topology fix in
prior investigation) is the explicit hedge — if Hexfy-specific work stalls, the fork does not
diverge into an unmergeable dead end; it can fall back to tracking upstream directly with minimal
loss.

---

## 8. Prior art: the specific combination Hexfy is building has not shipped elsewhere, as far as found

**Severity: informational** — affects roadmap confidence and differentiation claims, not
correctness.

**Evidence (web, verified this pass — see note on confidence below).**

- **Omnigent** (`github.com/omnigent-ai/omnigent`, Apache-2.0): open-sourced by Databricks on or
  around June 12, 2026, as a "meta-harness" that wraps Claude Code, Codex, Cursor, Pi, OpenCode,
  and other agents in a uniform interface, with cross-agent sessions that follow the user across
  terminal, browser, and phone. This is the closest match found to Hexfy's cross-harness handoff
  goal. It is Databricks-backed and public, not a small alpha project as characterized in prior
  investigation — that characterization ("alpha", "~9k stars") was not independently confirmed in
  this pass and should be treated as approximate at best. The search results describe composition,
  real-time collaboration, and cross-agent sessions as its three headline capabilities; they do
  **not** mention subscription-budget pooling or task-type capability routing as features — but
  their absence from marketing copy is not proof of their absence from the product, so this gap is
  an inference, not a confirmed negative.
- **ccflare** and **CLIProxyAPI** (and adjacent projects such as `ai-cli-proxy-api`, `ccproxy-api`)
  pool subscription-authenticated CLI access by extracting OAuth tokens and re-exposing them
  behind a local OpenAI/Anthropic-compatible API endpoint. This is the pattern risk 1 calls out as
  the one Hexfy's design deliberately avoids (Hexfy does not extract or re-expose credentials
  outside each provider's own CLI process). One correction to prior characterization: CLIProxyAPI
  in particular spans multiple vendors (Claude, Gemini, Codex, Kimi, and others) via this
  extraction approach, so "same-vendor only" describes ccflare's narrower original scope more
  precisely than it describes the category as a whole.
- **claude-code-router** points Claude Code at alternative backends (including routing through a
  CLIProxyAPI sidecar to reach GPT models via a ChatGPT subscription) — task-type/model routing,
  but across API-key-compatible backends reached through a proxy, not across each vendor's own
  native CLI harness the way Hexfy's adapters do.

**What this means for Hexfy's differentiation claim.** As far as this search found, no project
combines all three of: (a) cross-vendor handoff at the harness layer (not just an API-compatible
proxy layer), (b) budget-aware pooling using each vendor's own reported quota, and (c) task-type
capability routing. Omnigent is the nearest neighbor on (a); the proxy projects are the nearest
neighbors on quota access (via a method Hexfy avoids) but not on (a) or (c). This is a snapshot
from one search pass, not an exhaustive prior-art review, and the landscape is moving quickly (see
risk 7) — this section should be re-checked before it is used in any external-facing claim.

**Mitigation.** None needed — this is a positioning note, not a defect. Re-run this check before
making a public "nobody else does this" claim, given how fast this specific space is moving.

---

## Summary table

| #   | risk                                                            | severity        | resolved by                                                                   |
| --- | --------------------------------------------------------------- | --------------- | ----------------------------------------------------------------------------- |
| 1   | Anthropic ToS wall for multiplexed auth                         | high            | authoritative ToS answer (not yet obtained)                                   |
| 2   | "Without losing context" is approximate; payload untyped        | medium-high     | type `OrchestrationThreadActivity.payload` per `kind`                         |
| 3   | Switching to save quota costs quota (unquantified)              | medium          | measure seed cost; per-(thread,provider) resume cursors (designed, not built) |
| 4   | 3/5 providers estimated-only; OpenAI tolerance is a forum reply | medium          | three-tier model with tier traveling with the number                          |
| 5   | Build toolchain (`vp`) not vendored, installs via curl\|bash    | medium          | none recorded; inherited from upstream                                        |
| 6   | Four peer surfaces multiply every feature                       | medium, ongoing | discipline against the `AGENTS.md` checklist                                  |
| 7   | ~690k LOC, moving upstream, thinning market                     | low-medium      | additive/stay-mergeable posture                                               |
| 8   | No confirmed prior art for the full combination                 | informational   | re-check before external claims                                               |
