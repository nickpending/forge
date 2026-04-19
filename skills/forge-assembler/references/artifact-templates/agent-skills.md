# Agent Skill Set Template

Use this template when composing the skill set for an agent. The agent reasons; the skills provide methodology. An agent loads inline skills as its capabilities.

---

## Skill Set Structure

An agent typically loads 2-4 inline skills, each covering a distinct methodology concern.

### Example: Recon Investigator Skill Set

```
Agent: recon-investigator

Loaded skills (inline, tagged `inline`):
├── recon-methodology        # HOW to approach reconnaissance
├── asset-classification     # HOW to classify discovered assets
└── recon-output-schema      # HOW to structure findings
```

### How Skills Coordinate

Each skill covers a **distinct methodology domain**. They do not overlap:

| Skill | Domain | What it provides |
|-------|--------|-----------------|
| `recon-methodology` | Process | Step-by-step approach: enumeration, validation, probing, correlation |
| `asset-classification` | Classification | Taxonomy for asset types, criticality ratings, exposure levels |
| `recon-output-schema` | Output | Structured format for findings: JSON schema, narrative template |

The agent decides **when** to apply each skill. The skills define **how** to do each part.

---

## Composition Guidelines

### How many skills?

- **Minimum: 2** — an agent with one skill is likely a forked skill instead (same artifact type, `invocation_mode: forked`)
- **Maximum: 4** — more than 4 skills suggests the agent's scope is too broad; split into two agents
- **Sweet spot: 2-3** — one for methodology, one for classification/judgment, one for output format

### How skills are loaded

The agent loads skills via `Skill()` references during execution:

```
Skill("recon-methodology")           # Load methodology for current phase
Skill("asset-classification")        # Load classification taxonomy for findings
Skill("recon-output-schema")         # Load output format for report generation
```

Skills are loaded on demand, not all at once. The agent loads the relevant skill when it enters a phase that requires that methodology.

### Tagging requirements

| Tag | Meaning |
|-----|---------|
| `inline` | Skill runs in conversation thread — loadable by agents as knowledge/methodology |
| `forked` | Skill runs as autonomous subprocess — invoked as a separate task, NOT loaded as methodology |

Only `inline` skills can be loaded by agents as methodology. `forked` skills are invoked as separate sub-tasks and produce structured output — they don't inject knowledge into the agent's context.

---

## Designing a New Skill Set

When composing skills for a new agent:

1. **Identify methodology concerns** — what distinct domains does this agent need to reason about?
2. **Apply composition rule 1** — one skill per methodology concern
3. **Apply composition rule 6** — "would someone ever want just the first half?" If yes, it is two skills
4. **Check for existing skills in Kit** — `kit list --type skill --domain security --tags inline`
5. **Generate missing skills** using the skill template (`artifact-templates/skill.md`, inline mode)
6. **Verify each skill** passes Level 1 foundation checks

### What NOT to do

- Do not create one monolithic skill that covers everything — split by domain
- Do not create skills so granular they are individual commands — that is direct tool use
- Do not create campaign-specific skills — skills are reusable across agents and campaigns
- Do not load forked skills as methodology — they have different execution contexts

---

## Example: Detection Builder Skill Set

```
Agent: detection-builder

Loaded skills (inline, tagged `inline`):
├── detection-methodology     # HOW to author detection rules
├── attack-technique-mapping  # HOW to map findings to MITRE ATT&CK
└── detection-output-schema   # HOW to format detection rules (Sigma, YARA, etc.)
```

| Skill | Domain | What it provides |
|-------|--------|-----------------|
| `detection-methodology` | Process | Approach for translating advisory/vuln data into detection logic |
| `attack-technique-mapping` | Classification | MITRE ATT&CK technique identification and mapping |
| `detection-output-schema` | Output | Rule format standards (Sigma, YARA, Suricata) with quality criteria |

---

## Kit Registration (per skill)

Each skill in the set registers individually:

```bash
kit add --name {skill-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path skills/{skill-slug} \
  --type skill \
  --domain security \
  --tags {domain-tag},{methodology-tag},inline,campaign:{skill-slug} \
  --description "One-line description of this methodology skill"
```

---

## Checklist

- [ ] Agent has 2-4 inline skills covering distinct methodology domains
- [ ] No skill overlap — each covers a separate concern
- [ ] All skills tagged `inline` (not `forked`)
- [ ] Each skill follows skill template structure (inline mode)
- [ ] Existing Kit skills checked before generating new ones
- [ ] Each skill passes Level 1 foundation verification independently
