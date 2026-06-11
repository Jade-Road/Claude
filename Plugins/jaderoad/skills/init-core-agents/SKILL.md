---
name: init-core-agents
description: Use when the user asks to initialize global {ORG_NAME} subagents on their machine (tech-lead, qa-lead, ux-lead, etc.). Requires init-agent-infrastructure to have been run first.
version: 1.0.0
---

# Init Core Agents

Deploy global subagent personalization and memory for the {ORG_NAME} agent infrastructure. Creates `.user.md` personalization stubs for each plugin subagent and scaffolds the shared memory namespaces they read from.

**Coexistence with other org plugins:** tech-lead, qa-lead, and ux-lead are universal agents shared between orgs. Their `.user.md` personalization files and `Memory/` directories are shared — this skill creates them only if they don't already exist (another org plugin may have created them). The `{MARKETING_AGENT}` agent is {ORG_NAME}-specific and always created. **Never create `{any_other_org}-marketing-sme.user.md` — that file belongs to Provaxus.**

**Prerequisite:** Run `/{ORG_ID}:init-agent-infrastructure` first.

**What this skill does:**
1. Creates `.user.md` personalization stubs for each plugin subagent (shared ones only if missing)
2. Scaffolds global memory namespaces for shared agents (tech-lead, qa-lead read these)
3. Scaffolds `ux-lead/Memory/` for UX pattern memory

## Versioning

| Component | Version Marker | Overwritten on Update |
|-----------|---------------|----------------------|
| `tech-lead.md` (plugin) | `@{ORG_ID}-agent:tech-lead@1.0.0` | Yes |
| `qa-lead.md` (plugin) | `@{ORG_ID}-agent:qa-lead@1.0.0` | Yes |
| `ux-lead.md` (plugin) | `@{ORG_ID}-agent:ux-lead@1.0.0` | Yes |
| `{MARKETING_AGENT}.md` (plugin) | `@{ORG_ID}-agent:{MARKETING_AGENT}@1.0.0` | Yes |
| `*.user.md` | None | **Never** |
| `*/Memory/*.md` | None | **Never** |

## Usage
```
/{ORG_ID}:init-core-agents       # Interactive
```

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

### Step 1: Check Prerequisite

Check `~/.claude/CLAUDE.md` for a {ORG_NAME} managed marker: `<!-- @{ORG_ID}:init-agent-infrastructure@`.

Also accept `~/.claude/agents/{ORG_ID}/core-rules.md` existing as proof of infrastructure.

If neither is found:
> "{ORG_NAME} agent infrastructure has not been set up yet. Run `/{ORG_ID}:init-agent-infrastructure` first to configure your CLAUDE.md, {ORG_NAME} core rules, and discipline library, then return to run this skill."

Stop if the user does not want to run init-agent-infrastructure first.

### Step 2: Detect Existing Subagent Infrastructure

Before making any changes, scan for ALL existing subagent files.

#### 2a: {ORG_NAME}-specific files

Check for the {ORG_NAME} marketing SME personalization file:
- `~/.claude/agents/{MARKETING_AGENT}.user.md`

#### 2b: Shared agent files (may have been created by Provaxus or a prior {ORG_NAME} run)

Check for `.user.md` files and Memory directories for the shared agents:
- `~/.claude/agents/tech-lead.user.md`
- `~/.claude/agents/qa-lead.user.md`
- `~/.claude/agents/ux-lead.user.md`
- `~/.claude/agents/tech-lead/Memory/`
- `~/.claude/agents/qa-lead/Memory/`
- `~/.claude/agents/ux-lead/Memory/`

Note which already exist so the report is accurate.

#### 2c: Other plugins' files — do not touch

Explicitly confirm these are NOT touched:
- `~/.claude/agents/{any_other_org}-marketing-sme.user.md` — belongs to another org plugin
- `~/.claude/agents/core-rules.md` — belongs to another org plugin
- `~/.claude/agents/disciplines/` — belongs to another org plugin

### Step 3: Summarize & Request Approval

**Skip entirely for fresh installs** (no `.user.md` files and no Memory directories).

Present a clear summary:

```
## Existing Subagent Infrastructure Detected

### {ORG_NAME}-Specific Files:
- {MARKETING_AGENT}.user.md: {exists | missing}

### Shared Agent Files (tech-lead, qa-lead, ux-lead):
- tech-lead.user.md: {exists (from another org plugin or prior run) | will create}
- qa-lead.user.md: {exists | will create}
- ux-lead.user.md: {exists | will create}
- tech-lead/Memory/: {exists | will create}
- qa-lead/Memory/: {exists | will create}
- ux-lead/Memory/: {exists | will create}

### Other Orgs' Files (will NOT be touched):
- {any_other_org}-marketing-sme.user.md: {exists | not present}
- core-rules.md: {exists | not present}

### Plan:
| Action | Files |
|--------|-------|
| **Create** ({ORG_NAME}-specific) | {MARKETING_AGENT}.user.md |
| **Create if missing** (shared) | {list of missing shared files} |
| **Skip** (already exist) | {list of existing shared files} |
| **Never touch** (other orgs) | {any_other_org}-marketing-sme.user.md, core-rules.md, disciplines/ |

A backup will be created before any changes.
```

