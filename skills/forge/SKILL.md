---
name: forge
description: Security campaign orchestrator. USE WHEN user says "forge", "run forge", "new campaign", "improve campaign", "forge init", or needs to build/improve security tooling.
argument-hint: init|improve|<security intent>
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
user-invocable: true
---

# Forge — Security Campaign Orchestrator

Coordinates security campaign lifecycle: disambiguation, strategy, pressure testing, planning, and assembly. Also manages the forge runtime and improves existing campaign artifacts.

## Determine Action

Parse `$0` for the action keyword.

| `$0` | Workflow |
|------|----------|
| init | `${CLAUDE_SKILL_DIR}/workflows/init.md` |
| improve | `${CLAUDE_SKILL_DIR}/workflows/improve.md` |
| (anything else / empty) | `${CLAUDE_SKILL_DIR}/workflows/campaign.md` |

READ the matching workflow file and follow it. Pass `$ARGUMENTS` through to the workflow.
