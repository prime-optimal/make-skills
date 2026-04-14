---
name: make-scenario-building
description: Build and refactor Vinny Make scenarios using repo patterns, scenario docs, and reusable subscenario architecture.
---

# Make Scenario Building (Vinny)

Use this skill when designing or modifying Make scenario architecture in `vinny-urvenue`.

## Inputs

- Goal or behavior change
- Target scenario key(s)
- Optional blueprint JSON path(s)

## Required references

- `subscenarios.md`
- `vinny-patterns.md`
- `../../../../../../docs/make/scenarios/`
- `../../../../../../make/PATTERNS.md`

## Workflow

1. Read current scenario docs first.
2. Apply Vinny architecture patterns (routing, channel split, DS state handling).
3. Prefer reusable subscenario composition over monolith growth.
4. Produce concrete module/route plan with IDs and data contracts.

## Outputs

- Proposed scenario/module layout
- Route and filter logic
- DS + AITable interaction plan
- Subscenario interface contracts (inputs/outputs)
