---
name: subscenarios
description: Composing scenarios using parent/child subscenario calls for modularity and reuse.
---

# Subscenarios

## What It Is

Subscenarios allow you to break complex automations into smaller, reusable scenarios. A parent scenario triggers one or more subscenarios via the `Scenarios > Call a scenario` module. Subscenarios can return data back to the parent or run independently.

## When to Use It

- A scenario is too large or complex — split it into focused subscenarios for maintainability.
- The same logic is needed in multiple scenarios — build it once as a subscenario and call it from all parents.
- You want to expose automation logic as a tool for AI agents or MCP clients.
- You want to reduce credit usage — the Scenarios app doesn't consume credits for calling subscenarios.

## How It Works in Make

### Calling Modes

| Mode | Behavior |
|---|---|
| **Synchronous** | Parent calls subscenario, pauses, waits for outputs, then resumes. Use when the parent needs results. |
| **Asynchronous** | Parent calls subscenario and continues immediately without waiting. Use for fire-and-forget tasks. |

### Key Modules

- **`Scenarios > Call a scenario`** — In the parent scenario. Passes inputs and (in sync mode) receives outputs.
- **`Scenarios > Start scenario`** — The trigger module in the subscenario. Receives inputs from the parent.
- **`Scenarios > Return outputs`** — In the subscenario. Sends results back to the parent (sync mode only).

### Inputs and Outputs

You define a structured set of **scenario inputs** (data the parent passes in) and **scenario outputs** (data the subscenario returns). This gives a clear contract between parent and child.

### Best Practice: Start Scenario Trigger + SetVariable Buffer

**Always use `Start scenario` as the trigger** for subscenarios (not a webhook or polling trigger). This gives better debugging visibility in the parent — the `Call a scenario` module in the parent's execution log shows the input/output data flowing through.

**Place a `Set Variable(s)` module as module 2** immediately after the trigger. Re-assign all inputs to variables:

```
Start scenario (phone, message) → Set Variables (phone = {{var.input.phone}}, message = {{var.input.message}}) → ... rest of flow
```

This pattern provides two benefits:
1. **Trigger decoupling.** If the trigger is swapped later (e.g., webhook → Start Scenario, or vice versa), no downstream modules break — they all reference the SetVariable output, not the trigger.
2. **Fast debugging.** Temporarily set the trigger on the SetVariable module and click "Run" to start from there with hardcoded test values. No need to fire a webhook or call from a parent scenario. This saves significant time during development.

## Flowchart Notation

```
Trigger: Webhook → Process Order → Scenarios - Call a scenario [Check Inventory] (sync) → If-Else
  ├─ If (in_stock = true): Fulfill Order
  └─ Else: Notify Backorder
```

Async:
```
Trigger: Webhook → Process Order → Scenarios - Call a scenario [Send Notifications] (async) → Continue Processing
```

## Example

A parent scenario that handles incoming orders and delegates inventory checking to a subscenario:

```
Shopify - Watch Orders → Scenarios - Call a scenario [Inventory Check] (sync)
  → If-Else
    ├─ If (available = true): Shopify - Create Fulfillment
    └─ Else: Slack - Notify #backorders
```

The "Inventory Check" subscenario:
```
Scenarios - Start scenario [inputs: product_id, quantity] → Database - Query Stock → Scenarios - Return outputs [available: true/false, stock_count]
```

## Setting the Interface (Inputs & Outputs)

The scenario interface is what controls which fields are available in the `Start scenario` trigger (inputs) and the `Return outputs` module (outputs). **This is the most common source of confusion:** if the `Return outputs` module shows no fields or wrong fields, the scenario interface has not been set.

### Via MCP / API

Use `scenarios_set-interface` to define inputs and outputs programmatically:

```json
{
  "scenarioId": 12345,
  "interface": {
    "input": [
      {"name": "message", "type": "text", "required": true, "help": "Raw inbound message"},
      {"name": "phone", "type": "text", "required": true, "help": "Phone in E.164 format"}
    ],
    "output": [
      {"name": "fname", "type": "text", "required": false, "help": "Extracted first name"},
      {"name": "priority", "type": "text", "required": true, "help": "normal / high / urgent"}
    ]
  }
}
```

### CRITICAL — Both sides need interfaces

Setting the interface is required on **both** the subscenario and the calling (parent) scenario:

1. **Subscenario interface** — Defines what `Return outputs` can send back. Without this, the `Return outputs` module has no fields to map.
2. **Parent scenario interface** — Defines what the parent's own `Return outputs` module can surface. If the parent is itself a subscenario (or needs to pass data through), its output interface must include the fields from the child.

The `Call a scenario` module reads the **child's** output interface to know what fields it receives. The parent's `Return outputs` module reads the **parent's** output interface to know what fields it can emit. If either is missing, the corresponding module shows no mappable fields.

### Via the Make UI

Go to the scenario editor → click the three-dot menu (⋯) next to the scenario name → **Scenario Inputs/Outputs**. Define the fields there. This is the same as calling `scenarios_set-interface` via the API.

## Gotchas

- **Interface determines ReturnData fields.** The `Return outputs` module only exposes fields defined in the scenario's interface. If you add a `Return outputs` module but see no fields, you haven't set the output interface. This applies to both the subscenario AND the calling scenario. See "Setting the Interface" above.
- **`Return outputs` terminates the ENTIRE scenario.** When a `Return outputs` module executes, it kills the whole scenario execution — including any Router routes that haven't fired yet. If the calling scenario has a Router with multiple routes after the `Call a scenario` module, and one route hits `Return outputs`, the other routes will never run. **Workaround:** Use a Data Store as a shared state bus instead of `Return outputs`. Subscenarios write results to the DS (keyed by a shared identifier like phone number), and the calling scenario reads from DS after the call. Each route reads DS at the top and writes back at the bottom, so data flows through progressively without terminating execution.
- **Same team only.** You can only call scenarios within your team. For cross-team or cross-organization calls, use `Make > Run a scenario` or `Webhooks > Custom webhook` instead.
- **Nesting depth.** A subscenario can itself call other subscenarios (acting as a parent). Be mindful of execution depth and timeouts.
- **Async has no outputs.** In async mode, the parent doesn't receive any data back. If you need results, use sync mode.
- **Credits.** Scenarios called via the Scenarios app don't consume credits. This is cheaper than the Webhooks-based approach.
- **Subscenario must be active and on-demand.** The subscenario must be active and scheduled "on demand" to be callable. If it shows an "Inactive" label, it must be previewed and activated before calling.
- **Accurate input/output structure.** The official docs emphasize that you must "define the structure of scenario inputs or outputs accurately to ensure data passes without errors."

## Official Documentation

- [Subscenarios](https://help.make.com/subscenarios)

See also: [Routing](./routing.md) for splitting flow within a scenario, [AI Agents](./ai-agents.md) for using subscenarios as agent tools.
