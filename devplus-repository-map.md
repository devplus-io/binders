---
id: 43e8626c-53a6-42d9-a3b9-bcfd755536dd
binder: binder-mvvle32a-1
---
DevPlus repository map

Curated 26 October 2026 after sync.sh and map.py. All 32 active DevPlusInc repositories fetched successfully; the mirror was already current. Source mirror: ~/Library/Application Support/Pennant/knowledge/devplus-repos/. Generated inventory: ~/Documents/DevPlus Architecture/map.md and map.json. Repository roles below are curated; the script's extension-based classification incorrectly calls the Python agent an infrastructure repo.

## Product runtime

| Repository | Role and entry points |
|---|---|
| devplus-backend | Tenant/project API; internal/server/routes.go; handlers/ and services/. Calls OIDC, governance, agent and OpenTofu services. |
| devplus-frontend | Next.js UI; app/ and components/. Generated API types in lib/api/schema.d.ts are not a hand-maintained oversized module. |
| devplus-agent-service | Python orchestration; src/agent_service/domain/agents/ and domain/backend_client.py. Uses backend tools rather than directly provisioning clouds. |
| devplus-executor | Dispatcher and isolated cloud/OpenTofu jobs; services/job_dispatcher.go, services/executor_service.go, internal/app/app.go and internal/repoanalyze/. |
| devplus-governance-service | Cedar policy evaluation and simulation. |
| devplus-oidc-service | Project/deployment/service tokens; internal/token/service.go. |
| devplus-opentofu-server | State locking/storage and module registry endpoints. |
| devplus-mcp | Programmatic model-context surface over DevPlus APIs. |

## Shared code, clients and delivery

| Repository | Role |
|---|---|
| devplus-shared | Shared Go models, helpers, cost estimator and API contracts. |
| devplus-sdk-go | Go API client. |
| devplus-cli | Command-line client. |
| terraform-provider-devplus | Terraform/OpenTofu provider; tracked vendor tree and -mod=vendor CI/release builds. |
| devplus-platform | Helm chart, deployment/release workflows, local development and platform tests. |
| terraform-helm-devplus-services | Terraform wrapper for service Helm releases. |
| terraform-devplus-tenant | Tenant namespace/database/identity provisioning. |
| devplus-docs | Product architecture, API, security, operations and service documentation. |
| roadmap | Planning material, not implementation evidence. |
| devplus-public-website | Public website and frontend build tooling. |

## Infrastructure and utilities

| Repository | Role |
|---|---|
| devplus-base-infra | Foundational state infrastructure. |
| devplus-hub-infra | Shared network, private endpoints, Covee and separate Binders PostgreSQL infrastructure. |
| devplus-core-infra | Environment clusters, edge/Cloudflare, model catalogue and other platform cloud resources. |
| devplus-cicd-infra | CI/CD identity and infrastructure. |
| devplus-github-settings | GitHub organization/repository configuration. |
| terraform-cloudflare-ingress | Reusable Cloudflare ingress module. |
| terraform-aws-checkout-api | Checkout API infrastructure source. |
| terraform-aws-checkout-ci | Checkout CI infrastructure source. |
| terraform-aws-checkout-network | Checkout network infrastructure source. |
| terraform-aws-lambda-function | Reusable Lambda infrastructure source. |
| terraform-aws-rds-instance | Reusable RDS infrastructure source. |
| terraform-aws-s3-bucket | Reusable S3 infrastructure source. |
| bootstrapme | Bootstrap utility. |
| compliance-scripts | Compliance utilities/scripts. |

Reading AWS module source does not establish that DevPlus owns or uses an AWS account; no AWS account was inspected.

## Current architecture and release caveats

0.1.17 is chart-published, and the source release PRs are merged. Running tenant versions could not be checked because the Kubernetes API requests timed out. DevPlus tenant data remains in MongoDB; hub Binders PostgreSQL is a separate system. Consult the DevPlus product brief for publication versus runtime details and the unresolved authorization findings.

Weekly scan: 30 raw open Dependabot alerts, including 7 critical and 11 high. Two high-alert manifests disagree with current main: governance compress and core-infra callback grpc have already advanced beyond the reported vulnerable pins. Counts are raw GitHub metadata, not proof of exploitability or deployment.

Diagram inventory: platform-overview, deployment-lifecycle, tenancy-and-identity, ai-agent-architecture, discovery-and-adoption, repository-analysis and hub-staff-network. Captions in diagrams/README.md state the evidence date and baseline for each.
