---
id: 56ad6350-532a-4b73-a9cf-2e63a6ee4bc8
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/TILT.md

Source: DevPlusInc/devplus-platform/TILT.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, TILT.md).

# Tilt Development Guide

This guide explains how to use Tilt to deploy and develop DevPlus across different environments.

## Prerequisites

1. **Tilt** — https://docs.tilt.dev/install.html
2. **kubectl** — configured for your target cluster
3. **Helm v3** — Tilt renders manifests from the chart in `k8s/helm/devplus`
4. **Docker** — for building images
5. **Azure CLI (`az`)** logged in to the `devplus-dev` subscription, with **Cloudflare WARP on**: every mode reads secrets from Key Vault `kv-devplus-dev` (public network access disabled)

### Mode-specific prerequisites

| Mode | Additional requirements |
|------|------------------------|
| `local` | k3d cluster with local registry |
| `atlas` | k3d cluster with local registry |
| `aks` | AKS cluster context (`az aks get-credentials -g devplus-dev-eus2 -n aks-devplus-dev-eus2`), GitHub CLI (`gh`), Docker logged into GHCR, MongoDB Atlas CLI (`atlas auth login`) |

### Auth setup

Before running Tilt, make sure you're logged in:

```bash
# Required in every mode (fetches secrets from Azure Key Vault; WARP must be on)
az login

# Required for aks mode (pushes images to GHCR, creates pull secrets)
gh auth login -h github.com -p https -w -s write:packages
gh auth token | docker login ghcr.io -u $(gh api user -q .login) --password-stdin

# Required for aks mode (creates the tenant's Atlas OIDC database users)
atlas auth login
```

## CLI Arguments

All arguments are passed after `--`. Run from the umbrella parent directory
(the examples below omit `-f devplus-platform/Tiltfile` for brevity):

```bash
tilt up -f devplus-platform/Tiltfile -- --tenant=<name> --mode=<mode> [--tag=<version>] [--llm=<deployment>] [--reasoning=<effort>]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `--tenant` | Always | Tenant name. Used as the K8s namespace and MongoDB database name. |
| `--mode` | Always | Deployment mode: `local`, `atlas`, or `aks`. |
| `--tag` | In `aks` mode | Image version tag for GHCR push (e.g. `0.2.0`). Prevents accidental overwrites of `:latest`. |
| `--llm` | No | Foundry chat deployment on `ais-devplus-dev` for agent-service (default `DeepSeek-V4.1-Flash`, e.g. `gpt-5.4`). Switch live with `tilt args`. |
| `--reasoning` | No | Thinking effort for agent-service agent and design calls: `high` (default), `medium`, `low`, or `off`. |

Set `OTEL_ENABLED=true` (and optionally `OTEL_ENDPOINT`, `OTEL_SAMPLER_ARG`) in the environment of `tilt up` to export traces to an OTLP collector; it is off by default.

## Modes

### `local` — Fully local development

```bash
tilt up -- --tenant=acme --mode=local
```

- Runs on a **k3d** cluster with a local registry at `localhost:5111`
- Deploys an **in-cluster MongoDB** StatefulSet with default credentials (`admin`/`admin123`)
- Compute and data are local, but secrets still come from Key Vault (WorkOS, Infracost, GitHub App key, GHCR puller PAT, optional HubSpot token), AI calls go to Azure OpenAI `ais-devplus-dev` with a host-minted `aoai-token`, and login round-trips through the WorkOS hub at `auth.dev.devplus.io`
- Installs Stakater Reloader and emberstack Reflector into the k3d cluster (on AKS, core-infra provides them)
- Images are pushed to the k3d registry, pulled with `IfNotPresent`
- Open the app at `http://<tenant>.localhost:3000` (Tilt sets the chart's `tenantHost` to that host; Chrome resolves `*.localhost` without an `/etc/hosts` entry)

**k3d cluster setup:**
```bash
k3d cluster create dev --registry-create k3d-registry:0.0.0.0:5111
```

### `atlas` — Local compute, shared database

```bash
tilt up -- --tenant=acme --mode=atlas
```

