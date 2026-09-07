---
name: on-demand-structure
description: How an on-demand scenario is actually wired — the required trigger module, how declared inputs and outputs bind to it, and the failure modes when they don't.
---

# On-Demand Scenario Structure

## The trigger module is not optional

An on-demand scenario (`scheduling: {"type": "on-demand"}`) needs a real starter module in the
`root: "main"` position — specifically `scenario-service:StartSubscenario` ("Start scenario";
`module_spec` reports its type as `"starter"`). This is not a style preference: `scenario_create`
accepts a plain action module as root with no warning, but a plain action module has no
platform-level "receive the run's injected data as my own output" behavior — only a genuine
starter/trigger type has that. Put a regular action module in root and the declared
`interface.input` fields have **no module output to bind to at all** — no mapping syntax fixes
that, because there is nothing to map from.

**`app_find` under-serves this search.** A query like `"on-demand trigger"` returns unrelated
third-party apps — `scenario-service` never shows up (confirmed live). Query toward what the module
*does* instead: `"start scenario when called by another source, receive scenario inputs"` reliably
surfaces `scenario-service:StartSubscenario` and its pair `scenario-service:ReturnData` ("Return
output"; `module_spec` reports its type as `"returner"`). Note that `app_find` types both `"other"`,
not `"trigger"` — don't filter them out for lacking that kind.

## The three-part shape

Every on-demand scenario needing declared inputs/outputs has this shape:

```
id 1: scenario-service:StartSubscenario   root: "main"
id 2..N: whatever the scenario actually does, follows: id 1 (then chained)
id N+1: scenario-service:ReturnData        follows: id N
```

- `StartSubscenario` (id 1) is where the run's `data` — the values a caller passes for the declared
  `interface.input` fields — lands, as *its own output bundle* (confirmed: its `module_spec` output
  schema is `{"$ref":"scenario://inputSpec"}`, resolved from the scenario's own declared inputs, not
  a static schema). See [Mapping](./mapping.md) for the reference syntax.
- `ReturnData` (the last module) is where the scenario's `interface.output` values come from. Its
  input schema puts **no constraint at all** on `mapper` (`module_spec` reports
  `"mapper":{"type":"object","properties":{},"required":[]}` — anything goes). Nothing here checks a
  mapper key against the declared output field `name`s, which is exactly why a mismatch doesn't
  error — it silently returns `null` for the unmatched output field. This is the single most common
  way a build "succeeds" (`scenario_run` returns `"status":"success"`) while the outputs a user asked
  for come back empty.

## Interface declaration

`interface: {input: [...], output: [...]}` is a **top-level sibling** of the module list on
`scenario_create` — not nested inside `StartSubscenario` or `ReturnData`, even though those are the
modules that functionally carry the data. Required inputs are only valid when
`scheduling.type` is `"on-demand"` — declaring a required input on any other schedule type is
rejected at validation time.

## Symptoms of getting this wrong

- **Wrong root, plain action module:** the build either fails validation with an unhelpful generic
  `BlueprintValidationError` (no field named), or — worse — succeeds, activates, and every
  `scenario_run` returns `null` for every declared output, because nothing ever received the input
  and nothing ever populated `ReturnData`'s mapper meaningfully.
- **Right root, wrong mapping reference:** same null-output symptom, but the fix is a mapper change,
  not a structural rebuild — see [Mapping](./mapping.md).
- **`ReturnData` present but its mapper doesn't cover every declared output name:** partial or total
  null outputs on an otherwise "successful" run — check every declared `output[].name` has a matching
  `ReturnData` mapper key before calling this done.

## Official Documentation

- [Subscenarios](https://help.make.com/subscenarios)

See also: [Mapping](./mapping.md) for how to reference `StartSubscenario`'s output once it's correctly
in place.
