---
name: make-scenario-operations
description: Use when running a scenario on demand, switching one on or off, reviewing or debugging past executions, or investigating a webhook that does not seem to be receiving data. Covers scenario_run, scenario_activate, scenario_deactivate, scenario_execution_list(_show), scenario_execution_get(_show), scenario_execution_inspect, scenario_execution_module_get, scenario_trigger_learn, and scenario_trigger_inspect. Not for creating or editing a scenario (see make-scenario-building) or for explaining an existing scenario's structure without running anything (see make-scenario-explore). Read make-scenario-reference first for scopes and the content-remark contract.
metadata:
  version: "0.1.7" # x-release-please-version
---

# Make scenarios — running and debugging

This skill covers everything that happens *after* a scenario exists: running it, switching it on or off, and
the run → inspect → drill-down chain for figuring out why a run failed. `scenario_run`'s behavior depends
entirely on the scenario's trigger kind — always know `trigger.kind` (from `scenario_get`,
`make-scenario-explore`) before calling it.

## Running a scenario: `scenario_run`

What happens depends on `trigger.kind`:

- **on-demand** — runs with the `inputs` object supplied (keyed by each `interface.input` entry's `name`) and
  returns declared outputs.
- **polling / scheduled** — runs once immediately, ahead of its normal schedule ("run once now"). Takes no
  `inputs` — passing any is rejected outright rather than silently dropped, since these triggers read nothing
  from run-time input.
- **webhook** — **nothing runs.** The response instead gives the webhook's URL to send a request to (or, for an
  app-connected trigger like a Slack event, explains that the scenario runs when the event happens in the
  connected app — that URL is not something to hand a user for that kind). Don't report a webhook scenario as
  "run" — no execution was started, and nothing will appear in its history from this call.

The scenario must be active first — an inactive scenario refuses with a pointer to `scenario_activate`, since
new scenarios always start inactive by platform default.

`status` in the response is `"success"`, `"warning"`, `"error"`, or `"pending"` (still running after the ~40s
wait — check back with `scenario_execution_get` using the returned `executionId`, don't assume it failed).
`error`, when present, names the failing `module` in the same `"app:moduleName"` form `scenario_get` uses, so
it can be matched against the scenario's module list directly.

If the call is rejected for declaring inputs the scenario doesn't recognize, the error names the declared
interface so the next attempt can converge in one more call — read it rather than guessing at field names
again. If `inputs` were passed to a scenario that declares no interface, they're silently unused (a Make
platform behavior, not a bug in this tool) — a `content` remark says so; relay that to the user rather than
assuming the values took effect.

## Activating and deactivating: `scenario_activate` / `scenario_deactivate`

Two separate tools, on purpose — treat "turn it on" and "turn it off" as distinct actions, never as one call
with a direction flag to get right. Both treat an already-satisfied request as success (activating an already-active
scenario, or deactivating an already-inactive one) — report it as done, not as an error. An activation refusal
almost always means the scenario's configuration still has an error somewhere; point at `scenario_get`'s
per-module `issues` to find it (Make records no other queryable reason).

## Reviewing history: `scenario_execution_list` → `scenario_execution_get`

`scenario_execution_list` returns at most 25 runs, newest first, each with `errorMessage` already inline for
failed runs — often enough to answer "did it work" and "why not" without a second call. Narrow with
`status`/`from`/`to`, not paging — there's no `limit`/`offset`, and a run started moments ago can lag the list
by a few seconds (a real ingestion delay, not data loss — a `content` remark says so; don't report a very
recent run as missing).

For one specific run's outcome, outputs, and consumption (credits, data transferred — usage-metering units, not
money), use `scenario_execution_get` — cheaper than the debugging tools below because it carries no per-module
breakdown. It's the right choice for "did my run finish" and "what did it return," not for "which module broke."
Use the `_show` twins (`scenario_execution_list_show`, `scenario_execution_get_show`) only when the user
explicitly wants to see a rendered timeline or card; they return identical data.

## Debugging a failed run: `scenario_execution_inspect` → `scenario_execution_module_get`

This is the map-then-drill-down pair, mirroring `scenario_get` → `scenario_module_get`:

1. **`scenario_execution_inspect`** returns the run's overall status/error plus every module that ran, in flow
   order, with how many times each ran and how many of those failed. Make records exactly **one** error per
   run (the one that ended it) — a multi-cycle run with several failures only surfaces the last one here;
   `modules[].errorCycles` is where the rest show up as counts. A **warning** run records no top-level `error`
   at all, so on a warning-status run, `modules[].errors` is the *only* failure signal — don't report "no error
   found" as "nothing went wrong."
2. Take the suspect module's `id` (and, if reported, its failing `cycle` — most runs have exactly one cycle, so
   this is usually 1) and call **`scenario_execution_module_get`** for the actual request/response data that
   module received and produced. Never guess a `cycle` number independently of what `scenario_execution_inspect`
   reported — an unrelated cycle returns unrelated data with no signal that it's the wrong one.
3. Cross-reference with **`scenario_module_get`** for how that module is *currently* configured — but check
   `scenario_get`'s `lastEdit` against the execution's `startedAt` first. If the scenario was edited after the
   run started, the configuration read back is not necessarily what actually ran during the failure — say so
   before attributing the bug to what's configured now. This surface has no way to read the exact blueprint
   version that executed; the timestamp comparison is the only substitute, and nothing enforces it automatically.

A truncated large module payload is marked in place (`…[truncated]`) rather than silently cut — treat a
`…[truncated]` suffix as text, not as valid JSON to parse further.

A module with no downstream consumer — `ReturnData` (ends the scenario), likely also `StopScenario`/
`ThrowError`/`Rollback` — always reports an empty output record from `scenario_execution_module_get`, even when
correctly configured: it has nothing to hand to a next module. Don't read that as a broken mapper; the
scenario's actual declared output lives in `scenario_execution_get`'s `outputs` field, not here.

## Debugging a webhook that "isn't receiving data"

Two tools cover this, and the order matters:

- **`scenario_trigger_learn`** puts the webhook into learning mode: the next request that arrives is captured
  as the detected structure, without running the scenario (safe to use on an active scenario — nothing is
  queued or executed by a learning request). Give the user the returned webhook URL and have them send one real
  request.
- **`scenario_trigger_inspect`** shows what the webhook has actually been receiving: the detected structure,
  recent deliveries with status, and (with a `deliveryId`) one delivery's raw request. Critically, it also
  reports **`queued`** — deliveries stored and waiting, which is what "I sent data and nothing happened" usually
  means: the scenario is inactive, paused, or rate-limited, so incoming requests pile up instead of running.
  A non-zero `queued.count` is the answer to that complaint far more often than a broken mapping — check it
  before assuming the webhook itself is misconfigured.

Payload content may be withheld for confidential or shared webhooks, and bodies over 1MB or binary are replaced
with a placeholder string — a `content` remark says when this happens; don't treat a withheld payload as
"nothing arrived."

## What this skill does not cover

- Creating or editing scenario structure/configuration — `make-scenario-building`.
- Explaining what a scenario does without running or debugging it — `make-scenario-explore`.
