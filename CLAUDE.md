# Architecture Hub

This repository is a knowledge base and working environment for a solution architect
who designs cloud architectures across GCP, AWS, and Azure for multiple client accounts.
Your job in this repo: read engagement inputs, build understanding, generate architecture
diagrams and supporting material, and continuously learn from feedback and deal outcomes.

## Repository layout

- `standards/` — MY conventions: diagram style guide, reusable cloud patterns. Cross-account by design.
- `templates/` — skeleton files used when creating new accounts/projects. Never edit during normal work; copy from them.
- `accounts/<account>/` — one folder per client account. Contains account profile, reviewer preferences, deal history, and a nested CLAUDE.md.
- `accounts/<account>/projects/<project>/` — one folder per engagement: `inputs/` (raw client docs), `learning.md` (distilled understanding), `diagrams/`, `decisions.md`.
- `.claude/skills/` — workflows: /intake, /diagram, /feedback, /outcome, /retro, /new-project.

## Hard rules — confidentiality

1. **Never cross-reference client material.** When working inside `accounts/X/`, do not read, quote,
   or draw on anything under any other `accounts/Y/` folder. Not for inspiration, not for comparison,
   not "just to check how we did it before." The ONLY cross-account knowledge you may use is the
   generalized, client-anonymized content in `standards/`.
2. Anything promoted from an account into `standards/` (via /outcome or /retro) must first be
   stripped of client names, system names, figures, and any identifying detail.
3. Never include client names from one account in files of another account. Ever.

## Working conventions

- Before generating or editing any diagram, read `standards/diagram-style-guide.md` AND the
  current account's `reviewer-preferences.md`. Preferences override the global style guide
  when they conflict — the reviewer is always right for their own account.
- Diagram sources are text and live in git: prefer `.drawio` XML for client deliverables and
  Mermaid (```mermaid blocks or .mmd files) for quick/internal diagrams. Always keep the
  editable source next to any exported image, same base filename.
- Every generated artifact goes under the relevant project folder — never at repo root.
- When you learn something durable during a session (a correction, a preference, an assumption
  confirmed or broken), write it to the right file immediately: reviewer taste →
  `reviewer-preferences.md`; facts about the client/engagement → `learning.md`; decisions and
  open questions → `decisions.md`. Do not let learning die in the chat transcript.
- Distinguish three kinds of feedback and file them separately:
  - **Taste** ("I prefer left-to-right flow") → account `reviewer-preferences.md`
  - **Fact** ("we actually use Cloud SQL, not Spanner") → project `learning.md`
  - **Universal improvement** ("always show trust boundaries") → propose adding to `standards/diagram-style-guide.md`, ask me first
- Keep `learning.md` files following the structure in `templates/learning.md` so every project reads the same way.
- Ask before overwriting a diagram a reviewer has already seen; prefer creating a new version
  (e.g. `-v2`) so we can diff against what was reviewed.

## Tone and output

- I'm a senior architect: be direct, technical, and concise. No hand-holding.
- When assumptions are needed to produce a diagram, make them, list them explicitly in
  `decisions.md` under "Assumptions", and flag the risky ones.
