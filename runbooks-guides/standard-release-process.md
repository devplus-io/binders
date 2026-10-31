---
id: 1d8724b6-e88d-4537-849e-6c9f48bddeb0
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
Standard Release Process

The five release rules (none can be skipped):
1. Code coverage on every repo in the release is at or above 80%.
2. Every Dependabot (SCA) alert is addressed: fixed, or dismissed with a written reason.
3. The platform test suite (`devplus-platform/tests`) has tests for every new feature and behaviour change in the release, merged before the release is cut.
4. The test suite reports no failures.
5. The release reaches the `test` tenant first in every tier (dev, stage, prod), and the test suite passes there before any real tenant gets it.

Rules 3 and 4 are both checked at Gate 3 (section 4).

This note says how to check each rule and run any release, X.Y.Z. `devplus-platform/docs/RELEASE.md` explains how the pipeline works (promotion by digest, version rules, troubleshooting). This note is the checklist. Where they disagree, stop and ask on the release card.

---

## 0. Ground rules for agents

- **One release card.** Every release has one card on the DevPlus board: "Release X.Y.Z". The release agent:
  - claims it (`claim_task`);
  - reports on it after every phase and every gate result (`update_task`), with run IDs and command output;
  - never lets the claim go quiet for 30 minutes during a release.
- **Ask before acting outside Binders.** Use `request_approval` on the card and wait for "Approved" before you:
  - merge a PR;
  - push a tag;
  - run Release or Deploy Tenants for stage or prod, or for any real (non-`test`) tenant;
  - dismiss or ignore a Dependabot alert;
  - deploy to a customer tenant;
  - send anything to a customer.
  Deploying the `test` tenant in dev needs no approval.
- **Push release branches as the Pennant App.** Push as `penant-agent[bot]`, using the Pennant App installation token. The `main` rulesets set `require_last_push_approval`, `dismiss_stale_reviews_on_push` and `require_extra_approval_for_unattributed_changes`, so a human who pushed last can't approve the PR. Rules for the bot's last push:
  - make it a real change (an empty commit does not count; use a one-line README comment);
  - make it before the founder approves, because a push dismisses earlier approvals;
  - expect a second founder to approve app-opened PRs if review stays REVIEW_REQUIRED after the first approval.
- **The founders approve.** A founder approves every release PR and the prod gate. Agents never approve PRs.
- **Stop on red.** If any check below fails, stop. Record the exact output on the card, move the card to Blocked, and ask. Don't work around a gate: no admin-override merges, no new audit `--ignore` entries, no `quality_gate=never`, no skipping the canary.
- **Use versions exactly as published.** Copy every version string from GitHub (newest tag plus its publish timestamp). Never type one from memory or from an early log line.
- **Check, don't assume, that the gates ran.** A rule can only be counted as met when you've seen its result. Branch rulesets may not require every status check, so read the checks yourself before every merge (section 4).

---

## 1. Scope and code freeze

