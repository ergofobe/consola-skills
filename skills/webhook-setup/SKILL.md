> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: webhook-platform-setup
description: Setup guide and gotchas for the Hermes webhook platform — enabling the platform, configuring static routes, template variables, and cross-platform delivery. Covers pitfalls discovered through live testing.
version: 1.0.0
metadata:
  hermes:
    tags: [webhook, events, automation, integrations, notifications, devops]
---

# Webhook Platform Setup Guide

Practical guide for enabling and configuring the Hermes webhook platform. This covers the setup path that actually works, and the pitfalls that don't.

## Enabling the Platform

### Config placement (CRITICAL)

Webhook config MUST go under `platforms.webhook` in the Hermes config file. A top-level `webhook:` key will NOT work — it only bridges specific keys like `channel_prompts`, not `enabled`, `extra`, or `routes`.

```yaml
# ✅ CORRECT — under platforms:
platforms:
  webhook:
    enabled: true
    extra:
      host: "127.0.0.1"
      port: 8644
      routes:
        notify:
          secret: "your-hmac-secret"
          prompt: "Notification: {__raw__}"
          deliver: telegram
```

```yaml
# ❌ WRONG — top-level key does NOT load routes/enabled/extra:
webhook:
  enabled: true
  extra:
    host: "127.0.0.1"
    port: 8644
```

### Alternative: Environment variables

Set `WEBHOOK_ENABLED=true`, `WEBHOOK_PORT=8644`, `WEBHOOK_SECRET=*** in the Hermes env file.

**Limitation:** Env vars enable the platform and set port/secret, but cannot define routes. For routes, you must use the `platforms.webhook` config section.

### After configuration

Gateway restart is required — there is no hot-reload for static routes.

Verify with health check: `curl http://127.0.0.1:8644/health` → `{"status": "ok", "platform": "webhook"}`

## Template Variables

The `_render_prompt()` method uses regex `\{([a-zA-Z0-9_.]+)\}` to resolve keys from the POST payload.

### `{__raw__}` — Full payload dump

Dumps the entire JSON payload as indented text (truncated to 4000 chars). Use this when the agent needs to see everything:

```yaml
prompt: "Event notification: {__raw__}"
```

### Dot-notation access

Reference nested fields from the payload:
```yaml
prompt: "New issue: {issue.title} by {issue.user.login}"
```

### ⚠️ `{body}` does NOT exist

There is no special `body` token. If you use `{body}` and the payload has no `body` key, it renders as the literal string `{body}`. This is the most common misconfiguration.

## Delivery Modes

| `deliver` value | Behavior |
|----------------|----------|
| `log` | Silent — response goes to Python logger only. User sees nothing. |
| `telegram` | Cross-platform delivery to Telegram home channel. ~5s latency (LLM thinking). |
| `discord` | Cross-platform delivery to Discord. |
| `github_comment` | Post as GitHub comment (needs deliver_extra with repo/pr_number). |

### `deliver_only: true` — Skip the LLM

The rendered prompt template becomes the literal message delivered to the target. Zero LLM cost, sub-second delivery. Requires `deliver` to be a real target (not `log`).

## Route Configuration Reference

```yaml
platforms:
  webhook:
    enabled: true
    extra:
      host: "127.0.0.1"     # Listen address (127.0.0.1 for localhost-only)
      port: 8644            # Listen port
      secret: "global-secret"  # Optional: default HMAC secret for routes without one
      rate_limit: 30        # Requests per minute per route (default: 30)
      max_body_bytes: 1048576  # Max payload size (default: 1MB)
      routes:
        route-name:
          secret: "INSECURE_NO_AUTH"  # Per-route HMAC secret. INSECURE_NO_AUTH for testing.
          prompt: "Template with {__raw__}"  # Agent prompt template
          deliver: log              # Where to send the response
          deliver_extra:            # Optional: delivery target details
            chat_id: "CHAT_ID"      # Telegram chat ID (falls back to home channel if omitted)
          events: []                # Optional: filter by event type header
          skills: []                # Optional: load skills before agent run
          deliver_only: false       # Skip LLM, send prompt as-is
```

