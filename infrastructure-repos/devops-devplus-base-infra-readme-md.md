---
id: 07b5892c-8d70-4978-8e05-8d53102bcaba
binder: binder-mvvle32a-1
folder: folder-mvw0u6iq
---
DevOps | devplus-base-infra/README.md

Source: DevPlusInc/devplus-base-infra/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-base-infra, README.md).

# devplus-base-infra

OpenTofu state storage for DevPlus Azure infrastructure. Creates the Azure Storage Account and blob container used by all infrastructure layers.

## What it creates

All in the prod subscription (`a7242621-0d31-4681-9154-d1d0e9d7a4e6`, `var.subscription_id`):

| Resource | Name | Notes |
| --- | --- | --- |
| Resource group | `devplus-tf-base-infra` | `eastus` (`var.location`) |
| Storage account | `devplusstateprod` | Standard, `GZRS`, TLS 1.2 minimum, HTTPS only, blob versioning on, no public blob access |
| Blob container | `tfstate` | private |
| Private endpoint | `pe-devplus-state-prod-blob` | `blob` subresource, see below |
| Management locks | `lock-devplus-state-rg`, `lock-devplus-state` | `CanNotDelete` on the resource group and the storage account |
| Role assignment | Storage Blob Data Owner | for the state admins Entra group (`var.state_admin_group_object_id`, default `362ddfce-f160-468c-9bb6-712a5634e42f`) |

### State keys in `tfstate`

| Key | Written by |
| --- | --- |
| `core.tfstate` (one state per workspace: `dev`, `stage`, `prod`) | `devplus-core-infra` |
| `hub.tfstate` | `devplus-hub-infra` |
| `ghidentity.tfstate` | `devplus-cicd-infra` |
| `github.tfstate` | `devplus-github-settings` |
| `tenants/<name>.tfstate` | `devplus-platform` (`terraform/tenants/devplus`, key set per tenant by `_deploy-environment.yml`) |

## Network access

The state storage account is reachable only through a blob private endpoint:

- public network access is disabled on the storage account
- storage firewall default action is `Deny`, with no trusted-service bypass (`bypass = ["None"]`)
- a private endpoint is created for the `blob` subresource
- the endpoint is registered in the `privatelink.blob.core.windows.net` private DNS zone

By default this stack attaches the private endpoint to the prod hub network:

| Setting | Default |
| --- | --- |
| Private endpoint location | `eastus2` |
| Network resource group | `devplus-hub-eus2` |
| Virtual network | `vnet-devplus-hub-eus2` |
| Private endpoint subnet | `snet-private-endpoints` |
| Blob private DNS zone | `privatelink.blob.core.windows.net` in `devplus-hub-eus2` |

The subnet and private DNS zone resource IDs are built from these defaults. Override `private_endpoint_subnet_id` or `blob_private_dns_zone_id` to pin exact resource IDs, or override `private_endpoint_location`, `private_endpoint_network_resource_group_name`, `private_endpoint_virtual_network_name`, `private_endpoint_subnet_name`, `blob_private_dns_zone_resource_group_name`, or `blob_private_dns_zone_name` if hub networking is renamed.

## Tagging

Every taggable resource carries the `local.common_tags` schema from `locals.tf` (`project`, `environment`, `region`, `owner`, `data-classification`, `managed-by`), identical in shape to `devplus-hub-infra` so the asset inventory reads the same across repos. Terraform enforces them — there are no `ignore_changes = [tags]` opt-outs, so an out-of-band tag edit reverts on the next apply.

The resource group, storage account, and private endpoint are taggable resources here; the blob container, the two management locks, and the role assignment have no `tags` attribute in the provider schema.

## Prerequisites

- Azure tenant admin signed in via `az login`
- Network access to the hub private endpoint subnet, or an apply runner that can resolve and reach `devplusstateprod.blob.core.windows.net` through the private endpoint after the first apply
- OpenTofu installed locally

## Deployment

This layer runs with local state — it creates the storage account that all other layers use as their backend. Shared access keys are disabled (`shared_access_key_enabled = false`), so all data-plane operations use Entra ID auth. After public network access is disabled, future `tofu plan` and `tofu apply` runs must execute from a network path that can reach the private endpoint.

### 1. Init and apply
```bash
tofu init
tofu plan
tofu apply
```

> **Note:** Creating the `tfstate` container uses data-plane access. The state admins group's Storage Blob Data Owner assignment is created in the same apply and may not have propagated yet. If apply fails on the container with a 403, grant yourself `Storage Blob Data Contributor` on the storage account, wait ~1 min for propagation, then re-apply:
> ```bash
> az role assignment create \
>   --assignee $(az ad signed-in-user show --query id -o tsv) \
>   --role "Storage Blob Data Contributor" \
>   --scope $(az storage account show --name devplusstateprod --query id -o tsv)
> ```

### 2. Verify
```bash
az storage container list --account-name devplusstateprod --auth-mode login --output table
az storage account show --name devplusstateprod --resource-group devplus-tf-base-infra --query publicNetworkAccess -o tsv
az network private-endpoint-connection list --name devplusstateprod --resource-group devplus-tf-base-infra --type Microsoft.Storage/storageAccounts -o table
```

## Outputs

| Name | Description |
| --- | --- |
| `resource_group_name` | Resource group containing the state storage |
| `storage_account_name` | State storage account name |
| `storage_account_id` | State storage account resource ID |
| `container_name` | Blob container for state files |
| `private_endpoint_id` | Blob private endpoint resource ID |
| `private_endpoint_subnet_id` | Subnet the private endpoint is attached to |
