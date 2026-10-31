---
id: 08229032-2943-4346-aaf8-de6fe3cf3eb0
binder: binder-mvvle32a-1
---
DevOps | devplus-core-infra/README.md

Source: DevPlusInc/devplus-core-infra/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-core-infra, README.md).

# devplus-core-infra

Core infrastructure for all environments (dev, stage, prod). Deploys regional VNets, AKS clusters, private endpoints, MongoDB Atlas, AI Foundry, monitoring and alerting, in-cluster platform services (`github-callback`, `auth-service`) with OpenTelemetry telemetry to Application Insights, and supporting services. The prod workspace also owns the shared Cloudflare zone security rules and the R2 edge-log bucket. DNS zones are centralized in the hub; container images come from GHCR (there is no ACR).

## Architecture

```
                        +---------------------------+
                        |     devplus-hub-infra      |
                        |       (hub VNet)           |
                        |     10.100.0.0/16          |
                        |                            |
                        |  - Private DNS Zones       |
                        |  - Cloudflared Tunnel      |
                        |  - GitHub Runners          |
                        +---+-----------+---+--------+
                            |           |   |
                   VNet Peering    Peering  Peering
                            |           |   |
          +-----------------+     +-----+   +----------------+
          |                       |                          |
+---------v----------+  +---------v----------+  +------------v-------+
|    dev (eus2)      |  |  stage (eus2+cus)  |  |  prod (eus2+cus)   |
|   10.2.0.0/16      |  | 10.10.0.0/16       |  | 10.20.0.0/16       |
|                     |  | 10.11.0.0/16       |  | 10.21.0.0/16       |
|  AKS: eus2          |  |  AKS: eus2 + cus   |  |  AKS: eus2 + cus   |
|  Atlas: M10 (eus2)  |  |  Atlas: M10 (eus2) |  |  Atlas: M10 (eus2) |
+---------------------+  +--------------------+  +--------------------+
```

### Per-environment resource map

```
devplus-core-infra (per environment via Terraform workspaces)
  |
  |-- Shared (one per environment)
  |     |-- Resource Group
  |     |-- Central Key Vault (hub-owned; referenced via data source)
  |     |-- MongoDB Atlas Project + Cluster
  |     |     |-- IP Access List (NAT IPs per region)
  |     |     |-- Workload Identity Federation (per AKS cluster)
  |     |     |-- Org federation config (prod workspace only, all envs' IdPs)
  |     |     |-- auth-service DB user + role (auth.workos, least-privilege)
  |     |     |-- Database auditing
  |     |     |-- Backup Schedule + Alerts (stage/prod)
  |     |-- AI Foundry Hub + Project
  |     |     |-- AI Services Account (S0, private, AAD-only)
  |     |     |-- Model: gpt-5.4 (GlobalStandard)
  |     |     |-- Model: gpt-5.4-mini (GlobalStandard)
  |     |     |-- Model: text-embedding-3-small (GlobalStandard)
  |     |     |-- Model catalogue: 86 GlobalStandard deployments (ai-foundry-models.tf)
  |     |     +-- Fireworks: FW-GLM-5.3-Flash (opt-in, dev only)
  |     |-- AI Foundry Storage Account
  |     |-- Application Insights (OpenTelemetry sink)
  |     |-- Microsoft Defender for Cloud
  |     |     |-- Storage, Containers, Key Vault, ARM, AI (AI off in dev)
  |     |-- Security action group + activity-log alerts (Policy, NSG, public IP,
  |     |     SQL firewall, security solution) + AKS node CPU metric alerts
  |     |-- Marketplace spend budget (alerts on the first $1)
  |     +-- Cloudflare zone WAF rulesets + R2 edge-log bucket (prod workspace only)
  |
  |-- In-cluster services (eus2 AKS, gated on deploy_aks)
  |     |-- github-callback           (GitHub App install callback hub)
  |     |-- auth-service              (WorkOS org callback + webhook)
  |     |-- OpenTelemetry Collector   (OTLP -> Application Insights)
  |     +-- Reflected secrets (fan-out to tenant namespaces: github-app-secret,
  |           github-install-state-shared, hubspot-access-token-shared)
  |
  |-- Per AKS cluster (every AKS region)
  |     |-- Cluster bootstrap (Karpenter pools, storage class, overprovisioning)
  |     |-- External Secrets + Reflector + Reloader (ghcr-pull-secret fan-out)
  |     +-- Image-attestation verification (Kyverno, ghcr.io/devplusinc/*)
  |
  +-- Per Region (modules/region/)
        |-- Resource Group + Tag Policies
        |-- VNet + 3 Subnets
        |     |-- aks_nodes    (/22, 1024 IPs)
        |     |-- general      (/24, 256 IPs)
        |     +-- private_endpoints (/24, 256 IPs)
        |-- NAT Gateway + Public IP
        |-- NSGs (deny-all-inbound baseline)
        |-- AKS Cluster (private, Cilium, workload identity)
        |     |-- System node pool
        |     |-- Karpenter autoscaling
        |     +-- Cluster bootstrap (Helm)
        |-- Private Endpoints
        |     |-- AI Services
        |     +-- AI Foundry Storage (blob)
        |-- VNet Flow Logs -> hub-managed Network Watcher
        |-- Flow Logs Storage Account + Private Endpoints (blob, queue)
        +-- Diagnostic Settings (NSGs x3, NAT GW, NAT IP, VNet,
              AKS control plane: kube-audit-admin, guard)
              +-- all -> central hub Log Analytics workspace
```

