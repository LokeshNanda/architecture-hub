---
name: intake
description: Process client input documents (PDF, DOCX, PPTX, MD, XLSX, emails, meeting notes) for an engagement and build or update the project's learning.md. Use this whenever the user drops new files into a project's inputs/ folder, says "process the inputs", "read what the client sent", "new RFP came in", mentions a requirements document, transcript, or discovery notes — or whenever a project has files in inputs/ that learning.md doesn't yet reflect.
---

# Intake — turn raw client documents into engagement understanding

Goal: after this skill runs, `learning.md` is the single trustworthy summary of the
engagement, and nobody needs to re-open the raw inputs for day-to-day work.

## Steps

1. **Locate the project.** Confirm which `accounts/<account>/projects/<project>/` you're
   working in. If ambiguous, ask — never guess between clients.
2. **Inventory `inputs/`.** List files with dates. Compare against the "Sources processed"
   table at the bottom of `learning.md` (if it exists) and process only new/changed files.
3. **Read the documents.** Reading many/large documents is noisy — delegate to the
   `document-reader` subagent (one per large document or batch) so the main context stays
   clean. Instruct each subagent to return a structured extraction, not a raw dump:
   requirements, constraints, current-state systems, stakeholders, numbers/SLAs, risks,
   open questions, and any diagram-relevant detail (regions, data flows, integration points).
4. **Merge into `learning.md`** using the exact structure from `templates/learning.md`.
   Rules for merging:
   - New info fills gaps; conflicting info goes to "Open questions & conflicts" with both
     sources cited — do not silently pick a winner.
   - Preserve anything a human wrote by hand unless the new source clearly supersedes it.
   - Keep every claim traceable: suffix with the source, e.g. `(RFP §3.2)` or `(kickoff call 2026-07-02)`.
5. **Update the "Sources processed" table** (filename, date processed, one-line gist).
6. **Update `account-profile.md`** at the account level if you learned durable facts about
   the client themselves (industry shifts, org changes, strategic direction) — not
   project-specific detail.
7. **Report back** in chat with: 5-bullet executive summary, the top 3 open questions worth
   asking the client, and whether there's now enough to run /diagram.

## Quality bar

- learning.md must be readable in under 5 minutes by someone who never saw the inputs.
- Requirements are testable statements, not vibes ("ingest 2TB/day with <15min latency",
  not "handle big data fast").
- Never invent facts to fill template sections; write "Unknown — ask client" instead.
  A confident blank beats a plausible hallucination in a document I'll pitch from.
