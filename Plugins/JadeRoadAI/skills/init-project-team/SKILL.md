---
name: init-project-team
description: Use when the user asks to initialize project agents, set up a project agent team, or bootstrap the Agents/ directory for the current project.
version: 3.0.0
---

# Init Project Team

Initialize project-specific agents for the current working directory. Deploys subagent definitions to `.claude/agents/` and scaffolds the `Agents/` memory directory.

**What gets created:**
- `.claude/agents/{codename}-tracker.md` — project subagent (haiku)
- `.claude/agents/{codename}-specialist.md` — project subagent (sonnet)
- `.claude/agents/{codename}-brand-manager.md` — project subagent (haiku)
- `Agents/Architect/` — memory directory only (Tech-Lead reads directly, no subagent)
- `Agents/Tracker/`, `Agents/Specialist/`, `Agents/Brand-Manager/` — memory + knowledge directories

## Usage
```
/jaderoad:init-project-team                              # Interactive — prompts for all values
/jaderoad:init-project-team ProjectNova                  # Provide project name, prompt for the rest
```

## Arguments

Parse from `$ARGUMENTS`:
- **project_name** (optional): The project name. If omitted, ask.

---

## Instructions

### Step 1: Gather Info

Ask the user for these values (skip any provided in arguments or discoverable from context):

