---
name: init-core-agents
description: Use when the user asks to set up their Claude Code agent infrastructure, initialize core agents, or bootstrap the JadeRoad agent architecture on their machine.
version: 2.0.0
---

# Init Core Agents

Set up the JadeRoad agent infrastructure on a user's machine. Deploys organizational rules, discipline-specific standards, memory directories, and configures the user's CLAUDE.md with @imports.

**What this skill does:**
1. Deploys `core-rules.md` — org-wide standards (token efficiency, quality)
2. Deploys discipline file(s) — role-specific rules (e.g., TDD for developers)
3. Creates `.user.md` personalization stubs for each subagent
4. Scaffolds global memory directories for subagent state
5. Creates or updates `~/.claude/CLAUDE.md` with personal identity + @imports
6. Registers the orchestrator's initial state

**What the plugin auto-provides (no skill needed):**
- Subagent definitions (tech-lead, qa-lead, ux-lead, azure-solutions-architect) are auto-available via the plugin's `agents/` directory once installed.

## Versioning

All managed components carry version markers (`<!-- @jaderoad:component@version -->`). This enables:
- **Safe upgrades:** Re-running the skill detects the installed version and upgrades in place
- **Selective updates:** Individual components can be updated without touching others
- **User file protection:** `.user.md` files and Memory/ directories are never versioned or overwritten

| Component | Version Marker | Overwritten on Update |
|-----------|---------------|----------------------|
| `core-rules.md` | `@jaderoad:core-rules@1.0.0` | Yes |
| `disciplines/development.md` | `@jaderoad:discipline-development@1.0.0` | Yes |
| `tech-lead.md` (plugin) | `@jaderoad-agent:tech-lead@1.0.0` | Yes |
| `qa-lead.md` (plugin) | `@jaderoad-agent:qa-lead@1.0.0` | Yes |
| `ux-lead.md` (plugin) | `@jaderoad-agent:ux-lead@1.0.0` | Yes |
| `azure-solutions-architect.md` (plugin) | `@jaderoad-agent:azure-solutions-architect@1.0.0` | Yes |
| `CLAUDE.md` section | `@jaderoad:init-core-agents@2.0.0` | Yes (between markers only) |
| `*.user.md` | None | **Never** |
| `*/Memory/*.md` | None | **Never** |

## Usage
```
/jaderoad:init-core-agents                        # Interactive — prompts for all values
/jaderoad:init-core-agents "Your Name"            # Provide owner name
```

## Arguments

Parse from `$ARGUMENTS`:
- **owner_name** (optional): The user's name. If omitted, ask.

---

## Instructions

### Step 1: Gather Info

Ask the user for these values (skip any provided in arguments or discoverable):

1. **Owner name** — e.g., "Alex Smith"
2. **Orchestrator name** — the name for their lead orchestrator agent. Default: "Operator". The user can choose any name.

**Note:** Discipline selection has moved to `init-project-team`. Disciplines are project-scoped, not user-scoped — the same user may work on code projects (development discipline) and planning projects (project-management discipline). This skill deploys all available discipline files as a library; projects import the ones they need.

### Step 2: Detect Existing Infrastructure

Before making any changes, scan for ALL existing agent infrastructure. Multiple systems may coexist.

#### 2a: Own previous version

Search `~/.claude/CLAUDE.md` for `<!-- @jaderoad:init-core-agents@`. If found, extract the version number. Check for the matching end marker `<!-- @jaderoad:init-core-agents-end -->`.

#### 2b: Legacy pre-marker installation

Check for agent infrastructure without any version markers:
- Old `## Governance Agents` section in CLAUDE.md without `<!-- @` markers
- Agent directories in `~/.claude/agents/Governance/` without corresponding CLAUDE.md markers

#### 2c: Inventory existing files

Scan `~/.claude/agents/` recursively and classify every file:

| Classification | Examples | Action on Migration |
|---------------|---------|-------------------|
| Memory files | `*/Memory/*.md` | **NEVER touched** |
| User knowledge | `*.user.md`, `*/KNOWLEDGE.md` | **NEVER touched** |
| Core definitions | `*/IDENTITY.md`, `core-rules.md`, `disciplines/*.md` | **Replaced** with latest |
| Orchestrator memory | `{Name}/Memory/STATE.md`, `DECISIONS.md`, `SUBAGENTS.md` | **NEVER touched** |

Build a complete detection report. If nothing is detected, note this is a fresh install.

### Step 3: Summarize & Request Approval

