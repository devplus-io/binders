---
id: f63d914d-9f28-4b6f-b203-c06936ccc44d
binder: binder-mvvle32a-1
---
DevOps | terraform-devplus-tenant/README.md

Source: DevPlusInc/terraform-devplus-tenant/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/terraform-devplus-tenant, README.md).

# terraform-devplus-tenant

Terraform module for deploying a single DevPlus tenant to Kubernetes. Provisions the
WorkOS organization, optional Atlas OIDC users, optional Azure workload identity, calls
`terraform-helm-devplus-services` for the Helm release, and provisions the tenant's
Cloudflare tunnel ingress via `terraform-cloudflare-ingress`.

For local development use [Tilt](../devplus-platform/TILT.md) instead.

## What it deploys

- Kubernetes namespace and secrets, through `terraform-helm-devplus-services`: `jwt-secret`,
  `internal-api-token`, `secret-encryption-key`, `workos-api-key` / `workos-client-id` /
  `workos-cookie-password`, `infracost-secret`, `mongodb-secret` (in-cluster MongoDB) or
  `mongodb-atlas-uri` (Atlas), and `github-install-state` when `github_install_state_secret`
  is set. `ghcr-pull-secret`, `github-app-secret` and `hubspot-access-token-shared` are
  Reflector mirrors from `devplus-core-infra`, not created here.
- Helm release of the **published OCI chart** `oci://ghcr.io/devplusinc/charts/devplus`,
  pinned by `chart_version` (all DevPlus services, plus the hosted MCP service while
  `devplus_mcp_enabled` is true)
- WorkOS organization (named after `namespace`) with `tenant_sso_domain` as a verified
  domain. Its id and the environment's centralized redirect URI are passed to the chart
  (`workos.organizationId`, `workos.redirectUri`), and `admin_email` / `admin_name` become
  `adminSeed.*`. The organization starts with no members: the backend invites `admin_email`
  on first boot.
- MongoDB Atlas federated database users (when `mongodbatlas_project_id` is set):
  `<mongodbatlas_idp_id>/system:serviceaccount:<namespace>:<namespace>-app` and
  `...:executor-job-dispatcher`, each with `readWrite` + `dbAdmin` on the tenant database
  only (plus a `CLUSTER` scope when `mongodbatlas_cluster_name` is set)
- Azure workload identity (only when `manage_workload_identity = true`): user-assigned
  identity `id-devplus-<namespace>-<environment>`, federated credentials for `<namespace>-app`
  and `executor-job-dispatcher` per `aks_oidc_issuer_urls` entry, and `Cognitive Services
  OpenAI User` on `ais-devplus-<environment>`
- Cloudflare tunnel, proxied DNS CNAME and `cloudflared` Deployment (always; see
  `terraform-cloudflare-ingress`)

## Usage

```hcl
module "tenant" {
  source = "github.com/DevPlusInc/terraform-devplus-tenant?ref=main"

  namespace   = "tftest"
  environment = "dev"
  base_domain = "devplus.io" # → public host tftest.dev.devplus.io (prod omits the env segment)

  admin_email      = "admin@devplus.io"
  tenant_sso_domain = "devplus.io"

  # Deploy version: pulls the published chart at this version; the chart's appVersion
  # pins every service image to ghcr.io/devplusinc/<service>:<chart_version>.
  chart_version   = "0.1.0-dev1"
  ghcr_pull_token = var.ghcr_pull_token # the chart is a private OCI artifact

  # Secrets (bind to a SecretReference / source from Key Vault)
  workos_api_key         = var.workos_api_key
  workos_client_id       = var.workos_client_id
  workos_cookie_password = var.workos_cookie_password
  cloudflare_api_token   = var.cloudflare_api_token
}
```

`devplus-platform/terraform/tenants/devplus` is the reference caller: it also passes
`kubeconfig_path` / `kubeconfig_context`, the Atlas inputs (project id from core-infra
remote state), `github_install_state_secret` (core-infra output), and creates the Azure
identity itself, handing its client id in through `service_account_annotations`.

## Providers

The module **self-configures** its providers from its inputs (callers pass no `providers`
map), so the DevPlus executor can run it standalone for in-product tenant provisioning (the
`deploy-devplus-tenant` catalog item): azurerm from `ARM_*` env; kubernetes and helm from
in-cluster config unless `kubeconfig_path` is set; helm authenticates to `oci://ghcr.io`
with `ghcr_pull_token`; cloudflare, mongodbatlas and restapi (WorkOS Management API) from
the secret inputs.