- Runs on a **k3d** cluster with a local registry
- Connects to **MongoDB Atlas** instead of local MongoDB
- Atlas credentials are fetched from Azure Key Vault (`kv-devplus-dev`) via `az` CLI
- No in-cluster MongoDB is deployed

### `aks` — Fully remote

```bash
tilt up -- --tenant=acme --mode=aks --tag=0.2.0
```

- Deploys to an **AKS** cluster (`aks-devplus-dev-eus2`)
- Connects to **MongoDB Atlas** via OIDC workload identity (no passwords)
- Images are built locally, tagged with the provided `--tag`, and pushed to **GHCR** (`ghcr.io/devplusinc`)
- Creates Azure federated identity credentials and Atlas OIDC database users for the tenant
- Uses `imagePullPolicy: Always` and a `ghcr-pull-secret` (created from your `gh auth token`)
- Service accounts are annotated for Azure workload identity (shared identity `id-devplus-shared-dev`; its client-id and the cluster OIDC issuer are resolved with `az` at load time)
- Grants that identity `Cognitive Services OpenAI User` on `ais-devplus-dev` (`azure-openai-role`); backend and agent-service call Azure OpenAI as the managed identity, so no `aoai-token` is created
- Exposes the tenant at `https://<tenant>.dev.devplus.io` through a per-tenant Cloudflare tunnel (`cloudflare-ingress`, `scripts/cf-ingress.sh`) and seeds the WorkOS org mapping (`workos-org-mapping`, `scripts/seed-workos-org.sh`). Turn WARP off in the browser to open it

**GHCR auth setup** (required before first use):
```bash
gh auth refresh -s write:packages
gh auth token | docker login ghcr.io -u $(gh api user -q .login) --password-stdin
```

## Port Forwards

Available in all modes (AKS uses the same ports via kubectl port-forward):

| Service | Port |
|---------|------|
| Frontend | `http://<tenant>.localhost:3000` |
| Backend API | http://localhost:8080 |
| OIDC Service | http://localhost:8081 |
| OpenTofu Server | http://localhost:8082 |
| Agent Service | http://localhost:8083 |
| Governance | http://localhost:8084 |
| MongoDB (local mode only) | localhost:27017 |

## What Tilt Does

### 1. Builds Docker images

All seven service images are built with `linux/arm64` platform. In `aks` mode, each image is additionally tagged with the `--tag` value for GHCR. (`devplus-mcp` is not part of the Tilt loop.)

### 2. Creates secrets and storage

Tilt creates K8s secrets and PVCs before deploying services:

| Resource | Scope | Description |
|----------|-------|-------------|
| `mongodb-secret` | local only | MongoDB credentials (admin/admin123) |
| `mongodb-atlas-uri` | atlas/aks | Atlas connection string (Key Vault or OIDC) |
| `mongodb-pvc` | local only | MongoDB persistent storage |
| `ghcr-pull-secret` | all | aks: per-tenant, from your `gh auth token`. local/atlas: source in the `external-secrets` namespace from Key Vault `ghcr-puller-pat`, Reflector-mirrored |
| `github-app-secret` | local/atlas | GitHub App config (app id, slug, private key from Key Vault `github-app-key`), source in `external-secrets`, Reflector-mirrored. aks: mirrored by core-infra's Reflector |
| `github-install-state` | all | GitHub App install-state HMAC (aks: copied from the `github-callback` hub; local/atlas: fixed placeholder). Tilt overrides the chart's default `github-install-state-shared` Reflector mirror with this per-tenant copy |
| `jwt-secret` | all | JWT signing secret for OIDC service |
| `oidc-keys` | all | OIDC signing keys (auto-generated if empty) |
| `secret-encryption-key` | all | Envelope-encryption key for the backend's connector secret store and for OpenTofu state at rest in opentofu-server (fixed dev value) |
| `infracost-secret` | all | Infracost API key from Azure Key Vault |
| `hubspot-secret` | all | Support-desk HubSpot token from Key Vault `hubspot-access-token`; optional, deleted when absent (support desk off) |
| `workos-api-key`, `workos-client-id`, `workos-cookie-password` | all | WorkOS AuthKit credentials from Key Vault |
| `aoai-token` | local/atlas | Host-minted Entra ID token for Azure OpenAI (~1h). Use the Tilt UI refresh button on 401s |
| `internal-api-token` | all | Created by the chart (`internalApi.createSecret=true`, dev value) for `X-Internal-Token` calls |

