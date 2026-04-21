# Migration Guide

How to migrate from harness-specific memory to the cross-harness `~/.agents/memory/` convention.

## General principles

- **Never delete originals.** Copy relevant content, don't move it. Let the user decide when to clean up.
- **Show what you're migrating.** Display the entries before writing them to the new location.
- **Confirm with the user.** Don't silently migrate.
- **Rescope if needed.** Project-specific entries from a harness-specific location may need to go in `.agents/memory/` instead of `~/.agents/memory/`.

## From Claude Code

Claude Code stores memory in `~/.claude/projects/<project-hash>/memory/`:

```
~/.claude/projects/abc123def456/memory/
├── MEMORY.md           # Index (first 200 lines loaded at session start)
├── debugging.md        # Topic file
├── api-conventions.md  # Topic file
└── ...
```

**Migration steps:**

1. Find the project directory in `~/.claude/projects/` that corresponds to the current project. The directory name is a hash, so you may need to look at the contents to identify it.
2. Read `MEMORY.md` and all topic files.
3. Determine which entries are project-specific vs. cross-project preferences.
4. Project-specific entries → `.agents/memory/MEMORY.md` and topic files.
5. Cross-project preferences → `~/.agents/memory/MEMORY.md`.
6. Write the migrated content, adding `[YYYY-MM-DD]` date prefixes to entries that lack them.
7. Leave the originals in place.

## From Gemini CLI

Gemini CLI appends to `~/.gemini/GEMINI.md` via the `save_memory` tool. There is no separate memory directory — everything goes in one file.

**Migration steps:**

1. Read `~/.gemini/GEMINI.md`.
2. Extract memory-relevant sections (anything the agent "learned" vs. instructions).
3. Separate project-specific entries from cross-project preferences.
4. Project-specific entries → `.agents/memory/MEMORY.md`.
5. Cross-project preferences → `~/.agents/memory/MEMORY.md`.
6. Leave the original `GEMINI.md` intact — it serves as Gemini's instruction file, not just memory.

## From opencode plugins

opencode itself doesn't have native memory, but some plugins (like `@chiaboon/opencode-agent-memory`) store data in LanceDB or other formats.

**Migration steps:**

1. If the plugin uses a database (LanceDB, SQLite, etc.), export the data using the plugin's tools.
2. If the plugin uses markdown files (some do), read them directly.
3. Convert to the `MEMORY.md` + topic file format.
4. Project-specific entries → `.agents/memory/`.
5. Cross-project entries → `~/.agents/memory/`.
6. Leave the plugin's data in place — the user may still be using it.

## From other formats

For any other memory system:

1. Identify where the memory is stored.
2. Read whatever format it's in.
3. Extract learnable facts, preferences, decisions, and gotchas.
4. Classify each entry as project-specific or cross-project.
5. Write to the appropriate scope with date prefixes.
6. Leave originals in place.

## After migration

Once migration is complete:

1. Review the migrated content with the user.
2. Suggest running the `agent-dream` skill to consolidate and clean up any duplicates.
3. Remind the user they can delete the original harness-specific memory when they're confident the migration is complete.
