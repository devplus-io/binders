---
id: c7c04e89-6fa6-4926-9c56-f34295c7c9dc
binder: binder-mvvle32a-1
folder: folder-mvw0u6iq
---
DevOps | devplus-platform/docs/OIDC_IMPLEMENTATION.md

Source: DevPlusInc/devplus-platform/docs/OIDC_IMPLEMENTATION.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/OIDC_IMPLEMENTATION.md).

# OIDC Implementation Summary

This document summarizes the OIDC implementation in DevPlus: the issuer (`devplus-oidc-service`) and how cloud connectors (AWS, Azure, GCP) use its tokens. The deep reference for the service is `docs/services/oidc-service.md` in `DevPlusInc/devplus-docs`; the per-cloud setup guides are [AWS_OIDC_SETUP.md](AWS_OIDC_SETUP.md), [AZURE_SETUP.md](AZURE_SETUP.md) and [GCP_SETUP.md](GCP_SETUP.md).

## What Was Implemented

### 1. OIDC Service (`devplus-oidc-service` repo)
- **Discovery Endpoints**: `.well-known/openid-configuration` and `.well-known/jwks.json` (the discovery document also advertises authorize/token/userinfo endpoints, which are not implemented)
- **Token Generation** (all RS256):
  - `/token/project/{projectId}` - Generate JWT for projects (no caller today)
  - `/token/deployment/{deploymentId}` - Generate JWT for deployments. Body: `projectId` (required), `audience`, `claims` (a string map merged over the defaults, which is how callers set `sub` and `connector_id`). Used for every cloud federation: deploys, discovery scans, secret reads and connection tests (the last three pass the placeholder id `000000000000000000000000`).
  - `/token/service` - Exchange a Kubernetes service-account token (validated with the TokenReview API) for a short-lived service JWT; the account must be listed in `OIDC_ALLOWED_SERVICE_ACCOUNTS`. The executor dispatcher and job pods use it to call the backend.
  - `/token/user` - Mint a user token; requires `Authorization: Bearer <OIDC_BACKEND_SECRET>`. The chart does not set that secret, so it returns 503.
- **Token Validation**: `/token/validate` - Validate JWT tokens (RS256 against the current key, then an HMAC fallback with `JWT_SECRET`); the backend's auth middleware uses it for service tokens
- **RSA Key Support**: Loads the key pair from `OIDC_PRIVATE_KEY` / `OIDC_PUBLIC_KEY` (the optional `oidc-keys` secret), otherwise generates a 2048-bit key at startup. The `kid` in the JWKS and in token headers is the key's RFC 7638 thumbprint, so a new key always gets a new `kid` and federated clouds refetch the JWKS.
- **Health**: `GET /health`
- **Kubernetes Deployment**: `k8s/helm/devplus/templates/oidc-service.yaml` (also the `oidc-service` ServiceAccount and the cluster role that lets it create `tokenreviews`)

### 2. Connector Model Updates
- OIDC fields on the `Connector` model (`devplus-shared` `models/connector.go`):
  - `OIDCProviderURL` - AWS OIDC provider URL (stored and shown; the credential flow does not read it)
  - `IAMRoleARN` - IAM role ARN to assume (inferred from the account ID and `IAMRoleName()` when not supplied)
  - `OIDCAudience` - OIDC audience (defaults applied at mint time: `sts.amazonaws.com` for AWS, `api://AzureADTokenExchange` for Azure, the WIF provider resource for GCP)
  - `UseOIDC` - Boolean flag to enable OIDC authentication (AWS, Azure, and GCP Workload Identity Federation)
  - `Slug` - Per-connector id set at creation; `OIDCSubject()` returns `devplus:<slug>` (legacy connectors without a slug: `devplus`), the `sub` each cloud trust pins
- Provider update requests carry the auth flags as tri-state (`*bool`): a `PUT` that omits `useOidc` / `useManagedIdentity` keeps the stored value

