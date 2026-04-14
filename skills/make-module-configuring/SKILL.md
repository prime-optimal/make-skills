---
name: make-module-configuring
description: Configure Make modules for Vinny scenarios with correct mapper shape, auth, and validation-safe patterns.
---

# Make Module Configuring (Vinny)

Use this skill for module-level mapper/parameter work.

## Required references

- `vinny-patterns.md`
- `../../../../../../docs/make/modules/`
- `../../../../../../make/PATTERNS.md`

## Workflow

1. Find the exact module doc in `docs/make/modules/`.
2. Apply Vinny mapper/auth rules from `vinny-patterns.md`.
3. Validate shape against known Make API constraints.
4. Provide final mapper JSON and any guard/filter requirements.

## Outputs

- Module config snippet
- Required parameters and connection assumptions
- Failure modes and validation checklist
