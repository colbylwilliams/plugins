# Copilot Plugins

A [GitHub Copilot plugin marketplace](https://docs.github.com/en/copilot/concepts/agents/about-plugins) — a single repository that indexes installable **plugins** (skills, custom agents, hooks, MCP/LSP servers) through one `marketplace.json` feed.

Plugins here use the standard [Open Plugins](https://open-plugins.com/) format, so a plugin installed from this marketplace works anywhere the Copilot agent runs, including:

- **Copilot CLI** (`copilot`) — `copilot plugin install ...`
- **Copilot SDK** apps — Node, Python, Go, .NET, Java, and Rust (via installed plugins or the `pluginDirectories` session option)
- **Copilot cloud agent** (via `enabledPlugins` in a repo's `.github/copilot/settings.json`)

## Install plugins from this marketplace

Register the marketplace once, then browse and install any plugin:

```bash
copilot plugin marketplace add colbylwilliams/plugins
copilot plugin marketplace browse colby-plugins
copilot plugin install hello-copilot@colby-plugins
```

Or install a single plugin directly, without registering the marketplace:

```bash
# whole repo (root plugin) or a subdirectory
copilot plugin install colbylwilliams/plugins:plugins/hello-copilot
```

### Declarative install (teams / CI / cloud agent)

Add to `~/.copilot/settings.json` (personal) or a repo's `.github/copilot/settings.json` (also read by the cloud agent):

```jsonc
{
  // only needed for marketplaces that aren't registered by default
  "extraKnownMarketplaces": {
    "colby-plugins": { "source": { "source": "github", "repo": "colbylwilliams/plugins" } }
  },
  "enabledPlugins": {
    "hello-copilot@colby-plugins": true
  }
}
```

## What's here

| Plugin | Components | Description |
| ------ | ---------- | ----------- |
| [`hello-copilot`](plugins/hello-copilot) | skill + agent | Minimal starter — copy it as a template for your own plugins. |
| [`filesystem-mcp`](plugins/filesystem-mcp) | MCP server | Example MCP-only plugin using `${PLUGIN_ROOT}`. |

## Repository layout

```text
.
├── .github/plugin/marketplace.json   # the feed (Copilot also reads .claude-plugin/)
└── plugins/
    └── <plugin-name>/
        ├── plugin.json               # required manifest
        ├── skills/<name>/SKILL.md    # skills   (SKILL.md + YAML frontmatter)
        ├── agents/<name>.agent.md    # custom agents
        └── .mcp.json                 # MCP servers (optional; use ${PLUGIN_ROOT})
```

## Add a new plugin

1. `mkdir -p plugins/my-plugin` and add a `plugin.json` (see [`hello-copilot/plugin.json`](plugins/hello-copilot/plugin.json)).
2. Add components: `skills/<name>/SKILL.md`, `agents/<name>.agent.md`, `.mcp.json`, etc.
3. Add an entry to [`marketplace.json`](.github/plugin/marketplace.json) with `"source": "plugins/my-plugin"`.
4. Test locally: `copilot plugin install ./plugins/my-plugin` then `copilot plugin list`.
5. Commit and push. Existing users refresh with `copilot plugin marketplace update colby-plugins`.

### Authoring tips (portable across every surface)

- Keep `SKILL.md` frontmatter `name` and `description` on single lines.
- Use `${PLUGIN_ROOT}` for paths inside `.mcp.json` / `lsp.json`.
- For launch scripts, provide both `bash` and `powershell` commands so the plugin works cross-OS.
- The manifest is also discovered at `.plugin/plugin.json` and `.claude-plugin/plugin.json`, so these plugins are Claude Code compatible too.

## Documentation

- [About plugins](https://docs.github.com/en/copilot/concepts/agents/about-plugins)
- [Creating a plugin](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating)
- [Creating a plugin marketplace](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace)
- [CLI plugin reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference)

## License

[MIT](LICENSE)