### 3. Project-Connector Authorization
- **Model**: `ProjectConnector` - Links projects to authorized connectors
- **Service**: `ProjectConnectorService` - Manages authorizations
- **Handler**: `ProjectConnectorHandler` - REST API endpoints
- **Endpoints**:
  - `POST /api/v1/project-connectors` - Authorize project to use connector
  - `GET /api/v1/project-connectors/projects/{projectId}` - Get connectors for project
  - `GET /api/v1/project-connectors/connectors/{connectorId}` - Get projects for connector
  - `DELETE /api/v1/project-connectors/projects/{projectId}/connectors/{connectorId}` - Remove authorization
- The backend refuses to create or update a deployment whose project is not authorized for its connector (403)

### 4. AWS Credentials Service
- **Service**: `AWSCredentialsService` in `devplus-executor/services/aws_credentials_service.go`
- **Functionality**:
  - Generates deployment JWT tokens via OIDC service (`sub` = `devplus:<slug>` for slugged connectors, plus `connector_id`)
  - Calls AWS STS `AssumeRoleWithWebIdentity` with AWS SDK v2 (session name `devplus-<deploymentId>`)
  - Returns temporary AWS credentials
  - Falls back to access keys if OIDC is not enabled
- Siblings: `azure_credentials_service.go` (passes the token as `ARM_OIDC_TOKEN`, mints `KUBE_TOKEN`), `gcp_credentials_service.go` (Google STS exchange + service-account impersonation through `devplus-shared/gcpauth`), and `oidc_mint.go` (connector-scoped mints for discovery and secret reads)

### 5. Executor Service Updates
- **Credentials Integration**: Executor service now:
  - Uses the deployment's own connector (`deployment.connectorId`)
  - Resolves its credentials through one per-cloud resolver (`services/connector_credentials.go`): AWS (OIDC or access keys), Azure (`ARM_*` + `KUBE_TOKEN`), GCP (`GOOGLE_*`)
  - Passes credentials to the OpenTofu executor
- **OpenTofu Executor Updates**:
  - `buildTofuEnv` sets the AWS environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) and the Azure/GCP variables from the connector, and drops the pod's in-cluster Kubernetes variables
  - `createMainTF` writes root provider blocks only for `aws` (region) and `azurerm` (`features {}`); google modules self-configure
  - Applies to every OpenTofu run: `ExecuteModuleWithLogs`, `PlanModuleWithLogs`, `DestroyModuleWithLogs`, `PlanForDriftCheck`, `PlanForAdoption`

### 6. Documentation
- **AWS OIDC Setup Guide**: `docs/AWS_OIDC_SETUP.md` - Complete guide for setting up AWS OIDC provider and IAM roles
- **Azure Setup Guide**: `docs/AZURE_SETUP.md` - Service principal, OIDC (workload identity federation) and managed identity
- **GCP Setup Guide**: `docs/GCP_SETUP.md` - Workload Identity Federation and service-account keys

## What Still Needs to Be Done

The original list has shipped: the AWS SDK v2 STS call, per-deployment connector selection with backend-enforced project authorization, `OIDC_ISSUER_URL` from the Helm `publicUrl`, the connector UI (OIDC fields, generated setup commands, project access, status), and connection tests (`POST /api/v1/connectors/providers/{aws|azure|gcp}/{id}/test`, which persist `connected` / `error`).

Still open:

### 1. Persistent Signing Keys
Nothing provisions the `oidc-keys` secret with real keys: Tilt creates it empty, and `terraform-devplus-tenant` does not pass `oidc_private_key` / `oidc_public_key` to `terraform-helm-devplus-services`. So every oidc-service restart generates a new key, and tokens minted before the restart stop validating (the JWKS serves only the current key). The key-derived `kid` keeps federated clouds working after a restart, but in-flight tokens are lost.

## Architecture Flow

