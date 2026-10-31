---
id: 7a2143a3-f753-4e78-a1fb-873f7712dce7
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/docs/RELEASE.md

Source: DevPlusInc/devplus-platform/docs/RELEASE.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/RELEASE.md).

# Releasing DevPlus

The runbook: how to ship code to dev, stage, and prod. For *how the pipeline
works* (why promote instead of rebuild, attestation, auth), see
[CICD.md](CICD.md) — this doc is what you **do**.

---

## The one thing to understand

**A version is an artifact, not a destination.**

`0.1.0-dev7` means "the 7th unverified pre-release". It does **not** mean "the dev
environment's build". Any version can be deployed to any environment, and the
same version is what moves from dev → stage → prod.

A release has **two halves that must stay together**: the **images** (8: the 7
services plus `devplus-mcp`) and the **chart** (the Kubernetes templates that wrap them). Promotion re-tags
*both* by digest. You never rebuild anything on the way to prod — the bits that
reach customers are byte-identical to the bits you verified.

---

## Shipping something: the normal path

### 0. Before you cut anything

Every service you merged must have finished its **main-branch image build**. A
release promotes each service's `:latest` **by digest** — cut too early and you
will silently ship the *pre-merge* images under a shiny new version number.

```bash
for r in devplus-backend devplus-executor devplus-frontend devplus-agent-service \
         devplus-oidc-service devplus-opentofu-server devplus-governance-service devplus-mcp; do
  printf "%-28s " "$r"
  gh run list -R DevPlusInc/$r --branch main --workflow build.yml --limit 1 \
    --json status,conclusion -q '.[] | "\(.status)/\(.conclusion // "-")"'
done
```

All must read `completed/success`. (Only services you actually changed matter —
the rest keep whatever `:latest` they already had.)

### 1. Cut a pre-release and deploy it to dev

`Actions → Release → Run workflow`:

| input | value |
|---|---|
| `version` | *(empty — auto-computes the next `0.1.0-dev<N>`)* |
| `source_tag` | `latest` |
| `deploy` | ✅ |
| `environment` | `dev` |
| `tfvars_name` | `test-dev` ← deploy the **canary first** |

This promotes all 8 images `:latest → :0.1.0-dev<N>` by digest, packages the
chart at that version, pushes it to `oci://ghcr.io/devplusinc/charts`, and
deploys it to the `test` canary tenant in dev — and the **quality gate runs
automatically** (see "The quality gate" below). Only when it is green, deploy
the same version to the real dev tenant: `Actions → Deploy Tenants` →
`environment=dev`, `chart_version=0.1.0-dev<N>` (tfvars_name empty).

**Confirm the version it actually cut — do not assume the number.** The fast loop
auto-computes `MAX(existing -dev<N>) + 1`, so if a stale `-dev7` was lying around
from a week ago, today's cut is `-dev8`, not the "next" one you had in your head.
Read it from the newest published tag (and note the timestamp), *not* from the
run's early "Resolve release version" log — that step *lists existing tags* to
find the max, and grepping it will happily hand you the wrong one:

```bash
gh api "/orgs/DevPlusInc/packages/container/charts%2Fdevplus/versions" \
  --jq '.[0].metadata.container.tags[0]  + "   (published " + .[0].updated_at + ")"'
# 0.1.0-dev8   (published 2026-07-18T02:40:55Z)   ← THIS is your version
```

Write that exact string down. It is the `source_tag` you will promote from in
step 4 — getting it wrong ships a different artifact than the one you verify.

### 2. Verify dev — the quality gate, then the pod

