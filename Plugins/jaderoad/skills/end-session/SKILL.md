---
name: end-session
description: Use at the end of a Claude Code session. Extracts decisions, backlog items, and blockers from the conversation for your confirmation, writes approved items to agent memory, commits agent files to the shared Project-Agents repo, and generates a session summary.
version: 1.0.0
---

# End Session

Close out a working session cleanly. Extracts what was decided and learned during the session, presents it for your review, writes approved items to agent memory files, commits everything to the shared Project-Agents repo, and generates a branded session summary.

**What it does:**
- Scans conversation history for decisions, backlog items, and blocker changes
- Presents extracted items for your confirmation before writing anything
- Writes approved items to `Agents/*/Memory/` files
- Commits all changed agent files to the Project-Agents repo
- Generates a branded HTML session summary (using project branding if available)
- Updates the global orchestrator STATE.md

Note: Agent files are also committed automatically by the `Stop` hook whenever Claude Code stops. `end-session` ensures memory files are written first and the commit includes meaningful context — run it before closing for the cleanest result.

## Usage
```
/{ORG_ID}:end-session
```

No arguments.

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

### Step 1: Identify Project

Walk up from the current working directory looking for `.claude-project`. Read:
- `project`, `codename`, `client`, `root`, `src_root`

If not found: "This directory is not under an initialized project. Run `/{ORG_ID}:start-session` first." Stop.

### Step 2: Read Current Memory Files

Read the current state of these files before extracting — to avoid duplicating existing entries and to assign correct sequential IDs:

- `Agents/Architect/Memory/DECISIONS.md` — find the highest existing decision ID by grepping for the last `### [D-` heading and extracting the number. If no entries exist, start at D-001.
- `Agents/Tracker/Memory/BACKLOG.md`
- `Agents/Tracker/Memory/BLOCKERS.md` — find the highest existing blocker ID. If no entries exist, start at B-001.
- `Agents/Tracker/Memory/STATE.md`

**Get the actual current time** for the session summary filename (Step 8):
```bash
date +"%Y%m%d_%I%M%p_%Z"
```
Store the result — do not infer time from conversation context.

### Step 3: Extract from Conversation History

Scan the full conversation for:

**Decisions** — architectural, design, or process choices that were explicitly made:
- Look for: "we decided", "going with", "the approach is", confirmed tradeoffs, patterns chosen
- Exclude: passing comments, rejected ideas, things already in DECISIONS.md

**Backlog items** — work identified but not completed this session:
- Look for: "we should", "TODO", "follow-up", "next steps", deferred items
- Exclude: items already in BACKLOG.md, things completed this session

**Blocker changes:**
- New: things explicitly blocking progress
- Resolved: blockers fixed or worked around this session

**STATE update** — what was completed, what is now active, what's next

### Step 4: Present for Confirmation

Present all extracted items before writing anything:

```
Here's what I found in this session. Confirm what to record:

── Decisions ────────────────────────────────────────────────────────────
[x] [D-{next}] Use event sourcing for the audit log
    Context: Evaluated polling vs. events — events avoid N+1 overhead
    Decision: Implement using {framework} event store
    Status: ACTIVE

[x] [D-{next+1}] ...

── Backlog Items ─────────────────────────────────────────────────────────
[x] Add rate limiting to /api/export endpoint
[ ] Investigate flaky test in AuthController (unchecked — skip?)

── Blocker Updates ───────────────────────────────────────────────────────
[x] RESOLVED: [B-004] Auth token expiry — fixed by extending TTL in config
[ ] NEW: [B-{next}] Staging deployment blocked — need credentials from Gerry

── STATE Update ──────────────────────────────────────────────────────────
Completed this session:
  - {list}
Now active:
  - {list}
Up next:
  - {list}

Confirm, edit, or deselect items. Reply when ready.
```

Wait for user response. User can accept, edit text, uncheck to skip, or add missed items.

**Do not write anything until the user confirms.**

### Step 5: Write Confirmed Memory Updates

Append only confirmed items. Never rewrite or truncate existing files.

**DECISIONS.md** — append each decision:
```markdown
### [D-{ID}] {Title}
- **Date:** {YYYY-MM-DD}
- **Context:** {context}
- **Decision:** {decision}
- **Consequences:** {consequences}
- **Status:** ACTIVE
```

**BACKLOG.md** — append to Priority Queue:
```markdown
- {item description}
```

**BLOCKERS.md:**
- Resolved: update `Status` to `RESOLVED`, add `- **Resolved:** {date} — {how}` line
- New blocker:
```markdown
### [B-{ID}] {Title}
- **Severity:** {CRITICAL | HIGH | MEDIUM | LOW}
- **Status:** OPEN
- **Blocks:** {what is blocked}
- **Owner:** {who can unblock}
- **Description:** {details}
```

**STATE.md** — update three sections:
- Move completed work from "What's Active" to "What's Complete"
- Update "What's Active" with current in-progress items
- Update "What's Next" with queued items
- Update "Last Updated" date

