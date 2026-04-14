---
name: document-make-scenario
description: Convert Make blueprint JSON into Obsidian-ready scenario documentation.
---

# Document Make Scenario

Generate a scenario document matching `docs/make/scenarios/*.md` structure.

## Inputs

- Blueprint JSON path or JSON content
- Optional screenshot of Make canvas

## Required output sections

- Frontmatter
- Overview + trigger/status/systems
- Module flow table
- Mermaid diagram
- Connections / downstream effects
- AITable interactions
- Data Store interactions
- Embedded tools / subscenario calls
- Known issues
- Post-mortem (if applicable)
- Related links

## Rules

- Cross-check module IDs/flow from blueprint JSON.
- If screenshot is provided, reconcile visual flow with JSON-derived flow.
- Prefer concise, deterministic wording over narrative drift.
