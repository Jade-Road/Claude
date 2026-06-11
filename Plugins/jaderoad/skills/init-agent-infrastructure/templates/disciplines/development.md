# Development Discipline
<!-- @{ORG_ID}:discipline-development@1.0.0 -->

# Rules for software development with Claude Code.
# Imported alongside core-rules.md in developer CLAUDE.md files.
# This file is managed by this org plugin. Do not edit — updates will overwrite.

---

## Activated Subagents

This discipline activates the following global subagents for projects that import it:

| Subagent | Role |
|----------|------|
| **tech-lead** | Architecture, scope decisions, feature evaluation |
| **qa-lead** | Security, stability, test quality |
| **ux-lead** | Accessibility, usability, responsiveness |
| **{MARKETING_AGENT}** | Marketing concerns in this org's code projects (analytics tracking, ad pixels, consent, marketing landing pages, SEO metadata) |

### Cross-Domain Delegation

Development work in this org's projects often touches marketing surfaces. **Delegate to `{MARKETING_AGENT}` before** changing or adding any of the following:

- Analytics tracking — GTM containers, GA4 events, custom dimensions, eCommerce schema
- Advertising pixels — Google Ads conversion tags, Meta Pixel, LinkedIn Insight, etc.
- Consent management — CMP configuration, consent banner, cookie categories
- Marketing-facing pages — landing pages, campaign pages, SEO metadata, schema.org markup
- Customer-facing copy with brand claims, product positioning, or campaign messaging
- Market routing, geolocation logic, or market-specific feature flags

The SME owns the *what* and *why*; engineering owns the *how*. Confirm intent and event schema with the SME, then implement.

---

## TDD Requirements

<tdd_rules>
  <rule id="1" severity="critical">Never write production code without a failing test first.</rule>
  <rule id="2" severity="critical">Tests must fail for the right reason before you make them pass.</rule>
  <rule id="3" severity="critical">One logical assertion per test. Named for what they verify.</rule>
  <rule id="4" severity="high">Tests are first-class code — same quality as production.</rule>
  <rule id="5" severity="high">Bug found? Write failing test BEFORE fixing.</rule>
  <rule id="6" severity="high">Refactoring only when tests are green.</rule>
</tdd_rules>

### Coverage Tiers

| Component Type | Minimum |
|----------------|---------|
| Core Business Logic / API Contracts | 100% |
| Domain Models | 95% |
| Feature Modules | 85% |
| UI / Browser Automation | 70% |

### Agentic Test Strategy

When agents produce code or artifacts, test deterministic behavior only — not non-deterministic agent choices:

| Test This (Deterministic) | Don't Test This (Non-Deterministic) |
|--------------------------|-------------------------------------|
| Tool/function correctness — given input X, output is Y | Agent word choice or phrasing in generated text |
| Schema conformance — output matches expected structure | Exploration order — which files the agent reads first |
| Artifact quality — generated files are valid, parseable, complete | Subjective quality — whether prose is "good enough" |
| Invariants — security rules, data constraints, business rules | Implementation strategy — how the agent chose to solve it |
| Regression — previously working behavior still works | Token efficiency — how many attempts the agent needed |

Source test fixtures from real data (sanitized), not synthetic mocks. Real data preserves edge cases that synthetic data hides.

### Acceptance Test Stubs First

Before writing implementation, generate test stubs named after acceptance criteria that throw `NotImplementedException`. Tests verify requirements, not what was incidentally built.

**Pattern:** `{Persona}_{CanAction}_{WhenCondition}()` — stub throws, implementation fills it in.

### Run-All Script Requirement

Every project must have a single entry-point test script (`run-all.ps1` or `run-all.sh`):
- No flags to skip test categories — always runs the full suite
- Test count is monotonically increasing — baseline can only go UP, never down
- Wire to CI gate with coverage thresholds enforced by Coverlet or equivalent

---

## Quality Gates

1. **TDD** — No production code ships without tests. Non-negotiable.
2. **QA Validation** — Every deliverable passes QA before handoff. Zero critical issues.
3. **Independent QA** — Builder does not validate own work. (Skip conditions in `core-rules.md` Quality Gate Proportionality.)
4. **Governance** — Consult relevant SME agent before new features, migrations, or architectural changes. (Skip conditions in `core-rules.md` Quality Gate Proportionality.)

---

## Global SME Subagents (On-Demand)

These are proper Claude Code custom subagents with enforced model/tools. Claude auto-delegates based on task context.

