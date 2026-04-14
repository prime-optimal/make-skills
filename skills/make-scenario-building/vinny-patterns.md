---
name: vinny-scenario-patterns
description: Vinny-specific scenario-building patterns for chaining, routing, and state management.
---

# Vinny patterns — scenario building

## Canonical chain patterns

- **Legacy chain:** `S1 -> S7` (historical intake into monolith)
- **Bridge chain:** `S6/S6B -> S7` (legacy inbound bridge handoff)
- **Current architecture:** inbound bridges and shortcut paths feed the orchestrator, which fans into reusable subscenarios

## Channel routing (Blooio/Telnyx)

- Keep dual-path routing explicit with clear branch filters.
- Route iMessage-capable contacts through Blooio.
- Keep Telnyx path as fallback/alternate channel path.
- Always persist channel decisions to DS state for downstream sender modules.

## Approval queue is required

- Human approval queue in AITable Messages is a core product constraint.
- Draft generation must feed approval records; send modules must consume approval state.
- Do not bypass approval with direct-send shortcut routes.

## DS state + `ifempty()` merge pattern

- Treat Data Store as the conversation/state bus.
- Read DS near route start; write DS at route end.
- Use `ifempty()` merges so partial updates do not wipe valid prior state.
- Keep one clear DS owner per flow segment to avoid contradictory writes.
