---
id: 35e234df-1d3e-4ee4-8f58-72f6743ab985
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
DevOps | core-infra Atlas downsize plan (2026-07-14)

Source: DevPlusInc/devplus-core-infra/docs/superpowers/plans/2026-07-14-atlas-downsize.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-core-infra, docs/superpowers/plans/2026-07-14-atlas-downsize.md). Status: implemented (historical plan).

# Atlas Stage/Prod Downsize Implementation Plan

> **Status (checked 2026-10-04): implemented.** Merged in PR #37 (merge
> `a825bf5`, 2026-07-15). Evidence: `mongodb.tf` `mongodbatlas_advanced_cluster.prod`
> is 3x M10 / 30 GB in a single eastus2 `region_configs` with
> `analytics_specs`/`read_only_specs` at 0 nodes and `auto_scaling` off;
> `environments/{stage,prod}.tfvars` set `deploy_atlas_cluster = false` on
> centralus. Changed since (PRs #41/#42, 2026-07-26): weekly snapshot retention
> is 1 month (not 4 weeks), weekly snapshots are copied to centralus, and
> snapshot-failure alerts were added. Restore steps: README "Restoring
> production sizing (Atlas)".

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Resize the stage and prod MongoDB Atlas clusters in place from 7× M30 multi-region to a dev-shaped 3× M10 single-region replica set, and trim the backup schedule.

**Architecture:** Pure Terraform/OpenTofu change in `devplus-core-infra`. Both stage and prod are served by the same `mongodbatlas_advanced_cluster.prod` resource (selected per workspace); the secondary region is dropped via tfvars (`deploy_atlas_cluster = false` on centralus), which collapses `local.atlas_regions` to `[eastus2]`. All cluster changes are update-in-place; CI (`.github/workflows/deploy.yml`) applies dev → stage → prod on merge to main.

**Tech Stack:** OpenTofu 1.11.x (CI-pinned; `terraform` ≥ 1.9 also works locally), mongodbatlas provider `~> 2.9`, GitHub Actions deploy pipeline.

**Spec:** `docs/superpowers/specs/2026-07-14-atlas-downsize-design.md`

## Global Constraints

- Scope is Atlas ONLY. Do not touch AKS, networking, IdPs, or the dev cluster resource (`mongodbatlas_advanced_cluster.dev`).
- Never remove `termination_protection_enabled = true` or `backup_enabled = true`.
- Never remove or alter the OIDC resources (`mongodbatlas_federated_settings_identity_provider.aks`, `mongodbatlas_federated_settings_org_config.main`, custom role, DB user) or the three alert configurations.
- Every plan output must show the cluster as `~ update in-place` — a `-/+ destroy and then create replacement` on `mongodbatlas_advanced_cluster.prod[0]` is a STOP-and-investigate failure.
- Work happens on the existing `downsizemdb` branch.
- `docs/` is gitignored in this repo — spec/plan files need `git add -f`; everything else in this plan (`.tf`, `.tfvars`, `README.md`) is tracked normally.

---

### Task 1: Drop the centralus Atlas region in stage/prod tfvars

**Files:**
- Modify: `environments/stage.tfvars:26-32` (centralus region block)
- Modify: `environments/prod.tfvars:26-32` (centralus region block)

**Interfaces:**
- Produces: `local.atlas_regions == [eastus2]` in the stage and prod workspaces (consumed by Task 2's single-region `region_configs`). `local.any_region_atlas` stays `true`, so `auth_service_enabled`, alerts, and the backup schedule remain enabled.

- [ ] **Step 1: Edit `environments/stage.tfvars`**

Change the centralus entry (leave eastus2 untouched):

```hcl
  {
    location             = "centralus"
    location_short       = "cus"
    vnet_address_space   = "10.11.0.0/16"
    deploy_aks           = true
    # Atlas downsized to single-region 2026-07-14 (no customers).
    # To restore: see "Restoring production sizing" in README.md.
    deploy_atlas_cluster = false
  }
```

- [ ] **Step 2: Edit `environments/prod.tfvars`**

Change the centralus entry (leave eastus2 untouched):

```hcl
  {
    location             = "centralus"
    location_short       = "cus"
    vnet_address_space   = "10.21.0.0/16"
    deploy_aks           = true
    # Atlas downsized to single-region 2026-07-14 (no customers).
    # To restore: see "Restoring production sizing" in README.md.
    deploy_atlas_cluster = false
  }
```

- [ ] **Step 3: Sanity-check with grep**

Run: `grep -A7 centralus environments/stage.tfvars environments/prod.tfvars`
Expected: both centralus blocks show `deploy_atlas_cluster = false`; `deploy_aks = true` unchanged. `environments/dev.tfvars` untouched.

- [ ] **Step 4: Commit**

```bash
git add environments/stage.tfvars environments/prod.tfvars
git commit -m "downsize: drop centralus from the stage/prod Atlas clusters"
```

---

### Task 2: Resize the stage/prod cluster resource to 3× M10

**Files:**
- Modify: `mongodb.tf:134-195` (the `Stage / Prod` header comment + `mongodbatlas_advanced_cluster.prod` resource)

**Interfaces:**
- Consumes: `local.atlas_regions` (now single-element in stage/prod, from Task 1), `local.azure_to_atlas_region` (locals.tf).
- Produces: unchanged resource address `mongodbatlas_advanced_cluster.prod[0]` and unchanged `connection_strings.standard_srv` (cluster name/DNS survive a resize), so `local.auth_service_atlas_host`, the DB-user cluster scope, and the backup schedule's `cluster_name` all keep working.

- [ ] **Step 1: Replace the header comment and resource**

Replace everything from the `# Stage / Prod: Dedicated M30 cluster...` comment block (mongodb.tf line 134) through the closing `}` of `mongodbatlas_advanced_cluster.prod` (line 195) with:

```hcl
# ---------------------------------------------------------------------------
# Stage / Prod: Dedicated M10 cluster (single region, 3-node replica set).
#
# DOWNSIZED 2026-07-14 (no customers yet) from the production-grade shape:
# 3x M30 electable (100 GB) in eastus2 + 1x M30 analytics and 1x M30
# read-only node in each of eastus2/centralus, compute auto-scaling M30-M50,
# disk auto-scaling. To restore that shape, follow "Restoring production
# sizing (Atlas)" in README.md.
# ---------------------------------------------------------------------------
resource "mongodbatlas_advanced_cluster" "prod" {
  count        = local.any_region_atlas && local.is_production ? 1 : 0
  project_id   = mongodbatlas_project.main.id
  name         = "${var.prefix}-${terraform.workspace}"
  cluster_type = "REPLICASET"

  backup_enabled                 = true
  termination_protection_enabled = true

  # Pin oplog to force a cluster republish (rolling restart) that loads the
  # workload-OIDC IdP onto the mongod nodes — without a republish AFTER the IdP is
  # enabled, MONGODB-OIDC auth fails ("Unknown Identity Provider") for both
  # auth-service and tenants (same reason .dev pins oplog above). prod = 5 GiB
  # (replication headroom); stage = 2 GiB. Atlas can't revert oplog once set.
  advanced_configuration = {
    oplog_size_mb = terraform.workspace == "prod" ? 5120 : 2048
  }

  replication_specs = [{
    region_configs = [{
      provider_name = "AZURE"
      region_name   = local.azure_to_atlas_region[local.atlas_regions[0].location]
      priority      = 7
      electable_specs = {
        instance_size = "M10"
        node_count    = 3
        disk_size_gb  = 30
      }

      # Explicitly zeroed/disabled rather than deleted: per the provider docs,
      # REMOVING these blocks sends no change to Atlas — the live M30
      # analytics/read-only nodes and auto-scaling would silently survive (and
      # Atlas rejects a disk shrink while disk auto-scaling is enabled). Safe
      # to delete these blocks in a later PR once Atlas state reflects them.
      analytics_specs = {
        instance_size = "M10"
        node_count    = 0
        disk_size_gb  = 30
      }
      read_only_specs = {
        instance_size = "M10"
        node_count    = 0
        disk_size_gb  = 30
      }
      auto_scaling = {
        disk_gb_enabled            = false
        compute_enabled            = false
        compute_scale_down_enabled = false
      }
    }]
  }]
}
```

Notes on what this changes relative to the current file: the `for i, r in local.atlas_regions` loop with its `merge()` is replaced with a single literal `region_configs` entry. The `analytics_specs`, `read_only_specs`, and `auto_scaling` blocks are NOT removed — per the mongodbatlas provider docs (v2.11.0), removing those blocks from the config sends no change to Atlas at all, silently leaving the live M30 analytics/read-only nodes and auto-scaling running (and Atlas rejects a disk decrease while disk auto-scaling stays enabled). Instead they are kept and explicitly zeroed/disabled: `analytics_specs`/`read_only_specs` get `node_count = 0`, and `auto_scaling` gets `disk_gb_enabled = false`, `compute_enabled = false`, `compute_scale_down_enabled = false`. The oplog comment and `advanced_configuration` are kept verbatim.

- [ ] **Step 2: Verify the disable is explicit, not a removal**

Run: `grep -n "analytics_specs\|read_only_specs\|auto_scaling" mongodb.tf`
Expected: matches inside `mongodbatlas_advanced_cluster.prod` for all three blocks — `analytics_specs`/`read_only_specs` with `node_count = 0`, and `auto_scaling` with `disk_gb_enabled = false`, `compute_enabled = false`, `compute_scale_down_enabled = false`. If any of the three blocks is absent from the resource, STOP — a removed block sends no change to Atlas (see the Notes above Step 1).

- [ ] **Step 3: Commit**

```bash
git add mongodb.tf
git commit -m "downsize: stage/prod Atlas clusters M30 multi-node -> 3x M10"
```

---

### Task 3: Trim the backup schedule

**Files:**
- Modify: `mongodb.tf` — `mongodbatlas_cloud_backup_schedule.main` (currently lines 200-231; line numbers shift after Task 2)

**Interfaces:**
- Consumes: `mongodbatlas_advanced_cluster.prod[0].name` (unchanged by Task 2).
- Produces: nothing consumed downstream.

- [ ] **Step 1: Replace the schedule's policy items**

Replace the whole `mongodbatlas_cloud_backup_schedule.main` resource with:

```hcl
# ---------------------------------------------------------------------------
# Backup policy for stage/prod. Trimmed 2026-07-14 alongside the cluster
# downsize (no customers): daily x7d + weekly x4w. The production-grade
# policy (6-hourly x7d + daily x30d + weekly x8w + monthly x12m) is in
# "Restoring production sizing (Atlas)" in README.md. Existing snapshots
# keep the retention they were created with (update_snapshots defaults to
# false).
# ---------------------------------------------------------------------------
resource "mongodbatlas_cloud_backup_schedule" "main" {
  count        = local.any_region_atlas && local.is_production ? 1 : 0
  project_id   = mongodbatlas_project.main.id
  cluster_name = mongodbatlas_advanced_cluster.prod[0].name

  reference_hour_of_day    = 3
  reference_minute_of_hour = 0

  policy_item_daily {
    frequency_interval = 1
    retention_unit     = "days"
    retention_value    = 7
  }

  policy_item_weekly {
    frequency_interval = 1
    retention_unit     = "weeks"
    retention_value    = 4
  }
}
```

(This deletes the `policy_item_hourly` and `policy_item_monthly` blocks and shrinks daily 30d→7d, weekly 8w→4w.)

- [ ] **Step 2: Commit**

```bash
git add mongodb.tf
git commit -m "downsize: trim stage/prod Atlas backup policy to daily x7 + weekly x4"
```

---

### Task 4: Document how to restore production sizing (README)

**Files:**
- Modify: `README.md` (append a section at the end)

**Interfaces:**
- Consumes: nothing. Referenced by the comments added in Tasks 1-3.

- [ ] **Step 1: Append the restore runbook to README.md**

(The README section text from the original plan: the "Restoring production sizing (Atlas)" runbook, now superseded by the current README section of the same name.)

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "downsize: document how to restore production Atlas sizing"
```

---

### Task 5: Validate locally

**Files:** none (verification only)

- [ ] **Step 1: Format check**

Run: `tofu fmt -check environments/ *.tf` (or `terraform fmt -check`)
Expected: no output, exit 0. If files are listed, run `tofu fmt` on them and amend the relevant commit.

- [ ] **Step 2: Validate**

Run: `tofu init -backend=false && tofu validate`
Expected: `Success! The configuration is valid.`

- [ ] **Step 3: Confirm the diff is complete and minimal**

Run: `git diff main...HEAD --stat`
Expected: exactly `environments/stage.tfvars`, `environments/prod.tfvars`, `mongodb.tf`, `README.md`, plus the force-added `docs/superpowers/` spec/plan files. Nothing else.

---

### Task 6: Pre-apply live checks (manual, Atlas UI)

**Files:** none (verification only)

Auto-scaling has been live on these clusters, so verify the real starting state before planning the resize:

- [ ] **Step 1: Check current tier** — Atlas UI → each of the `devplus-stage` and `devplus-prod` clusters → Overview. Note the current instance size (may be above M30 if compute auto-scaled). Any value is fine — the explicit M10 target still applies — this is just so the plan diff doesn't surprise you.

- [ ] **Step 2: Check used storage** — Atlas UI → cluster → Metrics → Disk Usage. Expected: well under 30 GB (the data is one small collection). If a cluster somehow uses ≥ 30 GB, STOP — a disk shrink below used storage fails; raise `disk_size_gb` in Task 2 accordingly.

---

### Task 7: PR, CI plans, review

**Files:** none (git/GitHub only)

- [ ] **Step 1: Push and open the PR** (`git push -u origin downsizemdb`, then `gh pr create` with the spec as the body, since `docs/` was gitignored at the time).

- [ ] **Step 2: Review the CI plan comments on the PR**

The deploy workflow comments one plan per environment. Expected:
- **dev:** `No changes.` (dev tfvars untouched; the edited resources are count=0 in dev)
- **stage & prod:** `mongodbatlas_advanced_cluster.prod[0]` as `~ update in-place` — instance_size → M10, disk_size_gb → 30, analytics/read-only `node_count` → 0 (instance_size/disk_size_gb also shown at M10/30 GB), `auto_scaling.disk_gb_enabled`/`compute_enabled`/`compute_scale_down_enabled` → false, and the centralus region_config removed; `mongodbatlas_cloud_backup_schedule.main[0]` as `~ update in-place`. **No resource may show `-/+` (replacement) or `- destroy`** — there are no expected destroys. If the cluster shows a replacement, STOP and investigate before merging.

  **Red flag:** if the plan diff does NOT show lines for `analytics_specs.node_count`, `read_only_specs.node_count`, or the `auto_scaling` flags changing, STOP — that means the disable isn't actually being sent to Atlas (the provider silently no-ops on removed blocks; it must see the explicit zeroed/disabled values to act).

- [ ] **Step 3: Merge**

Merge the PR once the plans match expectations.

---

### Task 8: Monitor rollout, verify stage, then prod

**Files:** none (operations only)

CI applies sequentially: deploy-dev → deploy-stage → deploy-prod, each behind its GitHub environment. An Atlas tier resize is rolling and typically takes 10-25 minutes per cluster; the apply step waits for it.

- [ ] **Step 1: Watch the run**

Run: `gh run watch` (or the Actions tab). Expected: deploy-dev is a fast no-op; deploy-stage performs the cluster update.

- [ ] **Step 2: Verify stage before prod applies**

If the `prod` GitHub environment has required reviewers, verify BEFORE approving; otherwise verify as soon as deploy-stage finishes:
- Atlas UI: `devplus-stage` cluster shows 3 nodes, M10, eastus2 only, no ongoing changes.
- Atlas UI: cluster shows 0 analytics nodes and 0 read-only nodes, and auto-scaling (compute + disk) shown as OFF.
- Atlas UI: Backup → Snapshot Schedule no longer lists hourly or monthly policy items (only daily x7 + weekly x4).
- Run: `curl -fsS https://auth.stage.devplus.io/health` → expected HTTP 200.
- Exercise a login round-trip on stage (callback path hits Mongo).

- [ ] **Step 3: Let prod apply, then verify the same way**

- Atlas UI: `devplus-prod` cluster shows 3 nodes, M10, eastus2 only.
- Atlas UI: cluster shows 0 analytics nodes and 0 read-only nodes, and auto-scaling (compute + disk) shown as OFF.
- Atlas UI: Backup → Snapshot Schedule no longer lists hourly or monthly policy items (only daily x7 + weekly x4).
- Run: `curl -fsS https://auth.devplus.io/health` → expected HTTP 200.

- [ ] **Step 4 (contingency): Atlas rejects the combined update**

If an apply fails with an Atlas API error about simultaneous changes (tier + disk + region removal in one update), split into two PRs with the SAME end state:
1. **PR A** — Task 1's tfvars change + explicitly disable auto-scaling and zero the analytics/read-only `node_count` (keep `analytics_specs`/`read_only_specs`/`auto_scaling` blocks present, just zeroed/disabled — do not remove them, per the provider-docs no-op behavior noted in Task 2), but keep `instance_size = "M30"` and `disk_size_gb = 100` in `electable_specs`. Merge, let CI apply.
2. **PR B** — change `electable_specs` to `instance_size = "M10"`, `disk_size_gb = 30`. Merge, let CI apply.
Then re-run the Step 2/3 verifications.

---

## Self-review notes

- Spec coverage: tfvars change → Task 1; cluster resize → Task 2; backup trim → Task 3; restore documentation (user addition) → Task 4 + comments in Tasks 1-3; rollout stage-first with verification → Tasks 7-8; two-phase contingency → Task 8 Step 4; pre-apply disk/tier check → Task 6. Reversal path documented in README (Task 4).
- No TDD test cycle: this is pure IaC — the failing-test/passing-test loop maps to `tofu validate` (Task 5) + CI plan review (Task 7), which is this repo's established verification path.
