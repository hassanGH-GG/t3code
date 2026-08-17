# Hexfy — Phase A build sheet

> Locked slices for the work happening now. Read `docs/hexfy/OBJECTIVE.md` first if you have not
> seen the planning conversation that produced this. 26 slices, three waves. File paths below were
> confirmed against the repository at the time this was written (branch
> `claude/agent-orchestration-platform-6ogqhh`) — if a path has moved, trust the grep, not this doc.

## How to read a slice

Each slice states: what it does, the real files it touches, the one-sentence demo that proves it
works, and what it depends on. Write the demo sentence before writing code — see "Definition of
done" at the bottom.

## Wave 1 — Ground

Nothing else can be built, verified, or demoed reliably until this wave lands.

### 0.1 — Reproducible bootstrap

**What.** `vp` (the internal task runner / Vite+ wrapper) is not on `PATH` in a plain checkout, and
every build, test, typecheck, and lint script in `package.json` routes through it (`"build": "vp
run --filter ... build"`, `"typecheck": "vp run -r --concurrency-limit 2 typecheck"`, `"lint": "vp
lint ..."`, `"test": "vp run -r test"`, and `"prepare": "... vp config --no-agent"`). This is a
genuine blocker, confirmed by running `which vp` in a fresh shell and getting nothing — it must be
resolved before any other slice can be verified the way `AGENTS.md` expects (`vp test run`, `vp i`).

**Files.** `package.json` (scripts), `scripts/dev-runner.ts`, whatever setup script wires `vp` onto
`PATH` in a worktree (`AGENTS.md` calls this "the t3.json setup script").

**Demo.** In a fresh clone of the Hexfy fork, `vp i && vp run dev` succeeds with no manual `PATH`
edits.

**Depends on.** Nothing. Do this first.

### 0.2 — Git topology + weekly upstream sync ritual

