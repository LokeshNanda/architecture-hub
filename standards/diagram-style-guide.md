# Diagram Style Guide (global)

My defaults for all architecture diagrams. Account `reviewer-preferences.md` files
override anything here for that account. Updated over time by /feedback (with approval)
and /retro. Log changes in `standards/CHANGELOG.md`.

## Layout
- Flow direction: left → right for data flows; top → bottom for layered/logical views.
- Group by trust boundary first (cloud account/project/subscription, VPC/VNet), then by tier.
- External actors (users, partners, SaaS) on the far left or top, outside all boundaries.
- Max ~15 components per view; beyond that, split into layered views (context → logical → detail).

## Notation
- draw.io with official provider shape libraries for client deliverables; Mermaid for drafts.
- Solid arrows = data flow (label with protocol/mechanism when non-obvious: HTTPS, gRPC,
  CDC, batch). Dashed arrows = control plane / auth / async triggers.
- Every boundary box gets a label in its top-left corner.
- Legend on every deliverable diagram. Title block: project, version, date, status.

## Content
- Always show: network/trust boundaries, data flow direction, where data is at rest.
- Show when in scope: IAM/identity flows, region/zone placement (HA/DR), encryption points.
- Name real services (Cloud Run, EKS, Synapse), not vague boxes ("compute") — unless the
  view is deliberately cloud-agnostic for an early pitch.

## Versioning
- Filenames: `<view>-v<N>.<ext>` (e.g. `logical-architecture-v2.drawio`).
- Never overwrite a version a reviewer has seen; add a new version and note the deltas.

## Anti-patterns (things I've been burned by)
- Icon soup: dozens of product icons with no boundaries or flow labels.
- One mega-diagram trying to show network + data + IAM at once.
- Diagrams that contradict learning.md — the doc of record wins; fix one or the other.
