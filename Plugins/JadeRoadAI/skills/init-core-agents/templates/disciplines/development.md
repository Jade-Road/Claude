# Development Discipline
<!-- @jaderoad:discipline-development@1.0.0 -->

# Rules for software development with Claude Code.
# Imported alongside core-rules.md in developer CLAUDE.md files.
# This file is managed by the jaderoad plugin. Do not edit — updates will overwrite.

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

---

## Quality Gates

1. **TDD** — No production code ships without tests. Non-negotiable.
2. **QA Validation** — Every deliverable passes QA before handoff. Zero critical issues.
3. **Independent QA** — Builder does not validate own work. **Skip for:** config-only, single-line, doc, infra-only changes, or changes covered by existing passing tests.
4. **Governance** — Consult relevant SME agent before new features, migrations, or architectural changes. **Skip for:** single-file changes covered by an existing decision, or pure bug fixes with no architectural surface area.

---

## Global SME Subagents (On-Demand)

These are proper Claude Code custom subagents with enforced model/tools. Claude auto-delegates based on task context.

| Subagent | Model | When It's Used |
|----------|-------|----------------|
| **tech-lead** | sonnet | Before new features, architectural changes, scope decisions. Reads project Architect memory. |
| **qa-lead** | sonnet | After implementation, before deploy, security/test/stability review. Can run tests. |
| **ux-lead** | sonnet | UI work, design review, accessibility audits, responsiveness checks. |
| **azure-solutions-architect** | sonnet | Azure service selection, architecture, cost analysis, infrastructure changes. |

## Project Subagents (Per-Project)

Created by `init-project-team`, prefixed with project codename (e.g., `nova-tracker`).

| Subagent | Model | When It's Used |
|----------|-------|----------------|
| **{codename}-tracker** | haiku | Session start, planning work, checking state/blockers. |
| **{codename}-specialist** | sonnet | Business logic, domain, API, integrations, dependency health. |
| **{codename}-brand-manager** | haiku | Project/client brand guidelines and enforcement. |

**Note:** Project Architect is not a subagent — Tech-Lead reads its memory files directly (`{project}/Agents/Architect/Memory/`).

---

## Subagent Patterns

- Scope tightly: specific task, specific files, acceptance criteria.
- Parallelize when tasks are independent.
- One-shot briefs — no round-trips.
- Structured returns: STATUS, FILES_MODIFIED, TESTS_ADDED, TESTS_PASSING, NOTES.
- QA returns: VERDICT, QA_SCORE, CRITICAL_ISSUES, WARNINGS.
