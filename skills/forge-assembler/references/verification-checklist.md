# Assembly Verification Checklist

Every artifact the assembler produces must pass verification before being committed to the armory. Two verification levels, applied based on artifact type.

---

## Level 1: Foundation Checks

Invoke the artifact-foundations validation skills for each artifact type. These check structural correctness — frontmatter validity, required sections, naming conventions.

### For each skill artifact:

```
Skill("artifact-foundations:skill-foundations")
```

**What it checks:**
- Frontmatter parses as valid YAML
- Required fields present: `name`, `description`, `allowed-tools`
- Description follows USE WHEN format
- Content structure matches pattern type (H1 title, sections, standards)
- No forward references to non-existent files

### For each command artifact:

```
Skill("artifact-foundations:command-foundations")
```

**What it checks:**
- Frontmatter parses as valid YAML
- Required fields present: `allowed-tools`, `description`
- Command references valid skills and external files
- No orphaned @reference directives

### PASS criteria (Level 1):
- Foundation skill returns no errors
- All required fields present and correctly formatted
- All internal references resolve

### FAIL criteria (Level 1):
- Any missing required field
- Malformed frontmatter
- Broken references to non-existent files or skills

---

## Level 2: Build and Integration Checks

Deeper verification that tests compilation, reference integrity, and runtime readiness.

### For tools:

```bash
cd <tool-dir>
# Detect single-file vs multi-file
if [ -f cli.ts ]; then
  bun build ./cli.ts --outdir /tmp/forge-verify-${RANDOM} --target bun
else
  TS_FILE=$(ls *.ts 2>/dev/null | head -1)
  bun build ./${TS_FILE} --outdir /tmp/forge-verify-${RANDOM} --target bun
fi
rm -rf /tmp/forge-verify-*/
```

**What it checks:**
- TypeScript compiles without errors
- Single-file: the .ts file builds correctly
- cli.ts has valid shebang and arg parsing
- package.json has correct dual exports (library + CLI)
- JSON output contract matches declared schema

### For skills:

**What it checks:**
- Frontmatter parses correctly (redundant with Level 1, but confirms no regression)
- All referenced files in `references/` directory exist
- All referenced tools in `tools/` directory exist
- Bundled reference content is non-empty and actionable (not placeholder stubs)

### For agent personas:

**What it checks:**
- Frontmatter has `name`, `model`, `allowed-tools`
- Content has Identity section, Expertise section, Behavioral Constraints, Communication Style
- Name follows role-based slug convention (not campaign-specific)

### For Pattern 3A orchestrator commands:

```
Skill("artifact-foundations:command-foundations")
```

**What it checks:**
- Frontmatter parses as valid YAML
- Required fields present: `allowed-tools`, `description`
- All Skill() references in the command body point to real skills/agents in Kit or the current assembly batch
- Command structure follows orchestration pattern (sequencing, data handoff, approval gates)

---

## PASS / FAIL Definitions

| Result | Meaning | Action |
|--------|---------|--------|
| Level 1 PASS | Structurally valid artifact | Proceed to Level 2 |
| Level 1 FAIL | Structural defect | Fix before proceeding. Do not attempt Level 2. |
| Level 2 PASS | Build-ready artifact | Commit to armory, register in Kit |
| Level 2 FAIL | Integration defect | Fix before committing. Report failure details. |

---

## Behavior on Failure

**Do NOT commit the artifact to the armory.** A failed artifact is not "close enough." Fix the issue and re-verify.

The assembler reports all verification failures with:
- Which level failed (1 or 2)
- Which specific check failed
- The error message or missing element
- Suggested fix

---

## Assembly Workflow Integration

Verification sits at steps 5-6 of the assembly workflow:

1. Read plan document
2. Load composition rules
3. Load artifact templates matching tier/pattern
4. Generate artifacts following templates and rules
5. **Level 1 verification** — invoke foundation skills per artifact
6. **Level 2 verification** — build checks, reference integrity
7. Write verified artifacts to forge-armory
8. `git add` + `git commit` in forge-armory
9. Run `kit add` for each artifact with campaign tags
11. Run `kit use` for each artifact
12. Update context.json with campaign entry
13. Return assembly report
