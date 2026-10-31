---
id: 67807e99-2f33-42a8-a32b-deee8f978fd8
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/SUPPORT_DESK.md

Source: DevPlusInc/devplus-platform/docs/SUPPORT_DESK.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/SUPPORT_DESK.md).

# Support desk (HubSpot Service Hub)

DevPlus users report issues, request features and ask questions from inside the
product (**Support** in the sidebar's Help group, or **Contact support** in the
top bar's Help menu; `/support`). Every request becomes a
ticket in DevPlus Inc.'s own HubSpot Service Hub portal, and the user follows it
from the same page: current stage, replies from support, and their own
comments. This document is the operator runbook: how the pieces fit, how to
turn the desk on for an environment, and how support agents should work
tickets so that users see their replies.

## How it works

| Piece | Where | What it does |
|---|---|---|
| `HubSpotClient` | `devplus-backend/services/hubspot_client.go` | CRM v3/v4 HTTP client (objects, properties, pipelines, associations, search) authenticated with a **private-app access token**. Bounded retries on 429/5xx. |
| `SupportService` | `devplus-backend/services/support_service.go` | The tenant boundary. The token is platform-wide (one portal serves every tenant), so the service adds `devplus_tenant = <tenant>` to every search, re-checks it on every read, and only lets non-admins see tickets whose `devplus_requester_email` is their own. |
| Routes | `GET /api/v1/support/status`, `GET/POST /api/v1/support/tickets`, `GET /api/v1/support/tickets/{id}`, `POST /api/v1/support/tickets/{id}/comments`, admin `GET /api/v1/admin/support/tickets` | Cedar actions `GetSupportStatus`, `ListSupportTickets`, `CreateSupportTicket`, `GetSupportTicket`, `CommentSupportTicket` (granted to every user by the system "User - No Role" policy) and `ListAllSupportTickets` (admin only, in the global forbid list). |
| Read markers | Mongo `support_ticket_reads` (per tenant DB) | `{userId, ticketId, lastSeenAt}`; a ticket is **unread** when its last user-visible activity (stage change, agent reply, portal comment, close) is newer than the user's last view. |
| SPA | `devplus-frontend/app/support/**`, `components/support/**`, `lib/support.ts`, `lib/hooks/useSupport.ts` | List (Open / Closed / All, unread dot, admin "all requests in this tenant" switch), new-request dialog, ticket page with timeline + comment box. The sidebar entry and the Help menu's **Contact support** item (`components/layout/navigation.ts`, `HelpMenu.tsx`) only appear when `GET /support/status` reports `configured: true`. |

### What lands in HubSpot

On first use the backend creates (idempotently) a **DevPlus** property group on
tickets and companies with these custom properties:

| Object | Property | Purpose |
|---|---|---|
| ticket | `devplus_tenant` | Tenant (namespace) the request came from. **The scoping key.** |
| ticket | `devplus_requester_email` | The DevPlus user who filed it. |
| ticket | `devplus_context` | Requester, tenant, page URL, app version, browser (captured automatically). |
| ticket | `devplus_last_portal_comment_at` | Stamped when the user comments from DevPlus. Build a HubSpot view / workflow on it to get notified. |
| company | `devplus_tenant` | One company per tenant (created on demand, name = tenant). |

Each ticket is filed into the configured pipeline's first **open** stage with
the built-in HubSpot fields `subject`, `content`, `hs_ticket_priority`
(LOW/MEDIUM/HIGH/URGENT) and `hs_ticket_category` (`issue` →
PRODUCT_ISSUE, `feature_request` → FEATURE_REQUEST, `question` →
GENERAL_INQUIRY, `billing` → BILLING_ISSUE), and is associated with the
requester's **contact** (upserted by email) and the tenant's **company**.

### How replies reach the user (read this if you work tickets)

- **Stage changes** are shown automatically (from the ticket's
  `hs_pipeline_stage` history). Open/closed comes from each stage's
  `ticketState` in HubSpot, never from the label. A stage whose label reads
  like "Pending customer response" / "Waiting on contact" is highlighted to
  the user as *waiting for your reply*.
- **To reply to the user, reply from the ticket by email** (Help Desk /
  the ticket record → Email). HubSpot logs the email on the ticket; DevPlus
  shows every logged email (`EMAIL` = your reply, `INCOMING_EMAIL` /
  `FORWARDED_EMAIL` = the user's email answer) on the ticket page, and the user
  also gets the email itself.
- **Notes are internal.** A note you add on the ticket is *not* shown to the
  user. The only notes DevPlus shows are the ones it created itself: a user's
  portal comment becomes a note that starts with the header
  `DevPlus portal comment · <name> <email> · tenant <tenant>`, so you can see
  who wrote it and from which tenant. Never start an internal note with that
  header.
- The user sees the HubSpot **ticket id** as `#<id>`; it is the same number as
  "Ticket ID" in HubSpot, so it is safe to quote in either direction.

## Turning the desk on

### 1. Create the HubSpot app (once)

New HubSpot portals no longer offer the classic **Private Apps** page (Settings
→ Integrations → Private Apps now shows only *Legacy apps*, empty on a fresh
account). The replacement is a **developer-platform project** whose app is
`distribution: private` + `auth.type: static`: it yields the same kind of
single-account access token. The project is checked in at
[`hubspot-app/`](../hubspot-app/README.md); a Super Admin of the DevPlus Inc.
portal uploads it:

```sh
npm install -g @hubspot/cli@latest
hs account auth            # browser flow: creates a personal access key for DevPlus Inc.
cd devplus-platform/hubspot-app
hs project upload          # creates project "devplus-support-desk" and builds it
```

Then in HubSpot: **Development → Projects → devplus-support-desk → Overview →
Project Components → DevPlus support desk → Distribution tab → Install now**,
approve the scopes, click **Show** next to the access token and copy it.

If you would rather let the CLI scaffold the project, run
`hs project create --distribution private --auth static --project-base app`
and paste the `requiredScopes` list from `hubspot-app/src/app/app-hsmeta.json`
into the generated file before uploading. Scopes:

| Scope | Why |
|---|---|
| `oauth` | HubSpot's base app scope (listed first in `app-hsmeta.json`) |
| `tickets` | create / read / update tickets, their pipelines and the `devplus_*` ticket properties |
| `crm.objects.contacts.read`, `crm.objects.contacts.write` | upsert the requester's contact; HubSpot's Notes API (portal comments) is gated on these |
| `crm.objects.companies.read`, `crm.objects.companies.write` | one company per tenant |
| `crm.schemas.companies.write` | create the `devplus_tenant` company property (first use only, but keep it) |
| `sales-email-read` | show your email replies on the ticket page |
| `crm.objects.owners.read` | show the agent's name on replies |

Editing scopes later = edit `app-hsmeta.json`, `hs project upload`, then
re-install from the Distribution tab (the token stays the same).

### 2. Store the token in Key Vault (per environment)

The vaults are `kv-devplus-dev`, `kv-devplus-stage`, `kv-devplus-prod` (WARP
on, `az login` to the matching subscription):

```sh
az keyvault secret set --vault-name kv-devplus-dev --name hubspot-access-token --value 'pat-na2-…'
```

Use the same token for every environment, or a second private app for
prod: the desk does not care, tickets are tagged with the tenant either way.

### 3. Enable it per environment (core-infra)

`devplus-core-infra` (`hubspot.tf`) creates the Reflector-mirrored source
Secret `hubspot-access-token-shared` (namespace `external-secrets`, key
`access-token`) when `hubspot_enabled = true` in the environment's tfvars
(`environments/<env>.tfvars`); dev, stage and prod all set it to `true`
today. `deploy.yml` loads the Key Vault secret as an *optional* mapping
(`?TF_VAR_hubspot_access_token=hubspot-access-token`), so a vault without the
token keeps applying green only while `hubspot_enabled` is `false`: with the
flag on and no token, the Secret's precondition fails the apply. For a new
environment, store the token (step 2) before flipping the flag.
`hubspot_reflector_namespace_pattern` (default `.*`) limits which namespaces
receive the mirror. Reflector mirrors the Secret into those namespaces within
seconds; verify with:

```sh
kubectl -n <tenant> get secret hubspot-access-token-shared
```

The DevPlus chart's default `hubspot.secretName` already points at that
mirror and the backend's `secretKeyRef` is `optional: true`, so nothing
else changes for Terraform-provisioned or in-product-provisioned tenants.
Backends that were running before the mirror appeared need a restart to pick
up the env var (Reloader does that where it is enabled; otherwise
`kubectl -n <tenant> rollout restart deployment/backend`).

**Single-owner rule:** nothing else may ever create a Secret named
`hubspot-access-token-shared` in any namespace (Reflector mirrors keep the
source name and adopt/overwrite same-named Secrets).

### 4. Tilt tenants (all modes)

`tilt up` in every mode (`local`, `atlas`, `aks`) runs the `hubspot-secret`
resource, which copies `hubspot-access-token` from `kv-devplus-dev` into the
tenant namespace as Secret `hubspot-secret` and sets
`hubspot.secretName=hubspot-secret`, exactly like the per-tenant
`github-install-state` copy: a dev tenant never waits on the core-infra
rollout of the mirror, and the two Secret names never collide. If the vault
has no such secret the resource logs a notice and the desk is simply off.

### 5. Pick the pipeline

`hubspot.ticketPipeline` (chart value, default **`Customer Support`**) names
the HubSpot ticket pipeline by **label or id**; `HUBSPOT_TICKET_PIPELINE` on
the backend. New tickets go to that pipeline's first stage whose
`ticketState` is OPEN. If the pipeline has no open stage, or the name does not
match any pipeline, every support request fails with 502 and the backend logs
`hubspot ticket pipeline "…" not found`.

## Configuration reference

| Env var (backend) | Chart value | Meaning |
|---|---|---|
| `HUBSPOT_ACCESS_TOKEN` | `hubspot.secretName` / `hubspot.secretKey` | Private-app token. Empty = desk off (`/support/status` → `configured: false`, sidebar entry hidden, every other route 503). |
| `HUBSPOT_TICKET_PIPELINE` | `hubspot.ticketPipeline` | Pipeline label or id. The chart renders it whenever `hubspot.secretName` is set (default `Customer Support`); an empty value falls back to HubSpot's default pipeline `0`. |
| `HUBSPOT_API_BASE_URL` | — | Override for tests only (default `https://api.hubapi.com`). |

Limits (backend, per pod): 20 new requests and 60 comments per user per hour
(429), subject ≤ 200 chars, description ≤ 20,000, comment ≤ 10,000; list pages
≤ 100 (HubSpot search cap).

## Troubleshooting

| Symptom | First check |
|---|---|
| No **Support** entry in the sidebar | `curl -b … https://<tenant>/api/v1/support/status` → `configured:false` means the backend has no token: is `hubspot-access-token-shared` (or `hubspot-secret` under Tilt) present in the namespace, and was the backend restarted after it appeared? |
| Every support call is **502** "temporarily unavailable" | Backend log `SupportHandler.*: HubSpot failure: …`. `401`/`403` from HubSpot = wrong token or missing scope (the category names the scope); `hubspot ticket pipeline "…" not found` = fix `hubspot.ticketPipeline`. |
| A user's request is filed but not visible in the list | The search index lags a few seconds after creation; the SPA navigates to the ticket page (a direct read) for that reason. Persisting? Check `devplus_tenant` / `devplus_requester_email` on the ticket in HubSpot. |
| Agent reply not visible to the user | It was added as a **note** (internal) instead of an **email** reply, or the app lacks `sales-email-read`. |
| User comment not visible to the agent | Notes appear on the ticket timeline in HubSpot; check `devplus_last_portal_comment_at` was stamped. Build a view sorted by it. |
| Property creation fails with 403 | Missing `tickets` (ticket properties) or `crm.schemas.companies.write` (company property); after adding a scope, upload and re-install the app. |
