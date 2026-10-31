---
id: b6a706ad-2658-4ff2-89a7-e3a8a3a5116e
binder: binder-mvvle32a-1
---
DevOps | devplus-hub-infra/docs/OWNERSHIP-AND-RUNBOOKS.md

Source: DevPlusInc/devplus-hub-infra/docs/OWNERSHIP-AND-RUNBOOKS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-hub-infra, docs/OWNERSHIP-AND-RUNBOOKS.md).

# Ownership, apply ordering & runbooks

How `devplus-hub-infra` and `devplus-core-infra` divide responsibility, the
order they must be applied in, and operational runbooks (secrets rotation,
migrations). Keep this in sync when the seams between the repos change.

## Who owns the shared singletons

Some Azure resources can only exist once per region/subscription, or must be
centrally shared. The **hub** owns all of them; spokes reference them.

| Resource | Owner | Notes |
|---|---|---|
| Private DNS zones (`privatelink.*`) | **hub** (`dns.tf`) | Linked to the hub VNet and every spoke, except the `*.azmk8s.io` zones (hub-only; core hands them to AKS as `private_dns_zone_id`). ACR zone removed (GHCR). |
| Network Watchers | **hub** (`network-watchers.tf`, `monitoring.tf`) | One per (subscription, region). Spokes attach flow logs via data source — they do **not** create watchers. |
| Log Analytics workspace + Sentinel | **hub** (`monitoring.tf`) | Single hardened, AAD-only, private workspace. All spoke diagnostics/flow logs/AKS Container Insights/App Insights ship here. Sentinel is onboarded **once** on it. Table plans are hub-owned too: `AKSAuditAdmin` is on Basic, while core's AKS diagnostic setting routes `kube-audit-admin` into it as a resource-specific table. |
| AMPLS (Azure Monitor Private Link) | **hub** (`ampls.tf`) | One per DNS namespace. |
| Central Key Vaults (per env) | **hub** (`keyvault.tf`) | One per env, region-less (`kv-devplus-{env}`), each in its own `devplus-keyvault-{env}` RG (dev/stage in their own subscriptions). |
| Encryption keys | **neither** (Microsoft-managed) | No CMK vaults or keys remain in either repo: hub `cmk.tf` was deleted in `3257d28` and core `cmk.tf` + `modules/region/cmk.tf` in core `c77c41e` (both 2026-09-20, Drata Test 254). Hub storage and cloudflared, and core storage and AI Foundry, use Microsoft-managed platform keys. Core ignores `disk_encryption_set_id` on AKS, so a cluster created before the switch keeps its Azure-side disk encryption set until it is rebuilt. |
| VNet peering (hub↔spoke) | **hub** (`peering.tf`) | Bidirectional, cross-subscription. |
| MongoDB Atlas project + clusters | **core** (`mongodb.tf`) | Hub only **adds** the runner NAT IP to the access list. |
| MongoDB Atlas IP access list | **both** (different entries) | core adds AKS NAT IPs; hub adds the runner NAT IP. |
| Defender for Cloud pricing plans | **core** (`defender.tf`), per env subscription | Subscription-level singletons — set by whichever repo runs in that sub. Core owns them today. |

## Apply ordering

The dependency runs both directions, so order matters for a new (env, region):