1. **Make the release card.** Create "Release X.Y.Z" and list in it:
   - the features (link the release's note in Releases);
   - the repos involved;
   - the target dates;
   - a **Feature → test** table: one row per new feature or behaviour change, to be filled in with the platform suite tests that cover it (Gate 3, step A).
2. **Freeze.** After the freeze, only bug fixes, security fixes and performance work merge into `release/X.Y.Z`. Anything else waits for the next release.
3. **Sweep open PRs** in every DevPlusInc repo:
   - `gh pr list -R DevPlusInc/<repo> --state open`
   - For each PR, decide one of: bundle into `release/X.Y.Z`, defer to the next release, or close as stale. Record every decision on the card.
   - Open Dependabot PRs are always bundled (they satisfy Gate 2).
4. **Cut the branches.** Create `release/X.Y.Z` from `main` in each repo in scope, pushed as the Pennant App.
5. **Note the release-specific steps on the card:** data migrations, new required config or secrets, executor-job draining, wire-contract changes between services. These feed the upgrade and rollback notes in the CHANGELOG.

---

## 2. Gate 1: coverage ≥ 80% in every repo in the release

Measure on the `release/X.Y.Z` head, over the whole repo. Exclude only generated code (OpenAPI/SDK clients, `*_gen.go`, generated `schema.d.ts`). Don't rely on a CI coverage number alone: a CI gate can be scoped to a subset of files and report more than the whole repo. Record both the whole-repo number and the CI number on the card for each repo.

| Stack | Command (run in the repo root) |
|---|---|
| Go (services, CLI, SDK, Terraform provider, shared) | `GOWORK=off go test -short -covermode=atomic -coverprofile=cover.out ./... && go tool cover -func=cover.out \| tail -1` (filter generated files out of `cover.out` first) |
| Python (agent-service, compliance-scripts) | `uv run pytest --cov --cov-report=term --cov-fail-under=80` |
| Frontend | `bun run test:coverage`. The coverage include list must cover the whole SPA source tree, not a hand-picked set of files. |
| Terraform modules | No line coverage. Gate 1 means `tofu validate` plus the module's tests or plan checks pass. |

**Pass:** every repo in scope is at 80% or more, and none went down from the previous release.

**Fail:** the release is blocked. The only way past it is a written exception from a founder on the release card that:
- names the repo and its current number;
- requires that the number did not go down from the last release;
- links the board card that raises it to 80%.
An exception covers this release only; it never carries over.

---

## 3. Gate 2: no unaddressed Dependabot / SCA alerts

1. **List the open alerts** for the whole org:
   ```
   gh api --paginate '/orgs/DevPlusInc/dependabot/alerts?state=open&per_page=100' \
     --jq '.[] | [.repository.name, .security_advisory.severity, .dependency.package.name, .dependency.manifest_path, (.security_vulnerability.first_patched_version.identifier // "none"), .security_advisory.ghsa_id] | @tsv'
   ```
   Put the list for the repos in scope on the card.
2. **Fix each alert on the release branch** by bumping to the first patched version (or later). Then run that repo's full tests and audit:
   - Go: `govulncheck ./...`
   - Python: `pip-audit`
   - frontend: `bun audit`
   Fold the fixes into that repo's release PR, labelled `security`. Critical and high alerts are always fixed, never dismissed, unless no patched version exists. In that case, write down the mitigation and get a founder's approval.
3. **Dismiss only when the code can't reach the vulnerable path** (e.g. a platform the product never runs on, or a dev-only tool). Ask for a founder's approval on the card first. Then dismiss with a reason: `gh api -X PATCH /repos/DevPlusInc/<repo>/dependabot/alerts/<n> -f state=dismissed -f dismissed_reason=<not_used|inaccurate|tolerable_risk|no_bandwidth> -f dismissed_comment="<why, and when to revisit>"`.
4. **Treat CI ignore lists as dismissals too.** Re-review every `--ignore GHSA-…` (or similar) entry in every workflow each release. Keep an entry only if a founder re-approves it on the card, with the date or event that removes it.
5. **Cover repos Dependabot can't see.** For any repo in scope without a `.github/dependabot.yml`, or whose ecosystem Dependabot doesn't cover, run its audit command by hand at this gate and record the output.
6. **Pass:**
   - the step 1 query returns nothing for any repo in the release;
   - every audit step in every main build is green;
   - every remaining ignore entry was re-approved this release.
   A red audit step means no image is pushed, so the release would quietly ship the previous `:latest`.

---

## 4. Gate 3: the test suite covers every new feature and reports no failures

### A. The platform suite covers every new feature (rule 3)

Each repo's unit tests don't satisfy this. The black-box suite in `devplus-platform/tests` must exercise each new feature through the deployed services, the way a tenant would.

1. **List what changed since the last release.** Build the list from these sources, not from memory:
   - the release's note in Releases (Features, and any bug fix that changes behaviour);
   - the backend API diff: `git diff <previous tag>..release/X.Y.Z -- api/openapi.json` in devplus-backend (new or changed operations, fields, status codes);
   - new Cedar actions (`internal/seed/seed.go`), executor job kinds, scheduled jobs, agent-service agents, cards and tools, and SPA routes (new `app/**/page.tsx`);
   - new env vars or config the services read.

   Put every item in the card's Feature → test table.
2. **Write the tests on a devplus-platform branch** (`tests/X.Y.Z`), one or more per item, at the layer that fits:

   | What changed | Where the test goes |
   |---|---|
   | New or changed API operation | `contract/`: the coverage ledger fails until the new operation is exercised. Add its role tier to `contract/expectations.go`; the spec-driven authorization and input sweeps (`authz_sweep_test.go`, `input_abuse_test.go`) then cover it automatically. The spec must carry its `x-devplus-action`. |
   | New behaviour, state machine, background job, or a flow across services | `regression/`: a journey that asserts the outcome, including refusals and cleanup. |
   | New AI agent, card or tool | `regression/ai_*`: script mock-aoai (`toolCalls`, `structured`, `faults`) to reach the card. Assert approve, reject and a stale answer, and what was or wasn't saved. |
   | New UI screen or control | `e2e/`: a Playwright journey per role that can see it, and the screen added to `accessibility.spec.ts`. |
   | A hot path or a new lock or queue | `perf/`: extend a k6 suite and its thresholds. |

   If a mock can't reach the feature, extend the mock in `tests/mocks` in the same branch rather than leaving the feature untested.
3. **Bug fixes in the release:** each fixed defect's pinned test (it logs `KNOWN DEFECT` and fails once the fix lands) is flipped to a hard assertion, and its row in `tests/README.md` moves to the fixed table. A fix without a pin gets a new regression test that fails before the fix.
4. **Live mode:** every new test either runs against the `test` tenant or skips with a stated reason (it needs a mock or direct database access). Run `make -C tests test TARGET=dev` once against the dev `test` tenant before cutting, and record the result on the card.
5. **Document and merge:** add a section for the release to `tests/README.md` (what's covered, new mock options, defects found and their issues), open the PR, and merge it before the release is cut. It merges with devplus-platform in the merge order (section 5). File any defect the new tests find as an issue, and list it in the release note.

**Pass:** every row in the Feature → test table names at least one test that is merged and green, and the dev run is recorded.

**Fail:** the release is blocked. The only way past it is a written founder exception on the card. It names the feature, says why it can't be tested yet, and links the board card that adds the test. It covers this release only.

### B. Everything is green (rule 4)

All of these must be green on the exact commits being released:

1. **Each repo's CI on the release PR head:** build/CI, CodeQL, Dependency Review, and any audit or coverage job. Check with `gh pr checks <n> -R DevPlusInc/<repo>`. Every check must pass before you ask a founder to merge. Don't count on the ruleset to block a red merge.
2. **The platform black-box suite in mock mode:** the Platform Test Matrix workflow in devplus-platform. The newest run on `release/X.Y.Z`, and then on `main` after merging, must be green: `gh run list -R DevPlusInc/devplus-platform --workflow platform-test-matrix.yml --limit 3`.
3. **Contract checks across repos:** the Helm chart against the env vars each service reads, and `tofu validate` on the tenant config. These can fail until the repos merge in order (section 5), so re-run them after each merge step. They must be green in the end.
4. **Known defects:** pinned defects are allowed. They are the tests that log `KNOWN DEFECT (pinned)` or are marked `test.fail()` / `knownDefect` in e2e, are listed in the pin tables in `tests/README.md` with their GitHub issues, and are the schema exemptions in `tests/contract/expectations.go` (`knownSchemaDefects`). List each one in the release notes. If a pinned defect changes behaviour (fixed or regressed), the suite fails. That counts as a failure: update the pin along with the fix (step A.3).

**Pass:** zero failing tests or checks. A flaky test still counts as a failure until it's fixed or quarantined with a board card.

---

## 5. Merge order (each step needs a founder's approval on its PR)

1. **devplus-shared**
   - Merge the PR, then tag its merge commit with the next version and push the tag.
2. **Bump the pins** in every Go service that uses the new shared code:
   - `GOWORK=off go get github.com/DevPlusInc/devplus-shared@<tag>`; never `go mod tidy` against an unpublished shared change;
   - verify with `GOWORK=off go build ./... && GOWORK=off go test ./...`.
   - If the backend API changed, regenerate the spec: `go run ./cmd/openapi-gen -out api/openapi.json`, copy it to the frontend's `openapi/openapi.json`, then run `bun run gen:api`.
3. **The Go services:** backend, executor, governance-service, oidc-service, opentofu-server, mcp.
4. **agent-service and frontend.** If the release changes the wire contract between them and the backend, merge them together.
5. **The Terraform modules, in this order:** terraform-helm-devplus-services → terraform-devplus-tenant (move its `?ref=` pin to the merged helm-services commit if it needs a new input) → devplus-platform. Then re-run devplus-platform's PR checks.
6. **Wait for every main-branch image build** before cutting. Each must read `completed/success`:
   ```
   for r in devplus-backend devplus-executor devplus-frontend devplus-agent-service devplus-oidc-service devplus-opentofu-server devplus-governance-service; do printf "%-28s " $r; gh run list -R DevPlusInc/$r --branch main --workflow build.yml --limit 1 --json status,conclusion -q '.[]|"\(.status)/\(.conclusion // "-")"'; done
   ```
   A failure or an in-progress build means that service's `:latest` is not the merged code. Stop.

---

## 6. Gate 4: the `test` tenant first, in every tier

The `test` canary tenant exists in every tier (`terraform/tenants/devplus/environments/test-<tier>.tfvars`).

**Before cutting, confirm the gate credentials:**
- The suite authenticates with the canary admin PAT: GitHub secret `DEVPLUS_TEST_PAT_<TIER>`, otherwise Key Vault `devplus-test-pat` in that tier's vault. As of 2026-10-25 no `kv-devplus-*` vault holds `devplus-test-pat`, so the GitHub secret is the only working source until the vault secret is stored. Locally, `run-suite.sh` / `make` read `DEVPLUS_PAT`, then `DEVPLUS_TEST_<TIER>` from your shell.
- Check it exists and still works for dev, stage and prod: every live stage of `run-suite.sh` stops up front with "rejected DEVPLUS_PAT (401)" when it doesn't.
- A missing or expired PAT makes the gate fail; it does not pass it.

**Hard rule:** the first deploy of a version in each tier uses `tfvars_name=test-<tier>`, with `quality_gate` left at `auto` or set to `always`. Never `never`, and never a real tenant first.
- On every Release or Deploy Tenants run, check that the "Quality gate (test tenant → <tier>)" job reads `success`. If it reads `skipped` or `failure`, stop.
- Always set `version` explicitly. Don't rely on the auto-computed pre-release number.

**Dev**
1. Run `Actions → Release` with:
   - `version=X.Y.Z-dev<N>`;
   - `source_tag=latest`, `deploy=true`, `environment=dev`, `tfvars_name=test-dev`.
2. Copy down the exact version that got published (newest chart tag plus timestamp) onto the card.
3. The gate runs smoke plus the OpenAPI contract tests against the dev `test` host. It must be green.
4. Check the pods: `kubectl --context aks-devplus-dev-eus2 get deploy,pods -n test`. Every image must be `:X.Y.Z-dev<N>`, and every pod `Running 1/1` with a fresh age.
5. If the release requires it, drain executor jobs first (`kubectl -n <tenant> get pods -l app=executor-job`; none may be Running). Then `Deploy Tenants` with `environment=dev`, `chart_version=X.Y.Z-dev<N>` and an empty `tfvars_name` (the real dev tenant). Check the pods there too.

**Stage** (approval required)
1. `Deploy Tenants` with `environment=stage`, the same `chart_version`, `tfvars_name=test-stage`.
2. The gate runs smoke, contract, the full regression suite and capped k6 against the stage `test` host. It must be green.
3. Check the pods in `aks-devplus-stage-eus2`, namespace `test`.
4. Drain executor jobs if required, then deploy to the real stage tenant (empty `tfvars_name`, namespace `app`), and check the pods.

**Prod** (approval required; the prod environment also requires a reviewer)
1. Run `Actions → Release` with `version=X.Y.Z`, `source_tag=X.Y.Z-dev<N>` (the exact version stage verified), `environment=prod`, `tfvars_name=test-prod`.
2. Before you click Run, check that the `source_tag` image digest equals the one stage ran (RELEASE.md, promotion cross-check).
3. The gate runs the non-destructive smoke test against the prod `test` host. It must be green.
4. Check that the promotion really promoted:
   - every service image carries both `X.Y.Z` and `X.Y.Z-dev<N>` (same digest);
   - the run log says the chart source is the published `X.Y.Z-dev<N>` chart (promoted, not rebuilt from main).
5. Drain executor jobs if required, then `Deploy Tenants` with `environment=prod`, `chart_version=X.Y.Z` for the `app` tenant.
6. Check the pods in `aks-devplus-prod-eus2`, namespaces `test` and `app`.

---

## 7. After prod

1. **CHANGELOG:** in `devplus-platform/CHANGELOG.md`, move `[Unreleased]` to `[X.Y.Z] - <date>`. Include the upgrade notes, rollback notes and known defects.
2. **Git tag and GitHub release** on devplus-platform:
   - `gh release create X.Y.Z -R DevPlusInc/devplus-platform --notes-file <changelog section>`
3. **Update docs:** umbrella `docs/` and CLAUDE.md, and the Binders DevOps notes the release changed.
4. **Regenerated clients:** if the API changed, the SDK, CLI, MCP and Terraform provider need regenerating. Do it now, or put them on the board as cards.
5. **Customer tenants:** with approval, deploy to each customer tenant, test-tenant-gated the same way where possible. Draft the customer email with the changelog on the card for approval before sending.
6. **Cleanup:** remove whatever the release made obsolete (secrets, flags, old resources). The owner does this, with approval.
7. **Retro:** add to this note any step that was missing or wrong during this release, and file cards for any gate that had to be checked by hand and could be automated.
8. **Close the card:** `complete_task` with the versions, run IDs, gate results for each tier, and any exceptions granted under Gates 1 and 2.

---

## 8. Rollback

`Deploy Tenants` with `environment=<tier>` and `chart_version=<previous version>`. Published versions can't be changed, so the previous one is still there.
- Use `plan_only: true` first if you want to see the diff.
- Rollback restores code and chart together. It does **not** roll back data (Mongo, Terraform state, cloud resources).
- Before shipping, every release must have rollback notes in the CHANGELOG saying what the previous version does with the new data and config. Read them before rolling back.
- Ask on the card before any stage or prod rollback.
