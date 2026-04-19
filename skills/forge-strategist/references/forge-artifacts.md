# Forge Artifacts: Six Types, Four Runtimes

Every security task produces artifacts. The artifact type is determined by work shape — what the artifact does, who consumes it, and how it executes. Artifact-type selection is part of the planner's expertise.

The primary axis is **artifact type × runtime**. Complexity (scope, judgment required, repeatability, who drives) informs scaffolding density and validation depth — it does not route artifact selection.

---

## Artifact Types

Six types. Each has a distinct shape, composition model, and Kit eligibility.

| Type | What it is | Composes | Kit-eligible | Armory path |
|------|-----------|----------|--------------|-------------|
| **Tool** | Deterministic code. No LLM at runtime. Binary, script, or wrapper. | — | Yes (`kit add --type tool`) | `tools/{name}/` |
| **Skill** | SKILL.md methodology package. May ship helper tools and sub-agents. Discovered by keyword/context in a Claude Code session. | Tools, sub-agents | Yes (`kit add --type skill`) | `skills/{name}/` |
| **Agent** | `.md` persona + instructions. Consumer-agnostic prompt artifact. Single file. | Skills, tools | Yes (`kit add --type agent`) | `agents/{name}/` |
| **Command** | Slash command `.md`. Human triggers explicitly. In-Claude-Code orchestrator — coordinates agents via `Skill()` chain. | Skills, agents, tools | Yes (`kit add --type command`) | `commands/{name}/` |
| **Automation config** | Scheduler / DAG definition. Orchestrates tools and can trigger LLM runtimes. No LLM in the config itself. Deterministic control flow. | Tools; triggers LLM runtimes as stages | **No** (armory-only) | Co-locates with parent tool in `tools/{name}/` |
| **Harness** | TypeScript or Python project using Claude Agent SDK. Has package manager, entrypoint, and typically its own `.claude/` directory embedding agents / commands / MCP tools. LLM-driven control flow. Standalone orchestrator. | Agents, skills, tools, MCP, commands | Yes (`kit add --type harness`) | `harnesses/{name}/` |

---

## Runtimes

Four runtimes. Artifacts target a runtime; the same artifact can run in multiple runtimes (skills and agents are consumer-agnostic).

| Runtime | What runs there | Typical runner (firing mechanism) |
|---------|-----------------|-----------------------------------|
| `human_session` | Commands, skills, agents (as sub-agents), tools. Practitioner at a terminal. | `claude-code` |
| `scheduled_claude_code` | Same as human_session, fired unattended. `claude -p` invoked by a scheduler. | `cron`, `n8n` |
| `agent_sdk_runtime` | Agents, skills, tools, MCP tools, commands — embedded in a harness project. | `cron`, user-launched |
| `deterministic_pipeline` | Tools directly. LLM artifacts invoked only as leaf-stage transforms, not control-flow drivers. | `justfile`, `cron`, `n8n` |

**Runtime is an attribute of invocation, not of the artifact.** One agent `.md` file runs in a human session, a scheduled `claude -p`, and inside an Agent SDK harness — same artifact, three runtimes.

---

## Orchestrator Trilemma

Three orchestrator shapes exist. The forcing question is **who decides the next step** — not just where the orchestrator runs.

| Where it runs | Control flow | Artifact type |
|---------------|--------------|---------------|
| In Claude Code | LLM-driven (`Skill()` chain) | **Command** |
| Outside Claude Code | **LLM-driven** (root agent dynamically decides next step) | **Harness** |
| Outside Claude Code | **Deterministic** (DAG/cron/Justfile; LLMs only as leaf-stage transforms) | **Automation config** |

