> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: agent-memory
description: >
  Cross-harness persistent memory for AI agents using plain markdown files.
  Use when remembering facts, preferences, decisions, or gotchas across sessions;
  when starting a new session to load context; or when a user says "remember this".
  Triggers: "remember", "save to memory", "load memory", "what do I know about",
  "recall", "forget".
license: MIT
compatibility: Requires a file system and an agent harness that supports Agent Skills
metadata:
  author: memory-author
  version: "1.0"
---

# Agent Memory

Persistent memory for AI agents using plain markdown files. Works across any harness that supports Agent Skills — Claude Code, opencode, Cursor, Amp, Gemini CLI, Codex, and others.

Memory is **learned knowledge** (what the agent has observed), distinct from **instructions** (what the agent should do, which belongs in AGENTS.md).

## Where memory lives

| Scope | Path | Purpose | Git |
|---|---|---|---|
| Global | `~/.agents/memory/` | Cross-project preferences and facts | Private repo or gitignored |
| Project | `.agents/memory/` | Shared team knowledge for this project | Committed to project repo |
| Local | `.agents/memory.local/` | Personal project overrides | Always gitignored |

The `.agents/` directory is a cross-harness convention. Tools scanning for skills only look in `.agents/skills/` — memory directories are not skills and will not collide.

## Reading memory

### At session start

1. Read `~/.agents/memory/MEMORY.md` (global index) — always.
2. Read `.agents/memory/MEMORY.md` (project index) — if it exists.
3. Do NOT read `.agents/memory.local/MEMORY.md` unless the user asks — local overrides are personal.
4. Do NOT read topic files at session start — load them on demand when working in a relevant domain.

### During a session

When working on a specific domain (e.g., auth, database, deployment), check the index for relevant topic files and read those.

### Important principle

**Treat memory as a hint, not a fact.** Verify against actual code and configuration before acting on remembered information. If memory contradicts reality, update the memory file.

## Writing memory

### Default: explicit with confirmation

- **Always write** when the user explicitly asks ("remember this", "save that for next time").
- **Suggest writing** when a significant fact, decision, or gotcha is discovered — but confirm with the user first. Example: "I noticed the API requires tokens in the X-Custom-Auth header. Should I save this to memory?"
- **Never silently auto-write.** The baseline is conservative. Individual harnesses may opt into more aggressive auto-write, but this skill defaults to explicit.

### When writing

1. **Always read before writing.** Load the current file to avoid overwriting recent changes.
2. **Show the diff.** Display what you're about to write before committing it.
3. **Choose the right scope:**
   - Project-specific knowledge (APIs, conventions, gotchas about this codebase) → `.agents/memory/`
   - Personal project preferences (my preferred branch naming, my editor) → `.agents/memory.local/`
   - Cross-project preferences (I prefer no cloud services, my repos live in ~/src) → `~/.agents/memory/`
   - **Never put project details in global memory.** Never put secrets anywhere.
4. **Append under the right heading.** Don't delete existing entries without user confirmation.
5. **Keep entries concise.** One fact per line. No prose.
6. **Date-prefix entries.** Use `[YYYY-MM-DD]` format so entries can be sorted and stale ones identified.

### Never write to memory

- API keys, credentials, or tokens (use `.env` or a secrets manager)
- Personally identifiable information
- Information the user asked you not to remember
- Transient state that changes frequently

## File structure

### MEMORY.md (index file)

Every memory scope has a `MEMORY.md` that acts as the index. Keep it under 200 lines. It lists what the agent knows and points to topic files for details.

```markdown
<!-- agent-memory v1.0 -->

# Memory Index

## User Preferences
- No cloud services — prefer local/self-hosted solutions
- Source repos in ~/src, project workspaces in ~/workspaces
- Never use /tmp for persistent work

## Project Facts
- Rust project using tokio + reqwest with rustls-tls
- Default model: openrouter/free
- See `deployment.md` for deployment conventions
- See `auth.md` for authentication gotchas

## Cross-Project Decisions
- [2026-04-11] Use AGENTS.md for instructions, memory files for learned knowledge
- [2026-04-11] Versioning: 0.x.0 for features, 0.x.1+ for fixes
```

### Topic files

Domain-specific files organized by what the agent needs to retrieve, not by entry type. An agent working on auth needs auth facts, decisions, AND gotchas — they belong in one file.

```
.agents/memory/
├── MEMORY.md          # Index (always loaded at session start)
├── auth.md            # Authentication: facts, gotchas, decisions
├── database.md        # Database: schema notes, migration patterns
├── deployment.md      # Deployment: CI, release process, env vars
└── api-conventions.md # API patterns, naming rules, error handling
```

Read `references/file-format.md` for detailed format specifications and examples.

## Budget

- **Index file**: Under 200 lines / 25KB. Loaded every session start.
- **Topic files**: Keep each under 100 lines. Loaded on demand.
- **Total memory directory**: Under 100KB. If exceeding, it's time to consolidate (see the `agent-dream` skill).

If the index exceeds 200 lines, split detailed content into topic files and keep only summaries and pointers in the index.

## Forgetting

When a user says "forget" or "remove from memory":

1. Read the relevant memory file.
2. Show the entry being removed.
3. Confirm with the user.
4. Remove the entry and save.

Do not silently remove entries. Always confirm.

## Migration from other harnesses

If you encounter memory files from other harnesses, offer to migrate them. See `references/migration-guide.md` for specific instructions for each harness.

Never delete the originals during migration — the user decides when to clean up.

## Privacy and security

- **Global memory** (`~/.agents/memory/`): If version-controlled, use a private repo only. Never commit to public repos.
- **Project memory** (`.agents/memory/`): Committed to the project repo for team sharing. Never write secrets here.
- **Local memory** (`.agents/memory.local/`): Always gitignored. Personal preferences only.

Include this in project `.gitignore`:

```gitignore
# Personal agent memory (not shared)
.agents/memory.local/
```

## Gotchas

- **Read before writing.** Always load the current file first to avoid overwriting recent changes from other sessions or agents.
- **Project facts in global memory waste context.** Keep project-specific knowledge in `.agents/memory/`, not `~/.agents/memory/`.
- **Memory is a hint, not a fact.** Code and configuration change. If memory says X but the code says Y, trust the code and update memory.
- **No secrets in memory files.** Use `.env` files or secrets managers for credentials.
- **Date your entries.** Undated entries become stale and untrustworthy. Use `[YYYY-MM-DD]` prefixes.
