# consolidate-dependabot

A single, ecosystem-agnostic skill for **clearing the Dependabot backlog**: it reviews every open Dependabot PR, validates each bump on its own (CI status, diff, lockfile and call-site impact), and rolls the safe ones into **one consolidation PR** that supersedes the individual ones — performing any small API migrations a bump needs and deliberately leaving out breaking upgrades that need dedicated work.

It works across whatever package managers a repo uses — npm/yarn/pnpm, pip/Poetry/uv, Cargo, Go modules, Maven/Gradle, NuGet, Bundler, Composer, SwiftPM, Docker, GitHub Actions, and more — by detecting the repo's ecosystems from `.github/dependabot.yml` and mirroring the repo's own CI/contributing gates rather than assuming one toolchain.

## Components

| Type | Path | Notes |
| ---- | ---- | ----- |
| Skill | [`skills/consolidate-dependabot/SKILL.md`](skills/consolidate-dependabot/SKILL.md) | The consolidation pass: inventory → validate → classify → build → verify → open a superseding PR. |

## Install

```bash
copilot plugin install consolidate-dependabot@colby-plugins
```

Or directly, without registering the marketplace:

```bash
copilot plugin install colbylwilliams/plugins:plugins/consolidate-dependabot
```

## Use it

In a repository that has Dependabot version updates enabled, ask Copilot to:

```text
Consolidate the open Dependabot PRs
```

Other phrasings that trigger the skill: *batch / sweep / roll up the Dependabot PRs*, *clear the Dependabot backlog*, or *turn the open dependency bumps into one PR*.

## What it will and won't do

- **Will** validate each PR individually, include the safe bumps, carry small well-understood migrations, verify resolved versions against the lockfile, and open one PR that `Closes` the superseded ones.
- **Won't** blindly squash every PR together, ship a lockfile that doesn't match the description, or pull in a breaking major upgrade that needs real reconciliation work (it leaves those PRs open and flags them).
