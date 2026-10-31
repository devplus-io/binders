---
id: 9f02ef5b-d34f-4d99-92fe-657ed990dc4f
binder: binder-mvvle32a-1
---
DevOps | devplus-hub-infra/README.md

Source: DevPlusInc/devplus-hub-infra/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-hub-infra, README.md).

# devplus-hub-infra

Centralized hub VNet providing private network access (via Cloudflare Tunnel) and GitHub Actions runner networking for all environments, plus the org's centralized private DNS zones and per-environment Key Vaults.

## Architecture

```
Developer / service
  | (Cloudflare Zero Trust)
  v
Cloudflare Tunnel (cloudflared on Azure Container Instance)
  |
Hub VNet (10.100.0.0/16, prod subscription, eastus2)
  |-- cloudflared subnet (ACI, private)   -> connects the tunnel to all peered VNets
  |-- GitHub Actions runner subnet        -> shared self-hosted runners
  |-- private endpoint subnet             -> Key Vault, flow-log storage and AMPLS private endpoints
  |-- Key Vaults (dev / stage / prod)     -> central secrets, private-endpoint only (dev/stage vaults sit in their own subscriptions)
  |-- NAT Gateway                         -> static outbound IP for runners + cloudflared
  |-- Private DNS zones                   -> centrally owned, linked to hub + every spoke
  |-- Log Analytics workspace + Sentinel  -> single SIEM workspace for hub and spoke logs (365-day retention)
  |
  |-- peering --> vnet-devplus-dev-eus2    (dev sub,   eastus2)
  |-- peering --> vnet-devplus-stage-eus2  (stage sub, eastus2)
  |-- peering --> vnet-devplus-stage-cus   (stage sub, centralus)
  |-- peering --> vnet-devplus-prod-eus2   (prod sub,  eastus2)
  +-- peering --> vnet-devplus-prod-cus    (prod sub,  centralus)

  (spoke VNets are created by devplus-core-infra; other products' spokes peer the same way, see "Adding a new spoke")
```

The hub peers bidirectionally to every spoke VNet (cross-subscription). cloudflared runs in the hub and reaches private resources in any peered environment, so there is no VPN gateway. Spoke-to-spoke traffic is not configured — environments remain isolated from each other.

## What this module manages

| Resource | File | Purpose |
|---|---|---|
| Hub resource group, VNet + cloudflared subnet | `vnet.tf` | Hub RG `devplus-hub-eus2`, VNet, ACI-delegated cloudflared subnet (the runner subnet is in `github-runners.tf`, the private-endpoint subnet in `keyvault.tf`) |
| Cloudflare Tunnel connector | `cloudflared.tf` | `cloudflared` container (Azure Container Instance) + NSG + diagnostic setting; private network access without a VPN gateway. Also the `founders` action group and a CPU metric alert on the container group |
| GitHub runner subnet + NSG + network settings | `github-runners.tf` | Network-isolated self-hosted runners with allowlisted GitHub/Actions IPs; registers the subnet with GitHub via `GitHub.Network/networkSettings` |
| Central Key Vaults (dev/stage/prod) | `keyvault.tf` | Per-environment vaults, private-endpoint only, RBAC authorization, audit logging. **Each vault has its own `devplus-keyvault-{env}` RG: prod in the hub (prod) subscription, dev/stage in their own subscriptions** so same-sub consumers (AI Foundry) can reference them. See `docs/OWNERSHIP-AND-RUNBOOKS.md`. |
| Microsoft Sentinel | `monitoring.tf` | Onboarded **once** on the hub Log Analytics workspace — the single SIEM pane for all spokes (spokes ship diagnostics/flow logs here and must not onboard Sentinel themselves). |
| NAT Gateway + public IP | `nat.tf` | Static outbound IP shared by the runner and cloudflared subnets |
| Centralized private DNS zones | `dns.tf` | Hub owns `privatelink.*` zones and links them to the hub VNet and every spoke VNet (the `*.azmk8s.io` zones are hub-only, see below) |
| VNet peering | `peering.tf` | Bidirectional hub-to-spoke peering for all environments (cross-subscription) |
| MongoDB Atlas IP access list | `mongodb.tf` | Allows the NAT gateway IP so runners can reach Atlas projects |
| Log Analytics workspace | `monitoring.tf` | `log-devplus-hub-eus2`: AAD-only (`local_authentication_enabled = false`), no internet ingestion or query, 365-day retention. Receives hub diagnostics plus core's spoke diagnostics, flow logs, AKS Container Insights and App Insights. Manages the `AKSAuditAdmin` table on the Basic plan. `founders` get `Log Analytics Reader` |
| Network Watcher + hub VNet flow log + storage account | `monitoring.tf` | Hub-owned Network Watcher for prod/eastus2; VNet flow log (azapi, user-assigned identity `id-flowlogs-devplus-hub-eus2`) with traffic analytics into Log Analytics; flow-log storage `devplusflhubeus2` (GRS, shared key and public access off, blob + queue private endpoints and diagnostics) |
| Azure Monitor Private Link Scope | `ampls.tf` | Single AMPLS + private endpoint for the workspace (access modes `Open`); keep exactly one per DNS namespace |
| Diagnostic settings | `diagnostics.tf` | Hub VNet, the three NSGs and the NAT public IP log to the hub workspace |
| Centralized Network Watchers | `network-watchers.tf` | One watcher per (subscription, region) pair used by any spoke — Azure allows only one per region per subscription, so they're shared infrastructure owned by the hub |

