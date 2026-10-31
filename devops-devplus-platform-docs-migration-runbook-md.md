---
id: 25f7d2fd-97ac-4880-8871-e6b542a35168
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/MIGRATION_RUNBOOK.md

Source: DevPlusInc/devplus-platform/docs/MIGRATION_RUNBOOK.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/MIGRATION_RUNBOOK.md).

# DevPlus Monorepo Split — Migration Runbook & Verification Checklist

This is the master checklist for finishing the InfraSync → DevPlus multi-repo
split. The code-level migration (9 repos, fresh history, rename sweep,
per-service CI, per-service image tags, deploy rework, Tilt parity, runtime
upgrades, coverage tooling) is complete in the local umbrella checkout.

The remaining steps below require **GitHub org-owner access to `DevPlusInc`** and
**Azure access to the `infrasync-*` core-infra subscription**, so they must be
run by a human operator. Each step lists the exact commands.

> **Status (2026-10): historical.** The split is done: every repo lives in
> `DevPlusInc`, CI builds and attests the images, and tenants deploy from a
> published chart version (see [RELEASE.md](RELEASE.md) and [CICD.md](CICD.md)).
> Keep this file as the record of how the split was finished. Where a step has
> since changed, a **Now:** note says what replaced it.

---

## 0. Repo inventory (umbrella checkout)

```
devplus-shared            # private Go module (github.com/DevPlusInc/devplus-shared)
devplus-backend           # Go API
devplus-executor          # Go OpenTofu job runner
devplus-governance-service  # Go governance service
devplus-opentofu-server   # Go module registry + state backend
devplus-oidc-service      # Go OIDC provider (uses devplus-shared for OTel; builds with Dockerfile.tilt under Tilt)
devplus-agent-service     # Python FastAPI/LangGraph agents
devplus-frontend          # React SPA (Vite) served by nginx (was Next.js until 2026-07-13)
devplus-platform          # k8s/, terraform/, Tiltfile, scripts/, docs/, deploy CI
terraform-devplus-tenant            # external TF (was terraform-infrasync-tenant)
terraform-helm-devplus-services     # external TF (was terraform-helm-infrasync-services)
```

`go.work` lives at the umbrella root only (gitignored in each repo) so local
builds/IDE resolve `devplus-shared` from the sibling checkout.

---

## 1. Create the GitHub repos and push (fresh history)

Each repo already has a single fresh initial commit locally (except the 7
service repos + platform, which are committed by `scripts/commit-all.sh` /
`git init` — see below). For each repo:

```bash
gh repo create DevPlusInc/<repo> --private --source=<repo-dir> --remote=origin
git -C <repo-dir> push -u origin main
```

`devplus-shared` must also be tagged so the services can resolve it:

```bash
git -C devplus-shared tag v0.1.0 && git -C devplus-shared push origin v0.1.0
```

> The service `go.mod` files already require `devplus-shared v0.1.0`. If shared
> changes, bump the tag and `go get github.com/DevPlusInc/devplus-shared@vX.Y.Z`
> in each consumer.
>
> **Now:** shared is on the `v1.x` line (latest tag `v1.16.0`); each Go service
> pins its own `v1.x` in `go.mod`.

---

## 2. GitHub App setup  (todo: github-app-setup)

The per-repo `build.yml` mints a short-lived **GitHub App installation token**
(via `actions/create-github-app-token`) to (a) read the private
`devplus-shared` during Go builds and (b) fire the cross-repo
`repository_dispatch` to `devplus-platform`. `GITHUB_TOKEN` cannot do either.

1. Create (or reuse) a GitHub App under the **DevPlusInc** org with permissions:
   - **Repository contents: Read** (to clone `devplus-shared`)
   - **Metadata: Read**
   - **Contents: Read & write** on `devplus-platform` (for dispatch) — or use a
     dedicated dispatch fine-grained PAT instead.
2. Install the App on the DevPlusInc org, scoped to **all `devplus-*` repos**.
3. Note the **App ID** and generate a **private key** (.pem).
4. Add these as **org-level Actions secrets** (or per-repo) so every build can read them:
   - `DEVPLUS_APP_ID`
   - `DEVPLUS_APP_PRIVATE_KEY`  (contents of the .pem)

```bash
gh secret set DEVPLUS_APP_ID --org DevPlusInc --visibility all --body "<app-id>"
gh secret set DEVPLUS_APP_PRIVATE_KEY --org DevPlusInc --visibility all < app-private-key.pem
```

