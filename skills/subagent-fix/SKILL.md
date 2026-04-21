> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: proactively-fix-subagents
version: 1.1.0
description: >
  After delegating work to a sub-agent, inspect the sub-agent's session
  history for failed/blocked/redundant tool calls and proactively fix the
  relevant skill files so the same friction doesn't recur.
metadata:
  hermes:
    tags: [sub-agents, delegation, debugging, skill-maintenance, self-healing]
    triggers:
      - after delegation
      - sub-agent failed
      - too many tool calls
      - approval prompt
      - fix skill
---

# Proactively Fix Sub-Agents

When a sub-agent delegation completes — especially if it took many tool
calls or had errors — inspect the session history to find root causes and
fix the skill files so it doesn't happen again.

## When to Run

- After any delegation with >10 tool calls (waste signal)
- After any delegation with `status: error` tool results
- After user mentions sub-agent friction ("why the approval prompts?", "what's
  with all the errors?")
- Proactively after routine delegations, to catch slow degradation

## Procedure

### 1. Identify the sub-agent session

After `delegate_task` returns, note the session files in `~/.hermes/sessions/`.
The most recent JSON files (sorted by mtime) are the sub-agent sessions.
Match by timestamp proximity to the delegation.

### 2. Extract failed/redundant tool calls

```bash
python3 -c "
import json, glob

files = sorted(glob.glob('~/.hermes/sessions/session_*.json'), reverse=True)[:10]
for f in files:
    data = json.load(open(f))
    msgs = data.get('messages', [])
    errors = []
    terminal_calls = []
    for msg in msgs:
        if msg.get('role') == 'assistant' and msg.get('tool_calls'):
            for tc in msg['tool_calls']:
                fn = tc.get('function', {}).get('name', '')
                args = tc.get('function', {}).get('arguments', '')
                if fn == 'terminal':
                    terminal_calls.append(args)
        if msg.get('role') == 'tool':
            c = str(msg.get('content', ''))
            if 'error' in c[:100].lower() or 'blocked' in c[:100].lower():
                errors.append(c[:300])
    if errors or len(terminal_calls) > 5:
        print(f'SESSION: {f}')
        print(f'  Terminal calls: {len(terminal_calls)}')
        print(f'  Errors: {len(errors)}')
        for e in errors[:5]:
            print(f'  ERR: {e[:200]}')
"
```

### 2a. Keyword search for specific failures

If you know what the sub-agent was doing (e.g., processing a CAT Scale ticket,
reading a domain registrar email), search sessions by keyword:

```bash
python3 -c "
import json, glob
files = sorted(glob.glob('~/.hermes/sessions/session_*.json'), reverse=True)[:15]
for f in files:
    data = json.load(open(f))
    msgs = data.get('messages', [])
    has_match = False
    for msg in msgs:
        c = str(msg.get('content', ''))
        if 'catscale' in c.lower() or 'namecheap' in c.lower():  # adjust keywords
            has_match = True
            break
    if has_match:
        # then extract terminal_calls and errors as in step 2
        ...
"
```

### 2b. Check for skill conflicts

After fixing a sub-agent's skill, check if conflicting skill versions exist
in other locations that could mislead agents:

```bash
# Search all SKILL.md files for Gmail/gws references
search_files pattern='gmail-read|gws-' target='content' file_glob='SKILL.md'

# Key locations to check:
# ~/.agents/skills/       — shared skills (canonical)
# ~/.hermes/skills/       — Hermes auto-synced copies (may be stale)
# ~/workspaces/*/.agents/ — per-agent workspace skills
```

If `~/.hermes/skills/` has a stale copy of a skill you updated, sync it:
```bash
cp ~/.agents/skills/<skill>/SKILL.md ~/.hermes/skills/<skill>/SKILL.md
```

Stale Hermes copies are especially dangerous — agents may find them via
auto-discovery and follow outdated instructions that conflict with your fixes.

### 3. Classify the failures

Common patterns and their fixes:

| # | Pattern | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | **`gmail-read` returns empty** → 40-50 call spiral of raw API/base64/Python | Skill only documents `gmail-read`, not the `+read --html` fallback | Add `gws gmail +read --id <id> --html` as fallback; add explicit "do NOT use raw API/base64/Python" rule |
| 2 | **GWS `--output /tmp/file.pdf` rejected** ("resolves outside current directory") | GWS CLI path safety rejects absolute paths | Add rule: `--output` must be relative path from workspace CWD; use `inbox/attachments/` not `/tmp/` |
| 3 | **Python pipes to read files** (`python3 -c "..."` or `python3 << 'EOF'`) | Skill doesn't tell agent to use `read_file` | Add tool usage rules: `read_file` not `cat`, `patch` not `sed`, no pipe chains |
| 4 | **`read-pdf.py` wrapper** | Skill references obsolete Python script | Replace with `pdftotext -layout` |
| 5 | **Guessing CLI paths** (tries `gws`, then `gws-account`, then full path) | Skill doesn't document the correct binary path | Add explicit path with working examples |
| 6 | **Wrong CLI flags/arguments** | Skill doesn't show working examples | Add command examples with correct syntax |
| 7 | **Duplicate/conflicting skill files** across workspaces | Auto-generated or copied skills with different instructions | Consolidate into shared skill at `~/.agents/skills/`; have agent skills reference it; delete duplicates |
| 8 | **Redundant retries** of same failed command | Skill doesn't have a "stop and ask" rule | Add error handling rules |
| 9 | **Timeout/blocked commands** | Command too slow or disallowed by Hermes | Use native tool or faster alternative |

**Pattern #1 is the #1 issue** across all sub-agents. When `gmail-read` returns
empty (HTML emails from domain registrars, Google, etc.), agents fall into a 40-50 call
spiral of raw API calls, base64 decoding, Python HTML parsing, and pipe chains.
The fix is always the same: add the `+read --html` fallback with an explicit
prohibition on raw API/Python workarounds.

### 3b. Cross-agent GWS fixes

All business agents share the same GWS account and CLI. When you fix a GWS issue
in one agent's skill, check whether the same gap exists in the others:

| Agent | Skill path |
|---|---|
| Agent A | `~/workspaces/business-a/.agents/skills/agent-a/SKILL.md` |
| Agent B | `~/workspaces/business-b/.agents/skills/agent-b/SKILL.md` |
| Agent C | `~/workspaces/business-c/.agents/skills/agent-c/SKILL.md` |
| Agent D | `~/workspaces/business-d/.agents/skills/agent-d/SKILL.md` |
| Agent E | `~/workspaces/business-e/.agents/skills/agent-e/SKILL.md` |
| Agent F | `~/workspaces/personal/.agents/skills/agent-f/SKILL.md` |

Each agent's Gmail section should reference the shared Gmail skill at
`~/.agents/skills/gmail/SKILL.md` rather than duplicating instructions.

### 4. Fix the skill file

- Edit the relevant sub-skill file directly (the one the agent was following)
- Add explicit instructions that prevent the failure pattern
- Include working command examples
- If the fix is cross-cutting (applies to all sub-skills), add it to the
  agent's base SKILL.md

### 5. Verify

On the next delegation that touches the same workflow, check if the tool
call count dropped and errors decreased. If not, the fix was insufficient —
iterate.

## Skill Consolidation

When the same knowledge (Gmail CLI, tool usage rules, etc.) is duplicated
across multiple agent skill files, consolidate it:

1. **Create or update the shared skill** at `~/.agents/skills/<topic>/SKILL.md`
2. **Replace inline instructions in agent skills** with a reference:
   "For Gmail operations, load `~/.agents/skills/gmail/SKILL.md`"
3. **Delete the duplicate copies** from workspace skill directories and
   `~/.hermes/skills/`
4. **Verify the shared skill** is discoverable via Hermes external_dirs
   (check the Hermes config for `external_dirs`)

This ensures fixes propagate to all agents at once instead of requiring
individual patches. The canonical shared skills live at `~/.agents/skills/`.

### Known shared skills

| Topic | Path | Used by |
|-------|------|---------|
| Gmail/GWS operations | `~/.agents/skills/gmail/SKILL.md` | All agents (A, B, C, D, E, F) |
| Sender rules | `~/.agents/skills/sender-block/SKILL.md` | Orchestrator (edits), all agents (read) |
| Email triage | `~/.agents/skills/email-triage/SKILL.md` | Triage pipeline |
| Namecheap domains | `~/.agents/skills/namecheap-domains/SKILL.md` | Agent B (business), Agent F (personal) |

### Known obsolete/conflicting skills (removed)

These were cleaned up in the April 2026 session. If they reappear (e.g., from
running `scripts/gws generate-skills`), delete them again:

- `~/.hermes/skills/gmail/` — stale copy with old `base64 | sed` instructions
- `~/.hermes/skills/productivity/google-workspace/` — Python wrapper approach
- `~/.hermes/skills/email/gmail-monitor/` — obsolete paths, superseded by systemd
- `~/workspaces/business-a/.agents/skills/skills/` — auto-generated GWS tree (80+ skills)
- `~/workspaces/business-a/.agents/skills/fetch-and-file/` — old pre-v3 pipeline
- `~/workspaces/business-a/.agents/skills/gmail-inbox/` — old inbox processor

## Rules

- **Fix root causes, not symptoms.** If the agent used Python to read a file,
  the fix is "use read_file" in the skill, not "install a different Python script."
- **Be specific.** Write exact command examples, not vague instructions like
  "use the appropriate tool." The agent follows instructions literally.
- **First match wins.** If the skill says two contradictory things (e.g.,
  "use pdftotext" in one place and "use read-pdf.py" in another), the agent
  will pick either one. Remove the wrong one entirely.
- **Don't over-engineer.** One-line fixes are better than restructuring. The
  Opus overhaul will handle the big picture — these are tactical patches.
- **Centralize shared knowledge.** Fix once in the shared skill, benefit all
  agents. Don't duplicate GWS CLI instructions across five skill files.
- **Sync stale Hermes copies.** After updating a shared skill, check that
  `~/.hermes/skills/` doesn't have a stale copy with conflicting instructions.
