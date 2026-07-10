---
name: diagram
description: Generate or revise cloud architecture diagrams (GCP, AWS, Azure, hybrid) for a project, honoring the global style guide and the account's reviewer preferences. Use whenever the user asks for an architecture diagram, solution design, data flow, network topology, "draw this", "sketch the solution", "update the diagram", or after /intake reports there is enough understanding to design from.
---

# Diagram — produce reviewer-ready architecture diagrams

## Before drawing anything (non-negotiable order)

1. Read the project's `learning.md` — the diagram must reflect documented understanding,
   not generic reference architecture. If learning.md is missing or thin, say so and offer
   to run /intake first rather than inventing an architecture.
2. Read `standards/diagram-style-guide.md`.
3. Read the account's `reviewer-preferences.md`. **Preferences beat the style guide** —
   the person who reviews the diagram defines what "good" looks like for their account.
4. Read the relevant `standards/<cloud>-patterns.md` for reusable building blocks.
5. Check `diagrams/` for existing versions — revise, don't fork, unless the reviewer has
   already seen the current version (then create `-v2`, `-v3`, ... and note what changed).

## Choosing the format

- **Client deliverable / will be reviewed** → draw.io XML (`.drawio`). Use official cloud
  provider shape libraries (mxgraph styles like `shape=mxgraph.gcp2.*`, `mxgraph.aws4.*`,
  `mxgraph.azure.*`). Hand-tweakable afterward, which reviewers appreciate.
- **Quick / exploratory / for chat discussion** → Mermaid. Fast to iterate, fine to show
  inline, easy to convert to draw.io later.
- If reviewer-preferences.md names a format, that wins.

## Content rules

- Every component on the diagram must be justified by learning.md or an explicit,
  recorded assumption. Log assumptions in `decisions.md` under "Assumptions".
- Show what architects get grilled on: trust/network boundaries (VPCs, subnets, projects/
  subscriptions/accounts), data flow direction, protocols where non-obvious, IAM/identity
  where it's a design point, and region/zone placement when HA/DR is in scope.
- Layered views beat one mega-diagram. Default set for a full solution: (1) context /
  high-level, (2) logical architecture, (3) data flow, (4) network/security — but produce
  only what the engagement stage needs. A first pitch usually needs (1) and (2).
- Title block on every deliverable diagram: project name, version, date, author, status
  (DRAFT / FOR REVIEW / APPROVED).

## Output

- Save sources to `diagrams/` with descriptive names: `logical-architecture-v1.drawio`,
  `data-flow-v1.mmd`. Export/render a PNG or SVG alongside when possible, same basename.
- In chat, summarize: what views were produced, the key design decisions embedded in them,
  and the assumptions that most need client validation.
- Append design decisions worth remembering to `decisions.md`.
