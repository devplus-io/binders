---
id: f036ba66-9a20-4ba0-adb6-d57f5d24d7fe
binder: binder-mvvle32a-1
---
DevOps | terraform-helm-devplus-services/README.md

Source: DevPlusInc/terraform-helm-devplus-services/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/terraform-helm-devplus-services, README.md).

# terraform-helm-devplus-services

Terraform module that deploys DevPlus to Kubernetes via Helm. Creates namespace,
secrets, PVCs, and a Helm release of the published OCI chart. Used by
`terraform-devplus-tenant`, which pins it by commit.

For local development use [Tilt](../devplus-platform/TILT.md) instead.

## What it deploys

- Kubernetes namespace (when `create_namespace`, default true)
- Secrets:
  - `jwt-secret` (key `secret`), `internal-api-token` (key `token`) and
    `secret-encryption-key` (key `key`, base64 of a 32-byte key that opentofu-server needs
    to encrypt state at rest): each generated unless its value is passed in
  - `workos-api-key`, `workos-client-id`, `workos-cookie-password` (key `api-key`), created
    here so the release can depend on them (the pods mount them)
  - `mongodb-secret` (in-cluster MongoDB only) or `mongodb-atlas-uri` (key `uri`, when
    `mongodb_atlas_uri` is set)
  - `github-install-state` (key `state-secret`, when `github_install_state_secret` is set;
    also turns on `githubInstallState.enabled`)
  - `infracost-secret`, `openai-secret` (key `api-key`) and `oidc-keys`, only when their
    inputs are set
  - `ghcr-pull-secret`, only when `ghcr_token` is set. Otherwise the chart's default
    `ghcr-pull-secret` is the Reflector mirror from `devplus-core-infra`, like
    `github-app-secret` (never created here; the chart is pointed at it)
- PVC `mongodb-pvc` (when `persistence_mode=terraform` and MongoDB is in-cluster)
- Helm release `release_name` (default `devplus`) of `chart_repository` (default
  `oci://ghcr.io/devplusinc/charts/devplus`) at `chart_version`, with `atomic` and
  `cleanup_on_fail` so a failed apply leaves no half-installed release

## Usage

```hcl
module "helm-deployment" {
  source = "github.com/DevPlusInc/terraform-helm-devplus-services?ref=<commit>"

  providers = {
    helm       = helm
    kubernetes = kubernetes
  }

  namespace     = "devplus"
  chart_version = "0.1.0-dev1"
  public_url    = "https://devplus.dev.devplus.io"

  workos_api_key         = var.workos_api_key
  workos_client_id       = var.workos_client_id
  workos_cookie_password = var.workos_cookie_password
}
```

## Images

The module takes no image inputs. The chart's `image.registry` (`ghcr.io/devplusinc`) and
`image.tag` (empty = the chart's `appVersion`, set to the release version at publish time)
pin every service to `ghcr.io/devplusinc/<image>:<chart_version>`:

| GHCR image name | K8s resource |
|-----------------|--------------|
| `backend` | Service `backend:8080` |
| `executor` | Deployment `executor-job-dispatcher` (no Service) |
| `governance-service` | Service `governance:8084` |
| `opentofu-server` | Service `opentofu-server:8082` |
| `oidc-service` | Service `oidc-service:8081` |
| `agent-service` | Service `agent-service:8083` |
| `frontend` | Service `frontend:80` |
| `devplus-mcp` | Service `devplus-mcp:8085` (only with `devplusMcp.enabled`) |

## Inputs

| Variable | Default | Description |
|----------|---------|-------------|
| `chart_version` | (required) | Published chart version to deploy |
| `workos_api_key`, `workos_client_id`, `workos_cookie_password` | (required) | WorkOS runtime secrets |
| `namespace` / `release_name` | `devplus` / `devplus` | Target namespace and Helm release |
| `chart_repository` | `oci://ghcr.io/devplusinc/charts/devplus` | OCI chart location |
| `create_namespace` | `true` | Create the namespace |
| `base_domain` / `public_url` | `devplus.io` / null | `publicUrl` (OIDC issuer) = `public_url`, else `https://<namespace>.<base_domain>` |
| `tenant_hostname` | null | `tenantHostname` for signed GitHub install states (chart derives it from `publicUrl` when null) |
| `persistence_mode` | `terraform` | `terraform`, `helm` or `external` |
| `mongodb_pvc_size`, `mongodb_pvc_storage_class_name` | `5Gi`, `standard` | Used with `persistence_mode=terraform` |
| `mongodb_deploy`, `mongodb_database`, `mongodb_replica_set` | `true`, `devplus`, `rs0` | In-cluster MongoDB |
| `create_mongodb_secret`, `mongodb_username`, `mongodb_password` | `true`, `admin`, generated | `mongodb-secret` |
| `mongodb_atlas_enabled`, `mongodb_atlas_uri` | `false`, null | Atlas OIDC connection |
| `service_account_create`, `service_account_annotations`, `service_account_labels` | `false`, `{}`, `{}` | App ServiceAccount for workload identity |
| `create_jwt_secret`, `jwt_secret` | `true`, generated | `jwt-secret` |
| `create_internal_api_token`, `internal_api_token` | `true`, generated | `internal-api-token` (executor to backend internal API) |
| `create_state_encryption_key`, `state_encryption_key` | `true`, generated | `secret-encryption-key` |
| `github_install_state_secret` | null | `github-install-state` |
| `azure_openai_endpoint`, `azure_openai_deployment` | `""`, `gpt-4.1` | `azureOpenai.*` (only rendered when the endpoint is set) |
| `openai_api_key`, `infracost_api_key` | null | `openai-secret`, `infracost-secret` |
| `oidc_private_key`, `oidc_public_key` | null | `oidc-keys` (set both or neither) |
| `ghcr_token`, `ghcr_username` | null, `devplus` | Module-owned `ghcr-pull-secret` |
| `admin_seed` | null | `adminSeed.*` (object with email, name, password) |
| `image_pull_policy` | null | `imagePullPolicy` (null = chart default) |
| `cache_enabled`, `cache_host_path` | `false`, `/var/lib/devplus/cache` | Executor provider cache on a hostPath |
| `otel_enabled`, `otel_collector_endpoint`, `otel_sampler_arg` | `false`, `http://otel-collector.observability:4317`, `0.1` | `otel.*` |
| `helm_wait`, `helm_timeout_seconds` | `true`, `600` | Release wait and timeout |
| `helm_set`, `helm_values` | `{}`, `[]` | Extra `--set` values (merged last, so they win) and values documents |

## Outputs

`namespace`, `release_name`, `mongodb_username`, `mongodb_password` (sensitive),
`jwt_secret` (sensitive).

## Notes

- `persistence_mode=terraform` (default) creates PVCs via the Kubernetes provider.
- `persistence_mode=helm` lets the chart create PVCs (better for WaitForFirstConsumer);
  `external` expects them to exist already.
- The chart includes ClusterRole/ClusterRoleBinding (oidc-service TokenReview, and the
  provisioner binding when `controlPlane.enabled`); cluster credentials must be allowed to
  create cluster-scoped RBAC resources.
- Providers: `hashicorp/helm >= 2.10.0` and `hashicorp/kubernetes >= 2.23.0` are declared
  (`hashicorp/random` is used undeclared). The release uses the helm provider 3.x
  `set = [...]` list form; the `devplus-platform` tenant root locks helm 3.1.1.

## CI

`.github/workflows/ci.yml` runs OpenTofu `~> 1.9`: `tofu fmt -check -recursive`, then
`tofu init -backend=false && tofu validate`.
