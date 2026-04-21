> **Part of [Consola Skills](https://github.com/ergofobe/consola-skills) — battle-tested Hermes agent skills**

---
name: cron-notify
version: 1.0.0
description: >
  Prepare cron job output for later pickup by the interactive agent.
  The writer half of the cron-context bridge — teaches cron jobs how
  to write context files and notification footers so catchup can find them.
metadata:
  hermes:
    tags: [cron, context, notify, bridge, writer]
    triggers:
      - cron notify
      - cron prompt
      - notification context
---

# Cron Notify — Context File Writer

The writer half of the cron-context bridge. When creating or running scheduled
jobs (cron, systemd timers, webhooks), this skill ensures the job writes its
output to a shared context directory so the interactive agent can pick it up
later via the `cron-context` (catchup) skill.

Solves the upstream gap where cron job deliveries aren't visible to the
interactive agent session ([#8793](https://github.com/NousResearch/hermes-agent/issues/8793)).

## How It Works

1. **Cron job runs** and generates output (briefing, triage summary, etc.)
2. **Cron job writes** a context file to the shared directory alongside its
   normal chat delivery
3. **Notification footer** tells the user how to bring it into context
4. **Interactive agent** uses `cron-context` (catchup) to read and clean up

## Context Directory

```
~/.agents/cron/context/
```

- Created automatically if missing
- One file per job slug — each run overwrites the previous
- If a notification ID is available, filename includes it for targeted lookups

## Slug Format

Derived from the job name: lowercase, spaces → hyphens, non-alphanumeric removed.

| Job Name | Slug |
|----------|------|
| Morning Briefing | `morning-briefing` |
| Email Triage | `triage` |
| Nightly Git Auto-Commit | `nightly-git-commit` |

## Context File Format

Each file is Markdown with a minimal header:

```markdown
# Cron Context: <Job Name>

- **Job:** <job-name>
- **Slug:** <slug>
- **Notification ID:** <notification-id> (if applicable, e.g., TRI-A3K7)
- **Delivered:** <ISO timestamp>
- **Status:** ok | error

---

<Full output of the cron job — the same content delivered to chat>
```

### File Naming

- **With notification ID:** `~/.agents/cron/context/<slug>-<notification-id>.md`
  (enables targeted `/catchup <id>` lookups)
- **Without notification ID:** `~/.agents/cron/context/<slug>.md`

## Cron Prompt Template

When creating or updating a cron job, append this to the agent's prompt:

```
After generating your response, also write the full response text to the cron
context file so the interactive agent can pick it up later:

1. Write to: ~/.agents/cron/context/<slug>-<notification-id>.md
   (If no notification_id in the payload, use ~/.agents/cron/context/<slug>.md)
2. Format:
   # Cron Context: <Job Name>

   - **Job:** <job-name>
   - **Slug:** <slug>
   - **Notification ID:** <notification-id>
   - **Delivered:** <current ISO timestamp>
   - **Status:** ok

   ---

   <your full response>

3. After writing, append this line to your delivered response:
   "📋 Notification ID: <notification-id> — use /catchup <notification-id> to review"
```
   If no notification ID, append: "_Use catchup to bring this into context._"
```

Replace `<slug>` with the job name lowercased, spaces replaced with hyphens,
non-alphanumeric characters removed.

## Notification ID Format

Notification IDs follow the pattern `XXX-YYYY` where:
- `XXX` is a 3-letter prefix identifying the source (e.g., `TRI` for triage)
- `YYYY` uses uppercase letters and digits, excluding I, O, 0, 1 for readability

These IDs are:
- Included in webhook payloads as `notification_id`
- Appended to agent responses for user reference
- Used for targeted `/catchup <id>` lookups

## SOUL.md Integration

Add to your agent's SOUL.md:

> **Cron context:** Scheduled jobs may deliver notifications you can't see in
> session history. If the user references something you lack context for, check
> `~/.agents/cron/context/` — recent cron output may be waiting there. The user
> can also say /catchup to load it explicitly.

## Pitfalls

- **Don't skip writing the context file.** The notification alone isn't enough —
  the interactive agent needs the file to load the full output.
- **Always include the ISO timestamp.** This lets catchup detect stale files (>24h).
- **Overwrite, don't append.** Each run replaces the previous file for that slug.
  This prevents stale data from accumulating.
- **If the cron job fails, write status: error.** Don't skip the file — write it
  with the error so catchup can surface it.

## Companion Skill

This is the **writer** half. The **reader** half is `cron-context` (catchup),
which loads and cleans up the context files.

## Upstream

Once [NousResearch/hermes-agent#8793](https://github.com/NousResearch/hermes-agent/issues/8793)
lands (cron output automatically injected into session history), both halves of
this bridge become obsolete and should be removed.
