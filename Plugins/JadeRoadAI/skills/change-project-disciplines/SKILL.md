---
name: change-project-disciplines
description: Use when the user wants to view, add, or remove discipline imports for the current project. Shows which disciplines are active and lets the user change them.
version: 1.0.0
---

# Change Project Disciplines

View and modify which disciplines are imported in the current project's CLAUDE.md. Disciplines define role-specific rules and standards (e.g., TDD for development, design system rules for prototyping).

## Usage
```
/jaderoad:change-project-disciplines              # Interactive — show current, prompt for changes
/jaderoad:change-project-disciplines add development
/jaderoad:change-project-disciplines remove development
/jaderoad:change-project-disciplines list
```

## Arguments

Parse from `$ARGUMENTS`:
- **action** (optional): `add`, `remove`, or `list`. If omitted, show current state and prompt.
- **discipline_name** (optional): Name of the discipline to add or remove (without path or extension).

---

## Instructions

### Step 1: Verify Project CLAUDE.md Exists

Read `CLAUDE.md` in the current working directory.

If it doesn't exist or is empty:
- Tell the user: "No project CLAUDE.md found. Run `/jaderoad:init-project-team` first to set up this project."
- Stop.

### Step 2: Scan Available Disciplines

Read all `.md` files in `~/.claude/agents/disciplines/`. For each file:
1. Extract the filename without extension as the discipline name
2. Read the first line that starts with `#` to get the display name
3. Read the version marker comment if present (e.g., `<!-- @jaderoad:discipline-development@1.0.0 -->`)

Build the available disciplines list.

If the disciplines directory doesn't exist or is empty:
- Tell the user: "No disciplines installed. Run `/jaderoad:init-core-agents` first to deploy the discipline library."
- Stop.

### Step 3: Identify Current Disciplines

Scan the project CLAUDE.md for lines matching the pattern:
```
@~/.claude/agents/disciplines/{name}.md
```

Build the current disciplines list from matches.

### Step 4: Display Current State

Show the user:

```
Project: {project directory name}

Active disciplines:
  ✓ development (v1.0.0)

Available disciplines:
  ✓ development — Software development (TDD, coverage tiers, quality gates)
  ○ (other available disciplines listed here)
```

Use ✓ for active, ○ for available but not active.

### Step 5: Handle Action

#### If action is `list` (or no arguments provided and user just wants to see):
- Display the state from Step 4. Done.

#### If action is `add`:
1. If discipline_name was provided, validate it exists in the available list. If not, show available options and stop.
2. If discipline_name was not provided, show available (inactive) disciplines and ask which to add. Allow multiple selections.
3. Check if already active. If so, tell the user and stop.
4. Find the discipline imports section in the project CLAUDE.md. This is identified by:
   - Existing `@~/.claude/agents/disciplines/` lines, OR
   - The `{{DISCIPLINE_IMPORTS}}` comment if no disciplines are active, OR
   - The line `<!-- No disciplines selected. Add @imports to load discipline rules. -->`, OR
   - If none found, insert after the first `---` separator
5. Add the new `@~/.claude/agents/disciplines/{name}.md` line alongside existing discipline imports.
6. Confirm: "Added {discipline_name} discipline to this project."

#### If action is `remove`:
1. If discipline_name was provided, validate it's currently active. If not, show active list and stop.
2. If discipline_name was not provided, show active disciplines and ask which to remove. Allow multiple selections.
3. Remove the `@~/.claude/agents/disciplines/{name}.md` line from the project CLAUDE.md.
4. If no disciplines remain, insert the placeholder comment: `<!-- No disciplines selected. Add @imports to load discipline rules. -->`
5. Confirm: "Removed {discipline_name} discipline from this project."

### Step 6: Show Updated State

After any add/remove, show the updated state (same format as Step 4) so the user can verify.
