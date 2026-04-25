## Capture Session Summary

Generate a structured summary of the current session for handoff, memory, and continuity.

### Instructions

Review the full conversation history in this session and produce a summary with the following sections. Be thorough but concise — this is a reference document, not a narrative.

### Output Format

```markdown
# Session Summary — {{DATE}}

## Session Context
- **Session name:** (if renamed, include it)
- **Project(s) touched:** (list projects/repos worked in)
- **Duration:** (approximate, based on first and last messages)

## Work Summary
Provide a 2-4 sentence overview of what was accomplished this session.

## Decisions Made
List every decision made during this session, including:
- What was decided
- Why (rationale or constraint)
- Who decided (user vs agent recommendation)

Format as a numbered list. If no decisions were made, state "None."

## Dev Backlog
List any work items identified but NOT completed this session. These are future tasks that surfaced during the work. Include:
- Description of the work item
- Context for why it matters
- Priority if discussed (otherwise mark as "unranked")

Format as a bulleted list. If none, state "None."

## Action Items
List concrete next steps that should happen in a future session. Include:
- [ ] The action to take
- Owner (user or agent)
- Urgency if known

Format as a checkbox list. If none, state "None."

## Memory Requests
List any requests the user made to save, update, or remove memories during this session. For each:
- What was requested
- Whether it was completed (saved/updated/removed) or still pending
- The memory file path if applicable

If no memory requests were made, state "None."

## Files Modified
List all files created, modified, or deleted this session with a one-line description of the change.

## Open Questions
List any unresolved questions or ambiguities that were raised but not settled.
If none, state "None."
```

### Guidelines

1. **Be factual.** Summarize what happened, not what was planned. If something was discussed but not done, it goes in Backlog or Action Items, not Work Summary.
2. **Capture decisions explicitly.** Even small decisions (e.g., "chose S1 over F0 tier") matter for continuity.
3. **Don't editorialize.** No "great progress" or "productive session" — just facts.
4. **Include all file paths.** Every file touched should be traceable.
5. **Date-stamp everything.** Use absolute dates, not relative ("today", "yesterday").
6. **If the session is short or trivial**, still produce the full template — just mark sections as "None."

### After Generating

Ask the user:
1. "Want me to save this to a session log file?" — If yes, save to `~/.claude/sessions/{{YYYY-MM-DD}}-{{session-name}}.md`
2. "Any items here that should be saved to memory?" — If yes, create or update the appropriate memory files.
3. "Any backlog items to add to a project tracker?" — If yes, update the relevant project's agent memory.
