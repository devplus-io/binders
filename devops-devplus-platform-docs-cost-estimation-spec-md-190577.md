---
id: b4f020eb-5311-43fc-9829-e494f4190577
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/COST_ESTIMATION_SPEC.md

Source: DevPlusInc/devplus-platform docs/COST_ESTIMATION_SPEC.md (untracked draft 2026-09-14), refreshed 2026-10-12 against release/0.1.17

Updated 2026-10-12 against release/0.1.17 (devplus-executor, devplus-shared, devplus-backend, devplus-frontend, devplus-agent-service, devplus-platform, terraform-devplus-tenant, terraform-helm-devplus-services). Spec first written 2026-09-14 (PROPOSED, untracked in devplus-platform/docs). Revived 2026-10-12: the owner wants it built in release 0.1.17. Instant estimate from inputs (decision 7, section F) added the same day.

# Replace Infracost with a DevPlus-owned cost estimator

## Status (2026-10-12)
- **Decision:** build it now, in release 0.1.17. No Infracost code, service or key after cutover.
- **Already done in 0.1.17 (not part of this program any more):**
  - Budget gate fail-open is fixed. Backend `services/money.go` `ParseMoneyAmount` reads "$123.45", "EUR 1.234,56" and the rest; `handlers/internal_handler.go:478` uses it and rounds UP to whole units (`WholeCurrencyUnitsUp`), so $50.40 cannot pass `>= 51`. When a policy reads cost and the cost is unknown, the plan gate says so.
  - Real billed spend shipped as `cost_sync` (AWS Cost Explorer, Azure Cost Management, GCP BigQuery export), with `cost_lines`, the Spend tab and attribution. The old spec listed this as "later".
  - Labels: "Plan estimate" (`costEstimate`), "Applied estimate" (`costActual`), "Spend" / "Billed". Project cost summary `GET /projects/{pid}/cost/summary` and the Cost drivers card.
  - Agent `run_preview` now returns the preview and points to `get_deployment_cost_estimate` (the "always N/A" bug is gone).
  - Previews never count toward totals (0.1.17 previews work).
- **Still open:** everything below. The estimator itself, the price snapshot, the coverage census, the accuracy gate, removal of every Infracost wire, and the copy that still names Infracost.

## Why
- Infracost 2.0: the new CLI requires a login, v0.10 shows an upgrade notice, and `breakdown` is deprecated for `scan`.
- Self-hosting their prices is not realistic: `cloud-pricing-api` was removed from GitHub, and loading its data dump needs an Infracost key.
- Owner (2026-09-14): "I really don't want to continue to use Infracost for cost estimation ... alternative solutions that I can run all myself without having to rely on a vendor."

## The integration today (narrow)
- **One file runs Infracost:** `devplus-executor/services/cost_service.go` (421 lines, NO tests, not in the 80% gate). Two methods:
  - `EstimateCostFromPlan(ctx, execDir, planFile, currency)`
  - `EstimateCostFromState(ctx, execDir, currency)`
- **Five call sites** in `services/executor_service.go`; a failure is only a warning, and budget rules then see an unknown cost:
  - preview :969, deploy plan :1016, post-apply :1138 (`costActual`), adoption :1435, retrofit `retrofitCostEstimate` :1510.
- **Infracost CLI install is unpinned:** `releases/latest`, no checksum (`Dockerfile:43-48`, `Dockerfile.tilt:35-39`).
- **Currency:** `infracost configure set currency` per run. Infracost still reports USD in a EUR project in places (frontend `deploymentMeta.ts:175` works around it).
- **Downstream reads `models.CostEstimate`** (`devplus-shared/models/deployment.go:261-297`): `totalMonthlyCost`, `totalHourlyCost`, `currency`, `projects[].resources[]{address,type,name,monthlyCost,hourlyCost,costComponents[],metadata}`, `generatedAt`. Most consumers read only the total, currency and date. The deployment detail table and the backend cost summary (`cost_summary.go`, "unpriced, never zero") read per-resource rows.
- **Money strings:** the executor still writes "$123.45" (`cost_overrides.go:622` `formatMoney`). Harmless now that the backend parses it, but the new engine writes plain numbers.
- **DevPlus already owns a pricing DSL:** `devplus.hcl` `resource_override` (`cost_overrides*.go`): conditional and formula prices over module vars. It runs after Infracost and keeps running after the new engine.
- **Plan JSON is already produced** by `tofu show -json` (`cost_overrides_exec.go` `tofuShowPlanJSON`; 0.1.17 also added `tofu_json.go` / `plan_show.go` for the run view and drift diff).
- **Imported modules can hold any resource type.** Prod's only module mixes azurerm, mongodbatlas, cloudflare, kubernetes and helm. Most resources are free; cost sits in a small set of types.