```
1. User creates deployment (backend checks the project is authorized for the connector)
   ↓
2. Executor service processes deployment
   ↓
3. Executor loads the deployment's connector (OIDC enabled)
   ↓
4. Executor calls OIDC service POST /token/deployment/{id} (aud = connector audience, sub = devplus:<slug>)
   ↓
5. AWS: executor calls STS AssumeRoleWithWebIdentity with the JWT
   (Azure: azurerm exchanges ARM_OIDC_TOKEN; GCP: executor exchanges at Google STS and impersonates the service account)
   ↓
6. The cloud returns temporary credentials
   ↓
7. Executor sets credentials as environment variables
   ↓
8. OpenTofu uses credentials to deploy infrastructure
```

The cloud validates the JWT by fetching `<OIDC_ISSUER_URL>/.well-known/openid-configuration` and the JWKS, so the issuer must be publicly reachable over HTTPS at the tenant's public host (the Cloudflare tunnel routes those two paths to `oidc-service:8081`).

## Environment Variables

### OIDC Service
- `PORT` - Listen port (default `8081`)
- `OIDC_ISSUER_URL` - Public URL of the issuer: the token `iss` and the discovery `issuer`. The chart sets it from `publicUrl` (default `https://<namespace>.<baseDomain>`); unset, the service falls back to `http://oidc-service:8081`
- `OIDC_PRIVATE_KEY` - RSA private key PEM (optional, from `oidc-keys`; auto-generated unless both keys are set)
- `OIDC_PUBLIC_KEY` - RSA public key PEM (optional, from `oidc-keys`)
- `JWT_SECRET` - HMAC fallback key for `/token/validate` only (from `jwt-secret`)
- `OIDC_ALLOWED_SERVICE_ACCOUNTS` - Comma-separated `<namespace>:<serviceaccount>` list allowed at `/token/service` (chart: `<ns>:executor-job-dispatcher`, plus `<ns>:<ns>-app` when `serviceAccount.create`)
- `OIDC_SERVICE_TOKEN_AUDIENCE` - Default service-token audience (default `backend`)
- `OIDC_SERVICE_TOKEN_TTL_MINUTES` - Service-token lifetime (default `10`)
- `OIDC_USER_TOKEN_TTL_HOURS` - User-token lifetime (default `24`)
- `OIDC_BACKEND_SECRET` - Shared secret for `/token/user` (unset: 503)

### Executor Service
- `OIDC_SERVICE_URL` - URL of OIDC service (default: `http://oidc-service:8081`)
- `OIDC_SERVICE_AUDIENCE` - Audience the executor requests for its service tokens (default `backend`)

### Backend
- `OIDC_SERVICE_URL` - URL of OIDC service (connection tests, AWS credential resolution for discovery and secrets, `/token/validate`)

## Security Considerations

1. **JWT Secret**: Only the oidc-service reads it (the HMAC fallback in `/token/validate`); use a strong, randomly generated value
2. **RSA Keys**: Generate 2048-bit or higher RSA keys for production, and persist them in `oidc-keys` (see "Persistent Signing Keys")
3. **Token Expiration**: Project and deployment tokens expire after 1 hour (fixed); service tokens after `OIDC_SERVICE_TOKEN_TTL_MINUTES`, user tokens after `OIDC_USER_TOKEN_TTL_HOURS`
4. **IAM Roles**: Use least-privilege IAM roles with minimal required permissions
5. **Audience Validation**: Ensure OIDC audience matches in both AWS and DevPlus configuration
6. **Subject Pinning**: Every cloud trust must pin `sub` = `devplus:<slug>` (AWS trust policy condition, Azure federated credential subject, GCP provider attribute condition); AWS cannot condition on custom claims
7. **Unauthenticated Mint Endpoints**: `/token/project` and `/token/deployment` accept any caller that can reach the service. Only the two `/.well-known/` paths are routed publicly, so the cluster network is the boundary

## Next Steps

1. Provision persistent signing keys in `oidc-keys` for deployed tenants (Terraform) and decide whether Tilt should keep per-start keys
