---
id: 38338d0b-2457-4f22-9e20-d194c98be838
binder: binder-mvvle32a-1
---
DevOps | devplus-cicd-infra/README.md

Source: DevPlusInc/devplus-cicd-infra/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-cicd-infra, README.md).

# devplus-cicd-infra

OpenTofu configuration for the Entra ID (Azure AD) identities GitHub Actions uses to deploy DevPlus across dev, stage, and prod, plus the prod DevPlus product connector principal. One root module, one state, no workspaces.

## What it creates

### GitHub Actions identities (one per environment)

- **App registrations + service principals**: `devplus-github-actions-dev`, `devplus-github-actions-stage`, `devplus-github-actions-prod` (`main.tf`, `for_each = var.environments`).
- **Federated identity credentials**: OIDC federation from `https://token.actions.githubusercontent.com` (audience `api://AzureADTokenExchange`), so there are no secrets to rotate:

| Credential | App | Subject | Repos |
|---|---|---|---|
| `github-<repo>-<env>` | each env's app | `repo:DevPlusInc/<repo>:environment:<env>` | `var.github_repos`: `devplus-core-infra`, `devplus`, `devplus-public-website`, `devplus-platform` |
| `github-devplus-hub-infra-prod` | prod | `repo:DevPlusInc/devplus-hub-infra:environment:prod` | `devplus-hub-infra` (deploys from the prod environment only) |
| `github-<repo>-ci` (flexible credential) | dev | `claims['sub'] matches 'repo:DevPlusInc/<repo>:*'` | `var.ci_build_repos`: `devplus-agent-service`, `devplus-backend`, `devplus-executor`, `devplus-frontend`, `devplus-governance-service`, `devplus-oidc-service`, `devplus-opentofu-server`, `devplus-platform`, `terraform-devplus-tenant` |

Environment-based credentials serve workflows that run in a GitHub Environment and read `vars.AZURE_CLIENT_ID` from it (`devplus-core-infra` and `devplus-hub-infra` `deploy.yml`, `devplus-platform` `_deploy-environment.yml`). The flexible credentials serve CI/build workflows that call `azure/login` with the dev client ID on branch pushes and pull requests, with no GitHub Environment; they assume the dev service principal and inherit its roles.

Azure AD caps an application at 20 federated credentials, and a flexible credential takes a single `matches` clause, so each CI repo gets exactly one wildcard credential. Current totals: dev 13, stage 4, prod 5. Adding a repo to either list adds a credential to the dev app.

### Role assignments

| Principal | Role | Scope | Resource |
|---|---|---|---|
| Each env SP | Owner | Its own subscription | `owner_dev`, `owner_stage`, `owner_prod` |
| prod SP | Owner | dev and stage subscriptions (hub peering, DNS zone VNet links) | `hub_prod_owner_dev`, `hub_prod_owner_stage` |
| dev, stage SPs | Reader | Hub RG `devplus-hub-eus2` (prod subscription) | `hub_reader` |
| dev, stage SPs | Role Based Access Control Administrator | Hub RG | `hub_rbac_admin` |
| dev, stage SPs | Log Analytics Contributor | Hub RG (diagnostics and Container Insights to the hub workspace) | `hub_log_analytics_contributor` |
| dev, stage SPs | Private DNS Zone Contributor | Hub RG (join private endpoints to the hub zones) | `hub_private_dns_contributor` |
| dev, stage SPs | Monitoring Contributor | prod (hub) subscription (Traffic Analytics DCR/DCE) | `hub_monitoring_contributor` |
| All three SPs | Storage Blob Data Contributor | State account `devplusstateprod` | `state_blob_access` |

### DevPlus product connector principal (prod)

The Azure connector "DevPlus - Production" in the prod DevPlus app authenticates as this principal through workload-identity federation.

- App registration `DevPlus (de58262c)` + service principal.
- Federated credential `devplus-de58262c`: issuer `https://app.devplus.io` (the prod tenant's oidc-service), subject `devplus:de58262c`.
- Roles on the prod subscription: Contributor; Role Based Access Control Administrator constrained by an ABAC condition (`local.connector_rbac_condition`) that allows writing and deleting role assignments only for the role IDs in `local.connector_assignable_role_ids` (Key Vault Secrets User `4633458b-17de-408a-b874-0445c86b69e6`, Cognitive Services OpenAI User `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`); Key Vault Secrets User on `kv-devplus-prod` (RG `devplus-keyvault-prod`).
- Any new role that tenant provisioning (or another DevPlus feature) starts assigning must be added to that allowlist, or those writes fail with `AuthorizationFailed`. azurerm treats `condition` as immutable, so changing the allowlist replaces that one role assignment.
- The principal was created by hand and adopted with the one-shot import blocks in `imports.tf`; they are inert once applied and safe to delete.

## Prerequisites

- OpenTofu (`required_version >= 1.6.0`; the lock file is from `registry.opentofu.org`). Use the same version as the other stacks on the shared backend (core-infra and hub-infra CI pin 1.11.5): state written by a newer tofu cannot be read by an older one.
- Azure CLI authenticated as a principal that can:
  - create Entra ID app registrations, service principals and federated credentials
  - assign roles on the dev, stage, and prod subscriptions
- Data-plane access and a network path to the state storage account created by [devplus-base-infra](../devplus-base-infra): it has public network access disabled and is reachable only through its private endpoint in the hub VNet.

## Subscriptions

| Environment | Subscription ID |
|-------------|----------------|
| dev | `53a006e0-623b-44f1-84ce-5889e9d8e8da` |
| stage | `61e4e1a9-ceef-4637-a794-aaee7d480a9e` |
| prod | `a7242621-0d31-4681-9154-d1d0e9d7a4e6` |

## Usage

There is no CI workflow in this repo; apply from a workstation. All inputs have defaults (`variables.tf`), and `*.tfvars` is gitignored.

```bash
az login
tofu init
tofu plan
tofu apply
```

## State

Azure Storage backend (`backend.tf`): account `devplusstateprod`, container `tfstate`, key `ghidentity.tfstate`, RG `devplus-tf-base-infra` in the prod subscription, Entra ID auth (`use_azuread_auth = true`).

## Inputs

| Name | Default | Description |
|------|---------|-------------|
| `prefix` | `devplus` | Naming prefix |
| `github_org` | `DevPlusInc` | GitHub organization in credential subjects |
| `github_repos` | see table above | Repos that deploy through a GitHub Environment |
| `ci_build_repos` | see table above | Repos whose CI assumes the dev identity on branch/PR |
| `hub_resource_group_name` | `devplus-hub-eus2` | Hub RG for the cross-subscription grants |
| `environments` | the three subscriptions above | Env name to subscription ID |

## Outputs

These cover the GitHub Actions identities only, not the connector principal.

| Name | Description |
|------|-------------|
| `application_client_ids` | App registration client IDs per environment |
| `service_principal_object_ids` | Service principal object IDs per environment |
| `tenant_id` | Azure AD tenant ID |
| `subscription_ids` | Subscription IDs per environment |
