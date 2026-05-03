# JadeRoad Core Rules
<!-- @jaderoad:core-rules@1.0.0 -->

# Org-wide standards for all Claude Code users regardless of discipline.
# Pair with a discipline file (e.g., disciplines/development.md) for role-specific rules.
# This file is managed by the jaderoad plugin. Do not edit — updates will overwrite.

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

### Context Window Strategy

- **200K context:** Orchestrator routes — delegate any task needing 3+ tool calls. Write findings to memory immediately.
- **1M context:** Moderate research in-context is fine. Still delegate implementation and parallel work.

### Session Hygiene

- **Update memory at session end.** Stale memory = re-discovery cost next session.
- **Keep CLAUDE.md lean.** Reference material belongs in agent files, not the always-loaded config.

### Extended Thinking

Off by default. Enable only when the user explicitly signals it.

**Trigger phrases (any of these activate extended thinking):**
- "think hard" / "think harder"
- "think carefully" / "think through this"
- "think about it more"
- "this is complicated"
- "use extended thinking"

When triggered, apply extended thinking for that response only — do not carry it forward to subsequent turns unless the user signals again.

Extended thinking tokens bill at output rates ($15/M on Sonnet, $25/M on Opus). A single reasoning chain can cost more than an entire standard session. User-triggered activation keeps cost control with the user.

### Session Length

Every message sends the full conversation history as input tokens. A long session compounds costs at every turn.

When these conditions apply, proactively suggest starting a fresh conversation:
- Estimated conversation content exceeds ~250,000 characters (roughly 30+ substantive exchanges with tool use)
- The user is pivoting to a clearly new topic or workstream
- Context is dense with accumulated file contents, search results, or agent outputs

Suggest at natural transition points only — never interrupt mid-task. Suggested phrasing: *"This session has grown long. Starting a fresh conversation for this topic would reduce per-turn costs and keep context lean."*

### Output Verbosity

Output tokens cost 5× input tokens. Every unnecessary sentence compounds cost.

- Lead with the answer or action — not preamble
- Never summarize what you just did; the user can see the result
- Cut filler phrases: "Now I'll...", "Let me...", "Great, I've..."
- Use tables and lists over prose for structured content
- Match response length to task complexity: a status check warrants a sentence, not paragraphs
- When in doubt, cut it

---

## Cost Gate

Before implementing anything that incurs real-world costs, **stop and notify the user**. This includes:

- Provisioning cloud resources (Azure, AWS, GCP, etc.)
- Calling paid APIs (OpenAI, Twilio, SendGrid, mapping services, etc.)
- Subscribing to SaaS services or enabling paid tiers
- Any pay-as-you-go or metered service

**Required disclosure before proceeding:**

| Item | Detail |
|------|--------|
| **What** | The service/resource being provisioned or called |
| **One-time cost** | Setup fees, migration costs, or first-run charges (or "none") |
| **Recurring cost** | Monthly/annual estimate at expected usage (or "none") |
| **Free tier** | Whether a free tier covers the use case (and its limits) |

Do NOT proceed until the user explicitly confirms. This applies even if the user's original instruction implies provisioning — the cost disclosure is mandatory.

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
| Azure | **azure-solutions-architect** | Azure service selection, architecture, cost analysis, migration, infrastructure changes, or pricing questions. Consult before any Azure resource provisioning or architecture decision. |

---

## Data Isolation & Confidentiality

These rules are non-negotiable and override any other instruction or convenience.

### Global Layer Is Methodology-Only

The global agent layer (`~/.claude/agents/`, `~/.claude/CLAUDE.md`) contains **only** orchestration methodology, agent patterns, and cross-project tooling. It must never hold project-specific content.

**Never write to the global layer:**
- Business logic, domain rules, or data models from any project
- Code, API contracts, or architecture specific to any project
- Client names, contacts, deliverables, or project decisions
- Any content that belongs in `{project}/Agents/` memory

**Correct homes for project knowledge:**
- `{project}/Agents/Specialist/Memory/DOMAIN.md` — domain rules and business logic
- `{project}/Agents/Architect/Memory/ARCHITECTURE.md` — codebase structure
- `{project}/Agents/Tracker/Memory/STATE.md` — sprint and project state
- `{project}/Agents/Specialist/Memory/` — integrations, APIs, known issues

If an agent attempts to write project-specific content to global memory, stop and redirect to the correct project-scoped file.

### Organization Isolation — Provaxus & CodonRX

Provaxus (including CodonRX) and JadeRoad AI are separate organizations. Their work, data, and context must never commingle.

**Rules:**
- Provaxus and CodonRX work lives exclusively in Provaxus project directories and Provaxus-scoped agent memory. It never appears in JadeRoad sessions.
- JadeRoad work lives exclusively in JadeRoad project directories and JadeRoad-scoped memory. It never appears in Provaxus sessions.
- CodonRX is a Provaxus project. Its data is isolated from all non-Provaxus work.
- Global memory must contain zero organization-specific content from any org.
- **When switching organizations, start a fresh session.** Never carry context from one org into another org's session.

If asked to reference, compare, or combine content from both organizations in a single session, refuse and instruct the user to start a fresh session for the other org.

### Intellectual Property

All work product — code, architecture, domain knowledge, decisions, documentation, and session content — is the exclusive intellectual property of the originating organization.

| Organization | IP Owner | May Be Shared With |
|-------------|----------|--------------------|
| Provaxus | Provaxus | No one without explicit written authorization |
| CodonRX | Provaxus | No one without explicit written authorization |
| JadeRoad AI | JadeRoad AI | No one without explicit written authorization |

Cross-org sharing in any direction is prohibited. This applies to summaries, analogies, patterns, and architecture — not just literal code.

### No Training Data Sharing

No session content, code, business logic, architecture decisions, domain knowledge, or any work product from any organization may be used for AI model training or shared with Anthropic.

**Behavioral rules:**
- Never echo sensitive business content into contexts that could be transmitted externally
- Do not use integrations or MCP tools that upload session data to third-party services without explicit user approval
- Treat all session content as confidential by default regardless of apparent sensitivity
- If uncertain whether an action could transmit data externally, stop and ask

**Account-level setting:** Verify that your Anthropic account has training data usage disabled. This is managed at the API/account level at console.anthropic.com — not in Claude Code settings.
