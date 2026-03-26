# Pattern 0: Inline Skill Template

Use this template when generating Pattern 0 (inline methodology) skills for Tier 2 plans.

---

## Skill File: SKILL.md

```markdown
---
name: {skill-slug}
description: USE WHEN {trigger conditions}. {What it does in one sentence}.
argument-hint: <target-or-scope>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
user-invocable: true
---

# {Skill Title}

You are a {domain} specialist. {One sentence identity statement}.

## Intent Matrix

| Intent | Workflow |
|--------|----------|
| {intent phrase 1} | -> {workflow-section-1} |
| {intent phrase 2} | -> {workflow-section-2} |
| {intent phrase 3} | -> {workflow-section-1} then {workflow-section-2} |

## {Workflow Section 1}

### Phase 1: {Name}

1. {Step with tool reference}
2. {Step with decision point}
3. {Step with output expectation}

### Phase 2: {Name}

1. {Step}
2. {Step}

**Decision point:** {What determines the next action}

## {Workflow Section 2}

### Phase 1: {Name}

1. {Step}
2. {Step}

## Standards

- {Quality standard 1}
- {Quality standard 2}
- {Output format requirement}

## Output Format

{Description of what this skill produces — structured findings, narrative report, etc.}
```

---

## Directory Structure

```
skills/{skill-slug}/
├── SKILL.md           # The skill definition above
├── references/        # Bundled reference docs (optional)
│   ├── {ref-1}.md
│   └── {ref-2}.md
└── tools/             # Utility tools (optional)
    └── {helper}.ts
```

---

## Kit Manifest

Place alongside the skill directory:

```yaml
name: {skill-slug}
type: skill
repo: git@github.com:nickpending/forge-armory.git
path: skills/{skill-slug}/
domain: security
tags: [{domain-tag}, {methodology-tag}, loadable]
description: One-line description of what this skill does
```

**Required tag:** `loadable` — marks this as a Pattern 0 skill that can be invoked inline or loaded into Pattern 2 agents.

---

## Checklist

- [ ] Frontmatter has `name`, `description` (USE WHEN format), `allowed-tools`, `user-invocable`
- [ ] Description starts with "USE WHEN"
- [ ] Intent matrix maps intents to workflow sections
- [ ] Each workflow section has numbered steps with concrete actions
- [ ] Standards section defines quality gates
- [ ] Output format is specified
- [ ] kit-manifest.yaml includes `loadable` tag