Ask: **"Proceed with {ORG_ID}:init-core-agents v1.0.0? (yes/no)"**

If no, stop.

### Step 4: Backup Existing Infrastructure

**Skip only for completely fresh installs** (no `.user.md` files and no `~/.claude/agents/{ORG_ID}/`).

Back up only {ORG_NAME}-relevant files. Do NOT back up Provaxus-owned files.

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
+-- manifest.json
+-- agents/
    +-- {ORG_ID}/                    (copy of ~/.claude/agents/{ORG_ID}/)
    +-- {MARKETING_AGENT}.user.md  (if exists)
    +-- tech-lead.user.md            (if exists — shared, but safe to back up)
    +-- qa-lead.user.md              (if exists)
    +-- ux-lead.user.md              (if exists)
    +-- tech-lead/                   (if exists — shared memory)
    +-- qa-lead/                     (if exists)
    +-- ux-lead/                     (if exists)
```

Manifest:
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Pre-migration backup before {ORG_ID}:init-core-agents v1.0.0",
  "skill": "{ORG_ID}:init-core-agents",
  "skill_version": "1.0.0",
  "scope": "global",
  "note": "Files from other org plugins' namespaces are NOT backed up here — they have their own backup mechanism.",
  "restore_command": "/{ORG_ID}:restore-agent-infrastructure"
}
```

### Step 5: Create Personalization Stubs

#### 5a: {ORG_NAME}-specific stub (always create if missing)

Create `~/.claude/agents/{MARKETING_AGENT}.user.md` only if it doesn't already exist:

```markdown
# {ORG_NAME} Marketing SME — User Personalization

<!-- This file is yours. Plugin updates never touch it. -->
<!-- {MARKETING_AGENT} reads this at the start of every consultation. -->
<!-- Add your preferences, learned patterns, and custom rules below. -->

(no personalizations yet)
```

#### 5b: Shared agent stubs (create only if missing — another org plugin may have already created these)