## Decisions
Confirmed 2026-09-14:
1. **Approach:** our own estimator (option 1). No vendor.
2. **Cutover bar** (revised 2026-10-12, no real customers yet): every type on the v1 list is priced, and on real golden plans each priced resource lands **within 10%** of Infracost's figure for the same plan. The long tail is labeled "not priced". (The 2026-09-14 bar, "at least 95% of historical estimated dollars", was dropped because history is test data only.)
3. **Billed actuals:** separate (now shipped as `cost_sync`).

Added 2026-10-12:
4. **Timing:** in 0.1.17. The release waits on the cutover bar. No dual mode: the release that ships the engine removes Infracost.
5. **Snapshot image is private** (see A).
6. **GCP pricing deferred** until DevPlus has a GCP project and API key (see A).
7. **Instant estimate, no plan** (owner, 2026-10-12, "let's ship it now", in 0.1.17): price a deployment from its inputs plus the module's source, in milliseconds, for the deploy and update forms, the chat's deploy cards, catalog cards inside a project and the agents. The plan estimate stays the authority for the budget gate and the deployment record. See section F.

Build (2026-10-12): two 0.1.17 workstreams, both merged into release/0.1.17 the same day.
- `costest` (devplus-executor): `internal/pricing` (awsprices, azureprices, ecb; reader and writer), `internal/costest` (`mappers.yaml` plus Go mappers in `gomappers.go`), CostService swap with `CostOptions{AWSRegion}` at all 5 call sites (the applied estimate now prices the STATE, not the module HCL), Infracost removed from both Dockerfiles and the job env, `.github/workflows/pricing-snapshot.yml` plus `scripts/pricing-snapshot.sh sync|build|push`. Coverage gate 91.5%.
- `costcut`: backend cost summary reads `priceState`; frontend Cost card shows "Prices as of" and "N resources not priced", rows say Free / Not priced; `parseCost` sign fix; agent cost answer gives the price date and unpriced types; Tilt `infracost-secret` gone; both Terraform modules drop `infracost_api_key`; platform `docs/COST_ESTIMATION.md` (new) and COST_OVERRIDES fixed. `platform-test-matrix.yml` now logs in to GHCR with `ghcr-puller-pat` because the executor build pulls the private snapshot image.
- Full sync: 17 AWS and 22 Azure regions in about 6 minutes; AWS 129,962 rows (5.1 MB), Azure 145,716 rows (4.2 MB).
- Accuracy: 12 real golden plans (6 AWS, 6 Azure) against Infracost outputs from the executor image: every priced resource within 10% except two named exceptions (zone-redundant Postgres HA standby billed, Infracost omits it; Elastic Premium cost on the plan, not the app). OpenSearch +4.8% is the next-largest gap.
- Live on helder (Tilt aks, 2026-10-12): no infracost binary or env in the executor; previews in test2 priced an ACR item at $5.07 (Infracost $5.00), an Azure VM at $46.80 (Infracost $46.796), and the Tiny AKS cluster at $78.83 (1 x D2s_v6 + P4 OS disk + 3 metric alerts; the old Infracost plan figure of $157.26 was a double count, its applied figure was $78.73).
- First publish of the private image: the sync runs `go run .` over the whole executor, so it builds only after shared v1.17.0 is tagged and the pins are bumped; the push of executor release/0.1.17 then triggers it. Before that, `scripts/pricing-snapshot.sh build` plus `docker tag ...:local ...:latest` lets Tilt build from the local image store.
- No census: the owner said there are no real customers yet.

## Options considered
| Option | Verdict |
|---|---|
| **1. DevPlus-native estimator:** own price snapshot plus Go mappers | **Chosen.** No vendor, deterministic, offline, stdlib HTTP like `gcpauth`. The cost is resource coverage, built from real usage data. |
| 2. Pinned Infracost CLI v0.10.45 (Apache-2.0) plus a self-hosted pricing API from the IBM-Cloud fork | Fastest, keeps ~1,100 mappers, but still Infracost code on an abandoned upstream, plus a TypeScript and Postgres service to run. Fork scrapers unverified. |
| 3. C3X (Apache-2.0, 3 clouds) | Too young (v0.2, 25 stars, unknown maintainer). Ideas only. |
| 4. terracost (MIT, Go) | ~28 AWS, 15 Azure, 1 GCP resources; needs MySQL. Reference for price ingestion only. |
| 5. OpenInfraQuote | Archived 2026-07-22, AWS only, OCaml. No. |
| 6. LLM-estimated cost | Not deterministic; breaks "LLM off the data path". No. |
| 7. Cloud billing APIs | Real spend after the fact. Complements an estimate; shipped separately as `cost_sync`. |

