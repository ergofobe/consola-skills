> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: reporting-upstream-bugs
description: >
  Report bugs and enhancement requests to external dependency repos. Use when discovering
  a bug, limitation, or missing feature in a third-party tool, library, or dependency
  that we depend on. Triggers: "upstream bug", "file an issue", "report this bug",
  "open a github issue for", "this dependency has a bug", "workaround for", or whenever
  a bug in an external dependency is discovered during development.
license: MIT
compatibility: Requires gh CLI, agent-memory skill
metadata:
  author: bug-reporter
  version: "1.0"
---

# Reporting Upstream Bugs

When we discover a bug, limitation, or missing feature in an external dependency during our work, we report it upstream so it gets fixed — and we track it locally so we know to remove our workaround when it's resolved.

## Why This Skill Exists

Without a systematic process, bugs in external dependencies accumulate as ad-hoc workarounds scattered across codebases and memory files. This skill ensures every upstream issue is:

1. **Verified** against the dependency's source code and existing issues
2. **Reported** with high-quality reproduction steps and proposed fixes
3. **Tracked** in our memory so we can clean up workarounds when resolved

## Core Process

When you discover a bug or limitation in an external dependency, follow these steps **in order**. Do not skip steps.

### Step 1: Identify the source repository

Determine where the dependency lives:

- Check `~/src/` for local clones: `ls ~/src/<dependency-name>`
- Check the dependency's package metadata (Cargo.toml, package.json, etc.) for the repo URL
- Look for `AGENTS.md` or `README.md` in the local clone for project context
- If no local clone exists, identify the GitHub repo URL from the package registry or docs

**Gate**: If you cannot identify a public repo to report to, record the workaround in memory and stop. Not all dependencies accept issues.

### Step 2: Read the source code

If a local clone exists (or you can clone it), **read the relevant source files** to:

- Confirm the bug exists in the current code
- Identify the exact lines/functions causing the issue
- Understand the root cause, not just the symptom
- Draft a proposed fix with specific code changes

This step is what separates a useful bug report from a vague complaint. Always include file paths and line numbers in the issue.

### Step 3: Search for existing issues

Before writing a new issue, search the upstream repo:

```bash
gh issue list --repo <owner>/<repo> --search "<keyword>"
gh issue list --repo <owner>/<repo> --state all --search "<keyword>"
```

Search for multiple keywords related to the bug. Check both open AND closed issues — the issue may have been reported and already fixed in a newer version.

**If an existing issue is found:**

1. Read the existing issue and comments carefully
2. If our workaround or proposed fix **is not mentioned**, add a comment with:
   - Our specific reproduction context (how we hit it)
   - Our workaround (if we have one)
   - Our proposed fix (if we have one and it differs from existing discussion)
3. Record the issue URL in our memory (Step 5)
4. **Do not create a duplicate issue**

**If no existing issue is found:** Proceed to Step 4.

### Step 4: Write a new issue

Create a thorough, actionable issue using `gh issue create`:

```bash
gh issue create --repo <owner>/<repo> --title "<title>" --body "$(cat <<'EOF'
<issue body>
EOF
)"
```

**Required sections in the issue body:**

1. **Summary** — One paragraph: what's wrong and why it matters
2. **Impact** — How this affects real users. Be concrete: "This blocks the email-triage pipeline because..."
3. **Reproduction** — Exact steps to reproduce, including commands. Must be runnable.
4. **Root Cause** — File paths, line numbers, and code snippets from the source. Explain *why* it fails, not just *that* it fails.
5. **Current Workaround** — What we're doing to work around it, with code snippets. Acknowledge limitations of the workaround.
6. **Suggested Fix** — Specific code changes that would fix it. Include diff-style snippets. Propose the simplest correct fix, not a redesign.
7. **Environment** — Version, OS, how it was discovered, relevant session IDs.

**Title format**: Use conventional prefix + brief description:
- `Bug: Strict UTF-8 decoding in file attachments causes failures`
- `Enhancement: Add --timeout flag for API request timeout`
- `Bug: Shell tool ignores configured timeout parameter`

**Quality bar**: The issue should be detailed enough that a developer unfamiliar with our use case can reproduce it and implement the fix without asking questions.

### Step 5: Record in memory

After the issue is created or an existing issue is found, record it in the appropriate memory scope.

**Global memory** (`~/.agents/memory/`) — for cross-project dependencies:

```markdown
## Upstream Issues
- [YYYY-MM-DD] <dependency>: <short description> — <issue URL> — workaround: <our workaround summary>
```

**Project memory** (`.agents/memory/`) — for project-specific dependencies:

Add a `## Upstream Issues` section (or append to existing one) with the same format.

**What to record:**

- Date discovered
- Dependency name
- Short description of the bug
- Issue URL (new or existing)
- Our workaround (one line)
- Whether our workaround is destructive (e.g., "strips all non-ASCII, losing Unicode")

**Why we track this**: When the dependency releases a new version, we check these entries to see which workarounds can be removed and which issues can be closed.

## Tracking and Follow-Up

### Checking for resolutions

When updating a dependency or during periodic reviews:

```bash
gh issue view <number> --repo <owner>/<repo> --json state,closedAt,milestone
```

If an issue is closed/fixed:
1. Verify the fix in the new version
2. Remove our workaround code
3. Update the memory entry: add `[RESOLVED YYYY-MM-DD]` prefix and remove the workaround
4. Close any loop — if we commented with a workaround, update the comment noting it's now fixed upstream

### Memory entry lifecycle

```
[2026-04-12] orai: strict UTF-8 in attachments — github.com/user/repo/issues/2 — workaround: strip non-ASCII with perl
→ [RESOLVED 2026-05-01] orai: strict UTF-8 in attachments — github.com/user/repo/issues/2 — workaround removed in v0.3.0
```

## Anti-Patterns

- **Don't file vague issues** without reading the source code first
- **Don't create duplicates** without searching existing issues
- **Don't just report — propose a fix** with specific code changes
- **Don't forget to track** — an untracked issue means a permanent workaround
- **Don't file bugs for unmaintained projects** — record the workaround and move on
