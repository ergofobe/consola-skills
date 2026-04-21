# Memory File Format Specification

Detailed format specification for agent memory files. Read this when creating new memory files or questioning the format.

## File types

### MEMORY.md — Index file (required)

Every memory scope (global, project, local) must have exactly one `MEMORY.md`. This is the entry point that gets loaded at session start.

**Requirements:**
- Under 200 lines / 25KB
- Contains summaries and pointers to topic files
- Does not contain detailed content — that goes in topic files

**Structure:**

```markdown
<!-- agent-memory v1.0 -->

# Memory Index

## Category Name

- [YYYY-MM-DD] Concise fact or preference
- [YYYY-MM-DD] Another fact
- See `topic-file.md` for details about [specific domain]

## Another Category

- [YYYY-MM-DD] Fact
- [YYYY-MM-DD] Decision: chose X over Y because Z
```

**Categories** are flexible. Use whatever groupings make sense for the content. Common categories:

- **User Preferences** — How the user likes things done
- **Project Facts** — Things true about this project
- **Cross-Project Decisions** — Decisions that apply broadly
- **Gotchas** — Non-obvious traps
- **Conventions** — Naming, formatting, structural conventions

### Topic files (optional, on-demand)

Domain-specific files loaded when the agent is working in that area. Organized by **domain** (what the agent needs to retrieve), not by entry type (fact vs. decision vs. gotcha).

**Naming convention:** Use lowercase with hyphens. The filename should describe the domain: `auth.md`, `database.md`, `deployment.md`, `api-conventions.md`.

**Structure:**

```markdown
<!-- agent-memory v1.0 -->

# Authentication

## Facts
- [2026-03-15] Auth tokens go in X-Custom-Auth header, not Authorization
- [2026-04-01] Token refresh endpoint is /api/v1/auth/refresh

## Decisions
- [2026-03-20] Decided to use JWT over session cookies for API auth

## Gotchas
- [2026-03-18] The /login endpoint returns 200 even on failed auth — check the response body
- [2026-04-05] Token expiry is 1 hour, not 24 hours as the docs say
```

## Entry format

Each entry in a memory file follows this pattern:

```
- [YYYY-MM-DD] Content
```

**Date prefix:**
- Always use ISO date format: `[YYYY-MM-DD]`
- The date is when the fact was learned or the decision was made
- Enables sorting and stale-entry detection
- Optional but strongly recommended

**Content:**
- One fact, preference, or gotcha per line
- Be concise — no prose, no explanation beyond what's necessary
- Be specific — "Default model is openrouter/free" not "Model is set to default"

## Examples

### Global index (~/.agents/memory/MEMORY.md)

```markdown
<!-- agent-memory v1.0 -->

# Memory Index

## User Preferences
- [2026-04-11] No cloud services — prefer local/self-hosted solutions
- [2026-04-11] Source repos in ~/src, project workspaces in ~/workspaces
- [2026-04-11] Never use /tmp for persistent work
- [2026-04-11] Development goes straight to main — no feature branches

## Tool Preferences
- [2026-04-11] Preferred model: openrouter/free
- [2026-04-11] Preferred shell: zsh
- [2026-04-11] Preferred editor: not specified

## Cross-Project Decisions
- [2026-04-11] Use AGENTS.md for instructions, memory files for learned knowledge
- [2026-04-11] Versioning: 0.x.0 for features, 0.x.1+ for fixes
- [2026-04-11] Use ~/.agents/ for cross-harness skills and memory
```

### Project index (.agents/memory/MEMORY.md)

```markdown
<!-- agent-memory v1.0 -->

# Memory Index — example-project

## Project Facts
- [2026-04-11] Rust CLI project using tokio + reqwest with rustls-tls
- [2026-04-11] Default model: openrouter/free
- [2026-04-11] Max agentic loop iterations: 25
- [2026-04-11] SSE responses may come from non-streaming requests (handled in parse_sse_response)
- See `api-conventions.md` for OpenRouter API patterns
- See `build-and-release.md` for CI and release process

## Gotchas
- [2026-04-11] macOS sed requires empty string arg: sed -i ''
- [2026-04-11] GITHUB_TOKEN can't push to other repos — need token PAT
- [2026-04-11] cross tool fails for Android — use cargo-ndk with pre-installed NDK
- [2026-04-11] Android target is aarch64-linux-android, not aarch64-unknown-linux-musl
```

### Project topic file (.agents/memory/build-and-release.md)

```markdown
<!-- agent-memory v1.0 -->

# Build and Release — example-project

## Facts
- [2026-04-11] CI builds 8 targets on tag push via .github/workflows/release.yml
- [2026-04-11] Targets: Linux musl/glibc x2 arch, macOS x2 arch, Android x2 arch
- [2026-04-11] Landing page deployed via .github/workflows/pages.yml

## Decisions
- [2026-04-11] Use musl for universal Linux binary, glibc as alternative
- [2026-04-11] Use cargo-ndk for Android builds (not cross)

## Gotchas
- [2026-04-11] macOS sed -i requires '' arg (BSD vs GNU difference)
- [2026-04-11] Version bump happens right before tagging, not during development
```

## File version header

Every memory file should start with a version comment:

```markdown
<!-- agent-memory v1.0 -->
```

This enables future format changes while maintaining backward compatibility. Agents reading memory files should check this header to determine the format version.

## What NOT to put in memory files

- **Secrets** — API keys, passwords, tokens, credentials. Use `.env` or a secrets manager.
- **PII** — Personally identifiable information about users or customers.
- **Transient state** — Things that change every session (current git branch, open files).
- **Duplicate information** — If it's already in AGENTS.md, don't also put it in memory. Memory is for learned knowledge, not instructions.
- **Large code blocks** — Reference file paths instead of pasting code. "See src/auth.rs for the token refresh logic" is better than copying the function.

## Size limits

| File type | Line limit | Size limit |
|---|---|---|
| MEMORY.md (index) | 200 lines | 25KB |
| Topic file | 100 lines | 10KB |
| Total directory | — | 100KB |

If you're exceeding these limits, it's time to consolidate. See the `agent-dream` skill for memory cleanup guidance.
