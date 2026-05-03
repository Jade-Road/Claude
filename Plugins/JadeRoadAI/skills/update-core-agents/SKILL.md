---
name: update-core-agents
description: Use when the user wants to submit their core agent training updates to the JadeRoad Claude repo for review. Pushes user-authored changes to agent core files as a pull request.
version: 1.0.0
---

# Update Core Agents

Submit core agent training updates to the JadeRoad Claude repository for review. When a user trains a core agent (by explicitly saying "save to core" or "train core agent"), those changes are written to the agent's core files rather than the `.user.md` personalization file. This skill packages those changes as a pull request for the repo managers to review.

**Repository:** `JadeRoadAI/claude`
**Branch strategy:** Creates a feature branch from main, commits changes, pushes, and opens a PR.

## Usage
```
/jaderoad:update-core-agents                           # Interactive — detect and submit all pending changes
/jaderoad:update-core-agents tech-lead                 # Submit changes for a specific agent only
```

## Arguments

Parse from `$ARGUMENTS`:
- **agent_name** (optional): Specific agent to submit. If omitted, detect all agents with pending changes.

---

## Instructions

### Step 1: Locate the Source Repo

Check if the JadeRoad Claude repo is cloned locally. Search in order:
1. `~/src/JadeRoadAI/claude`
2. `~/code/JadeRoadAI/claude`
3. Current working directory (if it's the Claude repo)
4. Ask the user for the path

Verify it's the correct repo by checking:
- `git remote -v` contains `JadeRoadAI/claude`
- The `Plugins/JadeRoadAI/` directory exists

If not found, tell the user:
```
The JadeRoad Claude repo isn't cloned locally. Clone it first:
  git clone https://github.com/JadeRoadAI/claude.git ~/src/JadeRoadAI/claude
```
Stop.

### Step 2: Identify Changes to Submit

Compare the user's installed agent files against the repo source to find core training updates.

For each agent in the plugin's `agents/` directory, compare:
- **User's installed copy:** `~/.claude/agents/{agent-name}.md`
- **Repo source:** `{repo}/Plugins/JadeRoadAI/agents/{agent-name}.md`

Also check for changes to core memory files that were updated via "save to core" training:
- `~/.claude/agents/Governance/*/Memory/*.md` → compare against repo equivalents if they exist

**Only include files that differ.** Use `diff` to compare. Skip `.user.md` files — those are personal and never submitted.

If a specific `agent_name` was provided, only check that agent.

If no differences found:
- Tell the user: "No core agent changes detected. Training updates saved to `.user.md` files are personal and don't need to be submitted."
- Stop.

### Step 3: Show Changes for Review

For each file with differences, show:
```
Changes detected:

1. agents/tech-lead.md
   - Added: new architecture principle about event sourcing
   - Modified: auto-reject patterns section

2. Governance/Compass/Memory/PATTERNS.md
   - Added: event sourcing pattern for audit-heavy domains
```

Show a concise summary, not the full diff. Ask:
- "Submit these changes as a pull request? (yes/no)"
- If no, stop.

### Step 4: Create Feature Branch

In the repo directory:

```bash
git checkout main
git pull origin main
git checkout -b agent-training/{date}-{summary}
```

Where:
- `{date}` is today's date in `YYYY-MM-DD` format
- `{summary}` is a short kebab-case summary (e.g., `tech-lead-event-sourcing`)

If a specific agent was provided, use its name in the branch: `agent-training/{date}-{agent-name}`

### Step 5: Copy Changes to Repo

For each changed file, copy from the user's installed location to the repo:

| User Location | Repo Destination |
|---------------|-----------------|
| `~/.claude/agents/{agent}.md` | `Plugins/JadeRoadAI/agents/{agent}.md` |
| `~/.claude/agents/Governance/*/Memory/*.md` | *(only if repo has corresponding paths)* |
| `~/.claude/agents/core-rules.md` | `Plugins/JadeRoadAI/skills/init-core-agents/templates/core-rules.md` |
| `~/.claude/agents/disciplines/*.md` | `Plugins/JadeRoadAI/skills/init-core-agents/templates/disciplines/*.md` |

**Do NOT copy:**
- `.user.md` files (personal)
- `{OrchestratorName}/Memory/*` (personal state)
- Files that haven't changed

### Step 6: Commit and Push

Stage the changed files and commit:

```bash
git add {changed files}
git commit -m "agent-training: {summary of what was trained}

Trained by: {user name from CLAUDE.md}
Agents updated: {list of agent names}
Session date: {today's date}

Co-Authored-By: Claude <noreply@anthropic.com>"

git push -u origin agent-training/{branch-name}
```

### Step 7: Create Pull Request

Use `gh pr create`:

```bash
gh pr create \
  --title "Agent Training: {summary}" \
  --body "$(cat <<'EOF'
## Agent Training Submission

**Trained by:** {user name}
**Date:** {today's date}
**Agents updated:** {list}

## Changes

{summary of each change}

## Review Notes

These changes were made during a core agent training session. The user explicitly requested these be saved to core agent definitions for redistribution to all users.

Please review for:
- Accuracy of new knowledge/patterns
- Compatibility with existing agent behavior
- No user-specific preferences leaked into core definitions

---
Submitted via `/jaderoad:update-core-agents`
EOF
)"
```

### Step 8: Report

Report:
- PR URL (clickable)
- Branch name
- Files changed
- Remind the user:
  - The PR needs approval from repo managers before it's merged
  - Once merged, all users will receive the update when the plugin is next refreshed
  - Their personal `.user.md` preferences remain local and are unaffected
  - To check PR status: `gh pr status` or visit the repo on GitHub
