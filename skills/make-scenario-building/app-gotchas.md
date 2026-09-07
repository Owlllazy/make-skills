---
name: app-gotchas
description: High-frequency third-party app configuration mistakes — Google Sheets, Gmail, Make AI Tools, and IML date functions. Check these before finalizing a module's `parameters` or `mapper` in `scenario_create` or `scenario_patch`.
---

# Common App Gotchas

High-frequency configuration mistakes that cause silent failures or hard-to-diagnose runtime errors on this
surface specifically — several of these are missed by `scenario_create`/`scenario_patch`'s validation and only
surface on the first `scenario_run`; there is no separate module-configuration pre-check.

### Google Sheets: `valueInputOption` Required for Write Modules — Nothing Catches It Early

`google-sheets:addRow` and `updateRow` always require `"valueInputOption": "USER_ENTERED"` in the module's
`mapper` (not `parameters`). There is no default. `scenario_create`/`scenario_patch` validate and save
successfully with this field missing — the failure only appears at execution time, as
`400: INVALID_ARGUMENT - 'valueInputOption' is required but not specified` on the first `scenario_run`. Set it
explicitly whenever configuring one of these modules, rather than waiting for a run to fail and patching
afterward:

```json
"mapper": {
  "valueInputOption": "USER_ENTERED",
  "values": { "0": "{{1.text}}", "1": "{{1.ts}}" }
}
```

Set `"insertDataOption": "INSERT_ROWS"` alongside it for `addRow` — without it, writes can land on a fixed row
(e.g. every run overwriting row 1) instead of appending a new one each time.

### Google Sheets: Row-Writing Mapper Uses Zero-Based Positional Keys, Not Column Letters

`addRow`'s (and similar row-writing modules') `values` mapper object is keyed `"0"`, `"1"`, `"2"`, ... — one per
column position — never `"A"`, `"B"`, `"C"`. Using column letters doesn't produce a validation error; it silently
writes to the wrong place (in practice, every run can end up targeting the same single row instead of
appending). This is easy to confuse with a **different, correct** convention that applies elsewhere in the same
app: Google Sheets' own internal filters (a module's own row-matching condition, not a Make `filter`) *do* use
uppercase column letters (`"a": "G"`). The two conventions cover different parts of the same app — don't
cross-apply one to the other.

### Google Sheets: Don't Guess a Spreadsheet or Sheet When the Lookup Comes Back Rejected

Resolving `spreadsheetId`/`sheetId` via `module_options_get` is the right path when the field is listed under
`module_spec`'s `dynamicFields`. For Google Sheets specifically, this lookup can come back rejected as "not
listed in `dynamicFields`" even though the field is genuinely account-dependent — a known gap, not a signal that
the value is a plain literal to set directly. When this happens, don't guess a spreadsheet id or assume a tab is
named `Sheet1` because the user said "the first one is fine" — state the assumption before acting on it (the
"state an assumption before acting on it" rule in `make-scenario-reference`), or ask which sheet/tab to use.

### Gmail Is a Different Connection Than Google Sheets/Calendar/Drive

Gmail modules (`google-email:sendAnEmail`, `google-email:TriggerNewEmail`, ...) authenticate with a distinct
connection type from Google Sheets, Calendar, and Drive, even though all four read as "Google" apps.
`module_spec`'s `connection.existing[]` is already scoped correctly per module, so this rarely surfaces as a
tool-usage error directly — but don't tell a user "you're already connected to Google, so Gmail is covered too"
based on a connection seen offered for a different Google app. Check the specific module's own
`connection.existing[]`.

### IML Date Boundaries: No `endOfDay()` / `startOfDay()` Functions

IML has no `endOfDay()`, `startOfDay()`, `beginningOfDay()`, or equivalent boundary function — using one
produces an "Unknown function" error. Build a day boundary from `formatDate` plus a literal time component
instead:

```
Start of day: {{formatDate(now; "YYYY-MM-DD")}}T00:00:00Z
End of day:   {{formatDate(now; "YYYY-MM-DD")}}T23:59:59Z
```

### Make AI Tools (`ai-tools:Ask` and similar): `model` Is Required, No Default

The `model` field on Make AI Toolkit modules has no default — omitting it fails at execution with a 400. When
using Make's own AI Provider connection, use tier slugs (`"small"`, `"medium"`, `"large"`), not a
provider-specific model id like `"gpt-4o-mini"` — those are rejected outright. If the user has no Make AI
Provider connection, use an app-specific AI module instead (an OpenAI/Anthropic/Gemini connection via
`connection_create`), which does accept provider-specific model ids.

## See also

- [Mapping](./mapping.md) — reference syntax these mapper examples build on.
- The polling-trigger starting-position gap and the webhook scheduling-type rule are covered directly in
  [SKILL.md](./SKILL.md) rather than here, since both are about scenario-level structure, not a specific app's
  module quirks.
