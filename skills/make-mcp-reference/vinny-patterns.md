---
name: vinny-mcp-reference
description: Vinny Make MCP constants and preferred tool usage.
---

# Vinny patterns — Make MCP reference

## Fixed identifiers

- **Team ID:** `14785`
- **Org ID:** `37575`
- **Zone:** `us1`

## Preferred usage

- Use `@d1dx-make` for blueprint pull/push/validation and module-level checks.
- Use `@make` for broader official app/webhook/connection operations when needed.
- Keep recipe-first invocation via `just d1dx-*` / `just make-*` when possible.

## High-value tools

- `extract_blueprint_components` — summarize scenario structure into components before editing.
- `validate_blueprint_schema` — validate full blueprint payload shape before push.
- `validate_module_configuration` — verify module-level mapper/parameter correctness.