### Step 6: Code Changes

Check for uncommitted code changes and offer to commit and/or push them.

#### Step 6a: Detect Uncommitted Changes

```bash
git status --short
```

If the working tree is clean: skip to Step 7. Say nothing — clean is expected.

If uncommitted changes exist, list them and present:

```
── Uncommitted code changes ──────────────────────────────────────────────

  M  src/server/routes/costs.ts
  ?? client/src/components/NewWidget.tsx
  ...

How would you like to handle these?

  [1] Commit local only  — stage + commit, no push
  [2] Sync              — commit → pull → merge → push ({branch})
  [3] Sync & PR         — sync + open PR                    [feature branches only]
  [4] Full workflow     — build → test → commit → sync → rebuild → test → changelog → push → deploy  [main only]
  [s] Skip

```

Show only valid options for the current branch:
- On `main`: options 1, 2, 4, s (no option 3)
- On any other branch: options 1, 2, 3, s (no option 4)

Wait for selection before proceeding.

#### Step 6b: Collect Commit Message

Ask: "Commit message?" Suggest a one-line message based on the changed file names. User can accept or override.

#### Step 6c: Execute Selected Option

**Option 1 — Commit local only:**
```bash
git add -A
git commit -m "{message}"
```
Report: "Committed locally — not pushed."

---

**Option 2 — Sync:**
```bash
git add -A
git commit -m "{message}"
git pull
```
If conflicts: surface them, stop. "Merge conflict — resolve manually then push." Do not push.

If on `main`: run build then tests before pushing (see build/test detection in Option 4). If either fails, stop.

If clean:
```bash
git push
```
Report: "Synced to origin/{branch}."

---

**Option 3 — Sync & PR (feature branches only):**
```bash
git add -A
git commit -m "{message}"
git pull
```
If conflicts: surface and stop as above.

If clean:
```bash
git push -u origin {branch}
```

Then create PR:
```bash
gh pr create \
  --title "{commit message}" \
  --body "$(cat <<'EOF'
## Summary

{one-paragraph summary of session changes}

## Test plan

- [ ] Verify changes in staging/dev environment

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Report: "Synced and PR created: {PR URL}"

---

**Option 4 — Full workflow (main only):**

Detect build and test commands from `CLAUDE.md` in the project root, the `## Build Commands` section.

1. **Build** — run build command. If fails: stop.
2. **Tests** — run test command. If fails: stop.
3. **Commit** — `git add -A && git commit -m "{message}"`
4. **Pull** — `git pull`. If conflicts: stop.
5. **Rebuild** — run build again. If fails: stop.
6. **Retest** — run tests again. If fails: stop.
7. **Changelog** — update `docs/changelog.html`: prepend a new entry with today's date and summary.
8. **Push** — `git push`

Report: "Full workflow complete — pushed to main."

### Step 7: Commit Agent Files to Project-Agents

If the directory `{src_root}/Project-Agents-{ORG_ID}` does not exist: report "Agent repo not found at `{src_root}/Project-Agents-{ORG_ID}` — skipping commit. Run `/{ORG_ID}:start-session` to set up." Proceed to Step 8.

```
cd "{src_root}/Project-Agents-{ORG_ID}"
# Safety guard: verify remote before operating
REMOTE_URL=$(git remote get-url origin 2>/dev/null)
if [[ "$REMOTE_URL" != *"{GITHUB_ORG}/Project-Agents"* ]]; then
  echo "ABORT: {src_root}/Project-Agents-{ORG_ID} tracks '$REMOTE_URL' — not {GITHUB_ORG}/Project-Agents. Skipping agent file commit."
else
git pull --rebase
git status
```

Pull first to avoid a non-fast-forward push failure. If `git pull --rebase` fails with conflicts, stop and report.

If changes under `{client}/{codename}/`:
```
git add {client}/{codename}/
git commit -m "agent: end-session — {YYYY-MM-DD HH:mm} ({project-name})

{N decisions, M backlog items, K blocker changes}"
git push
fi
```

Report: "Committed N file(s) with session memory updates." or "No changes to commit."

### Step 8: Generate Session Summary

**Project branding step (run before building HTML):**
1. Check if `Agents/Brand/Memory/BRAND.md` exists and has color values in the `## Colors` section
2. If yes: extract primary color (`<!-- Primary: #hex -->`) and accent color (`<!-- Accent: #hex -->`)
3. Override default CSS variables in the HTML template using the `/{ORG_ID}:session-summary` skill template
4. If BRAND.md is missing or colors are unpopulated: use {ORG_NAME} default brand colors

Build the HTML session summary using the session-summary skill template. Populate with:
- **Key Metrics** — files changed, decisions recorded, backlog items added
- **What Was Accomplished** — group by workstream, lead with outcomes not process
- **Blockers Resolved** — real blockers diagnosed and fixed this session
- **Decisions** — confirmed decisions from Step 4
- **Next Steps** — confirmed backlog items and open blockers
- **Files Changed** — if significant