## Design

### A. Price snapshot (data, no vendor)
**Sources (all first-party):**
- AWS bulk offer files (per service, per region): no auth.
- Azure Retail Prices API (`prices.azure.com/api/retail/prices`): no auth.
- GCP Cloud Billing Catalog API: needs an API key from a DevPlus-owned GCP project. **Deferred (2026-10-12):** DevPlus has no GCP project yet. In 0.1.17 GCP resources are "not priced" and the manifest records gcp `not_configured`; once a project and key exist, GCP is one new source package plus mappers.
- ECB daily reference rates: FX for EUR, GBP, JPY and the other `models.Currency` values; no auth.

**Sync job:** new executor mode `EXECUTOR_MODE=pricing-sync` (`internal/pricing`, one package per source, the `internal/<provider>scan` pattern).
- Streams each source, keeps only the services and product families the mappers reference (on-demand, shared tenancy, Linux and Windows).
- Normalizes each price to `{provider, service, family, region, attrs, unit, usdPrice, tierStart}`.
- Writes files partitioned by provider and region (`aws/us-east-1.jsonl.gz`, ...) plus `manifest.json` (generatedAt, source versions, FX rates).
- An estimate loads only the regions its plan touches.

**Distribution:**
- A weekly scheduled workflow in `devplus-executor` publishes a scratch image `ghcr.io/devplusinc/pricing-snapshot:<date>` plus `:latest`.
- **Private package** (owner, 2026-10-12: "Why would it need to be public?"). The image is published by devplus-executor's own workflow with label `org.opencontainers.image.source=https://github.com/DevPlusInc/devplus-executor`, so the package links to that repo and its CI pulls it with GITHUB_TOKEN. Tilt hosts need `docker login ghcr.io` (already an aks-mode step). Clusters never pull it: it is baked into the executor image.
- `Dockerfile` and `Dockerfile.tilt` `COPY --from` it, pinned by digest at build. Each executor image carries a fixed offline snapshot, promoted with the release's digest-preserving copy. No runtime pricing service.

### B. Estimator engine (`devplus-executor/internal/costest`)
**Input:** the plan JSON for estimates; the state JSON for post-apply (`costActual` prices what was actually applied).

**Resource walk:**
- Skip deletes and data sources.
- Resolve the region from provider config, `location`, `region` / `zone`, or the provider default.

**Mapper registry:**
- Declarative rules in embedded YAML: resource type -> cost components, each a price filter over resource attributes plus a quantity expression.
- Quantity expressions reuse the go-cty/HCL evaluator already in `cost_overrides_vars.go`.
- Go escape-hatch functions for complex resources (node pools, RDS multi-AZ, tiered disks).

**Every resource gets exactly one state:**
- **priced**
- **free:** a curated list (IAM, role assignments, identities, vnet/subnet/SG, `random_*`, `time_*`, `null_*`, `kubernetes_*`, `helm_release`, ...)
- **not priced:** shown in the UI, never counted as $0.

**Usage-based components** (storage GB, requests, egress): unit price with quantity 0 unless `devplus.hcl` sets a default.

**Currency:** USD converted with the snapshot's FX rates. Fixes "USD in a EUR project".

**Output:**
- The unchanged `models.CostEstimate` tree, so no consumer breaks. Money strings are plain numbers.
- Additive shared fields (done, devplus-shared 9df7fa1 on release/0.1.17): `CostEstimate.pricesAsOf` (YYYY-MM-DD), `pricedResourceCount`, `freeResourceCount`, `unpricedResourceCount`, `unpricedTypes[]`, and `CostResource.priceState` ("priced" | "free" | "not_priced"). A not-priced resource has an empty `monthlyCost`; a free one has "0". Shared PR #51 is on hold for this commit.

**Overrides and non-cloud providers:** the `devplus.hcl` layer runs after the engine, unchanged. mongodbatlas and cloudflare stay priced through `devplus.hcl`, as today (Infracost did not price them either).

