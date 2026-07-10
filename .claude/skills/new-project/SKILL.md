---
name: new-project
description: Scaffold a new account and/or project folder from templates. Use whenever the user mentions a new client, new account, new engagement, new pitch, or new project ("new client Contoso", "we're pitching X to Acme", "set up a project for…"), or drops documents for a client that doesn't have a folder yet.
---

# New project — scaffold accounts and engagements

## Steps

1. **Slugify names**: lowercase, hyphens (e.g. "Acme Corp" → `acme-corp`;
   "Data Platform Modernization" → `data-platform-modernization`).
2. **Account exists?** If `accounts/<account>/` is missing, create it and copy from
   `templates/`:
   - `templates/account-CLAUDE.md` → `accounts/<account>/CLAUDE.md` (fill in the account name)
   - `templates/account-profile.md` → `account-profile.md`
   - `templates/reviewer-preferences.md` → `reviewer-preferences.md`
   - `templates/deal-history.md` → `deal-history.md`
   Ask the user 3 quick seed questions and fill what you can: industry & what they do;
   existing client or new pitch; known stakeholders/reviewers.
3. **Create the project**: `accounts/<account>/projects/<project>/` with `inputs/`,
   `diagrams/`, plus `learning.md` and `decisions.md` copied from templates. Set
   `STATUS: active` and today's date in learning.md.
4. **Hand off**: tell the user to drop client documents into the new `inputs/` folder,
   and offer to run /intake as soon as files land.

Keep this fast — scaffolding should take seconds, not a conversation.