| Subagent | Model | When It's Used |
|----------|-------|----------------|
| **tech-lead** | sonnet | Before new features, architectural changes, scope decisions. Reads project Architect memory. |
| **qa-lead** | sonnet | After implementation, before deploy, security/test/stability review. Can run tests. |
| **ux-lead** | sonnet | UI work, design review, accessibility audits, responsiveness checks. |
| **{MARKETING_AGENT}** | sonnet | {ORG_NAME} marketing: analytics, advertising, brand voice, consent, SEO. |

## Project Subagents (Per-Project)

Created by `start-session`, prefixed with project codename (e.g., `nova-specialist`).

| Subagent | Model | When It's Used |
|----------|-------|----------------|
| **{codename}-specialist** | sonnet | Business logic, domain, API, integrations, dependency health. |

**Note:** Architect, Tracker, and Brand are memory-only — no subagent is spawned for these roles. Tech-Lead reads `Agents/Architect/Memory/` directly, Operator reads `Agents/Tracker/Memory/` directly, UX-Lead reads `Agents/Brand/Memory/` directly.

---

## Subagent Patterns

- Scope tightly: specific task, specific files, acceptance criteria.
- Parallelize when tasks are independent.
- One-shot briefs — no round-trips.
- Structured returns: STATUS, FILES_MODIFIED, TESTS_ADDED, TESTS_PASSING, NOTES.
- QA returns: VERDICT, QA_SCORE, CRITICAL_ISSUES, WARNINGS.

---

## Development Plans

Mid-task ideas cause plan drift when chased immediately. Park them, finish the current task, review at session boundaries.

### Lifecycle

- **Park** — when an idea surfaces mid-task, write `Plans/{Name}.md` with what's known (unknowns as TBD). Return to current work immediately. No exceptions, regardless of how compelling the idea seems.
- **Review** — at session start or end, assess effort vs. value across open plans. Activate 2–3 at most.
- **Archive** — when work ships, move plan to `Plans/Completed/`. A clean `Plans/` directory is signal.

### Rule

Any agent with a mid-task idea creates a plan and returns to current work. Derailing to chase the idea is not permitted regardless of apparent urgency.

Use `Plans/Plan_Template.md` — the Test Plan section precedes Implementation Steps by design.

---

## Security Baseline

Not theoretical — each item has documented real-world incidents (2026):

- **`.env` exclusion:** Add `.env*` to `ignorePatterns` in `.claude/settings.json`. Claude Code auto-loads `.env` files into context — API keys travel to Anthropic. Move production secrets to a secrets manager.
- **Review `.claude/` before opening unfamiliar repos.** Malicious hooks in cloned repos can achieve RCE. Three CVEs documented this attack chain in early 2026.
- **Pre-commit secret scanning.** Run gitleaks or trufflehog on all commits. AI-generated code leaks secrets at 2× baseline rate (GitGuardian 2026).
- **SAST on generated code.** Run Semgrep or SonarQube before committing AI-generated code. Veracode measured a 45% OWASP Top 10 failure rate in AI-generated code samples.
- **Version floor:** Claude Code 2.1.116 is the minimum safe version (patches silent sandbox bypass and deny-rule wrapper escapes). Stay current.

## Dev Server File Lock

When a build or file operation fails with MSB3027/MSB3021 errors indicating a running dev server is locking DLLs, **offer to kill it** rather than working around the lock or silently failing.

Offer: "The dev server (PID XXXXX) is locking the build output. Want me to kill it so the build can complete? Run `! Stop-Process -Id XXXXX` to stop it."

**Never silently accept the lock.** File lock errors from the dev server are always recoverable — the right fix is to stop the process, not to skip the verification step.

---

## Commit Process

**VCS Detection:** Check for `.git` (Git) and `.hg` (Mercurial) artifacts in the project root. Use whichever is present. If both exist, ask the developer before proceeding.

**Owner:** Operator executes. QA-Lead must clear quality gates before step 3.

**Steps (execute in order, stop on failure):**

1. **Build** — Run the project build. Must succeed before continuing.
2. **Changelog** — Update `docs/changelog.html`: prepend a new entry at the top with today's date and a summary of changes. Apply project branding (colors, typography) when brand guidelines are available.
3. **Local commit** — Commit all changes with a descriptive message reflecting the work done.
4. **Pull** — Pull latest from origin.
5. **Merge** — Merge pulled changes. If conflicts exist, surface them to the developer and wait for resolution before continuing.
6. **Build after merge** — Run the build again. Must succeed before continuing.
7. **Commit merge result** — If the merge or post-merge build produced changes, commit them locally.
8. **Push** — Push to origin.