### C. Coverage
- **No census (owner, 2026-10-12):** "We dont have real customers yet, so the census wont really be much helpful". Historical `cost_events` hold only test data, so they cannot rank types.
- **v1 list (fixed):** about 30 AWS and 30 Azure types: compute, managed Kubernetes and node pools, managed databases, disks and buckets, load balancers, NAT, public IPs, caches, key vaults, registries, app plans, AI accounts and deployments, monitor alerts. The full list is in the 0.1.17 brief `briefs/costest.md`; the shipped list goes into `devplus-platform/docs/COST_ESTIMATION.md`.
- Revisit with real `cost_events` once customers deploy. The census script is kept in the session scratchpad and needs the owner's OK to read tenant databases.

### D. Copy and contract (what 0.1.17 has not done yet)
- Executor writes plain numeric money strings (`formatMoney` and the zero fallbacks).
- Frontend: "Infracost list prices" -> "List prices" (`OverviewTab.tsx:439, 477`, `CostDisplay.tsx`, `CostDriversCard.tsx`, `lib/cost.ts`, `lib/types.ts`, `deploymentMeta.ts`). Show "N resources not priced" and the price date.
- Frontend `parseCost` drops minus signs (`dashboardData.ts:80`, `/[^0-9.]/`). Fix while there.
- Agent: `cost_driver_tools.py:6, 148` "list prices from Infracost" -> "list prices". Never name a vendor.
- Backend: route docs (`routes.go:1075, 2098`, regenerate openapi), comments, seed knowledge `well-architected-digest.md:37`.
- Shared: `deployment.go:31-32, 260` and `cost_spend.go:15` comments.

### E. Cutover and removal
**Executor:** swap `CostService` internals behind the same two methods. Drop the `INFRACOST_API_KEY` checks (`cost_service.go`, `executor_service.go` retrofit and others), the env injection (`job_dispatcher.go:927-930`), the CLI install in both Dockerfiles, README rows.

**Deploy wiring:**
- `devplus-platform/Tiltfile:614-617` (`infracost-secret` resource) and `:812` (`resource_deps`).
- `terraform-helm-devplus-services` `main.tf:132-145` (secret), `:303` (depends_on), `variables.tf:135`.
- `terraform-devplus-tenant` `main.tf:286`, `variables.tf:109`, README.
- `devplus-platform/terraform/tenants/devplus` `main.tf:126`, `variables.tf:142`, `test-stage.tfvars:12` comment, `_deploy-environment.yml:87`.

**Removal order:** the executor stops reading the key; the Terraform wiring stops creating the secret; the Key Vault secret `infracost-api-key` is deleted after the prod release (owner action).

**Docs:** `docs/` (executor, FLOWS, DATA_MODELS, GLOSSARY, ARCHITECTURE, DIAGRAMS, REPO_MAP), workspace `CLAUDE.md` §1, §3, §4, §5, `COST_OVERRIDES.md`, `TILT.md`, `ARCHITECT_AGENT_SPEC.md`, `MIGRATION_RUNBOOK.md`, `SECRETS_MANAGEMENT_SPEC.md`, `CHANGELOG.md`, and the binder entries for each. Public website names no vendor.

### F. Instant estimate from inputs (decision 7; built and live-tested 2026-10-12)
**Why it is possible now:** Infracost needed a plan. The DevPlus estimator only needs each resource's type and attributes, so a static evaluation of the module's HCL with the form's inputs feeds the same engine, mappers and snapshot.

