> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: email-triage
version: 5.0.0
description: >
  Inbox Zero email triage via v3 pipeline. Two-stage classification (sender rules + classifier),
  two dispositions (delete, escalate). Escalated emails batched and sent to Consola
  via Hermes webhook. No defer, no direct delegation — Consola is the gateway.
  Script: ~/.agents/scripts/gmail-triage-v3.ts
---

# Email Triage v5 — Delete or Escalate

Every email is either deleted (spam/marketing) or escalated to Consola via webhook.
Consola decides what to surface to the user, what to delegate, and what to handle silently.

---

## Architecture

```
Gmail Inbox
  │
  ├─ Stage 1: Sender rules (senders-v3.conf)
  │   ├─ Blacklist match → delete
  │   └─ Whitelist match → escalate (skip classifier)
  │
  ├─ Stage 2: ORAI content classifier
  │   ├─ delete (spam, marketing, promos)
  │   └─ escalate (everything else)
  │
  └─ Disposition
      ├─ delete → trash + log
      └─ escalate → archive + collect → batch webhook to Consola
```

### Key Design Decisions

1. **No "defer"** — every non-spam email goes to Consola. No Deferred label limbo.
2. **No direct delegation** — sub-agents don't get emails directly. Consola is the gateway/supervisor.
3. **Archive before classify** — Inbox Zero. Every message leaves the inbox immediately.
4. **Batch escalation** — all escalated emails collected, then ONE webhook POST per account run (not one per email).
5. **Lightweight webhook payload** — headers + 150-char preview only. Full body saved to individual JSON files on disk.
6. **Notification IDs** — each run gets a `TRI-XXXX` ID for `/catchup` integration.

---

## Script

**Location:** `~/.agents/scripts/gmail-triage-v3.ts`
**Runtime:** `bun run`

### CLI Options

```
--accounts <list>     Comma-separated (default: personal,business)
--dry-run             Classify but don't delete; use /notify route
--max-messages <N>    Max per account (default: 50)
--model <model>       ORAI classifier model
--webhook-url <url>   Override webhook endpoint
--help                Show help
```

### Running

```bash
# Dry run (Telegram notification, no deletions)
cd ~/.agents/scripts && bun run gmail-triage-v3.ts --dry-run --max-messages 5

# Production (silent processing, auto-delete spam, escalate rest)
cd ~/.agents/scripts && bun run gmail-triage-v3.ts --webhook-url http://127.0.0.1:8644/webhooks/triage
```

### Signal Handling

- **First ^C**: Sets `cancelled` flag, finishes current message, then stops. Sends any collected batch.
- **Second ^C**: Force exit (loses unsent batch).

---

## Sender Rules

**Config:** `~/.agents/skills/email-triage/references/senders-v3.conf`

Unified file for ALL accounts. Format:

```
# from                to                     subject     disposition  rule
*@marketing.*         *                      *           delete       marketing-spam
receipts@example.ai   *                      *           escalate      financial
driver@yourdomain.org *                      "Dispatch*" escalate      dispatch-alerts
```

- Multi-word subjects must be quoted: `"Dispatch*"`
- Wildcards: `*` (any), `@domain` (domain match), `*.domain` (subdomain), `*text*` (contains)
- `from` and `to` match both display name and email address
- Lines starting with `#` are comments

---

## Batch Escalation Payload

### Webhook (lightweight)

Sent to Hermes webhook. Contains headers + 150-char body_preview only:

```json
{
  "notification_id": "TRI-A3K7",
  "dry_run": false,
  "timestamp": "2026-04-20T18:00:00Z",
  "detail_dir": "~/data/email-triage/triage-detail-TRI-A3K7",
  "accounts": [{
    "account": "personal",
    "emails": [{
      "message_id": "19dabc1e00129ff3",
      "from": "receipts@example.ai",
      "subject": "Your receipt [#1750-3493]",
      "date": "Mon, 20 Apr 2026",
      "body_preview": "Thank you for your purchase of $26.38...",
      "stage": "sender-whitelist",
      "rule": "financial",
      "confidence": "high",
      "notes": "whitelist match",
      "detail_file": "~/data/email-triage/triage-detail-TRI-A3K7/19dabc1e00129ff3.json"
    }]
  }],
  "summary": { "total_escalated": 5, "total_deleted": 2, "total_errors": 0 }
}
```

### Detail Files (one per email)

Saved to `~/data/email-triage/triage-detail-TRI-XXXX/<message_id>.json`:

```json
{
  "account": "personal",
  "message_id": "19dabc1e00129ff3",
  "thread_id": "19dabc1e00129ff3",
  "from": "receipts@example.ai",
  "to": "user@gmail.com",
  "subject": "Your receipt [#1750-3493]",
  "date": "Mon, 20 Apr 2026",
  "labels": "INBOX,UNREAD",
  "body_snippet": "<full 2000-char body text>",
  "stage": "sender-whitelist",
  "rule": "financial",
  "confidence": "high",
  "notes": "whitelist match"
}
```

