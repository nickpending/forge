---
type: manifest
project: forge
generated: 2026-06-12
source: /Users/rudy/development/projects/forge/docs/architecture
reconciled_at: 405b3a2504f43e374c1c6c2341b09217c70e5ebc
---

### Components

- **forge-strategist** — CONOPS from sensemaker decomposition; composition model (forked, opus).
- **forge-planner** — Reads approved CONOPS, produces a plan (forked).
- **forge-assembler** — Generates artifacts, 3-level verification, commits to armory.
- **forge skill** — Workflow router: init, campaign, improve.
- **forge-armory** — Repo storing ALL artifact types. → components/forge-armory.md
- **kit-catalog** — Registry backing store. → components/kit-catalog.md
- **forge runtime** — Shared XDG config/state.
- **CONOPS** — Strategist→planner interface doc.
- **composition model** — Strategist refs: shapes × substrates, taxonomy, evals, data deps.

### Where to look

Base dir: /Users/rudy/development/projects/forge/docs/architecture/

- Overview: architecture.md
- Components: components.md
- Decisions: decisions.md
- Contracts: boundaries.md
