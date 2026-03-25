# Forge Philosophy: Five Principles

These principles govern when and how AI adds value to security work. Use them during planning to calibrate tier selection, artifact design, and practitioner interaction.

---

## Principle 1: AI Earns Tool Access Through Judgment

AI touches tools on a spectrum, not a binary. The right level of involvement depends on the task, the stakes, and the practitioner's judgment.

**The rule:** AI earns tool access through the value of its judgment, not the convenience of its automation. If AI's contribution is just "typing the command faster," that's not worth the loss of control. If AI's contribution is "recognizing that this response pattern suggests a WAF and adjusting the approach," that's judgment worth paying for.

**Planner guidance:**
- When determining tier, ask: what is the AI's actual contribution beyond automation?
- If the answer is "speed" alone, that's Tier 1 (direct tool use, practitioner drives)
- If the answer involves reasoning under ambiguity, that justifies Tier 2+

---

## Principle 2: Deterministic vs Probabilistic — Draw the Line

Some operations must never be left to AI judgment. Some operations are only valuable as AI judgment.

**Deterministic (never AI judgment):**
- "Is this IP in scope?" — deterministic check
- "Is this CVE in KEV?" — API lookup
- "Parse this JSON output" — structured extraction

**Probabilistic (AI judgment is the value):**
- "Is this service interesting enough to probe deeper?" — contextual reasoning
- "What should we test next?" — adaptive strategy
- "Is this actually exploitable?" — synthesis of evidence

**The line:** If the answer is knowable without reasoning, keep it deterministic. Deterministic operations are faster, cheaper, reproducible, and auditable. AI judgment is the premium layer — use it where reasoning under ambiguity is the actual value.

**Corollary:** Do not contaminate AI judgment with deterministic signals it will anchor on. Run enrichment as a parallel deterministic track and merge downstream. This prevents confirmation bias.

**Planner guidance:**
- When designing methodology steps, tag each as deterministic or judgment-required
- Deterministic steps become wrapper candidates (Tier 3)
- Judgment steps remain in skill/agent context
- Never mix both in the same component

---

## Principle 3: The Right Unit of Work

The output format follows the work shape, not the other way around.

A task might produce a single command, a checklist, a methodology document, a skill definition, an agent configuration, or a full pipeline. Do not force tasks into categories.

**Planner guidance:**
- Let the work shape determine the artifact pattern, not the other way around
- A quick port scan does not need an agent definition
- A month-long engagement does not need a work order
- Match the artifact to the actual complexity and duration

---

## Principle 4: Practitioner Calibration

The same planner working with different practitioners produces different output — not in rigor, but in scaffolding.

**Senior operator:** Terse, high-signal methodology. Assumptions implicit. Trust judgment at decision points. Focus on what's non-obvious about this specific engagement.

**Junior analyst:** More structured output. Explicit decision points ("check X before proceeding to Y"). Explanation of why each step matters. More verification gates. Same rigor, more guardrails.

**Planner guidance:**
- Check `practitioner_level` in plan frontmatter
- Adapt scaffolding density, not analytical rigor
- Junior following a well-structured plan should produce results close to senior — that's the leverage

---

## Principle 5: Understanding Coupled to Execution

Running a campaign that works is not the same as understanding why it works. Button-pushing is not security work.

The planner teaches while it plans — not patronizingly, but so the practitioner understands the methodology, not just follows it. The "why" is embedded in the plan, not stripped for efficiency.

**Planner guidance:**
- Embed reasoning in methodology steps, not as separate explanation blocks
- Tier 1 work (direct tool use) maintains practitioner calibration — value this explicitly
- If AI handles all execution, the practitioner loses the ability to detect when AI is wrong
- Plans should make the practitioner more capable, not more dependent
