# Composition Rules

These rules govern how the assembler decomposes plans into artifacts. Apply all 11 rules during artifact generation.

---

## The 11 Rules

1. **One skill per methodology concern** — split when plan crosses 2+ domains with different tool sets
2. **Wrapper boundary** — deterministic (same input → same output) = wrapper. Reasoning required = skill. Never mix.
3. **Naming signal** — if you need "and" in the name, it's two things
4. **Granularity floor** — "run this one command" is Tier 1, not a skill. Skills encode methodology, not individual commands.
5. **Granularity ceiling** — if instructions exceed ~200 lines, split along judgment boundaries
6. **Reuse test** — "would someone ever want just the first half?" If yes, split.
7. **Agent personas are roles, not campaigns** — `recon-investigator` not `acme-recon-agent`
8. **Wrappers are TypeScript only** — llmcli-tools pattern (index.ts + cli.ts, JSON output). No bash.
9. **Pipeline stages declare data contracts** — no implicit handoffs between agents
10. **One-off work stays in the work system** — don't assemble a skill for something you'll do once
11. **Skill-local vs shared tools** — if exactly one skill uses it, bundle in scripts/. If 2+ consumers need it, it's a shared utility (llmcli-tools package). When a "local" tool is obviously general-purpose, make it shared from the start.

---

## Agent Persona Format

```markdown
---
name: {role-slug}
model: sonnet
allowed-tools: [scoped tool list]
---
# {Role Name}

{Identity — WHO this agent is, 2-3 sentences}

## Expertise
[What this agent knows and is good at]

## Behavioral Constraints
[Boundaries and rules]

## Communication Style
[How this agent communicates]
```

Naming: role-based slugs (`recon-investigator`, `detection-builder`), never campaign-specific.

---

## Naming Conventions

- **Skills:** methodology-focused slugs — `recon-methodology`, `cve-triage`, `detection-authoring`
- **Wrappers:** tool-focused slugs — `nuclei-template-search`, `shodan-query`, `censys-lookup`
- **Agents:** role-based slugs — `recon-investigator`, `detection-builder`, `triage-analyst`
- **Pipelines:** workflow-focused slugs — `cve-response`, `perimeter-sweep`, `asset-discovery`

If you need "and" in any name, it is two things. Split it.

---

## Decision Quick Reference

| Question | Answer |
|----------|--------|
| Same input always produces same output? | Wrapper |
| Requires reasoning or judgment? | Skill |
| Will it be used once? | Work order, not a skill |
| Does it cross 2+ domains? | Split into multiple skills |
| Is it over ~200 lines? | Split along judgment boundaries |
| Would someone want just the first half? | Split |
| Does a tool serve exactly one skill? | Bundle in scripts/ |
| Does a tool serve 2+ skills? | Shared utility package |
