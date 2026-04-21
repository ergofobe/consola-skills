> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: script-placement
version: 1.0.0
description: >
  Where to put scripts on this system and why. Covers the directory conventions,
  naming rules, and decision framework for new scripts.
metadata:
  hermes:
    tags: [scripts, conventions, filesystem, devops]
    triggers:
      - where should I put this script
      - new script
      - script placement
      - where does this go
---

# Script Placement Guide

## Decision Tree

```
Is it tied to a specific app's config/auth?
  YES → ~/.config/<app>/bin/           (e.g., gws OAuth wrappers)
  NO ↓

Is it a skill-specific utility (only used by one skill)?
  YES → ~/.agents/skills/<skill>/scripts/   (bundled with skill)
  NO ↓

Is it an agent automation (triage, sensors, processors, cron jobs)?
  YES → ~/.agents/scripts/             (this is the default for most new scripts)
  NO ↓

Is it a project/workspace-specific script?
  YES → ~/workspaces/<project>/scripts/
  NO ↓

Is it a system service or daemon?
  YES → ~/bin/ or /usr/local/bin/      (on PATH, referenced by systemd)
  NO ↓

Default → ~/.agents/scripts/
```

## Directories

| Directory | Purpose | Examples | Git |
|---|---|---|---|
| `~/.agents/scripts/` | Agent operational scripts — automation, sensors, cron jobs, processors | `gmail-triage-v3.ts` | Yes (part of `~/.agents/` repo) |
| `~/.config/<app>/bin/` | App-specific CLI wrappers tied to that app's config/auth | `gws-personal`, `gmail-read` (tied to OAuth in `~/.config/gws/`) | No |
| `~/.agents/skills/<skill>/scripts/` | Utilities bundled with and only used by a single skill | A skill's helper script | Yes (part of skill) |
| `~/workspaces/<project>/scripts/` | Project-specific build/deploy utilities | Deploy scripts for a specific repo | Per-project |
| `~/bin/` | Personal CLI tools on PATH | System-wide utilities | No |
| `/usr/local/bin/` | System-installed binaries | `bun`, `gws` (package-managed) | No |

## Rules

1. **Auth-tied scripts stay with their config.** If a script needs `~/.config/foo/` to exist for OAuth/credentials, it lives in `~/.config/foo/bin/`. Moving it breaks the auth path.

2. **Agent automations go in `~/.agents/scripts/`.** If it's run by a systemd timer, cron job, or agent webhook, it belongs here. This is the **default** for new scripts.

3. **Don't couple unrelated concerns.** A triage sensor shouldn't contain package tracking logic. A monitoring script shouldn't contain filing logic. One script, one job.

4. **Shared data goes in `~/.agents/data/`.** Scripts read/write data under `~/.agents/data/<feature>/`. This is gitignored.

5. **Systemd services reference the script path.** When moving a script, always update the corresponding `.service` file in `~/.config/systemd/user/` and run `systemctl --user daemon-reload`.

6. **Skill references must be updated too.** When moving a script, grep both `~/.agents/skills/` and `~/.hermes/skills/` for the old path and update all references.

7. **Git commit after moves.** `~/.agents/scripts/` is tracked in the `~/.agents/` repo. Commit after adding/moving scripts.

## Naming Conventions

- **Kebab-case**: `gmail-triage-v3.ts`, `package-tracker.ts`
- **Version suffix for iterations**: `gmail-triage-v3.ts` (not `gmail-triage-final-v2-really.ts`)
- **Descriptive prefix**: If it processes email, starts with `gmail-` or `email-`
- **Language extension always included**: `.ts` for Bun/TypeScript, `.py` for Python, `.sh` for shell

## Pitfalls

- **Never put a new git repo inside `~/.agents/`.** The whole directory is one repo. Adding nested `.git/` directories causes confusion. Use `git rm --cached` if accidentally added.
- **Don't mix gmail CLI wrappers with agent scripts.** The `~/.config/gws/bin/` scripts are thin OAuth wrappers — they depend on `~/.config/gws/{personal,business}/` existing. They are NOT agent scripts.
- **Moving a script = updating N+1 references.** Count: systemd service, skill files (both `~/.agents/` and `~/.hermes/` copies), memory, and any cron jobs that reference the old path.