### 3. Deploys via Helm

All K8s resources are rendered from the Helm chart at `k8s/helm/devplus` with mode-specific values. A `tilt-values.yaml` is auto-generated for structured values (e.g. service account annotations for Azure workload identity in `aks` mode).

WorkOS: Tilt pins one `workos.organizationId` (hardcoded in the Tiltfile, and passed again to `workos-org-mapping`) and the centralized redirect URI `https://auth.dev.devplus.io/callback`. The cloud auth-service resolves that org to a tenant URL through its `auth.workos` mapping, which Tilt seeds only in `aks`; in `local`/`atlas` the login round-trip completes only if that mapping points the org at `http://<tenant>.localhost:3000`.

### 4. Pushes to GHCR (aks only)

After each service deploys successfully, Tilt pushes the versioned image to GHCR:
```
ghcr.io/devplusinc/<service>:<tag>
```
The manual `publish-helm-chart` button (`scripts/publish-chart.sh <tag>`) packages the chart at that version and pushes it to `oci://ghcr.io/devplusinc/charts`. Releases normally go through the `Release` workflow instead (see [docs/RELEASE.md](docs/RELEASE.md)).

### 5. Azure/Atlas setup (aks only)

- Creates (or corrects the issuer of) Azure federated identity credentials for the tenant's service accounts (`<tenant>-app`, `executor-job-dispatcher`)
- Creates Atlas OIDC database users for the tenant
- Ensures the `Cognitive Services OpenAI User` role for the shared identity

## Service Dependency Order

```
namespace
  ├── mongodb-pvc + mongodb-secret → mongodb (local only)
  ├── mongodb-atlas-uri (atlas/aks)
  ├── azure-federated-credentials → atlas-oidc-dbusers (aks only)
  ├── jwt-secret + oidc-keys → oidc-service
  ├── github-install-state + secret-encryption-key + hubspot-secret
  │     + github-app-secret/aoai-token (local/atlas) or azure-openai-role (aks) → backend
  ├── backend + secret-encryption-key → opentofu-server
  ├── backend → governance
  ├── opentofu-server + oidc-service + infracost-secret → executor-job-dispatcher
  ├── aoai-token (local/atlas) or azure-openai-role (aks) → agent-service
  ├── backend + agent-service + workos-* secrets → frontend
  └── frontend + backend → cloudflare-ingress → workos-org-mapping (aks only)
```

## Troubleshooting

### Services not starting
- Check the Tilt UI at http://localhost:10350 for error messages
- Verify your cluster is reachable: `kubectl get nodes`

### GHCR push failures (aks mode)
- Ensure Docker is authenticated: `gh auth token | docker login ghcr.io -u $(gh api user -q .login) --password-stdin`
- Ensure your `gh` token has `write:packages` scope: `gh auth status`
- If missing, run: `gh auth refresh -s write:packages`

### Atlas connection failures (atlas mode)
- Ensure Azure CLI is logged in: `az login`
- Verify Key Vault access: `az keyvault secret show --vault-name kv-devplus-dev --name mongodb-atlas-username`

### Image pull errors
- For k3d: images are pushed to the local registry automatically
- For AKS: check that `ghcr-pull-secret` exists in the namespace

### Port conflicts
- Edit the `port_forwards` in the Tiltfile, or stop conflicting services

## Stopping and Cleanup

```bash
# aks only: delete the Cloudflare tunnel + DNS record first (Tilt cannot on `down`)
# Trigger `cloudflare-ingress-down` in the Tilt UI, or:
devplus-platform/scripts/cf-ingress.sh down <tenant> dev <tenant>

# Stop all services (keeps namespace and PVCs)
tilt down

# Full cleanup
tilt down
kubectl delete namespace <tenant>
```
