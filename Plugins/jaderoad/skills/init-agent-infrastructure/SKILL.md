---
name: init-agent-infrastructure
description: Use when the user asks to set up their Claude Code agent infrastructure, initialize core agents, or bootstrap the {ORG_NAME} agent architecture on their machine.
version: 1.0.0
---

# Init Agent Infrastructure

Set up the foundational {ORG_NAME} agent infrastructure on a user's machine. Deploys organizational rules, discipline library, and configures the user's CLAUDE.md with a {ORG_NAME}-namespaced managed section.

**Coexistence with Provaxus plugin:** {ORG_NAME} infrastructure is fully namespaced under `~/.claude/agents/{ORG_ID}/` so it can live side-by-side with the Provaxus plugin without file conflicts. The two plugins share the universal agents (tech-lead, qa-lead, ux-lead) and their memory directories, but each has its own org-specific rules and disciplines.

**What this skill does:**
1. Deploys `{ORG_ID}/core-rules.md` — {ORG_NAME} org-wide standards (isolated from Provaxus core-rules.md)
2. Deploys the {ORG_NAME} discipline file library to `~/.claude/agents/{ORG_ID}/disciplines/`
3. Adds a {ORG_NAME}-namespaced managed section to `~/.claude/CLAUDE.md` (does NOT overwrite any Provaxus section)
4. Scaffolds the orchestrator's memory directories (shared if multiple org plugins are installed; creates only what's missing)

**What comes next:**
- Run `/{ORG_ID}:init-core-agents` to install global subagent personalization and memory
- Run `/{ORG_ID}:start-session` in each project directory to set up project agents

**What the plugin auto-provides (no skill needed):**
- Subagent definitions (tech-lead, qa-lead, ux-lead, {MARKETING_AGENT}) are auto-available via the plugin's `agents/` directory once installed.

## File Locations — {ORG_NAME} Namespace

All {ORG_NAME} infrastructure is isolated from Provaxus:

| File | {ORG_NAME} Path | Provaxus Path (do not touch) |
|------|--------------|------------------------------|
| Org rules | `~/.claude/agents/{ORG_ID}/core-rules.md` | `~/.claude/agents/core-rules.md` |
| Disciplines | `~/.claude/agents/{ORG_ID}/disciplines/*.md` | `~/.claude/agents/disciplines/*.md` |
| CLAUDE.md section | Between `@{ORG_ID}:init-agent-infrastructure@` markers | Between `@provaxus:init-agent-infrastructure@` markers |
| Orchestrator memory | `~/.claude/agents/{Name}/Memory/` | Same — shared if both installed |
| tech-lead/qa-lead/ux-lead Memory | `~/.claude/agents/{agent}/Memory/` | Same — shared if both installed |

## Versioning

| Component | Version Marker | Overwritten on Update |
|-----------|---------------|----------------------|
| `{ORG_ID}/core-rules.md` | `@{ORG_ID}:core-rules@1.0.0` | Yes |
| `{ORG_ID}/disciplines/development.md` | `@{ORG_ID}:discipline-development@1.0.0` | Yes |
| `{ORG_ID}/disciplines/marketing.md` | `@{ORG_ID}:discipline-marketing@1.0.0` | Yes |
| `CLAUDE.md` {ORG_NAME} section | `@{ORG_ID}:init-agent-infrastructure@1.0.0` | Yes (between markers only) |
| `*/Memory/*.md` | None | **Never** |
| `*.user.md` | None | **Never** |

## Usage
```
/{ORG_ID}:init-agent-infrastructure                        # Interactive — prompts for all values
/{ORG_ID}:init-agent-infrastructure "Eric White"            # Provide owner name
```

## Arguments

Parse from `$ARGUMENTS`:
- **owner_name** (optional): The user's name. If omitted, ask.

---

## Instructions

### Bootstrap: Load Org Configuration

**Run this before any other step.** Read the plugin configuration to bind org variables for this skill:

```bash
cat "{SKILL_BASE_DIR}/../../.claude-plugin/plugin.json"
```

