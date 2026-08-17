# Hexfy — Objective

> For contributors picking this up cold. If you have never seen the planning conversation that
> produced this document, read this file, then `docs/hexfy/PHASE-A.md`, before touching code.

## What Hexfy is

Hexfy is a fork of T3 Code (`LICENSE`: MIT, "Copyright (c) 2026 T3 Tools Inc.") that adds three
things on top of the existing product, without removing anything:

1. **A Claude Code-style terminal client.** T3 Code today ships web, desktop, and mobile UIs and a
   server-control CLI (`apps/server/src/bin.ts`, the `t3` command). It has no conversational
   terminal client — no way to type a prompt and watch an agent work from a shell. Hexfy adds one.
2. **Honest per-provider budget and per-project cost/leverage reporting.** T3 Code already receives
   live quota and token-usage signals from provider CLIs but discards most of them (see
   `docs/hexfy/PHASE-A.md` Wave 2). Hexfy types, persists, and surfaces that data so a user can see
   what a subscription is actually worth against what the same work would cost on a metered API.
3. **Eventually, one thread that hands off between vendors.** A single conversation that can
   continue on Claude, then Codex, then back, without the user manually starting over and without
   silently losing context. This is the hardest and last-landed of the three.

Every surface — terminal, web, desktop, mobile — is a peer. None is retired to build another.
Hexfy can run local-only or hosted (using T3 Code's existing relay/cloud layer, `infra/relay/`,
once configured with the operator's own Cloudflare and Clerk accounts). Authentication is always
through subscriptions: a user's own vendor account driven through that vendor's own official CLI.
Hexfy never asks for or stores a raw API key in place of that.

## The five claims

These are the product promises Hexfy makes. Each is deliberately narrow — none claims something
the codebase cannot yet back up.

1. **One thread, many harnesses.** A single durable conversation (a T3 Code `thread`, see
   `packages/contracts/src/orchestration.ts`) is not permanently welded to one provider CLI.
2. **Pooled capacity.** A user's total available capacity across the providers they subscribe to is
   visible in one place, not five separate CLIs each with their own opaque quota indicator.
3. **Right harness for the task.** Work can be routed to whichever provider fits it — by capability
   or by remaining budget — instead of the user manually remembering which CLI to open.
4. **Every surface, one brain.** Terminal, web, desktop, and mobile all read and act on the same
   server-side state (events → projector → read model). None of them re-implements its own idea of
   what happened.
5. **Local or hosted, by choice.** Nothing about pooling or handoff requires Hexfy's own cloud.
   Cloud is additive, gated the same way T3 Code's own cloud features already are (build-time
   constants in `apps/server/src/cloud/publicConfig.ts`).

## Non-goals, and why

This section is legally load-bearing. Read it before proposing anything that pools credentials,
talks to a provider's API directly, or automates account behavior across many logins.

### Never a proxy or gateway

Hexfy does not sit between a vendor's servers and the user's own CLI pretending to be a generic
client. Every provider call in T3 Code today runs through that provider's own official CLI or SDK,
spawned as a local subprocess and resolved by binary path — see, for Claude,
`resolveClaudeSdkExecutablePath(claudeSettings.binaryPath, ...)` in
`apps/server/src/provider/Layers/ClaudeAdapter.ts:1666-1668`. Hexfy keeps this. A proxy or gateway
that re-implements a vendor's wire protocol to multiplex traffic is explicitly out of scope, no
matter how it is framed internally.

### Never extracts an OAuth token

Hexfy does not read a provider's session credentials out of its config directory and reuse them
outside that provider's own binary — not to call the vendor's API directly, not to hand to another
tool, not to feed the Anthropic Agent SDK or any other SDK in place of the official CLI. The binary
authenticates itself; Hexfy never touches what it authenticates with.

### One account per vendor per credential — never pooling many accounts of one vendor behind a proxy

Two Claude accounts each driven through their own, separate official Claude Code CLI install (own
config directory, own login, own process) is a supported pattern — T3 Code's provider driver model
already anticipates multiple instances of one driver (`apps/server/src/provider/ProviderDriver.ts`;
`BUILT_IN_DRIVERS` in `apps/server/src/provider/builtInDrivers.ts`), and its own docs describe
running two Codex or two Claude accounts side by side (`docs/user/providers-codex.md`,
`docs/user/providers-claude.md`). What Hexfy does not do is pool many accounts of the same vendor
behind a single proxy identity, or make it look to the vendor like one client is fanning traffic
out across accounts it does not disclose as separate.

### Not a new coding agent

Hexfy does not build its own model-calling agent loop to compete with Claude Code, Codex, Cursor,
Grok, or OpenCode. It orchestrates the official CLIs; it does not replace their reasoning.

### Not a rewrite

Hexfy does not re-architect T3 Code's event-sourced core, its adapter boundary, or its wire
contracts. New capability is layered on by adding new event types, new services, and new files —
see "Decisions already taken" below.

### The compliance context

In early 2026, Anthropic enforced a ban on using Free/Pro/Max plan OAuth tokens with third-party
tools or with the Anthropic Agent SDK, and sent legal demands to at least one open-source project
that had built exactly that: a tool that extracted or reused a Claude Code OAuth token to drive
requests outside the official CLI. The compliant pattern — the one T3 Code already implements and
Hexfy keeps unchanged — is spawning the vendor's own official CLI binary by path and letting that
binary handle its own authentication end to end. Hexfy never sees, extracts, or forwards the token
itself.

