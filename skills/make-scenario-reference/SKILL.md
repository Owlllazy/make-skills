---
name: make-scenario-reference
description: Load this first when working with Make tools named environment_get, scenario_*, app_find, module_spec, module_options_get, or connection_*. Covers naming conventions, the all-or-nothing scope model, the data/remark contract, and which capabilities these tools deliberately cannot perform.
metadata:
  version: "0.1.7" # x-release-please-version
---

# Make scenario-management tools — reference

This is Make's scenario-management tool surface. Load this skill before touching any tool on it, and revisit it
whenever a task seems to need a capability not covered by one of the three companion skills
(`make-scenario-explore`, `make-scenario-building`, `make-scenario-operations`) — the
answer is very often "this surface refuses that on purpose."

## Tool naming and UI conventions

Every tool here is `{subject}_{action}` snake_case, subject first: `environment_get`, `scenario_list`,
`scenario_execution_inspect`, `app_find`, `module_spec`, `module_options_get`, `connection_create`.

Two naming rules worth knowing before reading any single tool's behavior:

- **`scenario_` prefix means "operates on one scenario the caller already holds the id of."** Run history is
  `scenario_execution_*`, webhook learning is `scenario_trigger_*` — not `execution_*` or `trigger_*` — so that
  once a model holds a `scenarioId`, a `scenario_*` glob search surfaces everything it can do next. Tools that
  take no scenario id (`app_find`, `module_spec`, `module_options_get`, `connection_*`) keep their own names.
- **A `_show` suffix is a UI twin, never a superset.** `scenario_list_show` returns byte-identical data to
  `scenario_list` — same input, same output — and only additionally renders a widget. Never call the `_show`
  variant expecting more data than the headless one; call it only when the user explicitly asked to *see*
  something. While working through a multi-step task, always use the headless tool.
- **Names and schemas are permanent once shipped.** Clients can snapshot tool metadata and cache schemas
  indefinitely, so nothing here is ever renamed — a breaking change gets a new tool instead. This
  is why the surface reads schema-conservative: no mode flags, no output-shape discriminators, closed
  operation enums treated as "slightly over-provisioned" rather than minimal.

## Scopes: all-or-nothing

The whole surface is gated by one bundle of OAuth scopes. A connection missing even one scope required by *any*
tool on the surface gets rejected with 403 — on `initialize`, on `tools/list`, and on every `tools/call` alike.
There is no partial surface: either every tool works, or none of them do. Practically, this means a 403 or a
missing-scope error is never "this one tool needs re-authorization" — it means the whole connection needs to be
redone. Point the user at reconnecting the app, not at a narrower fix.

## The data contract: read `content`, not just the structured fields

Data rides in `structuredContent`, matched to a declared `outputSchema`. `content` is either empty or a short
remark carrying something the structured data cannot say on its own — a missing scope, a truncated result, a
webhook's URL and auth requirement, "the run is still executing, check back with `scenario_execution_get`."
**Treat every `content` remark as an instruction, not decoration** — several tools (`scenario_run`'s webhook
refusal, `scenario_execution_list`'s ingestion-lag note, `module_options_get`'s empty-vs-blocked list) put their
actual guidance nowhere else. A remark that explains an absence also says to mention it to the user only if
they seem to be missing something — don't volunteer it otherwise.

Output schemas are intentionally non-strict (`additionalProperties` open, enums typed as plain strings even
where a fixed value set exists) so the underlying Make API can grow without invalidating a client that
snapshotted the schema at submission. Don't assume a field is exhaustively enumerated just because its
description names a value set.

## Structure vs. configuration: two read tools, one write tool

Two different questions get two different tools on the read side, but collapse into one on the write side:

| layer | what it means | read | write |
|---|---|---|---|
| structure | modules, wiring, filters, trigger, schedule, declared inputs/outputs — what a scenario *is* | `scenario_get` | `scenario_patch` (`add_module`, `remove_module`, `set_filter`, `set_scheduling`, `set_interface`, `rename`) |
| configuration | one module's parameters and field mapping — how a part is *set up* | `scenario_module_get` | `scenario_patch` (`set_module_config`) |

`scenario_get` is enough to explain what a scenario does — don't fan `scenario_module_get` across every module
just because the structural read didn't include parameters; call it only when a specific value is the question
("which spreadsheet," "what does the message say"). Both layers share one upstream save, so `scenario_patch`
edits either or both in a single call — there is no reason to make two edits where one covers it.

## Don't fan `scenario_get` across every scenario, either

The same restraint applies one level up. For any request touching more than one scenario — "give me an
overview," "what do I have in this account," "check everything" — answer from `scenario_list`'s own fields
first (name, status, folder, trigger kind, `incompleteExecutions`); don't call `scenario_get` for more than
about three scenarios in one reply just to be thorough. Drill into `scenario_get` only for a scenario the user
named specifically, or one `scenario_list`'s own fields already flag as suspect (`status: "error"`, a non-zero
`incompleteExecutions`). Summarize at the list level and offer to look closer at anything specific instead of
fetching every scenario's full structure to answer a question the list already answers.

## The write model: one call, one save, whole scenario or nothing