**Skip entirely for fresh installs** (no `~/.claude/CLAUDE.md` and no `~/.claude/agents/` directory).

Present a clear summary to the user:

```
## Existing Agent Infrastructure Detected

### Currently Installed:
- {System name} v{version} — {what it provides}
  {N} agent directories, {N} memory files
{repeat for each detected system}

### Migration Plan:

| Action | Files |
|--------|-------|
| **Preserved** (Memory/, KNOWLEDGE.md, .user.md) | {list} |
| **Replaced** (IDENTITY.md, core definitions) | {list} |
| **Added** (new to this architecture) | {list} |
| **Removed** (obsolete, backed up first) | {list} |

A full backup will be created before any changes.
```

Ask: **"Proceed with migration to jaderoad:init-core-agents v2.0.0? (yes/no)"**

If no, stop.

### Step 4: Backup Existing Infrastructure

**Skip only for completely fresh installs** (no `~/.claude/CLAUDE.md` and no `~/.claude/agents/`).

Create a timestamped backup directory:

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
+-- manifest.json
+-- CLAUDE.md                    (copy of ~/.claude/CLAUDE.md)
+-- agents/                      (full recursive copy of ~/.claude/agents/)
```

Generate **manifest.json**:
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Pre-migration backup before jaderoad:init-core-agents v2.0.0",
  "skill": "jaderoad:init-core-agents",
  "skill_version": "2.0.0",
  "scope": "global",
  "detected_infrastructure": {
    "{marker_name}": "{version}"
  },
  "files_backed_up": [
    "~/.claude/CLAUDE.md",
    "~/.claude/agents/"
  ],
  "restore_command": "/jaderoad:restore-agent-infrastructure"
}
```

**Backup rules:**
- Copy `~/.claude/CLAUDE.md` to backup root
- Recursively copy the entire `~/.claude/agents/` directory tree to `backup/agents/`
- Preserve exact directory structure and file contents
- Do NOT skip any files — back up everything including Memory/, KNOWLEDGE.md, .user.md

Tell the user: "Backup created at `~/.claude/backups/{timestamp}/`. Use `/jaderoad:restore-agent-infrastructure` to restore if needed."

### Step 5: Deploy Core Rules

Copy `templates/core-rules.md` to `~/.claude/agents/core-rules.md`.

If the file already exists, overwrite it (this is plugin-managed content — always latest).

### Step 6: Deploy Discipline Library

Copy **all** discipline files from `templates/disciplines/` to `~/.claude/agents/disciplines/`.

Create `~/.claude/agents/disciplines/` if it doesn't exist.

If files already exist, overwrite them (plugin-managed content — always latest).

These are deployed as a library — projects import the ones they need via `init-project-team`. The user's CLAUDE.md does **not** import them directly.

### Step 7: Deploy CLAUDE.md

1. Read the template from `templates/claude-md-template.md`
2. Replace placeholders:
   - `{{ORCHESTRATOR_NAME}}` → orchestrator name from Step 1
   - `{{OWNER_NAME}}` → owner name from Step 1
3. Wrap the generated content in version markers:
   ```
   <!-- @jaderoad:init-core-agents@2.0.0 -->
   {content}
   <!-- @jaderoad:init-core-agents-end -->
   ```
