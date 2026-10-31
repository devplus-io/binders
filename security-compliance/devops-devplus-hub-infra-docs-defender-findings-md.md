---
id: 4582a942-9bae-434a-8fdd-92018577567e
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevOps | devplus-hub-infra/docs/DEFENDER-FINDINGS.md

Source: DevPlusInc/devplus-hub-infra/docs/DEFENDER-FINDINGS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-hub-infra, docs/DEFENDER-FINDINGS.md).

# Microsoft Defender for Cloud — findings disposition

Status of Defender for Cloud findings against resources defined in this repo.
First written 2026-06-06 (remediations landed in the `release/fixfindings`
branch) and revised 2026-09-20 for the Drata follow-ups (hub CMK decommission,
cloudflared CPU alert). Items below are the ones intentionally **not** fixed,
with rationale.

## Won't fix / false positives

| Finding | Resource(s) | Rationale |
|---|---|---|
| Flow logs should be configured for every network security group (policy `c251913d-7d24-4958-af87-478ed3b9ba41`) | all hub NSGs | NSG flow logs are platform-deprecated: creation blocked since 2025-06-30, full retirement 2027-09-30. The hub VNet flow log (`monitoring.tf`) covers every subnet and is Microsoft's sanctioned replacement; the policy does not recognize VNet flow logs and is itself being deprecated. Cannot go green. |
| Network Watcher should be enabled / Deploy network watcher when virtual networks are created | `vnet-devplus-hub-eus2` | False positive. The built-in policy looks for a watcher in a resource group named `NetworkWatcherRG`; ours intentionally live in `devplus-hub-eus2` and `devplus-network-{env}` (see `network-watchers.tf` header for the design). Watchers exist for every (subscription, region) pair in use. |
| Audit flow logs configuration for every virtual network | `vnet-devplus-hub-eus2` | The VNet flow log exists and is enabled. Believed to be assessment staleness — re-check after the next Defender refresh; escalate to support if it persists. |
| Saved-queries in Azure Monitor should be saved in customer storage account | `log-devplus-hub-eus2` | Legacy recommendation; no saved queries hold sensitive content and Microsoft no longer recommends the customer-storage pattern. |
| Ensure Storage logging is enabled for Queue service (checkov `CKV_AZURE_33`) | flow-log storage account | Classic Storage Analytics logging is legacy and would require shared-key data-plane access, which is disabled on this account. Modern diagnostic settings on the blob and queue services (`monitoring.tf`) provide the same audit trail in Log Analytics. Suppressed in `.checkov.yaml`. |
| Ensure private endpoint is configured to key vault (checkov `CKV2_AZURE_32`) | key vaults | False positive. Private endpoints exist for every central secrets vault (`keyvault.tf`); checkov's graph resolver can't link `azurerm_private_endpoint` to `azurerm_key_vault` across `for_each`. Suppressed in `.checkov.yaml`. |
| Ensure storage for critical data is encrypted with customer-managed key (checkov `CKV2_AZURE_1`) | flow-log storage account | Drata Test 254 direction is to eliminate hub CMKs rather than add key expiration dates. The account uses Microsoft-managed platform keys, which Azure rotates and which avoid Key Vault key-expiration calendar-bomb outages. Suppressed in `.checkov.yaml`. |

## Out of scope for this repo

| Finding | Where it belongs |
|---|---|
| All findings on `devplusstateprod` (private link, VNet rules, GRS, infrastructure encryption, CMK, SAS expiration, shared key) | `devplus-base-infra` — the Terraform state storage account is defined there, this repo only consumes it as a backend. |
| GitHub posture findings (two-reviewer approval, code scanning, Copilot public-code matching) | GitHub org settings / `devplus-github-settings`. |
| AKS / spoke resource findings (`aks-devplus-*`, `MC_*` node resource groups) | `devplus-core-infra` (owns the spoke VNets and AKS clusters; the legacy `infrasync` spokes are no longer peered). |
| Drata Test 254 failures for `kv-devplus-prod`, `kv-devplus-cmk-prod-fkx`, and `kv-devplus-cus-prod-lt7` | Product/spoke repos or out-of-band keys. This repo creates the central secrets vault `kv-devplus-prod`, but defines no key resources inside it. The other two were core's CMK vaults (`cmk.tf` and `modules/region/cmk.tf` naming), and both files were removed from core on 2026-09-20 (`c77c41e`). |
| Drata Test 118 failures for prod AKS clusters / node VMs | `devplus-core-infra` (AKS node CPU alerts in `metric-alerts.tf`). The hub-owned cloudflared ACI alert is defined here as `azurerm_monitor_metric_alert.cloudflared_cpu`, with `target_resource_type` / `target_resource_location` set because Drata only counts alerts that carry them. |

## Hub CMK decommission

Hub flow-log storage (`devplusflhubeus2`) and the cloudflared container group
now use Azure/Microsoft-managed platform keys instead of hub-managed CMKs.
This removes the `kv-devplus-cmk-eus2` key consumers and the `key-flowlogs` /
`key-cloudflared` Terraform resources entirely for Drata Test 254. `cmk.tf` is
gone, including the `kv-devplus-cmk-eus2` vault, its private endpoint,
diagnostic setting and role assignments, and the `id-cloudflared` identity; the
`id-flowlogs-devplus-hub-eus2` identity stays (`monitoring.tf`) because the flow
log writes to storage with Entra ID, shared key being disabled.

Operational rationale: adding future expiration dates to active encryption
CMKs creates calendar-bomb outage risk if the date lapses. Microsoft-managed
platform keys are rotated by Azure and avoid the repo needing key expiration
dates, key-unwrap firewall exceptions, or manual CMK rotation runbooks for
these hub resources.

## Operational notes after the lockdown

- **Log Analytics queries** (`log-devplus-hub-eus2`) only resolve from the
  hub/spoke network via the AMPLS private endpoint (`ampls.tf`). Portal/CLI
  access from outside the VNets (e.g. a laptop not routing through the
  Cloudflare tunnel with private DNS) will fail with the workspace's
  `internet_query_enabled = false`.
- Keep a **single AMPLS** — a second scope with private endpoints in linked
  VNets overrides the Azure Monitor private DNS zones and breaks resolution.