1. **hub** — DNS zones, central Key Vault, Network Watcher, hub workspace.
2. **core** — spoke VNet + subnets, Atlas project. (Needs hub DNS/KV/workspace/watcher to exist — they're consumed via data sources.)
3. **hub** — peering + Atlas IP access list. (Needs the spoke VNet + Atlas project.)
4. **core** — private endpoints, AKS, flow logs. (Needs peering + DNS links live.)

For a brand-new region with no spoke yet, pre-seed the Network Watcher with
`network_watcher_regions` in the hub (see README) to break the chicken-and-egg.

## Required deployer permissions (cross-subscription)

The prod deployment identity now also creates/manages resources in the dev and
stage subscriptions. It needs, in **dev** and **stage**:

- `Network Contributor` — cross-sub peering, DNS links, flow logs.
- `Contributor` on the `devplus-keyvault-{env}` resource group — to create the
  relocated Key Vault and approve its private endpoint connection.
- `Microsoft.Authorization/roleAssignments/write` (Owner or User Access
  Administrator) at the KV scope — Key Vault role assignments.
- `Monitoring Contributor` — diagnostic settings on cross-sub resources.

`devplus-cicd-infra` currently satisfies all of these by granting the prod CI/CD
service principal `Owner` on the dev and stage subscriptions
(`hub_prod_owner_dev` / `hub_prod_owner_stage`) as well as on prod.

## Central Key Vault names

Uniform, region-less, one per environment:

| env | vault | resource group / subscription |
|---|---|---|
| prod | `kv-devplus-prod` | `devplus-keyvault-prod` (hub == prod sub) |
| dev | `kv-devplus-dev` | `devplus-keyvault-dev` (dev sub) |
| stage | `kv-devplus-stage` | `devplus-keyvault-stage` (stage sub) |

The earlier `kv-devplus-{dev,stage,prod}-eus2` names were soft-deleted with purge
protection and could not be purged or reused until 2026-09-06 (no admin/Microsoft
override; that's purge protection working as designed). The region-less names
side-stepped that lock and remain the names in use.

## Completed migrations (historical)

One-time changes already applied — recorded for the audit trail, not repeatable
runbooks.

- **2026-06 — Prod Key Vault moved to its own RG.** `kv-devplus-prod` relocated
  from the shared hub RG (`devplus-hub-eus2`) into a dedicated
  `devplus-keyvault-prod` RG, uniform with dev/stage. Done via a
  same-subscription `az resource move` (vault name unchanged, so no CI repoint),
  then reconciled into state; the prod private endpoint, diagnostic setting, and
  KV role assignments were recreated against the new vault ID.
- **2026-06 — Central Key Vault consolidation.** dev/stage central vaults moved
  into their own subscriptions (`kv-devplus-dev`, `kv-devplus-stage`) so AI
  Foundry can reference a same-subscription vault; prod renamed to
  `kv-devplus-prod`. Region-less names were adopted because the prior `*-eus2`
  names are soft-deleted under purge protection until 2026-09-06. Core's per-env
  `KEYVAULT` variables and the hub `KEY_VAULT_NAME` were repointed accordingly.
- **2026-06 — Logging centralized to the hub.** Core stopped creating per-region
  Log Analytics workspaces; all spoke diagnostics, flow logs, AKS Container
  Insights, and App Insights now ship to the hub workspace, with Sentinel
  onboarded once there. The per-region workspaces were removed (their historical
  logs were not retained).
- **2026-08-15: Hub workspace retention raised to 365 days** (was 90) for the
  Data Retention Policy's 12-month requirement (`85ec360`).
- **2026-09-20: Hub and core CMKs removed.** Hub `cmk.tf` (the
  `kv-devplus-cmk-eus2` vault, its private endpoint, diagnostic setting and
  role assignments, keys `key-flowlogs` / `key-cloudflared`) and core's CMK
  files were deleted in favour of Microsoft-managed keys (Drata Test 254). See
  `docs/DEFENDER-FINDINGS.md`.

> Renaming a Key Vault is a recreate (names are immutable) and `prevent_destroy`
> is on, so any *future* vault rename must be done by hand off-pipeline:
> back up secrets → `tofu state rm` the vault → apply → restore secrets →
> repoint CI. The old name then soft-deletes and locks for 90 days.

## Private DNS over the Cloudflare tunnel (WARP)

Private DNS already works over WARP — the existing Local Domain Fallback resolves
the `privatelink.*` zones to private-endpoint IPs (e.g.
`kv-devplus-prod.privatelink.vaultcore.azure.net → 10.100.7.5`).

When a **new** private service is added, its domain must be added to the WARP
**Local Domain Fallback** list, or WARP resolves it publicly and the request gets
a "public networks denied" error. Log Analytics centralization needs these
domains added to the same fallback config that already resolves
`vaultcore.azure.net`:

- `loganalytics.io`  ← the Logs **query** entry point (`api.loganalytics.io`)
- `oms.opinsights.azure.com`
- `ods.opinsights.azure.com`
- `monitor.azure.com`
- `agentsvc.azure-automation.net`

> **Gotcha:** Local Domain Fallback matches the **queried name**, not the CNAME
> targets it chains through. The query starts at `api.loganalytics.io` (which
> CNAMEs into `*.privatelink.monitor.azure.com`), so `monitor.azure.com` alone is
> not enough — `loganalytics.io` itself must be on the list, or WARP resolves the
> whole chain on public DNS before it ever reaches the private `api` record.

Verify: `nslookup api.loganalytics.io` should return a `10.100.7.x` address (the
private `api` record), and `az monitor log-analytics query --workspace <id>
--analytics-query "print ok=1"` should succeed. Access is still gated by RBAC
(`Log Analytics Reader`, granted to `founders`).

## Secrets rotation runbook

Secrets are fetched at deploy time (masked, never stored in GitHub). The hub
pipeline reads only the prod Key Vault (`kv-devplus-prod`, `KEY_VAULT_NAME` in
`deploy.yml`). Core's `deploy.yml` matrix and the platform tenant deploys read
each environment's own vault (`KEYVAULT` environment variable =
`kv-devplus-{dev,stage,prod}`), so a core or tenant secret must be updated in
every env vault that holds it. None auto-rotate: rotate on a schedule and
re-run the relevant pipeline.

| Secret (KV name) | Used by | Rotation |
|---|---|---|
| `ghcr-puller-pat` / `ghcr-puller-username` | GHCR image pulls (reflected to tenant namespaces) | Fine-grained PAT, **rotate ≤90 days**. Mint a new PAT (Packages: Read on DevPlusInc), update the KV secret, re-run core deploy (Terraform rewrites the `ghcr-pull-secret` dockerconfigjson and Reflector re-mirrors it; ESO is installed but unused). Tighten `ghcr_reflector_namespace_pattern` from `.*` if scope allows. |
| `hub-cloudflare-tunnel-token` / `cloudflare-terraform-token` | cloudflared tunnel (hub, prod vault) / Cloudflare provider (core, env vaults) | Rotate in Cloudflare Zero Trust, update KV, re-run hub/core deploy (the ACI/container group rolls on token change). |
| `hub-mongodbatlas-client-id` / `-secret` | Atlas provider: hub (prod vault), core and platform tenant deploys (env vaults) | Rotate the Atlas org service-account, update every vault that holds it, re-run hub, core and tenant deploys. Core and platform use the same `hub-` secret names; there is no separate core Atlas secret. |
| `github-app-key` | DevPlus Connector App (tenant token minting) | Regenerate the App private key, update KV, re-run core deploy. |
| `workos-api-key` / `auth-service-workos-webhook-secret` | auth-service (core) | Rotate in WorkOS, update the env vault, re-run core deploy. |
| `hubspot-access-token` | Support desk token mirrored as `hubspot-access-token-shared` (core, optional per env) | Rotate the HubSpot private-app token, update the env vault, re-run core deploy. |
| `github install-state` HMAC | github-callback signing | Auto-generated (`random_password`); rotate by tainting and coordinating tenant backends (shared secret). |

Hub flow-log storage and cloudflared use Microsoft-managed platform keys, so
there is no hub CMK rotation runbook. Core's CMKs were removed too (core `c77c41e`), so neither repo has one.
