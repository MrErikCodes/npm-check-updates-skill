---
name: npm-check-updates
description: Use when updating, upgrading, or bumping project dependencies, checking for outdated npm/yarn/pnpm/bun packages, or when the user mentions npm-check-updates or ncu — before changing any package versions in package.json.
---

# npm-check-updates (ncu): Safe Dependency Upgrades

## Overview

`npm-check-updates` (`ncu`) rewrites the version ranges in `package.json` to newer releases. It **ignores** your existing semver constraints, so a blanket `ncu -u` *will* pull in breaking major versions. The tool is the easy part.

Your job is not to run `ncu -u` and declare success. It is to: **find what's outdated → research what breaks → check whether our code uses the broken surface → prove the project still builds and tests pass → report a verdict per package.**

**Core rule:** Never bump a major version (or a 0.x minor) without reading its changelog/release notes and checking whether our code touches the changed API. Never claim an upgrade is "safe" or "fine" without actually running install + build + tests and seeing them pass.

## The workflow

1. **Start clean.** Confirm `git status` is clean (or stash) so the upgrade diff is isolated and revertible. Detect the package manager from the lockfile: `package-lock.json`→npm, `yarn.lock`→yarn, `pnpm-lock.yaml`→pnpm, `bun.lockb`→bun. Pass `-p <pm>` to ncu if not npm. Monorepo? add `-ws` (workspaces) or `--deep`.
2. **See what's outdated.** Run `ncu` to list every dependency with a newer version (`current range → latest`). Also run `ncu --format group` (or `--target minor`) so majors are separated from patch/minor at a glance — anything in `latest` but not in `minor` is a major bump.
3. **Categorize** every update as patch / minor / major / 0.x (see *Categorize every update*).
4. **Research breaking changes** for every major and every 0.x bump *before* touching it (see *Researching breaking changes*). Pure patch/minor on a ≥1.0 package can skip deep research — but still verify with tests.
5. **Check our usage.** A breaking change only matters if we use the affected API. Grep/search the codebase for the imported symbols, options, or config keys that changed. Check peer-dependency compatibility with `ncu --peer`.
6. **Upgrade in batches, safe first.**
   - **Safe batch:** `ncu -u --target minor` (patch + minor) → install → build → test.
   - **Each major:** upgrade it alone — `ncu -u -f <pkg>` — apply any required code changes, then install → build → test, then move to the next major. One at a time so you always know what broke.
   - Or let **doctor mode** isolate breakage automatically (see *Doctor mode*).
7. **Verify, every batch.** Run install (`npm/yarn/pnpm/bun install`), then typecheck/build, then the test suite. Failing tests = stop and investigate; do not proceed or claim success.
8. **Report** a verdict per package (see *Reporting*), including exact code changes required and the build/test result.

## Quick reference

| Goal | Command |
|------|---------|
| Install globally (enables the `ncu` shorthand) | `npm i -g npm-check-updates` |
| Run without installing | `npx npm-check-updates` — with npx **only the long name works**; `npx ncu` won't resolve |
| List all available updates | `ncu` |
| Group output by patch/minor/major | `ncu --format group` |
| List only non-major (patch+minor) | `ncu --target minor` |
| List only patch | `ncu --target patch` |
| Show repo links + publish dates | `ncu --format repo,time` |
| Upgrade package.json (all) | `ncu -u` |
| Upgrade one package / a pattern | `ncu -u -f react` · `ncu -u -f "/^@types\/.*/"` |
| Exclude a package | `ncu -u -x webpack` |
| Pick interactively, grouped | `ncu -i --format group` |
| Filter updates to compatible peer deps | `ncu --peer` |
| Auto-test each upgrade, revert breakers | `ncu --doctor -u` |
| Custom test/install for doctor | `ncu --doctor -u --doctorTest "npm test" --doctorInstall "npm ci"` |
| Non-npm / monorepo | `ncu -p pnpm` · `ncu -ws` · `ncu --deep` |

