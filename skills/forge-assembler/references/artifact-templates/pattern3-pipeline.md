# Pattern 3: Multi-Agent Pipeline Template

Use this template when generating pipeline definitions for Tier 5 plans. Pipelines coordinate multiple Pattern 1 and Pattern 2 agents with explicit data contracts and approval gates.

---

## Pipeline Definition

```yaml
---
name: {pipeline-slug}
description: "{What this pipeline does end-to-end}"
trigger: "{event or schedule that starts the pipeline}"
version: "1.0"
---
```

## Stage Definitions

```yaml
stages:
  - name: {stage-1-slug}
    agent: {agent-ref}                # Pattern 1 or Pattern 2 agent
    pattern: 1                        # Which pattern the agent uses
    input:
      type: "{input-type}"           # What this stage receives
      source: "trigger"              # Where input comes from (trigger, previous stage)
      schema:
        target: string
        raw_data: string
    output:
      type: "{output-type}"          # What this stage produces
      schema:
        findings: array
        confidence: number
        summary: string
    approval_gate: false              # Does a human review before next stage?

  - name: {stage-2-slug}
    agent: {agent-ref}
    pattern: 2
    input:
      type: "{input-type}"
      source: "{stage-1-slug}"       # Receives output from stage 1
      schema:
        findings: array
        confidence: number
        summary: string
    output:
      type: "{output-type}"
      schema:
        prioritized_findings: array
        recommended_actions: array
    approval_gate: true               # Human reviews before proceeding

  - name: {stage-3-slug}
    agent: {agent-ref}
    pattern: 1
    input:
      type: "{input-type}"
      source: "{stage-2-slug}"
      schema:
        prioritized_findings: array
        recommended_actions: array
    output:
      type: "{output-type}"
      schema:
        artifacts: array
        validation_results: object
    approval_gate: false
```

---

## Example: CVE Response Pipeline

```yaml
---
name: cve-response
description: "End-to-end CVE analysis, triage, research, and detection generation"
trigger: "New CVE advisory published (manual or webhook)"
version: "1.0"
---

stages:
  - name: analysis
    agent: vuln-analyst
    pattern: 1
    input:
      type: "cve-advisory"
      source: "trigger"
      schema:
        cve_id: string
        advisory_text: string
        source_url: string
    output:
      type: "initial-analysis"
      schema:
        cve_id: string
        severity_assessment: string
        affected_components: array
        attack_vector: string
        exploitability: string
    approval_gate: false

  - name: triage
    agent: triage-analyst
    pattern: 2
    input:
      type: "initial-analysis"
      source: "analysis"
      schema:
        cve_id: string
        severity_assessment: string
        affected_components: array
    output:
      type: "triage-decision"
      schema:
        priority: string
        detection_needed: boolean
        rationale: string
        recommended_detection_types: array
    approval_gate: true

  - name: detection
    agent: detection-builder
    pattern: 2
    input:
      type: "triage-decision"
      source: "triage"
      schema:
        cve_id: string
        recommended_detection_types: array
        rationale: string
    output:
      type: "detection-rules"
      schema:
        rules: array
        validation_results: object
        coverage_assessment: string
    approval_gate: false
```

---

## Data Contracts

Every stage declares its input and output schema explicitly. No implicit handoffs.

### Contract Rules

1. **Output schema of stage N must be compatible with input schema of stage N+1**
2. **Every field in the input schema must be populated by the source stage's output**
3. **Extra fields in output are allowed** (forward compatibility) but unused by the next stage
4. **Missing required fields cause pipeline failure** — the pipeline halts, does not guess

### Contract Verification

The assembler verifies contract compatibility during Level 2 checks:
- Parse all stage schemas
- For each stage transition, confirm output fields satisfy next stage's input requirements
- Report any schema mismatches as Level 2 verification failures

---

## Approval Gates

| Value | Behavior |
|-------|----------|
| `false` | Stage output passes directly to next stage |
| `true` | Pipeline pauses. Human reviews output. Approves, modifies, or rejects. |

**When to use approval gates:**
- After triage/prioritization decisions (high-impact choices)
- Before actions that are expensive or irreversible
- At any point where human judgment improves outcomes

**When NOT to use:**
- Between deterministic stages (mechanical operations)
- Where delay defeats the purpose (real-time response)

---

## Kit Manifest

```yaml
name: {pipeline-slug}
type: pipeline
repo: git@github.com:nickpending/forge-armory.git
path: pipelines/{pipeline-slug}/
domain: security
tags: [{domain-tag}, {workflow-tag}, pipeline]
description: One-line description of what this pipeline does end-to-end
```

---

## Checklist

- [ ] Pipeline has `name`, `description`, `trigger`, `version` in frontmatter
- [ ] Each stage has `name`, `agent`, `pattern`, `input`, `output`, `approval_gate`
- [ ] Input `source` references either "trigger" or a valid previous stage name
- [ ] Output schema of each stage satisfies input schema of the next stage
- [ ] All referenced agents exist in Kit or are included in the assembly batch
- [ ] Approval gates are explicitly declared (not defaulted)
- [ ] At least one approval gate exists for pipelines with high-impact decisions
- [ ] kit-manifest.yaml type is `pipeline`
