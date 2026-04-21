> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: skill-creator
description: >
  Create, validate, and refine Agent Skills following the agentskills.io specification.
  Use when building a new skill, editing an existing skill, or converting a workflow
  into a reusable skill. Triggers: "create a skill", "make a skill", "new skill",
  "build a skill", "skill from workflow", "validate a skill".
license: MIT
compatibility: Requires a terminal and file system access
metadata:
  author: skill-author
  version: "1.0"
---

# Skill Creator

Create Agent Skills that work across all compatible agents (Claude Code, opencode, Cursor, Gemini CLI, etc.) following the [agentskills.io](https://agentskills.io) specification.

## Skill locations

| Scope | Path | Use for |
|---|---|---|
| Global | `~/.agents/skills/` | Skills available across all projects |
| Project | `.agents/skills/` | Skills specific to one project |

Some clients also support their own paths (e.g., Claude Code uses `~/.claude/skills/`). Use `~/.agents/skills/` as the default for maximum compatibility.

## Step 1: Gather requirements

Ask the user:

1. **What task or workflow should this skill handle?** — Get a concrete description, not just a label.
2. **When should an agent invoke this skill?** — What triggers it? What would the user say?
3. **Global or project-scoped?** — Global goes in `~/.agents/skills/`, project goes in `.agents/skills/`.
4. **What does the agent already know?** — Avoid instructions that state the obvious. Focus on what the agent would get wrong without this skill.

If the user describes a workflow they just completed, extract the steps that worked and the corrections they made. These corrections become gotchas.

## Step 2: Design the skill

### Name

- Lowercase alphanumeric and hyphens only
- Max 64 characters
- No leading, trailing, or consecutive hyphens
- Must match the directory name exactly
- Examples: `pdf-processing`, `release-workflow`, `database-migrations`

### Description

This is the most critical field — it determines when the skill activates.

Rules:
- Max 1024 characters
- Describe both **what** the skill does and **when** to use it
- Include trigger keywords and use cases
- Write in third person

Good: "Extracts text and tables from PDF files, fills PDF forms, and merges PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction."

Bad: "PDF helper skill."

### Structure

Decide what supporting files the skill needs:

```
skill-name/
├── SKILL.md              # Required
├── scripts/              # Optional: executable code
├── references/           # Optional: detailed documentation
└── assets/               # Optional: templates, resources
```

Keep SKILL.md under 500 lines. Move detailed content to `references/` with intention-revealing filenames.

## Step 3: Write the SKILL.md

Use `references/skill-template.md` as a starting template. Fill in each section:

### Frontmatter

```yaml
---
name: skill-name          # Required
description: >             # Required, max 1024 chars
  What the skill does and when to use it.
license: MIT              # Optional
compatibility: Requires git, docker  # Optional, max 500 chars
metadata:                 # Optional
  author: your-name
  version: "1.0"
---
```

Do NOT include `model`, `tools`, or `allowed-tools` fields (unless you intentionally need the experimental `allowed-tools`).

### Body content

Structure the body for progressive disclosure:

1. **Core instructions only** in SKILL.md — what the agent needs every time
2. **Detailed references** in separate files — loaded on demand
3. **Tell the agent WHEN to load each file** — not just "see references/ for details"

Example of good file reference:
```markdown
Read `references/api-errors.md` if the API returns a non-200 status code.
```

Example of bad file reference:
```markdown
See references/ for more information.
```

### Writing style

- **Be concise** — Only include what the agent wouldn't know. Skip explanations of common concepts.
- **Be actionable** — Start instructions with verbs: "Run", "Create", "Check", "Extract".
- **Be specific** — Provide exact commands, file paths, and formats.
- **Provide defaults** — Pick one recommended approach; mention alternatives briefly, not as equals.
- **Favor procedures** — Teach how to approach a class of problems, not what to produce for one instance.

### Sections to include

Not every skill needs all of these, but consider each:

- **Gotchas** — Environment-specific facts that defy assumptions. Highest-value content in most skills.
- **Checklists** — For multi-step workflows with dependencies or validation gates.
- **Validation loops** — "Do the work, run validation, fix, repeat until passing."
- **Plan-validate-execute** — For batch or destructive operations: create a plan, validate it against truth, then execute.
- **Output templates** — When the agent must produce output in a specific format, provide a template to pattern-match against.

## Step 4: Add supporting files

### File naming

Use intention-revealing names. The filename should tell you what's inside.

Good: `references/api-error-codes.md`, `scripts/validate-schema.sh`, `assets/report-template.md`
Bad: `references/utils.md`, `scripts/helper.py`, `assets/template.md`

### File references

Use relative paths from the skill root. Keep references one level deep — avoid deep nesting.

```markdown
See `references/api-error-codes.md` for [what it contains].
Run `scripts/validate-schema.sh` before submitting.
```

### Scripts

If the skill includes executable scripts:
- Make them self-contained or clearly document dependencies
- Include helpful error messages
- Handle edge cases gracefully
- Any language is fine (bash, python, node, etc.)

## Step 5: Validate

Check every requirement from the specification. See `references/specification-summary.md` for the complete validation checklist.

Quick checks:
- [ ] Name matches directory name exactly
- [ ] Name is lowercase, no leading/trailing/consecutive hyphens, max 64 chars
- [ ] Description is 1-1024 chars, describes what AND when, includes triggers
- [ ] SKILL.md body is under 500 lines
- [ ] File references use relative paths, one level deep
- [ ] Supporting files have intention-revealing names
- [ ] Frontmatter has only valid fields: `name`, `description`, `license`, `compatibility`, `metadata`
- [ ] No `model` or `tools` fields in frontmatter

## Step 6: Refine with real execution

The first draft usually needs refinement. After creating a skill:

1. **Test it** — Invoke the skill on a real task matching its description.
2. **Read the traces** — Where did the agent waste time? What did it get wrong?
3. **Add gotchas** — Every correction you make is a candidate for the gotchas section.
4. **Cut the unnecessary** — If the agent handled something well without instruction, remove that instruction.
5. **Clarify triggers** — If the skill didn't activate when expected, improve the description.

One pass of execute-then-revise noticeably improves quality.

## Gotchas

- **Name must match directory** — If the skill directory is `pdf-processing`, the `name` field must be `pdf-processing`. Mismatches cause discovery failures.
- **Description is for activation, not documentation** — A description that says "Handles PDFs" won't activate when the user says "extract text from an invoice". Include trigger keywords.
- **Progressive disclosure matters** — A 1000-line SKILL.md wastes context. The agent loads the entire body every time the skill activates. Keep it under 500 lines; move details to `references/`.
- **Tell the agent WHEN to load files** — "See references/ for details" is useless because the agent won't know to look. "Read `references/api-errors.md` if the API returns a non-200 status" is useful because it gives a trigger condition.
- **Avoid duplicating what the agent already knows** — Don't explain what HTTP is, what a database does, or how to write a for loop. Focus on what the agent would get wrong without your skill.
