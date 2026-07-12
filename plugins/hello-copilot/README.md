# hello-copilot

A minimal starter plugin. Use it to verify the marketplace works and as a template for your own plugins.

## Components

| Type | Path | Notes |
| ---- | ---- | ----- |
| Skill | [`skills/hello-world/SKILL.md`](skills/hello-world/SKILL.md) | Confirms plugin skills load. |
| Custom agent | [`agents/hello.agent.md`](agents/hello.agent.md) | Confirms plugin agents load. |

## Try it

```bash
copilot plugin install hello-copilot@colby-plugins
```

Then, in an interactive Copilot CLI session:

```text
/skills list      # hello-world should appear
/agent            # hello should appear
```

## Use as a template

Copy this directory to `plugins/<your-plugin>/`, edit `plugin.json`, replace the
skill and agent with your own, and add an entry to the repo's
`.github/plugin/marketplace.json`.
