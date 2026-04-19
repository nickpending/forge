# Forge Artifact Shapes

Forge produces different artifact types depending on the work. Six types, each with different execution characteristics and assembler templates. The planner selects the artifact type based on the decision tree in `forge-artifacts.md`.

---

## Tool

Deterministic code. No LLM at runtime. Wrapper tools have structured JSON I/O; utility tools have CLI interfaces.

**Assembler template:** `artifact-templates/tool.md`

**Characteristics:**
- Same input always produces same output
- TypeScript only for wrapper tools (llmcli-tools pattern: index.ts + cli.ts, JSON output)
- Kit type: `tool`
- Install: `~/.local/bin/`

**When to use:** The mechanical part should never vary. AI's value is in what comes after the output, not in the execution itself.

---

## Skill

Methodology package loaded into a Claude Code session. The core artifact type — most forge output is skills.

**Assembler template:** `artifact-templates/skill.md`

**Two invocation modes** (constraint classifier, not deployment switch):

| `invocation_mode` | Runs in | May contain | Must not contain |
|-------------------|---------|-------------|------------------|
| `inline` | Main conversation thread | Approval gates, clarifying questions, option presentations, adaptive branching | (no restriction beyond general skill rules) |
| `forked` | Isolated subagent (`context: fork`) | Bounded autonomous work; all inputs known upfront; structured output | Any interaction-marker pattern (assembler lint rejects; see `forge-artifacts.md` for the four categories) |

**Selection rule:** Default to `forked`. Pick `inline` only when the skill requires practitioner interaction by design. A forked skill is invocation-agnostic — it runs correctly in both modes.

**Characteristics:**
- `SKILL.md` + optional `references/`, `tools/`, `workflows/` directories
- May ship helper tools and sub-agents inside the skill directory
- Forked skills: own model selection, scoped tool access, isolated context
- Inline skills: share conversation context, can reference prior discussion
- Kit type: `skill`
- Install: `~/.claude/skills/{name}/`

**When to use:** The task has a known methodology but requires judgment in execution — ordering, prioritization, interpretation of results. Also: teaching the model how to approach a class of work.

---

## Agent

Persistent persona with its own identity that invokes skills as capabilities. The agent is the reasoner; the skills are the knowledge it draws on.

**Assembler templates:** `artifact-templates/agent-persona.md` (WHO) + `artifact-templates/agent-skills.md` (WHAT skills it loads)

**Characteristics:**
- Agent has a persona (identity, expertise, constraints, communication style) separate from instructions
- Loads inline skills as methodology — agent provides reasoning, skills provide knowledge
- Sustained reasoning over complex problems
- Can invoke multiple skills during execution
- Consumer-agnostic: same `.md` file runs in human session, scheduled Claude Code, or Agent SDK harness
- Kit type: `agent`
- Install: `~/.config/sable/agents/`

**When to use:** Complex work requiring sustained judgment. Investigation, analysis, multi-step reasoning where the agent needs to adapt strategy.

---

## Command

In-Claude-Code orchestrator. Coordinates multiple agents via `Skill()` chain. The `/forge` command itself is a command artifact.

**Assembler template:** `artifact-templates/command.md`

**Characteristics:**
- Markdown file with `allowed-tools` and `description` in frontmatter
- Runs in main thread, shares conversation context
- Delegates domain work to agents via `Skill()` calls — the command routes, it does not do domain work
- Defines agent sequence, data handoff contracts, approval gates
- Each coordinated agent can be a forked skill or a full agent
- Kit type: `command`
- Install: `~/.claude/commands/{name}.md`

**When to use:** Repeatable multi-agent workflows that run inside Claude Code. Event-triggered responses where the practitioner is present.

---

## Automation Config

Scheduler / DAG definition with deterministic control flow. Can invoke tools directly and trigger LLM runtimes as leaf stages, but the orchestration logic itself is hardcoded — a config file decides next step, not an LLM.

**Assembler template:** `artifact-templates/automation-config.md`

**Characteristics:**
- Justfile recipe, cron entry, n8n DAG definition, or GitHub Action workflow
- No LLM in the control flow — LLMs only appear as leaf-stage transforms
- Co-locates with its parent tool in `tools/{name}/` (not a separate directory)
- **Armory-only** — never Kit-registered
- Typical runtimes: `deterministic_pipeline`

**When to use:** Scheduled data collection, enrichment pipelines, cron jobs, CI workflows. The workflow is fixed and repeatable; human judgment is not needed at runtime.

**Distinguishing from harness:** If an LLM decides what happens next, it's a harness. If a config file decides, it's an automation config. n8n with LLM nodes is still an automation config — n8n's routing engine is deterministic.

---

## Harness

TypeScript or Python project using the Claude Agent SDK. Standalone LLM-driven orchestrator that creates its own SDK runtime programmatically.

**Assembler template:** `artifact-templates/harness.md`

**Characteristics:**
- Full project: `package.json` / `pyproject.toml`, entrypoint, `.claude/` directory embedding agents/commands/MCP tools
- LLM-driven control flow — a root agent dynamically decides what runs next
- Defaults: TypeScript + bun (primary), Python + uv (alt); plan can override
- Verification: `bun run tsc --noEmit` (TS), `python -m py_compile` + import smoke (Python)
- Plan-schema fields: `harness_agents[]`, `harness_schedule`, `harness_env[]`
- Kit type: `harness`
- Install: user workspace dir (e.g. `~/forge-harnesses/{name}/`); `kit use` clones + installs deps; user owns launch

**When to use:** Repeatable multi-agent workflows that run outside Claude Code — scheduled, long-lived, or event-triggered on infrastructure you control.

**Distinguishing from command:** Commands orchestrate inside Claude Code (human present). Harnesses orchestrate standalone (unattended or infra-triggered).

---

## Artifact Type Selection

| Work shape | Artifact type |
|------------|---------------|
| Deterministic code, no LLM needed | `tool` or `automation_config` |
| Methodology or analysis in a session | `skill` |
| Sustained persona-driven reasoning | `agent` |
| Multi-agent orchestration inside Claude Code | `command` |
| Multi-agent orchestration outside Claude Code, LLM-driven | `harness` |
| Scheduled pipeline, deterministic control flow | `automation_config` |

See `forge-artifacts.md` for the full decision tree.

---

## Kit Tracking

Every artifact tracks its type:

- **Skills** — tagged with `invocation_mode` (`inline` or `forked`); inline skills can be loaded into agents as knowledge
- **Agents** — separate persona definitions referencing skills
- **Commands** — orchestrators coordinating agents/skills (Kit type: `command`)
- **Harnesses** — Agent SDK projects coordinating agents (Kit type: `harness`)
- **Tools** — deterministic executables (Kit type: `tool`)
- **Automation configs** — co-located with parent tool, armory-only

The assembler uses artifact type + invocation mode to make composition and registration decisions.
