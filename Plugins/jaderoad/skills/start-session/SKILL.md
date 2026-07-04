---
name: start-session
description: Use at the start of a Claude Code session. Detects whether project agents are initialized, offers to set them up or update them if needed, then syncs agent knowledge and code.
version: 1.0.0
---

# Start Session

Run at the start of every working session. Verifies prerequisites, detects the project's agent infrastructure, initializes it if missing or outdated (with your approval), then syncs agent files and code.

**What it does:**
- Verifies `gh` CLI is installed — offers to install it; aborts cleanly if declined
- Verifies global agent infrastructure is set up — runs `init-agent-infrastructure` inline if not
- Recommends `init-core-agents` if subagent personalization hasn't been deployed
- Walks up the directory tree to find the project's `.claude-project` marker
- If not found: offers to initialize project agents (full setup flow)
- If found but outdated: offers to update agent templates and migrate legacy structures
- If current: skips directly to sync
- Commits any pending agent file changes from last session
- Pulls latest shared agent knowledge from Project-Agents repo
- Syncs agent files into the project directory
- Pulls latest code from the project repo
- Warns if uncommitted code changes exist

## Usage
```
/{ORG_ID}:start-session
```

No arguments. All context is detected automatically from `.claude-project` or inferred from the current directory.

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

### Phase 0: Prerequisites

Run all three checks before touching any project files. If any check aborts, stop — make no changes to the project or global infrastructure.

#### Step 0a: Check gh CLI

Run `gh --version`.

If the command is not found, present:

```
The GitHub CLI (gh) is required for agent infrastructure. It manages the
shared Project-Agents repository that syncs agent knowledge across your team.

Install gh CLI now? (yes/no)
```

- **Yes** → install using the platform-appropriate method:
  - Windows: `winget install --id GitHub.cli`
  - macOS: `brew install gh`
  - Linux: `(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) && sudo mkdir -p -m 755 /etc/apt/keyrings && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null && sudo apt update && sudo apt install gh -y`

  After install, run `gh --version` to verify. If it still fails, stop:
  > "gh install failed — install manually then re-run `/{ORG_ID}:start-session`. No changes were made."

- **No** → stop:
  > "gh CLI is required for agent infrastructure (Project-Agents repo management). No changes were made. Install gh from https://cli.github.com and re-run `/{ORG_ID}:start-session`."

If `gh` is found but not authenticated, continue — authentication will be prompted naturally by the repo steps in Phase 3.

#### Step 0b: Check init-agent-infrastructure

Detect whether {ORG_NAME} global infrastructure has been deployed. Check in order:

1. `~/.claude/CLAUDE.md` contains `<!-- @{ORG_ID}:init-agent-infrastructure@` or `<!-- @{ORG_ID}:init-core-agents@` markers
2. `~/.claude/agents/{ORG_ID}/core-rules.md` exists

Note: `~/.claude/agents/core-rules.md` (without the `{ORG_ID}/` prefix) is an unnamespaced path that may belong to another org's plugin — its presence does NOT satisfy this check.

If **any** signal is found, infrastructure is deployed — skip to Step 0c.

If **none** found, print:

```
Global agent infrastructure has not been set up yet.
Running /{ORG_ID}:init-agent-infrastructure now...
```

Run the full `init-agent-infrastructure` skill inline — all steps in that skill's instructions, in order.

**If the user declines any confirmation step within the init-agent-infrastructure flow, stop entirely. Do not proceed with start-session. No project files are changed.**

On successful completion, continue to Step 0c.

#### Step 0c: Check init-core-agents

Detect whether subagent personalization has been deployed. Check for any of:

- `~/.claude/agents/tech-lead.user.md`
- `~/.claude/agents/tech-lead/Memory/`

If either is found, skip to Phase 1.

If neither is found, print the following recommendation block and continue — **do not block**:

