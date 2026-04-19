# Composition Rules

These rules govern how the assembler decomposes plans into artifacts. Apply all 15 rules during artifact generation.

---

## The 15 Rules

1. **One skill per methodology concern** — split when plan crosses 2+ domains with different tool sets
2. **Wrapper boundary** — deterministic (same input → same output) = wrapper. Reasoning required = skill. Never mix.
3. **Naming signal** — if you need "and" in the name, it's two things
4. **Granularity floor** — "run this one command" is direct tool use, not a skill. Skills encode methodology, not individual commands.
5. **Granularity ceiling** — if instructions exceed ~200 lines, split along judgment boundaries
6. **Reuse test** — "would someone ever want just the first half?" If yes, split.
7. **Agent personas are roles, not campaigns** — `recon-investigator` not `acme-recon-agent`
8. **Wrappers are TypeScript only** — llmcli-tools pattern (index.ts + cli.ts, JSON output). No bash.
9. **Pipeline stages declare data contracts** — no implicit handoffs between agents
10. **One-off work stays in the work system** — don't assemble a skill for something you'll do once
11. **Skill-local vs shared tools** — if exactly one skill uses it, bundle in tools/. If 2+ consumers need it, it's a shared utility (llmcli-tools package). When a "local" tool is obviously general-purpose, make it shared from the start.
12. **Forked skills have no conversation context** — Skills with `invocation_mode: forked` cannot ask the practitioner questions, present options, or wait for approval during execution. Any skill requiring user interaction MUST be `invocation_mode: inline`. All inputs to a forked skill must be provided upfront. The assembler lints for interaction-marker patterns at Level 1 verification (see `forge-artifacts.md` for the four categories).
13. **Automation configs co-locate with their tool** — Justfiles, cron configs, and n8n workflow JSONs live in `tools/{name}/` alongside the tool binary. Kit registers the tool binary only. Automation configs are armory-stored (in the tool directory) but never `kit add`'d. There is no separate `automations/` directory.
14. **Kit eligibility is fixed at five types** — Kit registers exactly five artifact types: `skill`, `tool`, `agent`, `command`, `harness`. All other artifacts (automation configs, Justfiles, workflow JSONs, plans) are armory-only. This is the Kit eligibility rule.
15. **Hook composition when the plugin model calls for it** — When a CONOPS specifies safety constraints, scope guardrails, or rate-limiting requirements, the assembler must produce hook artifacts alongside the main skill. Two hook patterns:
    - **PreToolUse hooks** (shell scripts, fast, deterministic): scope checking ("is this target in scope?"), rate limiting (check last-run timestamp before proceeding), input sanitization. Register in SKILL.md frontmatter under `hooks:` with the nested hook syntax:
      ```yaml
      hooks:
        PreToolUse:
          - matcher: "Bash"
            hooks:
              - type: command
                command: "bash tools/{hook-name}.sh"
      ```
    - **Stop hooks** (model-evaluated, prompt-based): quality gates that evaluate whether the skill's output meets exit criteria before stopping. Register in SKILL.md frontmatter under `hooks:` with the nested hook syntax:
      ```yaml
      hooks:
        Stop:
          - hooks:
              - type: prompt
                prompt: "Verify the skill output meets the exit criteria defined in the CONOPS before stopping."
      ```
    Hook scripts live in the skill's `tools/` subdirectory. Assembler adds hook entries to SKILL.md frontmatter and writes the shell scripts when CONOPS specifies safety or scope constraints. Hooks require a consuming skill (SKILL.md frontmatter) — standalone tool artifacts without a consuming skill cannot have hooks.

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
- **Commands:** workflow-focused slugs — `cve-response`, `perimeter-sweep`, `asset-discovery`
- **Harnesses:** project-focused slugs — `overnight-recon`, `n-day-response`, `queue-hunter`

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
| Does a tool serve exactly one skill? | Bundle in tools/ |
| Does a tool serve 2+ skills? | Shared utility package |
