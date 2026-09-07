---
name: make-scenario-building
description: Use when finding Make apps or modules for a goal, resolving or requesting a connection, creating a scenario, or editing its structure or module configuration. Covers app_find, module_spec, module_options_get, connection_create, connection_get, scenario_create, and scenario_patch. Not for explaining an already-built scenario (see make-scenario-explore) or running/debugging one (see make-scenario-operations). Read make-scenario-reference first for scopes, the write model, and the refusal contract.
metadata:
  version: "0.1.7" # x-release-please-version
---

# Make scenarios — creating and editing

The build chain on this surface is: **`app_find` → `module_spec` (→ `connection_create`/`connection_get`,
`module_options_get` as needed) → `scenario_create`**, and then **`scenario_patch`** for every edit after that.
Everything here writes to the user's Make account — read `make-scenario-reference`'s write-model and
refusal-contract sections before using any of it.

## Never guess module names

`app_find` and `module_spec` exist because hallucinated module names are the most common blueprint-authoring
failure. Call `app_find` with the user's own words describing the goal ("save email attachments," "post to
Slack," "split into two paths") — not a guessed app name — and use the exact `"app:moduleName"` strings it
returns. Never invent one, and never proceed to `scenario_create`/`scenario_patch` with a module name that
didn't come from `app_find` or an existing scenario's own `scenario_get` output.

`app_find` also returns the flow-control modules a scenario needs beyond straight-line apps: `builtin:BasicRouter`
(every item goes down every arm), `builtin:BasicIfElse` (each item takes the first matching arm), `builtin:BasicMerge`
(where if-else arms rejoin — place it immediately after the if-else), `builtin:BasicAggregator`,
`builtin:BasicFeeder` (iterator), and the generic webhook trigger `gateway:CustomWebHook`. These are typed
`flow_control` in `app_find`'s results, not tied to any one app, and are needed for anything past a linear
chain of modules.

