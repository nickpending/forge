# Plan Output Schema

The planner produces a structured plan document with YAML frontmatter and markdown body. This schema defines the exact format.

---

## YAML Frontmatter

```yaml
---
forge_version: "2.0"
created: 2026-04-16
intent: "free-form description of what the practitioner wants"
artifact_type: skill
complexity_score: 6
complexity_rationale: "why this complexity was assessed — references rubric signals: scope, judgment, repeatability, who drives, time horizon"
runtime: human_session
runner: claude-code
practitioner_level: senior
target: "target description"
scope: "scope boundaries"
tools_required: [nmap, nuclei, httpx]
components_available: []   # populated from Kit query
components_needed:
  - name: nuclei-template-search
    type: tool
    purpose: "deterministic CVE template lookup"
    reusability: reusable
    reusability_reason: "general-purpose search — useful to any CVE analysis composite"
  - name: cve-analysis
    type: skill
    purpose: "interpret search results, assess exploitability"
    invocation_mode: inline
    reusability: parent-scoped
    reusability_reason: "designed for this campaign's methodology but portable in theory"
    parent: "command:cve-response"
  - name: cve-response-bootstrap
    type: tool
    purpose: "initializes the CVE response pipeline state"
    reusability: private
    reusability_reason: "wires campaign-specific paths; no value outside this composite"
    parent: "command:cve-response"
harness_agents: []         # list of agent slugs the harness orchestrates (harness plans only)
harness_schedule: null     # cron expression or null (harness plans only)
harness_env: []            # env var names the harness requires (harness plans only)
validation: deferred   # or: level-1, level-2
kit_eligible_components: []   # skill, tool, agent, command, harness types — populated by planner
armory_only_components: []    # automation_config type and private-reusability components — armory-only, not Kit-registered — populated by planner
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
| `forge_version` | string | Schema version. `"2.0"` for the artifact-type model. |
| `created` | date | ISO date when the plan was generated. |
| `intent` | string | The practitioner's original request in their own words. Preserved verbatim. |
| `artifact_type` | enum | Primary artifact type for this plan. One of: `tool`, `skill`, `agent`, `command`, `automation_config`, `harness`. Determined by planner via the artifact-type decision tree in `forge-artifacts.md`. |
| `complexity_score` | integer (1-10) | Complexity assessment. Not a routing key — informs scaffolding density and validation depth. References rubric signals from `forge-artifacts.md`. |
| `complexity_rationale` | string | Explanation of the complexity score. Must reference: Scope, Judgment, Repeatability, Who drives, Time horizon. |
| `runtime` | enum | Target runtime for the primary artifact. One of: `human_session`, `scheduled_claude_code`, `agent_sdk_runtime`, `deterministic_pipeline`. Per-component runtime can differ — see `components_needed`. |
| `runner` | enum | Firing mechanism for deterministic components. One of: `none`, `justfile`, `cron`, `n8n`, `claude-code`. Describes HOW the runtime is triggered, not WHAT the runtime is. |
| `practitioner_level` | enum | One of: `senior`, `mid`, `junior`. Controls scaffolding density. |
| `target` | string | What is being assessed, scanned, or investigated. |
| `scope` | string | Boundaries of the engagement. What is in/out of scope. |
| `tools_required` | list[string] | Tools needed for execution. Used by assembler to check availability. |
| `components_available` | list | Populated from Kit query results. Empty if Kit is unavailable. |
| `components_needed` | list[object] | Components the assembler must select or create. Each entry has `name`, `type`, `purpose`, `reusability`, `reusability_reason`, and optionally `parent`. |
| `harness_agents` | list[string] | Agent slugs the harness orchestrates. Each gets Kit-resolved or built in the same batch. Only populated for harness plans. |
| `harness_schedule` | string or null | Cron expression if the harness runs on a schedule; `null` if user-launched. Only populated for harness plans. |
| `harness_env` | list[string] | Env var names the harness requires. Assembler emits matching entries in `.env.example`. Only populated for harness plans. |
| `validation` | enum | One of: `deferred`, `level-1`, `level-2`. Controls assembler verification depth. |
| `kit_eligible_components` | list | Subset of `components_needed` that are Kit-eligible (type is `skill`, `tool`, `agent`, `command`, or `harness`). Assembler runs `kit add` for these. |
| `armory_only_components` | list | Components that go into the armory but are NOT Kit-registered (`automation_config` type, and `private`-reusability components co-located with their parent). |
| `status` | enum | One of: `draft`, `approved`, `assembled`, `executed`. Lifecycle tracking. |

### Component Entry Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | YES | Slug identifier for the component. |
| `type` | enum | YES | One of: `skill`, `tool`, `agent`, `command`, `automation_config`, `harness`. |
| `purpose` | string | YES | What this component does in the context of this plan. |
| `reusability` | enum | YES | One of: `reusable`, `parent-scoped`, `private`. Determines Kit registration vs armory-only. |
| `reusability_reason` | string | YES | One-line justification for the reusability classification. |
| `parent` | string | when reusability ≠ reusable | Format: `{type}:{slug}`. Names the parent composite this component bundles with. Kit tag becomes `bundle:{parent}`. |
| `invocation_mode` | enum | when type == skill | One of: `inline`, `forked`. Default: `forked`. Determines whether the skill runs in the main conversation thread (inline) or as an isolated subprocess (forked). The assembler applies the forked-mode lint (interaction-marker categories in `forge-artifacts.md`) when `forked`. |

### Reusability → Registration Mapping

| Reusability | Kit registration | Bundle tag |
|-------------|-----------------|------------|
| `reusable` | `kit add` with component's own tags | None |
| `parent-scoped` | `kit add` with `bundle:{parent-type}:{parent-slug}` tag added | Yes |
| `private` | Skip Kit; co-locate in parent's armory directory | N/A |

### Runtime vs Runner

`runtime` describes the **process model** — where the artifact lives at execution time.
`runner` describes the **firing mechanism** — what triggers the runtime.

| Runtime | What it means | Typical runners |
|---------|---------------|-----------------|
| `human_session` | Practitioner in a Claude Code session | `claude-code` |
| `scheduled_claude_code` | `claude -p` fired unattended | `cron`, `n8n` |
| `agent_sdk_runtime` | Standalone Agent SDK program | `cron`, `none` (user-launched) |
| `deterministic_pipeline` | No LLM in control flow; tools execute directly | `justfile`, `cron`, `n8n` |

A single plan may have components targeting different runtimes. The top-level `runtime` field is the primary artifact's runtime; individual components may override.

### Markdown Body Sections

| Section | Purpose |
|---------|---------|
| **Objective** | What the campaign accomplishes and why it matters. |
| **Methodology** | Step-by-step approach with explicit decision points. Adapts density to practitioner level. |
| **Infrastructure Requirements** | What must be available before execution: tools, access, permissions, Docker. |
| **Success Criteria** | Measurable outcomes that define completion. The assembler uses these for validation. |
| **Results Format** | Expected output structure. Layered: structured core for models, narrative for humans. |

---

## Migration Note

This schema replaces the v1.0 schema which used `tier` (integer 1-5) as a routing key and `artifact_pattern` (integer 0-3) as the artifact shape selector. In v2.0:

- `tier` → split into `complexity_score` (rubric assessment, not routing) + `artifact_type` (direct selection, replaces tier→pattern cascade)
- `artifact_pattern` → replaced by `artifact_type` (explicit enum, not pattern number)
- `execution_mode` → absorbed into `runtime` (process model)
- `runner` → rescoped to firing mechanism only; `runtime` is the new companion field for process model

Plans generated under v1.0 are recognizable by the presence of `tier:` and `artifact_pattern:` fields with absent `artifact_type:` and `runtime:`. The campaign workflow falls back to v1.0 routing when these fields are detected.
