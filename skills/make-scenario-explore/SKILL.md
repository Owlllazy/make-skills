---
name: make-scenario-explore
description: Use when orienting inside a user's Make account — listing organizations, teams, and scenarios; explaining an existing scenario; checking connection health; or auditing several scenarios for problems. Covers environment_get, scenario_list(_show), scenario_folder_list, scenario_get, and scenario_module_get. Not for creating or editing a scenario (see make-scenario-building) or for running one or debugging a past execution (see make-scenario-operations). Read make-scenario-reference first for the tool naming, scope, and refusal-contract conventions.
metadata:
  version: "0.1.7" # x-release-please-version
---

# Make scenarios — orienting and explaining

This skill covers the read-only path from "what does this account have in it" to "explain what this specific
scenario does" — the entry point of almost every conversation on this surface, and the tool of choice whenever
a non-technical user asks a plain-language question about their automations.

## Always start with `environment_get`

Call it first, with no arguments — it lists every organization and team the connection can reach, including
private spaces, and almost every other tool needs a `teamId` or `organizationId` from here. Its `zone` field is
also the only source of the account's Make zone, needed to build correct `https://<zone>.make.com/...` deep
links back into the Make editor.

If the listing includes a team typed `"private"`, that's the user's private space — map "my personal account,"
"my own team," or similar phrasing onto that id. If the account is organization-bound, the response narrows to
one organization and a remark says how many others exist behind
re-authentication — mention that only if the user seems to expect something that isn't there; otherwise it's
noise.

## Finding a scenario: `scenario_list` / `scenario_list_show` / `scenario_folder_list`

`scenario_list` returns at most 25 scenarios, most-recently-edited first by default (`orderBy: "name"` sorts
A→Z instead), with a composite `status`
(`error ▷ paused ▷ active ▷ inactive` — precedence in that order) and a coarse `trigger` derived only from the
schedule type (`instant`/`scheduled`/`on-demand`; it can't distinguish a webhook from a polling trigger without
reading the blueprint — that finer distinction lives in `scenario_get`). Narrow with its filters
(name search, folder, status) rather than trying to page through more — there is no `limit`/`offset`, and
`hasMore` just says whether narrowing would find more.

Omitting `status` returns every status — active, inactive, paused, error alike — it is not a hidden filter down
to "currently running." Don't assume an unfiltered call already excludes disabled scenarios; for a "show me
everything" style request, that's exactly why no `status` argument is needed.

Use `scenario_list_show` instead only when the user explicitly wants to *see* a list rendered — it returns
identical data and additionally renders an interactive widget. Default to the headless `scenario_list` while
working through a task.

`scenario_folder_list` is worth calling when the user references "the ones in my Sales folder" or similar —
otherwise it's not part of the everyday path.

## Explaining one scenario: `scenario_get`

`scenario_get` is *enough by itself* to explain what a scenario does: every module in flow order, how they're
wired (including router/if-else/error-handler/agent-tool nesting via each module's `parent`), what filters gate
each one, what triggers the scenario and on what schedule, the health of every connection it uses, and its
declared inputs/outputs. It deliberately excludes per-module `parameters`/`mapper` — the configuration layer —
so don't read a configuration-free explanation as incomplete, and don't fan `scenario_module_get` across every
module just because it wasn't included. Call `scenario_module_get` only when a specific value is the actual
question: which spreadsheet a module writes to, what a message says, how a field is calculated.

A few fields worth knowing how to use when explaining a scenario to someone non-technical:

- **`trigger.kind`** is the fine-grained version of `scenario_list`'s coarse `trigger`: `webhook` (runs when
  data hits a URL), `polling` (checks the source app on a schedule), `scheduled` (runs on a clock regardless of
  new data), or `on-demand` (runs only when explicitly asked — the only kind that returns outputs from a direct
  run). This is the fact that decides whether "make it run" means "give me a URL to post to" or "wait for the
  schedule" — get it from here before answering that question, not from the module name.
- **`connections[].status`** — `"ok"`, `"expiring"` (within about a week), or `"missing"` (the connection no
  longer exists or is invalid). `missing` covers both "deleted" and "exists but broken" — both resolve to the
  same fix: reconnect the app in Make. This is the single most likely explanation for "why did my scenario stop
  working," and it's a structural fact `scenario_get` composes for the model — no other tool surfaces it.
- **`isWaitingOnIncompleteExecutions`** — true when the scenario processes items sequentially and is *frozen*
  behind stored failed runs (its DLQ). This is a materially different situation from "the scenario is just
  slow" or "the scenario is broken": it means new data has stopped flowing entirely until the stuck runs are
  resolved in Make (this surface cannot fix or retry them — see the refusal contract in
  `make-scenario-reference`). Don't conflate it with `incompleteExecutions` alone (a non-zero count on a
  non-sequential scenario just means some runs are stored but the scenario keeps working).
- **`status: "error"`** carries no reason — Make records no queryable cause for this state. Don't guess at one;
  if the user wants to know why, the path is `scenario_execution_*` (see `make-scenario-operations`),
  not a fabricated explanation here.
- **`modules[].issues`** — problems Make itself recorded on a module (e.g. "the module is not set up"). Surface
  these directly; they're often the actual answer to "why won't this activate."

Every scenario id used here comes from `scenario_list` (or from `scenario_create`/`scenario_patch`'s own
output, once created) — there's no `teamId` argument on `scenario_get` or `scenario_module_get`, since a
scenario id already resolves globally.

## Account-wide health checks

For "how are my automations doing" or "check everything for problems" style requests, `scenario_list`'s
`status` and `incompleteExecutions` are a cheap first-pass triage — filter by `status: "error"`, or scan for a
non-zero `incompleteExecutions` — but treat that as a lead, not the answer. `status` only turns `error` after
Make deactivates a scenario following *repeated* failures, so a scenario that just started failing, or fails
occasionally without triggering deactivation, still reads `active` with `incompleteExecutions: 0` — invisible
to this filter alone. **Blueprint/structure (`scenario_get`) says nothing about whether recent runs succeeded**
— never conclude "nothing is broken" from a structural read; it can only describe what a scenario *is*, not
whether it's currently working.

For a real answer, call `scenario_execution_list` (narrowed to recent failures, e.g. `status: "error"`) for
every scenario actually in scope — not only the ones `scenario_list` already flagged — then use `scenario_get`
on whichever ones turn up a failure, for the connection-health and issue detail. There's no dedicated audit
tool and no way to raise the 25-row cap on either list — for an account with more scenarios than that, narrow
by folder or status rather than trying to enumerate everything in one pass, and say so if the sweep is
necessarily partial rather than presenting it as complete.

## What this skill does not cover

- Running a scenario, checking on a past execution, or debugging a failure module-by-module —
  `make-scenario-operations`.
- Creating a new scenario or editing an existing one's structure or configuration —
  `make-scenario-building`.
