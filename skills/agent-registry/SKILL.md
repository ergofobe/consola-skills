> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: agent-registry
version: 1.0.0
description: >
  Canonical registry of all agents in the organizational ecosystem.
  Contains the agent map, domain-to-agent routing, and the boot-skill
  delegation pattern. Load this skill whenever you need to delegate
  a task to another agent.
metadata:
  hermes:
    tags: [agents, delegation, routing, registry]
    triggers:
      - delegate task
      - agent registry
      - route to agent
---

# Agent Registry

This is the single source of truth for all agents, their domains, and
their base skill locations. Any skill or agent that needs to delegate
work loads this file.

---

## Agents

```yaml
ollie:
  entity: Business Logistics Co
  domains: [logistics.example.com]
  skill: ~/workspaces/logistics/.agents/skills/ollie/SKILL.md
  toolsets: ["terminal", "file", "web"]

obi:
  entity: Business Group (umbrella organization, not a legal entity)
  domains: [business.group]
  skill: ~/workspaces/businessgroup/.agents/skills/obi/SKILL.md
  toolsets: ["terminal", "file", "web"]

og:
  entity: Business Holdings (all jurisdictions — US, Colombia, Panama, etc.)
  domains: [holdings.example.org]
  skill: ~/workspaces/holdings/.agents/skills/og/SKILL.md
  toolsets: ["terminal", "file", "web"]

patty:
  entity: PTYcoin
  domains: [ptycoin.example, panamabitcoins.example]
  skill: ~/workspaces/ptycoin/.agents/skills/ptycoin/SKILL.md
  toolsets: ["terminal", "file", "web"]

oforica:
  entity: Oforica
  domains: [oforica.example]
  skill: ~/workspaces/oforica/.agents/skills/oforica/SKILL.md
  toolsets: ["terminal", "file", "web"]

polly:
  entity: Personal (User)
  domains: []
  accounts: [user@gmail.com]
  skill: ~/workspaces/personal/.agents/skills/polly/SKILL.md
  toolsets: ["terminal", "file", "web"]
```

### Organizational Structure

```
Business Group (umbrella — Obi)
├── Business Holdings US
│   ├── Business Logistics Co (Ollie)
│   └── Oforica (Oforica)
├── Business Holdings Colombia
│   └── (future subsidiaries)
├── Business Holdings Panama
│   ├── PTYcoin (Patty)
│   └── (future subsidiaries)
└── Consola — AI Chief of Staff (cross-cutting orchestrator)
```

Business Holdings is one agent (Og) handling all jurisdictions. Pass the
jurisdiction as context when delegating — do not try to route to
jurisdiction-specific agents.

### Shared Skills

Common skills that any agent can load live at `~/.agents/skills/`.
Each agent's base skill is responsible for knowing when and how to
load the relevant shared skill. The orchestrator passes a category
tag; the agent's base skill handles the rest.

```
~/.agents/skills/
├── agent-registry/SKILL.md       ← this file
├── newsletter-digest/SKILL.md
├── shipment-tracking/SKILL.md
└── (future shared skills)
```

---

## Routing

### General-purpose routing

When the user asks you to do something (via Telegram, chat, or any non-email
trigger), determine the right agent using these rules. Work top to
bottom; stop at the first match.

| # | Signal | Route to |
|---|---|---|
| 1 | User names the agent directly ("tell Agent A to…", "ask Agent D about…") | Named agent |
| 2 | User names or implies the entity ("for Business Logistics Co…", "PTYcoin needs…") | Agent for that entity (see agent table) |
| 3 | User references a domain ("renew example.com", "check business.group DNS") | Agent whose `domains` list includes that domain |
| 4 | Task is clearly personal ("my bank account", "schedule my dentist", "my truck maintenance") | Polly |
| 5 | Task spans multiple entities or is org-level ("group-wide policy", "cross-entity reporting") | Obi |
| 6 | Ambiguous — could belong to more than one agent | Ask user |

### Email routing

When routing emails from the triage pipeline, the **recipient domain**
is the primary routing key — check `to`, `cc`, and `delivered-to`
headers.

**Business inbox (GWS shared account):**

All businesses share one Google Workspace inbox with domain aliases.
Look up the recipient domain below.

| Recipient domain | Agent |
|---|---|
| `logistics.example.com` | Ollie |
| `business.group` | Obi |
| `holdings.example.org` | Og |
| `ptycoin.example` | Patty |
| `panamabitcoins.example` | Patty |
| `oforica.example` | Oforica |

If the recipient domain is not in this table → surface to user. Do not
guess.

Emails from user's personal address (`user@gmail.com`) appearing in
this inbox: route by recipient domain as above. These are user-as-driver
or user-as-personal communicating with a business entity.

**Personal inbox (`user@gmail.com`):**

All emails route to **Polly**, regardless of sender. This includes
emails from business domains (e.g., `*@logistics.example.com`). These
are driver-facing or personal-facing documents that have already been
processed on the business side. Polly files and labels them.

---

## Boot-Skill Delegation Pattern

Sub-agents start with a completely fresh conversation. They have zero
knowledge of the parent session, prior tool calls, or conversation
history. The only context they receive is what you pass in `goal` and
`context`.

### Template

```python
delegate_task(
    goal=(
        "Read your base skill at {agent.skill} and follow its instructions. "
        "Then perform this task: {task_description}\n"
        "\n"
        "Category: {category_tag}  # optional, for email triage"
    ),
    context=(
        "{Everything the sub-agent needs to do this specific task.\n"
        " Include file paths, identifiers, relevant data, and any\n"
        " other detail. Do not assume the sub-agent knows anything.}"
    ),
    toolsets={agent.toolsets}
)
```

### Rules

- The `goal` MUST start with the instruction to read the base skill.
- The `context` MUST be self-contained.
- Sub-agents cannot: delegate further, send messages, access memory,
  interact with the user, or execute code blocks.
- Use paths and toolsets from the agent table above. Do not hardcode
  paths elsewhere.
- When delegating to Og, include the jurisdiction (US, Colombia,
  Panama) in the context so Og knows which entity is involved.
