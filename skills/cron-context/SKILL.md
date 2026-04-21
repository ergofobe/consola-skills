> **Part of [Consola Skills](https://github.com/consola-ai/consola-skills) — battle-tested Hermes agent skills**

---
name: cron-context
version: 2.0.0
description: >
  Load pending cron job output into the current session. The reader half
  of the cron-context bridge — discovers, reads, summarizes, and cleans up
  context files written by cron-notify.
metadata:
  hermes:
    tags: [cron, context, catchup, session, bridge, reader]
    triggers:
      - /catchup
      - cron context
      - bring that into context
---

# Cron Context (Catchup) — Context File Reader

The reader half of the cron-context bridge. Cron jobs deliver output to the
user's chat, but the interactive agent session doesn't see those deliveries.
This skill reads the context files written by `cron-notify` and brings them
into the active session.

Solves the upstream gap where cron job deliveries aren't visible to the
interactive agent session ([#8793](https://github.com/NousResearch/hermes-agent/issues/8793)).

## How It Works

1. **Cron jobs write context files** via the `cron-notify` skill
2. **User says catchup** (or the agent checks proactively)
3. **Agent reads** all files in the context directory
4. **Agent summarizes** the content into the conversation
5. **Agent deletes** the files (they're now in conversation history)

## Context Directory

```
~/.agents/cron/context/
```

Files follow the naming pattern `<slug>.md` or `<slug>-<notification-id>.md`.

## /catchup Procedure

When the user invokes `catchup` or says something like "bring that cron into context":

### Without a notification ID (plain `catchup`)

1. List files in `~/.agents/cron/context/`
2. If empty, say "No pending cron context — everything's current."
3. For each file, read it and note the job name, timestamp, notification ID, and content
4. Summarize in the conversation: "Loaded cron context from N job(s):"
   - `<notification-id>` — `<job-name>` (delivered <relative time>)
   - Brief one-line summary of each
5. Ask if the user wants details on any specific one
6. Delete all files in the context directory (they're now in conversation history)
7. If any file is >24 hours old, note it as stale before deleting

### With a notification ID (e.g., `catchup TRI-A3K7`)

1. List files in `~/.agents/cron/context/`
2. Search for the file containing that notification ID
3. If found, read and present that file's content in full
4. Delete only that file (leave other context files untouched)
5. If not found, say "No pending context for ID `<id>`. Available:" and list all files

## Proactive Check

If the user references something the agent lacks context for (a list, a report,
a notification), check the context directory proactively — recent cron output
may be waiting there.

## Context File Format

Files are written by `cron-notify` in this format:

```markdown
# Cron Context: <Job Name>

- **Job:** <job-name>
- **Slug:** <slug>
- **Notification ID:** <notification-id>
- **Delivered:** <ISO timestamp>
- **Status:** ok | error

---

<Full output of the cron job>
```

## Pitfalls

- **Always delete after reading.** Files are now in conversation history —
  leaving them causes stale data on next catchup.
- **Note stale files.** If a file is >24 hours old, flag it before deleting.
  The cron job may have failed to clean up.
- **Don't use leading slash in triggers.** `/catchup` is a system command in
  some platforms; the natural language "catchup" works better.
- **Don't skip error status files.** If status is "error", surface the error
  to the user — it means a cron job failed.

## Companion Skill

This is the **reader** half. The **writer** half is `cron-notify`, which
teaches cron jobs how to prepare and write context files with the correct
format and notification footers.

## Upstream

Once [NousResearch/hermes-agent#8793](https://github.com/NousResearch/hermes-agent/issues/8793)
lands (cron output automatically injected into session history), both halves of
this bridge become obsolete and should be removed.