```
── Recommended: complete subagent setup ────────────────────────────────────────

The {ORG_NAME} plugin ships 4 specialist agents that auto-delegate based on
task context — no commands needed. Run /{ORG_ID}:init-core-agents after
this session to unlock personalized memory for each agent.

  Agent                       Auto-invoked when...
  ──────────────────────────  ────────────────────────────────────────────────
  tech-lead                   New features, architectural changes, scope decisions
  qa-lead                     After implementation, before deploy, security/test questions
  ux-lead                     UI work, design review, frontend, accessibility audits
  {MARKETING_AGENT}      {ORG_NAME} marketing: Google Ads, GA4/GTM, brand, market decisions

Without init-core-agents, agents work but cannot save personalized preferences
or cross-session memory. Run it once — it takes under a minute.

────────────────────────────────────────────────────────────────────────────────
```

Continue to Phase 1.

---

### Phase 1: Detect Project

Walk up from the current working directory, checking each directory level for a `.claude-project` file.

**Scenario A — Found in current directory:**
Read the file, extract metadata, proceed to Phase 2.

**Scenario B — Found in an ancestor directory:**
Present to the user:
```
Found project agents at `{path}` ({project-name}, v{version}).
Is this your current project root? (yes/no)
```
- Yes → use that path as project root, proceed to Phase 2
- No → treat as uninitialized, proceed to Scenario C

**Scenario C — Not found anywhere:**
```
Project agents are not initialized for this location.
Set up project agents now? (yes/no)
```
- Yes → proceed to Phase 3 (full init flow)
- No → skip to Phase 4 (sync code only, skip all agent sync steps)

---

### Phase 2: Version Check

Compare the `version` field in `.claude-project` against the `version` field in this skill's own frontmatter (the `version:` line at the top of this file).

**If versions match:** run Phase 2a (legacy discipline check) then proceed to Phase 4.

**If `.claude-project` version is older:**
```
Project agents are at v{old}. Current skill version is v{skill-version}.
Update now? (yes/no)
```
- Yes → run migration steps (Steps 5a–5e below) including Phase 2a, refresh outdated templates, update `version` and `updated` fields in `.claude-project`, then proceed to Phase 4
- No → run Phase 2a then proceed to Phase 4 with existing setup

### Phase 2a: Detect and Migrate Legacy Discipline Imports

Even when versions match, discipline import paths in the project CLAUDE.md may be stale if the project was set up before the `{ORG_ID}/` namespace design was adopted.

Scan the project CLAUDE.md for legacy unnamespaced discipline imports of the form:
```
@~/.claude/agents/disciplines/{name}.md
```

For each found:
1. Check if `~/.claude/agents/disciplines/{name}.md` exists and contains a `@{ORG_ID}:discipline-` version marker. If yes: this import points to a legacy file that was deployed by this plugin.
2. Check if the new namespaced path `~/.claude/agents/{ORG_ID}/disciplines/{name}.md` exists (it should, after `init-agent-infrastructure` upgrade has run). If yes: update the import in the project CLAUDE.md:
   - OLD: `@~/.claude/agents/disciplines/{name}.md`
   - NEW: `@~/.claude/agents/{ORG_ID}/disciplines/{name}.md`
3. If the namespaced file does NOT yet exist: run `/{ORG_ID}:init-agent-infrastructure` first to deploy it, then update the import.

**Do NOT update** discipline imports for other orgs' plugins (imports where the discipline file does NOT have a `@{ORG_ID}:discipline-` marker).

After updating imports, also verify `.claude/settings.json` has the correct permissions entry:
- OLD: `Read(~/.claude/agents/disciplines/**)` and `Glob(~/.claude/agents/disciplines/**)`
- NEW: `Read(~/.claude/agents/{ORG_ID}/disciplines/**)` and `Glob(~/.claude/agents/{ORG_ID}/disciplines/**)`

If the settings file has old-style unnamespaced permission entries for this plugin's disciplines, replace them with the new namespaced paths. Preserve any other entries (other plugins' discipline permissions, hooks, etc.).

---

### Phase 3: Init Flow