## Network layout

Hub VNet address space: `10.100.0.0/16`

| Subnet | CIDR | Purpose |
|---|---|---|
| `snet-devplus-github-runners` | `10.100.5.0/24` | Self-hosted GitHub Actions runners (`GitHub.Network` delegation) |
| `snet-devplus-cloudflared` | `10.100.6.0/28` | cloudflared container instance (`Microsoft.ContainerInstance` delegation) |
| `snet-devplus-private-endpoints` | `10.100.7.0/24` | Private endpoints (3 Key Vaults, flow-log storage blob + queue, AMPLS); NSG admits only `VirtualNetwork` TCP 443 inbound |

The runner and cloudflared subnets route outbound through the NAT gateway, giving a single static egress IP.

## Key Vaults

One region-less vault per environment (`kv-devplus-{dev,stage,prod}`), each in its own `devplus-keyvault-{env}` resource group: prod in the hub (prod) subscription, dev and stage in their own subscriptions. All three private endpoints sit in the hub's private-endpoint subnet:

- **Private-endpoint only** — `public_network_access_enabled = false`, network ACLs default-deny. Reachable from the hub and all peered spokes via the `privatelink.vaultcore.azure.net` zone; the deploy workflow reads secrets over the private endpoint from VNet-injected runners. Portal data-plane access requires being on the private network (e.g. via the Cloudflare tunnel).
- **RBAC authorization** (no legacy access policies):
  - Each environment's CI/CD service principal → `Key Vault Secrets User` (read-only) on its own vault only.
  - `founders` group → `Key Vault Administrator` on all vaults.
  - The prod CI/CD service principal (deployment identity) → `Key Vault Administrator` on all three vaults (so secrets can be managed via Terraform).
- **Purge protection** + 90-day soft delete; `kv-devplus-prod` also has `prevent_destroy` (it holds the CI/CD bootstrap secrets).
- **Audit logging** — `AuditEvent` + metrics to the Log Analytics workspace.

## Centralized DNS zones

The hub owns these private DNS zones (`dns_zones` in `variables.tf`) and links them to the hub VNet and every spoke (no `privatelink.azurecr.io` — images come from GHCR, not ACR):

- `privatelink.blob.core.windows.net`
- `privatelink.queue.core.windows.net`
- `privatelink.vaultcore.azure.net`
- `privatelink.cognitiveservices.azure.com`
- `privatelink.eastus2.azmk8s.io`
- `privatelink.centralus.azmk8s.io`
- `privatelink.postgres.database.azure.com`
- Azure Monitor private link (AMPLS, `ampls.tf`): `privatelink.monitor.azure.com`, `privatelink.oms.opinsights.azure.com`, `privatelink.ods.opinsights.azure.com`, `privatelink.agentsvc.azure-automation.net`

Exception: the `*.azmk8s.io` zones are linked to the hub VNet only (`local.spoke_link_zones` in `locals.tf` skips them for spokes). Core passes them to AKS as the private cluster's `private_dns_zone_id`, and the hub must not add its own link to the cluster VNet.

## Centralized Network Watchers

Azure allows **one Network Watcher per region per subscription**, which makes the watcher shared infrastructure — like the DNS zones, it's owned by the hub so all products' spokes can use it:

