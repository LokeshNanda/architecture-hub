---
name: retro
description: Periodic cross-account retrospective that evolves the global knowledge base. Scans all accounts' reviewer preferences, feedback history, and deal outcomes for recurring patterns and proposes updates to standards/ (style guide + cloud pattern files). Use when the user says "run a retro", "what have we learned", "update the standards", "evolve the knowledge base", or roughly monthly / after every few closed deals.
---

# Retro — make the knowledge base evolve on its own

This is the ONE skill allowed to read across all accounts — but only to extract
anonymized patterns. Nothing client-identifying may appear in its outputs.

## Steps

1. **Scan** every account's `reviewer-preferences.md`, `deal-history.md`, and each
   project's `decisions.md`. Large scan → use subagents per account, each returning an
   anonymized pattern summary (recurring preferences, recurring loss/win reasons,
   recurring design corrections). This both keeps context clean and enforces the
   anonymization boundary structurally.
2. **Find patterns, not incidents.** A signal counts when it appears in ≥2 independent
   accounts, or in 1 account but tied to a WON/LOST outcome. Examples of what to look for:
   - The same taste preference appearing across reviewers → candidate global style rule.
   - The same architecture pattern repeatedly praised or repeatedly corrected →
     update the relevant `standards/<cloud>-patterns.md`.
   - Recurring loss reasons → candidate change to how solutions are pitched/diagrammed.
3. **Propose, don't apply.** Present a numbered list of proposed changes to `standards/`
   files, each with: the anonymized evidence ("seen in 3 accounts", "correlated with 2
   losses"), the exact edit, and the risk of adopting it. The user approves per item.
4. **Apply approved changes** and append a dated entry to `standards/CHANGELOG.md`
   (create it if missing) so the evolution of the standards is auditable.
5. **Flag decay**: preferences or patterns not referenced in 12+ months — suggest
   confirming or archiving them so the knowledge base doesn't fossilize.

## Guardrails

- Client names, system names, and figures NEVER leave their account folder. If evidence
  can't be stated anonymously, it doesn't graduate.
- One reviewer's strong taste is not a global rule, no matter how strongly worded.