Pins: `hashicorp/azurerm >= 4.0.0, < 5.0.0` (do not lift without a migration: the executor
resolves providers with no lock file), `hashicorp/helm >= 2.10.0`,
`hashicorp/kubernetes >= 2.23.0`, `hashicorp/random >= 3.5.0`,
`cloudflare/cloudflare >= 5.17.0`, `mongodb/mongodbatlas >= 2.9.0`,
`Mastercard/restapi = 3.0.0`.

## Versioning

`chart_version` is the single deploy input for images. It pulls the published umbrella
chart from `oci://ghcr.io/devplusinc/charts/devplus`, whose `appVersion` defaults every
service image to `ghcr.io/devplusinc/<service>:<chart_version>` — no per-service tags.
(The chart must be published at that version, e.g. via `devplus-platform/scripts/publish-chart.sh`.)

## Public host

Derived as `<namespace>.<environment>.<base_domain>` (e.g. `tftest.dev.devplus.io`); prod
serves at the bare `<namespace>.<base_domain>`. This single host feeds the chart `publicUrl`
/ OIDC issuer, `tenantHostname`, and the Cloudflare ingress (exposed as the `public_host` /
`public_url` outputs). The WorkOS redirect follows the same rule:
`https://auth.<environment>.<base_domain>/callback` (prod `https://auth.<base_domain>/callback`).
`environment` is required.

## Key variables

| Variable | Default | Description |
|----------|---------|-------------|
| `namespace` | — (required) | Tenant name = namespace = Mongo database |
| `environment` | — (required) | Env name (dev/stage/prod); part of the public host + Atlas IdP |
| `chart_version` | — (required) | Published chart version to deploy (e.g. `0.1.0-dev1`) |
| `admin_email` | (required) | First admin (`adminSeed.email`, invited to the WorkOS org) |
| `admin_name` | `Admin` | First admin display name |
| `tenant_sso_domain` | (required) | Verified domain of the WorkOS organization |
| `base_domain` | `devplus.io` | Base domain for the public host |
| `ghcr_pull_token` | `""` | GHCR read token the helm provider uses to pull the private chart; empty = no registry auth |
| `persistence_mode` | `helm` | PVC creation: `helm`, `terraform` or `external` |
| `image_pull_policy` | `Always` | Image pull policy for app images |
| `mongodb_atlas_uri` | null | Atlas URI (disables in-cluster MongoDB) |
| `service_account_annotations` / `service_account_labels` | `{}` | App ServiceAccount workload-identity wiring when the caller owns the identity |
| `manage_workload_identity` | `false` | Module creates the Azure identity itself (standalone); with `azure_resource_group_name`, `azure_location` (`eastus2`), `aks_oidc_issuer_urls`, `azure_openai_account_resource_group` (unset or `""` derives `devplus-<environment>`, `none` skips the role) |
| `devplus_mcp_enabled` | `true` | Deploy the hosted MCP service and accept AuthKit tokens |
| `authkit_issuer` | shared dev AuthKit domain | AuthKit issuer for the MCP service |
| `control_plane_enabled` | `false` | Bind job pods to the provisioner ClusterRole (staff-only tenant) |
| `kubeconfig_path` / `kubeconfig_context` | `""` / null | Empty = in-cluster config |
| `otel_enabled`, `otel_collector_endpoint`, `otel_sampler_arg` | `false`, `http://otel-collector.observability:4317`, `0.1` | OpenTelemetry env on the Go services |

## Optional secrets

- `azure_openai_endpoint` / `azure_openai_deployment` — AI assistant (auth via Entra ID, no API key)
- `infracost_api_key` — cost estimation
- `github_install_state_secret`: shared GitHub App install-state HMAC (core-infra output)
- `mongodbatlas_client_id` / `mongodbatlas_client_secret`: Atlas provider auth (federated DB
  users), with `mongodbatlas_project_id`, `mongodbatlas_idp_id` and optional `mongodbatlas_cluster_name`
- `cloudflare_account_id` / `cloudflare_zone_id`: tunnel ingress (default to the devplus.io
  account and zone; token via `cloudflare_api_token`)

## Outputs

| Output | Value |
|--------|-------|
| `public_host` | The tenant's public hostname |
| `public_url` | `https://<public_host>`, also the OIDC issuer |
| `workos_redirect_uri` | The environment's centralized WorkOS redirect URI |

## CI

`.github/workflows/ci.yml` runs OpenTofu `~> 1.11`: `tofu fmt -check -recursive`, then
`tofu init -backend=false && tofu validate` (private module clones authenticate with a
GitHub App token).

## Dependencies

- `terraform-helm-devplus-services` (Helm release + secrets), pinned by commit in `main.tf`
- `terraform-cloudflare-ingress` (tunnel ingress), pinned by commit in `main.tf`
