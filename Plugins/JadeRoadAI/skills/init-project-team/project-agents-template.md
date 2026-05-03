# {{PROJECT_NAME}} — Project CLAUDE.md

# This file goes in {project}/CLAUDE.md and is loaded alongside ~/.claude/CLAUDE.md.
# It defines project-specific context. Global agents (tech-lead, qa-lead, ux-lead,
# global agents (tech-lead, qa-lead, ux-lead, azure-solutions-architect) are defined in ~/.claude/agents/ — do NOT duplicate them here.
# Disciplines are imported below — they define the rules and standards for this project.

---

{{DISCIPLINE_IMPORTS}}

---

<project_context>
  <name>{{PROJECT_NAME}}</name>
  <codename>{{CODENAME}}</codename>
  <framework>{{FRAMEWORK}}</framework>
  <type>{{PROJECT_TYPE}}</type>
  <purpose>{{PURPOSE}}</purpose>
</project_context>

---

## Build Commands

```bash
{{BUILD_COMMAND}}                 # Build all projects
{{TEST_COMMAND}}                  # Run all tests (MUST pass before any work is complete)
{{RUN_COMMAND}}                   # Run locally
```

---

## Project Structure

```
{{PROJECT_NAME}}/
├── CLAUDE.md                    # This file
├── .claude/agents/              # Project subagent definitions
│   ├── {{CODENAME}}-tracker.md
│   ├── {{CODENAME}}-specialist.md
│   └── {{CODENAME}}-brand-manager.md
├── Agents/                      # Agent memory and knowledge
│   ├── Architect/               # No subagent — Tech-Lead reads directly
│   ├── Tracker/
│   ├── Specialist/
│   └── Brand-Manager/
├── src/
│   └── {{PROJECT_MODULES}}
├── tests/                       # Mirror src/ structure
├── tools/
├── config/
├── docs/
└── docker/
```

---

## Project-Specific Constraints

<constraints>
  <!-- Inherit global constraints (security, TDD, quality gates) from ~/.claude/CLAUDE.md -->
  <!-- Add only project-specific rules below -->
  {{PROJECT_CONSTRAINTS}}
</constraints>

---

## Agent Knowledge Architecture

Each agent has three layers of knowledge with distinct ownership:

| Layer | File | Owner | Survives Upgrade |
|-------|------|-------|-----------------|
| **Core** | `IDENTITY.md` | Plugin (shipped) | Overwritten |
| **Local** | `KNOWLEDGE.md` | User (your additions) | Preserved |
| **Session** | `Memory/*.md` | Agent (runtime data) | Preserved |

- **IDENTITY.md** — Who the agent IS. Shipped by the plugin, overwritten on upgrade.
- **KNOWLEDGE.md** — What you've TAUGHT the agent. Always safe to edit.
- **Memory/*.md** — What the agent has LEARNED. Written at runtime.

### Training Mode

By default, teaching an agent writes to `KNOWLEDGE.md` (local). To modify core identity for redistribution:

- "Update [agent-name] core info" or "Train [agent-name] for redistribution"

In training mode, changes write to `IDENTITY.md` instead.

---

## Project Subagents

Defined in `.claude/agents/`, these are proper Claude Code custom subagents with enforced model/tools.

### {{CODENAME}}-tracker (haiku)
State manager. Consulted at session start and before planning work.
- **Memory:** `Agents/Tracker/Memory/` (STATE.md, BLOCKERS.md, BACKLOG.md)

### {{CODENAME}}-specialist (sonnet)
Domain expert. Consulted before changes to business logic, data model, API, integrations, or dependency-sensitive operations.
- **Memory:** `Agents/Specialist/Memory/` (DOMAIN.md, API.md, INTEGRATIONS.md, STATUS.md, KNOWN_ISSUES.md)

### {{CODENAME}}-brand-manager (haiku)
Project brand specialist. Learns and enforces project/client-specific brand guidelines.
- **Memory:** `Agents/Brand-Manager/Memory/` (BRAND.md, AUDIT.md)

### Architect (memory only — no subagent)
Codebase structure, patterns, and architectural decisions. Tech-Lead reads these directly.
- **Memory:** `Agents/Architect/Memory/` (ARCHITECTURE.md, PATTERNS.md, DECISIONS.md)

---

## Coordination: Global ↔ Project

| Question | Who Answers |
|----------|-------------|
| "Should we build this? Does it fit this codebase?" | Global **tech-lead** (reads Architect memory directly) |
| "Is this secure/stable/tested?" | Global **qa-lead** |
| "Is this usable/accessible?" | Global **ux-lead** |
| "Does this match project/client brand?" | **{{CODENAME}}-brand-manager** |
| "What's the current state?" | **{{CODENAME}}-tracker** |
| "How does the domain work? Are deps healthy?" | **{{CODENAME}}-specialist** |

### Conflict Resolution

1. **Tech-Lead defers to Architect memory** on project-specific patterns (the memory explains why a pattern exists here)
2. **QA-Lead overrides everyone** on security (non-negotiable)
3. **QA-Lead overrides everyone** on test coverage (quality gates are universal)
4. **Escalate to {{OWNER_NAME}}** if two agents with veto power conflict

---

## Session Protocol

1. **Read {{CODENAME}}-tracker** STATE.md — understand current sprint position and test baseline
2. **Read {{CODENAME}}-tracker** BLOCKERS.md — know what's blocked before planning work
3. **Skim Architect DECISIONS.md** — if the session involves structural changes
4. **After work:** Update Tracker STATE.md (always) and relevant Specialist/Architect memory if decisions were made or domain knowledge was learned

---

## Test File Protection Hook

Configure in `.claude/settings.json` for this project:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "scripts/guard-test-edits.sh \"$FILE_PATH\"",
        "description": "Blocks modifications to existing test files (TDD protection)"
      }
    ]
  }
}
```

Creating new test files is allowed (TDD red phase). Editing existing tests requires explaining WHY + explicit approval.

---

## Placeholder Reference

| Placeholder | Example |
|-------------|---------|
| `{{PROJECT_NAME}}` | ProjectNova |
| `{{CODENAME}}` | Nova |
| `{{FRAMEWORK}}` | C# .NET 8 + React |
| `{{PROJECT_TYPE}}` | SaaS Platform |
| `{{PURPOSE}}` | Unified customer analytics dashboard |
| `{{BUILD_COMMAND}}` | dotnet build Nova.sln |
| `{{TEST_COMMAND}}` | dotnet test Nova.sln |
| `{{RUN_COMMAND}}` | dotnet run --project src/Nova.API |
| `{{PROJECT_MODULES}}` | Nova.Core/, Nova.API/, Nova.Engine/ |
| `{{PROJECT_CONSTRAINTS}}` | (project-specific rules) |
| `{{DISCIPLINE_IMPORTS}}` | `@~/.claude/agents/disciplines/development.md` (one line per discipline) |