For each of these, create ONLY if the file does not already exist. If the file exists (from any source, including another org plugin's init-core-agents), skip it entirely — never overwrite.

| Stub File | Notes |
|-----------|-------|
| `tech-lead.user.md` | Universal agent — shared between orgs |
| `qa-lead.user.md` | Universal agent — shared between orgs |
| `ux-lead.user.md` | Universal agent — shared between orgs |

Template for each:
```markdown
# {Agent-Name} — User Personalization

<!-- This file is yours. Plugin updates never touch it. -->
<!-- {Agent-Name} reads this at the start of every consultation. -->
<!-- Add your preferences, learned patterns, and custom rules below. -->

(no personalizations yet)
```

**NEVER create** `.user.md` files for other orgs' marketing agents — those files belong to their respective plugins.

### Step 6: Scaffold Subagent Memory Directories

Create the global memory namespaces. Skip any directories/files that already exist — do not overwrite content that another org plugin may have written.

All of these directories are shared between Provaxus and {ORG_NAME}. They hold universal agent knowledge that applies regardless of which org project is active.

```
~/.claude/agents/
├── tech-lead/Memory/      (PRINCIPLES.md, PATTERNS.md, DECISIONS.md, BACKLOG.md)
├── qa-lead/Memory/        (POSTURE.md, WATCHLIST.md, HEALTH.md, GATES.md, SCORES.md, BACKLOG.md, STANDARD.md)
└── ux-lead/Memory/        (PATTERNS.md)
```

Create each directory and file only if it doesn't already exist.

#### Memory File Templates (create only if missing)

**tech-lead/Memory/PRINCIPLES.md:**
```markdown
# Architecture Principles — Amendments & Exceptions

**Last Updated:** {DATE}

---

(Record amendments or exceptions to the default architecture principles here)
```

**tech-lead/Memory/PATTERNS.md:**
```markdown
# Universal Patterns

**Last Updated:** {DATE}

---

(Record cross-project patterns: naming conventions, DI patterns, config patterns, error handling)
```

**tech-lead/Memory/DECISIONS.md:**
```markdown
# Tech-Lead Verdict Log

**Last Updated:** {DATE}

---

(Record GO/NO-GO/MORE-INFO-NEEDED verdicts with reasoning)
```

**tech-lead/Memory/BACKLOG.md:**
```markdown
# Tech-Lead Backlog — MORE-INFO-NEEDED

**Last Updated:** {DATE}

---

(Proposals parked pending answers to specific questions)
```

**qa-lead/Memory/POSTURE.md:**
```markdown
# Security Posture

**Last Updated:** {DATE}

---

## Open Items

| ID | Severity | Description | Found | Status |
|----|----------|-------------|-------|--------|

(none — newly initialized)
```

**qa-lead/Memory/WATCHLIST.md:**
```markdown
# Security Watchlist

**Last Updated:** {DATE}

---

(Items being monitored but not yet actionable)
```

**qa-lead/Memory/HEALTH.md:**
```markdown
# Test Health Dashboard

**Last Updated:** {DATE}

---

| Project | Test Count | Last Verified | Status |
|---------|-----------|---------------|--------|

(populate per-project as test suites are established)
```

**qa-lead/Memory/GATES.md:**
```markdown
# Quality Gates Status

**Last Updated:** {DATE}

---

| Project | All Tests Pass | Coverage Met | No Regression | TDD Compliant | Independent QA |
|---------|---------------|-------------|---------------|---------------|----------------|

(populate per-project)
```

**qa-lead/Memory/SCORES.md:**
```markdown
# Test Quality Scores

**Last Updated:** {DATE}

---

| Project | Score | Trend | Last Assessed | Notes |
|---------|-------|-------|--------------|-------|

(populate per-project as test suites are reviewed)
```

**qa-lead/Memory/BACKLOG.md:**
```markdown
# Test Remediation Backlog

**Last Updated:** {DATE}

---

(tests flagged for quality issues, awaiting remediation)
```

**qa-lead/Memory/STANDARD.md:**
```markdown
# Test Quality Standard — Quick Reference

A good test:
1. Fails when the behavior it guards is broken
2. Passes when the behavior is correct
3. Tests ONE logical behavior
4. Has a name that describes the expected behavior
5. Uses the real system under test (mocks only at boundaries)
6. Asserts on output/state, not on setup data
7. Survives cosmetic refactors without breaking
```

**ux-lead/Memory/PATTERNS.md** (skip if exists):
```markdown
# UX Patterns & Recurring Issues

**Last Updated:** {DATE}

---

(Record UX patterns, accessibility findings, and recurring issues across projects)
```

### Step 7: Report

Report what was created, distinguishing between {ORG_NAME}-specific and shared:

**{ORG_NAME}-specific:**
- `{MARKETING_AGENT}.user.md` — created or already existed

**Shared (universal agents):**
- `tech-lead.user.md` — created fresh / already existed (preserved)
- `qa-lead.user.md` — created fresh / already existed (preserved)
- `ux-lead.user.md` — created fresh / already existed (preserved)
- `tech-lead/Memory/` — created / already existed (preserved)
- `qa-lead/Memory/` — created / already existed (preserved)
- `ux-lead/Memory/` — created / already existed (preserved)

**Not touched (other org plugins):**
- `{any_other_org}-marketing-sme.user.md`, `core-rules.md`, `disciplines/`

Remind the user:
- Subagents auto-delegate based on task context — no manual commands needed
- tech-lead, qa-lead, ux-lead apply to ALL projects regardless of org
- {MARKETING_AGENT} applies to {ORG_NAME} projects only
- When a shared agent learns something from a {ORG_NAME} context, it writes to `tech-lead.user.md` — be aware this file is also read in Provaxus contexts (that's intentional: the user's technical preferences apply to both orgs)
- Use `/{ORG_ID}:start-session` to set up {ORG_NAME} project agents
- Use `/{ORG_ID}:restore-agent-infrastructure` to restore from any backup

---

## Architecture Reference

After running this skill alongside other org plugins, the user's machine will additionally have:

```
~/.claude/
└── agents/
    ├── {MARKETING_AGENT}.user.md    ← {ORG_NAME}-specific (NEVER overwritten)
    ├── tech-lead.user.md                 ← Shared — may already exist from another installed org plugin
    ├── qa-lead.user.md                   ← Shared — may already exist from another installed org plugin
    ├── ux-lead.user.md                   ← Shared — may already exist from another installed org plugin
    ├── tech-lead/Memory/                 ← Shared — created if missing
    ├── qa-lead/Memory/                   ← Shared — created if missing
    └── ux-lead/Memory/                   ← Shared — created if missing
```

**Other org plugins' files are never touched:**
```
    ├── {any_other_org}-marketing-sme.user.md     ← Other org plugin only — DO NOT TOUCH
    ├── core-rules.md                     ← Other org plugin only — DO NOT TOUCH
    └── disciplines/                      ← Other org plugin only — DO NOT TOUCH
```
