---
id: 35278bb7-ae89-489a-81ce-0ea0307d8e60
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
DevOps | devplus-core-infra/docs/superpowers/specs/2026-07-14-atlas-downsize-design.md

Source: DevPlusInc/devplus-core-infra/docs/superpowers/specs/2026-07-14-atlas-downsize-design.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-core-infra, docs/superpowers/specs/2026-07-14-atlas-downsize-design.md).

# Downsize stage/prod MongoDB Atlas clusters to M10

> **Status (checked 2026-10-04): implemented.** Merged in PR #37 (merge
> `a825bf5`, 2026-07-15). Evidence: `mongodb.tf` `mongodbatlas_advanced_cluster.prod`
> is 3x M10 / 30 GB in a single eastus2 `region_configs` with
> `analytics_specs`/`read_only_specs` at 0 nodes and `auto_scaling` off;
> `environments/{stage,prod}.tfvars` set `deploy_atlas_cluster = false` on
> centralus. Changed since (PRs #41/#42, 2026-07-26): weekly snapshot retention
> is 1 month (not 4 weeks), weekly snapshots are copied to centralus, and
> snapshot-failure alerts were added. Restore steps: README "Restoring
> production sizing (Atlas)".

**Date:** 2026-07-14
**Status:** Approved
**Scope:** MongoDB Atlas clusters only. AKS and all other infrastructure are
explicitly out of scope. Dev is untouched (already the target shape).

## Context

Stage and prod each run a dedicated Atlas cluster sized for production traffic
that does not exist yet (no customers):

- 3× M30 electable nodes (100 GB disk) in eastus2
- 1× M30 analytics + 1× M30 read-only node in each of eastus2 and centralus
  (7 M30 nodes total per env)
- Compute auto-scaling M30–M50, disk auto-scaling
- Continuous backup: 6-hourly×7d + daily×30d + weekly×8w + monthly×12m

Approximate cost: $2.5–3k/month per environment. Dev runs a 3-node M10
(~$250/month).

### Alternatives considered

- **Atlas Flex tier (~$8–30/mo per env):** rejected. Flex does not support
  OIDC Workload Identity Federation (M10+ dedicated only), and
  `auth-service/main.go` hardcodes `MONGODB-OIDC`; the tenant-provisioning
  model (`mongodb_atlas_idp_ids` output → federated DB users) also assumes
  OIDC. Flex would force a SCRAM password (static secret in TF state + K8s)
  and an auth-service code change. Also unavailable in eastus2 on Azure.
- **M30 single-region trimmed (~$1.1k/mo per env):** rejected; more headroom
  than needed with zero customers.

**Chosen: resize stage/prod in place to the dev shape (3× M10, single
region).** Zero application changes; OIDC, tenant provisioning, alerts, and
org federation config all keep working. ~85% cost reduction.

## Changes

All changes apply identically to the `stage` and `prod` workspaces (both are
served by the `mongodbatlas_advanced_cluster.prod` resource).

### 1. `environments/stage.tfvars`, `environments/prod.tfvars`

Set `deploy_atlas_cluster = false` on the `centralus` region entry.
`local.atlas_regions` collapses to `[eastus2]`, which removes the secondary
region_config from `replication_specs` automatically.

Not affected (keyed off `var.regions`, not `atlas_regions`):
`mongodbatlas_project_ip_access_list.nat` (both NAT IPs stay allowlisted) and
`mongodbatlas_federated_settings_identity_provider.aks` (both AKS IdPs stay
registered). `local.auth_service_enabled` remains true.

### 2. `mongodb.tf` — `mongodbatlas_advanced_cluster.prod`

In-place resize (no destroy/recreate; termination protection untouched):

- `electable_specs`: `instance_size` M30 → **M10**, `disk_size_gb` 100 → **30**
- `analytics_specs` and `read_only_specs`: **`node_count` → 0** (removes the
  4 M30 nodes per env)
- `auto_scaling`: **all flags → `false`** (also required: Atlas rejects a
  disk decrease while disk auto-scaling is enabled)

Note: these blocks are explicitly zeroed/disabled rather than deleted — per
the mongodbatlas provider docs, removing them from config sends NO change to
Atlas, which would silently leave the M30 nodes and auto-scaling live. They
can be deleted in a later PR once Atlas state reflects the disable.

Kept as-is: `backup_enabled = true`, `termination_protection_enabled = true`,
oplog pins (prod 5120 MB / stage 2048 MB — both fit a 30 GB disk), the OIDC
custom role/DB user, all three alert configurations, and the org federation
config block.

### 3. `mongodb.tf` — `mongodbatlas_cloud_backup_schedule.main`

Trim retention to: **daily×1 retained 7 days + weekly×1 retained 4 weeks**.
Remove the hourly and monthly policy items.

## Rollout

1. `terraform plan` in the **stage** workspace; verify the plan is
   update-in-place only for the cluster (no replaces).
2. Apply stage. Atlas performs a rolling resize; wait for the cluster to
   return to steady state in the Atlas UI.
3. Verify: auth-service `/health` on auth.stage.devplus.io and a login
   round-trip (webhook/callback path exercises Mongo).
4. Repeat plan/apply/verify for **prod**.

**Known Atlas API caveat:** a tier downsize + region removal + disk shrink in
one update is occasionally rejected. If the apply errors, split into two
applies: first remove analytics/read-only/secondary-region and auto-scaling,
then shrink tier and disk. Same end state.

**Pre-apply check:** compute auto-scaling may have scaled a live cluster above
M30; the explicit M10 target still applies cleanly, but eyeball current tier
and disk usage in the Atlas UI first — a disk shrink below used storage would
fail.

## Reversal

`git revert` the change and apply. Every change is an in-place resize in both
directions. (Restoring analytics/read-only nodes and the second region is
likewise additive.)

## Cost impact (approximate)

Per env: 7× M30 + auto-scale headroom (~$2.5–3k/mo) → 3× M10 (~$250/mo).
Across stage+prod: ~$5–6k/mo → ~$500/mo.
