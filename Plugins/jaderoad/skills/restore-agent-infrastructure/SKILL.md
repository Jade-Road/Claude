---
name: restore-agent-infrastructure
description: Use when the user wants to restore a previous agent infrastructure from a backup. Lists available backups, compares to current state, and restores with a safety backup.
version: 1.0.0
---

# Restore Agent Infrastructure

Review and restore from agent infrastructure backups created by `init-agent-infrastructure`, `init-core-agents`, `start-session`, or `uninstall-agent-infrastructure`. Every restore creates a safety backup of the current state first, ensuring no work is ever lost.

## Usage
```
/{ORG_ID}:restore-agent-infrastructure                    # List all backups, prompt for selection
/{ORG_ID}:restore-agent-infrastructure 2026-04-30         # Filter backups by date
/{ORG_ID}:restore-agent-infrastructure latest              # Show most recent backup
```

## Arguments

Parse from `$ARGUMENTS`:
- **filter** (optional): A date string (YYYY-MM-DD) to filter backups, or `latest` to select the most recent. If omitted, list all.

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

### Step 1: List Available Backups

Scan `~/.claude/backups/` for timestamped directories. For each, read `manifest.json` to get metadata.

If `~/.claude/backups/` does not exist or contains no valid backup directories:
- Tell the user: "No agent infrastructure backups found. Backups are created automatically when running `/{ORG_ID}:init-agent-infrastructure`, `/{ORG_ID}:init-core-agents`, or `/{ORG_ID}:start-session`."
- Stop.

A valid backup directory contains a `manifest.json` with at least `timestamp`, `reason`, `scope`, and `files_backed_up` fields.

If a filter was provided:
- **Date filter (YYYY-MM-DD):** Show only backups whose directory name starts with the given date.
- **`latest`:** Select the most recent backup by timestamp. Skip to Step 3.

Display a table:

```
## Available Backups

| # | Timestamp           | Scope              | Reason                                          |
|---|---------------------|--------------------|-------------------------------------------------|
| 1 | 2026-04-30 14:30:00 | Global             | Pre-migration before init-core-agents v1.0.0    |
| 2 | 2026-04-28 09:15:00 | Project (Nova)     | Pre-migration before start-session v1.0.0       |
| 3 | 2026-04-28 09:14:00 | Global             | Safety backup before restore from 2026-04-27... |
```

### Step 2: Select Backup

If only one backup matches the filter (or `latest` was specified), use it automatically.

Otherwise, ask: **"Which backup to review? (enter number)"**

### Step 3: Compare Backup to Current State

Read the selected backup's `manifest.json` to determine scope and source paths.

#### For global backups (`scope: "global"`):

Compare backup `CLAUDE.md` (if present) against current `~/.claude/CLAUDE.md`, and compare `agents/{ORG_ID}/` recursively.

Look for {ORG_NAME} managed markers `<!-- @{ORG_ID}:init-agent-infrastructure@ -->` and `<!-- @{ORG_ID}:init-agent-infrastructure-end -->`. If the {ORG_NAME} managed section differs between backup and current, flag it. Also flag if any Provaxus managed section exists in current but not in backup (or vice versa) — these are different orgs' infrastructure and should not be conflated.

**When restoring CLAUDE.md:** Only restore the {ORG_NAME} managed section. Do NOT overwrite or remove any other plugin's managed section. Use Option A (managed section only) by default for global restores when Provaxus sections are present.

#### For project backups (`scope: "project"`):

Read `source_directory` from manifest. Compare:
- Backup `CLAUDE.md` vs `{source_directory}/CLAUDE.md`
- Backup `Agents/` vs `{source_directory}/Agents/` (recursive)
- Backup `.claude/agents/` vs `{source_directory}/.claude/agents/` (recursive)

If `source_directory` no longer exists, warn the user and ask whether to proceed.

#### Classification

For each {ORG_NAME}-managed file, classify as one of:

| Status | Meaning |
|--------|---------|
| **Identical** | Same in backup and current — no action needed |
| **Modified** | Exists in both but content differs — will be overwritten with backup version |
| **Added since backup** | Exists now but not in backup — will be removed by restore |
| **Missing (deleted since backup)** | Exists in backup but not now — will be restored |

Additionally flag any Memory file, Knowledge file, or `.user.md` file — these contain user-created content and deserve special attention.

**Never remove or overwrite files belonging to other org plugins' namespaces** during restore — even if they appear in a backup's `agents/` tree. Skip them silently.

#### Present the comparison

```
## Backup vs Current State

Backup: {timestamp}
Scope:  {Global | Project ({name})}
Reason: {reason from manifest}

### CLAUDE.md — Managed Section:
  {Identical | Modified (N lines changed) | Not in backup}

### CLAUDE.md — User Content Outside Markers:
  {Identical | ⚠ N lines differ — restoring full file would overwrite your changes}

### Org-Managed Files (core-rules, disciplines):
  {list files with status — Modified / Identical / Added / Missing}

### Subagent Personalizations (.user.md):
  {list with status}

### Subagent & Orchestrator Memory (*/Memory/):
  {list with status}

### Files that would be REMOVED (added after backup, not in backup):
  {list}

### Files that are IDENTICAL (no change needed):
  {count} files unchanged

### User-Created Files Affected:
  {list any Memory/*.md, KNOWLEDGE.md, or .user.md files from the modified/added/removed categories}
  {or "None — all user-created files are unaffected"}
```

### Step 4: Request Approval

If the backup includes CLAUDE.md and user content outside the markers differs:

Present the restore options:
```
CLAUDE.md restore options:
  A) Restore managed section only (recommended) — preserves your personalizations outside the markers
  B) Restore full file — overwrites everything including your changes outside the markers

Which? (A/B)
```

If the comparison shows Memory/, KNOWLEDGE.md, or .user.md files would be affected, add an extra warning.

Ask: **"Restore this backup? A safety backup of the current state will be created first. (yes/no)"**

If no, stop.

### Step 5: Safety Backup of Current State

Before restoring anything, back up the current state using the same backup format.

**manifest.json:**
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Safety backup before restoring from {restore_backup_timestamp}",
  "skill": "{ORG_ID}:restore-agent-infrastructure",
  "skill_version": "1.0.0",
  "scope": "{same scope as restore target}",
  "source_directory": "{if project scope, absolute path}",
  "restoring_from": "{timestamp of backup being restored}",
  "files_backed_up": ["{list of paths}"]
}
```

Tell the user: "Safety backup created at `~/.claude/backups/{timestamp}/`"

### Step 6: Restore Files

Restore per user choice from Step 4. For each file in backup, copy to target path (creating directories as needed), overwrite existing files. For each file in current state that does NOT exist in the backup, remove it — except Memory/*.md, KNOWLEDGE.md, and .user.md files that were created after the backup (skip removal unless user approved the warning in Step 4).

### Step 7: Report

```
## Restore Complete

Restored from: {backup timestamp}
Safety backup:  ~/.claude/backups/{safety_timestamp}/

| Action   | Count |
|----------|-------|
| Restored | {N} files overwritten with backup version |
| Added    | {N} files restored from backup (were missing) |
| Removed  | {N} files removed (not in backup) |
| Skipped  | {N} files identical (no change needed) |
| Kept     | {N} user files kept despite not being in backup |

CLAUDE.md: {managed section restored | full file restored | not in backup — unchanged}

To undo this restore:
  /{ORG_ID}:restore-agent-infrastructure {safety_timestamp_date}
```