**Pieces (four 0.1.17 workstreams):**
- **Engine moves to devplus-shared** so the backend can use it: `pricing` (format and reader; the executor keeps the sources and the sync), `costest`, `costest/overrides` (devplus.hcl), and `tfinputs` (how `Deployment.Inputs` strings become module arguments, called by both the executor's `createMainTF` and the evaluator, so they cannot disagree).
- **Evaluator** `costest/configeval`: rebuilds the root the executor writes (`module "catalog_item"`, root providers, connector AWS region), evaluates variables, locals, `count`/`for_each`, conditionals, `dynamic` blocks, module calls and the common built-in functions. Data sources and computed attributes are unknown. A resource whose cost needs an unknown value is `not_priced` with reason `known_after_plan`. Azure location unknown (resource group data source): the location every resolved Azure resource agrees on, else unknown. Limits: 5,000 instances, 50 nesting levels; past them "too large, run a preview". Parity test: equal to the plan estimate on every golden case.
- **Module source bundle** (executor): at each module analysis, and on demand through a new module job action `bundle`, the executor stores the module root plus its installed submodules (`tofu get` layout, `.terraform/modules/modules.json`) as a gzip tar in `module_source_bundles` (one per module, current when ref and source commit match). The backend never clones or downloads anything.
- **API** (backend): `POST /api/v1/projects/{projectId}/deployments/estimate` (opID and Cedar action `EstimateProjectDeploymentCost`, seeded where `RunProjectPreview` is). Same input merge as preview and create. Returns `status` ready, preparing (bundle being built, poll) or unavailable, the estimate, the count priced only after the plan, for updates the current total and the monthly delta, and a budget advisory (which budget policies would block it at the plan gate). The backend image carries the same price snapshot as the executor.
- **UI and agents:** a debounced price line under the deploy form ("About $X/month", price date, not-priced count), the update form's delta, catalog cards "From $X/month" at defaults, and an agent tool `estimate_deployment_cost` used once when drafting a deploy or update.

**Owner action:** grant devplus-backend read access to the private `pricing-snapshot` package (its CI pulls the image); the package is linked only to devplus-executor, so the backend's own token cannot read it.

**Built and merged into release/0.1.17 (2026-10-12):** shared `pricing`, `costest`, `costest/overrides`, `costest/configeval`, `costest/costtest`, `tfinputs`, `models.AWSRegionFor`, `models/quick_cost.go`, `models/module_source_bundle.go`; executor `internal/pricingsync`, bundle builder and the `bundle` module job; backend `POST /projects/{pid}/deployments/estimate` (Cedar `EstimateProjectDeploymentCost` seeded in the user Write and Admin roles, CostAgent, ArchitectAgent, DeployAgent and UpdateAgent); frontend `QuickCostEstimate` in the deploy and update modals, the chat deploy, batch and update cards, and catalog cards; agent tool `estimate_deployment_cost`.
- Parity: all 16 golden cases (12 original plus count/for_each, local submodule, terraform-aws-modules/vpc, form inputs) price exactly as their plans, resource by resource. Evaluation 0.3 to 2.5 ms per module, 12 ms for the VPC registry module; about 33 ms over HTTP. A full snapshot holds 68 MB once every region is loaded.
- **Update change semantics (changed after the live test):** "current" is the deployment's current inputs priced the same instant way, not its stored estimate, so an untouched update form shows no change. Stored estimates from before 0.1.17 are Infracost figures (Tiny AKS stored 157.26, a double count), which made an untouched form read -78 a month. The stored estimate is only the fallback when the current inputs cannot be priced.
- **Live on helder (Tilt aks):** six module bundles built on first catalog view (under 0.1 s each); catalog cards "From $8.23/month" (EC2) and "From $35.47/month" (Tiny AKS at its B2s preset); the deploy form with Standard_D2s_v6 shows $78.83, exactly the earlier plan-based preview; ACR update Basic to Premium shows "+$45.62/month from $5.07/month"; the cost agent prices by chat, and after a wrong input name it gets the item's real input names and retries.
- Found while testing (not cost work): the Tiny AKS catalog item presets `node_vm_size = "Standard_B2s"` while the module says B-series is not allowed in that subscription, and the deploy form shows string presets with literal quotes.

## Ship order (in 0.1.17)
1. **Census (C)** first, so the mapper list comes from data.
2. **Snapshot pipeline (A)** and **engine plus v1 mappers (B)** in parallel.
3. **Accuracy gate:** compare against Infracost figures on the same captured plans (decision 2).
4. **Copy (D) and removal (E)** in the same release branches.
5. Live check on helder (Tilt aks), then the release flow as for the rest of 0.1.17.

## Verification
- **Unit:** a golden `tofu show -json` per mapper from small real modules per cloud, with the expected components; a small checked-in snapshot subset; `internal/costest/` and `internal/pricing/` added to the executor `LOGIC_INCLUDE` 80% gate. Source fixtures from the real APIs (captured), never hand-written.
- **Price sanity:** spot-check about 20 SKUs against the official calculators: m5.large us-east-1, Standard_D4s_v5 eastus, e2-standard-4 us-central1, NAT gateway, EKS/AKS control plane, gp3 and Premium SSD disks.
- **Accuracy:** representative catalog items, within 10% of Infracost on priced resources; dollar coverage at least 95%.
- **End to end (Tilt aks, helder):** a preview shows the estimate with the price date and the not-priced count; a Cedar "Cost Limit per Deployment" policy blocks an over-budget apply; CLI and MCP still show totals; a plan with an unsupported type lists it as not priced; `GOWORK=off go build/test` passes in every Go repo touched.

## Later (separate)
- A price-lookup API for the Architect agent, as structured facts ("what does a D4s_v5 cost in eastus").
- Reserved and savings-plan pricing, spot.
