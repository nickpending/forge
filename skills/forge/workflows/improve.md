# /forge improve <artifact-name>

Improve an existing forge-armory artifact against current pipeline capabilities. Reads the artifact, loads current forge references, identifies gaps via AI judgment, presents proposed changes for practitioner approval, and applies approved changes.

## Step 1: Locate Artifact

Parse `$1` for the artifact name.

If no artifact name provided, ask:

> Which artifact do you want to improve? (e.g., http-recon, detection-builder)

Locate the artifact in forge-armory:

```bash
test -d ~/development/projects/forge-armory/skills/$1 && echo "FOUND: skill" || \
test -d ~/development/projects/forge-armory/tools/$1 && echo "FOUND: tool" || \
test -f ~/development/projects/forge-armory/commands/$1.md && echo "FOUND: command" || \
test -f ~/development/projects/forge-armory/agents/$1.md && echo "FOUND: agent" || \
echo "NOT_FOUND"
```

If NOT_FOUND: report what was searched and stop.

READ the artifact in full — SKILL.md for skills, the .md file for commands/agents, all files in the directory for tools.

## Step 2: Load Forge References

Read current forge pipeline references to establish what "current" looks like:

- READ `~/.claude/skills/forge-strategist/references/forge-timing.md` — timing profiles
- READ `~/.claude/skills/forge-strategist/references/forge-philosophy.md` — five principles
- READ `~/.claude/skills/forge-strategist/references/forge-tiers.md` — tier rubric
- READ `~/.claude/skills/forge-assembler/references/forge-runtime.md` — config/state/ledger contract
- READ `~/.claude/skills/forge-assembler/references/composition-rules.md` — composition rules

These are installed versions (via Kit). The improve flow compares the artifact against these references.

**Graceful degradation:** If any reference file is missing, log which could not be loaded. Continue with loaded references. Flag skipped references in the analysis output.

## Step 3: Analyze and Propose

With the artifact content and all loaded references in context, identify every gap between what the artifact does and what current forge capabilities define.

### Tool Change Decision

When a gap involves a tool (not just skill instructions), classify the change:

#### Skill-only

- IF: The tool already accepts what the skill needs to pass (flag exists, interface exists)
- THEN: Change the skill only. Read from config, pass the values.
- EXAMPLES:
  - Tool has `--rate-limit` flag but the skill never passes it → add flag to skill invocation
  - Tool reads from config file but skill doesn't write the field → add field to config defaults

#### Mechanical tool change

- IF: The tool lacks the interface AND adding it follows an existing pattern in the tool with no design decisions
- THEN: Apply the tool change directly alongside the skill change.
- EXAMPLES:
  - Tool accepts `--rate-limit` and `--threads` via identical CLI parsing. Adding `--timeout` follows the same pattern → add it
  - Tool has a ProbeConfig interface with `rateLimit` and `threads`. Adding `timeout` follows the same shape → add it

#### Architectural (recommend only)

- IF: The change alters default behavior, requires new dependencies, restructures interfaces, or affects consumers beyond this skill
- THEN: Report as a recommendation with rationale. Do not apply. Flag as a separate work item.
- EXAMPLES:
  - Changing a tool's default rate limit from 50 to 40 → design decision about what the default means
  - Adding a new output format the tool doesn't currently support → structural change
  - Refactoring how the tool reads config → affects all consumers

Produce a structured list of proposed changes:

```
### Change N: <short title>

**Layer:** skill | tool (mechanical) | tool (architectural — recommend only)
**Old:** <exact current text, or "missing" if the artifact lacks this entirely>
**New:** <proposed replacement or addition>
**Reason:** <which reference defines this and why the artifact needs it>
**Depends on:** <other change numbers, if interdependent — omit if independent>
```

Present the full list:

> **Improvement analysis for {artifact-name}:**
>
> Found {N} gaps against current forge capabilities.
>
> {change list}
>
> **Approve all, reject all, or specify which changes to apply (e.g., "apply 1, 3, skip 2").**

Flag interdependencies explicitly — if approving Change A without Change B creates an inconsistent state, say so.

STOP. Wait for practitioner response before proceeding.

**Approval parsing:**
- "approve all" / "yes" / "all" / "go" → apply all changes
- "reject all" / "no" / "skip" → stop, report no changes made
- Partial selection (e.g., "apply 1, 3") → apply only listed changes
- If practitioner approves a change but rejects its dependency, warn and confirm

## Step 4: Apply Changes

For each approved change, apply the edit to the artifact file in forge-armory using Edit. Preserve unapproved content exactly.

After all edits:

```
Applied {N} of {M} proposed changes to {artifact-path}.
Skipped: {list of skipped change titles, if any}
```

## Step 5: Commit and Install

```bash
cd ~/development/projects/forge-armory && \
git add skills/{artifact-name}/ && \
git commit -m "improve({artifact-name}): apply forge pipeline updates" && \
git push
```

Then run:
```bash
kit sync
```

Report what was committed, pushed, and synced.