### Network diagram

```
+---------------------------------------------------------------+
|  Region (e.g. eastus2)                                        |
|                                                               |
|  VNet: 10.X.0.0/16                                           |
|  +---------------------------+                                |
|  | aks_nodes    10.X.0.0/22  |  AKS system + Karpenter nodes |
|  +---------------------------+                                |
|  | general      10.X.4.0/24  |  General workloads            |
|  +---------------------------+                                |
|  | priv_endpts  10.X.8.0/24  |  PE: AI, AIF blob, flow logs  |
|  +---------------------------+                                |
|       |                                                       |
|       v                                                       |
|  +----------+     +------------------+                        |
|  | NAT GW   |---->| Public IP        |---> MongoDB Atlas      |
|  +----------+     | (static)         |---> External APIs      |
|                    +------------------+                        |
|                                                               |
|  AKS Overlay Networking:                                      |
|    Service CIDR: 10.1.0.0/16                                  |
|    Pod CIDR:     10.244.0.0/16                                |
+---------------------------------------------------------------+
        |
   VNet Peering
        |
+---------------------------------------------------------------+
|  Hub VNet: 10.100.0.0/16                                      |
|  Private DNS Zones (used by core-infra):                      |
|    privatelink.blob.core.windows.net                          |
|    privatelink.queue.core.windows.net                         |
|    privatelink.cognitiveservices.azure.com                    |
|    privatelink.<region>.azmk8s.io                             |
+---------------------------------------------------------------+
```

### CI/CD pipeline

```
  Pull request / push to main / workflow_dispatch
           |
           v
  +------------------+  +---------+  +-----------------------------+
  | Plan (matrix)    |  | Checkov |  | Lint: fmt, validate, tflint |
  | dev, stage, prod |  +---------+  +-----------------------------+
  +------------------+  <-- -lock=false; PR: posts plan as comment
           |
           v
  +------------------+
  | Deploy dev       |  (push to main or workflow_dispatch; needs plan,
  +------------------+   checkov, lint; optional dev_replace / dev_target)
           |
           v
  +------------------+
  | Deploy stage     |  (requires dev success; GitHub Environment `stage`)
  +------------------+
           |
           v
  +------------------+
  | Deploy prod      |  (requires stage success; GitHub Environment `prod`)
  +------------------+
```

`.github/workflows/deploy.yml` runs on the `devplus-runners` self-hosted runner group with OpenTofu `1.11.5` (kept in sync with `devplus-hub-infra`, which shares the backend). Each job signs in with the environment's `vars.AZURE_CLIENT_ID` (OIDC) and loads its secrets from the environment's Key Vault (`vars.KEYVAULT`) via `.github/actions/load-kv-secrets`. No plan artifact is saved: each deploy job re-plans and applies in one job. The in-repo Go services build separately: `auth-service-build.yml` and `github-callback-build.yml` push attested images to `ghcr.io/devplusinc/{auth-service,github-callback}`.

