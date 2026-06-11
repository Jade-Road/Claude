# Marketing Discipline
<!-- @{ORG_ID}:discipline-marketing@1.0.0 -->

# Rules for marketing operations work with Claude Code.
# Imported alongside core-rules.md in marketing-focused project CLAUDE.md files.
# This file is managed by this org plugin. Do not edit — updates will overwrite.

---

## Activated Subagents

This discipline activates the following global subagents for projects that import it:

| Subagent | Role |
|----------|------|
| **{MARKETING_AGENT}** | Primary expert — campaigns, analytics, brand, market segmentation, CMP, SEO, content |
| **ux-lead** | Accessibility, layout, contrast, responsiveness for marketing surfaces |
| **qa-lead** | Compliance review for consent/CMP/tracking changes; pre-launch QA on analytics events |

### Primary Delegation: {MARKETING_AGENT}

For any marketing-domain question, **consult `{MARKETING_AGENT}` first**. This includes:

- Campaign strategy, keyword research, bid management (Google Ads)
- Analytics interpretation, funnel analysis, conversion optimization (GA4/GTM)
- Brand voice, messaging, copy, claims, tone
- Market-specific recommendations (regulatory differences, audience differences)
- CMP/consent strategy and compliance
- SEO, landing-page architecture, content strategy
- Schema.org markup, meta tags, structured data

The SME starts every consultation by reading `Agents/{ORG_NAME}MarketingSME/Memory/STRATEGY.md` (when project memory is available) — that file is the source of truth for positioning, channel strategy, and current campaign state.

---

## Marketing Quality Gates

1. **CMP compliance** — Any tracking/cookie change must pass through the SME's compliance review. No tracking ships without confirmed CMP categorization.
2. **Brand consistency** — Copy and claims must match approved brand language in `Agents/{ORG_NAME}MarketingSME/Memory/BRAND.md`. Unverified claims or competitive comparisons require SME approval.
3. **Analytics schema integrity** — Event names, custom dimensions, and parameter schemas are tracked in `Agents/{ORG_NAME}MarketingSME/Memory/ANALYTICS.md`. Adding/changing events requires SME review to avoid breaking funnel reports.
4. **Independent QA** — Anyone other than the change author validates landing-page changes, ad creative changes, and analytics implementations before launch.

---

## Decision Boundaries

The SME owns:
- Campaign strategy, audience targeting, bid strategy
- Brand voice, messaging, copy approval
- Market-specific decisions
- Analytics event taxonomy and reporting requirements
- CMP categorization and consent UX
- SEO strategy and content priorities

The SME does NOT own:
- Code implementation of tracking (defer to development team / **tech-lead**)
- Visual design system, accessibility specifics (defer to **ux-lead**)
- Build/deploy infrastructure (defer to **qa-lead** for pre-launch gates)

---

## Working Patterns

- **Always read STRATEGY.md first** when invoking the SME — it's the canonical reference and answers most "what's the priority?" questions before they get asked.
- **Convert relative dates** (e.g., "next quarter") to absolute dates when saving to memory — campaign timelines are time-sensitive and shift fast.
- **Cite the source** — When the SME makes a recommendation, it should reference which memory file (BRAND, MARKETS, ADS, ANALYTICS, CMP, DIGITAL_PRESENCE, STRATEGY) backed the answer, so the reader can dig deeper.

---

## Return Format (for SME consultations)

```
VERDICT: [recommendation]
DATA_BASIS: [which memory files / external sources informed this]
MARKET_CONTEXT: [relevant market context]
RISKS: [list or "none"]
NEXT_STEPS: [actionable items, prioritized]
```
