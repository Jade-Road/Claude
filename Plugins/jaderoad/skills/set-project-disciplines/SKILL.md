---
name: set-project-disciplines
description: Use when the user wants to view, add, or remove discipline imports for the current project. Lists active disciplines with the option to remove, and available disciplines with the option to include.
version: 1.0.0
---

# Set Project Disciplines

View and modify which {ORG_NAME} disciplines are imported in the current project's CLAUDE.md. Disciplines define role-specific rules and standards (e.g., TDD for development).

**Coexistence note:** {ORG_NAME} disciplines live at `~/.claude/agents/{ORG_ID}/disciplines/`. This skill only manages imports that point to that path. Provaxus discipline imports (at `~/.claude/agents/disciplines/`) are not touched.

## Usage
```
/{ORG_ID}:set-project-disciplines       # Interactive — show current state, prompt for changes
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

### Step 1: Verify Project CLAUDE.md Exists

Read `CLAUDE.md` in the current working directory.

If it doesn't exist or is empty:
- Tell the user: "No project CLAUDE.md found. Run `/{ORG_ID}:start-session` first to set up this project."
- Stop.

### Step 2: Scan Available {ORG_NAME} Disciplines

Read all `.md` files in `~/.claude/agents/{ORG_ID}/disciplines/`. For each file:
1. Extract the filename without extension as the discipline name
2. Read the first line that starts with `#` to get the display name
3. Read the version marker comment if present (e.g., `<!-- @{ORG_ID}:discipline-development@1.0.0 -->`)

Build the available disciplines list from `~/.claude/agents/{ORG_ID}/disciplines/` only.

If the directory doesn't exist or is empty:
- Tell the user: "No {ORG_NAME} disciplines installed. Run `/{ORG_ID}:init-agent-infrastructure` first to deploy the discipline library."
- Stop.

### Step 3: Identify Active {ORG_NAME} Disciplines

Scan the project CLAUDE.md for lines matching the {ORG_NAME} pattern specifically:
```
@~/.claude/agents/{ORG_ID}/disciplines/{name}.md
```

Build the active disciplines list from these matches only. Do NOT count Provaxus discipline imports (`@~/.claude/agents/disciplines/{name}.md`) as {ORG_NAME} disciplines.

### Step 4: Display Current State

Show the user a unified list of all {ORG_NAME} disciplines with their activation status:

```
Project: {project directory name}

{ORG_NAME} Disciplines:

  [active]
  ✓ development (v1.0.0) — Software development (TDD, coverage tiers, quality gates)

  [available]
  ○ marketing — Marketing operations
  ○ (other available disciplines)
```

Use `✓` for active, `○` for available but not active.

If disciplines from other org plugins are present in CLAUDE.md (any `@~/.claude/agents/<other_org_id>/disciplines/` imports), note them separately but do not manage them:
```
  [Other-org disciplines — managed by that org's plugin]
  ✓ development (other-org)
```

### Step 5: Prompt for Changes

Ask the user what they want to change:

```
What would you like to change?
  - Add: type discipline name(s) to add (e.g., "add marketing")
  - Remove: type discipline name(s) to remove (e.g., "remove development")
  - Nothing: press Enter or type "done" to exit without changes
```

If the user types nothing / "done" / "no" / "n", exit without changes.

Parse the user's response to extract one or more add/remove instructions. Allow shorthand like "add dev" (partial match against available discipline names).

### Step 6: Validate and Apply Changes

**Before making any changes:** Read and snapshot the full content of the project CLAUDE.md. If the edit produces a result that looks malformed, restore from the snapshot and report the issue.

For each requested change:

#### Adding a {ORG_NAME} discipline:
1. Match the name against available disciplines in `~/.claude/agents/{ORG_ID}/disciplines/`.
2. Check it isn't already active — if so, skip and inform the user.
3. Find the {ORG_NAME} discipline imports location in the project CLAUDE.md:
   - Existing `@~/.claude/agents/{ORG_ID}/disciplines/` lines, OR
   - The `{{DISCIPLINE_IMPORTS}}` placeholder, OR
   - The line `<!-- No {ORG_NAME} disciplines selected. Add @imports to load discipline rules. -->`, OR
   - After the last `@~/.claude/agents/{ORG_ID}/` line if any exist, OR
   - After the first `---` separator
4. Add `@~/.claude/agents/{ORG_ID}/disciplines/{name}.md`.
5. Remove the `<!-- No {ORG_NAME} disciplines selected ... -->` placeholder if present.

**Do NOT insert after** `@~/.claude/agents/disciplines/` lines — those belong to another org plugin and must stay separate.

#### Removing a {ORG_NAME} discipline:
1. Check it's currently active — if not, skip.
2. Remove the `@~/.claude/agents/{ORG_ID}/disciplines/{name}.md` line.
3. If no {ORG_NAME} disciplines remain, insert: `<!-- No {ORG_NAME} disciplines selected. Add @imports to load discipline rules. -->`

**Do NOT remove** `@~/.claude/agents/disciplines/` lines — those belong to another org plugin.

#### Configure project permissions (if not already present):
After any add, ensure `.claude/settings.json` has pre-approved read access to the {ORG_NAME} disciplines folder:

```json
{
  "permissions": {
    "allow": [
      "Read(~/.claude/agents/{ORG_ID}/disciplines/**)",
      "Glob(~/.claude/agents/{ORG_ID}/disciplines/**)"
    ]
  }
}
```

If `.claude/settings.json` already exists: read it, parse the JSON, merge the two new entries into `permissions.allow` (skip any already present), write the merged result back. Never overwrite the full file — preserve existing entries including any other-plugin discipline permissions.

### Step 7: Show Updated State

After any changes, re-read the project CLAUDE.md and show the updated {ORG_NAME} discipline state in the same format as Step 4.

Confirm each change: "Added {name} {ORG_NAME} discipline." / "Removed {name} {ORG_NAME} discipline."

If no changes were made, confirm: "No changes made."
