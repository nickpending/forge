# Plan Output Schema

The planner produces a structured plan document with YAML frontmatter and markdown body. This schema defines the exact format.

---

## YAML Frontmatter

```yaml
---
forge_version: "1.0"
created: 2026-03-24
intent: "free-form description of what the practitioner wants"
tier: 3
tier_rationale: "why this tier was selected"
artifact_pattern: 1
execution_mode: interactive
practitioner_level: senior
target: "target description"
scope: "scope boundaries"
tools_required: [nmap, nuclei, httpx]
components_available: []   # populated from Kit query
components_needed:
  - name: nuclei-template-search
    type: wrapper
    purpose: "deterministic CVE template lookup"
  - name: cve-analysis
    type: skill
    purpose: "interpret search results, assess exploitability"
validation: deferred   # or: level-1, level-2
status: draft
---
```

## Markdown Body

```markdown
# Campaign Plan: [Name]

## Objective
[What we're trying to accomplish and why]

## Methodology
[Step-by-step approach with decision points]

## Infrastructure Requirements
[What needs to be available]

## Success Criteria
[How we know it worked]

## Results Format
[What the output looks like]
```

---

## Field Descriptions

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `forge_version` | string | Schema version. Always "1.0" for this iteration. |
| `created` | date | ISO date when the plan was generated. |
| `intent` | string | The practitioner's original request in their own words. Preserved verbatim. |
| `tier` | integer (1-5) | Selected tier of AI involvement. See forge-tiers.md for rubric. |
| `tier_rationale` | string | Explanation of why this tier was selected. References signals from the rubric. |
| `artifact_pattern` | integer (0-3) | Which artifact pattern to produce. See forge-patterns.md. |
| `execution_mode` | enum | One of: `interactive`, `automated`, `scheduled`. Planner recommends; practitioner decides. |
| `practitioner_level` | enum | One of: `senior`, `mid`, `junior`. Controls scaffolding density (Principle 4). |
| `target` | string | What is being assessed, scanned, or investigated. |
| `scope` | string | Boundaries of the engagement. What is in/out of scope. |
| `tools_required` | list[string] | Tools needed for execution. Used by assembler to check availability. |
| `components_available` | list | Populated from Kit query results. Empty if Kit is unavailable. |
| `components_needed` | list[object] | Components the assembler must select or create. Each entry has `name`, `type`, and `purpose`. |
| `validation` | enum | One of: `deferred`, `level-1`, `level-2`. Controls assembler verification depth. |
| `status` | enum | One of: `draft`, `approved`, `assembled`, `executed`. Lifecycle tracking. |

### Component Entry Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Slug identifier for the component. |
| `type` | enum | One of: `skill`, `wrapper`, `agent`, `command`. |
| `purpose` | string | What this component does in the context of this plan. |

### Markdown Body Sections

| Section | Purpose |
|---------|---------|
| **Objective** | What the campaign accomplishes and why it matters. Embeds reasoning per Principle 5. |
| **Methodology** | Step-by-step approach with explicit decision points. Adapts density to practitioner level. |
| **Infrastructure Requirements** | What must be available before execution: tools, access, permissions, Docker. |
| **Success Criteria** | Measurable outcomes that define completion. The assembler uses these for validation. |
| **Results Format** | Expected output structure. Layered: structured core for models, narrative for humans. |