**Filter never shows up in that list — it isn't a module.** It's a gate on any module's own `filter` field
(set at creation via this schema below, or afterward via `scenario_patch`'s `set_filter`), not a step added to
the flow. This matters because it's easy to reach for `BasicIfElse`/`BasicRouter` by habit once "conditional"
is in the request, when the actual requirement is a plain gate:

- **A single continue-or-stop condition with no alternate output for the non-matching case** ("only process if
  status is new," "skip unless X") — use a `filter` on the module, not `BasicIfElse`/`BasicRouter`. A filter
  makes the run genuinely incomplete/skipped when the condition fails; `BasicIfElse`/`BasicRouter` always
  execute one arm or another and always complete, so they cannot produce that outcome no matter how the arms
  are configured — including an "empty" arm meant to simulate skipping.
- **The non-matching case also needs its own distinct output or action** — that's a real branch, not a gate.
  Use `BasicIfElse` (+ `BasicMerge` to rejoin) when exactly one of several mutually exclusive arms should run;
  use `BasicRouter` when more than one arm can independently fire with no convergence.

## When an app offers more than one trigger, say why you picked one

`app_find` regularly returns more than one plausible trigger for the same goal — a scoped polling trigger (e.g.
Slack's "watch direct messages") alongside a broad instant trigger (e.g. Slack's generic "watch events"). Name
the tradeoff as part of *proposing* the scenario, not only if the user later asks "why not a webhook": which one
is instant vs. polling, how narrowly each is scoped to the request, and whether the broad option would need an
added filter to match what the narrower trigger already does for free. Landing on the right choice silently
still leaves the user unable to tell whether it was deliberate or accidental.

## Before authoring, always call `module_spec`

`module_spec` takes up to 8 `"app:moduleName"` strings at once — spec the whole planned scenario in one call,
not one call per module. For each module it returns:

- The field schema (`spec`) — parameter and mapper field names, types, constraints. Field names must be copied
  verbatim into `scenario_create`/`scenario_patch` — there's no translation step, and a reformatted name is a
  chance to get it wrong.
- **`connection`**, when the module authenticates — critically, `connection.existing[]` lists the team's
  connections that *already* satisfy this requirement, with usable ids. Check this before assuming a new
  connection is needed; reuse an existing id as the module's connection parameter instead of starting an OAuth
  flow the user doesn't need to repeat.
- **`dynamicFields`** — fields whose valid values come from the connected account (a spreadsheet id, a channel
  id) rather than something to type or guess. **This is the single most common friction point on this
  surface**: these values are almost always opaque ids that cannot be inferred from what the user said in
  plain language ("the Sales channel" is not `C0123ABC`). Never guess one — resolve it with
  `module_options_get` first. If a field isn't listed under `dynamicFields`, it's *usually* a plain literal or
  mapped expression as `spec` describes, and can be set directly — but a handful of modules have a known gap
  where a genuinely account-dependent field is missing from `dynamicFields` anyway (see
  [Common App Gotchas](./app-gotchas.md), e.g. Google Sheets' `spreadsheetId`/`sheetId`). Don't treat every
  omission as license to guess a value for those; state the assumption or ask instead.
- **`webhook`**, for instant triggers — `autoCreatable` (whether `scenario_create` can make the hook itself, or
  the user has to create it in Make and hand over its id) and `payloadShape` (`"known"` — the app defines the
  fields up front, mappable immediately; `"learned"` — nothing can be mapped until a real request has arrived,
  which forces the gradual create-then-learn path below).

Call `module_spec` on the chosen modules *before* committing to a plan, even during the `app_find` search turn
— `app_find` doesn't say what a module needs to run, and a plan proposed before checking will regularly
discover a missing auth step or an unresolvable field one turn too late.

## Resolving a dynamic field: `module_options_get`

One field per call, by design — this isn't a batchable "resolve everything" tool, and there's no array input.
Required: `organizationId`/`teamId` (from `environment_get`), `module`, `field` (copied verbatim from
`module_spec`'s `dynamicFields`), and `connectionId` (from
`module_spec`'s `connection.existing`; if that list is empty, the fix is `connection_create` first, not a call
here). When a field's options depend on another field already being set (e.g. picking a sheet inside a
spreadsheet), pass the already-chosen values as `context`, keyed by the same field paths `module_spec` listed
under `dependsOn` — set those fields first.

Returns at most 25 options (`hasMore` says if more exist; narrow with `search`). **Set the field to an option's
`value`, never its `label`** — the label is what the user sees, the value is what Make needs. An *empty*
`options` array is not automatically "the account has nothing": if `dependsOn` comes back populated instead,
a prerequisite field needs setting first — set it and retry, don't report the account as empty.

## Getting a connection: `connection_create` / `connection_get`

Credentials are never entered in the conversation. `connection_create` takes a `teamId` and a list of
`{appName, moduleNames?}` (one request can cover several apps in one authorization link — don't create N
requests for N apps when one link would do) and returns a `url` the user has to open themselves; nothing is
connected until they finish there. Poll progress with `connection_get` only when the user says they've finished
authorizing — there's no push signal, and a human browser flow takes minutes, so don't tight-poll it. Once a
credential's `state` is `authorized`, its `connectionId` is what a module's connection parameter needs.

## Creating a scenario: `scenario_create`

Modules are described as a **flat list**, the same shape `scenario_get` reports back — never author a nested
blueprint object. Each module gets a caller-chosen `id` (kept as sent — other modules reference it via
`follows`/`parent`, and mapper values reference its output as `"{{<id>.field}}"` — see [Mapping](./mapping.md)
for the full reference syntax). Exactly one module carries
`root: "main"` (the trigger); every other module carries either `follows` (runs next in the same flow) or
`parent` (starts a nested flow — a router arm, an if-else arm, an error handler, or an agent tool, with
`kind`/`index`/`conditions`/`mergesTo` as appropriate). `scheduling` (required) and `interface` (optional) are
top-level siblings, not nested inside a module — `scheduling.type` is dictated by the trigger chosen
(`"immediately"` for a webhook, `"indefinitely"` with an `interval` for polling, `"on-demand"` for
run-on-request, or a clock schedule). Using `"indefinitely"` for a webhook-triggered scenario fails activation
with "Invalid interval" — a recognizable symptom of scheduling type mismatched to trigger kind, not a sign the
webhook itself is misconfigured. `interface` only makes sense — and only accepts required inputs — on
an on-demand scenario.

The whole design is validated before anything is created, and refused with every problem at once if it doesn't
pass — treat a populated `errors` array as an expected outcome, not a broken call: fix everything listed and
resubmit, since nothing was written. `autoActivate: true` switches the scenario on immediately after a
successful create; leave it off to let the user review first, and note its one real hazard: a freshly created
webhook scenario maps against field names nobody has verified yet (see the learning flow below), so activating
immediately means live traffic flowing into an unverified mapping.

**When a trigger's webhook has `payloadShape: "learned"`** (from `module_spec`), don't try to build the whole
scenario in one call — the field names to map don't exist yet:

1. `scenario_create` with the trigger module alone.
2. Have the user send one real request to the returned webhook URL (or use `scenario_trigger_learn` first if
   nothing has been sent yet — see `make-scenario-operations`).
3. Read the detected fields back (`scenario_trigger_inspect`).
4. Add the rest of the modules with `scenario_patch`, now that real field names are known.

This is one of exactly two points on this surface where a human has to act mid-build — the other is the OAuth
consent flow above. Everywhere else, build the whole scenario in one call; there's no server-side draft to
stage a multi-call edit in, so a scenario built in fragments is a live, runnable (if incomplete) scenario at
every intermediate step.

A polling trigger also has a starting point ("all," "from now on," "since a date") that this surface currently
has no field for — a scenario created here always takes the platform default, which on a busy source can mean
an unrequested backfill once activated. **Ask the user's replay preference before calling `scenario_create` for
any polling trigger** — especially when they've said something like "new items only" — rather than waiting to
see whether the created scenario happens to report a lint about it. If a lint appears anyway, treat it as
confirmation to raise the question, not as the first time it's raised, and don't activate until the user has
answered.

**Building an on-demand scenario (a declared `interface` with no other trigger) has its own required shape** —
the root module must be `scenario-service:StartSubscenario`, and the declared inputs are that module's own
output bundle, referenced like any other module's output (`{{<id>.field}}`, never `{{input.x}}`). See
[On-Demand Structure](./on-demand-structure.md) and [Mapping](./mapping.md) before building one; getting either
part wrong produces a scenario that "runs successfully" with silently null outputs, not an error.

## Editing a scenario: `scenario_patch`

The only tool that edits an existing scenario, and — like `scenario_create` — it makes exactly one upstream
write covering everything named in the call. Pass `expectedLastEdit` (the scenario's `lastEdit` from the most
recent `scenario_get`); a stale value means someone else edited it since, and the call is refused — re-fetch
and retry rather than forcing it. Seven operations, applied in order then validated and saved as one change:

- **`add_module`** — same shape as `scenario_create`'s module entries, including `position` (`{after: moduleId}`
  to insert into an existing flow, or `{parentModuleId, kind, index, ...}` to start/extend a nested flow).
- **`remove_module`** — takes the module out; whatever ran after it now runs directly after what preceded it.
  Its configuration is discarded, and any `{{<id>.field}}` reference to it elsewhere breaks — fix those
  references in the *same* call, since a half-broken edit still gets validated as one whole.
- **`set_module_config`** — **replaces**, never merges, a module's `parameters`/`mapper`. Read the module first
  with `scenario_module_get`, apply the changes to the full object, and send the complete result back —
  omitting the `parameters` property leaves it untouched, but sending `{}` means "no parameters." This is the
  one place a whole-object write is the contract; there is no partial-update semantics to use instead.
- **`set_filter`** — replaces a module's gate; send `{}` to remove it entirely. Filters are structure, not
  configuration — `set_module_config` never touches them, matching the read-side split
  (`scenario_get` reports filters, `scenario_module_get` doesn't).
- **`set_scheduling`**, **`set_interface`**, **`rename`** — scenario-level declarations, each replacing the
  side given (an `interface` operation replaces `input` or `output` independently — omit one to leave it
  alone, send an empty list to remove it).

A batch of several operations is free in one call (they're applied in memory and validated once before saving)
— reconfiguring three modules is three operations in one call, not three calls. One `hook` (for adding/changing
an app-trigger's webhook) is allowed per call: it always creates a new webhook and points the trigger at it,
cleaning up the one used before where Make allows it.

**An operation can't target a module `add_module` is adding in the same call** — the id a new module gets is
assigned server-side and returned in `applied[].moduleId` after the save, not known before it. Wiring a filter
or a follow-on module onto a module that doesn't exist yet is two calls: `add_module` first, then a second
`scenario_patch` (with a fresh `expectedLastEdit`) once its real id is known.

Read the response's `errors`/`warnings` the same way as `scenario_create`'s: a populated `errors` array means
nothing was saved — fix and resubmit. `warnings` covers things that didn't block the save, including
pre-existing problems the call didn't touch.

## The refusal contract, in a build context

If a planned edit would reference a data store, a custom IML function, or another entity this surface can't
author (see `make-scenario-reference`), both write tools refuse by name before saving anything, pointing at
the Make editor. Don't try to work around that refusal by omitting the reference and hoping it resolves at run
time — relay it to the user as a hard boundary of this surface, not a bug to route around.

## Common app gotchas

Before finalizing a module's `parameters`/`mapper`, check [Common App Gotchas](./app-gotchas.md) — high-frequency
mistakes (Google Sheets, Gmail, Make AI Tools, IML date functions) that pass `scenario_create`/`scenario_patch`
validation cleanly and only surface as a runtime failure or a silent wrong-data write.

## What this skill does not cover

- Explaining an already-built scenario's structure — `make-scenario-explore`.
- Running the scenario, checking on executions, or debugging a failure — `make-scenario-operations`.