## Testing

Test script at `~/.agents/scripts/test-webhook.sh`:
```bash
./test-webhook.sh test                                    # Simple test (deliver: log)
./test-webhook.sh notify '{"event":"test","msg":"hello"}' # Telegram delivery test
```

## Pitfalls

1. **Config under `platforms:`, not top-level** — Top-level `webhook:` only bridges channel_prompts. Routes and enabled flag won't load.

2. **No `{body}` variable** — Use `{__raw__}` or reference actual payload keys. Unknown keys render as literal `{key_name}`.

3. **Static routes need gateway restart** — No hot-reload. Dynamic routes (via `hermes webhook subscribe`) do hot-reload.

4. **Each POST = separate agent session** — Webhook responses don't appear in the current conversation. They go through an independent session.

5. **First delivery after restart may queue** — If the target platform adapter hasn't reconnected, delivery may delay until the next inbound message triggers a flush.

6. **Home channel fallback** — If `deliver_extra.chat_id` is omitted, the adapter uses the target platform's home channel. If no home channel is configured, delivery fails with "No chat_id or home channel".

7. **`deliver: log` is invisible** — The agent processes the prompt but nobody sees the output. Only useful for debugging or silent processing.

8. **TELEGRAM_HOME_CHANNEL must be numeric** — If `TELEGRAM_HOME_CHANNEL` in `.env` is set to a Telegram username (e.g., `@some_user`) instead of a numeric chat ID, cross-platform delivery crashes with `ValueError: invalid literal for int() with base 10`. The Telegram `send()` method does `int(chat_id)` unconditionally. Always set `TELEGRAM_HOME_CHANNEL` to the numeric ID (find it in `~/.hermes/channel_directory.json`). Belt-and-suspenders: also set `deliver_extra.chat_id` explicitly in webhook route configs. Upstream bug: NousResearch/hermes-agent#13206.

9. **Webhook platform needs toolsets** — The `platform_toolsets` config must include a `webhook` entry, or the webhook agent runs **with no tools at all** — it can only generate text. If your prompt asks the agent to call `write_file`, `read_file`, `terminal`, etc., it will silently skip those steps because it physically cannot make tool calls. Add to config:
    ```yaml
    platform_toolsets:
      webhook:
      - hermes-cli
    ```

10. **Prompt instructions to write files are unreliable** — Even with toolsets enabled, LLM agents may skip side-effect instructions buried in prompts. If a context file or other artifact MUST be written, either: (a) have the *calling script* write it before the webhook fires, or (b) add a catchup fallback that reconstructs from on-disk data when the context file is missing. Don't rely solely on prompt instructions for critical path file writes.

11. **Batch your payloads** — Don't send one webhook POST per event (e.g., one per email). The gateway spawns a separate agent session per POST, which overwhelms context windows and delivery queues. Instead, collect events in the calling script and send one batch payload. For large batches, send a lightweight summary (headers + short preview) in the webhook payload and save full details to per-item JSON files on disk — the agent can `read_file` individual items on demand.

12. **Unique notification IDs per call** — If you send multiple webhook POSTs in one run (e.g., one per account), each MUST have a unique notification ID. Otherwise, the agent's context file writes will overwrite each other and you'll only see the last one via `/catchup`. Embed an account indicator in the ID (e.g., `TRI-PA3K` for personal, `TRI-OV7N` for business).

13. **Two-step prompt design** — When you need the agent to both write a file AND deliver a message, structure the prompt as explicit numbered steps:
    - Step 1: "Use write_file tool to save to <path>" (mandatory, always)
    - Step 2: "Your <platform> message should be a concise summary. Do NOT mention the file write or tool usage in your <platform> message."
    - Always end with: `📋 <ID> → /catchup <ID>`
    
    Without this separation, the agent may: (a) bleed tool status into the user message ("Done. Context file written to..."), (b) skip the file write in dry_run mode, or (c) include the `/catchup` line only in the file but not the message.
