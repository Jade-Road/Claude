---
name: publish-agent-infrastructure-changes
description: Use when the user wants to submit infrastructure training updates to the {ORG_NAME} Claude repo for review. Detects changes to files owned by init-agent-infrastructure, init-core-agents, and init-project-agents, summarizes them for approval, then creates a PR.
version: 1.0.0
---

# Publish Agent Infrastructure Changes

Submit {ORG_NAME} agent infrastructure updates to the {ORG_NAME} Claude repository for review. Detects changes across all plugin-owned files — {ORG_NAME} core rules, discipline library, orchestrator identity (CLAUDE.md template), agent definitions, skill templates — summarizes them for user approval, and opens a pull request.

**Coexistence note:** This skill manages ONLY {ORG_NAME}-namespaced files. It reads from `~/.claude/agents/{ORG_ID}/` and compares against `Plugins/{ORG_ID}/` in the repo. It never reads from or submits changes to Provaxus-owned paths (`~/.claude/agents/core-rules.md`, `~/.claude/agents/disciplines/`).

**Covers files owned by:**
- `/{ORG_ID}:init-agent-infrastructure` — `{ORG_ID}/core-rules.md`, `{ORG_ID}/disciplines/*.md`, `claude-md-template.md`
- `/{ORG_ID}:init-core-agents` — plugin agent definitions (`agents/*.md`)
- `/{ORG_ID}:init-project-agents` — skill file and templates

**Repository:** `{GITHUB_ORG}/Claude`

## Usage
```
/{ORG_ID}:publish-agent-infrastructure-changes
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

### Step 1: Locate the Source Repo

Check if the {ORG_NAME} Claude repo is cloned locally. Search in order:
1. `~/src/{ORG_NAME}/Claude`
2. Current working directory (check `git remote -v` for `{GITHUB_ORG}/Claude`)
3. Ask the user for the path

Verify it's the correct repo:
- `git remote -v` contains `{GITHUB_ORG}/Claude`
- The `Plugins/{ORG_ID}/` directory exists

If not found:
```
The {ORG_NAME} Claude repo isn't cloned locally. Clone it first:
  git clone https://github.com/{GITHUB_ORG}/Claude.git ~/src/{ORG_NAME}/Claude