Run all steps below. On completion, write `.claude-project` and fall through to Phase 4.

#### Step 1: Gather Project Info

Ask the user for these values (skip any discoverable from context):

1. **Project name** — e.g., "ProjectNova"
2. **Codename** — short lowercase, e.g., "nova"
3. **Framework** — e.g., "C# .NET 8 + React"
4. **Project type** — e.g., "SaaS Platform", "Internal Tool"
5. **Purpose** — one-line description
6. **Build command** — e.g., `dotnet build Nova.sln`
7. **Test command** — e.g., `dotnet test Nova.sln`
8. **Run command** — e.g., `dotnet run --project src/Nova.API`
9. **Project modules** — src/ subdirectories, e.g., "Nova.Core/, Nova.API/"
10. **Disciplines** — scan `~/.claude/agents/{ORG_ID}/disciplines/` for available `.md` files and present as options. Multiple selections allowed.

Try to infer from the current directory where possible.

#### Step 1b: Detect Client ID

Determine `{client-id}` for the Project-Agents repo path. Check in order:

1. **Check `.claude-project`** — if a partial file exists, read the `client` field
2. **Parse git remote URL** — extract org/owner from `git remote get-url origin`
3. **Check parent directory name** — e.g., `~/src/{ORG_NAME}/ProjectNova` → `{ORG_ID}`
4. **Ask the user** if none of the above yields a confident value

**Special case:** Remote org `{GITHUB_ORG}` or parent dir `{ORG_NAME}` → default to `{ORG_ID}`.

Present: _"Detected client ID: `{value}`. Confirm or enter a different value."_

#### Step 2: Detect Existing Project Infrastructure

Scan for ALL existing agent infrastructure before making any changes.

#### Step 3: Summarize & Request Approval

**Skip for fresh installs** (no existing `Agents/`, no project CLAUDE.md, no `.claude/agents/` files).

Present migration plan and ask: **"Proceed with migration to {ORG_ID}:start-session v1.0.0? (yes/no)"**

If no, stop.

#### Step 4: Backup Existing Infrastructure

**Skip only for completely fresh installs.**

```
~/.claude/backups/{YYYY-MM-DD_HHmmss}/
+-- manifest.json
+-- CLAUDE.md
+-- Agents/
+-- .claude/
    +-- agents/
```

**manifest.json:**
```json
{
  "timestamp": "{ISO 8601 UTC}",
  "reason": "Pre-migration backup before {ORG_ID}:start-session v1.0.0",
  "skill": "{ORG_ID}:start-session",
  "skill_version": "1.0.0",
  "scope": "project",
  "source_directory": "{absolute path}",
  "project_name": "{project name}",
  "restore_command": "/{ORG_ID}:restore-agent-infrastructure"
}
```

Tell the user the backup path.

#### Step 5: Migrate Legacy Structures

**Skip for fresh installs.**

##### 5a: Pulse → Specialist merge

If `Agents/Pulse/` detected:
1. Create `Agents/Specialist/Memory/` if missing
2. Move `Agents/Pulse/Memory/STATUS.md` → `Agents/Specialist/Memory/STATUS.md` (skip if target exists)
3. Move `Agents/Pulse/Memory/KNOWN_ISSUES.md` → `Agents/Specialist/Memory/KNOWN_ISSUES.md` (skip if target exists)
4. If `Agents/Pulse/KNOWLEDGE.md` exists, append to `Agents/Specialist/KNOWLEDGE.md`
5. Remove `Agents/Pulse/`

##### 5b: Architect subagent → memory-only

If `.claude/agents/*-architect.md` detected:
1. Remove the file
2. Keep `Agents/Architect/Memory/` and `Agents/Architect/KNOWLEDGE.md` intact

##### 5c: Add KNOWLEDGE.md stubs

If any agent directories lack `KNOWLEDGE.md`, create stubs.

##### 5d: Tracker and Brand-Manager → memory-only