Write to: `docs/session-summaries/{Agent}_{yyyyMMdd}_{hhmmtt}_{TZ}_{BriefTitle}.html`

Use the timestamp acquired in Step 2. Create `docs/session-summaries/` if it doesn't exist.

### Step 9: Update Global State

Detect the orchestrator name: read the managed section of `~/.claude/CLAUDE.md` between `<!-- @{ORG_ID}:init-agent-infrastructure@` and `<!-- @{ORG_ID}:init-agent-infrastructure-end -->`. Extract the first `# {Name} — Lead Orchestrator` heading. If not found, scan all `<!-- @*:init-agent-infrastructure@` managed sections for that heading. Fall back to scanning `~/.claude/agents/` for a subdirectory containing `Memory/STATE.md` but no `IDENTITY.md`. Fall back to `Operator` if neither yields a name.

If `~/.claude/agents/{OrchestratorName}/Memory/STATE.md` exists:
- Update Active Projects row for this project: set "Last Session" to today, update "Current Focus"
- Update "Last Updated" date

### Step 10: Infrastructure Drift Check

After memory writes and summary generation, check whether locally trained infrastructure differs from what's committed in the Claude plugin repo.

#### Step 10a: Locate Claude Repo

Check `~/src/{ORG_NAME}/Claude`. Confirm `git remote -v` contains `{GITHUB_ORG}/Claude` and `Plugins/{ORG_ID}/` exists.

If not found, present:

```
── Plugin repo not found ───────────────────────────────────────────────────────

The Claude plugin repo is not cloned locally. Without it, changes you make to
core-rules.md or discipline files cannot be detected or submitted to the org.

Clone it now to ~/src/{ORG_NAME}/Claude? (yes/no)
```

- **Yes** → run:
  ```bash
  git clone https://github.com/{GITHUB_ORG}/Claude.git ~/src/{ORG_NAME}/Claude
  ```
  If clone fails, report the error and skip Step 10.
  On success, report `✓ Infrastructure in sync` and skip to Step 11.

- **No** → add to Step 11 report:
  ```
    ⚠ drift check skipped — plugin repo not cloned
      Core rule and discipline training will remain local only.
  ```
  Skip to Step 11.

#### Step 10b: Detect Drift

Run both checks in parallel. Never include `.user.md` files or `Memory/*.md` files.

**Category A — Installed infrastructure vs. committed repo templates:**

```bash
git -C "{claude-repo}" show HEAD:Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/core-rules.md | sed 's/\r$//' > /tmp/repo-{ORG_ID}-core-rules.md
sed 's/\r$//' ~/.claude/agents/{ORG_ID}/core-rules.md > /tmp/local-{ORG_ID}-core-rules.md
diff /tmp/repo-{ORG_ID}-core-rules.md /tmp/local-{ORG_ID}-core-rules.md
```

Repeat for each discipline file in `~/.claude/agents/{ORG_ID}/disciplines/`. Clean up temp files after.

**Do NOT compare** `~/.claude/agents/core-rules.md` or `~/.claude/agents/disciplines/` — those belong to other org plugins' namespaces — not managed by this plugin.

**Category B — Uncommitted changes in the Claude repo working tree:**
```bash
git -C "{claude-repo}" diff --name-only HEAD -- Plugins/{ORG_ID}/
git -C "{claude-repo}" ls-files --others --exclude-standard -- Plugins/{ORG_ID}/
```

#### Step 10c: Report and Offer PR

**No drift:** add to Step 11 report: `✓ Infrastructure in sync`

**Drift detected:** present grouped summary and ask:

```
── Infrastructure drift detected ─────────────────────────────────────────

── Infrastructure files (core-rules / disciplines) ──────────────────────
  • agents/core-rules.md — N lines changed

── Uncommitted repo changes ──────────────────────────────────────────────
  • Plugins/{ORG_ID}/skills/end-session/SKILL.md — modified

Total: {N} file(s) pending

Submit as pull request to {GITHUB_ORG}/Claude? (yes/no)
```

- **No:** add to Step 11 report: `  ⚠ {N} infrastructure file(s) pending — answer yes at next start-session or end-session.`
- **Yes:** delegate to `/{ORG_ID}:publish-agent-infrastructure-changes`.

### Step 11: Report

```
Session closed — {project-name}

✓ Memory updated: {N decisions, M backlog items, K blocker changes}
✓ Code: {Committed locally | Synced to origin/{branch} | Synced + PR: {URL} | Deployed via full workflow}   [omit if skipped]
✓ Agent files committed to Project-Agents/{client}/{codename}
✓ Session summary: docs/session-summaries/{filename}
✓ Global state updated
✓ Infrastructure in sync                            [or: PR URL, or pending warning]

Teammates: run /{ORG_ID}:start-session to pick up these updates.
```
