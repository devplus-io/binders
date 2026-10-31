---
id: 0b65ec41-e678-4834-8e1d-9fa30bbad664
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/hubspot-app/README.md

Source: DevPlusInc/devplus-platform/hubspot-app/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, hubspot-app/README.md).

# DevPlus support desk — HubSpot app definition

The HubSpot app the DevPlus backend authenticates with for the in-product
support desk (see [`docs/SUPPORT_DESK.md`](../docs/SUPPORT_DESK.md)). HubSpot
no longer lets new portals create classic "private apps"; the replacement is a
developer-platform **project** whose app has `distribution: private` and
`auth.type: static`. This directory is that project, checked in so the app
definition (name, scopes) is versioned instead of clicked together.

Upload it into the DevPlus Inc. portal (once, by a Super Admin):

```sh
npm install -g @hubspot/cli@latest
hs account auth            # browser flow: create a personal access key for DevPlus Inc.
cd devplus-platform/hubspot-app
hs project upload          # creates project "devplus-support-desk" and builds it
```

Then in HubSpot: **Development → Projects → devplus-support-desk → Overview →
Project Components → DevPlus support desk → Distribution tab → Install now**,
approve the scopes, then **Show** and copy the access token. Store it as Key
Vault secret `hubspot-access-token` (see the runbook). Re-run `hs project
upload` after editing this file (e.g. adding a scope); re-install from the
Distribution tab when scopes change.

Notes:
- `oauth` is in `requiredScopes` because HubSpot adds that scope group to
  every app anyway and warns on upload when it is missing; it grants nothing
  DevPlus uses.
- Do NOT add `redirectUrls` under `auth`: the CLI's schema for static auth
  rejects it (the error it prints, "allowedValues: oauth, no_api_access", is
  misleading; it is the extra property that fails the static branch).
  `redirectUrls` is an OAuth-only field.
- Notes are created/read under the contacts scopes (HubSpot's Notes API
  requires `crm.objects.contacts.read/write`); logged emails need
  `sales-email-read`; `tickets` covers tickets and their custom properties;
  `crm.schemas.companies.write` lets the backend add `devplus_tenant` to
  companies on first use.
- `crm.objects.owners.read` resolves the agent on a logged email to a display
  name in the user's timeline (falls back to "DevPlus Support").
- Do not put the token in this directory or anywhere in git.
