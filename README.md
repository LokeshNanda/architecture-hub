# Architecture Hub — Claude Code starter kit

A self-evolving knowledge base for a multi-cloud solution architect: accounts → projects,
document intake, diagram generation, and feedback loops that make every output more
personalized over time. Designed to be driven almost entirely by Claude Code.

## Get started (5 minutes)

1. Unzip this folder anywhere, then:
   ```bash
   cd architecture-hub
   git init && git add -A && git commit -m "architecture hub v0"
   claude
   ```
2. Skim `accounts/acme-corp-EXAMPLE/` to see what a "learned" account looks like
   (filled reviewer preferences, deal history, how-to-win notes). Delete it when done.
3. Create your first real account — just say:
   > new client <Name>, we're pitching <project>
   (the `new-project` skill scaffolds everything from `templates/`)
4. Drop the client's PDFs/DOCX/PPTX into the new project's `inputs/` and say `/intake`.
5. Say `/diagram` — it reads learning.md + your style guide + the account's reviewer
   preferences before drawing.

## The daily loop

| You say                  | What happens                                                                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| "new client X…"          | account + project scaffolded from templates                                                                                                    |
| `/intake`                | subagents read inputs/, distill into `learning.md`                                                                                             |
| `/diagram`               | style-guide- and reviewer-aware diagrams into `diagrams/`                                                                                      |
| "Ravi's feedback: …"     | `/feedback` fixes the diagram AND files the lesson (taste → reviewer-preferences, fact → learning.md, universal → proposes style-guide change) |
| "we won / lost because…" | `/outcome` logs deal-history + how-to-win lessons                                                                                              |
| `/retro` (monthly)       | scans all accounts, proposes anonymized updates to `standards/`                                                                                |

## How the personalization actually works

- Nested `CLAUDE.md` files: the root one carries your rules; each account's one auto-loads
  when you work in that folder — so account context is always on without re-explaining.
- `reviewer-preferences.md` is read by `/diagram` **before** drawing, so corrections a
  reviewer made once are applied automatically forever after.
- `standards/` only ever receives client-anonymized, generalized lessons — and only with
  your approval (via `/feedback` UNIVERSAL items or `/retro` proposals).

## First-week tips

- Seed `standards/<cloud>-patterns.md`: ask Claude to write up the 3-5 architectures you
  draw most often per cloud. This pays off immediately in `/diagram` quality.
- Edit `standards/diagram-style-guide.md` to match your actual taste — it ships with
  sensible defaults, but it's meant to be yours.
- Commit often. Git history is your diagram version trail ("what changed since the
  version Ravi reviewed?").

## Confidentiality

The root `CLAUDE.md` forbids cross-account reading (except anonymized `standards/`).
Still: check your firm's policy on client documents in AI tooling, and consider one repo
per client if any account demands hard isolation — the structure works the same.
