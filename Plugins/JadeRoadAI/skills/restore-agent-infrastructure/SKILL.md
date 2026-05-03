---
name: restore-agent-infrastructure
description: Use when the user wants to restore a previous agent infrastructure from a backup. Lists available backups, compares to current state, and restores with a safety backup.
version: 1.0.0
---

# Restore Agent Infrastructure

Review and restore from agent infrastructure backups created by `init-core-agents` or `init-project-team`. Every restore creates a safety backup of the current state first, ensuring no work is ever lost.

## Usage
```
/jaderoad:restore-agent-infrastructure                    # List all backups, prompt for selection
/jaderoad:restore-agent-infrastructure 2026-04-30         # Filter backups by date
/jaderoad:restore-agent-infrastructure latest              # Show most recent backup
```

## Arguments

Parse from `$ARGUMENTS`:
- **filter** (optional): A date string (YYYY-MM-DD) to filter backups, or `latest` to select the most recent. If omitted, list all.

---

## Instructions

### Step 1: List Available Backups

Scan `~/.claude/backups/` for timestamped directories. For each, read `manifest.json` to get metadata.

If `~/.claude/backups/` does not exist or contains no valid backup directories:
- Tell the user: "No agent infrastructure backups found. Backups are created automatically when running `/jaderoad:init-core-agents` or `/jaderoad:init-project-team`."
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
| 1 | 2026-04-30 14:30:00 | Global             | Pre-migration before init-core-agents v2.0.0    |
| 2 | 2026-04-28 09:15:00 | Project (Nova)     | Pre-migration before init-project-team v3.0.0   |
| 3 | 2026-04-28 09:14:00 | Global             | Safety backup before restore from 2026-04-27... |
```

### Step 2: Select Backup

If only one backup matches the filter (or `latest` was specified), use it automatically.

Otherwise, ask: **"Which backup to review? (enter number)"**

### Step 3: Compare Backup to Current State

Read the selected backup's `manifest.json` to determine scope and source paths.

#### For global backups (`scope: "global"`):

Compare the backup against the current installed state:
- Backup `CLAUDE.md` vs current `~/.claude/CLAUDE.md`
- Backup `agents/` vs current `~/.claude/agents/` (recursive)

#### For project backups (`scope: "project"`):

Read `source_directory` from manifest. Compare:
- Backup `CLAUDE.md` vs `{source_directory}/CLAUDE.md`
- Backup `Agents/` vs `{source_directory}/Agents/` (recursive)
- Backup `.claude/agents/` vs `{source_directory}/.claude/agents/` (recursive)

If `source_directory` no longer exists, warn the user and ask whether to proceed (files will be created at the original path).

#### Classification

For each file, classify as one of:

| Status | Meaning |
|--------|---------|
| **Identical** | Same in backup and current — no action needed |
| **Modified** | Exists in both but content differs — will be overwritten with backup version |
| **Added since backup** | Exists now but not in backup — will be removed by restore |
| **Missing (deleted since backup)** | Exists in backup but not now — will be restored |

Additionally flag any file that is a:
- Memory file (`*/Memory/*.md`) — session data
- Knowledge file (`*/KNOWLEDGE.md`) — user's local teaching
- User file (`*.user.md`) — personalizations

These contain user-created content and deserve special attention.

#### Present the comparison

```
## Backup vs Current State

Backup: {timestamp}
Scope:  {Global | Project ({name})}
Reason: {reason from manifest}

### Files that would be RESTORED (overwritten with backup version):
- {path} (modified since backup)
  ...

### Files that would be ADDED BACK (exist in backup, missing now):
- {path}
  ...

### Files that would be REMOVED (added after backup, not in backup):
- {path}
  ...

### Files that are IDENTICAL (no change needed):
  {count} files unchanged

### User-Created Files Affected:
  {list any Memory/*.md, KNOWLEDGE.md, or .user.md files from the above categories}
  {or "None — all user-created files are unaffected"}
```

### Step 4: Request Approval

Ask: **"Restore this backup? A safety backup of the current state will be created first. (yes/no)"**

If no, stop.

If the comparison shows Memory/, KNOWLEDGE.md, or .user.md files would be modified, added back, or removed, add an extra warning before the prompt:

```
⚠ This restore will affect user-created files (Memory, Knowledge, or
  personalizations). These contain session data and personal preferences
  that may have been updated since this backup was taken.
```

Ask: **"Restore anyway? (yes/no)"**
If no, stop.

### Step 5: Safety Backup of Current State

Before restoring anything, back up the current state using the same backup format.

#### For global restores:

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
├── manifest.json
├── CLAUDE.md
└── agents/
```

#### For project restores:

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
├── manifest.json
├── CLAUDE.md
├── Agents/
└── .claude/
    └── agents/
```

**manifest.json:**
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Safety backup before restoring from {restore_backup_timestamp}",
  "skill": "jaderoad:restore-agent-infrastructure",
  "skill_version": "1.0.0",
  "scope": "{same scope as restore target}",
  "source_directory": "{if project scope, absolute path}",
  "restoring_from": "{timestamp of backup being restored}",
  "files_backed_up": ["{list of paths}"]
}
```

Tell the user: "Safety backup created at `~/.claude/backups/{timestamp}/`"

### Step 6: Restore Files

#### For global restores:

1. Copy backup `CLAUDE.md` → `~/.claude/CLAUDE.md` (overwrite)
2. For each file in backup `agents/`:
   - Copy to corresponding path under `~/.claude/agents/`, creating directories as needed
   - Overwrite existing files
3. For each file in `~/.claude/agents/` that does NOT exist in the backup:
   - Remove it
   - **Exception:** If a Memory/*.md, KNOWLEDGE.md, or .user.md file was created after the backup AND the user did NOT approve the extra warning in Step 4, skip removal and note it in the report.

#### For project restores:

1. Copy backup `CLAUDE.md` → `{source_directory}/CLAUDE.md`
2. Copy backup `Agents/` → `{source_directory}/Agents/` (same rules as global)
3. Copy backup `.claude/agents/` → `{source_directory}/.claude/agents/` (same rules)
4. Remove files that exist in current state but not in backup (same exception rules)

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

To undo this restore:
  /jaderoad:restore-agent-infrastructure {safety_timestamp_date}
```