## Environments

Managed via Terraform workspaces with per-environment tfvars.

| Environment | Subscription | Regions | AKS | MongoDB Atlas |
|---|---|---|---|---|
| dev | `53a006e0...` | eastus2 | Yes (Free, 1 node) | M10 single-region |
| stage | `61e4e1a9...` | eastus2, centralus | Yes, both regions (Standard) | M10 single-region (eastus2) |
| prod | `a7242621...` | eastus2, centralus | Yes, both regions (Standard) | M10 single-region (eastus2) |

Stage and prod Atlas were downsized from M30 multi-region on 2026-07-14; see [Restoring production sizing](#restoring-production-sizing-atlas).

### AKS sizing per environment

| | Dev | Stage | Prod |
|---|---|---|---|
| SKU Tier | Free | Standard | Standard |
| System VM | D2pds_v6 | D4pds_v6 | D4pds_v6 |
| System nodes | 1 | 2 | 3 |
| Zones | none | eus2: 1, 3; cus: 1, 2, 3 | eus2: 2, 3; cus: 1, 2, 3 |
| Karpenter CPU limit | 32 | 64 | 128 |
| Karpenter memory limit | 64Gi | 128Gi | 256Gi |
| General pool capacity | spot | spot | on-demand only |
| On-demand fallback pool | yes (16 CPU / 32Gi, chart default) | yes (32 CPU / 64Gi) | yes (64 CPU / 128Gi) |
| Overprovisioning | 1x 1CPU/2Gi | 2x 1CPU/2Gi | 3x 2CPU/4Gi |

Per-region zones come from `system_node_zones` on each `regions` entry in the tfvars (eastus2 zone support differs per subscription), falling back to the env-level value. Prod's general pool is on-demand only because the prod subscription gets no arm64 D/B spot capacity (`environments/cluster-bootstrap-prod.yaml`).

Every cluster is private with local accounts disabled, Entra ID RBAC, the `stable` auto-upgrade channel confined to a Sunday 02:00 UTC window, and host encryption on the system pool. Disks use Microsoft-managed keys: the customer-managed key setup (disk encryption sets and CMK vaults) was removed on 2026-09-20, and `disk_encryption_set_id` is in `ignore_changes`, so a cluster created before then keeps its existing disk encryption set until it is rebuilt. `prevent_destroy` refuses any plan that would replace a cluster.

## Network layout

VNet CIDRs are staggered per environment. Spokes peer with the centralized hub (`10.100.0.0/16`) but not with each other.

| Environment | Region | VNet CIDR |
|---|---|---|
| hub | eastus2 | `10.100.0.0/16` |
| dev | eastus2 | `10.2.0.0/16` |
| stage | eastus2 | `10.10.0.0/16` |
| stage | centralus | `10.11.0.0/16` |
| prod | eastus2 | `10.20.0.0/16` |
| prod | centralus | `10.21.0.0/16` |

### Subnets per region

| Subnet | CIDR (relative) | Purpose |
|---|---|---|
| aks_nodes | `/22` at offset 0 | AKS system and workload nodes |
| general | `/24` at offset 4 | General-purpose workloads |
| private_endpoints | `/24` at offset 8 | Private endpoints for all services |

AKS overlay networking: service CIDR `10.1.0.0/16`, pod CIDR `10.244.0.0/16`.

## DNS

Private DNS zones are centralized in the hub (`devplus-hub-eus2` resource group, prod subscription). Core-infra references them via data sources using the `azurerm.hub` provider. Spoke VNet links are managed by the hub, except for the `azmk8s` zones: the hub does not link spokes to them, and each AKS cluster uses its region's hub zone as its custom private DNS zone (the AKS identity holds Private DNS Zone Contributor on it).

| Zone | Purpose |
|---|---|
| `privatelink.blob.core.windows.net` | Blob Storage (AI Foundry storage, flow-log storage) |
| `privatelink.queue.core.windows.net` | Queue Storage (flow-log storage) |
| `privatelink.cognitiveservices.azure.com` | AI Services |
| `privatelink.<region>.azmk8s.io` | AKS private API server |

## AI Services

Single AI Foundry hub per environment (`aif-devplus-<env>`, AI Services account `ais-devplus-<env>`) with GlobalStandard model deployments. Each region gets private endpoints for the AI Services account and the Foundry storage, but only the primary region's endpoint registers the shared DNS record (hub-spoke has no spoke-to-spoke transit, so a secondary-region record would be unreachable from eastus2).

| Model | SKU | Capacity |
|---|---|---|
| gpt-5.4 | GlobalStandard | 200 |
| gpt-5.4-mini | GlobalStandard | 100 |
| text-embedding-3-small | GlobalStandard | 100 |

`ai-foundry-models.tf` also deploys a catalogue of 86 pay-per-token GlobalStandard models in every environment (capacity 100 by default, clamped to each model's quota, raised per env through `foundry_capacity_overrides`). Azure Marketplace models are excluded because sponsorship credits do not cover them; the `marketplace-spend-<env>` budget alerts on the first dollar of Marketplace charges. Fireworks-hosted models (`FW-GLM-5.3-Flash`) are opt-in per env through `fireworks_enabled` (on in dev only).

Capacity is a rate cap (1 unit = 1K tokens/min), not committed spend. Calling a non-OpenAI model needs the `MaaS/*` data actions in `Azure AI Developer`, which this repo grants to the admins group and the Foundry project identity; `Cognitive Services OpenAI User` covers only the OpenAI models.

## MongoDB Atlas

| | Dev | Stage/Prod |
|---|---|---|
| Tier | M10 | M10 (downsized from M30 on 2026-07-14) |
| Regions | 1 (eastus2, single replica set) | 1 (eastus2, single replica set) |
| Nodes | 3 electable | 3 electable (analytics and read-only specs kept at 0 nodes) |
| Auto-scaling | no | no (explicitly disabled) |
| Disk | 30 GB | 30 GB |
| Oplog | 2048 MB | stage 2048 MB, prod 5120 MB |
| Backup | Enabled (Atlas default policy) | Daily x7 days + weekly x1 month; weekly copies to centralus |
| Alerts | none | CPU > 80%, replication lag > 1m, connections > 80%, snapshot fallback/behind |
| Database auditing | yes | yes |

Workload identity federation is configured per AKS cluster, allowing pods to authenticate to Atlas via OIDC. The org-level federation config (`dataAccessIdentityProviderIds`) is a single org-wide object, so only the prod workspace manages it; it reads the dev and stage IdP ids from their states (`terraform_remote_state`), which is one reason deploys run dev, then stage, then prod.

## What this module manages

### Shared resources (root level)

| File | Resources |
|---|---|
| `dns.tf` | Resource group, data sources for centralized hub DNS zones + central hub Log Analytics workspace |
| `keyvault.tf` | Data source for the central per-env Key Vault `kv-devplus-<env>` (owned by the hub; images come from GHCR, so there is no ACR) |
| `mongodb.tf` | Atlas project, IP access list, cluster, workload identity federation, org federation config (prod), auth-service DB user, backup, alerts, auditing |
| `ai-foundry.tf` | AI Services account, Foundry hub + project, core model deployments, storage, private endpoints |
| `ai-foundry-models.tf` | Model catalogue, Fireworks models, Marketplace spend budget, `Azure AI Developer` grants |
| `defender.tf` | Defender for Cloud pricing tiers (Storage, Containers, Key Vault, ARM, AI) + Application Insights |
| `activity-log-alerts.tf` | Security action group + activity-log alerts |
| `metric-alerts.tf` | AKS node CPU alerts (prod adds VMSS/VM CPU alerts) |
| `bootstrap.tf` | Helm-based AKS cluster bootstrap (Karpenter, storage classes, overprovisioning) + Kyverno image verification |
| `external-secrets.tf`, `reflector.tf`, `reloader.tf` | External Secrets Operator + `ghcr-pull-secret`, Reflector, Stakater Reloader (per AKS cluster) |
| `github-callback.tf`, `auth-service.tf`, `otel-collector.tf` | In-cluster hub services on the eus2 cluster, exposed through Cloudflare Tunnel (`modules/cloudflare-ingress-core`) |
| `github-app.tf`, `github-install-state.tf`, `hubspot.tf` | Reflector-mirrored `github-app-secret`, `github-install-state-shared`, `hubspot-access-token-shared` |
| `cloudflare-security.tf`, `cloudflare-r2.tf` | Prod-only: zone custom firewall + managed WAF rulesets, R2 `devplus-edge-logs` bucket with a 365-day lifecycle |

### Regional resources (modules/region/)

| File | Resources |
|---|---|
| `main.tf` | Resource group, tag inheritance policies |
| `vnet.tf` | VNet, subnets (AKS nodes, general, private endpoints) |
| `nsg.tf` | NSGs per subnet (deny-all-inbound baseline, VNet-only for private endpoints) |
| `nat.tf` | NAT gateway + public IP for all subnets |
| `aks.tf` | Private AKS cluster (Cilium, overlay, workload identity, Karpenter node auto-provisioning, `prevent_destroy`); Container Insights → central hub workspace |
| `identity.tf` | AKS managed identity + role assignments |
| `monitoring.tf` | Flow-log storage (GRS, soft-delete, AAD-only, blob + queue private endpoints) + flow log attached to the **hub-managed** Network Watcher; all diagnostics, including AKS control-plane audit logs, → the **central hub** Log Analytics workspace. No per-region workspace or Network Watcher. |

## Tagging

Every taggable Azure resource carries the SOC 2 asset-inventory tag schema — `environment`, `project`, `region`, `owner`, `data-classification`, `managed-by`. Root-module resources use `local.common_tags` (`locals.tf`); regional resources use `local.tags` (`modules/region/main.tf`). Both maps are the same shape as `devplus-hub-infra`'s, so the inventory reads consistently across repos.

Terraform is the source of truth: there are no `ignore_changes = [tags]` opt-outs, so a tag added or edited out-of-band is reverted on the next apply. The tag-inheritance Azure Policy assignments on each region resource group (`modules/region/main.tf`) stay as defence in depth for resources created in those groups outside Terraform — they inherit the same `environment`/`project` values Terraform sets, so the two agree rather than fighting.

Three deliberate exceptions:

- `azurerm_ai_foundry.main` — tags are declared but not enforced (`ignore_changes = [tags]`). Azure ML writes service-managed `__SYSTEM__<kind>_<account>` tags onto the hub recording its AIServices / AzureOpenAI connections; enforcing the tag set deletes them on every apply.
- `azapi_resource.ai_services_connection` — `Microsoft.MachineLearningServices/workspaces/connections` is an ARM *proxy* resource and carries no tags. `azapi_resource` exposes a `tags` argument for every type, so a schema-driven audit will flag it; setting it would fail the PUT.
- MongoDB Atlas and Cloudflare resources are untagged. Their `tags` attributes are provider-specific concepts unrelated to the Azure asset inventory this schema serves.

## AKS cluster bootstrap

In the same apply, after AKS deploys and a 180 s wait for RBAC propagation, Terraform installs a Helm release on every cluster (`bootstrap.tf`):

```
k8s/charts/cluster-bootstrap/
environments/cluster-bootstrap-{dev,stage,prod}.yaml
```

This configures:
- **Karpenter** node provisioning (spot-first general pool + on-demand fallback pool, D/B series, arm64; prod's general pool is on-demand only)
- **Storage class** (`managed-csi-retain` with StandardSSD_LRS)
- **Overprovisioning** pods for faster scheduling

Kyverno (`modules/image-verification`) is installed next and verifies GitHub-native SLSA provenance + SBOM attestations on `ghcr.io/devplusinc/*` images. Its mode comes from `image_verification_failure_action`: the variable defaults to `Enforce`, but all three tfvars set `Audit` for now.

## Usage

```bash
# Select environment
tofu workspace select dev

# Plan and apply
tofu plan -var-file=environments/dev.tfvars
tofu apply -var-file=environments/dev.tfvars
```

Normal changes apply through CI. A workstation run also needs the `TF_VAR_*` secrets that `deploy.yml` loads from the environment Key Vault (Atlas client id/secret, Cloudflare token, GHCR puller, GitHub App key, WorkOS keys, optional HubSpot token), a network path to the private state storage account, and a local `override.tf` (gitignored) if you need kubelogin to use the Azure CLI instead of the CI workload-identity login in `providers.tf`.

## State

Remote state in Azure Storage (`devplusstateprod` in the prod subscription), key `core.tfstate`, one state per workspace (`dev`, `stage`, `prod`). OIDC authentication. Managed by `devplus-base-infra`.

## Related repos

| Repo | Purpose |
|---|---|
| `devplus-hub-infra` | Hub VNet, DNS zones, cloudflared tunnel, GitHub runners, peering, central Key Vaults, Log Analytics + Sentinel, Network Watchers |
| `devplus-cicd-infra` | OIDC service principals, federated credentials, role assignments |
| `devplus-base-infra` | Terraform state storage account and management locks |
| `terraform-devplus-tenant` | Per-tenant deploys; reads this repo's outputs (AKS OIDC issuers, Atlas IdP ids, install-state secret) |

## Restoring production sizing (Atlas)

The stage and prod Atlas clusters were downsized on 2026-07-14 (no customers
yet) from the production-grade configuration to a dev-shaped 3x M10
single-region replica set. Design/rationale:
`docs/superpowers/specs/2026-07-14-atlas-downsize-design.md` (tracked in
this repo: `docs` is in `.gitignore`, so the spec and plan were force-added).

Every change was an in-place resize, so restoring is too (rolling, no
downtime beyond replica-set elections):

1. `git revert` the downsize commits (find them with
   `git log --oneline --grep downsize`; later commits changed the same
   blocks, so expect conflicts), or re-apply by hand:
   - `environments/{stage,prod}.tfvars`: set `deploy_atlas_cluster = true`
     on the centralus region.
   - `mongodb.tf` `mongodbatlas_advanced_cluster.prod`: add back a second
     `region_configs` entry for centralus with `priority = 0` and NO
     `electable_specs` (all electable nodes live in the priority-7 eastus2
     region) — only `analytics_specs` and `read_only_specs`. In the
     eastus2 `region_configs` entry, bump `electable_specs` to 3x M30
     electable, `disk_size_gb = 100`, and add matching `analytics_specs`
     and `read_only_specs` of 1x M30 (100 GB) to BOTH region_configs.
     Restoring means flipping the (still-present, explicitly
     disabled/zeroed) `auto_scaling`/`analytics_specs`/`read_only_specs`
     back to their production values on each `region_configs` block, NOT
     re-adding deleted blocks — unless a later PR has removed them, in
     which case add the blocks back explicitly rather than relying on the
     provider to infer them. Set `auto_scaling` on EVERY region_config to
     `disk_gb_enabled = true`, `compute_enabled = true`,
     `compute_scale_down_enabled = true`, min M30 / max M50.
   - `mongodb.tf` `mongodbatlas_cloud_backup_schedule.main`: restore
     `policy_item_hourly` (interval 6, 7 days) and `policy_item_monthly`
     (interval 1, 12 months); daily back to 30 days, weekly (now 1 month)
     to 8 weeks; add `"DAILY"` to the centralus `copy_settings` frequencies.
2. Apply via the normal PR flow — CI plans all three envs on the PR and
   applies dev -> stage -> prod on merge to main.
3. Verify after each env: cluster steady in the Atlas UI, then
   `curl -fsS https://auth.stage.devplus.io/health` /
   `curl -fsS https://auth.devplus.io/health`.

Scaling UP never conflicts with disk auto-scaling, so the restore can be a
single apply per environment.

The original production config pinned `instance_size = "M30"` in
`electable_specs` while compute auto-scaling was live, which causes
perpetual plan drift whenever Atlas actually scales the cluster (the
provider sees the live tier diverge from the pinned value on every plan).
When restoring, consider setting `use_effective_fields = true` on the
cluster resource, or add `ignore_changes` on `instance_size` /
`disk_size_gb` for the electable specs, to avoid that fight.
