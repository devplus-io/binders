---
id: 9ec3476f-d491-4deb-9ba8-045eff63a13d
binder: binder-mvvle32a-1
---
Cross-Deployment Output Bindings

**Target Release:** [[V0.18]]
**Status:** Planned (P1)
**Scope:** `devplus-shared`, `devplus-backend`, `devplus-executor`, `devplus-agent-service`, `devplus-frontend`

---

## Overview & Problem Statement

In multi-resource architectures (e.g., VPC → EKS → RDS → App Service), downstream deployments routinely depend on outputs produced by upstream deployments in the same project. Today, DevPlus supports two parameter binding mechanisms on `CatalogItem` and `Deployment`:
1. **`parameterSecretBindings` / `secretInputs`** — resolves credentials at execution time from a cloud secret store (Key Vault, Secrets Manager, Secret Manager) and injects them via `TF_VAR_<param>`.
2. **`parameterDiscoveryBindings`** — queries live cloud APIs via `devplus-cloud-scanner` to populate environment-specific dropdowns (VPCs, subnets, clusters).

However, when chaining one deployment's outputs into another deployment's inputs, DevPlus currently falls back to **copying literal string values into `Deployment.Inputs` at creation time**:
- **Stale values on upstream re-apply:** If an upstream deployment is updated and its output changes, downstream deployments retain the stale literal string in `Deployment.Inputs`.
- **Sensitive outputs cannot be chained:** Because `Deployment.Outputs` masks sensitive outputs (`SensitiveOutputs` are replaced with `"(sensitive)"` in MongoDB by `maskRootOutputs`), agents and users cannot pass sensitive outputs (such as generated database endpoints, internal tokens, or connection strings) to downstream deployments (`FLOWS.md §1`).
- **Brittle heuristic promotion:** During environment promotion (`devplus-backend/services/deployment_promotion.go`), `sourceEnvironmentValues` heuristically scans all sibling deployments in the source project to guess whether a literal string in `source.Inputs` happened to match a sibling's output (`fromEnvironment`), and clears it to `needsValue` in the target environment instead of re-binding it to the promoted upstream twin.
- **No dependency visibility or delete guardrails:** Deleting or destroying an upstream deployment gives no warning when active downstream deployments in the same project depend on its outputs.

**First-Class Cross-Deployment Output Bindings (`outputBindings` / `parameterOutputBindings`)** makes cross-deployment references explicit, dynamically resolved at OpenTofu execution time, safe for sensitive outputs, and automatically re-pointed across environment promotions.

---

## Security & Architecture Invariants

1. **Strict Same-Project Isolation:** A deployment can only bind to a `sourceDeploymentId` that belongs to the **exact same `projectId`** (and `tenantId`). `devplus-backend` and `devplus-executor` must both verify `source.ProjectID == target.ProjectID` before resolving or saving a binding, preventing cross-project state exfiltration.
2. **DAG / Cycle Enforcement:** Self-references (`sourceDeploymentId == deployment.ID`) and circular dependency chains (`A → B → A`) across `outputBindings` within a project must be rejected with `400 Bad Request` on create, update, and preview.
3. **Zero-Persistence Sensitive Output Chaining:** When an upstream output is marked sensitive (in raw state, `source.SensitiveOutputs`, or `models.IsCredentialName(outputName)`), `devplus-executor` reads the cleartext value from `devplus-opentofu-server` (`GET /state/deployments/{sourceDeploymentId}`) in memory only, injects it into the downstream OpenTofu run via the existing `env_tfvar` mechanism (`TF_VAR_<param>` + `sensitive = true` root variable in `main.tf`), and registers it with the log scrubber. Sensitive bound outputs **never** land in `Deployment.Inputs`, MongoDB, API responses, MCP tool outputs, or LLM prompts.
4. **Input Hash & Approval Integrity:** `DeploymentInputsHash` (`devplus-shared/models/environments.go`) must include canonicalized `outputBindings` (`param=sourceDeploymentId:outputName@outputsVersion`) so environment approval gates (`EnvironmentApproval`) bind to the exact upstream output revision being consumed.

---

## Detailed Design by Repository

### 1. `devplus-shared` — Data Models & Hashing

- **File:** `models/deployment.go`
  - Add `DeploymentOutputBinding`:
    ```go
    type DeploymentOutputBinding struct {
        SourceDeploymentID bson.ObjectID `bson:"source_deployment_id" json:"sourceDeploymentId"`
        OutputName         string        `bson:"output_name" json:"outputName"`
    }
    ```
  - Add `OutputBindings map[string]DeploymentOutputBinding` (`bson:"output_bindings,omitempty" json:"outputBindings,omitempty"`) to:
    - `Deployment`
    - `DeploymentAppliedConfig`
    - `CreateDeploymentRequest`
    - `UpdateDeploymentRequest`
    - `PreviewDeploymentRequest`