4. Write to `~/.claude/CLAUDE.md` (respecting user's choice from Step 3)

### Step 8: Create Personalization Stubs

For each subagent provided by the plugin, create a `.user.md` personalization file in `~/.claude/agents/` if one doesn't already exist:

| Stub File | For Subagent |
|-----------|-------------|
| `tech-lead.user.md` | tech-lead |
| `qa-lead.user.md` | qa-lead |
| `ux-lead.user.md` | ux-lead |
| `azure-solutions-architect.user.md` | azure-solutions-architect |

Each stub uses this template:
```markdown
# {Agent-Name} — User Personalization

<!-- This file is yours. Plugin updates never touch it. -->
<!-- {Agent-Name} reads this at the start of every consultation. -->
<!-- Add your preferences, learned patterns, and custom rules below. -->

(no personalizations yet)
```

**CRITICAL:** Never overwrite `.user.md` files. If they already exist, skip them entirely.

### Step 9: Scaffold Memory Directories

Create the orchestrator's memory structure. Skip any directories/files that already exist.

```
~/.claude/agents/
├── {OrchestratorName}/
│   └── Memory/
│       ├── STATE.md
│       ├── DECISIONS.md
│       └── SUBAGENTS.md
├── Governance/
│   ├── Compass/Memory/    (PRINCIPLES.md, PATTERNS.md)
│   ├── Crit/Memory/       (DECISIONS.md, BACKLOG.md)
│   ├── Bastion/Memory/    (POSTURE.md, WATCHLIST.md)
│   ├── Gavel/Memory/      (HEALTH.md, GATES.md)
│   └── Assayer/Memory/    (SCORES.md, BACKLOG.md, STANDARD.md)
└── Lumen/Memory/          (PATTERNS.md)
```

**Note:** These directories store memory that the subagents read from. The subagent definitions themselves live in the plugin's `agents/` directory and are auto-available.

#### Memory File Templates

**STATE.md:**
```markdown
# {OrchestratorName} State — Cross-Project

**Last Updated:** {DATE}
**Active Projects:** (none yet)

---

## Active Projects

| Project | Directory | Status | Current Focus |
|---------|-----------|--------|---------------|

---

## Global Notes

- Agent infrastructure v2.0.0 deployed {DATE}
- Four-layer architecture: Core (org rules) → Discipline (role rules) → Personal (CLAUDE.md) → Project (domain agents)
- Global subagents: tech-lead, qa-lead, ux-lead, azure-solutions-architect
- Distributed via JadeRoad plugin
```

**DECISIONS.md:**
```markdown
# Global Methodology Decisions

**Last Updated:** {DATE}

---

## Format

### [ID] Decision Title
- **Date:** YYYY-MM-DD
- **Context:** What prompted this decision
- **Decision:** What was decided
- **Consequences:** What follows
- **Status:** ACTIVE | SUPERSEDED by [ID]
```

**SUBAGENTS.md:**
```markdown
# Subagent Registry

**Last Updated:** {DATE}

---

## Global Subagents (auto-delegated by Claude Code)

| Subagent | Core Question | Model | Tools |
|----------|---------------|-------|-------|
| tech-lead | Should we build this, and does it fit? | sonnet | Read, Grep, Glob |
| qa-lead | Is this safe, stable, and well-tested? | sonnet | Read, Grep, Glob, Bash |
| ux-lead | Is this usable and accessible? | sonnet | Read, Grep, Glob |
| azure-solutions-architect | Azure architecture, cost, and service selection? | sonnet | Read, Grep, Glob |

## Project Subagents (per-project, created by init-project-team)

| Pattern | Model | Scope |
|---------|-------|-------|
| {codename}-tracker | haiku | Sprint state, blockers, backlog |
| {codename}-specialist | sonnet | Business rules, domain, API, integrations, dependency health |
| {codename}-brand-manager | haiku | Project/client brand guidelines |

Project Architect is memory-only — Tech-Lead reads its memory directly.
```

#### Governance Memory File Templates

For each governance memory directory, create empty placeholder files if they don't exist:

**Compass/Memory/PRINCIPLES.md:**
```markdown
# Architecture Principles — Amendments & Exceptions

**Last Updated:** {DATE}

---

(Record amendments or exceptions to the default architecture principles here)
```

**Compass/Memory/PATTERNS.md:**
```markdown
# Universal Patterns

**Last Updated:** {DATE}

---

(Record cross-project patterns: naming conventions, DI patterns, config patterns, error handling)
```

**Crit/Memory/DECISIONS.md:**
```markdown
# Feature Evaluation Verdicts

**Last Updated:** {DATE}

---

(Record GO/NO-GO/MORE-INFO-NEEDED verdicts with reasoning)
```

**Crit/Memory/BACKLOG.md:**
```markdown
# Parked Proposals

**Last Updated:** {DATE}

---

(Proposals that received MORE-INFO-NEEDED, waiting for answers)
```

**Bastion/Memory/POSTURE.md:**
```markdown
# Security Posture

**Last Updated:** {DATE}

---

## Open Items

| ID | Severity | Description | Found | Status |
|----|----------|-------------|-------|--------|

(none — newly initialized)
```

**Bastion/Memory/WATCHLIST.md:**
```markdown
# Security Watch List

**Last Updated:** {DATE}

---

(Items being monitored but not yet actionable)
```

**Gavel/Memory/HEALTH.md:**
```markdown
# Test Health Dashboard

**Last Updated:** {DATE}

---

| Project | Tests | Passing | Coverage | Trend |
|---------|-------|---------|----------|-------|

(populate per-project as test suites are established)
```

**Gavel/Memory/GATES.md:**
```markdown
# Quality Gate Status

**Last Updated:** {DATE}

---

| Project | TDD | QA | Independent QA | Governance | Docs |
|---------|-----|----|----------------|------------|------|

(populate per-project)
```

**Assayer/Memory/SCORES.md:**
```markdown
# Test Quality Scores

**Last Updated:** {DATE}

---

| Project | Score | Anti-Patterns | Last Audit |
|---------|-------|---------------|------------|

(populate per-project as test suites are reviewed)
```

**Assayer/Memory/BACKLOG.md:**
```markdown
# Test Remediation Backlog

**Last Updated:** {DATE}

---

(tests flagged for quality issues, awaiting remediation)
```

**Assayer/Memory/STANDARD.md:**
```markdown
# Good Test Checklist

1. Tests the right thing (behavior, not implementation)
2. Fails for the right reason (clear, specific assertion)
3. One logical assertion per test
4. Named for what it verifies
5. Independent (no test-order dependencies)
6. Fast (no unnecessary I/O or sleep)
7. Deterministic (same result every run)
```

**Lumen/Memory/PATTERNS.md:**
```markdown
# UX Patterns & Recurring Issues

**Last Updated:** {DATE}

---

(Record UX patterns, accessibility findings, and recurring issues across projects)
```

### Step 10: Report

Report what was created:
- Confirm CLAUDE.md was written/updated (show the @import lines)
- Confirm core-rules.md was deployed
- List discipline files deployed to the library (note: projects import these via init-project-team)
- Note which memory directories were created vs already existed
- Confirm plugin subagents are available (tech-lead, qa-lead, ux-lead, azure-solutions-architect)
- If migration was performed:
  - Confirm backup location (`~/.claude/backups/{timestamp}/`)
  - Confirm all Memory/ files were preserved
  - Confirm all user files (.user.md, KNOWLEDGE.md) were preserved
- List `.user.md` personalization stubs created (or note "already existed")
- Remind the user:
  - Subagents auto-delegate based on task context — no manual commands needed
  - Use `/jaderoad:init-project-team` to add project agents to specific projects
  - Personal preferences go in `~/.claude/CLAUDE.md` — plugin updates never touch this file
  - When a subagent learns something, it saves to your `.user.md` file by default — safe from updates
  - Say "save to core" or "train core agent" to write to shared core memory instead
  - Agent memory in `Governance/*/Memory/` persists across sessions and plugin updates
  - All managed files carry version markers — re-run this skill to upgrade safely
  - Use `/jaderoad:restore-agent-infrastructure` to restore from any backup

---

## Architecture Reference

After running this skill, the user's machine will have:

```
~/.claude/
├── CLAUDE.md                           ← Personal (user owns, never overwritten)
│   @import core-rules.md               ← Org standards (overwritten on update)
├── agents/
│   ├── core-rules.md                   ← Plugin-managed (versioned, overwritten on update)
│   ├── disciplines/                    ← Discipline library (projects import what they need)
│   │   └── development.md              ← Plugin-managed (versioned, overwritten on update)
│   ├── tech-lead.user.md               ← User personalization (NEVER overwritten)
│   ├── qa-lead.user.md                 ← User personalization (NEVER overwritten)
│   ├── ux-lead.user.md                 ← User personalization (NEVER overwritten)
│   ├── azure-solutions-architect.user.md ← User personalization (NEVER overwritten)
│   ├── {OrchestratorName}/Memory/      ← Cross-project state (NEVER overwritten)
│   ├── Governance/*/Memory/            ← Subagent memory (NEVER overwritten)
│   └── Lumen/Memory/                   ← UX patterns (NEVER overwritten)
```

Plugin auto-provides (no files deployed to user machine):
- `tech-lead.md` — architectural + scope advisor (sonnet) — reads `tech-lead.user.md`
- `qa-lead.md` — security, stability, test quality gate (sonnet) — reads `qa-lead.user.md`
- `ux-lead.md` — accessibility, usability, responsiveness (sonnet) — reads `ux-lead.user.md`
- `azure-solutions-architect.md` — Azure cloud architecture + cost (sonnet) — reads `azure-solutions-architect.user.md`

## Three-Layer Knowledge Model (per subagent)

| Layer | File | Owner | Updated By |
|-------|------|-------|-----------|
| **Core Definition** | `{agent}.md` (plugin) | Organization | Plugin update (overwritten) |
| **User Knowledge** | `{agent}.user.md` | Individual | User or agent (NEVER overwritten) |
| **Shared Memory** | `Governance/*/Memory/*.md` | Agents | Runtime (NEVER overwritten) |

Default save target: `.user.md`. Core memory only written when user explicitly requests "save to core" or "train core agent".