Extract and bind:
- `{ORG_ID}` ← `name` field (e.g., `jaderoad`, `provaxus`, `acme`)
- `{ORG_NAME}` ← `org_name` field (e.g., `JadeRoad`, `Provaxus`, `Acme Corp`)
- `{GITHUB_ORG}` ← `github_org` field (e.g., `JadeRoad-AI`, `Provaxus-AI`, `Acme-Corp`)
- `{MARKETING_AGENT}` ← `marketing_agent` field if present; otherwise `{ORG_ID}-marketing-sme`

All `{ORG_ID}`, `{ORG_NAME}`, `{GITHUB_ORG}`, and `{MARKETING_AGENT}` tokens throughout this skill resolve to these values. The skill body contains no hard-coded organization names.

---

### Step 1: Gather Info

Ask the user for these values (skip any provided in arguments or discoverable):

1. **Owner name** — e.g., "Eric White"
2. **Orchestrator name** — the name for their lead orchestrator agent. Default: "Operator". Explain: this is the identity name for the top-level Claude agent. If another org plugin has already defined an orchestrator in CLAUDE.md (any `# {Name} — Lead Orchestrator` heading inside another plugin's managed section), suggest using the same name.

### Step 2: Detect Existing Infrastructure

Before making any changes, scan for ALL existing agent infrastructure.

#### 2a: Own previous version ({ORG_NAME})

Search `~/.claude/CLAUDE.md` for `<!-- @{ORG_ID}:init-agent-infrastructure@`. If found, extract the version number and check for the matching end marker `<!-- @{ORG_ID}:init-agent-infrastructure-end -->`.

Also check for `~/.claude/agents/{ORG_ID}/core-rules.md`.

#### 2b: Other-plugin coexistence detection

Search `~/.claude/CLAUDE.md` for any managed section from another org plugin: look for `<!-- @<something_other_than_{ORG_ID}>:init-agent-infrastructure@` patterns. If found, note the orchestrator name in that section. This tells us:
- An orchestrator identity is already defined — {ORG_NAME} will NOT redefine it
- Shared memory directories may already exist — skip creation rather than overwrite

#### 2c: Inventory existing files

Scan `~/.claude/agents/{ORG_ID}/` (if it exists) and `~/.claude/agents/` recursively. Classify files:

| Classification | Path | Action |
|---------------|------|--------|
| {ORG_NAME} org rules | `~/.claude/agents/{ORG_ID}/core-rules.md` | **Replaced** with latest |
| {ORG_NAME} disciplines | `~/.claude/agents/{ORG_ID}/disciplines/*.md` | **Replaced** with latest |
| Shared memory | `~/.claude/agents/{Name}/Memory/*.md` | **NEVER touched** |
| Shared agent memory | `~/.claude/agents/tech-lead/Memory/*.md` etc. | **NEVER touched** |
| User personalization | `*.user.md` | **NEVER touched** |
| Other-org rules | `~/.claude/agents/<other_org_id>/core-rules.md` | **NEVER touched — not ours** |
| Other-org disciplines | `~/.claude/agents/<other_org_id>/disciplines/*.md` | **NEVER touched — not ours** |

Build a detection report. If nothing is detected, note this is a fresh install.

### Step 3: Summarize & Request Approval

**Skip for fresh installs** (no `~/.claude/agents/{ORG_ID}/` and no {ORG_NAME} markers in `~/.claude/CLAUDE.md`).

Present a clear summary:

```
## Existing {ORG_NAME} Infrastructure Detected

### Currently Installed:
- {description of {ORG_NAME} files found}

### Other-Plugin Coexistence:
- {Other-plugin section found — orchestrator name: {Name} | Not found}

### Migration Plan:

| Action | Files |
|--------|-------|
| **Preserved** (Memory/, .user.md) | {list} |
| **Replaced** ({ORG_NAME} org rules, disciplines) | {list} |
| **Added** | {list} |
| **Not Touched** (other org plugin files) | {list} |

A full backup will be created before any changes.
```

Ask: **"Proceed with {ORG_ID}:init-agent-infrastructure v1.0.0? (yes/no)"**

If no, stop.

### Step 4: Backup Existing Infrastructure

**Skip only for completely fresh installs** (no `~/.claude/agents/{ORG_ID}/` and no {ORG_NAME} CLAUDE.md markers).

Back up only {ORG_NAME}-managed files — do NOT back up Provaxus files (they have their own backup mechanism):

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
+-- manifest.json
+-- CLAUDE.md                    (full copy — preserves both sections)
+-- agents/
    +-- {ORG_ID}/                (full recursive copy of ~/.claude/agents/{ORG_ID}/)
```

Generate **manifest.json**:
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Pre-migration backup before {ORG_ID}:init-agent-infrastructure v1.0.0",
  "skill": "{ORG_ID}:init-agent-infrastructure",
  "skill_version": "1.0.0",
  "scope": "global",
  "coexisting_plugins": ["any other installed org plugins"],
  "files_backed_up": [
    "~/.claude/CLAUDE.md",
    "~/.claude/agents/{ORG_ID}/"
  ],
  "restore_command": "/{ORG_ID}:restore-agent-infrastructure"
}
```

Tell the user: "Backup created at `~/.claude/backups/{timestamp}/`. Use `/{ORG_ID}:restore-agent-infrastructure` to restore if needed."

### Step 2e: Detect Legacy Unnamespaced Infrastructure

Check for files deployed by an **older version of this plugin** before the `~/.claude/agents/{ORG_ID}/` namespace design was adopted. These are unambiguously identifiable because they contain this plugin's own version markers.

**Core rules legacy file:**
```bash
if [ -f ~/.claude/agents/core-rules.md ]; then
  grep -c "@{ORG_ID}:core-rules@" ~/.claude/agents/core-rules.md
fi
```
If the file exists AND its version marker is `@{ORG_ID}:core-rules@` (not another org's marker): this is a legacy file from this plugin. Mark for migration.

**Legacy discipline files:**
```bash
for f in ~/.claude/agents/disciplines/*.md; do
  grep -l "@{ORG_ID}:discipline-" "$f" 2>/dev/null
done
```
Any `.md` in the unnamespaced `~/.claude/agents/disciplines/` directory with a `@{ORG_ID}:discipline-` marker is a legacy file from this plugin. Mark for migration.

**Legacy CLAUDE.md @import:**
Check the managed section content (between `<!-- @{ORG_ID}:init-agent-infrastructure@` markers) for the old-style import line:
```
@~/.claude/agents/core-rules.md
```
If present (without the `{ORG_ID}/` namespace prefix), mark this import for update.

Record all legacy files found. Steps 5, 6, and 7 handle their migration.

---

### Step 5: Deploy Core Rules

Create `~/.claude/agents/{ORG_ID}/` if it doesn't exist.

Copy `templates/core-rules.md` to `~/.claude/agents/{ORG_ID}/core-rules.md`. Replace `{ORG_ID}` and `{ORG_NAME}` placeholders in the template before writing. If the file already exists, overwrite it (org-managed content — always latest).

**Legacy migration (if Step 2e found a legacy file):**
If `~/.claude/agents/core-rules.md` was identified as belonging to this plugin in Step 2e:
1. The new file has already been deployed to `~/.claude/agents/{ORG_ID}/core-rules.md` above
2. Remove `~/.claude/agents/core-rules.md` — it is now superseded by the namespaced path
3. Log: "Migrated `~/.claude/agents/core-rules.md` → `~/.claude/agents/{ORG_ID}/core-rules.md` (namespace upgrade)"

**Do NOT remove** `~/.claude/agents/core-rules.md` if its version marker does NOT match `@{ORG_ID}:core-rules@` — that file belongs to another org plugin and must be left alone.

### Step 6: Deploy Discipline Library

Copy **all** discipline files from `templates/disciplines/` to `~/.claude/agents/{ORG_ID}/disciplines/`.

Create `~/.claude/agents/{ORG_ID}/disciplines/` if it doesn't exist. Replace `{ORG_ID}`, `{ORG_NAME}`, and `{MARKETING_AGENT}` placeholders in each template before writing. If files already exist, overwrite them (org-managed content — always latest).

**Legacy migration (if Step 2e found legacy discipline files):**
For each discipline file found in `~/.claude/agents/disciplines/` with this plugin's version marker:
1. The new file has already been deployed to `~/.claude/agents/{ORG_ID}/disciplines/{name}.md` above
2. Remove the old `~/.claude/agents/disciplines/{name}.md`
3. If `~/.claude/agents/disciplines/` is now empty, remove the directory
4. Log: "Migrated `~/.claude/agents/disciplines/{name}.md` → `~/.claude/agents/{ORG_ID}/disciplines/{name}.md` (namespace upgrade)"

**Do NOT remove** any discipline file from `~/.claude/agents/disciplines/` if its version marker does NOT match `@{ORG_ID}:discipline-` — those belong to another org plugin.

### Step 7: Deploy CLAUDE.md

The {ORG_NAME} managed section must coexist with any Provaxus managed section. The handling differs depending on what's already in the file.

#### 7a: Determine orchestrator name

1. Check `~/.claude/CLAUDE.md` for any other plugin's managed section — any `<!-- @<other_id>:init-agent-infrastructure@` marker where `<other_id>` is not `{ORG_ID}`. If found, extract the orchestrator name from the first `# {Name} — Lead Orchestrator` heading in that block.
2. If no other-plugin section found, use the name provided in Step 1.
3. If `~/.claude/CLAUDE.md` has a {ORG_NAME} section already, extract the name from it.

#### 7b: Build the {ORG_NAME} managed block

**Case A — Another org plugin's section already present in CLAUDE.md:**

The orchestrator is already defined. {ORG_NAME} adds only the namespaced org-rules import — no orchestrator redefinition. Build this minimal block:

```markdown
<!-- @{ORG_ID}:init-agent-infrastructure@1.0.0 -->
## {ORG_NAME} Org Rules

@~/.claude/agents/{ORG_ID}/core-rules.md
<!-- @{ORG_ID}:init-agent-infrastructure-end -->
```

**Case B — No Provaxus section ({ORG_NAME} standalone):**

Build the full orchestrator section using the `templates/claude-md-template.md` template, substituting `{{ORCHESTRATOR_NAME}}` and `{{OWNER_NAME}}`, then wrap in {ORG_NAME} markers:

```
<!-- @{ORG_ID}:init-agent-infrastructure@1.0.0 -->
{full template content with substituted values}
<!-- @{ORG_ID}:init-agent-infrastructure-end -->
```

#### 7c: Write to CLAUDE.md

- If `~/.claude/CLAUDE.md` has a {ORG_NAME} managed section already: replace only the content between `<!-- @{ORG_ID}:init-agent-infrastructure@` and `<!-- @{ORG_ID}:init-agent-infrastructure-end -->` (preserving all other content).
- If `~/.claude/CLAUDE.md` has no {ORG_NAME} section but has another plugin's managed section: append the {ORG_NAME} block after that section's end marker, separated by a blank line.
- If `~/.claude/CLAUDE.md` has neither section: write the {ORG_NAME} block as the entire file (Case B).
- If `~/.claude/CLAUDE.md` does not exist: write the {ORG_NAME} block as the entire file (Case B).

**Legacy @import migration (Step 2e):**
After writing the managed section, scan the new content for the legacy unnamespaced import line:
```
@~/.claude/agents/core-rules.md
```
If found, replace it with:
```
@~/.claude/agents/{ORG_ID}/core-rules.md
```
This updates the orchestrator's import to point to the new namespaced path. Only replace if the import is inside this plugin's managed section markers — never modify imports that belong to another plugin's section.

**CRITICAL:** Never modify or overwrite any other plugin's managed section. Any `<!-- @<other_org_id>:init-agent-infrastructure@` markers and the content between them are untouchable.

### Step 8: Scaffold Orchestrator Memory

Create the orchestrator's memory structure. Skip any directories/files that already exist — another installed org plugin may have already created them.

```
~/.claude/agents/
└── {OrchestratorName}/
    └── Memory/
        ├── STATE.md
        ├── DECISIONS.md
        └── SUBAGENTS.md
```

**Only create files that don't already exist.** If another org plugin has already scaffolded `STATE.md`, do not touch it.

If creating STATE.md fresh:
```markdown
# {OrchestratorName} State — Cross-Project

**Last Updated:** {DATE}
**Active Projects:** (none yet)

---

## Active Projects

| Project | Directory | Status | Current Focus | Org |
|---------|-----------|--------|---------------|-----|

---

## Global Notes

- {ORG_NAME} agent infrastructure v1.0.0 deployed {DATE}
- Foundation layer active: {ORG_ID}/core-rules.md, {ORG_ID} disciplines library
- Run /{ORG_ID}:init-core-agents to complete subagent setup
```

If creating SUBAGENTS.md fresh:
```markdown
# Subagent Registry

**Last Updated:** {DATE}

---

## Global Subagents (auto-delegated by Claude Code)

| Subagent | Core Question | Model | Tools | Org |
|----------|---------------|-------|-------|-----|
| tech-lead | Should we build this, and does it fit? | sonnet | Read, Grep, Glob | Shared |
| qa-lead | Is this safe, stable, and well-tested? | sonnet | Read, Grep, Glob, Bash | Shared |
| ux-lead | Is this usable and accessible? | sonnet | Read, Grep, Glob | Shared |
| {MARKETING_AGENT} | What does the marketing data say? | sonnet | Read, Grep, Glob, WebFetch, WebSearch | {ORG_NAME} |

*Run /{ORG_ID}:init-core-agents to deploy subagent memory and personalization files.*
```

If creating DECISIONS.md fresh:
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

### Step 9: Report

Report what was created:
- Confirm {ORG_NAME} section was added/updated in CLAUDE.md (show the section markers)
- Note if any other org plugin section was detected and preserved
- Confirm `{ORG_ID}/core-rules.md` was deployed (show path)
- List discipline files deployed to `{ORG_ID}/disciplines/` library
- Confirm orchestrator memory directories were created (or note "already existed — shared with another installed plugin")
- If legacy files were migrated (Step 2e): list migrated paths and confirm removed legacy files
- If backup was taken: confirm backup location
- Remind the user:
  - **Next step:** Run `/{ORG_ID}:init-core-agents` to deploy global subagent personalization and memory
  - Then use `/{ORG_ID}:start-session` in each {ORG_NAME} project directory
  - {ORG_NAME} infrastructure is isolated at `~/.claude/agents/{ORG_ID}/` — Provaxus files are untouched
  - Use `/{ORG_ID}:restore-agent-infrastructure` to restore from any backup

---

## Architecture Reference

After running this skill alongside other org plugins, the user's machine will have:

```
~/.claude/
├── CLAUDE.md
│   ├── <!-- @<other_org_id>:init-agent-infrastructure@ --> ... (untouched)
│   └── <!-- @{ORG_ID}:init-agent-infrastructure@ -->
│       @~/.claude/agents/{ORG_ID}/core-rules.md  ← {ORG_NAME} rules only
│       (or full orchestrator definition if standalone)
│
└── agents/
    ├── core-rules.md (unnamespaced)     ← Another org plugin (DO NOT TOUCH)
    ├── disciplines/ (unnamespaced)      ← Another org plugin (DO NOT TOUCH)
    │   └── development.md
    ├── {ORG_ID}/                       ← {ORG_NAME} ONLY
    │   ├── core-rules.md               ← {ORG_NAME} org rules
    │   └── disciplines/
    │       ├── development.md          ← {ORG_NAME} dev discipline
    │       └── marketing.md            ← {ORG_NAME} marketing discipline
    ├── {OrchestratorName}/Memory/      ← Shared (created by whichever ran first)
    │   ├── STATE.md
    │   ├── DECISIONS.md
    │   └── SUBAGENTS.md
    ├── tech-lead/Memory/               ← Shared (universal agent)
    ├── qa-lead/Memory/                 ← Shared (universal agent)
    └── ux-lead/Memory/                 ← Shared (universal agent)
```

Plugin auto-provides (no files deployed to user machine):
- `tech-lead.md`, `qa-lead.md`, `ux-lead.md` — universal agents (shared between orgs)
- `{MARKETING_AGENT}.md` — {ORG_NAME}-specific marketing SME

Run `/{ORG_ID}:init-core-agents` to deploy personalization stubs and memory namespaces.
