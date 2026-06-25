# npm-check-updates skill

A reusable [Claude Code / Agent skill](https://agentskills.io/specification) that turns
[`npm-check-updates`](https://www.npmjs.com/package/npm-check-updates) (`ncu`) into a **safe**
dependency-upgrade workflow.

Instead of blindly running `ncu -u`, the skill instructs the agent to:

1. Detect the package manager and list what's outdated, splitting majors from patch/minor.
2. Categorize each update (patch / minor / major / 0.x — pre-1.0 minors are treated as breaking).
3. **Research breaking changes** for every major/0.x bump via GitHub releases, the compare view,
   migration guides, the npm page, and open/recent issues — never from memory.
4. Check whether **our** code actually uses the changed APIs (grep the codebase) and verify
   peer-dependency compatibility (`ncu --peer`).
5. Upgrade safe-first in batches (majors one at a time, or `ncu --doctor -u` to auto-isolate breakage).
6. Verify with install + build + tests after every batch.
7. Report a **per-package verdict** (✅ Safe / ⚠️ Needs code change / ⛔ Hold) with the exact code
   changes required and the build/test result.

## Install

Copy the skill into your user-level skills directory as `npm-check-updates/`:

```sh
# macOS / Linux
mkdir -p ~/.claude/skills/npm-check-updates
cp SKILL.md ~/.claude/skills/npm-check-updates/

# Windows (PowerShell)
New-Item -ItemType Directory -Force "$HOME\.claude\skills\npm-check-updates"
Copy-Item SKILL.md "$HOME\.claude\skills\npm-check-updates\"
```

The agent auto-discovers it from the `description` frontmatter and loads it when you ask to
update / upgrade / bump dependencies or mention `npm-check-updates` / `ncu`.

## The tool itself

```sh
npm install -g npm-check-updates   # enables the `ncu` shorthand
npx npm-check-updates              # run without installing (npx needs the LONG name; `npx ncu` won't resolve)
```
