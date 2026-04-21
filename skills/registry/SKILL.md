> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: registry
description: >
  Master index of all available Consola Skills. Lists every skill with a one-line
  description and install instructions. Triggers: "list skills", "what skills are available",
  "show me all skills", "skill registry", "how do I install a skill".
---

# Consola Skills Registry

Master index of all battle-tested Hermes agent skills.

## Available Skills

| Skill | Description |
|-------|-------------|
| [agent-memory](./agent-memory/) | Cross-harness persistent memory using plain markdown files |
| [agent-registry](./agent-registry/) | Canonical registry of all agents, domain routing, and delegation patterns |
| [catchup](./cron-context/) | Load pending cron job output into current session; bridges cron→chat gap |
| [cron-context](./cron-context/) | Alias for catchup skill |
| [email-triage](./email-triage/) | Inbox Zero email triage via two-stage classification pipeline |
| [gmail-cli](./gmail-cli/) | Unified Gmail operations: read, send, archive, trash, label across multiple accounts |
| [hostmaster-deploy](./hostmaster-deploy/) | Deploy Docker services behind Cloudflare→Caddy→Authentik stack |
| [proactively-fix-subagents](./subagent-fix/) | Inspect sub-agent sessions, fix skill files to prevent recurring failures |
| [report-bugs](./report-bugs/) | Report upstream bugs with high-quality issues and track workarounds |
| [script-placement](./script-placement/) | Where to put scripts and why — directory conventions and naming rules |
| [sender-block](./sender-rules/) | Manage email triage sender rules (block/whitelist senders) |
| [sender-rules](./sender-rules/) | Alias for sender-block skill |
| [skill-creator](./skill-creator/) | Create, validate, and refine Agent Skills following agentskills.io spec |
| [subagent-fix](./subagent-fix/) | Alias for proactively-fix-subagents skill |
| [systemd-timers](./systemd-timers/) | Create and manage systemd user timers for tasks that survive gateway restarts |
| [webhook-setup](./webhook-setup/) | Setup guide for Hermes webhook platform — routes, templates, delivery modes |

## Install Instructions

### For Hermes Agent

Skills are discovered from `~/.agents/skills/` by default. To install a skill:

```bash
# Clone the skills repo
git clone https://github.com/oberon-logistics/consola-skills.git ~/src/consola-skills

# Link individual skills or copy them
cp -r ~/src/consola-skills/skills/<skill-name> ~/.agents/skills/

# Or add the entire skills directory to your Hermes config external_dirs
```

### For Claude Code

```bash
# Clone and link
git clone https://github.com/oberon-logistics/consola-skills.git ~/.claude/consola-skills
ln -s ~/.claude/consola-skills/skills/* ~/.claude/skills/
```

### For opencode / other harnesses

```bash
git clone https://github.com/oberon-logistics/consola-skills.git ~/.agents/skills
```

## Skill Structure

```
skill-name/
├── SKILL.md              # Required: metadata + instructions
├── scripts/              # Optional: executable code
├── references/           # Optional: detailed documentation
└── assets/               # Optional: templates, resources
```

## Contributing

See the [skill-creator](./skill-creator/) skill for how to create new skills following the agentskills.io specification.

## License

These skills are MIT licensed. See individual skill files for details.
