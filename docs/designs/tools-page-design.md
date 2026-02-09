# UI: MCP Tools Management Page

## Summary

Add a **Tools** page to the agentgateway admin UI that provides visual control over which MCP tools are exposed to connected clients. This surfaces the existing `mcpAuthorization` CEL policy system through a toggle-based interface, removing the need to hand-edit YAML config for common tool filtering use cases.

## Problem

When agentgateway multiplexes many MCP backends (8+ servers, 50+ tools), all tools are exposed to every connected client by default. This causes:

- **Tool overload** for AI clients that perform worse with large tool inventories
- **Unintended access** to dangerous tools (e.g., `restart_ha`, `retry_with_browser_use_agent`)
- **No visibility** into what's currently exposed without reading the config file
- **Manual YAML editing** required to change `mcpAuthorization` rules, even though file-watching hot-reload exists

The `mcpAuthorization` policy with CEL expressions already supports per-tool filtering via `mcp.tool.name` and `mcp.tool.target`, but there's no UI to manage it.

## Proposed Solution

A new `/tools` page in the admin UI (between Policies and Playground in the sidebar) that:

1. **Lists all MCP tools** grouped by their target server
2. **Shows enable/disable toggles** at both the target level and individual tool level
3. **Generates `mcpAuthorization` rules** from toggle state and saves via existing `POST /config`
4. **Hot-reloads automatically** through the existing file-watcher (250ms debounce)

### Interactive Prototype

See [`tools-page-prototype.html`](./tools-page-prototype.html) for a clickable mockup using the agentgateway UI style (dark theme, Radix-like components). Open it in a browser to interact with the toggles, search, and filtering.

## Design

### Frontend

**New files:**
```
ui/src/app/tools/page.tsx           # Page shell
ui/src/components/tools-config.tsx   # Main tools management component
```

**Page layout:**
- Header: Wrench icon, title "Tools", subtitle "Control which MCP tools are exposed to connected clients"
- Summary cards: target count, total tools, enabled count, denied count
- Search bar with status filters (All / Enabled / Denied)
- Collapsible target cards, each containing individual tool toggles

**Each target card shows:**
- Target name, protocol badge (SSE/MCP), connection status indicator
- Master toggle (enables/disables all tools in the target)
- Tool count badge (e.g., "9/12 tools")
- Expandable tool list with per-tool toggles showing name and description

**Bulk actions per target:**
- "Enable all" / "Disable all" links

### CEL Rule Generation

When no tools are disabled, no `mcpAuthorization` block is added (preserving default allow-all behavior).

Once a user disables their first tool, the UI generates an **allow-list** of CEL rules:

```yaml
policies:
  mcpAuthorization:
    rules:
      # Target-level: allow entire target
      - 'mcp.tool.target == "memory"'
      - 'mcp.tool.target == "sequential-thinking"'
      - 'mcp.tool.target == "context7"'
      # Tool-level: allow specific tools from partially-enabled targets
      - 'mcp.tool.target == "browser-use" && mcp.tool.name == "browser_navigate"'
      - 'mcp.tool.target == "browser-use" && mcp.tool.name == "browser_click"'
      - 'mcp.tool.target == "browser-use" && mcp.tool.name == "browser_type"'
      # ... (excluded: retry_with_browser_use_agent, browser_close_all, etc.)
```

**Optimization:** If all tools in a target are enabled, a single `mcp.tool.target == "name"` rule is used instead of listing every tool.

### Backend Changes

**New endpoint: `GET /tools`** (added to `ui.rs`)

Queries each configured MCP backend's `tools/list` to return the live tool inventory:

```json
{
  "targets": [
    {
      "name": "memory",
      "protocol": "sse",
      "tools": [
        { "name": "create_entities", "description": "Create multiple new entities..." },
        { "name": "search_nodes", "description": "Search for nodes based on a query" }
      ]
    }
  ]
}
```

This is merged with the current `mcpAuthorization` rules to determine each tool's enabled/disabled state.

### Data Flow

```
User toggles tool
    -> tools-config.tsx updates local state
    -> generates CEL rules from toggle state
    -> calls updateConfig() (existing POST /config)
    -> Rust backend validates and writes YAML
    -> file-watcher detects change (250ms)
    -> mcpAuthorization rules take effect
    -> toast: "Configuration updated - hot-reloaded"
```

### Integration Points

| Component | How it's used |
|-----------|--------------|
| `useServer()` context | Read binds/targets/policies state |
| `fetchConfig()` / `updateConfig()` | Read and write config via existing API |
| `refreshBinds()` | Re-sync after save |
| Radix UI components | Card, Switch, Collapsible, Badge, Input |
| Sidebar (`app-sidebar.tsx`) | Add Tools nav item with tool count badge |
| `mcpAuthorization` policy | Generated CEL rules written to route policies |

## Alternatives Considered

1. **Extend the Policies page** - Rejected because tool management is a distinct workflow from general policy config. Users want a quick "what's on/off" view, not a policy editor.

2. **External management UI** - Rejected because agentgateway already has a mature UI with the right API endpoints. Adding a separate service adds unnecessary infrastructure.

3. **CLI toggle script** - Could complement the UI but doesn't solve the visibility/dashboard need.

## Open Questions

- Should the `GET /tools` endpoint cache results, or query MCP backends on every request?
- Should tool state persist separately from `mcpAuthorization` rules (e.g., a `tools.yaml` sidecar) to avoid coupling UI state to CEL policy format?
- Should this page also display tool call metrics/counts if available from the tracing system?
