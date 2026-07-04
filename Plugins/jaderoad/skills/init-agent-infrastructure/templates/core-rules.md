# {ORG_NAME} Core Rules
<!-- @{ORG_ID}:core-rules@1.0.0 -->

# Org-wide standards for all Claude Code users regardless of discipline.
# Pair with a discipline file (e.g., disciplines/development.md) for role-specific rules.
# This file is managed by this org plugin. Do not edit — updates will overwrite.

---

## Principle: Tools Before AI

When a deterministic tool or process can achieve similar or better results than an AI call, use the tool. AI is for judgment, synthesis, and creativity — not for tasks that scripts, linters, formatters, or direct tool calls handle reliably.

Examples: Use Grep/Glob/Read instead of spawning research agents. Use linters instead of asking AI to check formatting. Use git log instead of asking AI to summarize history.

---

## Token Efficiency

### Model Selection

| Task Type | Correct Model | Never Use |
|-----------|--------------|-----------|
| Routine coordination, delegation, status | sonnet (orchestrator default) | opus |
| Complex planning, multi-domain tradeoffs, risk calls | opus (escalated) | haiku |
| Implementation, review, architecture | sonnet | opus |
| Pass/fail, lookups, validation, research | haiku | sonnet |
| Boilerplate from a clear template | haiku | — |
| Read-only research (any scope) | haiku or direct tools | opus |

**Never use opus for implementation or research. No exceptions.**

### Tiered Orchestration

The orchestrator runs on **sonnet** by default. Opus is reserved for escalation — not the baseline.

**Sonnet orchestrator handles:**
- Clear task delegation (known files, known agents)
- Single-domain work with an obvious SME to delegate to
- Status checks, memory updates, session bookkeeping
- Straightforward multi-step work where the plan is obvious

**Escalate to an Opus agent when:**
- Requirements are ambiguous and need interpretation before work begins
- Multiple domains conflict (e.g., cost vs. architecture vs. timeline)
- Risk assessment before irreversible or high-blast-radius actions
- Scope decisions where getting it wrong wastes significant effort
- Novel architectural decisions with no established pattern

**How to escalate:** Spawn an Opus agent with the decision context. Keep the prompt focused on the judgment call — don't send implementation work to Opus.

```
Agent(model: "opus", prompt: "Decision needed: [context]. Options: [A, B, C]. Constraints: [X, Y]. Return: recommended option + reasoning in <200 words.")
```

The Opus agent returns a decision. The Sonnet orchestrator executes it.

### Delegation Rules

1. **Direct tools first.** Grep/Glob/Read are zero-overhead. Only spawn an agent when the task needs 4+ queries or cross-referencing.
2. **≤3 operations → do it in context.** Never spawn an agent for a single tool call.
3. **File lists, not goals.** Tell agents exactly which files to work with. Vague goals cause over-exploration.
4. **Parallelize independent work.** Don't run sequentially what can run concurrently.
5. **Default to in-context.** Spawn only when parallel, isolated, or context-polluting.
6. **One-shot briefs.** Give agents everything upfront. Round-trips waste tokens.
7. **Stop early.** If the answer emerges mid-task, don't let work continue for its own sake.
8. **Don't re-read after editing.** Edit/Write errors on failure — verification reads waste tokens.
9. **Subagent returns are fields, not prose.** Key:value pairs. Max 200 tokens status, 500 implementation.
10. **Answer from memory when sufficient.** If a question is answerable from a memory file, answer it. Don't re-read source files to verify what memory already states — unless the memory is flagged as potentially stale.

### Context Window Strategy

Large context windows are a **capability ceiling, not a budget to fill**. Input tokens bill on every turn — a 500K-token conversation costs 500K input tokens on the next message, and every message after. Lean context is always cheaper regardless of available window size.

