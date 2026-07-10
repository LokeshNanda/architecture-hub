---
name: outcome
description: Record what happened to a deal, pitch, or engagement — won, lost, stalled, expanded — plus any account-level feedback, and extract lessons. Use whenever the user reports a deal result ("we won", "we lost to X", "client went silent", "they signed", "deal update"), relays account feedback after a pitch, or wants to log why an engagement ended the way it did.
---

# Outcome — close the loop on deals

## Steps

1. **Log the outcome** in the account's `deal-history.md`: date, project, outcome
   (WON / LOST / STALLED / EXPANDED / NO-DECISION), value if shared, competitor if known,
   and the stated + suspected reasons. Stated and suspected are different columns on
   purpose — clients rarely tell you the real reason, and both are worth keeping.
2. **Interview for lessons.** Ask the user 2-3 sharp questions if the "why" is thin:
   what did the client push back on, what did the winner do differently, would a different
   architecture/pricing/story have changed it? Don't interrogate — a couple of questions,
   then write.
3. **File the lessons:**
   - Account-specific ("this client is cost-anchored, lead with TCO") → `account-profile.md`
     under "How to win here".
   - Reviewer/stakeholder insight → `reviewer-preferences.md`.
   - Generalizable, client-anonymized lesson ("regulated-industry pitches need the
     compliance view in the first meeting") → propose for `standards/` (pattern files or
     style guide) and confirm with the user before writing.
4. **Update project status** in the project folder (a `STATUS: won/lost/...` line at the
   top of `learning.md`) so /retro can find closed engagements.
5. **Report back** with the entry as written and the lessons filed.

## Why this matters

Deal history is the only feedback signal that measures whether the diagrams and proposals
actually work — reviewer happiness is a proxy; signed deals are the ground truth. A rich
deal-history.md is what lets /retro evolve the standards in the direction of winning.
