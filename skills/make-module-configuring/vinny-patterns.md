---
name: vinny-module-patterns
description: Vinny-specific module configuration rules for Claude, AITable, HTTP, and validation recovery.
---

# Vinny patterns — module configuring

## Claude no-connection rule

For `anthropic-claude:simpleTextPrompt`, keep:

```json
"parameters": {}
```

Do not add a connection ID.

## AITable field nesting

For `createRecords` and `updateRecords`, use nested `records[].fields` structure. Flat field mappings cause runtime/API failures.

## AITable lookups and `filterByFormula`

- Prefer `filterByFormula` for dynamic record selection.
- After UI edits, verify formulas were not silently replaced by hardcoded `recordId` values.

## HTTP template for linked-field writes

Use `http:MakeRequest` PATCH with `authType` explicitly set and body shape:

```json
{"records":[{"recordId":"{{id}}","fields":{"Event":["{{event_record_id}}"]}}]}
```

Link field values must be arrays.

## Sticky `isinvalid` clearing

If a bad push leaves scenario `isinvalid: true`, push a fixed blueprint and reactivate the scenario to clear stale invalid state.

## `metadata.designer.samples` stripping

Treat `metadata.designer.samples` as non-functional UI cache. Strip during analysis/summary to reduce noise and token load.