**Important**: Read detail files ONLY for emails you need to act on. Don't read them all upfront.

---

## Notification IDs & /catchup

Each triage run generates a `TRI-XXXX` notification ID. This ID is:

1. Included in the webhook payload (`notification_id` field)
2. Printed at the end of the script output
3. Included in the Telegram notification footer: `📋 Notification ID: TRI-XXXX — use /catchup TRI-XXXX to review`
4. Used to name the context file: `~/.hermes/cron/context/triage-TRI-XXXX.md`

User can use `/catchup TRI-XXXX` to pull up a specific notification's context.

---

## Webhook Routes

Configured in `~/.hermes/config.yaml` under `platforms.webhook.routes`:

| Route | Purpose | Deliver | Auth |
|-------|---------|---------|------|
| `/webhooks/test` | Testing | log | INSECURE_NO_AUTH |
| `/webhooks/triage` | Production triage | telegram (with deliver_extra.chat_id) | INSECURE_NO_AUTH |
| `/webhooks/notify` | Ad-hoc notifications | telegram (with deliver_extra.chat_id) | INSECURE_NO_AUTH |

### Critical: deliver_extra.chat_id

The `deliver` field only accepts platform names (`telegram`, `discord`, etc.) — NOT `telegram:123456`.
To target a specific chat, use `deliver_extra.chat_id`:

```yaml
deliver: telegram
deliver_extra:
  chat_id: "8782255778"
```

Without this, the gateway falls back to `TELEGRAM_HOME_CHANNEL` from `.env`, which MUST be a numeric
chat ID (not a username) — see upstream bug #13206.

---

## Data Paths

| Data | Path |
|------|------|
| Script | `~/.agents/scripts/gmail-triage-v3.ts` |
| Sender rules | `~/.agents/skills/email-triage/references/senders-v3.conf` |
| Triage logs | `~/data/email-triage/logs/v3-<account>-<date>.ndjson` |
| Detail files | `~/data/email-triage/triage-detail-TRI-XXXX/<msgid>.json` |
| Context files | `~/.hermes/cron/context/triage-TRI-XXXX.md` |
| Lock file | `~/data/email-triage/gmail-triage-v3.lock` |
| Scratch dir | `~/data/email-triage/scratch/` |

---

## Log Format

NDJSON, one line per email action:

```json
{
  "timestamp": "2026-04-20T18:00:32Z",
  "version": "v3",
  "mode": "dry-run",
  "account": "personal",
  "message_id": "19dabc1e00129ff3",
  "from": "receipts@example.ai",
  "subject": "Your OpenRouter receipt",
  "classification_stage": "sender-whitelist",
  "disposition": "escalate",
  "confidence": "high",
  "rule": "financial",
  "model": "sender-config",
  "notes": "whitelist match",
  "action_taken": "[dry-run] queued for escalation"
}
```

---

## Pitfalls

1. **TELEGRAM_HOME_CHANNEL must be numeric** — usernames cause `int()` crash (upstream bug #13206). Check `.env` if webhook deliveries silently fail.
2. **One webhook per batch, not per email** — if you see individual POSTs, the script hasn't been updated.
3. **Gmail API rate limits** — fetching one message at a time in a loop can cause FETCH FAILED errors on large inboxes.
4. **Multi-word subjects in senders-v3.conf must be quoted** — unquoted `Dispatch #1234` gets split into multiple fields.
5. **`deliver: telegram:123456` doesn't work** — the gateway matches against a hardcoded platform name list. Use `deliver_extra.chat_id` instead.
6. **Webhook secrets are currently INSECURE_NO_AUTH** — replace with real secrets before production exposure.
7. **LLM prompts are unreliable for mandatory side-actions** — asking the webhook agent to `write_file` (for /catchup context) as part of its response doesn't reliably happen, no matter how forceful the prompt. The agent delivers to Telegram and skips the file write. For critical path actions, don't rely on LLM prompt compliance — use deterministic code (the script) or post-delivery hooks instead.
8. **Gateway `--replace` exit codes are misleading** — `exit-code=1` in systemd logs is normal when using `gateway run --replace` (old process killed, new starts). Check `systemctl --user is-active` for actual status.

---

## Upstream Bugs

- [2026-04-20] hermes-agent: webhook delivery crashes when TELEGRAM_HOME_CHANNEL is a username — https://github.com/NousResearch/hermes-agent/issues/13206 — workaround: use numeric chat ID + deliver_extra.chat_id
- [2026-04-18] hermes-agent: cron→session context gap — https://github.com/NousResearch/hermes-agent/issues/8793 — workaround: cron writes to stable file, /catchup reads it
