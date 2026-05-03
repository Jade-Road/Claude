---
name: session-summary
description: Use when the user asks to summarize the session, generate a session report, or requests a session summary HTML document. Produces a branded session summary.
version: 1.0.0
---

# Session Summary

Generate a branded, standalone HTML document summarizing the current session's accomplishments.

**Brand customization:** The HTML template uses `--brand-*` CSS variables. To override the default JadeRoad palette, add a `brand-overrides` block to your CLAUDE.md or a project brand file:

```css
/* In your brand config — paste into the <style> block or reference via a brand file */
:root {
  --brand-primary: #your-color;
  --brand-heading: #your-heading-color;
  /* ... etc */
}
```

## Usage
```
/jaderoad:session-summary                           # Auto-generate title from session context
/jaderoad:session-summary Dashboard UX Overhaul     # Explicit title
```

## Arguments

Parse from `$ARGUMENTS`:
- **title** (optional): Brief session title. If omitted, infer from the major work done in the session.

---

## Instructions

### Step 1: Gather Session Context

Review the conversation history to identify:
1. **What was accomplished** — features built, bugs fixed, deployments, decisions made
2. **Key metrics** — tests added, files changed, deploys, imports, etc.
3. **Blockers resolved** — what was broken and how it was fixed
4. **What's next** — open items, follow-ups, handoff notes
5. **Agent name** — which agent ran this session (check the project CLAUDE.md for the orchestrator name; fall back to "Claude" if none defined)
6. **Company name** — check `~/.claude/CLAUDE.md` for a `<company>` or `<owner>` tag; default to "JadeRoad AI" if none found

### Step 2: Determine Filename

Format: `{Agent}_{yyyyMMdd}_{hhmmtt}_{BriefTitle}.html`

Examples:
- `Operator_20260323_0415pm_DashboardUXOverhaul.html`
- `Claude_20260404_0130pm_BugFixSprint.html`

Use the current date/time. Title should be PascalCase, no spaces, max ~30 chars.

### Step 3: Generate HTML

Write a standalone HTML file using the template below. The file must be:
- **Standalone** — inline CSS, no external stylesheets (Google Fonts link is OK)
- **Branded** — uses `--brand-*` CSS variables (JadeRoad defaults, user-overridable)
- **Print-friendly** — clean at 8.5x11 print
- **Comprehensive but scannable** — use tables, callout boxes, and bullet lists

### Step 4: Output Location

Default output path: `docs/session-summaries/{filename}`

**Create both `docs/` and `docs/session-summaries/` directories if they don't exist.**

---

## HTML Template

