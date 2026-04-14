---
name: make-sub-scenario
description: Extract reusable subscenario candidates from complex Make scenarios.
---

# Make Sub-Scenario

Identify and design reusable child scenarios from an existing parent flow.

## Inputs

- Parent scenario blueprint
- Reuse goal (what should become shared)

## Outputs

- Candidate subscenarios
- Proposed input/output interfaces
- Parent rewiring plan

## Rules

- Prefer `Start scenario` + `Return outputs` conventions from `make-scenario-building/subscenarios.md`.
- Keep extraction boundaries aligned to clear data contracts.
- Highlight sync vs async call mode per candidate.
