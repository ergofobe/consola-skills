# Agent Skills Specification Summary

Reference for validating skills against the [agentskills.io](https://agentskills.io/specification) specification. Read this when validating a newly created or edited skill.

## Directory structure

A skill is a directory containing, at minimum, a `SKILL.md` file:

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
└── assets/           # Optional: templates, resources
```

Any additional directories or files are allowed. The structure is flexible beyond the required `SKILL.md`.

## SKILL.md frontmatter

YAML frontmatter between `---` delimiters at the top of the file.

### Required fields

| Field | Description | Constraints |
|---|---|---|
| `name` | Skill identifier | 1-64 chars, lowercase alphanumeric + hyphens only, no leading/trailing/consecutive hyphens, must match directory name |
| `description` | When to invoke | 1-1024 chars, non-empty, should describe what AND when |

### Optional fields

| Field | Description | Constraints |
|---|---|---|
| `license` | License name or reference | Any string |
| `compatibility` | Environment requirements | 1-500 chars if provided |
| `metadata` | Additional properties | String-to-string map |
| `allowed-tools` | Pre-approved tools | Space-separated string (experimental) |

### Forbidden fields

Do NOT include these fields (they are from other formats or are legacy):

- `model` — Not applicable to skills
- `tools` — Legacy sub-agent field, not for skills
- `allowed-tools` is experimental; omit unless you have a specific need

### Frontmatter examples

Minimal:
```yaml
---
name: pdf-processing
description: Extracts text from PDFs and merges files. Use when handling PDF documents.
---
```

With optional fields:
```yaml
---
name: pdf-processing
description: Extracts text from PDFs and merges files. Use when handling PDF documents.
license: Apache-2.0
compatibility: Requires poppler-utils and Python 3.10+
metadata:
  author: example-org
  version: "1.0"
---
```

## Body content

The Markdown body after the frontmatter contains the skill instructions. No format restrictions on the body.

Recommended sections:
- Step-by-step instructions
- Examples of inputs and outputs
- Common edge cases (gotchas)
- Checklists for multi-step workflows
- Validation loops

## Validation checklist

Use this when reviewing a skill before finalizing:

### Frontmatter
- [ ] `name` field present and valid
- [ ] `name` matches directory name exactly
- [ ] `name` is 1-64 characters
- [ ] `name` contains only lowercase letters, numbers, and hyphens
- [ ] `name` does not start or end with a hyphen
- [ ] `name` does not contain consecutive hyphens (`--`)
- [ ] `description` field present and valid
- [ ] `description` is 1-1024 characters
- [ ] `description` describes what the skill does AND when to use it
- [ ] `description` includes trigger keywords
- [ ] No `model` or `tools` fields present
- [ ] Optional fields (`license`, `compatibility`, `metadata`) are valid if present

### Body content
- [ ] Body is under 500 lines
- [ ] Instructions are actionable (start with verbs)
- [ ] No unnecessary explanations of common concepts
- [ ] Gotchas section includes environment-specific traps
- [ ] File references use relative paths from skill root
- [ ] File references include WHEN to load (not just "see references/")
- [ ] Supporting files have intention-revealing names

### Structure
- [ ] `SKILL.md` exists in the skill directory
- [ ] Supporting files are in appropriate subdirectories (`scripts/`, `references/`, `assets/`)
- [ ] File references are one level deep (no deep nesting)
- [ ] Scripts are self-contained or clearly document dependencies

### Progressive disclosure
- [ ] SKILL.md contains only core instructions (loaded every time skill activates)
- [ ] Detailed content moved to `references/` (loaded on demand)
- [ ] Agent is told WHEN to load each reference file

## Name validation rules

Valid examples:
- `pdf-processing`
- `data-analysis`
- `code-review`
- `vibe-dev`

Invalid examples:
- `PDF-Processing` (uppercase)
- `-pdf` (leading hyphen)
- `pdf--processing` (consecutive hyphens)
- `pdf_processing` (underscores — use hyphens)
- `pdf processing` (spaces)
- `this-name-is-way-too-long-to-be-a-skill-name-it-should-be-under-64-chars-total` (over 64 chars)

## Description writing

The description is the most critical field for skill activation. Agents use it to decide when to invoke the skill.

### Good descriptions

- "Extracts text and tables from PDF files, fills PDF forms, and merges PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction."
- "End-to-end vibe coding workflow for building software with AI agents. Use when starting a new feature, bug fix, or project from scratch."
- "Creates and validates Agent Skills following the agentskills.io specification. Use when building a new skill, editing an existing skill, or converting a workflow."

### Poor descriptions

- "Helps with PDFs." (too vague, no triggers)
- "A skill for PDF processing." (no "when to use")
- "This skill does things with documents and files and stuff." (no specifics)