**Examples:**
- n8n DAG with LLM classification nodes → **automation_config** (n8n's routing engine is deterministic; LLM is a leaf transform, not the decider)
- Agent SDK app where a root agent reads an event and dynamically spawns one of several subagents → **harness** (LLM decides next step)
- `/recon` command chaining `Skill("subdomain-enum")` then `Skill("http-probe")` then `Skill("correlator")` → **command** (orchestrated in Claude Code via LLM-reasoned `Skill()` chain)

---

## Artifact-Type Determination

The planner picks artifact type via a decision tree on work shape, not a rubric lookup.

```
What does this component do?
│
├─► Deterministic code, no LLM at runtime?
│   ├─► Orchestrates other tools/timers? ──────► automation_config
│   └─► Executes a function? ──────────────────► tool
│
├─► Orchestrates multiple agents as primary work?
│   ├─► Runs inside Claude Code? ──────────────► command
│   └─► Runs outside Claude Code?
│       ├─► LLM decides next step? ────────────► harness
│       └─► Config file decides next step? ────► automation_config
│
├─► Sustained persona-driven reasoning? ───────► agent
│
└─► Otherwise (methodology, analysis, bounded work) ──► skill
```

**Default when unsure:** `skill`. It's the most common artifact type and the one with the least infrastructure overhead.

---

## Invocation Modes (Skills Only)

Skills have two invocation modes. The mode is a **constraint classifier** — it determines what the skill is permitted to contain, enforced at assembly time.

| `invocation_mode` | What the skill MAY contain | What the skill MUST NOT contain | Assembler check |
|-------------------|-----------------------------|----------------------------------|-----------------|
| `inline` | Approval gates, clarifying questions, option presentations, adaptive branching based on practitioner input | (no restriction beyond general skill rules) | None specific to mode |
| `forked` | Bounded autonomous work; all inputs known before start; structured output to parent | Any pattern that requires conversational turn-taking with the practitioner mid-execution | Lint rejects interaction markers (see categories below); fails assembly on hit |

**Selection rule:** Default to `forked` when the skill has no interaction patterns. Pick `inline` only when the skill's design *requires* practitioner interaction. A `forked`-authored skill is invocation-agnostic by construction — it conforms to the stricter rules and therefore also runs correctly when invoked inline.

### Interaction-Marker Categories (Forked Lint)

The forked-mode lint rejects skills containing any of these four categories:

1. **Imperative requests for practitioner input.** Language directly instructing the skill to ask the human. Examples: "ask the practitioner," "prompt the user," "request confirmation from," "gather input from."
2. **Conditional waits on human response.** Constructs that pause execution until a human acts. Examples: "wait for approval," "do not proceed until," "pause and present," "block until confirmed."
3. **Option-presentation constructs.** Surfacing choices and expecting selection. Examples: "present options and wait," "offer the practitioner a choice," "which of these would you like."
4. **Adaptive branching on practitioner choice.** Decision logic depending on human input mid-execution. Examples: "if the practitioner says," "based on their response," "branch according to their selection."

Lint runs as part of Level 1 (structural) verification. Failure emits error category `interaction_marker_in_forked_skill` with the matched pattern and offending line cited, blocking Kit registration.

---

## Reusability Rubric and Bundle Tag Taxonomy

For every composite artifact (command, agent-with-skills, harness, or a skill with helper tools), each inner component is classified by reusability.

### Three Outcomes

| Reusability | Registration | Discovery |
|-------------|--------------|-----------|
| **Reusable** — genuinely useful to other composites | Kit-registered, no bundle tag | Normal Kit catalog |
| **Parent-scoped** — shipped alongside a parent; provenance matters | Kit-registered with `bundle:{parent-type}:{parent-slug}` tag | Searchable by parent |
| **Private** — only makes sense inside parent | Armory-only, co-located in parent directory | Not in Kit catalog |

### Bundle Tag Format

```
bundle:{parent-type}:{parent-slug}
```

- `parent-type` is the artifact type (`harness`, `command`, `agent`, `skill`)
- `parent-slug` is the artifact's slug
- Fully qualified to prevent slug collisions across types

**Examples:**
- `bundle:harness:overnight-recon`
- `bundle:command:forge`
- `bundle:agent:kira-voss`

### Invariants

1. **One-parent rule (batch-scoped).** Within a single assembly batch, an artifact with a bundle tag has exactly one bundle tag. If two parents want it in the same batch, it's reusable (no bundle tag). Cross-batch enforcement requires a Kit audit tool that does not exist today.
2. **Tag-to-immediate-parent only.** Don't chain tags through grandparents.
3. **Per-component explicit classification.** The plan's `components_needed[]` must carry `reusability` + `reusability_reason` per component.
4. **Slug scoping by type.** Slugs must be unique within an artifact type, not across all types.
5. **Promotion path.** An artifact classified `parent-scoped` or `private` today may become `reusable` later. Remove bundle tag or promote from armory to Kit. Administrative operation, not schema change.

### Query Patterns

- `kit search tag:bundle:harness:overnight-recon` → every Kit-registered inner artifact of the overnight-recon harness
- `kit search type:skill -tag:bundle:*` → only reusable skills (no bundle tag)

---

## Complexity Scoring

Complexity informs scaffolding density and validation depth. It does **not** route artifact selection.

| Signal | Low (1-3) | Medium (4-6) | High (7-10) |
|--------|-----------|--------------|-------------|
| Scope | Singular, known | Known approach, judgment in execution | Multi-step adaptive or repeatable pipeline |
| Judgment | None needed | Within guardrails or after deterministic output | Strategic throughout or at defined gates |
| Repeatability | One-off | Reusable methodology or mechanics | Event-triggered or ongoing |
| Who drives | Practitioner directs | Intent + review or intent only | Mission + approval gates or config + monitoring |
| Time horizon | Minutes | Session | Hours/days or ongoing |

**Disqualifying signals (cap complexity):** "I know exactly what I want" → 1-2. High-stakes novel target → cap at 3-4. First time doing this work → cap at 3-4.

**Promoting signals (raise complexity):** Output of step N changes step N+1 → 5+. "Run every time X happens" → 6+. Requires multi-agent coordination → 7+.

---

## Kit Eligibility

Five Kit-registered types:

| Type | Install location | Notes |
|------|-----------------|-------|
| `skill` | `~/.claude/skills/{name}/` | — |
| `tool` | `~/.local/bin/` | — |
| `agent` | `~/.config/sable/agents/` | — |
| `command` | `~/.claude/commands/{name}.md` | — |
| `harness` | User workspace dir (e.g. `~/forge-harnesses/{name}/`) | Project, not a single file. `kit use` clones + installs deps; user owns launch. |

**Automation configs are armory-only.** They co-locate with their parent tool in `tools/{name}/`. Kit does not register them.

---

## Planner Output by Artifact Type

| Artifact type | Planner produces | Assembler involvement |
|---------------|------------------|-----------------------|
| tool | Plan with tool spec + output contract | Yes — assembler builds from template |
| skill | Plan with methodology description + invocation_mode | Yes — assembler builds from template |
| agent | Plan with persona + skill set + investigation approach | Yes — assembler builds from template |
| command | Plan with agent sequence + data contracts + approval gates | Yes — assembler builds orchestrator + all inner components |
| automation_config | Plan with DAG structure + stage contracts | Yes — assembler builds config + any required tools |
| harness | Plan with harness_agents + harness_schedule + harness_env + orchestration logic | Yes — assembler builds project scaffold + inner components |

### Routing

- If plan's `components_needed[]` is empty and all required components are Kit-available → **direct execution** (create work order, no assembler)
- If plan's `components_needed[]` has items the assembler must create → **invoke assembler**