1. **Project name** — e.g., "ProjectNova" (also used as directory convention)
2. **Codename** — short lowercase name, e.g., "nova" (used as subagent prefix)
3. **Framework** — e.g., "C# .NET 8 + React", "Python FastAPI", "Node.js + Next.js"
4. **Project type** — e.g., "SaaS Platform", "Internal Tool", "API Service"
5. **Purpose** — one-line description of what it does
6. **Build command** — e.g., `dotnet build Nova.sln`
7. **Test command** — e.g., `dotnet test Nova.sln`
8. **Run command** — e.g., `dotnet run --project src/Nova.API`
9. **Project modules** — the `src/` subdirectories, e.g., "Nova.Core/, Nova.API/, Nova.Engine/"
10. **Disciplines** — which discipline(s) apply to this project. Scan `~/.claude/agents/disciplines/` for available `.md` files and present them as options. Multiple selections allowed.

    Present available disciplines with descriptions:
    - `development` — Software development (TDD, coverage tiers, quality gates)
    - *(list any other .md files found in the disciplines directory)*
    - `(none)` — No discipline-specific rules

    Try to infer a default from the project type and framework:
    - If framework contains code-related keywords (C#, .NET, Python, Node, React, etc.) → suggest `development`
    - Otherwise → ask without a default

Try to infer values from the current directory:
- Look for `*.sln` files to guess framework and build commands
- Look for `package.json` to detect Node.js projects
- Look for `requirements.txt` or `pyproject.toml` to detect Python
- Look for existing `src/` subdirectories to suggest project modules
- Use the current directory name as default project name

### Step 2: Detect Existing Project Infrastructure

Before making any changes, scan for ALL existing project agent infrastructure.

#### 2a: Own previous version (jaderoad:init-project-team)

Check for signs of a previous run of this skill:
- Subagent definitions in `.claude/agents/` matching `{codename}-tracker.md`, `{codename}-specialist.md`, `{codename}-brand-manager.md`
- `Agents/` directory with Tracker/, Specialist/, Brand-Manager/ structure
- Project CLAUDE.md referencing `/jaderoad:init-project-team`

If found, extract the codename from existing subagent filenames.

#### 2b: Legacy pre-marker project agents

Check for project agents without any plugin markers:
- `Agents/` directory exists but doesn't match either known structure
- CLAUDE.md with agent references but no known skill markers or references

#### 2d: Inventory existing project files

Scan `{project}/Agents/` and `{project}/.claude/agents/` recursively and classify:

| Classification | Examples | Action on Migration |
|---------------|---------|-------------------|
| Memory files | `*/Memory/*.md` | **NEVER touched** |
| User knowledge | `*/KNOWLEDGE.md` | **NEVER touched** |
| Core definitions | `*/IDENTITY.md` | **Replaced** with latest |
| Subagent defs | `.claude/agents/{codename}-*.md` | **Replaced** with latest |
| Obsolete | `Agents/Pulse/IDENTITY.md`, `.claude/agents/*-architect.md` | **Removed** (backed up first) |

Build a complete detection report. If nothing is detected, note this is a fresh install.

### Step 3: Summarize & Request Approval

**Skip entirely for fresh installs** (no existing `Agents/` directory, no project CLAUDE.md, and no `.claude/agents/` subagent files).

Present a clear summary:

```
## Existing Project Infrastructure Detected

### Currently Installed:
- {Description of what was found}
  Agent directories: {list}
  Subagent definitions: {list}

### Migration Plan:

| Action | Files |
|--------|-------|
| **Preserved** (Memory/, KNOWLEDGE.md) | {list} |
| **Replaced** (IDENTITY.md, subagent defs, CLAUDE.md) | {list} |
| **Added** (new agents/directories) | {list} |
| **Removed** (obsolete, backed up first) | {list} |

### Architecture Changes:
{Show only items relevant to what was detected:}
- Pulse agent → merged into Specialist (STATUS.md, KNOWN_ISSUES.md moved)
- Architect subagent → memory-only (Tech-Lead reads directly)
- Brand-Manager → new project-level brand agent added
- Discipline imports → project-scoped discipline selection added

A full backup will be created before any changes.
```

Ask: **"Proceed with migration to jaderoad:init-project-team v3.0.0? (yes/no)"**

If no, stop.

### Step 4: Backup Existing Project Infrastructure

**Skip only for completely fresh installs** (no existing project CLAUDE.md, no `Agents/`, no `.claude/agents/`).

Create a timestamped backup:

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
+-- manifest.json
+-- CLAUDE.md                    (project's CLAUDE.md)
+-- Agents/                      (full recursive copy of project's Agents/)
+-- .claude/
    +-- agents/                  (full recursive copy of project's .claude/agents/)
```

Generate **manifest.json**:
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Pre-migration backup before jaderoad:init-project-team v3.0.0",
  "skill": "jaderoad:init-project-team",
  "skill_version": "3.0.0",
  "scope": "project",
  "source_directory": "{absolute path to project root}",
  "project_name": "{project name}",
  "detected_infrastructure": {
    "{description}": "{version or 'unversioned'}"
  },
  "files_backed_up": [
    "{project}/CLAUDE.md",
    "{project}/Agents/",
    "{project}/.claude/agents/"
  ],
  "restore_command": "/jaderoad:restore-agent-infrastructure"
}
```

**Backup rules:**
- Copy project `CLAUDE.md` to backup root
- Recursively copy `{project}/Agents/` to `backup/Agents/`
- Recursively copy `{project}/.claude/agents/` to `backup/.claude/agents/`
- Preserve exact directory structure and file contents
- Do NOT skip any files — back up everything including Memory/, KNOWLEDGE.md
- Only include files that exist — skip missing directories silently

Tell the user: "Backup created at `~/.claude/backups/{timestamp}/`. Use `/jaderoad:restore-agent-infrastructure` to restore if needed."

### Step 5: Migrate Legacy Structures

**Skip if this is a fresh install or own previous version with no structural changes needed.**

Perform migrations AFTER the backup has been created. These actions handle structural differences from older architectures.

#### 5a: Pulse → Specialist merge

If `Agents/Pulse/` was detected:
1. Create `Agents/Specialist/Memory/` if it doesn't exist
2. Move `Agents/Pulse/Memory/STATUS.md` → `Agents/Specialist/Memory/STATUS.md` (skip if target exists)
3. Move `Agents/Pulse/Memory/KNOWN_ISSUES.md` → `Agents/Specialist/Memory/KNOWN_ISSUES.md` (skip if target exists)
4. If `Agents/Pulse/KNOWLEDGE.md` exists, append its contents to `Agents/Specialist/KNOWLEDGE.md`
5. Remove `Agents/Pulse/` directory after migration

#### 5b: Architect subagent → memory-only

If an Architect subagent definition was detected (`.claude/agents/*-architect.md`):
1. Remove the `.claude/agents/*-architect.md` subagent file
2. Keep `Agents/Architect/Memory/` and `Agents/Architect/KNOWLEDGE.md` intact

#### 5c: Add KNOWLEDGE.md stubs

If `Agents/` exists but any agent directories lack `KNOWLEDGE.md`:
- Create KNOWLEDGE.md stubs for each agent directory missing one (use the template from Step 8 — Scaffold Agent Memory Directories).

#### 5d: Check for existing CLAUDE.md

Read `CLAUDE.md` in the current project root. If it exists and is non-empty:
- Show the user the first 15 lines
- Ask: "This project already has a CLAUDE.md. Overwrite it, append the project agents to it, or abort?"
- If abort, stop.
- If append, add a section separator (`---`) and append the template content after the existing content.
- If overwrite, proceed with replacement.

### Step 6: Deploy Project CLAUDE.md

1. Read the bundled template from this skill's directory: `project-agents-template.md`
2. Replace all placeholders:
   - `{{PROJECT_NAME}}` → project name from Step 1
   - `{{CODENAME}}` → codename from Step 1
   - `{{FRAMEWORK}}` → framework from Step 1
   - `{{PROJECT_TYPE}}` → project type from Step 1
   - `{{PURPOSE}}` → purpose from Step 1
   - `{{BUILD_COMMAND}}` → build command from Step 1
   - `{{TEST_COMMAND}}` → test command from Step 1
   - `{{RUN_COMMAND}}` → run command from Step 1
   - `{{PROJECT_MODULES}}` → project modules from Step 1
   - `{{PROJECT_CONSTRAINTS}}` → leave as a comment placeholder for the user to fill in later
   - `{{OWNER_NAME}}` → read from `~/.claude/CLAUDE.md` (look for the `<owner>` tag). If not found, ask.
   - `{{DISCIPLINE_IMPORTS}}` → one `@~/.claude/agents/disciplines/{name}.md` line per selected discipline from Step 1. If no disciplines selected, replace with a comment: `<!-- No disciplines selected. Add @imports to load discipline rules. -->`
3. Write the result to `CLAUDE.md` in the project root (respecting the user's choice from Step 5d)

### Step 7: Deploy Project Subagents

Create `.claude/agents/` in the project root if it doesn't exist.

For each subagent template in `templates/subagents/`:

| Template | Deploys To |
|----------|-----------|
| `templates/subagents/tracker.md` | `.claude/agents/{codename}-tracker.md` |
| `templates/subagents/specialist.md` | `.claude/agents/{codename}-specialist.md` |
| `templates/subagents/brand-manager.md` | `.claude/agents/{codename}-brand-manager.md` |

Replace `{PROJECT_NAME}`, `{CODENAME}`, and `{DATE}` placeholders in each template.

### Step 8: Scaffold Agent Memory Directories

Create the agent memory structure under `{project}/Agents/`. Skip any directories that already exist.

```
{project}/Agents/
├── Architect/              # Memory only — no subagent
│   ├── KNOWLEDGE.md
│   └── Memory/
│       ├── ARCHITECTURE.md
│       ├── PATTERNS.md
│       └── DECISIONS.md
├── Tracker/
│   ├── IDENTITY.md         # Core (shipped — overwritten on upgrade)
│   ├── KNOWLEDGE.md         # Local (user's additions — never touched)
│   └── Memory/
│       ├── STATE.md
│       ├── BLOCKERS.md
│       └── BACKLOG.md
├── Specialist/
│   ├── IDENTITY.md         # Core (shipped)
│   ├── KNOWLEDGE.md         # Local (user's additions)
│   └── Memory/
│       ├── DOMAIN.md
│       ├── API.md
│       ├── INTEGRATIONS.md
│       ├── STATUS.md
│       └── KNOWN_ISSUES.md
└── Brand-Manager/
    ├── IDENTITY.md         # Core (shipped)
    ├── KNOWLEDGE.md         # Local (user's additions)
    └── Memory/
        ├── BRAND.md
        └── AUDIT.md
```

For IDENTITY.md files, use the corresponding template from `templates/`:

| Target Path | Template File |
|-------------|---------------|
| `Agents/Tracker/IDENTITY.md` | `templates/Tracker-IDENTITY.md` |
| `Agents/Specialist/IDENTITY.md` | `templates/Specialist-IDENTITY.md` |
| `Agents/Brand-Manager/IDENTITY.md` | `templates/Brand-Manager-IDENTITY.md` |

**Note:** `Agents/Architect/` has no IDENTITY.md — it's memory-only. Tech-Lead reads its memory directly.

Replace `{PROJECT_NAME}`, `{CODENAME}`, `{FRAMEWORK}`, and `{DATE}` placeholders in each template.

For each agent, also create a `KNOWLEDGE.md` stub if one doesn't already exist:
```markdown
# {AGENT_NAME} — Local Knowledge ({PROJECT_NAME})

<!-- This file is yours. Add project-specific context, preferences, and custom rules here. -->
<!-- It is never overwritten by plugin upgrades. -->

(no local knowledge yet)
```

**On upgrade:**
- **IDENTITY.md** — Overwrite with latest template (this is the shipped core)
- **KNOWLEDGE.md** — NEVER touch (this is the user's additions)
- **Memory/*.md** — NEVER touch (this is session data)
- **Subagent .md files** (`.claude/agents/`) — Overwrite with latest template

For Memory files, use the content templates below. Replace `{PROJECT_NAME}`, `{CODENAME}`, `{FRAMEWORK}`, and `{DATE}` with values from Step 1 and today's date. Skip any Memory files that already exist.

### Step 9: Register Project in Global State

If `~/.claude/agents/` exists (global team is set up), update the orchestrator's STATE.md:
- Read `~/.claude/agents/Operator/Memory/STATE.md`
- Add a row to the Active Projects table with the project name, directory, "Active", and current focus
- Update the "Last Updated" date

If the global team isn't set up, skip this step and suggest running `/jaderoad:init-core-agents` first.

### Step 10: Report

Report what was created:
- Confirm CLAUDE.md was written/updated
- List the disciplines imported (show the @import lines added)
- List the subagent files created in `.claude/agents/`
- List the agent memory directories created (or note "already existed" for each)
- If migration was performed:
  - Confirm backup location (`~/.claude/backups/{timestamp}/`)
  - Summarize what was migrated (Pulse merged, Architect converted, etc.)
  - Confirm all Memory/ files were preserved
  - Confirm all KNOWLEDGE.md files were preserved
- Count total files created vs skipped
- Confirm global STATE.md was updated (or note it was skipped)
- Remind the user:
  - Disciplines are project-scoped — different projects can have different disciplines
  - To add or change disciplines later, edit the @import lines at the top of the project's CLAUDE.md
  - Available disciplines are in `~/.claude/agents/disciplines/`
  - **Subagent .md** — Claude Code definition (frontmatter, auto-delegation, enforced model/tools)
  - **IDENTITY.md** — Agent knowledge core (shipped by plugin, overwritten on upgrade)
  - **KNOWLEDGE.md** — Your local teaching (never touched by upgrades)
  - **Memory/*.md** — Session data (never touched by upgrades)
  - Use `/jaderoad:restore-agent-infrastructure` to restore from any backup

---

## Agent Memory File Templates

#### Architect Memory Files (no IDENTITY.md — memory only)

**ARCHITECTURE.md:**
```markdown
# {PROJECT_NAME} Architecture

**Last Updated:** {DATE}
**Framework:** {FRAMEWORK}

---

## Module Map

(Document the project's module structure, dependencies, and data flow here after initial codebase exploration)

## Key Abstractions

(Document core interfaces, base classes, and patterns that define the architecture)

## Dependency Graph

(Document which modules depend on which — direction matters)
```

**PATTERNS.md:**
```markdown
# {PROJECT_NAME} Patterns & Conventions

**Last Updated:** {DATE}

---

(Document project-specific patterns here. Tech-Lead reads these when evaluating architectural fit.)
```

**DECISIONS.md:**
```markdown
# {PROJECT_NAME} Architecture Decisions

**Last Updated:** {DATE}

---

## Format

### [ID] Decision Title
- **Date:** YYYY-MM-DD
- **Context:** What prompted this decision
- **Decision:** What was decided
- **Consequences:** What follows from this decision
- **Status:** ACTIVE | SUPERSEDED by [ID]
```

#### Tracker Memory Files

**STATE.md:**
```markdown
# {PROJECT_NAME} State

**Last Updated:** {DATE}
**Current Milestone:** Sprint 0 — Project Setup
**Test Baseline:** 0 tests

---

## What's Complete

- Project initialized via /jaderoad:init-project-team

## What's Active

(nothing yet)

## What's Next

- Initial codebase exploration
- Populate ARCHITECTURE.md and PATTERNS.md
- First feature planning
```

**BLOCKERS.md:**
```markdown
# {PROJECT_NAME} Blockers

**Last Updated:** {DATE}

---

(none — newly initialized)

## Format

### [B-NNN] Blocker Title
- **Severity:** CRITICAL | HIGH | MEDIUM | LOW
- **Status:** OPEN | IN-PROGRESS | RESOLVED
- **Blocks:** what work is blocked
- **Owner:** who can unblock
- **Description:** details
```

**BACKLOG.md:**
```markdown
# {PROJECT_NAME} Feature Backlog

**Last Updated:** {DATE}

---

## Priority Queue

(add features as they're identified)

## Parking Lot

(ideas not yet evaluated — run through tech-lead before promoting)
```

#### Specialist Memory Files

**DOMAIN.md:**
```markdown
# {PROJECT_NAME} Domain Knowledge

**Last Updated:** {DATE}

---

## Business Rules

(document domain rules, invariants, and constraints as they're discovered)

## Data Model

(document key entities, relationships, and their meanings)
```

**API.md:**
```markdown
# {PROJECT_NAME} API Surface

**Last Updated:** {DATE}

---

## Endpoints

(document API endpoints, contracts, and versioning as they're built)

## Breaking Change Log

(track any breaking changes with date and migration notes)
```

**INTEGRATIONS.md:**
```markdown
# {PROJECT_NAME} External Integrations

**Last Updated:** {DATE}

---

(document external services, auth methods, rate limits, and known quirks)
```

**STATUS.md:**
```markdown
# {PROJECT_NAME} Dependency Health

**Last Updated:** {DATE}

---

## Dependencies

| Dependency | Type | Status | Notes |
|-----------|------|--------|-------|
| (add as discovered) | | | |
```

**KNOWN_ISSUES.md:**
```markdown
# {PROJECT_NAME} Known Issues

**Last Updated:** {DATE}

---

(none — newly initialized)
```

#### Brand-Manager Memory Files

**BRAND.md:**
```markdown
# {PROJECT_NAME} Brand Guidelines

**Last Updated:** {DATE}

---

## Colors

(document project/client brand colors as they're learned)

## Typography

(document fonts, weights, sizes)

## Voice & Tone

(document writing style, terminology preferences)

## Component Patterns

(document recurring UI patterns and their brand treatment)
```

**AUDIT.md:**
```markdown
# {PROJECT_NAME} Brand Audit Log

**Last Updated:** {DATE}

---

(audit entries will be added as brand reviews are performed)
```