Running multiple accounts, each through its own official CLI install with its own config
directory, is a different thing from the banned pattern, and is not what got that project a legal
demand. But two conditions still apply and are worth stating plainly for anyone extending this
work: the accounts must genuinely belong to the user (this is not a shared-account or
account-farming feature), and aggressive automated round-robining across accounts — hammering
through capacity as fast as possible, unattended, for hours — sits close to the language vendors
use in anti-abuse policy about "running Claude Code continuously in the background." Hexfy's
routing should stay need-driven (route because a provider is out of budget or returned a limit
error) rather than throughput-maximizing by design.

## Decisions already taken

| Decision                                                                                                        | Why                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stay mergeable with upstream. Keep the internal `@t3tools` package scope.                                       | Renaming the scope alone would touch roughly 1,400 files (1,397 files / 2,446 hits for the `t3tools` token, per the rebrand inventory). That single change would make every future `git merge upstream/main` a manual patch. Phase 0 rebrand is user-visible identity only: binary name, display name, home directory, domains, app IDs — not the package scope. |
| Keep every existing surface; add the terminal as a peer.                                                        | Web, desktop, and mobile are not deprecated. Purely additive.                                                                                                                                                                                                                                                                                                    |
| All five providers (Claude, Codex, Cursor, Grok, OpenCode) stay in scope.                                       | They are already first-class drivers (`BUILT_IN_DRIVERS`). Only two currently emit usage/quota signals (see PHASE-A.md, Wave 2) — that is a real scoping constraint on the budget feature, not a reason to drop a provider.                                                                                                                                      |
| Product name: "Hexfy".                                                                                          | Fixed spelling for all user-visible copy, binary name, env var prefix, and store/bundle identifiers going forward.                                                                                                                                                                                                                                               |
| Leverage is reported as API-equivalent cost vs. declared subscription cost, and labelled "leverage," not "ROI." | "ROI" implies a return the user realizes as money; a coding subscription's value is capacity, not income. "Leverage" is the honest word for "how much more this subscription is worth than what you're paying for it, if the same work were billed by the token."                                                                                                |
| Account mode (single account vs. multiple accounts of the same vendor) is selectable per provider.              | Not every user runs multiple accounts of every vendor; forcing one model onto all five providers would be dishonest given the coverage gap above.                                                                                                                                                                                                                |
| Mascot: a pixel-art crow in a hat.                                                                              | Fixed brand asset direction for icon/favicon/banner work (PHASE-A.md, slice 0.4).                                                                                                                                                                                                                                                                                |

## What T3 Code is architecturally

Hexfy inherits this shape as-is. New Hexfy capability is additive on top of it, not a replacement
for it.

**Event-sourced core.** Clients send typed WebSocket requests. The server turns them into
_commands_, a pure _decider_ turns commands into persisted _events_, and a _projector_ derives the
read model every client renders. Core files: `apps/server/src/orchestration/decider.ts`,
`apps/server/src/orchestration/projector.ts`, `apps/server/src/orchestration/OrchestrationEngine.ts`.
The durable event union — 30 event types today — is defined at
`packages/contracts/src/orchestration.ts:1343`.

**Provider CLIs run as subprocesses behind per-provider adapters.** The driver SPI
(`apps/server/src/provider/ProviderDriver.ts`) treats drivers as plain values, not Effect Layers, so
multiple instances of one driver are supported by design — the mechanism two-accounts-per-vendor
already relies on. The built-in driver list lives in
`apps/server/src/provider/builtInDrivers.ts` (Codex, Claude, Cursor, Grok, OpenCode); adding a
driver is a new sibling file plus a one-line array edit. The adapter SPI is
`apps/server/src/provider/Services/ProviderAdapter.ts`.

**~57 normalized runtime event types.** Providers speak different native protocols; adapters
translate them into a shared vocabulary defined in `packages/contracts/src/providerRuntime.ts` —
provider-neutral types like `item.*`, `content.delta`, `turn.*`, `turn.plan.updated`,
`turn.diff.updated`, `request.opened`/`request.resolved`, `tool.*`, `task.*` (subagents),
`thread.token-usage.updated`, and `account.rate-limits.updated`. A handful of provider-native
escape hatches also exist (`claude.sdk.message`, `codex.eventmsg`, `acp.jsonrpc`, and similar) for
information that has no normalized shape yet.

**Per-turn hidden git checkpoints.** Each turn ends with a checkpoint — a hidden git ref — so the
app can diff and restore. See `apps/server/src/orchestration/Layers/CheckpointReactor.ts` and the
durable `thread.turn-diff-completed` event
(`packages/contracts/src/orchestration.ts:1086` and `:1481`).

**Typed wire contracts.** Everything that crosses the WebSocket is typed in
`packages/contracts`. Changing a contract there is what keeps server, web, mobile, and desktop in
sync — there is no separate per-client copy of the wire shape to drift.

**Scale, for context.** The codebase is roughly 690,000 lines of TypeScript/TSX across
`apps/server` (611 files, ~230k LOC), `apps/web` (829 files, ~198k LOC), `apps/mobile` (493 files,
~83k LOC), `apps/desktop` (138 files, ~42k LOC), and the shared packages
(`packages/client-runtime`, `packages/contracts`, `packages/shared`, `packages/effect-acp`,
`packages/effect-codex-app-server`). As of this writing the fork carries 2,637 commits of shared
history with upstream `pingdotgg/t3code` — the merge base needed for "stay mergeable" is real, not
aspirational (see PHASE-A.md, slice 0.2).
