---
name: make-mcp-reference
description: This skill should be used when the user asks about "Make MCP server", "Make MCP client", "Make MCP tools", "MCP token", "Make OAuth", "scenario as tool", "MCP scopes", "Make API access", "connect Make to Claude", "MCP Client module", "DoAnAction", "CallTool", "mcp-client", "connect external MCP server to Make", "scenario not appearing", "MCP timeout", "MCP connection refused", or discusses configuring, troubleshooting, or understanding Make.com's MCP server or MCP client integration. Provides technical reference for connection methods, scopes, access control, the MCP Client module (CallTool vs DoAnAction), and troubleshooting.
---

# Make MCP Reference

Technical reference for Make.com's MCP integration — both as a **server** (exposing scenarios as tools to AI clients) and as a **client** (connecting to external MCP servers from within scenarios).

---

# MCP Server (Make → AI Clients)

Expose Make scenarios as callable tools for AI clients like Claude Code.

## Connection Methods

### OAuth (Default)

Connect via OAuth consent flow. Select organization and scopes during authentication.

**Endpoint:** `https://mcp.make.com`

**URL variants:**
| Transport | URL |
|-----------|-----|
| Stateless Streamable HTTP (default) | `https://mcp.make.com` |
| Streamable HTTP | `https://mcp.make.com/stream` |
| SSE | `https://mcp.make.com/sse` |

For clients without SSE support, a legacy transport using the Cloudflare `mcp-remote` proxy wrapper is available: `npx -y mcp-remote https://mcp.make.com/sse`.

**Configuration for Claude Code:**
```json
{
  "mcpServers": {
    "make": {
      "type": "http",
      "url": "https://mcp.make.com"
    }
  }
}
```

**Access control:** Restrict to specific organizations during OAuth consent. Teams plan or higher enables team-level restrictions.

### MCP Token

Generate a token in Make profile → API access tab → Add token.

**Endpoint:** `https://<MAKE_ZONE>/mcp/u/<MCP_TOKEN>/stateless`

**URL variants:**
| Transport | URL |
|-----------|-----|
| Stateless Streamable HTTP | `https://<ZONE>/mcp/u/<TOKEN>/stateless` |
| Streamable HTTP | `https://<ZONE>/mcp/u/<TOKEN>/stream` |
| SSE | `https://<ZONE>/mcp/u/<TOKEN>/sse` |
| Header Auth | `https://<ZONE>/mcp/stateless` + `Authorization: Bearer <TOKEN>` |

**Configuration for Claude Code:**
```json
{
  "mcpServers": {
    "make": {
      "type": "http",
      "url": "https://<MAKE_ZONE>/mcp/u/<MCP_TOKEN>"
    }
  }
}
```

Replace `<MAKE_ZONE>` with the organization's hosting zone (e.g., `eu1.make.com`, `eu2.make.com`, `us1.make.com`).

**Security:** Treat MCP tokens as secrets. Never commit them to version control.

## Scopes

### Scenario Run Scopes

Allow AI clients to view and run active, on-demand scenarios.

- **OAuth scope:** "Run your scenarios"
- **Token scope:** `mcp:use`
- **Available on:** All plans

### Management Scopes

Allow AI clients to view and modify account contents (scenarios, connections, webhooks, data stores, teams).

- **Available on:** Paid plans only
- Enable granular control over Make account management

## Configuring Scenarios as MCP Tools

For a scenario to appear as an MCP tool:

1. Set scenario to **active** status
2. Set scheduling to **on-demand**
3. Select the appropriate scope (`mcp:use` for tokens, "Run your scenarios" for OAuth)
4. Configure **scenario inputs** — these become tool parameters
5. Configure **scenario outputs** — these become tool return values
6. Add a detailed **scenario description** — strongly recommended to help AI understand the tool's purpose and improve discoverability

**Input/output best practices:**
- Write clear, descriptive names (AI agents rely on these)
- Add detailed descriptions explaining expected data
- Use specific data types over `Any`
- Keep execution time under timeout limits

## Access Control (Token Auth)

Restrict which scenarios are available via URL query parameters:

**Organization level:**
```
?organizationId=<id>
```

**Team level:**
```
?teamId=<id>
```

**Scenario level (single):**
```
?scenarioId=<id>
```

**Multiple scenarios:**
```
?scenarioId[]=<id1>&scenarioId[]=<id2>
```

Levels are mutually exclusive — cannot combine organization, team, and scenario filters.

## Timeouts

| Tool Type | OAuth | Token (Stateless) | Token (SSE/Stream) |
|-----------|-------|--------------------|--------------------|
| Scenario Run | 25s | 40s | 40s |
| Management | 30s | 60s | 320s |

When a scenario run exceeds the timeout, the response includes an `executionId`. The scenario continues running in Make for up to 40 minutes. Use `executions_get` with that ID to poll for results.

## Advanced Configuration

### Tool Name Length

Customize maximum tool name length with query parameter:
```
?maxToolNameLength=<32-160>
```
Default: 56 characters.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Scenario not appearing as tool | Verify: active status, on-demand scheduling, correct scope |
| Timeout errors | Switch from `https://mcp.make.com` to a zone-specific `https://<MAKE_ZONE>/mcp/<TRANSPORT>` URL for longer timeouts. Alternatively, reduce scenario complexity or use SSE transport |
| Permission denied | Check token scopes and access control parameters |
| Connection refused | Verify zone URL and token validity |
| Stale tool list | Reconnect MCP client to refresh available tools |

