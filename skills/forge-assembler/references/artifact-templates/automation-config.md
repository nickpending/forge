# Automation Config Template

Use this template when generating deterministic pipeline configurations. Automation configs orchestrate tools and can trigger LLM runtimes as leaf stages, but the control flow itself is hardcoded — a config file decides what runs next, not an LLM.

**Armory-only.** Automation configs are never Kit-registered. They co-locate with their parent tool in `tools/{name}/`.

---

## When to Use

- Scheduled data collection (cron)
- Enrichment pipelines (n8n DAG)
- Build/CI workflows (GitHub Actions, Justfile)
- Overnight recon collection with deterministic ordering
- Any workflow where the sequence is fixed and repeatable

**Distinguishing from harness:** If an LLM decides what happens next, it's a harness. If a config file decides, it's an automation config. An n8n DAG with LLM classification nodes is still an automation config — n8n's routing engine is deterministic; the LLM is a leaf transform.

---

## Supported Formats

| Format | When to use | Location |
|--------|-------------|----------|
| Justfile | Local task automation, make-like recipes | `tools/{parent-name}/Justfile` |
| Cron entry | Scheduled recurring execution | `tools/{parent-name}/crontab` |
| n8n workflow | Complex DAGs with conditional routing, webhooks, data transforms | `tools/{parent-name}/{workflow-name}.n8n.json` |
| GitHub Action | CI/CD, triggered by git events | `.github/workflows/{name}.yml` |

---

## Justfile Template

```makefile
# {parent-tool-name} automation
# Co-located with tool at tools/{parent-name}/

# Collect: run the deterministic collection tool
collect target:
    ./{parent-tool-name} {{target}} > data/{{target}}-$(date +%Y%m%d).json

# Enrich: augment collected data with additional sources
enrich target:
    ./{enricher-tool-name} data/{{target}}-*.json > data/{{target}}-enriched.json

# Triage: invoke LLM agent for triage (leaf stage — deterministic routing, LLM as transform)
triage target:
    claude -p "Read data/{{target}}-enriched.json and invoke the host-triage agent. Write dossier to data/{{target}}-dossier.md"

# Full pipeline: collect → enrich → triage
pipeline target: (collect target) (enrich target) (triage target)
    @echo "Pipeline complete for {{target}}"
```

---

## Cron Template

```
# {parent-tool-name} scheduled run
# Install: crontab -e and paste this block

# Run collection at 2am daily
0 2 * * * cd ~/forge-tools/{parent-name} && just collect {default-target} >> /tmp/{parent-name}.log 2>&1

# Run full pipeline weekly (Sunday 3am)
0 3 * * 0 cd ~/forge-tools/{parent-name} && just pipeline {default-target} >> /tmp/{parent-name}-pipeline.log 2>&1
```

---

## n8n Workflow Template (Structure)

n8n workflows are JSON exports. The assembler produces a skeleton that the practitioner imports and configures.

Key nodes:
1. **Trigger** — cron schedule, webhook, or manual
2. **Execute Command** — invoke the parent tool
3. **IF** — deterministic routing based on tool output fields
4. **HTTP Request** / **Execute Command** — enrichment stages
5. **Execute Command** — `claude -p` for LLM-as-leaf-stage (optional)
6. **Write File** / **HTTP Request** — persist results

The assembler documents the expected node structure and data contracts. The practitioner wires it in the n8n editor.

---

## Key Rules

- **Deterministic control flow only.** The config decides what runs next based on fixed rules (cron schedule, Justfile recipe order, n8n routing conditions). LLMs may appear as leaf transforms (e.g. `claude -p` to classify records) but never as control-flow drivers.
- **Co-locate with parent tool.** Automation configs live in `tools/{parent-name}/` alongside the tool binary. Not in a separate `automations/` directory.
- **Never Kit-registered.** Automation configs are armory-only. Kit registers the parent tool; the automation config ships as part of the tool's directory.
- **Data contracts between stages.** Each stage's output format must be documented — no implicit handoffs. Downstream stages must know what format upstream produces.

---

## Forge Runtime

Automation configs do not have their own forge runtime preamble (they are not skills or agents). The *tools* they invoke have runtime preambles per `forge-runtime.md`. The config just sequences those tools.

---

## Checklist

- [ ] Control flow is deterministic (no LLM-driven routing)
- [ ] Co-located in `tools/{parent-name}/`
- [ ] Not Kit-registered (armory-only)
- [ ] Data contracts documented between stages
- [ ] LLM stages (if any) are leaf transforms only — `claude -p` invocations, not control-flow decisions
- [ ] Schedule or trigger is explicitly specified
- [ ] Parent tool exists (automation config always accompanies a tool)
