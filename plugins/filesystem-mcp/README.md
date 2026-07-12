# filesystem-mcp

An example **MCP-only** plugin. It contributes a single [Model Context Protocol](https://modelcontextprotocol.io) server and no skills or agents, to show how a plugin can package an integration.

## What it does

[`.mcp.json`](.mcp.json) declares a `filesystem` server launched with `npx`. The
`${PLUGIN_ROOT}` template variable expands to the installed plugin's directory,
so the server is scoped to the plugin folder rather than your whole disk:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "${PLUGIN_ROOT}"]
    }
  }
}
```

Point the server at a different directory by editing the last argument.

## Try it

```bash
copilot plugin install filesystem-mcp@colby-plugins
```

The `filesystem` MCP server starts with your next session (Node.js and `npx`
must be available on your `PATH`). Check status in an interactive session with:

```text
/mcp
```

## Notes

- `${PLUGIN_ROOT}` (and the alias `${CLAUDE_PLUGIN_ROOT}`) resolve to the plugin
  directory on every surface, so paths stay portable.
- For servers you launch with a script, prefer `bash` + `powershell` entries over
  a raw `command` so the plugin works on Windows and Unix alike.