- **File:** `models/catalog.go`
  - Add optional catalog-level hint `ParameterOutputBindings map[string]ParameterOutputBinding` (`bson:"parameter_output_bindings,omitempty" json:"parameterOutputBindings,omitempty"`):
    ```go
    type ParameterOutputBinding struct {
        SourceModuleID      string `bson:"source_module_id,omitempty" json:"sourceModuleId,omitempty"`
        SourceCatalogItemID string `bson:"source_catalog_item_id,omitempty" json:"sourceCatalogItemId,omitempty"`
        OutputName          string `bson:"output_name" json:"outputName"`
    }
    ```
  - Allows catalog authors to declare that a parameter (e.g., `vpc_id` or `rds_endpoint`) can be satisfied by the `OutputName` of an existing deployment of a given module/catalog item in the same project.
- **File:** `models/environments.go`
  - Update `DeploymentInputsHash` (or add `DeploymentConfigurationHash`) so approval request hashes incorporate deterministic `outputBindings` keys alongside `Inputs`.
- **File:** `models/promotion.go`
  - Add `PromotionOriginDeploymentBinding = "deployment_binding"` to `PromotionInputOrigin`.
  - Include `OutputBinding *DeploymentOutputBinding` and `SourceOutputBinding *DeploymentOutputBinding` on `PromotionInputDiff` so promotion previews show exact upstream twin re-bindings.

---

### 2. `devplus-backend` — Validation, Deploy Form, Promotion & Delete Guardrails

