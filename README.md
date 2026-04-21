# Consola Skills

Battle-tested skills for the [Hermes](https://github.com/NousResearch/hermes-agent) community, built by [Consola](https://consola.ai) — an AI Chief of Staff.

These skills are derived from real daily operations, not theory. Each one has been tested in production and refined through actual use.

## Using These Skills

### Quick Install

Add a skill to your Hermes setup:

```bash
# Clone the repo
git clone https://github.com/consola-ai/consola-skills.git

# Copy a skill to your shared skills directory
cp -r consola-skills/skills/email-triage ~/.agents/skills/
```

### One-Line Install (from SKILL.md)

You can also point your agent to our registry skill, which tells it where to find and how to install skills:

```
Load skill from: https://consola.ai/skills/registry
```

Or add to your agent's skill config:

```yaml
skills:
  - url: https://consola.ai/skills/registry
```

## Available Skills

| Skill | Category | Description |
|-------|----------|-------------|
| [email-triage](skills/email-triage/) | Email | Two-stage inbox classification pipeline (whitelist + LLM) |
| [hostmaster-deploy](skills/hostmaster-deploy/) | DevOps | Docker + Caddy + Cloudflare Tunnel + SSO deployment pattern |
| [systemd-timers](skills/systemd-timers/) | DevOps | Scheduled tasks without root access |
| [cron-context](skills/cron-context/) | Agent | Bridge cron job output into interactive sessions |
| [subagent-fix](skills/subagent-fix/) | Agent | Diagnose and repair failed subagent sessions |
| [agent-registry](skills/agent-registry/) | Agent | Multi-agent routing and delegation pattern |
| [skill-creator](skills/skill-creator/) | Agent | Create and validate Hermes skills |
| [sender-rules](skills/sender-rules/) | Email | Config-driven email sender whitelist/blacklist |
| [agent-memory](skills/agent-memory/) | Agent | Cross-harness persistent memory with plain markdown |
| [webhook-setup](skills/webhook-setup/) | DevOps | Event-driven scripting with Hermes webhooks |
| [script-placement](skills/script-placement/) | Conventions | Where to put scripts and why |
| [report-bugs](skills/report-bugs/) | Workflow | Report bugs and features to upstream dependencies |
| [gmail-cli](skills/gmail-cli/) | Email | Gmail CLI operations with HTML reading and attachments |

## Philosophy

- **Battle-tested** — every skill comes from real daily use
- **Self-hosted first** — designed for people who run their own infrastructure  
- **One script, one job** — skills do one thing well
- **Pitfalls documented** — every gotcha we hit is in the skill so you don't have to

## License

MIT — use them however you want. Attribution appreciated but not required.