```
Stop.

### Step 2: Identify All Pending Changes

Check three categories of changes in parallel.

#### Category A: Infrastructure files (installed → repo template diff)

Compare {ORG_NAME}-namespaced installed files against repo source templates:

| Installed File | Repo Template |
|----------------|---------------|
| `~/.claude/agents/{ORG_ID}/core-rules.md` | `{repo}/Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/core-rules.md` |
| `~/.claude/agents/{ORG_ID}/disciplines/{name}.md` (iterate over every `.md`) | `{repo}/Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/disciplines/{name}.md` |
| `~/.claude/CLAUDE.md` {ORG_NAME} managed block (see normalization below) | `{repo}/Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/claude-md-template.md` |

**CRITICAL: Do NOT compare or submit:**
- `~/.claude/agents/core-rules.md` — belongs to another org plugin namespace
- `~/.claude/agents/disciplines/*.md` — belongs to another org plugin namespace

**CLAUDE.md managed block normalization:**

1. Extract the block between `<!-- @{ORG_ID}:init-agent-infrastructure@{version} -->` and `<!-- @{ORG_ID}:init-agent-infrastructure-end -->`. Drop the marker lines.
2. Detect the orchestrator name from the first `# {Name} — Lead Orchestrator` heading.
3. Detect the owner name from the second line.
4. Replace `{OrchestratorName}` with `{{ORCHESTRATOR_NAME}}` and `{OwnerName}` with `{{OWNER_NAME}}`.
5. Diff the normalized block against `claude-md-template.md`.

If the {ORG_NAME} CLAUDE.md section is the minimal coexistence variant (Case A from init-agent-infrastructure — just the `@import` line), note this in the summary — it's expected when Provaxus is also installed.

**Do NOT include:** `.user.md` files, `Memory/*.md` files, any other plugin's managed section, or content outside the {ORG_NAME} managed markers.

#### Category B: Agent definition files (plugin cache → repo agents/ diff)

Locate the {ORG_NAME} plugin cache directory:

```
~/.claude/plugins/cache/{ORG_ID}/{ORG_ID}/{version}/agents/
```

Compare each agent:

| Plugin Cache File | Repo Source |
|-------------------|-------------|
| `{cache}/agents/tech-lead.md` | `{repo}/Plugins/{ORG_ID}/agents/tech-lead.md` |
| `{cache}/agents/qa-lead.md` | `{repo}/Plugins/{ORG_ID}/agents/qa-lead.md` |
| `{cache}/agents/ux-lead.md` | `{repo}/Plugins/{ORG_ID}/agents/ux-lead.md` |
| `{cache}/agents/{MARKETING_AGENT}.md` | `{repo}/Plugins/{ORG_ID}/agents/{MARKETING_AGENT}.md` |

**Do NOT include:** `.user.md` files, `*/Memory/*.md` files, or files from other org plugins.

#### Category C: Uncommitted repo changes (git working tree)

```bash
git diff --name-only HEAD
git ls-files --others --exclude-standard
```

Collect modified or untracked files under `Plugins/{ORG_ID}/` only. Do NOT include `Plugins/provaxus/` changes.

---

If no changes found in any category:
> "No {ORG_NAME} infrastructure changes detected. All installed files match the current published versions."

Stop.

### Step 3: Show Changes for User Approval

Show a consolidated summary grouped by category:

```
{ORG_NAME} infrastructure changes detected:

── Infrastructure files ({ORG_ID}/core-rules.md / disciplines) ─────────
1. agents/{ORG_ID}/core-rules.md
   Modified: Token Efficiency table updated

2. agents/{ORG_ID}/disciplines/development.md
   Added: coverage tier for integration tests

── Agent definition files ──────────────────────────────────────────────
3. agents/tech-lead.md
   Added: new architecture principle

── Repo working tree (uncommitted skill / template changes) ────────────
4. Plugins/{ORG_ID}/skills/set-project-disciplines/SKILL.md
   Modified

Total: {N} file(s) changed
```

Ask: **"Submit these changes as a pull request to {GITHUB_ORG}/Claude? (yes/no)"**

If no, stop.

### Step 4: Create Feature Branch

```bash
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
git checkout "$DEFAULT_BRANCH"
git pull origin "$DEFAULT_BRANCH"
git checkout -b infra-training/{date}-{summary}
```

### Step 5: Copy Changed Files to Repo

**Category A — Infrastructure files:**

| Source (installed) | Destination (repo template) |
|--------------------|----------------------------|
| `~/.claude/agents/{ORG_ID}/core-rules.md` | `Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/core-rules.md` |
| `~/.claude/agents/{ORG_ID}/disciplines/{name}.md` | `Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/disciplines/{name}.md` |
| `~/.claude/CLAUDE.md` (normalized {ORG_NAME} managed block) | `Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/claude-md-template.md` |

For the CLAUDE.md template: write the normalized block (placeholders restored, marker lines removed). Confirm placeholder restoration succeeded. Do NOT include any other plugin's managed section.

**Category B — Agent definition files:**

| Source (plugin cache) | Destination (repo) |
|-----------------------|-------------------|
| `{cache}/agents/{agent}.md` | `Plugins/{ORG_ID}/agents/{agent}.md` |

**Category C — Uncommitted repo changes:** Already in working tree. Stage in Step 6.

**Never copy:**
- files from other org plugins' namespaces
- `.user.md` files
- `Memory/*.md` files

### Step 6: Commit and Push

```bash
git add \
  Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/core-rules.md \
  Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/disciplines/ \
  Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/claude-md-template.md \
  Plugins/{ORG_ID}/agents/ \
  Plugins/{ORG_ID}/skills/

git commit -m "infra-training: {summary}

Updated by: {user name}
Categories: {list}
Files updated: {list}
Session date: {today's date}

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

git push -u origin infra-training/{branch-name}
```

Only stage files that actually changed. Do not stage unmodified files.

### Step 7: Create Pull Request

```bash
gh pr create \
  --title "Infra Training: {summary}" \
  --body "$(cat <<'EOF'
## {ORG_NAME} Infrastructure Training Submission

**Updated by:** {user name}
**Date:** {today's date}
**Categories changed:** {infrastructure files / agent definitions / skill files}
**Files updated:** {list}

## Changes

{summary of each change grouped by category}

## Review Notes

Please review for:
- Accuracy of new rules, patterns, or agent behaviors
- No user-specific preferences or environment details leaked into shared definitions
- No Provaxus/CodonRX IP included in {ORG_NAME} files
- Compatibility with existing agent behavior
- Discipline/skill changes are consistent with documented architecture

---
Submitted via `/{ORG_ID}:publish-agent-infrastructure-changes`
EOF
)"
```

### Step 8: Report

- PR URL (clickable)
- Branch name
- Files changed (by category)
- Reminder that other org plugins' namespaced files were not included in this submission
