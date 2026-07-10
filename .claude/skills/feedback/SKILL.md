---
name: feedback
description: Capture reviewer or client feedback on diagrams and deliverables, apply the fixes, and — critically — convert the feedback into durable learning. Use whenever the user pastes review comments, says "the client said…", "Ravi wants…", "feedback from the review call", "they didn't like…", or reports any reaction to a diagram, proposal, or document.
---

# Feedback — fix the artifact AND make the system smarter

Every piece of feedback does two jobs: it fixes today's artifact, and it should prevent
the same correction from ever being needed again. Skipping the second job is how the
knowledge base stops evolving — so never skip it.

## Steps

1. **Parse the feedback** into individual items. For each item, classify:
   - **TASTE** — how this reviewer likes things presented (layout direction, icon usage,
     level of detail, color, naming style, format). → `reviewer-preferences.md` for this
     account, under the named reviewer if known.
   - **FACT** — a correction about the client's reality ("we're on Cloud SQL not Spanner",
     "latency SLA is 5min not 15"). → project `learning.md` (and fix any diagrams built on
     the wrong fact).
   - **DESIGN** — a decision or direction change ("drop the streaming path", "add DR region").
     → `decisions.md` decision log + revise the diagram.
   - **UNIVERSAL** — feedback that would improve ALL your diagrams, any client.
     → Propose an addition to `standards/diagram-style-guide.md` and ask the user to
     confirm before writing it (globals affect everything; taste masquerades as universal).
   If classification is genuinely ambiguous, ask — one good question beats filing feedback
   in the wrong place forever.
2. **Generalize before writing.** Store the rule, not the incident.
   - Incident: "Ravi told us to remove the Pub/Sub icon on the v1 data flow."
   - Rule: "Ravi: no product icons on data-flow views; labeled boxes only."
   Include a dated one-line example under the rule for provenance.
3. **Apply the fixes.** Run the /diagram skill's revision path: create a new version
   (`-v2`) since the reviewer saw the old one, and list exactly what changed in response
   to which comment — reviewers notice when their feedback is visibly honored.
4. **Report back**: what was fixed, what was learned and where it was filed, and any
   feedback you chose NOT to act on (with reasoning) so the user can override.

## Guardrails

- Reviewer preferences are per-account and usually per-person. If a different reviewer
  appears on the same account, start a new section for them — don't average people.
- Contradictory preference from the same reviewer? The newest wins; keep the old one
  struck through with a date so the history is visible.
- Never generalize a client FACT into `standards/` — facts are confidential; only
  presentation/technique learnings can graduate to global.
