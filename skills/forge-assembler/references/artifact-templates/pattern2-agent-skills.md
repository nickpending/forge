# Pattern 2: Agent Skill Set Template

Use this template when composing the skill set for a Pattern 2 (first-class agent). A Pattern 2 agent loads Pattern 0 skills as its capabilities — the agent reasons, the skills provide methodology.

---

## Skill Set Structure

A Pattern 2 agent typically loads 2-4 Pattern 0 skills, each covering a distinct methodology concern.

### Example: Recon Investigator Skill Set

```
Agent: recon-investigator (Pattern 2 persona)

Loaded skills (Pattern 0, tagged loadable):
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

- **Minimum: 2** — an agent with one skill is likely a Pattern 1 (forked skill-agent) instead
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
| `loadable` | Pattern 0 skill — can be loaded by Pattern 2 agents |
| `fork` | Pattern 1 skill — runs as autonomous subprocess, NOT loadable |

Only `loadable` skills can be loaded by Pattern 2 agents. `fork` skills run as separate subprocesses and are invoked differently (as forked tasks, not methodology references).

---

## Designing a New Skill Set

When composing skills for a new Pattern 2 agent:

1. **Identify methodology concerns** — what distinct domains does this agent need to reason about?
2. **Apply composition rule 1** — one skill per methodology concern
3. **Apply composition rule 6** — "would someone ever want just the first half?" If yes, it is two skills
4. **Check for existing skills in Kit** — `kit list --type skill --domain security --tags loadable`
5. **Generate missing skills** using the Pattern 0 template
6. **Verify each skill** passes Level 1 foundation checks

### What NOT to do

- Do not create one monolithic skill that covers everything — split by domain
- Do not create skills so granular they are individual commands — that is Tier 1
- Do not create campaign-specific skills — skills are reusable across agents and campaigns
- Do not tag Pattern 1 (fork) skills as loadable — they have different execution contexts

---

## Example: Detection Builder Skill Set

```
Agent: detection-builder (Pattern 2 persona)

Loaded skills (Pattern 0, tagged loadable):
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

Each skill in the set registers via CLI after committing to forge-armory:

```bash
kit add --name {skill-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path skills/{skill-slug} \
  --type skill \
  --domain security \
  --tags {domain-tag},{methodology-tag},loadable,campaign:{skill-slug} \
  --description "One-line description of this methodology skill"
```

---

## Checklist

- [ ] Agent has 2-4 Pattern 0 skills covering distinct methodology domains
- [ ] No skill overlap — each covers a separate concern
- [ ] All skills tagged `loadable` (not `fork`)
- [ ] Each skill follows Pattern 0 template structure
- [ ] Existing Kit skills checked before generating new ones
- [ ] Each skill passes Level 1 foundation verification independently