A green terraform apply proves *terraform* succeeded. It does **not** prove your
code is running: a pinned or stale image tag will apply green all day while the
cluster runs something else entirely. This has bitten us before ([WorkOS
multi-env auth](CICD.md#auth-what-the-pipeline-needs) — a "successful" deploy
shipped config with none of the code).

The **quality gate** is the first line: the `test-dev` deploy in step 1 ran the
smoke probe (every service, every route family, a lifecycle ping through the
ingress) and the OpenAPI contract harness (every operation in the deployed
backend's spec, 207 at 0.1.16: 401 sweep, Cedar, secret redaction, schema
conformity) against `https://test.dev.devplus.io`
with the deployed chart. A red gate blocks promotion — read the JUnit summary on
the run and the `quality-gate-dev-<run>` artifact; do not deploy further.

Then still look at the pod, because the gate proves behaviour, not provenance:

```bash
V=0.1.0-dev7   # the version you just cut

kubectl --context aks-devplus-dev-eus2 get deploy -n test \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image' --no-headers
kubectl --context aks-devplus-dev-eus2 get pods -n test
```

Every service must show `:$V`, and every pod must be `Running 1/1` with a fresh
age. Feature-specific manual checks are for the feature you shipped, not for the
platform — the gate covers that.

### 3. Deploy the same version to the stage canary, then to stage

`Actions → Deploy Tenants → Run workflow`: `environment=stage`,
`chart_version=0.1.0-dev7`, `tfvars_name=test-stage`.

The stage gate is the full one: smoke, contract, the **complete regression
suite** (deployment lifecycle through the real executor, WebSocket log
streaming over Cloudflare, a real Azure OpenAI conversation over SSE, Cedar
hot-reload observed through the backend, API keys, identity, secrets, RAG) and
the **k6 load suites** in their capped live profile, all against
`https://test.stage.devplus.io`. Stage is the last place a mistake is cheap;
this is where it gets caught. When green, deploy the same version to the `app`
stage tenant (`tfvars_name` empty) and re-run the pod check against
`aks-devplus-stage-eus2` (namespace `app`).

### 4. Promote to a stable version and deploy to prod

`Actions → Release → Run workflow`:

| input | value |
|---|---|
| `version` | `0.1.0` ← the stable number |
| `source_tag` | `0.1.0-dev7` ← **the version stage's gate verified** |
| `deploy` | ✅ |
| `environment` | `prod` |
| `tfvars_name` | `test-prod` ← canary first, again |

Prod's gate is the non-destructive smoke probe against `https://test.devplus.io`
(it creates and deletes one temporary project and one preview; no load, no
regression writes). When green, `Actions → Deploy Tenants` → `environment=prod`,
`chart_version=0.1.0` for the `app` tenant.

Nothing is rebuilt. The images are re-tagged **by digest**, and the chart is
pulled from the `0.1.0-dev7` release and re-versioned. Prod gets exactly what
stage ran, under a real version number.

**Before you click Run, confirm `source_tag` is the pre-release you actually cut
and verified** (the string you wrote down in step 1), not a plausible-looking
older one. A quick sanity check — the pre-release's image digest should equal the
current `:latest` you built from:

```bash
S=0.1.0-dev8   # the source_tag you're about to promote
gh api "/orgs/DevPlusInc/packages/container/backend/versions" \
  --jq "[.[] | select((.metadata.container.tags // []) | index(\"$S\"))][0].name as \$s
        | [.[] | select((.metadata.container.tags // []) | index(\"latest\"))][0].name as \$l
        | if \$s == \$l then \"OK: \($S) == :latest\" else \"STOP: \($S) != :latest — wrong/stale source\" end"
```

### 5. Verify the promotion actually promoted

Two checks. **The same image digest must carry both tags:**

```bash
for svc in backend executor frontend agent-service governance-service oidc-service opentofu-server devplus-mcp; do
  printf "%-20s " "$svc"
  gh api --paginate "/orgs/DevPlusInc/packages/container/$svc/versions" \
    --jq '[.[] | select((.metadata.container.tags // []) | index("0.1.0"))][0].metadata.container.tags | sort | join(",")'
done
```

Each line must list **both** `0.1.0` and `0.1.0-dev7` — same digest, two names.
If a service shows `0.1.0` *without* the pre-release tag, it was rebuilt, not
promoted. Stop.

**And the chart must have come from the release, not from `main`:**

```bash
gh run view <run-id> -R DevPlusInc/devplus-platform --log | grep "chart source"
# chart source: published chart 0.1.0-dev7 (promoted, not rebuilt from main)
```

Then verify prod pods (step 2, `--context aks-devplus-prod-eus2`, namespaces
`test` then `app`). The canary gate already hit the live endpoints; the same
probe can be run by hand. The PAT belongs to one tenant (keys live in that
tenant's database), so `devplus-test-pat` only works against `test`; for `app`,
use an admin PAT minted on `app`:

```bash
export BACKEND_URL=https://test.devplus.io FRONTEND_URL=https://test.devplus.io TEST_MODE=live
export DEVPLUS_PAT="$(az keyvault secret show --vault-name kv-devplus-prod --name devplus-test-pat --query value -o tsv)"
( cd devplus-platform/tests && go run ./cmd/seed -mode=api && go run ./cmd/smoke -target=edge )
```

---

## The quality gate

The `test` tenant (`terraform/tenants/devplus/environments/test-{dev,stage,prod}.tfvars`)
is a long-lived synthetic tenant in every environment with live cloud
connections. Deploying it (`tfvars_name=test-<env>`) triggers the `quality-gate`
job in `_deploy-environment.yml`, which runs the platform black-box suite in
`tests/` against the tenant's public host and **fails the workflow on any
failure**, which is what blocks the promotion:

| environment | host | stages | budget |
|---|---|---|---|
| dev | `https://test.dev.devplus.io` | smoke + OpenAPI contract | ~6 min |
| stage | `https://test.stage.devplus.io` | smoke + contract + full regression + k6 (capped) | ~25 min |
| prod | `https://test.devplus.io` | non-destructive smoke | ~2 min |

- The suite authenticates with the canary's admin PAT: GitHub Actions secret
  `DEVPLUS_TEST_PAT_<ENV>` first, else Key Vault `devplus-test-pat` in that
  environment's vault. Bootstrapping and rotation: `tests/README.md`.
- `quality_gate=always` runs it after any deploy (e.g. after deploying the
  `app` tenant, to re-check the canary); `never` skips it. Default `auto`.
- Everything the gate creates it deletes (temporary projects, previews, the
  bounded set of real deployments in the k6 stage). Failed executor jobs in
  the canary's history are expected: the seed module has no cloud connector.
- Run it locally with the same script the workflow uses:
  `tests/ci/run-suite.sh gate dev|stage|prod`.

---

## Rolling back

Redeploy the previous version. Published versions are **immutable**, so the
previous one is still sitting in GHCR exactly as it was:

`Actions → Deploy Tenants` → `environment=prod`, `chart_version=<previous>`.

This reverts **code and templates together** (that's the point of shipping them
as one artifact). It does **not** undo anything the old code did to data —
Mongo documents, terraform state, provisioned cloud resources. If a release
migrated data, rolling the image back does not roll the data back. Think about
that *before* you ship, not during.

Use `plan_only: true` for a dry run if you want to see the tofu diff first.

---

## Rules that keep this safe

- **Published versions are immutable.** The workflow refuses to re-point an
  existing tag. To change what a release contains, cut a new version.
- **Never promote from a bare image tag** (`sha-abc123`). It's refused — no chart
  travels with it, and a chart/image mismatch is invisible (both halves would
  carry the new version) and lethal (templates and images are versioned together;
  e.g. the SPA migration deleted `WORKOS_*` from `frontend.yaml` and moved the
  probe to `/healthz` — those templates around an older image yield a pod that
  never goes Ready).
- **Deploys are deliberate.** Merging to `main` only refreshes a service's
  `:latest`. Nothing reaches any environment until someone cuts a Release.
- **Verify the pod, never the apply.** And **never skip the canary**: the
  `test` tenant sees every version before a real tenant does.
- **Record every release in `CHANGELOG.md`.** Move the `[Unreleased]` entries
  under `## [<version>] - <date>` (GHCR publish date, UTC) when you cut a stable
  version, and say which pre-release it was promoted from.

## Version numbers

- Any SemVer: `0.1.0`, `0.2.0-rc.1`, `0.1.0-dev7`.
- **`+buildmetadata` is rejected.** It's legal SemVer but **illegal in a Docker
  tag** — and this one string is *both* the chart version and every image tag, so
  `helm push` would succeed and all 8 image promotes would then fail.
- `CHART_BASE` in `release.yml` only seeds the **auto-computed** `<base>-dev<N>`.
  An explicit `version` ignores it.

> **Open:** `CHART_BASE` is still `0.1.0` while stable releases have reached
> `0.1.16`. SemVer ranks every `0.1.0-dev<N>` *below* `0.1.0`, so auto-computed
> pre-releases (`0.1.0-dev15` was the last) sort behind every stable release.
> Harmless (deploys pin exact versions). The 0.1.16 cycle passed an explicit
> `version=0.1.16-dev2` instead; bump `CHART_BASE` or keep doing that.

## When it goes wrong

| symptom | cause |
|---|---|
| `source_tag '<x>' is not a published chart version` | You tried to promote from a raw image tag. Promote from a version cut by Release, or use `source_tag=latest` to cut from current `main`. |
| `version '<x>' is not a SemVer release` | Bad version string — or you used `+buildmetadata` (illegal in a Docker tag). |
| `<version> is already published — bump the version` | Working as designed. Versions are immutable. |
| `cannot read ghcr.io/devplusinc/<svc>:latest` | Either the service's main build hasn't finished (see step 0), or `ghcr-puller-pat` lost access. |
| GHCR `401` on push | A GitHub **App** token was used. The Packages REST API accepts those; the GHCR **Docker registry does not**. Use `ghcr-puller-pat` (it carries `repo, write:packages`). |
| Deploy is green but the feature isn't there | The classic. Check the running pod's image tag (step 2). |
| Quality gate red: `DEVPLUS_PAT rejected` / smoke `401` | The canary's PAT expired or was revoked. Mint a new one as the tenant admin (Settings → API keys) and update `DEVPLUS_TEST_PAT_<ENV>` / Key Vault `devplus-test-pat`. |
| Quality gate red: contract harness `known defect changed behaviour` | A pinned defect was fixed (or regressed). Remove/adjust the entry in `tests/contract/expectations.go` together with the fix. |
| Quality gate red: `TestLiveExecutorPicksUpDeployment` timed out | The executor dispatcher in the canary is not leasing work — check `executor-job-dispatcher` pods and the Atlas OIDC identity for namespace `test`. |
| Prod deploy green but running **old** code / a feature *regressed* / provisioning broke | You promoted from the **wrong `-dev<N>`** — a stale one, not the one you cut. The release is a faithful copy of whatever `source_tag` you named. Confirm the source (step 4 cross-check); cut a new version from the correct pre-release. |

---

## Worked example — the `0.1.0` release (2026-07-13)

The first stable release, shipping per-connector cloud identities:

1. Merged 5 repos; waited for every main-branch image build.
2. Release: `version=""`, `source_tag=latest`, `environment=dev` → cut
   **`0.1.0-dev7`**, deployed to dev. Verified all 7 pods on `0.1.0-dev7`.
3. Deploy Tenants: `environment=stage`, `chart_version=0.1.0-dev7`. Verified.
4. Release: `version=0.1.0`, `source_tag=0.1.0-dev7`, `environment=prod`.
5. Confirmed each image digest carried **both** tags, and the log read
   `chart source: published chart 0.1.0-dev7 (promoted, not rebuilt from main)`.

Prod ran the exact bits stage had proven.

## Postmortem — `0.1.4` → `0.1.5` (2026-07-18): promoting the wrong pre-release

A cautionary tale for why step 1's "confirm the version" and step 4's cross-check
exist.

- A fast-loop cut auto-computed **`0.1.0-dev8`** (an old `0.1.0-dev7` from five days
  earlier was still the max, so `+1` landed on `dev8`).
- The prod promotion was run with **`source_tag=0.1.0-dev7`** — the *stale* tag,
  assumed to be "the one I just cut". `0.1.4` therefore faithfully shipped the
  **July-13** artifact to prod.
- Fallout: `0.1.4`'s chart predated `control-plane-rbac.yaml`, so the prod deploy
  **deleted app-prod's provisioner ClusterRoleBinding** (`app-app` lost
  `cluster-admin`) and tenant provisioning began failing with
  `namespaces "…" is forbidden`. It also silently reverted `0.1.1`–`0.1.3`.
- The tell was in GHCR: `0.1.0-dev8` published `02:40`, `0.1.0-dev7` published five
  days prior. The `source_tag` named the old one.
- Fix: cut **`0.1.5` from `0.1.0-dev8`**, deploy prod. The correct chart re-rendered
  the ClusterRoleBinding; verified on-cluster that `app-app` could again get/create
  namespaces and that pods ran `:0.1.5`. `0.1.4` is immutable and burned.

Lesson: **the release is only ever as correct as its `source_tag`.** Confirm the
exact `-dev<N>` you cut (newest tag + timestamp), and never trust a remembered
version number.