If `{codename}-tracker.md` in `.claude/agents/`:
1. Remove `.claude/agents/{codename}-tracker.md`
2. Remove `Agents/Tracker/IDENTITY.md` if exists
3. Keep `Agents/Tracker/Memory/` and `Agents/Tracker/KNOWLEDGE.md`

If `{codename}-brand-manager.md` in `.claude/agents/`:
1. Remove `.claude/agents/{codename}-brand-manager.md`
2. Remove `Agents/Brand-Manager/IDENTITY.md` if exists
3. Rename `Agents/Brand-Manager/` → `Agents/Brand/`

##### 5e: Check for existing CLAUDE.md

If `CLAUDE.md` exists and is non-empty:
- Show first 15 lines
- Ask: "This project already has a CLAUDE.md. Overwrite, append, or abort?"
- Abort → stop. Append → add `---` separator and append. Overwrite → replace.

#### Step 6: Deploy Project CLAUDE.md

1. Read bundled template: `project-agents-template.md` (in this skill's directory)
2. Replace all placeholders:
   - `{{PROJECT_NAME}}`, `{{CODENAME}}`, `{{FRAMEWORK}}`, `{{PROJECT_TYPE}}`, `{{PURPOSE}}`
   - `{{BUILD_COMMAND}}`, `{{TEST_COMMAND}}`, `{{RUN_COMMAND}}`, `{{PROJECT_MODULES}}`
   - `{{PROJECT_CONSTRAINTS}}` → leave as comment placeholder
   - `{{OWNER_NAME}}` → read from `~/.claude/CLAUDE.md` (between managed markers). If not found, ask.
   - `{{DISCIPLINE_IMPORTS}}` → one `@~/.claude/agents/{ORG_ID}/disciplines/{name}.md` per selected discipline
3. Write to `CLAUDE.md` in project root (per user's Step 5e choice)

#### Step 7: Deploy Project Subagents

Create `.claude/agents/` if missing. Deploy from `templates/subagents/`:

| Template | Deploys To |
|----------|-----------|
| `templates/subagents/specialist.md` | `.claude/agents/{codename}-specialist.md` |

Replace `{PROJECT_NAME}`, `{CODENAME}`, `{DATE}` in template.

**Configure Project Permissions** — create or update `.claude/settings.json`:
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
If `.claude/settings.json` already exists: read it, parse the JSON, merge the two entries into `permissions.allow` (skip any already present, including any other-plugin discipline permissions), then write the merged result back. Never overwrite the full file — preserve all existing entries.

#### Step 8: Scaffold Agent Memory Directories

Create under `{project}/Agents/`. Skip directories that already exist.

```
{project}/Agents/
├── Architect/              # Memory only — Tech-Lead reads directly
│   ├── KNOWLEDGE.md
│   └── Memory/
│       ├── ARCHITECTURE.md
│       ├── PATTERNS.md
│       └── DECISIONS.md
├── Tracker/                # Memory only — Operator reads directly
│   ├── KNOWLEDGE.md
│   └── Memory/
│       ├── STATE.md
│       ├── BLOCKERS.md
│       └── BACKLOG.md
├── Specialist/             # Subagent reads these
│   ├── IDENTITY.md         (shipped, overwritten on upgrade)
│   ├── KNOWLEDGE.md        (team knowledge, never touched)
│   └── Memory/
│       ├── DOMAIN.md
│       ├── API.md
│       ├── INTEGRATIONS.md
│       ├── STATUS.md
│       └── KNOWN_ISSUES.md
└── Brand/                  # Memory only — UX-Lead reads directly
    ├── KNOWLEDGE.md
    └── Memory/
        ├── BRAND.md
        └── AUDIT.md
```

Deploy `Agents/Specialist/IDENTITY.md` from `templates/Specialist-IDENTITY.md`. Replace `{PROJECT_NAME}`, `{CODENAME}`, `{FRAMEWORK}`, `{DATE}`.

Create `KNOWLEDGE.md` stubs for any agent directory missing one:
```markdown
# {AGENT_NAME} — Local Knowledge ({PROJECT_NAME})

<!-- This file is yours. Add project-specific context, preferences, and custom rules here. -->
<!-- It is never overwritten by plugin upgrades. -->

(no local knowledge yet)
```

Memory file templates are defined inline below.

#### Step 8d: Scaffold Plans Directory

Create in project root. Skip entirely if `Plans/` already exists.

```
{project}/
└── Plans/
    ├── Completed/
    └── Plan_Template.md
```

Copy `Plan_Template.md` from this skill's `templates/Plan_Template.md`. Replace `{DATE}` with today's date.

#### Step 8e: Scaffold Tests/CLAUDE.md

Create `Tests/CLAUDE.md` in the project root **only if it doesn't already exist**.

```markdown
# Tests — {PROJECT_NAME}

**Last Updated:** {DATE}
**Framework:** [e.g., xUnit, NUnit, Jest — fill in]

---

Read this before writing any tests for this project.

## Test Framework

[Framework name and version]

## Running Tests

```
{TEST_COMMAND}
```

## Test Naming Convention

[e.g., `{Subject}_{ExpectedBehavior}_When{Condition}()` — fill in]

## Fixture Locations

[Where test data and fixtures live — fill in]

## Coverage Thresholds

| Component | Minimum |
|-----------|---------|
| Core / API | 100% |
| Domain | 95% |
| Features | 85% |
| UI | 70% |

## Mock Boundaries

[What is/isn't allowed to be mocked in this project — fill in]

## Anti-Patterns to Avoid

[Project-specific test anti-patterns discovered here — fill in as they emerge]
```

#### Step 8b: Set Up Project-Agents Repo

Detect `SRC_ROOT` — two levels above project root (e.g., `~/src` from `~/src/{ORG_NAME}/ProjectNova`).

The shared repo always clones to `{SRC_ROOT}/Project-Agents-{ORG_ID}/`.

**Check if repo exists:**
```
gh repo view {GITHUB_ORG}/Project-Agents 2>&1
```
- Exit 0 → exists, skip creation
- Exit non-zero + "Could not resolve to a Repository" → create:
  ```
  gh repo create {GITHUB_ORG}/Project-Agents --private --description "Agent knowledge for all {ORG_NAME} projects"
  ```
- Exit non-zero + "403" / "Must have admin rights" → stop with auth guidance.

**Clone or update:**
- Missing: `git clone https://github.com/{GITHUB_ORG}/Project-Agents.git "{SRC_ROOT}/Project-Agents-{ORG_ID}"`
- Exists: verify remote URL first — run `git -C "{SRC_ROOT}/Project-Agents-{ORG_ID}" remote get-url origin`. If it does not contain `{GITHUB_ORG}/Project-Agents`, abort: "Project-Agents-{ORG_ID} exists but tracks a different remote — remove or rename the directory and re-run." If correct, run `git -C "{SRC_ROOT}/Project-Agents-{ORG_ID}" pull`.

#### Step 8c: Scaffold Project Folder in Agent Repo

Create `{SRC_ROOT}/Project-Agents/{client-id}/{codename}/` with **only Claude agent files**:

```
{client-id}/{codename}/
├── Agents/              (copy of Agents/ — memory and knowledge files only)
└── .claude/
    └── agents/          ({codename}-specialist.md only)
```

**Scope rules — strictly enforced:**
- `Agents/` — copy the full directory as scaffolded in Step 8. Nothing else from the project root.
- `.claude/agents/` — copy only `{codename}-specialist.md`. Not settings.json, not settings.local.json.
- Do NOT copy: `CLAUDE.md`, `PROJECT.user.md`, `.claude-project`, source code, or any other project files.

Commit and push:
```
cd "{SRC_ROOT}/Project-Agents"
git add {client-id}/{codename}/
git commit -m "init: scaffold {project-name} ({client-id}/{codename}) agent knowledge"
git push
```

#### Step 9: Create PROJECT.user.md

Create only if it doesn't already exist. **Never overwrite.**

```markdown
# {PROJECT_NAME} — Personal Preferences

<!-- This file is yours. It is NOT checked into source control. -->
<!-- Each developer has their own copy. -->
<!-- Loaded via @PROJECT.user.md in the project's CLAUDE.md. -->

(no personalizations yet)
```

#### Step 10: Configure Version Control Ignores

**Files to exclude:**
- `PROJECT.user.md`
- `.claude/settings.local.json`
- `.claude/agents/`
- `.claude-project`
- `Agents/`

**For Mercurial:**
Detect by `.hg/` in project root. If confirmed:
- `.hgignore` exists → append (skip entries already present)
- `.hgignore` missing → create it

```
# Claude Code — agent and user-specific files
syntax: glob
PROJECT.user.md
.claude/settings.local.json
.claude/agents/**
.claude-project
Agents/**
```

**For Git (`.gitignore` exists):**
```
# Claude Code — agent and user-specific files
PROJECT.user.md
.claude/settings.local.json
.claude/agents/
.claude-project
Agents/
```

**Rules:**
- Mercurial: create `.hgignore` if missing
- Git: only modify if `.gitignore` already exists
- Only add entries not already present
- If no VCS detected, warn to add exclusions manually

#### Step 11: Write .claude-project

Write to the project root:

```json
{
  "project": "{PROJECT_NAME}",
  "codename": "{codename}",
  "client": "{client-id}",
  "root": "{absolute path to project root}",
  "src_root": "{SRC_ROOT}",
  "version": "{skill_version}",
  "installed": "{YYYY-MM-DD}",
  "updated": "{YYYY-MM-DD}"
}
```

#### Step 12: Register in Global State

Detect the orchestrator name: read the managed section of `~/.claude/CLAUDE.md` between `<!-- @{ORG_ID}:init-agent-infrastructure@` and `<!-- @{ORG_ID}:init-agent-infrastructure-end -->`. Extract the first `# {Name} — Lead Orchestrator` heading. If not found, scan all `<!-- @*:init-agent-infrastructure@` managed sections for that heading. Fall back to scanning `~/.claude/agents/` for a subdirectory containing `Memory/STATE.md` but no `IDENTITY.md`. Fall back to `Operator` if neither yields a name.

If `~/.claude/agents/{OrchestratorName}/Memory/STATE.md` exists:
- Add row: project name, directory, "Active", current focus, client ID, agent path
- Update "Last Updated" date

#### Step 13: Init Report

Report:
- CLAUDE.md written/updated
- Disciplines imported + activated global subagents per discipline
- Subagent files created in `.claude/agents/`
- Agent memory directories created or skipped
- Migration summary if applicable
- `PROJECT.user.md` created or skipped
- `Plans/` directory and `Plan_Template.md` created or skipped
- `Tests/CLAUDE.md` created or skipped
- VCS ignore rules added (which file updated)
- Agent files committed to `Project-Agents/{client-id}/{codename}/`
- `.claude-project` written
- Global STATE.md updated or skipped

**Fall through to Phase 4.**

---

### Phase 4: Session Sync

#### Sync Step 1: Commit Pending Agent Changes

```
cd "{src_root}/Project-Agents"
git status
```

If changes under `{client}/{codename}/`:
```
git add {client}/{codename}/
git commit -m "agent: pre-session snapshot — {YYYY-MM-DD HH:mm} ({project-name})"
git push
```

Report: "Committed N agent file(s)" or "No pending agent changes."

#### Sync Step 2: Pull Latest Agent Repo

```
cd "{src_root}/Project-Agents"
git pull
```

Report commits received or "Already up to date."

#### Sync Step 3: Sync Agent Files Into Project

**Only Claude agent files** — never copy source code or project config.

Copy from agent repo into project. **Never delete** files in project that aren't in agent repo.

- `{src_root}/Project-Agents/{client}/{codename}/Agents/` → `{project}/Agents/`
- `{src_root}/Project-Agents/{client}/{codename}/.claude/agents/` → `{project}/.claude/agents/`

Track: N files updated, M files unchanged.

#### Sync Step 4: Pull Latest Code Repo

- `.git` present → `git pull`
- `.hg` present → `hg pull -u`
- Neither → skip, note "No VCS detected"

#### Sync Step 5: Warn on Uncommitted Code Changes

After pull, check status. If uncommitted changes exist, list them:

> ⚠ Uncommitted code changes detected:
> - M  src/Nova.API/Controllers/UsersController.cs
>
> Commit or stash before starting new work.

Do NOT commit automatically. If clean, say nothing.

#### Sync Step 6: Report

```
Session ready — {project-name}

✓ Agent changes committed and pushed (N files)     [omit if none]
✓ Agent repo pulled (N commits)                    [or: already up to date]
✓ Agent files synced (N updated, M unchanged)
✓ Code repo pulled (N commits)                     [or: already up to date]
⚠ Uncommitted code changes: [N files]              [omit if clean]
✓ Infrastructure in sync                           [or drift warning — see Phase 5]

Agents: Project-Agents/{client}/{codename}
```

---

### Phase 5: Infrastructure Drift Check

After Phase 4 completes, check whether locally trained infrastructure differs from what's committed in the Claude plugin repo. Never block session start — run this as a background check and surface results in the report.

**Skip Phase 5 entirely if Phase 3 (init) ran this session.**

#### Step 5a: Locate Claude Repo

Check `~/src/{ORG_NAME}/Claude`. Confirm with `git remote -v` (must contain `{GITHUB_ORG}/Claude`) and `Plugins/{ORG_ID}/` must exist.

If not found, present:

```
── Plugin repo not found ───────────────────────────────────────────────────────

The Claude plugin repo is not cloned locally. Without it, changes you make to
core-rules.md or discipline files cannot be detected or submitted to the org —
they would sit in ~/.claude/agents/ with no pathway back.

Clone it now to ~/src/{ORG_NAME}/Claude? (yes/no)
```

- **Yes** → run:
  ```bash
  git clone https://github.com/{GITHUB_ORG}/Claude.git ~/src/{ORG_NAME}/Claude
  ```
  If clone fails, update report line to `  ⚠ drift check skipped — clone failed ({error})` and skip to end of Phase 5.
  On success, report `✓ Infrastructure in sync` and skip to end of Phase 5.

- **No** → update report line to:
  ```
    ⚠ drift check skipped — plugin repo not cloned
      Core rule and discipline training will remain local only.
  ```
  Skip to end of Phase 5.

#### Step 5b: Detect Drift

Run both checks in parallel. Never include `.user.md` files or `Memory/*.md` files.

**Category A — Installed infrastructure vs. committed repo templates:**

```bash
git -C "{claude-repo}" show HEAD:Plugins/{ORG_ID}/skills/init-agent-infrastructure/templates/core-rules.md | sed 's/\r$//' > /tmp/repo-{ORG_ID}-core-rules.md
sed 's/\r$//' ~/.claude/agents/{ORG_ID}/core-rules.md > /tmp/local-{ORG_ID}-core-rules.md
diff /tmp/repo-{ORG_ID}-core-rules.md /tmp/local-{ORG_ID}-core-rules.md
```

Repeat for each discipline file in `~/.claude/agents/{ORG_ID}/disciplines/`. Clean up temp files after.

**Do NOT compare** `~/.claude/agents/core-rules.md` or `~/.claude/agents/disciplines/` — those belong to another org plugin's namespace — not managed by this plugin.

**Category B — Uncommitted changes in the Claude repo working tree:**
```bash
git -C "{claude-repo}" diff --name-only HEAD -- Plugins/{ORG_ID}/
git -C "{claude-repo}" ls-files --others --exclude-standard -- Plugins/{ORG_ID}/
```

#### Step 5c: Report and Offer PR

**No drift in any category:** update report line to `✓ Infrastructure in sync` — done.

**Drift detected:** present grouped summary and ask:

```
── Infrastructure drift detected ─────────────────────────────────────────

── Infrastructure files (core-rules / disciplines) ──────────────────────
  • agents/core-rules.md — N lines changed
  • agents/disciplines/development.md — modified

── Uncommitted repo changes ──────────────────────────────────────────────
  • Plugins/{ORG_ID}/skills/start-session/SKILL.md — modified

Total: {N} file(s) pending

Submit as pull request to {GITHUB_ORG}/Claude? (yes/no)
```

- **No:** update report line to `  ⚠ {N} infrastructure file(s) pending — answer yes at next start-session or end-session to submit.`
- **Yes:** delegate to `/{ORG_ID}:publish-agent-infrastructure-changes`.

---

## Agent Memory File Templates

### Architect Memory Files

**ARCHITECTURE.md:**
```markdown
# {PROJECT_NAME} Architecture

**Last Updated:** {DATE}
**Framework:** {FRAMEWORK}

---

## Module Map

(document project module structure, dependencies, and data flow)

## Key Abstractions

(core interfaces, base classes, patterns)

## Dependency Graph

(which modules depend on which — direction matters)
```

**PATTERNS.md:**
```markdown
# {PROJECT_NAME} Patterns & Conventions

**Last Updated:** {DATE}

---

(project-specific patterns — Tech-Lead reads when evaluating architectural fit)
```

**DECISIONS.md:**
```markdown
# {PROJECT_NAME} Architecture Decisions

**Last Updated:** {DATE}

---

## Format

### [D-001] Decision Title
- **Date:** YYYY-MM-DD
- **Context:** what prompted this decision
- **Decision:** what was decided
- **Consequences:** what follows
- **Status:** ACTIVE | SUPERSEDED by [D-NNN]
```

### Tracker Memory Files

**STATE.md:**
```markdown
# {PROJECT_NAME} State

**Last Updated:** {DATE}
**Current Milestone:** Sprint 0 — Project Setup
**Test Baseline:** 0 tests

---

## What's Complete

- Project initialized via /{ORG_ID}:start-session

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
- **Blocks:** what is blocked
- **Owner:** who can unblock
- **Description:** details
```

**BACKLOG.md:**
```markdown
# {PROJECT_NAME} Feature Backlog

**Last Updated:** {DATE}

---

## Priority Queue

(add features as identified)

## Parking Lot

(feature-level ideas not yet evaluated — run through tech-lead before promoting)
(for mid-task idea stubs created by agents during execution, see Plans/ in the project root)
```

### Specialist Memory Files

**DOMAIN.md:**
```markdown
# {PROJECT_NAME} Domain Knowledge

**Last Updated:** {DATE}

---

## Business Rules

(domain rules, invariants, constraints)

## Data Model

(key entities, relationships, meanings)
```

**API.md:**
```markdown
# {PROJECT_NAME} API Surface

**Last Updated:** {DATE}

---

## Endpoints

(API endpoints, contracts, versioning)

## Breaking Change Log

(track breaking changes with date and migration notes)
```

**INTEGRATIONS.md:**
```markdown
# {PROJECT_NAME} External Integrations

**Last Updated:** {DATE}

---

(external services, auth methods, rate limits, known quirks)
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

### Brand Memory Files

**BRAND.md:**
```markdown
# {PROJECT_NAME} Brand Guidelines

**Last Updated:** {DATE}

---

## Colors

<!-- Format: Name: #hexvalue — used by end-session for session summary branding -->
<!-- Primary: #hexvalue -->
<!-- Accent: #hexvalue -->

(document project/client brand colors as learned)

## Typography

(fonts, weights, sizes)

## Voice & Tone

(writing style, terminology preferences)

## Component Patterns

(recurring UI patterns and their brand treatment)
```

**AUDIT.md:**
```markdown
# {PROJECT_NAME} Brand Audit Log

**Last Updated:** {DATE}

---

(audit entries added as UX reviews are performed)
```
