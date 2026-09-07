---
name: mapping
description: How data flows between modules using the flat module grammar — module ID references, the on-demand/webhook trigger-output special case, array indexing, and update-safety gotchas.
---

# Mapping

## The rule

A module's `mapper` values are IML expressions referencing an **upstream module's output by that
module's own `id`** — the same `id` you assigned it in the flat module list `scenario_create`/
`scenario_patch` take.

**Syntax:** `{{moduleId.fieldName}}`

- `{{1.email}}` — the `email` field from the module with id 1
- `{{3.items[1].name}}` — the first item's `name` in the `items` array from the module with id 3
- `{{5.status}}` — the `status` field from the module with id 5

There is no other reference form. In particular, **there is no `{{input.x}}` variable** — this is
the single most common wrong guess, and it silently resolves to nothing rather than erroring.

## The trigger-output special case (on-demand and webhook alike)

Whatever module holds `root: "main"` receives the run's injected data — a webhook's parsed payload,
or an on-demand scenario's declared `interface.input` values — as **its own output bundle**. It is
referenced exactly like any other module's output, by its own `id`: if `StartSubscenario` is id 1 and
the scenario declares an input named `city`, the reference is `{{1.city}}`, not `{{input.city}}`,
`{{trigger.city}}`, or a bare `{{city}}`. See [On-Demand Structure](./on-demand-structure.md) for why
the root module has to be a real starter type for this to work at all.

## Full bundle reference

To pass an upstream module's **entire output** rather than one field, wrap the id in backticks:
`` {{`1`}} ``. A bare `{{1}}` will not work — IML can't parse a bare numeric token. Use this when a
field wants the whole object (e.g. JSON-stringifying a bundle, or an HTTP request body).

## Discovering what's available to map

Call `module_spec` for every module, left to right, before writing its mapper — the fields it can
map from are whatever upstream modules already declared as output.

Some modules' outputs depend on their own configuration rather than being static (e.g. a Google
Sheets trigger's row fields depend on which sheet is selected and what its header row contains) —
`module_spec` reports these with an unresolved output schema
(`{"allOf":[{"$ref":"…call module_options_get to resolve them…"}]}`), but **`module_options_get`
only resolves dynamic *input* field options** (dropdown-style values keyed by a `dynamicFields[]`
entry) — confirmed live against `google-sheets:watchRows`, whose `dynamicFields` lists only
`parameters/sheetId`, nothing for the output. There is no equivalent on this surface to generic's
`rpc_execute` output-RPC step: **a dynamic module's real output field names can't be pre-resolved
before it runs.** Options in that situation: map by the module's commonly-known fields and verify
with a real `scenario_run`, or build/run once and read the actual bundle back via
`scenario_execution_module_get`, then fix the mapper if a field name guessed wrong.

## Building the mapper object

```json
{
  "city": "{{1.city}}",
  "greeting": "Hello, {{1.name}}!",
  "status": "active",
  "processedAt": "{{formatDate(now; \"YYYY-MM-DD\")}}"
}
```

- **Mapped values** reference upstream output (`{{1.city}}`)
- **Static values** are literal strings/numbers/booleans — valid in `mapper`, though check whether a
  fixed value belongs in `parameters` instead
- **Transformed values** apply IML functions to upstream data (`{{upper(1.name)}}`)

## Gotchas

- **Array indexing is 1-based.** `{{1.items[1]}}` is the first item; index `0` will not work.
- **Field omission on `set_module_config` updates.** `scenario_patch`'s `set_module_config` replaces
  a module's whole `mapper`/`parameters` object — there is no partial-update semantics. When rebuilding
  the object, omit a key entirely if you don't intend to touch it (particularly on update/upsert-style
  action modules); sending `""` for a field writes an actual empty string and can erase existing data
  downstream.
- **Removing a module breaks references to it.** `scenario_patch`'s `remove_module` discards the
  module's config; any `{{<id>.field}}` reference elsewhere pointing at it now dangles. Fix those
  references in the same `scenario_patch` call — a half-broken edit still gets validated as one whole.

## See also

[On-Demand Structure](./on-demand-structure.md) for the trigger-module requirement this reference
syntax depends on.
