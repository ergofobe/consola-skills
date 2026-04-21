# Skill Template

Copy this template as a starting point for a new SKILL.md. Replace all placeholder values.

```markdown
---
name: SKILL-NAME
description: >
  WHAT the skill does and WHEN to use it. Include specific trigger keywords
  that match what a user would say. Max 1024 characters.
license: MIT
compatibility: Required tools, packages, or environment (optional, max 500 chars)
metadata:
  author: YOUR-NAME
  version: "1.0"
---

# SKILL-NAME

One-line summary of what this skill enables.

## When to use this skill

Describe the specific situations where this skill should be activated. Include trigger phrases a user might say.

## Instructions

### Step 1: Brief action

Specific, actionable instruction.

### Step 2: Brief action

Specific, actionable instruction.

### Step 3: Brief action

Specific, actionable instruction.

Continue with as many steps as needed. Keep each step focused and actionable.

## Gotchas

- **GOTCHA 1** — Environment-specific fact that defies reasonable assumptions.
- **GOTCHA 2** — Common mistake the agent would make without being told.
- **GOTCHA 3** — Non-obvious constraint or requirement.

## Examples

### Example: Brief description

Input:
\```
concrete example input
\```

Output:
\```
concrete example output
\```

## References

- Read `references/SPECIFIC-TOPIC.md` when [specific trigger condition].
- Run `scripts/SPECIFIC-ACTION.sh` when [specific trigger condition].
```

## After creating from this template

1. Replace all `UPPERCASE` placeholders with actual values
2. Remove sections that don't apply (examples, gotchas, references)
3. Add sections that are needed but not in the template (checklists, validation loops, templates)
4. Keep the total body under 500 lines — move detailed content to `references/`
5. Validate against the specification checklist in `references/specification-summary.md`
