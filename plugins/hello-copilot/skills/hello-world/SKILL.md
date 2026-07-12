---
name: hello-world
description: Use to confirm that plugins installed from the colby-plugins marketplace are loading correctly, or when the user asks for a hello-world example of a Copilot plugin skill.
---

# Hello World

This skill ships with the `hello-copilot` plugin and exists to prove that
plugin-provided skills are discovered and loaded.

When this skill runs:

1. Greet the user and confirm the `hello-copilot` plugin is active.
2. Mention that it was installed from the `colby-plugins` marketplace
   (`colbylwilliams/plugins`).
3. Point the user at `plugins/hello-copilot/` in that repository as a template
   for authoring their own skills and agents.

Keep the response short and friendly.