`scenario_create` and `scenario_patch` are the only two write tools on the surface, and each makes exactly one
upstream write. A scenario's structure, every module's configuration, its filters, schedule, name and declared
interface all commit together or not at all — there is no server-side draft to stage a multi-step edit in, so
**put everything one change needs into a single call**. Splitting an intent across two calls ("wire the module,
then a second call to configure it") either wastes a call (`add_module` already carries `parameters`/`mapper`)
or, worse, risks a scenario left half-edited if the second call never lands.

Both write tools **validate the whole composed blueprint before saving and refuse with every problem at once**
— a refusal is a free dry run: nothing was written, and the full error list came back in one round trip. This
is also why there is no separate "check my blueprint" tool: a refused write already provides that. Read
refusals (an `errors` array) as normal, expected output, not as a broken call — retry with a fixed design, don't
treat the failure as something to work around.

`scenario_patch` additionally refuses on a concurrency conflict: it takes `expectedLastEdit` (the scenario's
`lastEdit` from the caller's own most recent `scenario_get`), and refuses if the scenario changed since then.
On that refusal, re-fetch with `scenario_get` and retry — don't guess at what changed.

## The refusal contract: what this surface cannot author, and how to say so

This surface deliberately omits whole subsystems that the *read* tools still describe, because a scenario can
use a data store or a custom function that someone built in the Make editor. When a write would reference one
of these, the write tool refuses **by name**, pointing at the Make editor — never at a tool that doesn't exist,
and never letting the reference through to fail opaquely at save time or silently at run time. Treat that
refusal the same way: name the entity kind to the user and point at the Make editor, don't attempt a workaround
that pretends this surface can create one.

As of this writing, cannot be authored here (confirm against `TOOLS.md`'s "Deferred, with reasons" section if
this list might have changed):

- **Data stores and custom IML functions** — a scenario using one is fully readable and debuggable
  (`scenario_get` reports the module, `scenario_module_get` its configuration), only *creating* one is out of
  reach.
- **Data structures (UDTs)** — reads work throughout (a webhook's detected structure, a resolved JSON Schema on
  `scenario_run`'s webhook refusal); creating a new one is not available.
- **Incomplete-execution (DLQ) fix-and-retry** — a different operation from replay, not available; Make's own UI
  is the resolution path.
- **Version-pinned reads** — no tool can fetch the exact blueprint version a past run actually executed. When
  explaining or debugging a past run, compare `scenario_get`'s `lastEdit` against the execution's `startedAt`
  (from `scenario_execution_get`/`scenario_execution_inspect`) before attributing a failure to the scenario's
  *current* configuration — if the scenario was edited after the run started, the configuration being read back
  may not be what actually ran. Nothing on the tool surface enforces this comparison; it has to be done
  deliberately.
- **`scenario_publish`** (drafts) and **connection re-authorization** — not available; point at Make's editor.

Do **not** assume this list is exhaustive forever, and do not assume everything not mentioned in a given
walkthrough is unsupported — for example, app-specific instant triggers (a Slack "watch events"-style webhook
trigger, not just a generic custom webhook) *are* supported: `module_spec`'s `webhook` object reports whether
Make can create the hook itself (`autoCreatable: true`) or whether the user has to create it in Make and hand
over its id (`autoCreatable: false`). Check `module_spec` for the specific module in question rather than
assuming instant triggers are out of reach.

## State an assumption before acting on it, not after

When a request carries an ambiguous parameter — a relative date, a place or channel named in prose, a
scenario/team referred to descriptively rather than by id — and a tool would silently accept whatever guess
resolves it, **say what you're resolving it to before the call that acts on it**, not only in the closing
summary. "I'm reading 'the Sales channel' as `C0123ABC` — creating the scenario now" gives the user a chance to
correct a wrong guess before it takes effect; stating it only after the fact does not.

This matters most wherever a wrong guess causes a real side effect — `scenario_create`, `scenario_patch`,
`scenario_run` with `inputs`, `connection_create`. It matters less where a wrong guess is rejected for free and
teaches the correct value back: `module_options_get` returns the actual option list rather than accepting a
guessed value, and `scenario_run` names the declared interface on a bad `inputs` key. Resolve-then-disclose is
fine there. Anywhere else, ask first — don't let the user discover the interpretation only after it already ran.

## Lists are capped, never paged

No list tool here takes `limit`/`offset`. `scenario_list`, `scenario_execution_list`, and `module_options_get`'s
option lists all cap at 25 rows (`hasMore` says when more exist) and expect **narrowing**, not paging, as the
way to see more — a scenario search takes `search`/`folderId`/`status`; execution history takes `from`/`to` or
`status`. Don't try to page through hundreds of rows to find one; ask a narrower question instead.

## Vocabulary: "private space," never "personal team"

When `environment_get`'s team listing includes an entry typed `"private"`, that is a private space — a member's
own single-person workspace, Make's product term for it. When a user says "my personal account," "my personal
team," or "my private team," they mean their private space — use that id wherever a `teamId` is expected, and
use the surface's own term ("private space") in anything said back to the user.

## Which companion skill to use

- **`make-scenario-explore`** — orienting, listing, explaining an existing scenario in plain language,
  account-wide health checks.
- **`make-scenario-building`** — finding apps/modules, resolving connections, creating a new scenario,
  editing an existing one.
- **`make-scenario-operations`** — running a scenario on demand, activating/deactivating it, and
  debugging a run or a misbehaving webhook.
