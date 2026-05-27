---
name: sync-upstream-fork
description: Sync this Grafana fork of release-please with upstream googleapis/release-please while preserving our intentional deviations (multi-branch release support from PR #2203, dry-run JSON output, zizmor CI fixes, and yarn build tooling). Use when the user says "sync upstream", "update from upstream", "merge upstream", "rebase on upstream", "pull in upstream changes", "catch up with googleapis/release-please", or otherwise wants to integrate new commits from the upstream repository.
disable-model-invocation: true
allowed-tools: Read, Bash, Glob, Grep
---

# Sync This Fork With Upstream `googleapis/release-please`

Helps integrate new commits from upstream `googleapis/release-please` into this Grafana fork while preserving the fork's intentional deviations. **A merge-based sync is the default** because `grafana/release-please` is consumed by other repos; rewriting `main`'s history would break them.

## Core Responsibilities

1. **Prerequisite validation** — clean working tree, both remotes wired up, branches up to date
2. **Identify the last shared commit** (merge-base) and what's new on upstream
3. **Verify intentional deviations are still detectable** before integrating (so you'll know if a merge silently loses one)
4. **Safety backup creation** — branch `backup/sync-upstream-<date>` before any merge/rebase
5. **Perform the merge** (preferred) or rebase (only if explicitly requested)
6. **Resolve conflicts while preserving every documented deviation**
7. **Validate** — `yarn install --immutable && yarn lint && yarn test` must pass
8. **Recovery assistance** if the sync goes sideways

## When Merge vs Rebase

| Situation | Approach |
| --- | --- |
| Normal upstream sync of `main` (default) | **Merge** (`git merge upstream/main`) |
| Long-running fork-only feature branch behind `main` | Rebase the feature branch onto fork `main` |
| User explicitly asks for "rebase on upstream" | Rebase, then warn that `main` will need force-push and downstream consumers may break |
| Upstream has done a force-push or history rewrite | Stop. Ask the user how to proceed — almost never the right thing to silently mirror. |

> **Best Practices**
>
> - ✅ Always create a backup branch before merging or rebasing
> - ✅ Run the full `yarn test` suite after resolving conflicts (984+ tests today)
> - ✅ Use `git commit -S` for the merge commit (signed commits are required in this repo)
> - ✅ Keep the merge commit on `main`; do not squash a sync merge
> - ❌ Never force-push `main` of this fork
> - ❌ Never drop one of the documented deviations to make a conflict "easier"

---

## Intentional Deviations (MUST be preserved)

These changes exist on `grafana/release-please` but **not** on `googleapis/release-please`. During every sync, verify each one is still present after the merge before declaring success.

### A. PR #2203 — multi-branch release support (CLOSED upstream, never merged)

- **Upstream PR**: <https://github.com/googleapis/release-please/pull/2203>
- **Fork branch**: `origin/multi-branch-release`
- **Fork commits**:
  - `b1c633a feat: look for prev release across all releases`
  - `c3e164a chore: rename includeAllReleases -> considerAllBranches`
  - `d9d0a44 fix: record size comparison`
- **Files touched**:
  - `src/bin/release-please.ts` — wires the `--consider-all-branches` CLI flag
  - `src/github.ts` — adds ~205 lines using GitHub's compare-commits API to find a previous release across branches
  - `src/manifest.ts` — threads `considerAllBranches` through the manifest path
- **Sanity checks after sync** (symbols verified against current `origin/main`):
  - `rg -nw "considerAllBranches|consider-all-branches" src/bin/release-please.ts src/manifest.ts` → must match
  - `rg -n "diffCommitIterator|diffCommitsGraphQL" src/github.ts` → must match (these are the helpers added by PR #2203)

### B. Dry-run JSON output + small CLI conveniences (PR #1 to fork)

- **Fork branch**: `origin/dry-run-output`
- **Merged via**: commit `c70ff8a Merge pull request #1 from trevorwhitney/dry-run-output`
- **Fork commits** (oldest → newest):
  - `dcc5a53 add option to output results of dry run as json`
  - `1d6381f chore: rename output dry run -> dry run output`
  - `6e5d24a chore: add debug logging to figure out footer issue`
  - `ff0a9ed feat: expose cli flag for separate pull requests`
  - `68b9244 fix: get consider-all-branches from config file`
  - `0b68acf feat: pass manifest file to fromConfig`
  - `cb9955e chore: remove some troubleshooting logging`
  - `9e713c2 feat: allow group pr title to be specified on the command line`
  - `9f208b4 fix: type error`
  - `789c90c feat: include headers in github error`
  - `8f8405d feat: ability to tag a specific sha when publishing a release`
  - `4c2c225 chore: add more logigng`
- **Files touched**:
  - `src/bin/release-please.ts` — new CLI flags (`--dry-run-output`, `--separate-pull-requests`, `--group-pull-request-title`, `--sha`, etc.)
  - `src/manifest.ts` — config plumbing, dry-run JSON output
  - `src/errors/index.ts` — include response headers in `GitHubAPIError`
  - `src/strategies/base.ts`, `src/strategy.ts` — accept a custom SHA when creating a release
- **Sanity checks after sync** (symbols verified against current `origin/main`):
  - `rg -nw "dryRunOutput|dry-run-output" src/bin/release-please.ts` → must match
  - `rg -nw "groupPullRequestTitlePattern|group-pull-request-title" src/bin/release-please.ts src/manifest.ts` → must match
  - `rg -nw "separatePullRequests|separate-pull-requests" src/bin/release-please.ts src/manifest.ts` → must match
  - `rg -nw "manifest-file" src/bin/release-please.ts` → must match
  - `rg -n "ResponseHeaders|headers\\?: ResponseHeaders" src/errors/index.ts` → must match (response headers attached to `GitHubAPIError`)

### C. Zizmor CI security fixes

- **Fork commit**: `13247c6 ci: apply zizmor fixes` (merged via `b07e6f4`)
- **Files touched**: `.github/workflows/ci.yaml`
- **What it does**:
  - Adds `persist-credentials: false` to every `actions/checkout@v4` step
  - Sanitizes `GITHUB_REF` (regex-validated, passed via `env:`) before injecting it into `npm install --save googleapis/release-please#<ref>` in the `build-action` job
- **Sanity checks after sync**:
  - `[ "$(grep -c 'persist-credentials: false' .github/workflows/ci.yaml)" -ge 5 ]` (one per `actions/checkout@v4` invocation)
  - `grep -Fq 'REF=${GITHUB_REF#refs/*/}' .github/workflows/ci.yaml`
  - `grep -Fq 'Invalid ref format detected' .github/workflows/ci.yaml`

### E. Removed `build-action` CI job (upstream-only, tests an abandoned action)

- **Status**: fork-only deletion. Upstream `.github/workflows/ci.yaml` ships a `build-action:` job; the fork does NOT.
- **Why removed**:
  - It installs `grafana/release-please` into `google-github-actions/release-please-action` and runs that action's build.
  - That action repo has been abandoned since 2024-05-13 — the last upstream commit there literally added a deprecation warning to the action.
  - The Grafana fork consumes the release-please CLI directly; we have no integration with the action and don't need to verify the action can build against our code.
  - Even with `corepack enable` and a SHA-based install, the job fails because the npm-via-git-dep prepare flow + yarn 4 lockfile semantics no longer cooperate on the action's repo.
- **Sanity check after sync**:
  - `! grep -q '^\s*build-action:' .github/workflows/ci.yaml` (job must NOT reappear)
  - There should be a comment block in `.github/workflows/ci.yaml` explaining why the job was removed (search for "build-action job" or "deviation E").
- **Conflict handling**: if an upstream sync re-introduces the `build-action:` job, delete it again and restore the explanatory comment. Do NOT spend time trying to make it work — there's no consumer.

### D. Yarn build tooling (replaces npm everywhere except npm-publish operations)

- **Status**: uncommitted as of the original yarn refactor; once committed it will be part of `main`.
- **Files touched / added / removed**:
  - `package.json` — scripts use `yarn …`; adds `"packageManager": "yarn@4.15.0"` and `"engines.node": ">=18.0.0"`
  - `yarn.lock` — committed (replaces `package-lock.json`, which is deleted and `.gitignore`d)
  - `.yarnrc.yml` — `nodeLinker: node-modules`, `enableGlobalCache: false`, plus supply-chain hardening: `enableScripts: false` (block third-party install/postinstall scripts; workspace scripts still run) and `npmMinimalAgeGate: "7d"` (refuse package versions <7 days old; requires yarn ≥ 4.10)
  - `.gitignore` — adds `.yarn/*` carve-outs (`releases`, `patches`, `plugins`, `sdks`, `versions` kept) and `.pnp.*`
  - `.github/workflows/ci.yaml` — `corepack enable` + `yarn install --immutable` + `yarn test/lint/docs`; production-deps sanity check uses `yarn workspaces focus --production`. Also pins `JustinBeckwith/linkinator-action` to a commit SHA (was `@v1`, flagged by zizmor's `unpinned-uses`).
  - `.kokoro/test.sh`, `.kokoro/test.bat`, `.kokoro/lint.sh`, `.kokoro/docs.sh`, `.kokoro/system-test.sh`, `.kokoro/samples-test.sh`, `.kokoro/release/docs.sh`, `.kokoro/release/docs-devsite.sh` — yarn-ified
  - `CONTRIBUTING.md` — `yarn install` / `yarn test` / `yarn fix`
- **Intentional npm holdouts** (do NOT yarn-ify these on sync):
  - `.kokoro/publish.sh` — keeps `npm pack` and `npm publish` (the wombat-dressing-room registry behavior is intentionally preserved)
  - `.kokoro/release/docs.sh` — `npm i json@9.0.6 -g` (a global CLI tool, not project dependency management)
  - Docs/tests/snapshots that describe release-please's npm-publish *feature* (`docs/**`, `test/**`, `__snapshots__/**`, `templates/**`, README badge, `bug_report.md`'s "npm version" question)
- **Sanity checks after sync**:
  - `test ! -f package-lock.json && test -f yarn.lock`
  - `grep -q '"packageManager": "yarn@' package.json`
  - `grep -q 'nodeLinker: node-modules' .yarnrc.yml`
  - `grep -q 'enableScripts: false' .yarnrc.yml`
  - `grep -q 'npmMinimalAgeGate:' .yarnrc.yml`
  - `! grep -E '^\s*-?\s*(run|call):.*\bnpm (install|test|run)' .github/workflows/ci.yaml .kokoro/test.sh .kokoro/test.bat .kokoro/lint.sh .kokoro/docs.sh .kokoro/system-test.sh .kokoro/samples-test.sh .kokoro/release/docs.sh .kokoro/release/docs-devsite.sh` (no `npm install/test/run` in build-tooling files since deviation E removed the `build-action` job that needed npm)

---

## Quick Reference

```bash
# One-time: wire upstream remote (idempotent)
git remote add upstream https://github.com/googleapis/release-please.git 2>/dev/null || true
git remote -v | grep upstream

# Fetch
git fetch upstream --no-tags
git fetch origin

# Inspect divergence
MB=$(git merge-base origin/main upstream/main)
git log --oneline "$MB" -1                               # last shared commit
git rev-list --count "$MB"..upstream/main                # upstream ahead of merge-base
git rev-list --count "$MB"..origin/main                  # fork ahead of merge-base
git log --oneline --graph --decorate origin/main ^"$MB"  # fork-only commits
git log --oneline upstream/main ^"$MB"                   # upstream-only commits

# Files the fork has changed vs the merge-base
git diff --stat "$MB" origin/main

# Files upstream has changed since the merge-base — these are the high-risk zones
git diff --stat "$MB" upstream/main

# Files where fork AND upstream both diverged (likely conflict surface)
comm -12 \
  <(git diff --name-only "$MB" origin/main   | sort -u) \
  <(git diff --name-only "$MB" upstream/main | sort -u)
```

---

## Safe Sync Workflow (Merge Strategy — Default)

### Step 1: Validate prerequisites

```bash
git status                                  # MUST be clean
git rev-parse --abbrev-ref HEAD             # MUST be on `main` (or your sync branch)
git remote get-url upstream || echo missing # MUST resolve to googleapis/release-please
git fetch origin && git fetch upstream --no-tags
```

Stop if:
- Working tree has uncommitted changes → commit or stash first
- You're not on `main` (or a dedicated `sync/upstream-<date>` branch)
- `upstream` remote is missing or points to the wrong repo

### Step 2: Inspect what's coming

```bash
MB=$(git merge-base origin/main upstream/main)
echo "merge-base: $MB ($(git log -1 --format='%s' $MB))"
git log --oneline "$MB"..upstream/main | wc -l   # how many upstream commits
git shortlog "$MB"..upstream/main                # grouped by author
git diff --stat "$MB" upstream/main              # files changed upstream
```

Pay special attention to upstream changes in any of these files — they overlap with our deviations:

- `src/bin/release-please.ts`
- `src/github.ts`
- `src/manifest.ts`
- `src/errors/index.ts`
- `src/strategies/base.ts`, `src/strategy.ts`
- `.github/workflows/ci.yaml`
- `package.json`, `package-lock.json` (we deleted this — see deviation D)
- `.kokoro/**`
- `CONTRIBUTING.md`

If upstream touched any of these, expect conflicts.

### Step 3: Create a safety backup and a sync branch

```bash
DATE=$(date +%Y-%m-%d)
git branch "backup/main-pre-sync-$DATE"
git checkout -b "sync/upstream-$DATE"        # do the work on a branch, PR into main
```

### Step 4: Merge upstream

```bash
git merge --no-ff -S upstream/main -m "chore: merge upstream googleapis/release-please@$(git rev-parse --short upstream/main)"
```

- `--no-ff` ensures a merge commit even if fast-forward is possible (preserves history)
- `-S` signs the merge commit (required by repo policy)

If git stops with conflicts, go to **Step 5**. Otherwise, jump to **Step 6**.

### Step 5: Resolve conflicts (preserving every deviation)

**Before resolving anything, understand what each side did.** For each conflicted file `<file>`:

```bash
git log -p -n 5 upstream/main -- <file>      # what did upstream change?
git log -p -n 5 origin/main   -- <file>      # what did we change?
git diff "$MB" origin/main   -- <file>       # full fork diff for this file
git diff "$MB" upstream/main -- <file>       # full upstream diff for this file
```

Then apply the rules below.

#### File-by-file conflict guidance

| File | Conflict strategy |
| --- | --- |
| `src/bin/release-please.ts` | Keep our CLI flags (`--consider-all-branches`, `--dry-run-output`, `--separate-pull-requests`, `--group-pull-request-title`, `--sha`, etc.) AND apply upstream's new flags/refactors. Re-derive a working `argv` definition that includes both sets. |
| `src/github.ts` | Keep our compare-commits–based "previous release across branches" code (deviation A). If upstream refactored `GitHub` class methods, port our additions onto the new shape. |
| `src/manifest.ts` | Keep our `considerAllBranches` plumbing, dry-run JSON output, and `fromConfig(manifestFile, ...)` overload. Apply upstream changes to the rest. |
| `src/errors/index.ts` | Keep our `headers` field on the error type (deviation B). |
| `src/strategies/base.ts`, `src/strategy.ts` | Keep our optional `sha` parameter used when publishing a release. |
| `.github/workflows/ci.yaml` | **Both** keep yarn commands (deviation D) **and** `persist-credentials: false` on every checkout (deviation C) **and** the SHA-pinned `JustinBeckwith/linkinator-action` (deviation D, zizmor). **Do not** re-add the upstream `build-action:` job (deviation E). |
| `package.json` | Keep `"packageManager": "yarn@…"`, `"engines.node": ">=18.0.0"`, and the yarn-form scripts. If upstream added/changed dependencies, merge those into the deps maps. |
| `package-lock.json` | If git recreates it from upstream, delete it again — we use `yarn.lock` only (deviation D). Run `yarn install` afterwards to regenerate `yarn.lock`. |
| `yarn.lock` | After dependency changes resolve, run `yarn install` to update — never hand-edit. If a newly added/updated dep fails the `npmMinimalAgeGate` check, either wait for the version to age in, or add the package to `npmPreapprovedPackages` in `.yarnrc.yml` (only with a clear justification). |
| `.kokoro/*.sh`, `.kokoro/test.bat` | Keep yarn commands (deviation D). Exceptions: `.kokoro/publish.sh` keeps `npm pack`/`npm publish`; `.kokoro/release/docs.sh` keeps `npm i json@9.0.6 -g`. |
| `CONTRIBUTING.md` | Keep `yarn install` / `yarn test` / `yarn fix`. |
| `.gitignore` | Keep the `.yarn/*` carve-outs and `.pnp.*` block. |

#### Generic conflict-resolution recipe

```bash
# 1. Inspect
git status                          # list conflicted files
git diff --check                    # find leftover conflict markers later

# 2. Edit each conflicted file. Conflict markers look like:
#     <<<<<<< HEAD (our fork)
#     ours
#     =======
#     upstream
#     >>>>>>> upstream/main
#    The fix is almost always "combine both" — see table above for per-file guidance.

# 3. Stage and continue
git add <resolved-files>
# (no `--continue` for merge; just commit when all conflicts are resolved)

# 4. Once all conflicts are staged:
git commit -S        # uses the merge message git prepared
```

### Step 6: Verify intentional deviations still exist

Run all sanity checks from the "Intentional Deviations" section above. A one-liner that exits non-zero on any miss (verified to pass against current `origin/main` + the yarn refactor):

```bash
set -e
# Deviation A — PR #2203 (multi-branch release)
rg -q "considerAllBranches" src/bin/release-please.ts src/manifest.ts
rg -q "diffCommitIterator" src/github.ts
# Deviation B — dry-run JSON output + CLI conveniences
rg -q "dryRunOutput" src/bin/release-please.ts
rg -q "groupPullRequestTitlePattern" src/bin/release-please.ts src/manifest.ts
rg -q "separatePullRequests" src/bin/release-please.ts src/manifest.ts
rg -q "manifest-file" src/bin/release-please.ts
rg -q "ResponseHeaders" src/errors/index.ts
# Deviation C — zizmor CI fixes
[ "$(grep -c 'persist-credentials: false' .github/workflows/ci.yaml)" -ge 5 ]
grep -Fq 'REF=${GITHUB_REF#refs/*/}' .github/workflows/ci.yaml
grep -Fq 'Invalid ref format detected' .github/workflows/ci.yaml
# Deviation D — yarn tooling
test ! -f package-lock.json && test -f yarn.lock
grep -q '"packageManager": "yarn@' package.json
grep -q 'nodeLinker: node-modules' .yarnrc.yml
grep -q 'enableScripts: false' .yarnrc.yml
grep -q 'npmMinimalAgeGate:' .yarnrc.yml
grep -q 'JustinBeckwith/linkinator-action@[0-9a-f]\{40\}' .github/workflows/ci.yaml
# Deviation E — build-action job intentionally removed
! grep -Eq '^\s*build-action:' .github/workflows/ci.yaml
echo "All intentional deviations preserved ✅"
```

If any check fails, **stop and re-investigate** — do not commit a sync that drops a deviation.

### Step 7: Validate the build

```bash
corepack enable
yarn install --immutable     # MUST succeed without lockfile mutation
yarn compile                 # tsc -p .
yarn lint                    # gts check
yarn test                    # 984+ tests must pass (today's baseline)
```

If `yarn install --immutable` fails because dependencies legitimately changed upstream, run `yarn install` (without `--immutable`) to update `yarn.lock`, inspect the diff, then commit the lockfile update on top of the merge commit.

### Step 8: Push and open a PR

Because `main` is protected and consumed by other repos, do **not** push the sync branch directly to `main`:

```bash
git push -u origin "sync/upstream-$DATE"
gh pr create \
  --base main \
  --head "sync/upstream-$DATE" \
  --title "chore: sync upstream googleapis/release-please" \
  --body "Merges upstream/main into fork main. Preserves intentional deviations A (PR #2203), B (dry-run JSON output + CLI flags), C (zizmor CI fixes), D (yarn tooling). See .agents/skills/sync-upstream-fork/SKILL.md."
```

Have a teammate review before merging.

### Step 9: Clean up after merge

```bash
git checkout main
git pull --ff-only origin main
git branch -d "sync/upstream-$DATE"
git branch -d "backup/main-pre-sync-$DATE"   # only after you're confident
```

---

## Alternative: Rebase Strategy (only if explicitly requested)

> ⚠️ Rebasing rewrites `main`. Downstream consumers using `googleapis/release-please#<sha>` or `grafana/release-please#main` will see history change. Confirm with the user before doing this.

```bash
git fetch upstream --no-tags
git branch "backup/main-pre-rebase-$(date +%Y-%m-%d)"
git rebase upstream/main         # resolve conflicts using the same per-file table above
yarn install && yarn lint && yarn test
git push --force-with-lease origin main
```

For long-running fork-only feature branches that need to catch up with fork `main`, the regular [git-rebase skill](../../../.pi/agent/skills/git-rebase/SKILL.md) applies — this skill is specifically for the fork-vs-upstream relationship.

---

## Recovery From a Bad Sync

```bash
# Abort an in-progress merge or rebase
git merge --abort     # or:  git rebase --abort

# Restore main from the backup branch you created in Step 3
git checkout main
git reset --hard "backup/main-pre-sync-$DATE"

# Find a lost commit via reflog
git reflog
git reset --hard HEAD@{<n>}
```

---

## Troubleshooting

| Issue | Solution |
| --- | --- |
| `upstream` remote missing | `git remote add upstream https://github.com/googleapis/release-please.git` |
| `package-lock.json` reappears after merge | `rm package-lock.json && yarn install && git add yarn.lock .gitignore` (the file is `.gitignore`d — deviation D) |
| `yarn install --immutable` fails post-merge | Upstream changed dependencies. Run plain `yarn install`, inspect the lockfile diff, commit. |
| `yarn install` errors with "package is too recent" / minimum age | A new/bumped dependency is younger than `npmMinimalAgeGate` (`7d`). Wait for it to age in, pin to an older patch, or add it to `npmPreapprovedPackages` in `.yarnrc.yml` with a justification. |
| A dep stops working because it needs a postinstall script | `enableScripts: false` blocks all dependency install scripts. Verify the package genuinely needs the script (most don't); if so, allow it narrowly via `dependenciesMeta` in `package.json` rather than flipping `enableScripts` globally. |
| Tests fail in `src/manifest.ts` or `src/github.ts` after merge | One of the deviation A/B threads was likely dropped during conflict resolution. Re-run the "Verify intentional deviations" checks. |
| CI lint fails on `.github/workflows/ci.yaml` | A `persist-credentials: false` line was probably dropped. Restore from deviation C. |
| Commit signing fails | **STOP.** Per repo policy, surface to the user — do not commit unsigned. |
| Upstream did a force-push / history rewrite | **STOP** and ask the user. Mirroring an upstream rewrite into this fork is almost never correct. |

---

## Updating This Skill

When new fork-only commits land that should be preserved through future syncs, add a new subsection under **Intentional Deviations** with:

1. A short name and description
2. Fork commit hashes (use `git log --oneline upstream/main..origin/main`)
3. Files touched
4. A sanity-check `rg`/`grep` command that proves the deviation survived

Keep the **Verify intentional deviations** block in Step 6 in sync with that list.
