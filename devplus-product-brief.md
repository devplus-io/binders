---
id: a055b82f-f6b7-451c-a1e0-19d5bb232ebd
binder: binder-mvvle32a-1
---
DevPlus product brief

Verified 26 October 2026 against the read-only DevPlusInc repository mirror, current main branches, GitHub Actions and the published Helm chart list. This is the current overview; older detailed capability descriptions live in ~/Documents/DevPlus Architecture/product-brief.md and are explicitly marked as a historical 0.1.16 baseline.

## What DevPlus is

DevPlus is an AI-driven platform engineering platform for teams designing and operating cloud infrastructure. Architecture design and agent orchestration sit alongside deployment, discovery, observability, cost and governance. It is not solely a security or compliance product.

Curated OpenTofu/Terraform modules become catalog items. Deployments execute in isolated Kubernetes Job pods. The backend exposes tenant/project APIs, the agent drives workflows through those APIs, and MongoDB stores tenant data, queues and durable run state. Customer cloud access belongs to executor jobs, not the language model process.

## Published versus running

Chart 0.1.17 was published on 25 October 2026, confirmed in the DevPlus GHCR chart package. The release source is merged across backend #85, executor #71, shared #51, governance #35, agent-service #57, frontend #128 and platform #79. Platform release run: https://github.com/DevPlusInc/devplus-platform/actions/runs/40199665335.

Current tenant versions were NOT verified: read-only requests to dev, stage and prod Kubernetes API endpoints timed out. Do not carry the old 5 October tenant snapshot forward as current. A published chart or successful workflow alone does not prove every tenant runs it.

Documentation drift: devplus-platform/CHANGELOG.md still places the 0.1.17 notes under [Unreleased], despite the published chart. Treat the release contents below as source-backed and chart-published, not independently verified live.

## Services and tenancy

The runtime surfaces are frontend, backend, agent-service, executor, governance-service, oidc-service, opentofu-server and devplus-mcp. A tenant maps to a namespace, Mongo database and WorkOS organization; project roles add authorization within it. MongoDB remains the product datastore and durable queue. There is no separate message broker or external secret vault in the product path. Connector secrets use envelope encryption in MongoDB.

The hub now also provisions a Binders team PostgreSQL database. That is separate staff/Binders infrastructure, NOT a second DevPlus tenant datastore. It uses a private endpoint in the hub and a Central US server; Terraform allows TCP 5432 from the Covee gateway address only. Schema, member logins and row-level security belong to Binders and were not verified. Sources: devplus-hub-infra/binders.tf:1-18,30-77,100-137; variables.tf:152-162.

## Capabilities and where to verify them

| Capability | Current source and publication status | Code entry point |
|---|---|---|
| Architecture design and agent orchestration | The supervisor and domain agents use backend tools; architecture runs have durable state and approval cards. 0.1.17 adds principles and activity workflows. Intended safety controls are not a claim that all authorization defects are fixed. | devplus-agent-service/src/agent_service/domain/agents/architect_agent.py; devplus-platform/CHANGELOG.md [Unreleased] |
| Deployment | Isolated OpenTofu plan/apply jobs, private registry/state, governance plan gate and durable run progress. Merged into published 0.1.17. | devplus-executor/services/{executor_service,opentofu_executor,job_dispatcher}.go |
| Approval and promotion | Plan-to-apply approval and cross-environment configuration promotion are in 0.1.17 source. Promotion copies configuration, not state or existing resources. Approval can depend on policy; do not promise mandatory fresh-plan approval for every path. | devplus-executor/services/deploy_approval.go; devplus-backend/services/deployment_promotion.go |
| Environments and previews | Projects carry environment tier and System grouping. Preview expiry and tombstones are modeled, not a new environment datastore. | devplus-shared/models/{environments,previews}.go |
| Modules and registry | Source inspection, private catalog/registry workflows and managed publishing are in the 0.1.17 source. | devplus-backend/services/module_registry_service.go |
| Discovery and adoption | Deterministic cloud inventory; separate isolated jobs analyze repositories and import existing infrastructure. Extracted repository facts are deterministic, not LLM-generated. | devplus-executor/internal/{scan,awsscan,azurescan,gcpscan,repoanalyze,githubscan}; services/adoption_*.go |
| Cost estimates | 0.1.17 replaces the old Infracost path with shared costest and an offline price snapshot. Backend quick estimates are advisory; plan-time governance is authoritative. No latency benchmark was run. | devplus-executor/services/cost_service.go:22-36; devplus-backend/services/quick_cost.go |
| Billed spend | Distinct from estimates: account/day syncs ingest AWS Cost Explorer, Azure Cost Management and GCP BigQuery billing exports into cost_sync_queue, cost_sync_runs and cost_lines. | devplus-shared/models/cost_spend.go; devplus-backend/services/cost_sync_scheduler.go; devplus-executor/internal/app/app.go |
| Scheduling, health and drift | Leased schedulers, stored next-run times and backoff; isolated health scans and Mongo health collections. Drift inspection is read-only, not automatic repair. | devplus-backend/services/schedulers.go; devplus-shared/models/{schedules,health}.go; devplus-frontend/components/deployments/detail/DriftPanel.tsx |
| Governance and secrets | Cedar policy evaluation, validation/audit and plan gates; connector secrets are encrypted and job pods obtain scoped cloud access. Important unresolved authorization findings are listed below. | devplus-docs/SECURITY_MODEL.md; devplus-backend/services/secret_store.go; devplus-governance-service; devplus-oidc-service |
| Support and programmatic access | HubSpot-backed support; OpenAPI-derived SDK/CLI, MCP and Terraform provider. Support cursor validation fix merged in backend #86. | devplus-backend/services/support_service.go; devplus-sdk-go; devplus-cli; devplus-mcp; terraform-provider-devplus |

## Current caveats for demos, support and public claims

Project writer-to-admin escalation remains tracked at https://github.com/DevPlusInc/devplus-backend/issues/94. Service-token identity override remains tracked at https://github.com/DevPlusInc/devplus-oidc-service/issues/42. These are source-backed defects, not evidence of a production compromise. Do not present project isolation or service identity as fully audited and defect-free.

All seven critical Dependabot alerts remain in terraform-provider-devplus, tracked at https://github.com/DevPlusInc/terraform-provider-devplus/issues/3. Alert severity is not proof the vulnerable functions are reachable. Agent SDK/PyMongo locks have advanced to langgraph-sdk 0.4.5 and pymongo 4.18.2; the previous SDK update PR #58 was closed without merge. Issue #59 still needs reconciliation, not repetition as an active SDK/PyMongo blocker.

No automatic drift repair, previewed output-binding approval fix or other unapproved proposal should be described as implemented. No current AWS account inventory was inspected.

## Companion assets

Repository overview: Binders note DevPlus repository map and ~/Documents/DevPlus Architecture/map.json. Diagrams: ~/Documents/DevPlus Architecture/diagrams/; captions distinguish source architecture from runtime evidence. Weekly raw scan: health/2026-10-26.json. Review scope is source architecture and read-only service metadata, not a fresh penetration test or deployed-system audit.