- **Prefer compaction and new conversations** over accumulating context. Starting fresh after a topic shift costs less than carrying a large history forward.
- **Delegate context-polluting work** (large file reads, exhaustive search results) to subagents so results stay out of the orchestrator's window.
- **Write to memory immediately.** Findings that need to survive past this session belong in memory files, not conversation history.
- **Trigger a new conversation** when context exceeds ~250K characters or the user pivots to a new topic.
- **Read surgically.** Never read a whole file speculatively. Use `offset` + `limit` on Read. Use Grep to find the exact line range first, then read only that range.
- **Summarize before context fills.** If a session has been running long (5+ back-and-forth exchanges), proactively write a status summary to STATE.md and offer to continue in a fresh session with full context.
- **Context rot is measurable.** Retrieval accuracy degrades ~2% per 100K tokens — at 500K expect ~10% degradation; at 800K, ~16%. Write decisions at the point they are made, not at session end. Don't rely on context retrieval in sessions exceeding ~300K characters.

### Session Hygiene

- **Update memory at session end.** Stale memory = re-discovery cost next session.
- **Keep CLAUDE.md lean.** Reference material belongs in agent files, not the always-loaded config.

### Quality Gate Proportionality

Apply quality gates proportionally — don't run full governance on a one-line fix.

**Skip independent QA for:**
- Config-only changes (no logic)
- Single-line fixes
- Documentation edits
- Infrastructure-only commits with no application logic change
- Changes already covered by an existing passing test

**Skip governance/SME consultation when:**
- The change is confined to a single file
- Architecture memory already has a recorded decision covering this pattern
- It's a pure bug fix with no architectural surface area

---

## Quality Controls

- All deliverables must meet brand standards before handoff.
- Surface concerns and risks before execution, not after.
- When SME agents are available for a domain, consult them before making decisions in that domain.
- All documentation must pass linter with no CRITICAL/HIGH issues.

## Domain SME Delegation

The following domain experts must be consulted for work in their area. Auto-delegation handles this, but if you're working in-context, delegate explicitly.

| Domain | Subagent | When to Consult |
|--------|----------|-----------------|
| Org Marketing | **{MARKETING_AGENT}** | Campaign strategy, analytics, Google Ads, brand voice, market decisions, CMP/consent, SEO, content. Consult proactively for any marketing-facing work in this org's projects. |

## Permissions Configuration

When initializing project infrastructure, configure these pre-approved read permissions in the project's `.claude/settings.json` to prevent spurious permission prompts when discipline files are loaded:

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

These paths are org-managed read-only resources. `/{ORG_ID}:start-session` configures this automatically at project init. To apply to an existing project, run `/{ORG_ID}:set-project-disciplines`.

**Note on coexistence with other org plugins:** This org's infrastructure is isolated under `~/.claude/agents/{ORG_ID}/`. Do not edit or reference files belonging to other org plugins — each plugin owns its own namespace. Shared infrastructure (CI, deployment tooling) belongs in the infrastructure context, never embedded in org-specific plugin files.

---

## Mercurial: Destructive Action Gate

Projects in this organization use Mercurial for version control. Before running any of the following commands, confirm with the user — they are hard or impossible to reverse:

| Command | Risk |
|---------|------|
| `hg strip` | Permanently removes changesets from repo history |
| `hg rollback` | Undoes last transaction (commit/pull/import); works only once, no undo |
| `hg update --clean` / `hg up -C` | Silently discards all uncommitted working directory changes |
| `hg revert --all` / `hg revert -a` | Reverts entire working directory to last commit |
| `hg purge --all` | Deletes all untracked and ignored files |
| `hg amend` | Rewrites the current changeset hash — breaks anyone who pulled it |
| `hg histedit` | Interactive history rewriting; never use on pushed changesets |
| `hg rebase` | Rewrites changeset hashes — only safe on purely local changesets |
| `hg push --force` | Force-pushes despite creating new remote heads; can permanently branch public history |
| `hg phase --force --draft` / `--secret` | Demotes public commits, enabling history rewrites on already-pushed changesets |
| `hg prune` | Marks changesets obsolete (evolve extension); combined with `hg evolve` rewrites public history |

**Rule:** If the command appears in this table, stop and confirm before running it. If the user explicitly requests one of these, proceed — but state what will happen first.
