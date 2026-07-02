---
name: publish-official-qstrike-builder
description: Use when publishing an official (stable) version of QStrike Builder packages to npm from the main branch, after a Release PR (staging → main) has been merged into main
---

'PROMPT: I need to publish the official QStrike Builder version. The Release PR was just merged into main. What do I do?'

# Publish Official QStrike Builder

## Overview

Step-by-step process for publishing an official (stable) QStrike Builder release via Lerna. Always runs from the `main` branch, always publishes to the default `latest` dist-tag — no `--preid`, no `--pre-dist-tag`. Companion to publish-prerelease-qstrike-builder (which handles alpha publishes from `staging`).

## Pre-conditions

- A Release PR (staging → main, e.g. `Release/next.`) has been merged into `main`
- You will end up on the `main` branch (`lerna.json` `allowBranch` permits it)

## Steps

### 1. Checkout and pull latest main

```bash
git checkout main && git pull origin main
```

### 2. Verify latest merged commit

```bash
git fetch origin
git log -1
```

Confirm the top commit is the Release PR merge just completed — check the message and author. If main's tip predates the expected PR, **stop and ask the user** — do not publish a stale commit.

### 3. Clean build cache

```bash
rm -rf node_modules/.cache
```

### 4. Ask the user which version bump

**The agent MUST ask the user** (via AskUserQuestion) whether this release is a `patch`, `minor`, or `major` bump. Never assume, never infer from the diff, and never substitute `from-git`/`from-package` as the primary publish strategy — those are recovery commands only (step 8).

Graduation semantics with lerna: if main carries prerelease versions from the Release PR (e.g. `2.16.5-alpha.2`), a `patch` bump graduates them to stable (`2.16.5`); `minor` → `2.17.0`; `major` → `3.0.0`.

### 5. Run a build first

Verify the build succeeds before publishing:

```bash
npx lerna run build
```

Fix any build errors before proceeding.

### 6. Publish official version

Run the publish command with the bump the user chose:

```bash
echo y | npm run publish -- <patch|minor|major>
```

This expands to: `npx lerna run build && npx lerna publish <bump>`

Lerna will show the version bump list — verify the new versions are stable (no `-alpha` suffix), then the piped `y` confirms automatically. If the versions look wrong, cancel with Ctrl+C and investigate.

### 7. Verify npm packages updated

```bash
npx lerna ls --json | grep '"name"' | awk -F'"' '{print $4}' | xargs -I{} npm info {}@latest version 2>/dev/null
```

Every package should print the new stable version. Or check manually via GitHub **qstrike-builder** repository → **Packages** tab. Also confirm nothing landed on the wrong tag: `npm info @qstrike/builder dist-tags` — `latest` must be the new version.

### 8. Handle branch protection errors

If publish fails with a branch protection / push error:

**8a. Check if a publish commit was already created:**

```bash
git log --oneline -5
```

Look for a commit like `chore(release): publish` in the log.

**8b. If publish commit exists but wasn't pushed:**

```bash
git push origin main --follow-tags
```

**8c. If no publish commit and packages weren't published:**

```bash
npm run publish -- from-package
```

This builds then publishes only packages whose `package.json` version doesn't yet exist on the registry. Deliberately **without** `--pre-dist-tag` — official versions must land on `latest`.

### 9. Create sync PR into staging from main

Do NOT merge or push to staging directly. First check if a sync PR already exists:

```bash
gh pr list --base staging --head main --state open --json title --jq '.[].title'
```

If none exists, create one:

```bash
gh pr create --title "Update Staging" --body "" --base staging --head main
```

This brings the stable version numbers and publish commit back into `staging` so the next pre-release cycle starts from the released baseline.

This requires the GitHub CLI (`gh`) to be installed and authenticated. See `GITHUB_CLI_SETUP.md` if needed.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running from `staging` instead of `main` | Official releases publish only from `main`; `staging` is for alpha pre-releases |
| Adding `--preid alpha` or `--pre-dist-tag next` | Those are pre-release flags — official publishes use a plain bump so packages land on `latest` |
| Choosing the bump yourself (or defaulting to `from-git`) | Always ask the user: patch / minor / major. `from-git`/`from-package` are recovery paths only |
| Forgetting `rm -rf node_modules/.cache` before publishing | Always clear the cache — stale builds cause silent failures |
| Skipping `git log -1` after pulling main | Ensures you're publishing the just-merged Release PR, not a stale commit |
| Merging/pushing `main` into `staging` directly after publish | Open a PR titled `Update Staging` (base `staging`, head `main`) instead |
| Using `from-package` recovery with `--pre-dist-tag next` | That tags stable versions as `next` — omit all pre-release flags in official recovery publishes |