- **Binding Validation & Cycle Detection (`services/deployment_binding_validator.go`)**:
  - On `CreateDeployment`, `UpdateDeployment`, and `PreviewDeployment`:
    1. Verify every `binding.SourceDeploymentID` exists and has `ProjectID == deployment.ProjectID` and `TenantID == deployment.TenantID`.
    2. Verify `binding.OutputName` exists in `source.Outputs` (or in the upstream module's declared `outputs` if the upstream deployment is queued/running in the same wave).
    3. Run cycle detection across all deployments in `projectId` (`A → B → C`) and reject cycles with `400 Bad Request`.
    4. Strip any parameter present in `OutputBindings` from plain `Deployment.Inputs` (or store a display placeholder) so literal values and dynamic bindings never conflict.
- **Deploy Form Integration (`services/deploy_form.go`)**:
  - Enhance `BuildDeployForm` to return `compatibleOutputSources` for parameters (matching `CatalogItem.ParameterOutputBindings` or matching output names/types on active deployments in the same project).
- **Dependency-Aware Environment Promotion (`services/deployment_promotion.go`)**:
  - In `BuildPromotionPreview` and `ExecutePromotion`:
    - When `source.OutputBindings[param]` is present, look up the upstream deployment's promoted twin in the target project via `FindPromotedTwin(ctx, target.ID, binding.SourceDeploymentID)` (and reverse-lookup if promoting back).
    - **If the upstream twin exists in the target project:** Automatically set `target.OutputBindings[param] = {SourceDeploymentID: upstreamTwin.ID, OutputName: binding.OutputName}`, classify the parameter diff as `Origin: PromotionOriginDeploymentBinding`, `Action: PromotionActionAutoFill`, and `FilledFrom: fmt.Sprintf("deployment:%s.%s", upstreamTwin.Name, binding.OutputName)`.
    - **If the upstream twin has NOT been promoted yet:** Emit an explicit promotion warning (`"Parameter %q binds to deployment %q (%s), which has not been promoted to %s yet"`), set `NeedsValue: true`, and block `ExecutePromotion` unless overridden or the upstream deployment is promoted first.
- **Downstream Dependents API & Delete Guardrail (`handlers/deployment_handlers.go`)**:
  - Add `GET /projects/{projectId}/deployments/{id}/dependents` returning active deployments in the project whose `outputBindings` reference `{id}`.
  - On `DELETE /projects/{projectId}/deployments/{id}`, if active downstream dependents exist and `?force=true` is not set, return `409 Conflict` listing the dependent deployments.

---

### 3. `devplus-executor` — Runtime State Resolution & Sensitive Injection

- **File:** `services/deployment_output_resolver.go` & `services/executor_service.go`
  - During `executeDeployment` and `PreviewDeployment`, before generating `main.tf` / `terraform.tfvars.json`:
    1. Iterate over `deployment.OutputBindings`.
    2. Fetch the upstream deployment document from MongoDB and verify `upstream.ProjectID == deployment.ProjectID` and `upstream.Status == DeploymentStatusCompleted`.
    3. Fetch the upstream deployment's raw decrypted state from `devplus-opentofu-server` via `StateClient.GetState(ctx, upstream.ID.Hex())` and extract the output via `models.ModuleOutputValueFromState(rawState, binding.OutputName)` (supporting both string and JSON-marshaled structured outputs).
    4. Determine sensitivity: `isSensitive := upstream.IsSensitiveOutput(binding.OutputName) || models.IsCredentialName(binding.OutputName) || stateOutputIsSensitive(rawState, binding.OutputName)`.
    5. **If sensitive:** Inject via the `env_tfvar` path (`TF_VAR_<paramName>` in the OpenTofu subprocess environment + `variable "<paramName>" { sensitive = true }` on the generated root module in `opentofu_executor.go`) and register the secret value with the log scrubber.
    6. **If non-sensitive:** Pass into the resolved module inputs map for `terraform.tfvars.json` (never mutating `Deployment.Inputs` in MongoDB so the binding remains dynamic).
    7. Record the resolved upstream `OutputsVersion` on `DeploymentAppliedConfig.OutputBindings` when the apply succeeds.

---

### 4. `devplus-agent-service` — Architect Wave Chaining

- **Files:** `domain/architect/wave_executor.go`, `mcp/tools_deployments.go`
  - Update `provision_catalog_item` and the Architect wave executor so that when Wave $N+1$ consumes an output from a deployment created in Wave $N$, it populates `outputBindings: { "<param>": { "sourceDeploymentId": "<waveNDeploymentId>", "outputName": "<output>" } }` rather than copying literal strings into `inputs`.
  - Unlocks multi-wave architectures where Wave 1 produces sensitive outputs (such as database passwords/connection strings or cluster certificates) needed by Wave 2 without exposing those secrets to the LLM context.

---

### 5. `devplus-frontend` — Binding Picker, Dependency Badges & Promotion Diff

- **Deploy Form (`CreateDeploymentModal.tsx`, `DeployInputCard.tsx`)**:
  - Allow users to switch any parameter between **Literal Value**, **Secret Store** (`secretInputs`), and **Deployment Output** (`outputBindings`).
  - When **Deployment Output** is selected, render a two-step selector: upstream deployment in the project (highlighting recommended matches from `parameterOutputBindings`) → output key (marking `(sensitive)` outputs clearly with a lock badge).
- **Deployment Detail & Overview (`projects/[id]/deployments/[deploymentId]/page.tsx`)**:
  - Render bound parameters with a clickable dependency pill (`↗ <UpstreamDeploymentName>.<outputName>`) linking to the upstream deployment.
  - Show a **"Referenced by"** section when downstream deployments bind to this deployment's outputs, and warn in the Delete / Destroy modal if downstream dependents exist.
- **Promotion Modal (`PromoteDeploymentModal.tsx`)**:
  - Render `deployment_binding` items with an automatic twin re-binding badge (`dev-vpc.vpc_id → prod-vpc.vpc_id`) or a clear warning if the upstream dependency has not been promoted yet.

---

## Acceptance Criteria ("Done When")

- [ ] `DeploymentOutputBinding` and `ParameterOutputBindings` models added to `devplus-shared` with full BSON/JSON serialization and unit tests.
- [ ] `devplus-backend` validates same-project ownership, output existence, and DAG acyclicity on create/update/preview, and exposes `GET /projects/{projectId}/deployments/{id}/dependents` + `409 Conflict` delete protection.
- [ ] `devplus-executor` resolves `outputBindings` at runtime from `devplus-opentofu-server` state, injecting sensitive outputs via `TF_VAR_<param>` without persisting cleartext to MongoDB or logs.
- [ ] `devplus-backend` promotion service automatically re-points `outputBindings` to the upstream deployment's promoted twin in the target project via `FindPromotedTwin`, or warns when the upstream twin is missing.
- [ ] `devplus-agent-service` wave executor uses `outputBindings` to chain outputs across waves, including sensitive outputs.
- [ ] `devplus-frontend` supports selecting deployment output bindings in the deploy form, renders upstream/downstream links on the deployment detail page, and displays twin re-binding in the promotion preview modal.
- [ ] Whole-repo unit test coverage remains `>= 80%` across `devplus-shared`, `devplus-backend`, `devplus-executor`, `devplus-agent-service`, and `devplus-frontend`.
