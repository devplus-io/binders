---
id: 695471c0-098d-40de-9196-3d8a6973c060
binder: binder-mvvle32a-1
---
# Microsoft Defender for Cloud Remediation & Hardening (V0.18)

**Status:** Planned for V0.18
**Priority:** P1 (Security & Compliance Posture)
**Affected Repositories:** `devplus-opentofu-server`, `devplus-governance-service`, `terraform-provider-devplus`, `devplus-core-infra`, `devplus-public-website`, `devplus-agent-service`, `devplus-executor`, `devplus-backend`, `devplus-shared`, `devplus-platform`, `devplus-hub-infra`, `devplus-base-infra`

---

## Overview & Motivation

A full query of Microsoft Defender for Cloud (`microsoft.security/assessments` where `status.code == "Unhealthy"`) across all 5 Azure subscriptions (`devplus-prod`, `devplus-stage`, `devplus-dev`, `devplus-sandbox`, and `Subscription 1`) and the GitHub DevOps security connector (`devplus-defender`) surfaced a set of high-leverage, low-effort security findings not yet captured in V0.18 or excluded in `devplus-hub-infra/docs/DEFENDER-FINDINGS.md`.

Resolving these 5 categories eliminates **8 dependency CVEs (6 High, 1 Medium, 1 Low)**, **19 High/Medium CodeQL & Bandit code-scanning alerts**, **~130+ Checkov false-positive IaC alerts on unit-test fixtures and dead manifests**, **7 Azure infrastructure posture findings**, and **~1,360 High/Medium AKS runtime container findings**.

---

## Scope & Implementation Details

### 1. Dependency CVE Bumps (6 High, 1 Medium, 1 Low)
Resolve all open Dependabot / Microsoft Security DevOps (MSDO) vulnerable package assessments:
- **`github.com/klauspost/compress` (High — `f15eb086-ccd1-cda0-a413-287c8ef57e21`)**:
  - Bump in `devplus-opentofu-server/go.mod` and `devplus-governance-service/go.mod`.
- **`google.golang.org/grpc` & `golang.org/x/crypto` (High — `3d63de4a-a8f1-88c9-f3c9-b8af44738c2f`, `2fff3a59-d996-42dc-271e-e841b676537e`)**:
  - Bump in `terraform-provider-devplus/go.mod` (`grpc` + `x/crypto`) and `devplus-core-infra` (`grpc` in `auth-service` / `github-callback`).
- **`vite`, `source-map-js` (High) & `postcss-selector-parser` (Medium)**:
  - Bump in `devplus-public-website/package.json` (`da66a04d-63af-1247-e6c4-8a5eb72cd9c3`, `3da59a95-e942-4334-58ec-f13285c8dc09`, `166a06ba-f2e3-7f90-96a3-0f3435a19960`).
- **`black` (Low — `759efd1f-803c-cc54-6145-6ac7fac1ac89`)**:
  - Bump in `devplus-agent-service/pyproject.toml` and `uv.lock`.

---

### 2. CodeQL & Bandit Static Analysis Fixes (11 High, 8 Medium)
- **`devplus-executor` — CodeQL `go/allocation-size-overflow` (Medium, 1 finding)**:
  - File: `services/deployment_secret_inputs.go:95` (`withSecretInputs`).
  - Fix: Replace `make(map[string]string, len(env)+len(secretEnv))` with `maps.Clone(env)` (or `make(map[string]string, len(env))`) followed by `maps.Copy(out, secretEnv)` so CodeQL does not flag potential integer addition overflow on map capacity allocation.
- **`devplus-agent-service` — CodeQL `py/incomplete-url-substring-sanitization` (Medium, 1 finding)**:
  - File: `tests/test_module_preflight.py:449` (`test_wrapper_sourcing_devplus_modules_from_terraform_cloud_is_flagged`).
  - Fix: Assert the complete diagnostic message substring rather than bare `"app.terraform.io" in messages` so CodeQL does not interpret the assertion as an incomplete URL host validation check.
- **`devplus-agent-service` — Bandit `B110` (`try_except_pass`) & `B112` (`try_except_continue`) (Medium, 6 findings)**:
  - `B110` (3 findings): `src/agent_service/domain/agents/architect_agent.py:1129`, `src/agent_service/domain/agents/tools/adoption_tools.py:395`, `src/agent_service/domain/agents/research_budget.py:214`.
  - `B112` (3 findings): `src/agent_service/domain/agents/architect_agent.py:2157`, `src/agent_service/domain/agents/tools/design_execution_tools.py:446`, `src/agent_service/domain/agents/tools/module_import_tools.py:184`.
  - Fix: Narrow the caught exception types and log at `logger.debug(..., exc_info=True)` instead of bare `pass` / `continue`.
- **`devplus-backend` — CodeQL `go/log-injection` (High, 11 findings)**:
  - Files: `handlers/secret_reference_handler.go:45`, `handlers/providers/github_repos.go:149`, `handlers/knowledge_handler.go:85`, `handlers/catalog_handler.go:74`, and remaining handler log calls.
  - Fix: Wrap user-controlled query/path fields with the existing `safelog` sanitizer before logging.

---

### 3. Checkov / MSDO Test-Fixture Exclusions & Dead Manifest Cleanup (~130+ Findings)
Microsoft Security DevOps (Checkov) scans unit-test fixture directories (including intentionally broken Terraform fixtures used to test preflight rules) and generated/vendored OpenAPI specs as production IaC:
- **Add `.checkov.yaml` `skip-path` exclusions for unit test fixtures (~95 High/Medium/Low findings)**:
  - `devplus-shared`: exclude `costest/costtest/testdata` (~55 AWS/Azure/GCP fixture findings).
  - `devplus-agent-service`: exclude `tests/fixtures/module_preflight` (14 findings on intentionally broken/minimal Terraform test fixtures).
  - `devplus-executor`: exclude `services/testdata` and `internal/repoanalyze/testdata` (26 findings on K8s Pod/Job JSON and Terraform test fixtures).
