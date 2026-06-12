---
type: architecture
subtype: boundaries
project: "forge"
status: active
created: "2026-03-24"
updated: "2026-06-12"
tags: [architecture, boundaries]
---

# Boundaries

Interface contracts between components and external systems.

## sensemaker ↔ forge-strategist

**Between:** sensemaker agent (Iris Novak) ↔ forge-strategist skill, brokered by the /forge campaign workflow
**Contract:** The campaign workflow (Stage 2) spawns the `sensemaker` agent, which produces a 5-section decomposition report (Synthesized Intent, Key Decisions, Flagged Gaps, Artifact Sources, Scope Boundaries) written to `{SCOPE_DIR}/agents/reports/sensemaker-forge-{slug}.md`. After the practitioner approval gate (Stage 3), the workflow (Stage 4) passes the strategist `SENSEMAKER_REPORT` (path), `PRACTITIONER_OVERRIDES` (string, empty if approved as-is), `RAW_ASK` (verbatim), and `CONTEXT_PATH` via $ARGUMENTS. The strategist treats approved Key Decisions as given and does NOT re-litigate them.
**Constraints:** This replaced the prior ferret-disambiguation input (2026-06-04). If the sensemaker returns no report, the workflow proceeds to Stage 4 with the raw ask only — decomposition is valuable but non-blocking. The strategist must not re-resolve decisions the practitioner approved (churn).

## forge-strategist ↔ forge-planner (CONOPS contract)

**Between:** forge-strategist skill ↔ forge-planner skill, via the CONOPS document
**Contract:** Strategist writes a CONOPS to `~/.local/share/forge/conops/{slug}.md` (status `draft`; /forge sets `approved` before the planner runs) per `conops-schema.md`. Required frontmatter: `id`, `created`, `status`, `practitioner_level`, `artifact_type_candidate`, `runtime_candidate`, `complexity_rationale`, `composition_rationale`; conditional `substrate` (when artifact type is harness or command) and `harness_weight` (when harness). Required body sections include `## Data Dependencies` (five dimensions, ending in a Reachable/Blocked feasibility verdict) and `## Practitioner Context` (verbose — the sole channel for unstructured context to the forked planner). The planner consumes the CONOPS as its only practitioner-context source and validates `artifact_type_candidate` / `runtime_candidate` without re-litigating absent evidence.
**Constraints:** A `Blocked` data-dependency verdict halts CONOPS approval. The planner cannot ask the practitioner questions — anything missing from Practitioner Context is permanently lost. The new composition fields (`substrate`, `harness_weight`, `composition_rationale`) are emitted by the strategist but, as of baseline, are not yet read by the planner SKILL.md.

## Kit CLI ↔ kit-catalog

**Between:** Kit CLI ↔ kit-catalog repo (external, GitHub)
**Contract:** Kit accesses catalog via SSH git URL (`git@github.com:nickpending/kit-catalog.git`). Catalog is a YAML file (`kit-catalog.yaml`) with type-keyed sections. Kit clones to `~/.cache/kit/catalog/` and reads/writes locally, pushing changes back.
**Constraints:** SSH keys must be configured for GitHub. Catalog YAML must be valid. Kit CLI must be in PATH (`/Users/rudy/.bun/bin/kit`). state.yaml is lazy-created on first `kit use`, not on init.

## forge-assembler ↔ forge-armory

**Between:** forge-assembler skill ↔ forge-armory repo
**Contract:** Assembler writes generated artifacts to type-specific directories (skills/, tools/, agents/, commands/). Automation configs (Justfiles, cron, n8n workflows) co-locate with their tool in `tools/{name}/`. Plans written by forge-planner to `plans/`. Assembler commits and pushes after writing. Only Kit-eligible artifacts (skill/tool/agent/command) get `kit add`; automation configs are armory-only. INV-003 requires all artifacts be committed — no orphaned files.
**Constraints:** Armory repo must be clean before assembler writes (no uncommitted tracked changes). context.json must never appear in armory (INV-005). .gitignore guards must be in place. Level 3 execution testing must pass before armory commit (Step 6.5).

## forge-planner ↔ forge context

**Between:** forge-planner skill ↔ context.json file
**Contract:** Planner lazy-creates `context.json` on first invocation if it doesn't exist. Reads practitioner profile and environment state from it. Writes updated context after plan completion.
**Constraints:** context.json must NEVER enter git history in any repo (INV-005). .gitignore guards in both forge and forge-armory repos are the protection mechanism. The file location is determined by forge configuration (not yet specified — likely `~/.config/forge/context.json`).
