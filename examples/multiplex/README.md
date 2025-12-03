## Multiplex Example

In the [basic](../basic) example, we exposed a single MCP server.
Agentgateway can also multiplex multiple MCP servers, and expose them as a single MCP server to clients.

This can centralize and simplify client configuration -- as we add and remove tools, only the gateway configuration needs to change, rather than all MCP clients.

### Running the example

```bash
cargo run -- -f examples/multiplex/config.yaml
```

Multiplexing is only a matter of adding multiple targets. Here we will serve the `everything` and `time` server.

```yaml
targets:
- name: time
  stdio:
    cmd: uvx
    args: ["mcp-server-time"]
- name: everything
  stdio:
    cmd: npx
    args: ["@modelcontextprotocol/server-everything"]
```

Now when we open the MCP inspector we can see the tools from both `time` and `everything`.
Because we have multiple tools, each tool is prefixed with the `<name>_` to avoid collisions.

![Tools List](./img/list.png)

### Multiple MCP Backend Groups

You can also define multiple MCP backend blocks in the same route's `backends` array. This is useful when you need different configurations for different groups of MCP servers.

**Example use cases:**
- Different authentication tokens per group
- Different timeout/retry policies per group
- Mixing stateful and stateless servers (note: currently all targets use the first group's stateful mode)

```yaml
backends:
  # First MCP backend group
  - mcp:
      stateful_mode: stateful
      targets:
      - name: internal-tools
        stdio:
          cmd: npx
          args: ["@example/internal-tools"]
    policies:
      auth:
        type: bearer
        token: $INTERNAL_TOKEN
  # Second MCP backend group
  - mcp:
      stateful_mode: stateful
      targets:
      - name: external-tools
        stdio:
          cmd: npx
          args: ["@example/external-tools"]
    policies:
      auth:
        type: bearer
        token: $EXTERNAL_TOKEN
```

When multiple MCP backends are configured at the same route level:
- All tools from all groups are merged and returned to clients
- Each backend group can have its own inline policies
- Target names must be unique across all groups to avoid collisions

