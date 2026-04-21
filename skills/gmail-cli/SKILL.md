> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: gmail
version: 1.0.0
description: >
  Unified Gmail operations skill: authenticate, read, send, reply, archive,
  trash, label, draft, and filter emails across multiple Google accounts.
  Use when the user asks to check email, send a message, archive mail, trash
  messages, apply labels, or any Gmail operation. Also use when another skill
  needs to interact with Gmail programmatically.
---

# Gmail — Unified Operations Skill

All Gmail operations across personal and business accounts.

---

## Accounts

| Account | Email | Config dir |
|---|---|---|
| `personal` | user@gmail.com | `~/.config/gws/personal/` |
| `business` | user@business.example | `~/.config/gws/business/` |

Every script requires `--account personal` or `--account business`.

---

## Prerequisites

- `gws` CLI installed at `/usr/bin/gws` (v0.18.1+)
- Config dirs at `~/.config/gws/personal/` and `~/.config/gws/business/` with valid OAuth credentials
- `jq` for JSON parsing
- Scripts at `~/.config/gws/bin/`

Verify auth before first use:

```bash
~/.config/gws/bin/gmail-whoami --account personal
~/.config/gws/bin/gmail-whoami --account business
```

If auth fails, re-authenticate:

```bash
~/.config/gws/bin/gws-personal auth login
~/.config/gws/bin/gws-business auth login
```

---

## Scripts

All scripts live in `~/.config/gws/bin/`. Every script takes `--account <personal|business>` as the first flag. Use full paths (e.g. `~/.config/gws/bin/gmail-list`) unless `~/.config/gws/bin/` is on your PATH.

### gmail-whoami

Verify which account is authenticated.

```bash
gmail-whoami --account personal
```

### gmail-list

List messages with sender, subject, date. Output: TSV (id, from, subject, date).

```bash
gmail-list --account personal
gmail-list --account business "is:unread in:inbox" 50
gmail-list --account personal "from:someone@example.com"
```

### gmail-read

Read the plain text body of a message.

```bash
gmail-read --account personal <messageId>
```

**If gmail-read returns empty**, the email is HTML-heavy. Use the `+read`
command instead:

```bash
~/.config/gws/bin/gws-business gmail +read --id <messageId> --html
~/.config/gws/bin/gws-personal gmail +read --id <messageId> --html
```

Do NOT fall back to raw API calls, `base64 -d`, Python scripts, or pipe
chains to decode HTML emails. The `+read --html` flag handles this.

### gmail-headers

Get key headers (From, Subject, Date, List-Unsubscribe).

```bash
gmail-headers --account business <messageId>
```

### gmail-archive

Remove INBOX label (archive). **This is the Inbox Zero operation** — removes the message from the inbox without deleting it.

```bash
gmail-archive --account personal <messageId>
```

### gmail-trash

Move a message to trash. Recoverable for 30 days.

```bash
gmail-trash --account business <messageId>
```

### gmail-batch-trash

Bulk trash all messages matching a query.

```bash
gmail-batch-trash --account personal "category:promotions older_than:30d" 500
```

**CAUTION**: Destructive. Always confirm with user before running.

### gmail-label

Add or remove a label on a message.

```bash
gmail-label --account personal add <messageId> <labelId>
gmail-label --account business remove <messageId> <labelId>
```

Common label IDs: `INBOX`, `UNREAD`, `STARRED`, `TRASH`, `SPAM`, `CATEGORY_PROMOTIONS`, `CATEGORY_UPDATES`, `CATEGORY_SOCIAL`, `CATEGORY_FORUMS`.

Custom labels (the script resolves display names to IDs automatically):
- `Deferred` = Label_3 (personal), Label_8 (business) — applied to deferred triage messages
- `ACTIVE_LOAD` = Label_2 (personal) — applied to current dispatch notification

### gmail-label-counts

Print message counts for key labels.

```bash
gmail-label-counts --account personal
```

### gmail-reply

Send a reply threaded into the original message.

```bash
gmail-reply --account personal <messageId> "Thanks, got it!"
```

### gmail-send

Send a new email message.