`ncu` only edits `package.json` — it does **not** install. You must run your package manager's install afterward to update the lockfile and pull the new code.

## Categorize every update

- **patch** `1.2.X` — bug fixes. Safe; verify with tests.
- **minor** `1.X.0` — new features, backward-compatible by semver. Usually safe; skim release notes, verify.
- **major** `X.0.0` — intentional breaking changes. **Always research.**
- **0.x packages** `0.X.0` — pre-1.0, semver allows breaking changes in *minor* bumps. Treat any `0.x` minor bump like a major and research it. Don't assume `ncu --target minor` keeps you safe here.

## Researching breaking changes

For each major / 0.x bump, **read the actual release notes — do not rely on memory** (your knowledge is stale and changelogs change). Use `WebFetch`/`WebSearch`, or `gh release view` / `gh api` for GitHub repos. `ncu --format repo` prints each package's repo URL.

Look, roughly in this order:

1. **CHANGELOG.md / GitHub Releases** — read every release note between our current version and the target. GitHub: `github.com/<owner>/<repo>/releases` and the compare view `…/compare/v<old>...v<new>`.
2. **Migration / upgrade guide** — many projects ship `MIGRATING.md`, an "Upgrading to vX" docs page, or a codemod.
3. **npm page** — `npmjs.com/package/<name>` for README, deprecation banners, and maintainer/owner changes (`ncu --format ownerChanged` flags those).
4. **Open & recent issues** — search the repo's issues for the target version: regressions, broken installs, peer-dep conflicts others already hit. A major released days ago with a stack of bug reports = wait.

**Extract specifically:** removed/renamed APIs, changed defaults, changed config/option shapes, raised minimum Node/engine version, new or bumped required peer deps, and CJS→ESM-only switches (a common silent breaker).

## Doctor mode — automated breakage isolation

`ncu --doctor -u` runs your tests, upgrades all dependencies, and re-runs tests; if they fail it **reverts and re-tests each upgrade individually** to pinpoint exactly which package broke the build. Customize with `--doctorTest "npm run test"` and `--doctorInstall "npm ci"`. It needs a working test command. Use it to confirm your impact analysis empirically — but you still own researching majors and reporting the verdict.

## Reporting

Give the user a per-package verdict, not just a diff:

| Package | Current → Target | Level | Breaking changes | We use it? | Verdict |
|---------|------------------|-------|------------------|------------|---------|
| lodash | 4.17.21 → 4.17.21 | patch | none | yes | ✅ Safe |
| react | 17.0.2 → 18.3.1 | major | new root API, automatic batching | yes (ReactDOM.render) | ⚠️ Needs code change — switch to `createRoot` |
| some-lib | 2.4.0 → 3.0.0 | major | ESM-only, drops Node 16 | yes | ⛔ Hold — we're on Node 16 / CJS |

Then state plainly: **what you upgraded**, **what you held back and why**, **the exact code changes required**, and the **result of install + build + tests** (paste the outcome). Verdict legend: ✅ Safe · ⚠️ Needs code change (give the change) · ⛔ Hold (give the reason).

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `ncu -u` then committing without installing | ncu only edits `package.json`. Run install to update the lockfile and get the code, then test. |
| Claiming "tests pass" without running them | Run install + build + test and report the actual outcome before saying safe. |
| Treating 0.x minor bumps as safe | Pre-1.0, minor = breaking. Research them like majors. |
| Ignoring peer dependencies | Use `ncu --peer`; a major can demand a peer your stack can't satisfy yet. |
| Upgrading everything at once, then can't tell what broke | Safe batch first; majors one at a time, or use `--doctor`. |
| Trusting memory for changelogs | Fetch the real release notes/issues. Your training cutoff is behind the latest releases. |
| Forgetting monorepo workspaces | Use `-ws`/`--deep`; otherwise only the root `package.json` is touched. |
