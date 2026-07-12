---
name: consolidate-dependabot
description: Reviews a repository's open Dependabot PRs across any ecosystem (npm/yarn/pnpm, pip/Poetry/uv, Cargo, Go modules, Maven/Gradle, NuGet, Bundler, Composer, SwiftPM, Docker, GitHub Actions, …), validates each bump on its own — CI status, diff, lockfile and call-site impact — and rolls the safe ones into a single consolidation PR that supersedes the individual ones, performing any small API migrations a bump needs and deliberately leaving out anything that is really a breaking upgrade needing dedicated work. Use when asked to consolidate / batch / sweep / roll up the Dependabot dependency PRs, clear the Dependabot backlog, or turn the open dependency bumps into one PR.
---

# Consolidate the open Dependabot PRs into a single superseding PR

When a repo has Dependabot version updates enabled (see its `.github/dependabot.yml`), it produces a steady drip of one-bump-per-PR noise: patch bumps that are trivially safe, a major bump that needs a one-line API migration, a grouped update that already contains another open PR's bump, and the occasional "bump" that is actually a breaking upgrade in disguise.

This skill is a **Dependabot consolidation pass**: review every open Dependabot PR, validate each one individually, then land the safe ones as a single PR that auto-closes the originals — while correctly excluding the ones that need real work and accurately describing what actually shipped. It is deliberately ecosystem-agnostic: it discovers which ecosystems the repo uses and adapts, rather than assuming one package manager.

## Mindset — validate each bump, don't bulk-merge

This is the most important instruction in this skill. The value here is in the **validation**, not the batching. Blindly squashing every open Dependabot PR into one branch and pushing it is worse than doing nothing — it can drag in a breaking major bump, ship a lockfile that doesn't match the description, or hide a compile failure behind a green-looking diff.

- **Treat each PR as a separate judgement call.** Read its diff, check its CI, and understand what the bump actually touches before you decide to include it. A patch bump to a CI action and a major bump to a package you call directly are not the same risk.
- **Distinguish "safe bump" from "reconciliation work."** A version number going up is sometimes a routine refresh and sometimes an API break with new required arguments and removed types. The first belongs in this sweep; the second belongs to a dedicated effort. Knowing which is which *is* the job.
- **Ship what's true, not what Dependabot guessed.** Dependabot's stated target version can be stale by the time the graph re-resolves. The **lockfile** is the source of truth — describe the versions that actually land, and never artificially pin to an older version just to match a PR title.
- **When a bump is ambiguous — does this major touch our call sites? is this break ours to fix? — stop and investigate** (grep the call sites, read the upstream changelog/release notes) rather than guessing. If it's still ambiguous after that, surface it for the user.
- **Prefer correctness over a tidy batch.** It's fine to include four PRs and exclude two; it's fine for the consolidation to carry a small code migration. Don't force a borderline bump in just to make the sweep look complete.

Lean on the `explore` agent (or parallel investigation) when several PRs each need independent digging — multiple major bumps, or several failing-CI PRs to triage at once.

## Familiarize yourself first

Before touching any PR, ground yourself in *this* repo — never assume conventions from another project:

- **Read the repo's own guidance.** Skim `AGENTS.md` / `CONTRIBUTING.md` / `README.md` (whichever exist) for the build / lint / test gates and the PR conventions (branch naming, commit-message/PR title style, review flow, merge strategy, and any `Co-authored-by` trailer the repo expects).
- **Read `.github/dependabot.yml`** to know which ecosystems and directories Dependabot watches, the update schedule, any `groups`, and — importantly — anything explicitly `ignore`d (a dependency that's ignored is out of scope for this sweep; if a PR for it appears anyway, treat it as excluded).
- **Learn the workspace layout** so you can tell a **direct** dependency from a **transitive** one. A direct dependency is declared in a manifest and is version-bumped there; a transitive dependency appears only in the lockfile. This distinction decides whether a bump needs a manifest edit or just a lockfile refresh.
- **Note how the repo pins GitHub Actions** (if it uses them): by commit **SHA** with a trailing `# vX.Y.Z` comment (the hardened style) or by **tag** (`@v4`). Match whatever the repo already does, in *every* workflow that references the action.
- **Mirror CI.** Open the CI workflow(s) under `.github/workflows/` and note the exact commands the repo runs (format, lint, build, test, audit/deny). Those are the gates your consolidation branch must pass — there is no universal command, so use the repo's.

### Ecosystem reference — where to bump and how to verify

Detect the ecosystems from `dependabot.yml` and the files present, then use the matching row. Whatever the ecosystem, the shape is identical: **for a direct dependency, raise the version in the manifest that declares it, then refresh the lockfile; for a transitive / lockfile-only bump, just refresh the lockfile.** The lockfile — not the PR title — is the truth for what shipped.

| Ecosystem | Manifest(s) — direct deps | Lockfile — source of truth | Refresh after a bump | Verify what resolved |
| --------- | ------------------------- | -------------------------- | -------------------- | -------------------- |
| GitHub Actions | `.github/workflows/*.yml`, composite `action.yml` | none — the SHA/tag *is* the pin | update the pinned SHA **and** `# vX.Y.Z` comment (or the tag) in every workflow that uses it | `grep -rn "<owner>/<action>@" .github/workflows/` |
| npm / yarn / pnpm | `package.json` | `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` | `npm install` · `yarn` · `pnpm install` (or `npm install <dep>@<ver>` for a direct bump) | `npm ls <dep>` / grep the lockfile |
| Python (pip) | `requirements*.txt`, `pyproject.toml`, `setup.py`/`setup.cfg` | `poetry.lock` / `Pipfile.lock` / `uv.lock` / `pdm.lock` (if any) | tool-specific: `poetry update <dep>` · `uv lock --upgrade-package <dep>` · `pip-compile` | `pip show <dep>` / grep the lock |
| Cargo (Rust) | `Cargo.toml` | `Cargo.lock` | `cargo update -p <dep>` (or `--workspace`) | `cargo tree -i <dep>@<ver>` / grep `Cargo.lock` |
| Go modules | `go.mod` | `go.sum` | `go get <module>@vX.Y.Z && go mod tidy` | `go list -m <module>` / grep `go.mod` |
| Maven | `pom.xml` | none by default | edit the version/property in `pom.xml` | `mvn dependency:tree` |
| Gradle | `build.gradle[.kts]`, `gradle/libs.versions.toml` | `gradle.lockfile` (if enabled) | edit the version catalog / build file | `./gradlew dependencies` |
| NuGet (.NET) | `*.csproj`, `Directory.Packages.props` | `packages.lock.json` (if enabled) | `dotnet add package <pkg> -v <ver>` | grep the csproj / lock |
| Bundler (Ruby) | `Gemfile`, `*.gemspec` | `Gemfile.lock` | `bundle update <gem>` | `bundle info <gem>` |
| Composer (PHP) | `composer.json` | `composer.lock` | `composer update <pkg>` | `composer show <pkg>` |
| SwiftPM | `Package.swift`, `*.xcodeproj` | `Package.resolved` | `swift package update <dep>` / `xcodebuild -resolvePackageDependencies` | grep `Package.resolved` |
| Docker | `Dockerfile`, `*compose*.y*ml` | none — the tag/digest *is* the pin | update the `FROM` tag/digest | grep the Dockerfile(s) |
| Others (elm, hex, pub, terraform, gitsubmodule, …) | the ecosystem's manifest | the ecosystem's lock, if any | the ecosystem's own update command | the ecosystem's own tree/why command |

## How Dependabot PRs show up

- **Author handle.** Dependabot PRs are authored by the Dependabot app. List them with `gh pr list --author "app/dependabot"` (some setups match `dependabot[bot]` — try that if the first returns nothing).
- **Title style.** `Bump <dep> from <a> to <b>`, optionally with a configured commit-message prefix (e.g. `Build(deps): Bump …`, `chore(deps): …`). Grouped updates read `Bump the <group> group …` and carry several bumps in one PR.
- **Branch style.** `dependabot/<ecosystem>/<path>/<dep>-<ver>` (e.g. `dependabot/npm_and_yarn/…`, `dependabot/cargo/…`, `dependabot/github_actions/…`).
- **What the diff touches.** A direct bump edits a manifest and the lockfile; a transitive/in-range bump edits only the lockfile; an Actions bump edits the SHA (+ comment) or tag in the workflow files.
- **Superseding.** Listing `Closes #N` for each source PR in the consolidation body links them; because the consolidation PR targets the default branch, merging it **closes each referenced Dependabot PR** (GitHub's closing keywords close a referenced *pull request*, not only an issue). Still confirm they closed after merge and close any straggler by hand.

## The consolidation pass

Treat the following as the shape of the work, not a rigid script. Adapt to what the open set actually contains.

### Phase 1 — Inventory the open Dependabot PRs

Pull the full open set with the signals you'll triage on — CI rollup, mergeability, labels, branch, age:

```sh
# The JSON carries every triage signal:
gh pr list --author "app/dependabot" --state open --limit 100 \
  --json number,title,headRefName,baseRefName,createdAt,labels,mergeable,statusCheckRollup

# For a quick scan, collapse to one line per PR:
gh pr list --author "app/dependabot" --state open --limit 100 \
  --json number,title,headRefName,mergeable \
  --jq '.[] | "#\(.number)\t\(.mergeable)\t[\(.headRefName)]\t\(.title)"'
```

Note which ecosystem each belongs to, and which are **grouped** (one PR carrying several bumps) — a grouped PR may already contain a bump another standalone PR also proposes.

### Phase 2 — Review and validate each PR individually

For every open PR, build a real understanding before classifying it:

1. **Read the change.** `gh pr view <n> --json title,body,files,additions,deletions` and `gh pr diff <n>`. Note whether the bump touches a manifest (a direct dependency, possibly out of the declared range) or only the lockfile, and whether it's patch / minor / major (semver). For Actions, note the exact SHA/tag Dependabot pinned.
2. **Check CI on the source PR.** `gh pr checks <n>`. A green source PR is the strongest signal the bump is safe on its own. For a **failing** one, read the failure — `gh run view <run-id> --log-failed` — and find out *why*: a benign flake, or real build/test errors the bump introduced.
3. **Assess call-site impact** for anything non-trivial (every major, and any minor that touches a package you call directly). Grep the codebase for the package's API at the call sites to see whether the new version breaks them — e.g. `grep -rnE "<import-or-symbol>" <source-dirs>`. For Actions, confirm which workflows reference it: `grep -rn "<owner>/<action>@" .github/workflows/`.

### Phase 3 — Classify each PR into a bucket

- **Include (safe).** Passes CI on its own and is a patch/minor bump, or a major with no real call-site impact. Goes straight into the consolidation.
- **Include with a small migration.** A bump (often a major) that's safe *once you make a small, well-understood code change* the standalone Dependabot PR can't carry on its own — e.g. a renamed API at the one call site that uses it. You do that migration as part of the consolidation, and the source PR (which was red without it) gets superseded.
- **Naturally covered.** A bump that's already contained in another included PR — e.g. a standalone bump whose exact version also ships inside a grouped PR. Don't apply it twice; note that it's superseded for free.
- **Exclude — needs real reconciliation.** A "bump" that is actually a breaking upgrade requiring genuine work: removed/renamed types, new required arguments, many compile errors. That is **not** a routine dependency sweep — **leave its PR open**, say so, and route it to a dedicated effort. If such an effort is already in flight on a branch, point at it. Anything explicitly `ignore`d in `dependabot.yml` is excluded by definition.

State your classification — which PRs land, which carry a migration, which are covered, which you're excluding and why — before you start editing. If the call is genuinely close, ask the user.

### Phase 4 — Build the consolidation branch

Branch from the current default branch (follow the repo's branch-naming convention), then apply the included bumps:

- **GitHub Actions bumps.** Update the pin in **every** workflow that references the action, matching the repo's style. If it pins by SHA, match the exact SHA Dependabot pinned (`gh pr diff <n>` shows it) and update the `# vX.Y.Z` comment too; if it pins by tag, update the tag. To resolve a tag to the commit SHA Actions pins to:

  ```sh
  # Use the commits endpoint — it dereferences annotated tags to the underlying commit,
  # whereas git/ref/tags/<tag>'s .object.sha can be the tag-object SHA, not the commit.
  gh api repos/<owner>/<action>/commits/v<X.Y.Z> --jq '.sha'
  grep -rn "<owner>/<action>@" .github/workflows/   # update every occurrence
  ```

- **Package bumps.** Use the ecosystem row above. For a **direct** dependency whose new version is within the declared range (the common minor/patch case), no manifest edit is needed — just refresh the lockfile. For a **major** bump that exceeds the declared range on a direct dependency, first raise the constraint in the manifest that declares it, then refresh the lockfile. For a **transitive / lockfile-only** bump, just refresh the lockfile. Prefer taking Dependabot's exact resolved change as the target when it's within range.
- **Migrations.** Make the small call-site edits the "include with migration" bumps require.

### Phase 5 — Verify what actually landed (the part that bites)

The lockfile, not Dependabot's PR title, is the truth. Before you write a word of the PR body, confirm every claimed version against the lockfile (`git diff -- <lockfile>` shows exactly which pins changed):

- **Resolution can pick a newer in-range patch than Dependabot's stated target** (a patch published after the PR opened). That's correct — **don't pin back**; describe the version that actually resolved.
- **Old versions can linger as transitive deps and that's fine.** A dependency you didn't bump may still resolve to an older version because an upstream package requires it. Expected — note it (use the ecosystem's "why/tree" command from the table) instead of fighting it.
- **Independent lockfiles may legitimately differ.** If the repo has several lockfiles (separate resolver roots / workspaces), each may pin a different compatible version of the same dependency; don't force them to agree. Update the lock that actually gates what you're shipping.

If a reviewer later flags a version mismatch (e.g. "body says 1.2.0, lock says 1.2.1"), this is the check that resolves it: verify against the lockfile and fix the **description** (or, only if the older version is genuinely required, the resolution) — don't pin artificially.

### Phase 6 — Validate with the repo's gates

Run the checks the repo itself runs — the ones you noted from its CI workflow and `CONTRIBUTING.md` / `AGENTS.md` (there is no universal command). Typically that's some subset of format, lint, build, test, and a dependency audit (e.g. `cargo deny`, `npm audit`, `pip-audit`). Don't consider the branch done until they're all green on the consolidation branch. If a resolve/install step is part of the flow, run it first so the lockfile is settled before build and test.

### Phase 7 — Commit, push, open the PR

- **Title:** a clear consolidation title following the repo's convention, e.g. `Consolidate safe Dependabot dependency bumps` (add the repo's commit-message prefix if it uses one).
- **Body:** a table of every included bump — the bump, its source PR, and *why it's safe* — plus the migrations you made, the PRs naturally covered, and a clear "what I left out and why" for the excluded ones.
- **Auto-close the originals:** add a `Closes #N` line for every superseded PR. Merging the consolidation PR into the default branch then closes each referenced Dependabot PR. Leave excluded PRs out of the `Closes` list (they stay open).
- Commit and open the PR per the repo's conventions (including any `Co-authored-by` trailer). Follow the repo's review/merge flow; after merge, confirm the superseded PRs actually closed and close any straggler by hand.

### Report

Summarize for the user: which PRs were consolidated (and the single PR that now carries them), which were naturally covered, which were **excluded and why** (with a pointer to the effort that should pick them up), any migrations performed, and any version drift worth flagging (newer-patch resolutions, lingering transitive versions).

## Guardrails

- **Validate each PR before including it.** CI status, diff, and call-site impact — per PR. Never squash the whole open set in blindly.
- **Exclude breaking upgrades.** A major break with compile errors is reconciliation work, not a dep sweep — leave its PR open and route it to the right effort. Anything `ignore`d in `dependabot.yml` is out of scope.
- **Describe what shipped, not what Dependabot guessed.** Verify every version against the lockfile; don't pin to an older version to match a stale title; document lingering transitive versions. Treat multiple lockfiles as independent resolver roots — don't force them to match.
- **Match the repo's Actions pinning style,** and update the pin (and version comment, if used) in *every* workflow that uses the action.
- **Use `Closes #N`** for each superseded PR so merging the consolidation PR closes it; keep excluded PRs out of that list, and verify the closures landed after merge.
- **Validate before declaring done** using the repo's own gates (mirror CI). Don't invent commands the repo doesn't use.
- **When a classification is genuinely ambiguous, ask** rather than guessing — especially before excluding a bump or pushing a migration the user didn't anticipate.

## Appendix — grounding commands

Starting points; the repo's own state is authoritative if it disagrees (`AGENTS.md`, `CONTRIBUTING.md`, `.github/dependabot.yml`, `.github/workflows/`).

**Inventory and read the open Dependabot PRs:**

```sh
gh pr list --author "app/dependabot" --state open --limit 100 \
  --json number,title,headRefName,mergeable,statusCheckRollup
gh pr view <n> --json title,body,files,additions,deletions
gh pr diff <n>
gh pr checks <n>
gh run view <run-id> --log-failed   # for a failing source PR
```

**Apply the bumps** (see the ecosystem table for the exact manifest/lockfile/refresh commands):

```sh
# GitHub Actions: resolve the tag to its commit SHA (dereferences annotated tags),
# then update the SHA + version comment (or the tag) everywhere
gh api repos/<owner>/<action>/commits/v<X.Y.Z> --jq '.sha'
grep -rn "<owner>/<action>@" .github/workflows/

# Packages: raise the manifest constraint for an out-of-range direct bump, then
# refresh the lockfile with the ecosystem's update command (npm install / cargo update
# -p <dep> / go get <mod>@ver && go mod tidy / bundle update <gem> / …).
```

**Verify what actually resolved** (against the lockfile, using the ecosystem's tree/why command):

```sh
git diff -- <lockfile>                     # exactly which pins changed
# e.g. cargo tree -i <dep>@<ver> · npm ls <dep> · go list -m <module> · bundle info <gem>
```

**Validate (must pass before opening the PR):** run the repo's own fmt / lint / build / test / audit gates as configured in its CI and contributing docs.