**Now:** `build.yml` no longer reads Actions secrets for the App. It logs in to
Azure with OIDC and loads `DEVPLUS_APP_ID` / `DEVPLUS_APP_PRIVATE_KEY` from Key
Vault `kv-devplus-dev` secrets `github-app-cicd-id` / `github-app-cicd-key`.

---

## 3. GHCR org migration  (todo: ghcr-org-migration)

Images now publish to `ghcr.io/devplusinc/<service>`. The deploy pulls them into
tenant namespaces via a Reflector-mirrored `ghcr-pull-secret`.

1. The first successful `build.yml` run on each service creates the
   `ghcr.io/devplusinc/<service>` package automatically (pushed with
   `GITHUB_TOKEN`). After the first push, set each package's visibility and grant
   `devplus-platform` read access in **Package settings → Manage Actions access**.
2. Mint a **classic PAT** (or fine-grained, `read:packages`) for the puller bot
   and store it in Azure Key Vault so Tilt/core-infra can pull:

```bash
az keyvault secret set --vault-name kv-devplus-dev --name ghcr-puller-pat       --value "<PAT>"
az keyvault secret set --vault-name kv-devplus-dev --name ghcr-puller-username  --value "<bot-login>"
```

3. In `devplus-core-infra` (modules/external-secrets), point the
   `ghcr-pull-secret` source at the new PAT/username; Reflector fans it into
   every tenant namespace. (The Helm chart + Tiltfile already reference
   `ghcr-pull-secret`.)

> **Legacy Azure / cloud names (keep as-is):**
> AOAI `ais-infrasync-dev`, RGs `infrasync-dev[-eus2]`, AKS `aks-infrasync-dev-eus2`,
> Atlas host `infrasync-dev.27xwgp.mongodb.net`, repo `devplus-core-infra`, and
> GitHub org modules under `github.com/infrasync/*` — these are real resources, not
> application branding.
>
> **Now:** those resources were since replaced by DevPlus-named ones: AOAI
> `ais-devplus-dev`, RGs `devplus-dev` / `devplus-dev-eus2`, AKS
> `aks-devplus-dev-eus2`, Atlas host `devplus-dev.wnzhnl.mongodb.net`, and the
> Terraform modules are sourced from `github.com/DevPlusInc/*`.
>
> **Application code (DevPlus):** cost override file `devplus.hcl` (the legacy
> `infrasync.hcl` is no longer read), metadata keys `devplus_cost_override*`, Cedar seed
> application `DevPlus`, AI prompt references DevPlus. Helm chart lives at
> `devplus-platform/k8s/helm/devplus`.

---

## 4. Deploy-time secrets  (todo: secrets)

Move deploy-time secrets to the **devplus-platform** repo only (they are read by
`_deploy-environment.yml` via `tofu` `TF_VAR_*` / `-var`). Set as repo secrets:

```bash
R=DevPlusInc/devplus-platform
gh secret set CLOUDFLARE_API_TOKEN   -R $R --body "..."
gh secret set CLOUDFLARE_ACCOUNT_ID  -R $R --body "..."
gh secret set CLOUDFLARE_ZONE_ID     -R $R --body "..."
gh secret set INFRACOST_API_KEY      -R $R --body "..."
gh secret set AZURE_OPENAI_API_KEY   -R $R --body "..."
gh secret set ADMIN_EMAIL            -R $R --body "..."
gh secret set ADMIN_PASSWORD         -R $R --body "..."
gh secret set ATLAS_PUBLIC_KEY       -R $R --body "..."
gh secret set ATLAS_PRIVATE_KEY      -R $R --body "..."
gh secret set MODULES_REPO_PAT       -R $R --body "..."
# Azure OIDC login for `tofu`:
gh secret set AZURE_CLIENT_ID -R $R --body "..."; gh secret set AZURE_TENANT_ID -R $R --body "..."; gh secret set AZURE_SUBSCRIPTION_ID -R $R --body "..."
```

Also set the App secrets from step 2 on `devplus-platform` if not org-wide.

**Now:** the deploy reads none of these as repo secrets. `_deploy-environment.yml`
logs in to Azure with the GitHub Environment's `vars.AZURE_*` and loads its
inputs from that environment's Key Vault (`vars.KEYVAULT`): `cloudflare-terraform-token`,
`infracost-api-key`, `hub-mongodbatlas-client-id` / `-client-secret` (Atlas
service account), `workos-api-key`, `workos-client-id`, `workos-cookie-password`
and `ghcr-puller-pat` (private module clones and the private chart pull). The
Azure OpenAI API key is gone (AKS uses workload identity) and the admin seed is
set in each `terraform/tenants/devplus/environments/*.tfvars` (an email and a
name, no password).