- **Remove legacy standalone manifest `devplus-platform/k8s/governance.yaml` (19 Medium/Low findings)**:
  - `k8s/governance.yaml` is a stale pre-Helm manifest (`image: devplus-governance-service:latest`, no probes/limits/securityContext) superseded by `k8s/helm/devplus/templates/governance.yaml`. Deleting it eliminates 19 `CKV_K8S_*` findings immediately.
- **Suppress `CKV_OPENAPI_5` (High) & `CKV_OPENAPI_21` (Medium) on generated/vendored `openapi.json` (12 findings)**:
  - Exclude generated/vendored OpenAPI specification files in `devplus-backend`, `devplus-sdk-go`, `devplus-frontend`, `devplus-cli`, `devplus-mcp`, and `terraform-provider-devplus`.

---

### 4. Azure Infrastructure Terraform Quick Fixes (`devplus-hub-infra`, `devplus-core-infra`, `devplus-base-infra`)
- **PostgreSQL Flexible Server Audit & Retention Config on `psql-binders-devplus-hub-cus` (`devplus-hub-infra/binders.tf`)**:
  - Add `azurerm_postgresql_flexible_server_configuration` resources to resolve 1 High and 3 Medium findings:
    - **[High]** `pgaudit.log = "ROLE,DDL,MISC"` (and append `,PGAUDIT` to `azure.extensions`) (`39bcf5d5-354e-4046-bad3-7a1dad887661`).
    - **[Medium]** `pgaudit.log_statement_once = "on"` (`bb7870a5-7aba-4fd5-8829-b79f0b614079`).
    - **[Medium]** `logfiles.retention_days = "7"` (must be `> 3`, `998d7eba-873f-4d8d-ba31-e6852ac3d025`).
    - **[Medium]** `connection_throttle.enable = "on"` (`3fd1757b-8eb3-4bbc-8ea1-99d1be6a9122`).
- **Link Application Insights to AI Foundry Hub (`devplus-core-infra/ai-foundry.tf`)**:
  - **[Medium]** Set `application_insights_id = azurerm_application_insights.main.id` on `azurerm_ai_foundry.main` (`aif-devplus-stage`, `aif-devplus-prod`, `44879142-bf14-47b1-aeb1-484c7d2b7ff3`).
- **State Storage Account Soft-Delete & Network Bypass (`devplus-base-infra/main.tf`)**:
  - **[Medium]** Configure `delete_retention_policy` and `container_delete_retention_policy` inside `blob_properties` (`CKV2_AZURE_38`) and set `bypass = ["AzureServices"]` in `network_rules` (`CKV_AZURE_36`) on `azurerm_storage_account.state`.
- **Move Reflector Source Secrets Out of `default` Namespace (`devplus-core-infra`)**:
  - **[Low]** Move `ghcr-pull-secret` (`reflector.tf`), `hubspot-access-token-shared` (`hubspot.tf`), and `github-install-state-shared` (`github-install-state.tf`) out of `default` into `devplus` (updating Reflector annotations accordingly) to clear 14 findings on `a6d6fb50-8526-45b8-a993-32263c8fe1be`.

---

### 5. AKS Pod & Container Security Context Hardening (`devplus-platform`, `devplus-executor`)
Resolves ~1,360 High/Medium runtime container assessments across `devplus-dev`, `devplus-stage`, and `devplus-prod` AKS clusters plus ~180 Checkov K8s/Docker static findings:
- **`automountServiceAccountToken: false` (High — 487 runtime instances)**:
  - Set `automountServiceAccountToken: false` on all Helm deployment templates in `devplus-platform/k8s/helm/devplus/templates/` that do not communicate with the K8s API (`frontend.yaml`, `governance.yaml`, `opentofu-server.yaml`, `devplus-mcp.yaml`, `backend.yaml`, `agent-service.yaml`) and all remaining Executor job builders (`buildJobSpec`, `buildDiscoveryJobSpec`, `health_dispatch.go`, `cost_sync_dispatch.go` in `devplus-executor/services/`), matching `buildRepoAnalysisJobSpec` (`job_dispatcher.go:1367`). (Azure Workload Identity uses its own webhook-injected projected volume and does not use the default K8s service account token.)
- **Hardened Container `securityContext` (Medium — 875 runtime instances)**:
  - Add pod/container `securityContext` (`runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities: { drop: ["ALL"] }`, `seccompProfile: { type: "RuntimeDefault" }`) across the `devplus-platform` Helm chart templates and `devplus-executor` job specs, matching the existing hardening on `otel-collector` (`devplus-core-infra/otel-collector.tf:279-295`) and `buildRepoAnalysisJobSpec`.

---

## Acceptance Criteria
1. Zero High/Medium/Low Dependabot / MSDO vulnerable dependency findings across all DevPlus GitHub repositories in Defender for Cloud.
2. Zero open CodeQL / Bandit alerts on `devplus-executor`, `devplus-agent-service`, and `devplus-backend` for the targeted rules.
3. Checkov no longer reports IaC findings against `testdata/` / `tests/fixtures/` directories or the removed `k8s/governance.yaml` manifest.
4. `psql-binders-devplus-hub-cus`, `aif-devplus-*`, and `devplusstate*` pass all targeted Defender for Cloud posture assessments after `tofu apply`.
5. All `devplus` namespace pods (except `executor-job-dispatcher` and `oidc-service`, which call the K8s API) run with `automountServiceAccountToken: false` and hardened container `securityContext`.