```bash
gmail-send --account personal --to alice@example.com --subject "Hello" --body "Hi Alice!"
gmail-send --account business --to bob@example.com --subject "Report" --body "See attached" --cc carol@example.com
gmail-send --account personal --to alice@example.com --subject "Hello" --body "<b>Bold</b> text" --html
```

Flags: `--to`, `--subject`, `--body` (required), `--cc`, `--bcc`, `--from`, `--html` (optional).

### gmail-draft

Create a Gmail draft.

```bash
gmail-draft --account personal --to alice@example.com --subject "Draft" --body "Work in progress"
```

### gmail-create-filter

Create a Gmail filter.

```bash
gmail-create-filter --account personal '{"from":"spam@example.com"}' '{"addLabelIds":["TRASH"]}'
gmail-create-filter --account business '{"query":"6164096-1"}' '{"addLabelIds":["TRASH"]}'
```

---

## Downloading Attachments

Email attachments (PDFs, images, etc.) can be downloaded via the GWS CLI.

### Step 1: Get attachment IDs

Read the full message to find attachment metadata:

```bash
~/.config/gws/bin/gws-business gmail users messages get \
  --params '{"userId":"me","id":"<messageId>","format":"full"}' \
  2>/dev/null | jq '[.payload.parts[] | select(.filename | length > 0) | {filename, attachmentId: .body.attachmentId}]'
```

### Step 2: Download

```bash
# --output MUST be a RELATIVE path (GWS rejects absolute paths like /tmp/)
# Always cd to your workspace first:
cd ~/workspaces/<workspace> && ~/.config/gws/bin/gws-business gmail users messages attachments get \
  --params '{"userId":"me","messageId":"<messageId>","id":"<attachmentId>"}' \
  --output inbox/attachments/<filename>
```

**Key rules:**
- `--output` must be a **relative path** — GWS path safety rejects absolute
  paths (`/tmp/file.pdf`) as a security measure
- Always run from your workspace directory as CWD
- Do NOT use Python to decode base64 attachment data — use the GWS CLI directly
- Do NOT write to `/tmp/` — use `inbox/attachments/` as the staging directory

---

## Direct gws API Access

For operations not covered by the scripts, use the gws wrappers directly:

```bash
# Personal account
~/.config/gws/bin/gws-personal gmail users messages list \
  --params '{"userId": "me", "q": "in:inbox", "maxResults": 10}'

# Business account
~/.config/gws/bin/gws-business gmail users messages get \
  --params '{"userId": "me", "id": "<MSG_ID>", "format": "full"}'
```

See `references/gws-auth.md` for authentication details and `references/gmail-api-patterns.md` for common API call patterns.

---

## Security Rules

- **Never** output secrets (API keys, tokens) directly
- **Always** confirm with user before executing write/delete commands (trash, batch-trash, send, reply)
- Prefer `--dry-run` for destructive operations when available
- Archive before trash — archiving is reversible, trashing starts a 30-day countdown

---

## Shell Tips

- **zsh `!` expansion**: Use double quotes for Gmail queries containing `!`:
  ```bash
  gmail-list --account personal "from:someone newer_than:1d"
  ```
- **JSON with --params/--json**: Wrap values in single quotes so the shell doesn't interpret inner double quotes:
  ```bash
  gws-personal gmail users messages list --params '{"userId": "me", "q": "in:inbox"}'
  ```
- **gws stderr**: Always pipe stderr through `2>/dev/null` when parsing JSON output programmatically
- **gmail-list label queries**: Use `"label:Deferred"` query string syntax — there is NO `--label` flag. The query is a positional arg: `gmail-list --account personal "label:Deferred" 10`

## Tool Usage Rules for Sub-Agents

When using Gmail as a sub-agent (via `delegate_task`), follow these rules:

- **Use `read_file` for reading local files** — never `cat`, Python, or shell pipes
- **Use `patch` for file edits** — never `sed`, `awk`, or Python scripts
- **No pipe chains** — each tool call should be one operation. No `cat | grep | awk`
  or `jq | base64 -d | grep`. Use the appropriate tool directly
- **No Python scripts** — never use `python3 -c "..."` or `python3 << 'EOF'` to
  read, decode, or transform data. Use native tools and CLI commands instead