---

## 5. External Terraform repos

`terraform-devplus-tenant/main.tf` now sources
`github.com/DevPlusInc/terraform-helm-devplus-services?ref=main` and consumes a
per-service `image_tags` map (falling back to `latest`). Push both:

**Now:** `terraform-devplus-tenant` pins both modules by commit SHA and takes a
single `chart_version` (the published chart, whose `appVersion` pins every image)
instead of `image_tags`.

```bash
gh repo create DevPlusInc/terraform-helm-devplus-services --private --source=terraform-helm-devplus-services --remote=origin
gh repo create DevPlusInc/terraform-devplus-tenant        --private --source=terraform-devplus-tenant        --remote=origin
git -C terraform-helm-devplus-services push -u origin main
git -C terraform-devplus-tenant push -u origin main
```

> Known gap: `terraform-devplus-tenant/main.tf` still sources
> `github.com/infrasync/terraform-cloudflare-ingress` (3rd-party module, out of
> the declared 2-repo scope). Migrate/clone it to DevPlusInc if the deploy
> identity loses access to the `infrasync` org.
>
> **Now:** closed. It sources `github.com/DevPlusInc/terraform-cloudflare-ingress`.

---

## 6. Per-service image-tags deploy flow (already wired)

- Each `build.yml` pushes `ghcr.io/devplusinc/<svc>:<sha>` then dispatches
  `repository_dispatch` (`event-type: service-image`) to `devplus-platform` with
  `{service, image_tag, head_branch}`.
- `devplus-platform/.github/workflows/deploy-tenants.yml` (`resolve` job) updates
  the committed `image-tags.json` for that one service, then `_deploy-environment.yml`
  runs `tofu apply -var 'image_tags=<json>'`. Other services keep their last tag.

**Now:** replaced on 2026-07-09 by the chart-version model. `image-tags.json` is
gone; a merge to a service's `main` pushes `:<sha>` and `:latest`, and its
`service-image` dispatch now only triggers the `Platform Test Matrix`. Shipping
is deliberate: the `Release` workflow promotes all eight images by digest and
publishes `oci://ghcr.io/devplusinc/charts/devplus:<version>`, and
`Deploy Tenants` (manual) deploys a `chart_version` to `dev` | `stage` | `prod`.

---

## 7. End-to-end verification checklist

Per-repo (local, from the umbrella with `go.work`):

- [ ] `devplus-shared`: `go test ./...` and CI `ci.yml` coverage gate
- [ ] each Go service: `go test ./...` and CI `build.yml` coverage gate (`GOWORK=off` uses tagged `devplus-shared`)
- [ ] `devplus-frontend`: `bun run test:coverage` and `bun run build`
- [ ] `devplus-agent-service`: `uv run pytest --cov=src/agent_service --cov-fail-under=80`
- [ ] `devplus-platform`: `helm lint` / `helm template` on `k8s/helm/devplus`

Org / deploy (the pre-2026-07-09 flow; see the **Now:** note in step 6):

- [ ] GitHub App installed on DevPlusInc; `DEVPLUS_APP_ID` + `DEVPLUS_APP_PRIVATE_KEY` set
- [ ] `ghcr.io/devplusinc/*` packages exist; `devplus-platform` has read access; puller PAT in KV + core-infra
- [ ] platform repo secrets set (step 4)
- [ ] push to a service `main` → `build.yml` green (incl. hard coverage gate) → image at `ghcr.io/devplusinc/<svc>:<sha>`
- [ ] dispatch lands in `devplus-platform`; `image-tags.json` updated for that service
- [ ] `tofu plan` on dev runs; feature-branch routing (hf/ss/as) maps correctly
- [ ] full push → image → dispatch → `tofu apply` deploy to dev succeeds
- [ ] `tilt up -f devplus-platform/Tiltfile -- --tenant=<t> --mode=local` brings everything up from the umbrella

---

## 8. Remaining engineering: Go service test coverage

The hard-80% gate is wired into every `build.yml` / `ci.yml` (inline in the
Test + coverage step). `devplus-shared`, `devplus-frontend`, and
`devplus-agent-service` already PASS the gate. Extend each repo's `LOGIC_INCLUDE`
regex in the workflow only for genuine integration-only files (Mongo repos,
K8s/external API clients) with a documented justification — do not exclude
handler/service logic to inflate percentages.
