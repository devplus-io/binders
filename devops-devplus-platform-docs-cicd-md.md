---
id: d0e63483-24d9-4e47-8b83-817b1535d6dd
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/CICD.md

Source: DevPlusInc/devplus-platform/docs/CICD.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/CICD.md).

# DevPlus CI/CD

How code becomes a running tenant. Two clean halves: **CI** builds+publishes
immutable artifacts; **CD** deploys one versioned chart with terraform.

> **Shipping something right now?** Follow **[RELEASE.md](RELEASE.md)** — the
> step-by-step runbook (cut → dev → stage → promote → prod, with the verification
> gates and rollback). This doc explains *how the pipeline works* and *why*.

## The model (uniform versioning)

One release version `X` (e.g. `0.1.0-dev7`) pins **every** service. There is no
per-service image map and no state committed back to git — the **published chart
version is the single source of truth**.

```
service repo (build.yml)                 devplus-platform
  push main ─► build arm64 image           Release  (release.yml, manual dispatch)
             ─► push :<sha> + :latest         1. compute next 0.1.0-dev<N>
                    (GHCR, per service)        2. promote each :latest ─(by digest)─► :X   (all 8 images)
                                               3. helm package --version X --app-version X
                                                  helm push oci://ghcr.io/devplusinc/charts
                                               4. deploy  (calls _deploy-environment)
                                                    │
                                                    ▼
                                          _deploy-environment.yml
                                            tofu apply -var chart_version=X
                                                    │
                                          terraform/tenants/devplus
                                            module.tenant ─► helm_release
                                              chart   = oci://ghcr.io/devplusinc/charts/devplus
                                              version = X
                                              ⇒ every image = ghcr.io/devplusinc/<svc>:X
                                                (chart appVersion default)
```

### Why "promote", not rebuild
Each service already publishes `:<sha>` + `:latest` on merge to `main`. A release
**re-tags** each service's current `:latest` to `:X` **by digest** (`crane copy` on
the resolved digest) — no rebuild, and because crane **preserves the manifest
digest**, the GitHub-native SLSA/SBOM attestation referrers (anchored to that
digest) carry over so Kyverno still verifies them in-cluster.

> Must be `crane` (or another digest-preserving copy), **not** `docker buildx
> imagetools create`: the latter wraps a single-arch manifest in a *new* image
> index with a *new* digest, and the attestation lives on the wrapped digest — so
> the promoted tag resolves to an unattested index and the `Enforce` gate rejects
> it (dev ran `Audit` so it slipped through; stage/prod, then on `Enforce`, caught
> it. Every env runs `Audit` today, see below).

## CI gates (before anything is published)

- **Service repos** (`build.yml`): lint and vulnerability scan (Go: `go vet`,
  golangci-lint `v2.12.2`, govulncheck `v1.3.0`; agent-service: ruff + pip-audit;
  frontend: eslint + bun audit), tests with a hard **80% coverage gate** on each
  repo's logic-layer files, then the `linux/arm64` image push + SBOM/SLSA
  attestations, only on a `main` push or a manual dispatch. The backend adds the
  OpenAPI spec gates (stale spec, oasdiff breaking changes, generated-client
  smoke) and a schemathesis conformance job; the frontend adds the stale
  API-types gate and the hardcoded-color ratchet.
- **devplus-platform**: `ci.yml` (helm lint + template, `tofu fmt`/`validate` of
  `terraform/tenants/devplus`), `platform-test-matrix.yml` (the hermetic
  black-box suite from `tests/`: every PR, pushes touching `tests/`, nightly, and
  each `service-image` dispatch) and `dependency-review.yml` (fails a PR on a
  moderate or worse dependency advisory).

## Cutting a release

`Actions → Release → Run workflow`:

| input | default | notes |
|---|---|---|
| `version` | *(empty)* | any SemVer — `0.1.0` stable, `0.2.0-rc.1`, `0.1.0-dev7`. Empty ⇒ auto-compute next `0.1.0-dev<N>`. (`+buildmetadata` is rejected: legal SemVer, **illegal in a Docker tag**, and this string is also the image tag.) |
| `source_tag` | `latest` | what to promote FROM. `latest` = current main. **Any other value must be a published chart version** (see promotion below) |
| `deploy` | `true` | deploy the release after publishing |
| `environment` | `dev` | where to deploy it (`dev` \| `stage` \| `prod`) |
| `tfvars_name` | `""` (dev) | per-tenant/-developer config: `dev` \| `helder` \| `aaron` \| `saamer` \| `test-dev` \| `test-stage` \| `test-prod` (the `test` canary) |
| `quality_gate` | `auto` | run the black-box suite against the `test` canary after the deploy: `auto` (when `tfvars_name` = `test-<environment>`) \| `always` \| `never`. See [RELEASE.md](RELEASE.md#the-quality-gate) |
| `tenants` | `["devplus"]` | terraform tenant dir(s) |

The version resolver **fails loudly** on any non-404 listing error (never
silently falls back to `dev1`) and **refuses to overwrite** an already-published
version — bump instead.

### A version is an artifact identity, not an environment

`0.1.0-dev7` means "7th unverified pre-release", **not** "the dev environment's
build". Any version deploys to any environment. Two flows:

- **Fast loop** — `version=""`, `source_tag=latest`, `environment=dev`. Cuts a
  pre-release from current main.
- **Stable release** — `version=0.1.0`, `source_tag=0.1.0-dev7` (a pre-release you
  already verified), `environment=prod`.

### Promotion promotes BOTH halves

A release is images **and** a chart. `source_tag` promotes the images by digest —
and when it names a published version, the **chart is pulled from that release**
too and re-versioned, rather than packaged from the current `main`.

That coupling is not pedantry. Templates and images are versioned together: the
SPA migration deleted `WORKOS_*`/`BACKEND_URL` from `frontend.yaml` and moved the
probe to `/healthz`. Packaging main's chart around an older verified image would
produce a pod that never goes Ready — and since both halves would carry the new
version, the skew would be **invisible**. So promoting from a bare image tag
(e.g. `sha-abc123`) is refused: there is no chart to promote with it.

> `CHART_BASE` in `release.yml` is still `0.1.0` while stable releases have
> reached `0.1.16`. SemVer ranks every `0.1.0-dev<N>` *below* `0.1.0`, so
> auto-computed pre-releases (`0.1.0-dev15` was the last) sort behind every
> stable release. Harmless (deploys pin exact versions), but confusing: bump it,
> or pass an explicit `version` as the 0.1.16 cycle did (`0.1.16-dev2`).

## Deploying an existing version — any environment

`Actions → Deploy Tenants → Run workflow` with **`environment`** (`dev` | `stage`
| `prod`) + `chart_version` (+ optional `tfvars_name`, `tenants`, `plan_only`,
`quality_gate`). Same `_deploy-environment` path, no re-publish. The `environment` input selects the
tofu **workspace**, the **GitHub Environment** (per-env `AZURE_CLIENT_ID`,
`KEYVAULT`, AKS vars — each env has its **own** deploy app registration), and
(when `tfvars_name` is empty) the `environments/<env>.tfvars` file. A non-empty
`tfvars_name` also moves the tofu state to its own key,
`tenants/<tfvars_name>.tfstate` (`-backend-config=key=...`). Any published
chart version may target any environment — there is deliberately no `-dev`-chart
guard (pre-launch decision; revisit with a real release cadence).

All three environments are live (first deployed 2026-07-10/11): the `app` tenant
serves at `app.dev.devplus.io`, `app.stage.devplus.io`, and `app.devplus.io`
(prod host is bare, no env segment). A second tenant, `test`, is the canary in
every environment (`test.dev.devplus.io`, `test.stage.devplus.io`,
`test.devplus.io`; `test-<env>.tfvars`); the quality gate runs against it.
Per-env values that can't come from core
remote-state (subscription, RG, Atlas URI host, Atlas IdP id) live in
`terraform/tenants/devplus/environments/{dev,stage,prod}.tfvars`; the Atlas
project id + AKS OIDC issuers are read from core-infra remote state
automatically.

Set **`plan_only: true`** for a dry run — `_deploy-environment` inits + plans and
**skips apply**, so you can review the tofu diff first. Re-run with `plan_only`
off (default) to apply. Handy after touching tenant terraform (e.g. remote-state
or Atlas wiring) before letting it apply.

> Deploys are **deliberate** (manual). A service merge only refreshes that
> service's `:latest`; cut a Release to ship it. Service repos still fire a
> `repository_dispatch(service-image)`: it deploys nothing, it starts the
> hermetic **Platform Test Matrix** (`platform-test-matrix.yml`) with the
> dispatching service checked out at the pushed branch (devplus-mcp does not
> dispatch).

## Versioning & registry

- Any SemVer version, stable or pre-release. Chart `version` == `appVersion` == the
  release version; images share it (`k8s/helm/devplus` — `image.tag` defaults to
  `.Chart.AppVersion`, registry defaults to `ghcr.io/devplusinc`). The 8 images:
  `backend executor governance-service opentofu-server oidc-service frontend
  agent-service devplus-mcp` (`SERVICES` in `release.yml`).
- `CHART_BASE` (`release.yml`) only seeds the **auto-computed** `<base>-dev<N>`
  pre-release; an explicit `version` ignores it.

## Image attestation (why the promote uses crane)

Service `build.yml`s attest every image (GitHub-native SLSA provenance + SBOM,
anchored to the **build digest**, pushed as OCI referrers). The Release promote
**must preserve that digest** — `crane copy` does; `docker buildx imagetools
create` wraps a new (unattested) index and the cluster gate rejects it. The
Kyverno gate (`verify-ghcr-images`, core-infra `modules/image-verification`)
trusts any `DevPlusInc/*` repo's main-branch workflow as signer, but currently
runs **Audit in every env**: in `Enforce` it rejects provably-valid attestations
(a `verifyImages` sigstore/rekor vs GitHub-Enterprise-transparency-log config
issue — fix that before re-enabling `Enforce`).

## Auth (what the pipeline needs)

- **GHCR writes** (cross-package image promote + chart push): the org's classic
  PAT **`ghcr-puller-pat`** from KV — despite the name it carries
  `repo, write:packages`, so it both pulls and pushes. A GitHub App installation
  token is accepted by the Packages REST API but **not** by the GHCR Docker
  registry, so a PAT is required. The promote loop fails fast, naming the
  service, if the PAT can't read a `:latest`.
- **Private chart pull** (CD): the terraform helm **v3** provider does not read an
  ambient `helm registry login`, so `_deploy-environment` passes
  `TF_VAR_ghcr_pull_token` (`ghcr-puller-pat`) into the module's `helm` provider
  `registries` block (`terraform-devplus-tenant`). The chart package stays
  private.

## What replaced the old flow

The legacy `image-tags.json` ledger — a `{service: sha}` map committed back to
`main` on every service push via the GitHub Contents API — is **gone** (it was
the source of the `sha wasn't supplied` 422, and terraform ignored it anyway).
The chart version is now the record of truth.