Use this exact structure. Fill in sections based on session context. Omit sections that don't apply (e.g., skip "Deploys" if nothing was deployed).

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{SESSION_TITLE}} — Session Summary</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <style>
        /* === Brand variables — override these to match your brand === */
        :root {
            --brand-primary: #3d8b6d;
            --brand-secondary: #81827e;
            --brand-heading: #1f5c45;
            --brand-text: #3a3a4a;
            --brand-dark: #2a5c4a;
            --brand-warm: #5a3929;
            --brand-gold: #ddb027;
            --brand-error: #a95b28;
            --brand-link: #4c7887;
            --brand-success: #418c7d;
            --brand-bg-soft: #e2ede8;
            --brand-bg-warm: #f0eecd;
            --brand-border: #e6e6e6;
            --brand-hover: #d4e9e2;
        }
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Inter', Arial, sans-serif; color: var(--brand-text); line-height: 1.7; background: #f8f9fa; }
        header { background: linear-gradient(135deg, var(--brand-heading) 0%, var(--brand-dark) 100%); color: white; padding: 2.5rem 2rem; }
        header h1 { font-size: 2rem; font-weight: 700; margin-bottom: 0.25rem; }
        header .subtitle { font-size: 1rem; font-weight: 300; opacity: 0.9; }
        header .meta { display: flex; gap: 1.5rem; margin-top: 1rem; font-size: 0.85rem; opacity: 0.8; flex-wrap: wrap; }
        header .meta span { display: flex; align-items: center; gap: 0.35rem; }
        .container { max-width: 900px; margin: 0 auto; padding: 2rem; }
        h2 { color: var(--brand-heading); font-size: 1.5rem; font-weight: 600; margin: 2rem 0 0.75rem; padding-bottom: 0.4rem; border-bottom: 3px solid var(--brand-primary); }
        h3 { color: var(--brand-heading); font-size: 1.1rem; font-weight: 500; margin: 1.25rem 0 0.5rem; }
        p, li { margin-bottom: 0.5rem; font-size: 0.95rem; }
        ul { padding-left: 1.25rem; }

        /* Stats bar */
        .stats { display: flex; gap: 1rem; flex-wrap: wrap; margin: 1.5rem 0; }
        .stat { background: white; border: 1px solid var(--brand-border); border-radius: 10px; padding: 1rem 1.25rem; text-align: center; flex: 1; min-width: 120px; }
        .stat .number { font-size: 1.75rem; font-weight: 700; color: var(--brand-heading); }
        .stat .label { font-size: 0.75rem; color: var(--brand-secondary); text-transform: uppercase; letter-spacing: 0.05em; margin-top: 0.15rem; }

        /* Callouts */
        .callout { border-left: 4px solid; padding: 0.75rem 1rem; margin: 1rem 0; border-radius: 0 6px 6px 0; font-size: 0.9rem; }
        .callout-green { border-color: var(--brand-primary); background: #ecf6f1; }
        .callout-gold { border-color: var(--brand-gold); background: #fdf8e8; }
        .callout-orange { border-color: var(--brand-error); background: #fdf0e8; }
        .callout-teal { border-color: var(--brand-heading); background: #e8f4f0; }
        .callout strong { font-weight: 600; }

        /* Tables */
        table { width: 100%; border-collapse: collapse; margin: 0.75rem 0 1.25rem; font-size: 0.9rem; }
        thead { background: var(--brand-bg-soft); }
        th { text-align: left; padding: 0.6rem 0.75rem; font-size: 0.8rem; font-weight: 600; color: var(--brand-heading); border-bottom: 2px solid var(--brand-primary); text-transform: uppercase; letter-spacing: 0.03em; }
        td { padding: 0.6rem 0.75rem; border-bottom: 1px solid var(--brand-border); }
        tr:hover { background: var(--brand-hover); }

        /* Badges */
        .badge { display: inline-block; padding: 0.1rem 0.5rem; border-radius: 1rem; font-size: 0.7rem; font-weight: 700; }
        .badge-green { background: #dcfce7; color: #166534; }
        .badge-blue { background: #dbeafe; color: #1e40af; }
        .badge-gold { background: #fef3c7; color: #92400e; }
        .badge-red { background: #fecaca; color: #991b1b; }

        /* File lists */
        .file-path { font-family: 'Courier New', monospace; font-size: 0.85rem; color: var(--brand-heading); }

        /* Footer */
        footer { text-align: center; padding: 1.5rem; color: var(--brand-secondary); font-size: 0.8rem; border-top: 1px solid var(--brand-border); margin-top: 2rem; }

        @media print {
            body { background: white; }
            header { padding: 1.5rem; }
            .container { padding: 1rem; }
            .stat { break-inside: avoid; }
        }
        @media (max-width: 600px) {
            header h1 { font-size: 1.5rem; }
            .stats { flex-direction: column; }
        }
    </style>
</head>
<body>
    <header>
        <h1>{{SESSION_TITLE}}</h1>
        <p class="subtitle">{{ONE_LINE_DESCRIPTION}}</p>
        <div class="meta">
            <span>&#128197; {{DATE}}</span>
            <span>&#129302; {{AGENT_NAME}}</span>
            <span>&#128204; {{PROJECT_NAME}}</span>
            <!-- Add more meta items as relevant: duration, deploy target, version, etc. -->
        </div>
    </header>

    <div class="container">

        <!-- KEY METRICS — always include -->
        <h2>Key Metrics</h2>
        <div class="stats">
            <!-- Include 3-6 stats. Common ones: -->
            <!-- Tests Added, Files Changed, Deploys, Domains Imported, Bugs Fixed, etc. -->
            <div class="stat"><div class="number">{{N}}</div><div class="label">Tests Added</div></div>
            <div class="stat"><div class="number">{{N}}</div><div class="label">Files Changed</div></div>
            <!-- ... -->
        </div>

        <!-- WHAT WAS ACCOMPLISHED — always include, this is the main section -->
        <h2>What Was Accomplished</h2>
        <!-- Use subsections (h3), tables, callouts, and bullet lists as appropriate -->
        <!-- Group by feature/workstream, not chronologically -->

        <!-- BLOCKERS RESOLVED — include if any were fixed -->
        <h2>Blockers Resolved</h2>
        <table>
            <thead><tr><th>Issue</th><th>Root Cause</th><th>Fix</th></tr></thead>
            <tbody>
                <!-- One row per blocker -->
            </tbody>
        </table>

        <!-- DECISIONS MADE — include if architectural/strategic decisions were made -->
        <h2>Decisions</h2>

        <!-- OPEN ITEMS / NEXT STEPS — include if there's follow-up work -->
        <h2>Next Steps</h2>
        <ul>
            <!-- Prioritized list of what's next -->
        </ul>

        <!-- FILES CHANGED — optional, include for significant sessions -->
        <h2>Files Changed</h2>
        <!-- Use badge-green for NEW, badge-blue for REWRITTEN, badge-gold for MODIFIED -->

    </div>

    <footer>
        <strong>{{COMPANY_NAME}}</strong> — {{PROJECT_NAME}}<br>
        Session Summary — {{DATE}} — {{AGENT_NAME}}
    </footer>
</body>
</html>
```

---

## Section Guidelines

### Key Metrics
- Always include. Pick 3-6 numbers that tell the story.
- Common: Tests Added, Files Changed, Deploys, Assessments Run, Bugs Fixed, Domains Imported

### What Was Accomplished
- **Lead with the outcome, not the process.** "Dashboard rewritten with activity feed" not "Read files, wrote code, ran tests"
- Group by workstream/feature, not chronologically
- Use callout boxes for important wins: `.callout-green` for shipped features, `.callout-gold` for insights, `.callout-orange` for incidents resolved
- Include before/after comparisons where impactful

### Blockers Resolved
- Table format: Issue | Root Cause | Fix
- Only include real blockers that were diagnosed and fixed, not routine work

### Decisions
- Only include if strategic/architectural decisions were made
- Include rationale (why, not just what)

### Next Steps
- Bullet list, prioritized
- Include owner if known
- Flag anything time-sensitive

### Files Changed
- Optional — include for sessions that touch many files
- Use badges: `badge-green` (NEW), `badge-blue` (REWRITTEN), `badge-gold` (MODIFIED)
- Show file paths in monospace

---

## After Writing

1. Confirm the file was written to `docs/session-summaries/`
2. Report the filename to the user
3. If this is the first session summary, note that `docs/session-summaries/` is a new directory
