# CONOPS Schema (Concept of Operations)

The CONOPS is the interface contract between forge-strategist and forge-planner. The strategist writes it; the planner reads it. The planner runs in a forked context with no conversation — anything missing from the CONOPS is permanently lost.

---

## Write Path

`~/.local/share/forge/conops/{slug}.md`

Slug convention: kebab-case from campaign intent. Examples: `fqdn-http-sweep`, `internal-perimeter-assessment`, `cve-2026-1234-detection`.

---

## YAML Frontmatter

```yaml
---
id: {slug}
created: 2026-03-27
status: draft          # draft | approved (set to approved before planner runs)
practitioner_level: senior   # senior | mid | junior (inferred by strategist)
artifact_type_candidate: skill   # strategist's recommended artifact type (tool | skill | agent | command | automation_config | harness)
runtime_candidate: human_session  # strategist's recommended runtime (human_session | scheduled_claude_code | agent_sdk_runtime | deterministic_pipeline)
complexity_rationale: "Scope: multi-step (probe→classify→rank). Judgment: after deterministic output. Repeatability: reusable across datasets. Who drives: mission-level. Time: session."
---
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | YES | Slug from campaign intent (kebab-case) |
| `created` | YES | ISO date |
| `status` | YES | `draft` until practitioner approves; `/forge` sets to `approved` before invoking forked planner |
| `practitioner_level` | YES | Inferred by strategist during conversation — never ask directly |
| `artifact_type_candidate` | YES | Strategist's recommended artifact type based on investigation. One of: `tool`, `skill`, `agent`, `command`, `automation_config`, `harness`. Determined via the artifact-type decision tree in `forge-artifacts.md`. The planner validates or overrides based on its own investigation. |
| `runtime_candidate` | YES | Strategist's recommended runtime. One of: `human_session`, `scheduled_claude_code`, `agent_sdk_runtime`, `deterministic_pipeline`. |
| `complexity_rationale` | YES | Rubric signals assessing complexity. Must reference: Scope, Judgment, Repeatability, Who drives, Time horizon. The planner uses this for scaffolding density and validation depth — not for artifact-type routing. |

---

## Markdown Body Sections

### `## Intent`

The practitioner's original statement, preserved verbatim. Do not paraphrase, summarize, or clean up.

### `## Target`

What is being assessed. Be specific: system, domain, dataset, organization, IP range, application. Not "a network" — "the 7,600 resolvable hosts from the FQDN dataset at /Volumes/datasets/fqdns.txt."

### `## Scope`

Two subsections:

- **In scope:** what is included in this campaign
- **Out of scope:** what is explicitly excluded

Boundaries must be unambiguous. The planner cannot ask for clarification.

### `## Flow`

How information moves through the campaign: input to processing to output. Free-form prose describing the data flow at a conceptual level. No implementation language — describe what happens, not how it is coded.

### `## Phases`

Numbered list. Each phase has:

| Element | Description |
|---------|-------------|
| **Name** | Short descriptive name |
| **Description** | What this phase accomplishes |
| **Inputs** | What data or access this phase requires |
| **Outputs** | What this phase produces |
| **Classification** | `deterministic` (same input always produces same output) or `judgment` (requires AI reasoning) |

Example:
```
1. **Host Discovery**
   - Description: Resolve FQDN dataset to identify live HTTP hosts
   - Inputs: Raw FQDN list
   - Outputs: List of resolvable hosts with HTTP/HTTPS status
   - Classification: deterministic
```

### `## Gotchas`

Named risks and fragile assumptions the strategist identified during investigation. Each gotcha has:
- **Name:** short identifier
- **Description:** what could go wrong
- **Impact:** what happens if this materializes

### `## Open Assumptions`

Things the strategist could not resolve from available evidence. Each assumption states:
- What is assumed
- Why it could not be verified
- What breaks if the assumption is wrong

The planner flags these in the plan as open questions. They do not block plan production.

### `## Practitioner Context`

Free-form section for constraints, preferences, prior attempts, goals, and any context that does not fit the structured fields above.

**CRITICAL: Write verbosely.** The forked planner cannot ask questions. Everything the practitioner shared during conversation that is not captured in structured fields above MUST go here. Include:
- Prior tools and approaches tried
- Why they worked or did not work
- Workflow preferences
- Infrastructure constraints
- Time and resource constraints
- Specific goals beyond the stated intent
- Anything the practitioner said that shaped the strategist's thinking

Err on the side of too much context. A verbose Practitioner Context section costs the planner nothing. A thin one costs the planner everything.

### `## Pressure Test Findings`

Left empty by the strategist. Appended by the `/forge` command after the hacker agent runs its CONOPS critique. Contains three sections:
- Fragile Assumptions
- Missing Elements
- Likely Failure Modes

---

## Required vs Optional Sections

**Required** (CONOPS is invalid without these — planner will hard-stop on missing required fields):
- Intent (verbatim from practitioner)
- Target (specific, not generic)
- Scope (in/out boundaries)
- Flow (conceptual data movement)
- Phases (numbered, each with inputs/outputs/classification)
- Practitioner Context (verbose — the ONLY channel for unstructured context to reach the forked planner)
- All frontmatter fields (id, created, status, practitioner_level, artifact_type_candidate, runtime_candidate, complexity_rationale)

**Optional** (can be empty if genuinely none identified):
- Gotchas (empty if no risks found — but this is unlikely)
- Open Assumptions (empty if all assumptions verified)
- Pressure Test Findings (appended by /forge, empty until then)

## Completeness Checklist

Before writing the CONOPS, verify ALL required sections:

- [ ] Intent is verbatim from practitioner
- [ ] Target is specific (not generic)
- [ ] Scope boundaries are explicit (in/out)
- [ ] Flow describes data movement without implementation language
- [ ] Every phase has inputs, outputs, and classification
- [ ] Practitioner Context is verbose — nothing from conversation is lost
- [ ] artifact_type_candidate is set based on the decision tree in forge-artifacts.md
- [ ] runtime_candidate is set based on consumer + invocation context
- [ ] complexity_rationale includes all 5 rubric signals (Scope, Judgment, Repeatability, Who drives, Time horizon)
- [ ] practitioner_level is inferred and recorded

A CONOPS that fails this checklist will produce a bad plan. The planner cannot compensate for missing information.