- Derived from the `spokes` variable: one watcher per distinct (subscription, region) pair, named `nw-devplus-{env}-{location_short}` (e.g. `nw-devplus-stage-cus`) in a hub-managed `devplus-network-{env}` resource group per subscription. prod/eastus2 is covered by the hub's own watcher (`nw-devplus-hub-eus2`, `monitoring.tf`).
- Spoke repos do **not** create watchers; they attach their VNet flow logs to the hub-managed watcher via a data source (see the `network_watchers` output for name + resource group per env-region).
- **Bootstrapping a new (env, region):** a new spoke's flow log needs the watcher to exist, but the spoke can only be registered in `spokes` once its VNet exists. Break the cycle with `network_watcher_regions` — add the (env, region) pair there first, apply (watcher created), stand up the spoke, then add it to `spokes`. The `network_watcher_regions` entry becomes redundant once the spoke is registered (same key, same resource) and can be removed or left in place.

## Security & SOC 2 posture

Controls in place:

| Area | Control |
|---|---|
| Access control | Azure RBAC everywhere; Key Vaults use RBAC authorization with per-environment least privilege (CI/CD principals are read-only and scoped to their own env's vault) |
| Network | Key Vaults have no public data plane (private endpoints + default-deny ACLs); CI reads secrets only from runners inside the hub VNet; all NSGs deny inbound by default; no spoke-to-spoke connectivity (environment isolation); private DNS for all PaaS endpoints |
| Secrets handling | Tunnel token and Atlas credentials live in the prod Key Vault and are fetched at deploy time (masked, never stored in GitHub); ACI secret passed via `secure_environment_variables` |
| Authentication | OIDC / Entra ID auth for Azure providers and the state backend (`use_azuread_auth` — no storage account keys); no static cloud credentials anywhere in the repo |
| Audit logging | Key Vault `AuditEvent` logs + metrics to Log Analytics, 365-day retention; ACI container, NSG, hub VNet, NAT public IP and flow-log storage (blob/queue) diagnostics to the same workspace; Sentinel onboarded on it |
| Alerting | `cloudflared` CPU metric alert (average `CpuUsage` > 400 millicores over 15 minutes, severity 3) to the `founders` action group (`founders@devplus.io`) |
| Network audit | VNet flow log on the hub VNet (tunnel, runner, and Key Vault PE traffic) with traffic analytics into Log Analytics, 90-day retention; spoke VNets have their own flow logs (spoke repo) attached to the hub-managed Network Watchers |
| Data protection | Encryption at rest via Microsoft-managed platform keys (hub CMKs were removed 2026-09-20, see `docs/DEFENDER-FINDINGS.md`); TLS-only paths (NSGs admit 443 only on inbound rules); Key Vault purge protection + 90-day soft delete |
| Supply chain | Pinned cloudflared image version; workflow actions pinned by commit SHA; Checkov and tflint installed without marketplace actions; GitHub runner subnet registered via first-party `GitHub.Network` integration |
| Asset inventory | Every taggable resource carries the `local.common_tags` schema from `locals.tf` (`project`, `environment`, `region`, `owner`, `data-classification`, `managed-by`). Terraform enforces them, so an out-of-band tag edit reverts on the next apply. The one `ignore_changes = [tags]` is on `azapi_resource.github_network_settings`, because the `GitHub.Network` RP writes a `GitHubId` tag that `outputs.tf` reads. Resources the hub owns in the dev/stage subscriptions or outside `eastus2` override `environment`/`region` via `local.env_tags` |

## Prerequisites

- Spoke VNets must exist before applying — looked up by naming convention `vnet-{prefix}-{env}-{location_short}` in resource group `{prefix}-{env}-{location_short}`, where `prefix` is set per spoke (e.g. `infrasync`) and defaults to `spoke_prefix` (`devplus`).
- The deploying identity needs:
  - `Network Contributor` on the dev and stage subscriptions (cross-subscription peering and DNS links).
  - `Contributor` on the `devplus-keyvault-{dev,stage}` resource groups and `Microsoft.Authorization/roleAssignments/write` (Owner or User Access Administrator) at the Key Vault scopes — the dev/stage central vaults now live in their own subscriptions. Full cross-subscription permission list and apply ordering: `docs/OWNERSHIP-AND-RUNBOOKS.md`.
  - In practice `devplus-cicd-infra` grants the prod CI/CD service principal `Owner` on the prod, dev and stage subscriptions (`owner_prod`, `hub_prod_owner_dev`, `hub_prod_owner_stage`), which covers all of the above, plus `Storage Blob Data Contributor` on the state account.
- OIDC federated credential on the prod app registration for `DevPlusInc/devplus-hub-infra` (environment `prod`, defined in `devplus-cicd-infra`), plus `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` variables on the repo's `prod` environment.
- The `devplus-runners` runner group (VNet-injected runners on this repo's network settings) available to this repo.
- Secrets in the prod Key Vault (`kv-devplus-prod`): `hub-mongodbatlas-client-id`, `hub-mongodbatlas-client-secret`, `hub-cloudflare-tunnel-token`.

## State & providers

- Backend: Azure Storage (`backend.tf`): account `devplusstateprod` (resource group `devplus-tf-base-infra`, prod subscription, owned by `devplus-base-infra`), container `tfstate`, key `hub.tfstate`, Entra ID auth (no storage keys).
- Providers (`providers.tf`): `azurerm` (default = prod with `storage_use_azuread = true`, aliases `dev`/`stage`), `azapi` (GitHub network settings, VNet flow log), `mongodbatlas`.
- Tooling: OpenTofu `>= 1.9.0` (CI pins `1.11.5` in `deploy.yml`); `azurerm ~> 4.65`, `azapi ~> 2.0`, `mongodbatlas ~> 2.9` (`versions.tf`), exact versions in the committed `.terraform.lock.hcl`.

This module does not use Terraform workspaces — it is a single deployment.

## Usage

Deployment is automated via GitHub Actions (`.github/workflows/deploy.yml`), triggered only by pushes to `main` and manual `workflow_dispatch`; there is no `pull_request` trigger, so the plan job's PR-comment step never runs and PRs get no plan. Each run executes `checkov`, `lint` (`tofu fmt -check`, `tofu validate`, `tflint`) and `plan` in parallel; `deploy` needs all three, re-plans and applies in the `prod` environment (required reviewers: the Founders team). A `workflow_dispatch` run applies from whatever ref it was dispatched on, not only `main`. No plan artifact is saved, because a binary plan embeds the secrets. The workflow runs on the `devplus-runners` group (VNet-injected runners in the hub) and fetches `TF_VAR_*` secrets from the prod Key Vault over its private endpoint (masked, never stored in GitHub).

To run locally:

```bash
az login   # identity per Prerequisites above

export TF_VAR_cloudflare_tunnel_token=...
export TF_VAR_mongodbatlas_client_id=...
export TF_VAR_mongodbatlas_client_secret=...

tofu init
tofu plan
tofu apply
```

(Locally you can also pull the values from `kv-devplus-prod` via `az keyvault secret show` if you're on the private network. A local plan needs that private path anyway: the provider reads the flow-log storage account's data plane through its private endpoint, since public access is off.)

## Adding a new spoke

Add an entry to `spokes` in `terraform.tfvars`:

```hcl
spokes = [
  # ... existing spokes ...
  { env = "prod", location = "westeurope", location_short = "weu" },              # devplus (default prefix)
  { prefix = "infrasync", env = "prod", location = "westeurope", location_short = "weu" },  # other products
]
```

`prefix` defaults to `spoke_prefix` (`devplus`); set it explicitly for other products' spokes. Then apply. Peering and DNS zone links are created automatically. The new spoke's address space must not overlap the hub (`10.100.0.0/16`) or any other peered spoke. If the new region runs AKS, add its `privatelink.{region}.azmk8s.io` zone to `dns_zones`.

## Outputs

| Output | Description |
|---|---|
| `hub_vnet_id` | Resource ID of the hub VNet |
| `github_network_settings_id` | GitHub Network Settings ID (for configuring the runner group) |
| `github_network_settings_github_id` | `GitHubId` of the network settings, pasted into GitHub's hosted compute networking config |
| `cloudflared_container_id` | Resource ID of the cloudflared container instance |
| `nat_public_ip` | NAT gateway public IP (e.g. for MongoDB Atlas / firewall allowlists) |
| `key_vault_uris` | Map of environment name to Key Vault URI (resolves privately from peered VNets) |
| `dns_zone_ids` | Map of private DNS zone name to resource ID |
| `network_watchers` | Map of env-region to hub-managed Network Watcher (name + resource group) for spoke flow logs |

## Cost (rough, monthly)

| Resource | Approx |
|---|---|
| cloudflared (ACI, 0.5 vCPU / 0.5 GB) | ~$15 |
| NAT Gateway + static public IP | ~$36 |
| Private endpoints (6 × ~$7.50: 3 Key Vaults, flow-log blob + queue, AMPLS) | ~$45 |
| Key Vaults (standard, per-operation) | ~$1 |
| Log Analytics (low volume; 365-day retention and Sentinel bill per GB, so this grows with ingest) | ~$6 |
| VNet flow log (storage + traffic analytics) | ~$8 |
| **Total** | **~$111** |

Replacing the previous VPN Gateway with a Cloudflare Tunnel removed the largest line item (~$140/mo VPN gateway).
