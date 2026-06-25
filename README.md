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

### With `npx skills` (recommended — the [skills.sh](https://www.skills.sh/) app)

One command, no clone, works across Claude Code, Cursor, Copilot, Cline, Windsurf, and 20+ agents:

```sh
# Global — available in every project (~/.claude/skills/)
npx skills add MrErikCodes/npm-check-updates-skill -g

# Project-scoped — just this repo (./.claude/skills/)
npx skills add MrErikCodes/npm-check-updates-skill
```

Manage it later:

```sh
npx skills list                       # show installed skills
npx skills remove npm-check-updates   # uninstall
```

### From git (clone & move) — one-liners

**macOS / Linux:**

```sh
git clone https://github.com/MrErikCodes/npm-check-updates-skill.git /tmp/ncu-skill && mkdir -p ~/.claude/skills/npm-check-updates && cp /tmp/ncu-skill/SKILL.md ~/.claude/skills/npm-check-updates/ && rm -rf /tmp/ncu-skill
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/MrErikCodes/npm-check-updates-skill.git "$env:TEMP\ncu-skill"; New-Item -ItemType Directory -Force "$HOME\.claude\skills\npm-check-updates" > $null; Copy-Item "$env:TEMP\ncu-skill\SKILL.md" "$HOME\.claude\skills\npm-check-updates\"; Remove-Item -Recurse -Force "$env:TEMP\ncu-skill"
```

### Download the file only (no clone) — one-liners

**macOS / Linux:**

```sh
mkdir -p ~/.claude/skills/npm-check-updates && curl -fsSL https://raw.githubusercontent.com/MrErikCodes/npm-check-updates-skill/main/SKILL.md -o ~/.claude/skills/npm-check-updates/SKILL.md
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills\npm-check-updates" > $null; Invoke-WebRequest https://raw.githubusercontent.com/MrErikCodes/npm-check-updates-skill/main/SKILL.md -OutFile "$HOME\.claude\skills\npm-check-updates\SKILL.md"
```

Once installed, the agent auto-discovers the skill from its `description` frontmatter and loads it
when you ask to update / upgrade / bump dependencies or mention `npm-check-updates` / `ncu`.

## The tool itself

```sh
npm install -g npm-check-updates   # enables the `ncu` shorthand
npx npm-check-updates              # run without installing (npx needs the LONG name; `npx ncu` won't resolve)
```
