---
id: 04685bad-0514-415f-81bb-afcff4797b51
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/terraform/README.md

Source: DevPlusInc/devplus-platform/terraform/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, terraform/README.md).

# DevPlus Tenant Deployments

> **Deploy model: see [`docs/CICD.md`](../docs/CICD.md) and
> [`docs/RELEASE.md`](../docs/RELEASE.md).** Deploys are driven by a single
> published chart version (`-var chart_version=X`); the old per-service
> `image-tags.json` / `image_tags` map is gone. The normal path is
> `Actions → Deploy Tenants` (or `Release` with `deploy`), which runs
> `_deploy-environment.yml` against this directory.

Multi-tenant OpenTofu setup. One root (`tenants/devplus`) deploys every
Terraform-managed tenant: a tfvars file per tenant, a state key per tfvars file,
and a workspace per environment.

## Structure

```
terraform/
└── tenants/
    └── devplus/              # The tenant root (the `tenants` input of the workflows)
        ├── backend.tf        # Remote state: devplusstateprod/tfstate, key tenants/devplus.tfstate
        ├── providers.tf      # azurerm only; the tenant module configures its own providers
        ├── main.tf           # Workload identity + federated creds + AOAI role, then terraform-devplus-tenant
        ├── variables.tf
        ├── outputs.tf
        └── environments/
            ├── dev.tfvars    # The `app` tenant, one file per environment
            ├── stage.tfvars
            ├── prod.tfvars   # control_plane_enabled = true
            ├── helder.tfvars # Per-developer dev tenants
            ├── aaron.tfvars
            ├── saamer.tfvars
            ├── test-dev.tfvars    # The `test` canary (quality gate), per environment
            ├── test-stage.tfvars
            └── test-prod.tfvars
```

**State isolation**: `environments/<env>.tfvars` (the `app` tenant) uses the
default key `tenants/devplus.tfstate`. Any other tfvars file gets its own key,
`tenants/<tfvars_name>.tfstate`, through
`tofu init -backend-config=key=tenants/<tfvars_name>.tfstate` (the workflow does
this whenever `tfvars_name` is set). Workspaces within each state separate
dev/stage/prod.

**Domain convention** (computed in `terraform-devplus-tenant`, output
`public_host`): `<tenant>.<env>.<base_domain>` for non-prod (e.g.
`app.dev.devplus.io`, `helder.dev.devplus.io`); prod drops the env segment
(`app.devplus.io`, `test.devplus.io`). The same host is the OIDC issuer
(`publicUrl`) and the Cloudflare ingress hostname.

## Deploy a tenant

Normally through `Actions → Deploy Tenants` (`environment`, `chart_version`,
`tfvars_name`). By hand, with the same `TF_VAR_*` secrets
`_deploy-environment.yml` loads from the environment's Key Vault (Cloudflare,
Infracost, Atlas service account, WorkOS, `ghcr_pull_token`), git credentials
for the private module clones, and a kube context on that environment's AKS
cluster:

```bash
cd terraform/tenants/devplus

tofu init                                   # app tenant
# tofu init -reconfigure -backend-config=key=tenants/helder.tfstate   # any other tfvars
tofu workspace select dev || tofu workspace new dev    # or: stage, prod
tofu apply -var-file=environments/dev.tfvars -var chart_version=<published version>
```

## Add a new tenant

1. Create `environments/<name>.tfvars` (copy the closest one) and set
   `tenant_name` (the namespace, database and host label), `tenant_sso_domain`,
   `admin_seed`, and the environment's subscription, resource group, AOAI
   endpoint and Atlas URI (plus `mongodbatlas_idp_id` outside dev: the default
   is the dev cluster's IdP).

2. Deploy it with `tfvars_name=<name>` (Deploy Tenants), or by hand:
   ```bash
   cd tenants/devplus
   tofu init -reconfigure -backend-config=key=tenants/<name>.tfstate
   tofu workspace select dev || tofu workspace new dev
   tofu apply -var-file=environments/<name>.tfvars -var chart_version=<published version>
   ```

A separate tenant directory (copy `tenants/devplus`, change the `backend.tf`
key) also works and is selected with the `tenants` input, but nothing uses one
today.

## Architecture

The root creates the tenant's workload identity (`id-devplus-<tenant>-<env>`),
its federated credentials for the `<ns>-app` and `executor-job-dispatcher`
service accounts (issuers from core-infra remote state) and the
`Cognitive Services OpenAI User` role on `ais-devplus-<env>`, then calls the
`terraform-devplus-tenant` module, which creates the WorkOS organization
(restapi) and the Atlas OIDC database users, and calls:

- **terraform-helm-devplus-services** — K8s namespace, secrets, PVCs, Helm release
- **terraform-cloudflare-ingress** — Cloudflare tunnel, DNS CNAME, cloudflared deployment

### Provisioned tenant GitHub App install

Tenants provisioned from inside DevPlus (the `deploy-devplus-tenant` catalog
item) do not go through this Terraform path, so nothing creates a per-tenant
`github-install-state` Secret for them. They rely on the chart default instead:
core-infra publishes the install-state HMAC once as
`github-install-state-shared` in its `external-secrets` namespace and Reflector
mirrors it into every namespace, the same way `github-app-secret` reaches
tenants. The chart ships with `githubInstallState.enabled=true` pointing at that
mirror, so a provisioned tenant can mint GitHub App install state with no extra
wiring.

Terraform-managed tenants keep their own copy: `terraform-helm-devplus-services`
creates `github-install-state` from core-infra remote state and overrides
`githubInstallState.secretName` to it. The two names must never be created by
the same owner (`github-install-state-shared` belongs to Reflector,
`github-install-state` to Terraform or Tilt). The secretKeyRef is required on
purpose: a backend pod in a namespace the mirror has not reached yet waits in
`CreateContainerConfigError` and starts on its own once the Secret appears.

That self-healing has a precondition: the mirror only lands on a cluster where
core-infra creates the Reflector source Secret, which today is the eus2 cluster
that also hosts the github-callback hub. On any other cluster nothing will ever
mirror `github-install-state-shared`, and the chart default leaves the backend
pod in `CreateContainerConfigError` for good. When deploying a tenant there,
set `githubInstallState.enabled=false` or override
`githubInstallState.secretName` to a Secret you create yourself.

The workspace name (`dev`/`stage`/`prod`) determines:

- The DNS subdomain (and the WorkOS redirect host, `auth.<env>.devplus.io`, bare
  `auth.devplus.io` in prod)
- Which core-infra remote-state workspace is read (Atlas project id, AKS OIDC
  issuers, the GitHub install-state secret)
- The names of the identity and AOAI account (`id-devplus-<tenant>-<env>`,
  `ais-devplus-<env>`)

The cluster is whatever kube context is current (`kubeconfig_context` overrides
it; CI sets it with `azure/aks-set-context` from the GitHub Environment), and the
Atlas connection string comes from the tfvars (`mongodb_atlas_uri`).

## Chart version

`chart_version` pins the published chart `oci://ghcr.io/devplusinc/charts/devplus`
and, through its `appVersion`, every image: `ghcr.io/devplusinc/<image>:<chart_version>`
for `backend`, `executor`, `governance-service`, `opentofu-server`,
`oidc-service`, `frontend`, `agent-service` and `devplus-mcp`. The variable
defaults to `0.1.0`; always pass the version you mean.

The `governance-service` image runs as the `governance` Deployment and K8s
Service at `http://governance:8084`; the `executor` image runs as the
`executor-job-dispatcher` Deployment (no Service).
