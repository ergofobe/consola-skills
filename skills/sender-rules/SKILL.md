> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: sender-block
version: 1.0.0
description: >
  Shared skill for managing the email triage sender rules. Any agent can
  request a sender be blocked (auto-delete) or whitelisted (always escalate).
  Sub-agents request changes through their response to Consola. Consola
  edits the config file directly.
metadata:
  hermes:
    tags: [email, triage, sender, block, whitelist]
    triggers:
      - block sender
      - whitelist sender
      - sender rules
---

# Sender Block / Whitelist Management

The email triage pipeline (`gmail-triage-v3.ts`) uses a two-stage
classification system. Stage 1 checks sender rules before any AI
classification happens. Matching a rule is instant and deterministic —
it's the most reliable way to ensure a sender is always deleted or
always escalated.

---

## Config File

**Path:** `~/.agents/skills/email-triage/references/senders-v3.conf`

This is a single unified file covering all accounts. Account scoping
is done via the `to` field.

### Format

Each line is a rule. Fields are whitespace-separated:

```
from  to  subject  disposition  rule-name
```

| Field | Description |
|---|---|
| `from` | Sender pattern (email or glob) |
| `to` | Recipient pattern (`*` for any, or specific domain/address) |
| `subject` | Subject pattern (`*` for any, or glob) |
| `disposition` | `delete` = auto-trash, `escalate` = always send to Consola |
| `rule-name` | Short identifier (no spaces) — describes why this rule exists |

For multi-word subject patterns, wrap in quotes: `"weekly digest"`

### Glob Patterns

| Pattern | Matches |
|---|---|
| `*` | Anything |
| `*.domain.com` | Any sender at domain.com or subdomains |
| `*keyword*` | Contains keyword anywhere |
| `@domain.com` | Any address at domain.com |
| `exact@email.com` | Exact match only |

### Examples

```conf
# Block all email from a spammy sender (any account, any subject)
spammer@example.com  *  *  delete  spam

# Block a domain across all accounts
*.marketingco.com  *  *  delete  marketing-spam

# Block a sender only for the personal account
promo@example.com  user@gmail.com  *  delete  promo

# Block by subject pattern across all accounts
*  *  "act now"  delete  subject-spam

# Whitelist a sender so they always reach Consola
important@vendor.com  *  *  escalate  vendor-critical

# Whitelist freight dispatches (always escalate, never auto-delete)
*@freight.example.com  *@logistics.example.com  *  escalate  freight-all
```

Lines starting with `#` are comments. Blank lines are ignored.

---

## For Sub-Agents

You **cannot** edit the sender rules file. If you encounter an email
that should be blocked or whitelisted in the future, include a
recommendation in your response to Consola:

```
SENDER RULE REQUEST:
  Action: block (or whitelist)
  From: sender@example.com (or *.domain.com)
  To: * (or specific recipient if scoped)
  Subject: * (or pattern)
  Reason: brief explanation
```

Consola will review and apply the rule if appropriate.

### When to request a block

- You received an email that is clearly junk, spam, or promotional
  with no record value
- The sender has sent similar junk before (pattern, not one-off)
- Blocking won't catch legitimate emails from the same domain

### When to request a whitelist

- A legitimate sender keeps getting auto-deleted by the classifier
- The sender's emails should always reach Consola for routing

### When NOT to request a rule

- One-off emails that won't recur
- Emails that are unwanted but potentially useful (unsubscribe instead)
- Anything you're unsure about — just flag it in your response

---

## For Consola

When you receive a sender rule request from a sub-agent, or when the user
asks you to block/whitelist a sender:

1. **Verify the rule makes sense.** Check that it won't accidentally
   catch legitimate emails. Domain-wide blocks (`*.domain.com`) are
   powerful — make sure the whole domain is junk, not just one address.

2. **Edit the config file:**
   ```bash
   # View current rules
   cat ~/.agents/skills/email-triage/references/senders-v3.conf

   # Append a new rule (use >> to append, not > which overwrites)
   echo 'spammer@example.com  *  *  delete  spam-reason' >> ~/.agents/skills/email-triage/references/senders-v3.conf
   ```

3. **Confirm to the user** (or note in your response) what rule was added.

The rule takes effect on the next triage run — no restart needed. The
triage script re-reads the config file on every run.

### Removing a rule

If a rule is causing false positives, find and remove the line:

```bash
# Find the rule
grep 'rule-name' ~/.agents/skills/email-triage/references/senders-v3.conf

# Edit the file to remove it (use sed or manual edit)
```

Surface the removal to the user so they know what changed.