**What.** Already done. Verified state: `git remote -v` shows `origin`
(`hassanGH-GG/t3code`), `upstream` (`pingdotgg/t3code`), and `hexfy`
(`Valsoft-Ventures/hexfy`); `git rev-list --count HEAD` returns 2,637 with no `.git/shallow`
marker present — the fork carries real, shared history with upstream, not a disconnected copy.
This slice is the ritual on top of that fact: `git fetch upstream && git merge upstream/main`
(or rebase, team's call) on a fixed weekly cadence, so drift never compounds into an unmergeable
fork.

**Files.** None in-repo — this is a process, not a code change. If a recurring reminder is wanted,
it belongs outside the repository (a scheduled task), not as a script committed here.

**Demo.** `git fetch upstream && git merge upstream/main` run today produces a clean merge or an
ordinary, resolvable conflict set — never "unrelated histories."

**Depends on.** Nothing.

### 0.3 — User-visible rename

**What.** Rebrand user-visible identity to "Hexfy": binary name, own state directory, own env var
prefix. This is the mechanism and its call sites — not the full ~272-file/1,299-hit `t3code` token
sweep or the ~1,397-file/2,446-hit `t3tools` package-scope rename, both of which stay deferred per
the "stay mergeable" decision in `OBJECTIVE.md`. Real call sites for the home-directory env var:

- `apps/server/src/serviceLauncher.ts:603` reads `process.env.T3CODE_HOME`; line 605 throws
  `"T3CODE_HOME is required by the T3 Code service launcher."` if it's unset.
- `apps/server/src/cli/config.ts:35` documents `--base-dir` as "equivalent to T3CODE_HOME"; line
  107 is the actual `Config.string("T3CODE_HOME")` read.
- `apps/server/src/cli/pair.ts:264` reads `Config.string("T3CODE_HOME")` to discover the pairing
  target.
- `apps/server/src/cloud/bootService.ts:64` writes `Environment=T3CODE_HOME=...` into the
  generated systemd unit; `BOOT_SERVICE_NAME = "t3code"` at line 28 names the unit itself.

**Files.** The four above, plus `apps/server/package.json` (bin entry `"t3":
"./dist/bin.mjs"`). Do not attempt the full rebrand-inventory sweep in this slice — that is a
separate, much larger effort tracked outside Phase A.

**Demo.** A fresh install runs as `hexfy`, stores state under a Hexfy-named home directory driven
by a Hexfy-named env var, and `T3CODE_HOME` is no longer required to boot.

**Depends on.** 0.1 (need working scripts to verify the rename didn't break the boot path).

### 0.4 — Crow icon/favicon/banner

**What.** Replace app icons, browser favicon, and marketing banner with the pixel-art crow-in-a-hat
mascot, on every surface that ships visual branding.

**Files.** Desktop app icon assets (`apps/desktop`), web favicon (`apps/web`), mobile app icon
(`apps/mobile`), marketing site banner (`apps/marketing`).

**Demo.** The crow-in-a-hat icon appears in the desktop app, the browser tab favicon, the mobile
app icon, and the marketing site banner.

**Depends on.** Nothing.

### 1.1 — Make silent cross-provider switching an explicit error

**What.** Today a thread's provider binding can be silently rewritten through a path that has no
guard at all. `decider.ts`'s `thread.meta.update` case
(`apps/server/src/orchestration/decider.ts:808-853`) accepts `command.modelSelection` and applies
it straight into the `thread.meta-updated` event payload (lines 845-847) with no provider or driver
check whatsoever.

Contrast with the turn-start path in
`apps/server/src/orchestration/Layers/ProviderCommandReactor.ts`, where the real guards against an
unintended switch — the started-thread rejection at `rejectStartedThreadModelChangeIfRequired`
(~line 566), the cross-driver rejection ("Thread '\<id\>' is bound to driver ... and cannot switch
to ...", ~584-589), and the continuation-key compatibility check ("... because their provider
resume state is incompatible.", ~591-600) — are all gated on `requestedModelSelection !==
undefined`, where `requestedModelSelection = options?.modelSelection` (line 483). A bare
`thread.turn.start` command that omits `modelSelection` skips every one of these guards, and the
thread proceeds under whatever `desiredModelSelection = requestedModelSelection ?? thread.modelSelection`
(line 512) resolves to — including a selection that a prior, unguarded `thread.meta.update` quietly
wrote in. Net effect: total provider-side context loss, and nobody is told it happened.

**How the context is actually lost.** `apps/server/src/provider/Layers/ProviderService.ts:568-573`
gates the resume cursor on the persisted binding matching the resolved instance:

```ts
const effectiveResumeCursor =
  input.resumeCursor ??
  (persistedBinding?.providerInstanceId === resolvedInstanceId
    ? persistedBinding.resumeCursor
    : undefined);
```

After an unguarded rewrite these differ, so the cursor resolves to `undefined` and the new driver
starts cold. That is the whole mechanism.

**Fix — and it belongs in the reactor, not the decider.** `decider.ts` imports no
`ProviderRegistry` and is a pure `(command, readModel) -> event` function, so it cannot resolve
`driverKind` or `ProviderContinuationIdentity` (`apps/server/src/provider/ProviderDriver.ts:76-88`)
for an arbitrary `instanceId` — those are owned per live `ProviderInstance`. Putting the check there
would break the "orchestration stays pure, complexity belongs at the adapter boundary" rule in
`AGENTS.md`. The reactor already owns this rejection class and already has `providerService` in
scope.

The change is one condition, not new machinery: stop keying the guards off
`requestedModelSelection !== undefined` and key them off `desiredInstanceId !== currentInstanceId`
instead. Both `desiredModelSelection` (~512) and `desiredInstanceId` (~513) are _always_ resolved,
whether the instance arrived as an explicit command override or from `thread.modelSelection`. Note
`instanceChanged` (~662-664) is gated the same way and feeds the restart decision at ~672-679, so it
needs the same treatment.

**Must not regress.** Same-driver account switching has to keep working. Codex's shared-home
instances share `continuationKey: codex:home:${sharedHomePath}`
(`apps/server/src/provider/Drivers/CodexHomeLayout.ts:55,64`), so for an account switch
`currentInfo.continuationIdentity.continuationKey === desiredInfo...` stays equal and the
continuation guard does not fire. Verify that explicitly rather than assuming it.

**Files.** `apps/server/src/orchestration/Layers/ProviderCommandReactor.ts` — the actual fix
(requestedModelSelection derivation ~483; `rejectStartedThreadModelChangeIfRequired` defined
~437-467 and called ~565-577; cross-driver reject ~579-590; continuation-key reject ~591-600;
`instanceChanged` ~662-664). Read-only context: `apps/server/src/orchestration/decider.ts`
(`thread.meta.update`, ~808-853) and `apps/server/src/provider/Layers/ProviderService.ts:568-573`.
There is also an independent client-side filter, `deriveLockedProvider` in
`apps/web/src/components/ChatView.logic.ts:385-406`, which locks the model picker once the thread has
started and feeds `ModelPickerContent.tsx`'s `matchesLockedProvider` (~182-190) — purely UI-side, so
any non-picker client bypasses it entirely. That is why the server-side fix is the real one.

**Tests.** In `apps/server/src/orchestration/Layers/ProviderCommandReactor.test.ts` (setup
convention at ~94-118), not a new `decider.*.test.ts`, since the fix lands in the reactor. Cases:
rejects a bare `turn.start` after `meta.update` rewrote `modelSelection` to a different driver;
allows a bare `turn.start` after a same-driver same-`continuationKey` rewrite (the account-switch
case); rejects `turn.start` when no live session exists but the persisted binding's instance differs
from `thread.modelSelection`'s instance (the recovery path).

**Demo.** Run the sequence that used to lose your context — `thread.meta.update` to another driver,
then a bare `thread.turn.start` — and get a clear typed error naming both providers instead of a
silent cold start. Then run the account-switch sequence and watch it still succeed.

**Depends on.** Nothing to build, but this is a hard prerequisite for 3.1: an intentional handoff
path cannot be built responsibly on top of an unintentional one that's still open.

### 1.2 — One writer per state directory

**What.** No lock exists on a T3/Hexfy state directory today. Confirmed in
`apps/server/src/cli/config.ts`: in `web` mode with no explicit port, `resolveServerConfig` calls
`findAvailablePort(ServerConfig.DEFAULT_PORT)` (lines 219, 265) instead of failing when the default
port is already taken — a second launch against the same state directory doesn't error, it just
drifts to a different port. Two processes can end up with two live handles on one SQLite state
file with nobody warned. This slice adds an actual lock, keyed on the resolved `baseDir`, so a
second launch against an already-open state directory fails loudly instead of drifting quietly.

**Why WAL does not already protect us.** `apps/server/src/persistence/Layers/Sqlite.ts:33-41` opens
with `busy_timeout = 5000`, `foreign_keys = ON`, `journal_mode = WAL`, and a comment saying CLI and
server deliberately write from separate processes. So SQLite serializes individual statements and the
bytes never corrupt. That is not the hazard. Two full server processes each run their own
event-sourced projections, sequence generation and background reactors _in memory_, each assuming it
is the sole writer — both would commit perfectly valid transactions while their in-memory projections
silently diverge from the file and from each other. Logically corrupt, byte-wise pristine, and far
harder to notice than a crash. WAL also depends on working advisory locks on the host filesystem,
which is a second reason not to lean on it as the safety mechanism.

**Where the lock goes.** As an `Effect.acquireRelease` at the top of `makeServerLayer`'s
`Effect.gen`, immediately after `const config = yield* ServerConfig.ServerConfig`
(`apps/server/src/server.ts:476`) and before `RuntimeDependenciesLive` / `PersistenceLayerLive` are
composed (~354, ~380). By that point `dbPath` and `stateDir` are already absolute — resolved by
`deriveServerPaths` / `ensureServerDirectories` (`apps/server/src/config.ts:99-155`, called from
`cli/config.ts:288`) — and SQLite has not yet been opened.

**Do not put it in `resolveServerConfig`.** That function is also called by one-shot CLI commands
(`cli/project.ts`, `cli/auth.ts`, `cli/connect.ts` via `resolveCliAuthConfig`) which legitimately open
the database briefly alongside a running server. Locking there would break them.

**Mechanism.** `stateDir/server.lock`, created with `open(path, "wx", 0o600)` holding
`{ pid, port, startedAt }`. That exclusive-create primitive already has two precedents in this repo:
`apps/server/src/serviceLauncher.ts:129,185` and `packages/shared/src/relayClient.ts:328-353`
(`acquireInstallLock`, an `wx` loop with mtime staleness for the cloudflared binary). On `EEXIST`,
read the file and apply the same liveness check `cli/pair.ts:234-241` already uses
(`isProcessAlive`, i.e. `kill(pid, 0)`); if the pid is dead, unlink and retry the exclusive create
once. If alive, fail fast — never fall back to port drift.

**Staleness reclaim is mandatory, not optional.** `apps/desktop` auto-restarts the backend child on
unexpected exit (`DesktopBackendManager.ts`, restart loop ~978-1023). A lock orphaned by SIGKILL or
OOM would wedge every subsequent auto-restart, so the dead-pid reclaim path is what keeps desktop
working rather than a nicety.

**Reuse the existing discovery record.** `apps/server/src/serverRuntimeState.ts:11-21` already
persists `{ version, pid, host?, port, origin, devUrl?, startedAt }` to
`stateDir/server-runtime.json`, written after the HTTP server binds (`server.ts:495-526`) and removed
by an `acquireRelease` finalizer on clean shutdown (~519-524) — so a crash leaves it stale, which is
exactly why `pair.ts` pairs the pid check with an HTTP probe of `state.origin`. Slice 4.3 needs the
same pattern, so build it once here.

**Error message.** Name the conflict so the user can act: _"Another Hexfy server is already running
on this state directory (pid X, port Y, started Z). Stop it first, or pass --base-dir to use a
different one."_

**Files.** New `apps/server/src/stateDirLock.ts` (placement mirrors `serverRuntimeState.ts`).
Edits: `apps/server/src/server.ts` (~476, the acquire point) and `apps/server/src/cli/config.ts`
(~259-266, remove the silent `findAvailablePort` drift for web mode; `DEFAULT_PORT = 3773` is in
`apps/server/src/config.ts:17`). Desktop already uses `DEFAULT_PORT` directly with no drift, so it
acquires the lock trivially — confirm WSL/secondary backends use distinct `baseDir`s, which makes
per-directory lock scope correct.

**Tests.** New `apps/server/src/stateDirLock.test.ts`. Copy setup conventions from
`serverRuntimeState.test.ts` (~1-45: `it.effect`, `makeTempDirectoryScoped`, real
`NodeServices.layer`), `serviceLauncher.test.ts` (~1-14: exercising the `wx` create-then-rename
primitive, for `EEXIST` and stale-pid reclaim), and `serverActivation.test.ts` (~1-22:
`Effect.scoped` + `Deferred`, to assert the finalizer removes the file on clean shutdown and leaves
it on a simulated crash).

**First, reproduce it.** Actually run two servers against one state directory and record the failure
mode before writing the fix, so it is aimed at something observed rather than something assumed.

**Demo.** Launch a second server against an already-open state directory; it fails immediately and
names the process holding the lock, instead of silently starting on another port.

**Depends on.** Nothing to build, but this is a hard prerequisite for 4.3: auto-spawning a daemon
is not safe until a double-launch is a loud failure rather than a silent drift.

## Wave 2 — Truthful numbers

Server- and contract-only. Because everything crossing the wire is typed centrally in
`packages/contracts`, this wave's work appears on web, desktop, and mobile together without any
client-specific plumbing beyond a UI to display it.

### 2.1 — Codex meter

**What.** Wire up the Codex quota signal that already arrives and is currently discarded.
`apps/server/src/provider/Layers/CodexAdapter.ts:1395` handles the `account/rateLimits/updated`
app-server notification and forwards it as `{ rateLimits: event.payload ?? {} }` into the
normalized `account.rate-limits.updated` runtime event — untyped. Codex also exposes a pull
request, `account/rateLimits/read`, so Hexfy can poll on demand rather than only reacting to
whatever the last push happened to contain. The real typed shape already exists, generated:
`V2GetAccountRateLimitsResponse` (`packages/effect-codex-app-server/src/_generated/schema.gen.ts:37816`)
carries `primary`/`secondary` windows, `planType`, `limitId`, `limitName`, `rateLimitReachedType`,
`spendControlReached`, and a multi-bucket `rateLimitsByLimitId` map; the per-window shape
`V2GetAccountRateLimitsResponse__RateLimitWindow` (`schema.gen.ts:4191`) is `{ usedPercent: int,
resetsAt?: int64, windowDurationMins?: int64 }`. This slice merges the sparse push with an
on-demand pull so Hexfy always has a current reading.

**Files.** `apps/server/src/provider/Layers/CodexAdapter.ts` (~1395). The generated schema file is
read-only reference — never hand-edit `_generated/` files.

**Demo.** Requesting Codex's current rate-limit state returns real numbers — primary/secondary
window `usedPercent` and `resetsAt` — pulled on demand, not just whatever arrived on the last push.

**Depends on.** Should land together with 2.2 — both currently flow through the same
`Schema.Unknown` payload type (`providerRuntime.ts:703`).

### 2.2 — Claude meter

**What.** Same discard problem on the Claude side. `ClaudeAdapter.ts:3507` handles the Claude Code
SDK's `rate_limit_event` and forwards it as `account.rate-limits.updated`. The bundled CLI's
message shape — probed directly from the installed `@anthropic-ai/claude-code` package — has
`status: "allowed" | "allowed_warning" | "rejected"`, `resetsAt`, `rateLimitType`, `utilization`
(0..1), `unifiedRateLimitFallbackAvailable`, `isUsingOverage`, `surpassedThreshold`. That shape is
confirmed against the bundled CLI's source, not against a live event captured from a real account.
**Before typing this payload as a schema**, probe `<home>/userdata/logs/provider/events.log` for a
real, observed `account.rate-limits.updated` event to confirm what actually arrives over the wire
matches what the CLI source implies. Only then replace the `Schema.Unknown` at
`packages/contracts/src/providerRuntime.ts:703` (`AccountRateLimitsUpdatedPayload = { rateLimits:
Schema.Unknown }`).

**Files.** `apps/server/src/provider/Layers/ClaudeAdapter.ts` (~3507),
`packages/contracts/src/providerRuntime.ts` (~703).

**Demo.** A captured, real `account.rate-limits.updated` event from a live Claude account decodes
cleanly against the new typed schema — no `Schema.Unknown` left in the rate-limits payload.

**Depends on.** Nothing to build, but the events.log probe must happen before the schema is
written — this is a hard ordering constraint, not a suggestion.

### 2.3 — Persist as durable event + projection

**What.** Make the now-typed rate-limit and token-usage data durable and queryable instead of
live-only. Follow the existing usage RPC path exactly, end to end: contract in
`packages/contracts/src/rpc.ts`, wire handling in `apps/server/src/ws.ts`, client-side state in
`packages/client-runtime/src/state/server.ts`, and authorization scope in
`apps/server/src/auth/RpcAuthorization.ts`. `ThreadTokenUsageSnapshot`
(`packages/contracts/src/providerRuntime.ts:309`) is the existing typed shape for token usage and
is the pattern to match for the new rate-limit projection.

**Files.** `packages/contracts/src/rpc.ts`, `apps/server/src/ws.ts`,
`packages/client-runtime/src/state/server.ts`, `apps/server/src/auth/RpcAuthorization.ts`, plus a
new projection (in `apps/server/src/orchestration/projector.ts` or a sibling budget-specific
projector).

**Demo.** After a server restart, a thread's last-known rate-limit and token-usage state is still
readable — it survived because it was persisted as an event, not just held in memory.

**Depends on.** 2.1, 2.2 (needs their typed payloads).

### 2.5 — Circuit breaker classifying limit errors per adapter

**What.** Cursor, Grok, and OpenCode emit neither `thread.token-usage.updated` nor
`account.rate-limits.updated` today — confirmed directly against
`apps/server/src/provider/Layers/{Cursor,Grok,OpenCode}Adapter.ts`. Proactive budget-aware routing
cannot work for these three; the fallback is reactive: classify limit/quota-shaped errors as they
happen, per adapter, and trip a breaker that routes away from that provider for a cooldown window.

**Files.** `apps/server/src/provider/Layers/CursorAdapter.ts`, `GrokAdapter.ts`,
`OpenCodeAdapter.ts` (error classification), plus a new shared circuit-breaker service (new file
under `apps/server/src/routing/` or `apps/server/src/budget/`).

**Demo.** When Cursor returns a rate-limit-shaped error, Hexfy stops routing new work to it for a
cooldown window and states why — even though Cursor never reported its quota in advance.

**Depends on.** Benefits from 2.1-2.3 as a pattern to follow but is technically independent — it
works off error responses, not proactive signals.

### 2.6 — Two Claude + two Codex accounts, each with own meter

**What.** Prove the per-account meter model, not just per-provider. Two Claude accounts (two
separate official CLI installs, two config directories, per the driver-instance model in
`apps/server/src/provider/ProviderDriver.ts` and `apps/server/src/provider/builtInDrivers.ts`) and
two Codex accounts, each reporting its own independent rate-limit state through the pipeline built
in 2.1-2.3.

**Files.** No new production files expected beyond what 2.1-2.3 already touch — this is a
configuration and verification slice, plus a test that opens two instances of a driver and expects
two distinct meter readings.

**Demo.** Two Claude accounts, each signed into its own Claude Code install, show two separate,
correctly-attributed remaining-budget readings in the same Hexfy session.

**Depends on.** 2.1, 2.2, 2.3.

### 5.1 — Join usage records to the thread's project

**What.** Usage records carry a `sessionId` and a provider binding; join that to the thread's
`projectId` so cost and leverage can be reported per project, not just per thread or globally.

**Files.** The budget projection from 2.3, joined against `OrchestrationThread` /
`OrchestrationProject` in `packages/contracts/src/orchestration.ts` and
`apps/server/src/orchestration/projector.ts`.

**Demo.** Selecting a project shows total spend and remaining budget rolled up across every thread
in that project, not just the one currently open.

**Depends on.** 2.3.

### 5.2 — Declared subscription cost per instance + a stated allocation rule

**What.** Let the user declare what each provider instance's subscription actually costs (for
example, "$20/mo Claude Pro," "$200/mo Codex"), and a stated, visible rule for how that flat cost
is allocated across the projects/threads that used it (even split, usage-weighted, or otherwise —
the rule must be shown, never implicit).

**Files.** New contract type in `packages/contracts`, new server-side store, new Settings UI on
every client per the "hit every surface" rule in `AGENTS.md` (web, desktop, mobile).

**Demo.** A user enters "$20/month" for their Claude Pro instance once, and every project that used
Claude shows its share of that cost under the stated allocation rule.

**Depends on.** 5.1 (needs the per-project join to allocate against).

### 5.3 — The leverage ratio

**What.** Compute and display leverage = (API-equivalent cost of the tokens actually used, priced
the same way `apps/server/src/usage/UsageService.ts` and `apps/server/src/usage/usagePricing.ts`
already price retrospective usage from LiteLLM's `model_prices_and_context_window.json`) divided by
(the declared subscription cost allocated to that scope, from 5.2). Labelled "leverage," never
"ROI" — see `OBJECTIVE.md` for why.

**Files.** New computation, likely `apps/server/src/budget/` (new), reusing
`apps/server/src/usage/UsageService.ts` and `apps/server/src/usage/usagePricing.ts` for pricing;
display surfaces on each client.

**Demo.** A project that would have cost $340 in metered API tokens, run against a $20/month
subscription allocation, shows a leverage ratio around 17x — and the label reads "leverage," never
"ROI."

**Depends on.** 5.1, 5.2.

### 6.1 — Advisory ranking only

**What.** A router that ranks providers/instances by fit for the current task (capability +
remaining budget) and shows the ranking to the user, without auto-switching anything. `ModelSelection`
(`packages/contracts/src/orchestration.ts:83`, effective wire shape `ModelSelectionWire` at line
65: `{ instanceId, model, options? }`) is the existing routing key that already comes from the
client; a Hexfy router is a pure function `(intent, budgets, capabilities, pins, history) ->
ModelSelection`. At this slice it only recommends — it never overrides — the selection. Recording
the recommendation as an event gives free auditability of what Hexfy suggested and why.

**Files.** New `apps/server/src/routing/` (new), reading the budget projection (2.3, 5.1-5.3) and
the per-provider capability declaration `ProviderAdapterCapabilities`
(`apps/server/src/provider/Services/ProviderAdapter.ts:28-32`, currently only declares
`sessionModelSwitch`).

**Demo.** Before starting a turn, Hexfy shows "Codex is recommended — Claude Pro is at 91% of its
5-hour window" as a suggestion the user can accept or ignore; nothing switches automatically.

**Depends on.** 2.3, 5.1 at minimum for a budget-aware ranking to mean anything. Also needs 4.9
(Wave 3) to land before it has a real demo: an advisory ranking that ignores a user's explicit
`/mode` pin isn't advisory, it's noise.

## Wave 3 — The prompt

### 4.1 — Attach and mirror read-only

**What.** The terminal client attaches to an existing thread (started from web, desktop, or mobile)
and mirrors its activity read-only — proving "every surface, one brain" before the terminal
originates anything itself. Blueprint (discovery only, do not generalize its in-process path):
`apps/server/src/cli/project.ts`'s `ProjectCommandExecutionMode = "live" | "offline"` (line 47).
Live mode talks HTTP to a running server via `EnvironmentHttpApi` / `HttpApiClient` /
`EnvironmentAuth` (line 411); offline mode opens `SqlitePersistenceLayerLive` +
`OrchestrationLayerLive` and dispatches through `OrchestrationEngine` in-process (line 429). This
is useful as a blueprint for the terminal's attach-or-embed decision structure — but a live,
conversational terminal must talk to a running server over the same wire contracts every other
client uses, not open the database directly mid-conversation the way offline mode does for
one-shot CLI commands.

**Files.** New terminal client package (new), consuming the same contracts as web
(`packages/contracts/src/rpc.ts`, `apps/server/src/ws.ts`).

**Demo.** Running the terminal client against an active thread that was started in the web app
shows the same activity stream, live, read-only.

**Depends on.** Nothing new — rides on existing contracts. Foundation for the rest of Wave 3.

### 4.2 — Type a prompt in a project directory

**What.** From 4.1's attached terminal, originate a turn: type a prompt while `cwd`'d into a
project directory, and have Hexfy create or reuse the matching project/thread.

**Files.** Terminal client (new), reusing the existing `thread.turn.start` command path.

**Demo.** Running the terminal client inside a project directory and typing a prompt starts a real
turn, visible simultaneously in the web app.

**Depends on.** 4.1.

### 4.3 — Auto-spawn the daemon

**What.** If no server is running for the current state directory when the terminal client starts,
spawn one automatically instead of requiring a manual `t3 start` first.

**Files.** Terminal client (new), plus `apps/server/src/serviceLauncher.ts` /
`apps/server/src/cli/server.ts` for the spawn path.

**Demo.** Running the terminal client in a directory with no server yet running starts one
automatically, with no separate command.

**Depends on.** **1.2.** Auto-spawn without a lock risks silently launching a second writer against
the same state directory — exactly the failure mode 1.2 exists to close. Do not build this slice
before 1.2 lands.

### 4.4 — Inline approvals

**What.** Provider tool-use approval requests (`request.opened` / `request.resolved` in the
normalized event vocabulary, `packages/contracts/src/providerRuntime.ts`) surface inline in the
terminal and can be approved or denied from there.

**Files.** Terminal client (new), reading existing `request.opened` / `request.resolved` runtime
events.

**Demo.** A tool-use approval requested mid-turn shows up in the terminal and can be approved with
a keypress — the same request a web user would otherwise have had to approve.

**Depends on.** 4.2.

### 4.5 — Slash commands showing their source

**What.** Providers' own slash commands already arrive over the wire typed:
`ServerProviderSlashCommand` at `packages/contracts/src/server.ts:81` (`{ name, description?,
input?: { hint } }`). Surface these in the terminal's `/` autocomplete alongside Hexfy's own
commands, labelled by source, so the user is never guessing whether a command is Hexfy's or the
active provider's.

**Files.** Terminal client (new), consuming `ServerProviderSlashCommand` from
`packages/contracts/src/server.ts`.

**Demo.** Typing `/` in the terminal lists both Hexfy's own commands and the active provider's
native slash commands, labelled by source.

**Depends on.** 4.2.

### 4.6 — Interrupt (Ctrl-C once) and detach (twice)

**What.** One Ctrl-C interrupts the running turn. A second Ctrl-C detaches the terminal from the
thread without killing the server-side session, so other clients keep working.

**Files.** Terminal client (new), using the existing turn-interrupt command path.

**Demo.** Pressing Ctrl-C once stops the current turn's output; pressing it again detaches the
terminal, and the thread is still alive and controllable from the web app.

**Depends on.** 4.2.

### 4.7 — /usage and /budget

**What.** In-terminal commands surfacing the truthful numbers built in Wave 2: `/usage` for
token/turn usage (`ThreadTokenUsageSnapshot`), `/budget` for remaining quota and leverage.

**Files.** Terminal client (new), reading the durable projection built in **2.3**.

**Demo.** Typing `/budget` in the terminal shows the current thread's provider, remaining quota
window, and leverage ratio — sourced from durable state, not a live-only guess.

**Depends on.** **2.3.** Needs the persisted projection to exist.

### 4.9 — /mode per provider

**What.** An in-terminal command to pin or change the active provider's mode. This is also what
makes 6.1's advisory ranking demonstrable: the ranking must visibly respect a user's explicit
`/mode` pin instead of recommending around it.

**Files.** Terminal client (new); reads/writes through existing per-provider settings/mode
contracts.

**Demo.** Setting `/mode` to pin Claude in the terminal makes Hexfy's advisory router stop
suggesting a switch away from Claude, even while its remaining budget would otherwise justify one.

**Depends on.** **6.1** (the advisory ranking it needs to visibly respect).

### 4.10 — @files and $skills

**What.** In-terminal `@` file-reference and `$` skill-reference autocomplete, matching the parsing
already proven correct in the web composer: `detectComposerTrigger` in
`apps/web/src/composer-logic.ts:225` is a pure function that parses `/`, `$`, and `@` triggers with
cursor ranges. Reuse the same trigger-detection logic (or its algorithm) rather than re-deriving
it.

**Files.** Terminal client (new), reusing/porting logic from `apps/web/src/composer-logic.ts`.

**Demo.** Typing `@` in the terminal prompt offers file completions from the project directory, and
`$` offers the project's skills, matching what the web composer already does.

**Depends on.** 4.2.

### 4.11 — /threads /resume /diff

**What.** In-terminal thread listing, resuming a prior thread, and viewing the current turn's diff
(backed by the checkpoint mechanism).

**Files.** Terminal client (new), reading existing thread-list / resume / checkpoint-diff RPCs.

**Demo.** `/threads` lists a project's threads in the terminal, `/resume <id>` reattaches to one,
and `/diff` shows the current turn's changes against the last checkpoint.

**Depends on.** 4.1, 4.2.

### 3.1 — Preview the handoff brief and send nothing

**What.** The first real step toward cross-provider handoff, deliberately inert. Given a thread
bound to one provider, synthesize the transcript brief that would seed a fresh session on a
different provider — and show it to the user for review. Sends nothing to another provider. This
exists to let a human sanity-check summarization quality before anything acts on it.

Why a fresh session is the only mechanism: Claude resume is opaque provider-owned state
(`{ resume: <session uuid>, resumeSessionAt: <assistant msg uuid> }`, re-entering Claude Code's own
session file by UUID). Codex resume is likewise its own thread id (`resumeCursor: { threadId }`).
Neither exposes a way to inject a foreign transcript. Cross-provider handoff cannot use native
resume; the only mechanism is a fresh session on the target provider, seeded from Hexfy's own event
log at a turn boundary.

**Files.** New `apps/server/src/handoff/` (new), reusing `apps/server/src/textGeneration/` — a
`summarizeForHandoff` operation following the existing pattern of `generateCommitMessage`,
`generatePrContent`, `generateBranchName`, and `generateThreadTitle`
(`apps/server/src/textGeneration/TextGeneration.ts:77-124`), each of which already takes a
`ModelSelection` and has an implementation for all five providers
(`ClaudeTextGeneration.ts`, `CodexTextGeneration.ts`, `CursorTextGeneration.ts`,
`GrokTextGeneration.ts`, `OpenCodeTextGeneration.ts`). Because these operations take a
`ModelSelection`, the summarization work can itself be routed to whichever provider has spare
budget. The brief is built from `OrchestrationThreadActivity`
(`packages/contracts/src/orchestration.ts:341`) via its `runtimeEventToActivities()` translation
(`apps/server/src/orchestration/Layers/ProviderRuntimeIngestion.ts:362`) — note that the durable
activity `payload` field is still `Schema.Unknown`, so this slice's brief is necessarily
summary-grade; faithful (not just summary-grade) handoff depends on typing that payload, which is
not in scope for Phase A.

**Demo.** Requesting a handoff preview on a Claude-bound thread produces a readable brief of what a
fresh Codex session would be seeded with — and confirms nothing was sent to Codex.

**Depends on.** **1.1.** An intentional handoff path should not be built while the unintentional
silent-switch path is still open.

## Key reusable assets

Recorded here so nobody re-invents them mid-wave.

| Asset                                              | Location                                                     | Why it matters                                                                                                                                                                                                               |
| -------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ServerProviderSlashCommand`                       | `packages/contracts/src/server.ts:81`                        | Providers' own slash commands (`name`, `description?`, `input.hint?`) already arrive over the wire, typed. Used by 4.5.                                                                                                      |
| `detectComposerTrigger`                            | `apps/web/src/composer-logic.ts:225`                         | Pure function parsing `/`, `$`, `@` triggers with cursor ranges — already correct for the web composer. Used by 4.10.                                                                                                        |
| `apps/server/src/textGeneration/`                  | `TextGeneration.ts` + one implementation file per provider   | Per-provider one-shot LLM calls, already implemented for all five providers, each taking a `ModelSelection`. The pattern to follow for `summarizeForHandoff` (3.1) and any future `classifyTurnIntent` work.                 |
| `ThreadTokenUsageSnapshot`                         | `packages/contracts/src/providerRuntime.ts:309`              | Existing typed token-usage shape. The pattern to match when typing the rate-limits payload (2.1, 2.2) and building its projection (2.3).                                                                                     |
| `apps/server/src/cli/project.ts` live/offline mode | `ProjectCommandExecutionMode = "live" \| "offline"`, line 47 | A DISCOVERY blueprint only for the terminal's attach-or-embed decision (4.1). Do **not** generalize its in-process (offline) path to conversational turns — a live terminal talks to a running server like any other client. |

## Definition of done

Applies to every slice above, without exception:

1. **The demo sentence is written before starting**, and the finished slice produces real
   input/real output in a real client — not a unit test standing in for a demo.
2. **Merges to trunk in 1-2 days.** If a slice is bigger than that, it is not actually one slice —
   split it before starting, don't discover that halfway through.
3. **The fork still builds and could rebase on upstream today.** Every slice preserves the "stay
   mergeable" posture from `OBJECTIVE.md` — new files over edits to existing ones wherever
   possible.
4. **Backend behavior ships with a focused test that waits on durable events, never sleeps.**
   `RuntimeReceiptBus`'s production layer is an intentional no-op —
   `apps/server/src/orchestration/Layers/RuntimeReceiptBus.ts:4-7`: "`RuntimeReceiptBusLive` is the
   production default and intentionally does not retain or broadcast receipts." Do not gate a test
   on it. Gate on the persisted event instead — for turn-completion timing, that's
   `thread.turn-diff-completed` (`packages/contracts/src/orchestration.ts:1086`, struct at
   `:1481`), or `RuntimeReceiptBusTest`'s in-memory PubSub layer in the same file if the test needs
   to observe an in-flight milestone directly.
5. **Every surface got a decision.** Even "not supported on mobile yet" is a decision — silence is
   not.
6. **Screenshot or GIF on the PR** for anything user-visible.