---

# MCP Client Module (Beta)

Make can also act as an MCP **client** — connecting to external MCP servers from within scenarios. The `mcp-client` app (Beta) has two modules that share the same connection but serve different purposes.

## Two Modules, One Connection

| Module | Label | AI Layer | Output | Use Case |
|--------|-------|----------|--------|----------|
| `DoAnAction` | Execute an action with AI | Qwen 3 32B via Groq | Markdown text summary | Natural language queries, fuzzy searches |
| `CallTool` | Call a tool | None | Structured JSON (`parsedResult`) | Deterministic pipelines, exact parameters |

**Critical:** The Make UI only exposes `DoAnAction`. The `CallTool` module is only accessible via the `tools_create` API or by editing blueprint JSON directly.

## Connection Setup

1. In Make UI: Connections → Create → search "MCP Client"
2. MCP Server dropdown → "New MCP server"
3. Enter the external MCP server URL and optional API key (sent as Bearer token)
4. Save — Make introspects the server and discovers available tools

## DoAnAction — AI-Wrapped Tool Calling

The `DoAnAction` module wraps an AI agent (Qwen 3 32B via Groq) that interprets a natural language prompt, decides which MCP tools to call, executes them, and returns a summarized response.

**Configuration:**
- **Connection** — the MCP server connection
- **Tools** — multi-select of available tools (restricts which tools the AI can call)
- **Task** — natural language prompt describing what to do
- **Disable system prompt** — toggle (default: false)

**System prompt injected by Make:**
```
You are a helpful assistant that uses tools to complete tasks efficiently.
- Select the most appropriate tool for the next step of the task
- Never use placeholder values when the real value should come from a previous tool call
- If you cannot complete the task, respond with "I am unable to complete this task due to <REASON>"
- Do not ask the user for additional information
```

**Output:**
- `result` — text summary of results (AI-generated, often markdown)
- `tokenUsage` — `{inputTokens, outputTokens}` for the Groq LLM calls
- `systemOutput.toolCalls` — array of tool names called
- `systemOutput.messages` — full LangChain message chain (system, human, AI, tool results)

**Cost:** ~17K Groq tokens per run (varies with tool result size). No Make credits charged for the LLM — only for the module execution.

**When to use:** Guest messages with vague intent, fuzzy event searches, any input where the AI needs to pick between multiple tools or chain tool calls.

## CallTool — Direct Tool Execution

The `CallTool` module calls a specific MCP tool directly with exact parameters. No AI interpretation, no token burn, structured JSON response.

**Not available in the Make UI.** Create CallTool-based tools via the `tools_create` API:

```
tools_create({
  teamId: 14785,
  name: "Venue Lineup (MCP Direct)",
  description: "Calls venue_lineup directly — no AI layer",
  inputs: [
    {name: "venue", type: "text", required: true, description: "Full venue name"},
    {name: "limit", type: "integer", required: false, description: "Max results"}
  ],
  module: {
    module: "mcp-client:CallTool",
    version: 1,
    mapper: {
      toolName: "venue_lineup",
      arguments: {
        venue: "{{var.input.venue}}",
        limit: "{{var.input.limit}}"
      }
    },
    parameters: {account: CONNECTION_ID},
    metadata: {}
  }
})
```

**Output:**
- `result` — raw text (the JSON string returned by the MCP tool)
- `parsedResult` — **already parsed** into a proper JSON object/array
- `type` — always `"text"`

**Run with:** `scenarios_run(scenarioId: TOOL_ID, data: {venue: "XS Nightclub", limit: 5})`

**When to use:** Deterministic data pipelines, event sync, any flow where inputs are structured and the exact tool + parameters are known at design time.

## Discovering Available Tools

Use the `RpcListTools` RPC to introspect an MCP server's available tools and their argument schemas:

```
rpc_execute({
  appName: "mcp-client",
  appVersion: 1,
  rpcName: "RpcListTools",
  data: {account: CONNECTION_ID},
  format: "options_instructions"
})
```

Returns an array of tool definitions with:
- `value` — tool name (e.g. `"search_events"`)
- `label` — display name
- `description` — what the tool does
- `x-nested` — argument schema (`{arguments: {properties: {...}}}`)

## Gotchas

- **DoAnAction returns markdown, not structured data.** The AI summarizes tool results into text. If you need structured JSON downstream (e.g., for an Iterator), use CallTool instead.
- **`add()` nests arrays, doesn't flatten.** When merging multiple CallTool results with `add(add(a; b); c)`, the second+ arrays become single elements instead of being flattened. Use subscenarios (one per data source) instead of merging in a Set Variable module.
- **Connection must be created in UI first.** The MCP server registration (URL + auth) happens in the Make UI. The `credential-requests_create` API can handle the auth step, but the MCP server itself must be registered via the UI.
- **CallTool in blueprints.** When adding `mcp-client:CallTool` directly to a blueprint (not via `tools_create`), the module needs `parameters: {account: CONNECTION_ID}` and `mapper: {toolName: "...", arguments: {...}}`.

---

## Resources

- **`references/transport-details.md`** — Detailed transport comparison, URL construction, and zone list
- **[Make MCP Server docs](https://developers.make.com/mcp-server)** — Official documentation
- **make-scenario-building** skill — Scenario construction: routing, filtering, iterations, aggregations, error handling, blueprint construction
- **make-module-configuring** skill — Module configuration: parameters, connections, mapping, webhooks, data stores
