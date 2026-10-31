---
id: 4243fa80-1e60-4ce7-b1f1-d53e66ff6935
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/GCP_SETUP.md

Source: DevPlusInc/devplus-platform/docs/GCP_SETUP.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/GCP_SETUP.md).

# Google Cloud Setup Guide for DevPlus

This guide explains how to connect a Google Cloud project to DevPlus, supporting two authentication methods: Workload Identity Federation (recommended, no stored secret) and a service account key.

## Overview

| Method | What DevPlus stores | How a deploy authenticates |
|---|---|---|
| **Workload Identity Federation (WIF)** | Nothing secret: project id/number, the service account to impersonate | DevPlus mints a short-lived OIDC token for the connector, Google STS exchanges it for a federated token, DevPlus impersonates the connector's service account and hands OpenTofu a one-hour access token |
| **Service account key** | The key JSON, **sealed** in the managed `connector_secrets` store (never on the connector document, never returned by the API) | DevPlus redeems a JWT-bearer assertion signed with the key and hands OpenTofu the key contents |

Either way OpenTofu receives `GOOGLE_*` environment variables and modules configure the google provider themselves: an empty `provider "google" {}` block is enough. The executor never writes a root google provider block (it writes root blocks only for `aws` and `azurerm`).

## Architecture

1. **OIDC Service**: mints JWTs for the connector (`sub` = `devplus:<slug>`, `aud` = the connector's WIF provider)
2. **GCP Connector**: project id + **number**, the identity pool/provider, the service account to impersonate (or a sealed key)
3. **Executor Service**: exchanges the JWT at `sts.googleapis.com`, impersonates the service account via `iamcredentials.googleapis.com`, injects `GOOGLE_*` into the OpenTofu process
4. **Terraform/OpenTofu**: the google provider reads `GOOGLE_PROJECT`, `GOOGLE_OAUTH_ACCESS_TOKEN` / `GOOGLE_CREDENTIALS`, `GOOGLE_REGION`, `GOOGLE_ZONE`

The same token path serves deployments, discovery scans (Cloud Asset Inventory) and Secret Manager reads: they all assume the connector's own identity.

## One Connection, One Service Account, One Provider

Every connector carries a **slug**, a short stable identifier DevPlus assigns when the connector is created. It appears in every Google object you create for a connection and in the token DevPlus mints for it:

| Thing | Value | Scope |
|---|---|---|
| Identity pool | `devplus` | **shared** per GCP project: create once ("create if missing"), never delete |
| OIDC provider | `devplus-<slug>` | per connector, with attribute condition `assertion.sub == 'devplus:<slug>'` |
| Service account | `devplus-<slug>@<project>.iam.gserviceaccount.com` | per connector; bound with `roles/iam.workloadIdentityUser` to this connector's principal only |
| Principal | `principal://iam.googleapis.com/projects/<number>/locations/global/workloadIdentityPools/devplus/subject/devplus:<slug>` | per connector |
| Subject (`sub`) | `devplus:<slug>` | per connector |
| JWT audience | `https://iam.googleapis.com/projects/<number>/locations/global/workloadIdentityPools/devplus/providers/devplus-<slug>` | the provider's own resource name, its default allowed audience |
| STS `audience` parameter | `//iam.googleapis.com/projects/<number>/.../providers/devplus-<slug>` | the same resource, schemeless (Google requires this form for the exchange) |

> **Why one provider per connector:** a workload identity provider pins exactly one issuer URI, and its attribute condition is what scopes the trust to one connector, the same way an AWS trust policy pins `sub`. Two DevPlus connections sharing a provider could use each other's identity; two DevPlus tenants (different issuers) sharing one would collide.

> **Why the service account is required:** federated tokens are not accepted by every Google API, and without an impersonated service account the request has no quota project. Impersonation makes the service account's project the quota project, which is what lets Cloud Asset Inventory scans work with no extra header.

> **The connector form generates the exact commands for you.** Open the GCP connector in DevPlus and copy the gcloud commands. The form is the source of truth for the values; this document explains the model behind them. GCP connectors have no legacy naming: the slug is required (4-22 characters, so `devplus-<slug>` fits Google's 30-character service account id) and the names are derived from it in `devplus-shared` (`Connector.GCPServiceAccountID()`, `GCPWorkloadIdentityProvider()`, `GCPPrincipal()`, `GCPDefaultOIDCAudience()`, `GCPSTSAudience()`), mirrored in the frontend's `lib/connectorNaming.ts`.

## Prerequisites

- A Google Cloud project with billing enabled; Owner (or IAM Admin + Service Account Admin + Workload Identity Pool Admin) on it
- The `gcloud` CLI, logged in
- The project **number** (every workload-identity resource name embeds it; it is not derivable from the id): `gcloud projects describe <project> --format='value(projectNumber)'`
- For WIF: the DevPlus OIDC issuer publicly reachable over HTTPS. Google fetches `<issuer>/.well-known/openid-configuration` and the JWKS, and matches the token's `iss` against the provider's issuer URI byte for byte. A local/atlas Tilt tenant (browser host `<tenant>.localhost:3000`) mints tokens for issuer `https://<tenant>.devplus.io`, which is not routed to the k3d cluster, so Google cannot fetch its keys; use an aks-mode tenant (or `--jwk-json-path` with a manually uploaded JWKS, which must be re-uploaded on key rotation: Tilt creates an empty `oidc-keys` secret, so the oidc-service generates a new key at every start).

## Method 1: Workload Identity Federation

```bash
# Values for THIS connection (shown in the DevPlus connector form)
CONNECTOR_SLUG="<slug>"          # e.g. 7f3a91c2
CONNECTOR_NAME="<name>"          # the connector's display name
PROJECT_ID="<project>"
ISSUER_URL="<issuer>"            # the DevPlus OIDC issuer, from the form

gcloud config set project "$PROJECT_ID"
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
echo "Project number: $PROJECT_NUMBER"   # enter this in the connector form

# 1. APIs DevPlus calls
gcloud services enable iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com \
  cloudresourcemanager.googleapis.com cloudasset.googleapis.com --project="$PROJECT_ID"

# 2. Shared identity pool (once per project)
gcloud iam workload-identity-pools describe devplus --location=global --project="$PROJECT_ID" >/dev/null 2>&1 \
  || gcloud iam workload-identity-pools create devplus --location=global --project="$PROJECT_ID" --display-name="DevPlus"

# 3. This connector's OIDC provider, pinned to its subject
gcloud iam workload-identity-pools providers create-oidc "devplus-$CONNECTOR_SLUG" \
  --location=global --workload-identity-pool=devplus --project="$PROJECT_ID" \
  --display-name="DevPlus $CONNECTOR_NAME" \
  --issuer-uri="$ISSUER_URL" \
  --attribute-mapping="google.subject=assertion.sub,attribute.connector_id=assertion.connector_id" \
  --attribute-condition="assertion.sub == 'devplus:$CONNECTOR_SLUG'"

# 4. This connector's service account
gcloud iam service-accounts create "devplus-$CONNECTOR_SLUG" --project="$PROJECT_ID" \
  --display-name="DevPlus $CONNECTOR_NAME ($CONNECTOR_SLUG)"
SERVICE_ACCOUNT="devplus-$CONNECTOR_SLUG@$PROJECT_ID.iam.gserviceaccount.com"

# 5. Only this connector's subject may impersonate it
gcloud iam service-accounts add-iam-policy-binding "$SERVICE_ACCOUNT" \
  --project="$PROJECT_ID" --role="roles/iam.workloadIdentityUser" \
  --member="principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/devplus/subject/devplus:$CONNECTOR_SLUG"

# 6. What the service account may do (least privilege in production)
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/editor"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/cloudasset.viewer"

# 7. Verify
gcloud iam workload-identity-pools providers describe "devplus-$CONNECTOR_SLUG" \
  --location=global --workload-identity-pool=devplus --project="$PROJECT_ID" --format='value(name)'
```

Then in DevPlus:

1. Navigate to **Admin > Connectors**, click Google Cloud Platform, then **New connector**
2. Select **Workload Identity Federation**
3. Enter the **Project ID** and **Project Number**; leave the service account and audience blank unless you used custom values (the form shows the derived defaults)
4. Save, wait about a minute for IAM to propagate, then **Test connection**. The test mints a token, federates, impersonates the account and reads the project through Cloud Resource Manager to confirm the number matches. A disabled Resource Manager API is reported as a success with a note (the exchange itself worked).

## Method 2: Service Account Key

Only where your organization policy allows key creation: `iam.disableServiceAccountKeyCreation` is enforced by default on organizations created on or after 2024-05-03.

```bash
PROJECT_ID="<project>"
gcloud iam service-accounts create "devplus-<slug>" --project="$PROJECT_ID" --display-name="DevPlus <name> (<slug>)"
SERVICE_ACCOUNT="devplus-<slug>@$PROJECT_ID.iam.gserviceaccount.com"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/editor"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/cloudasset.viewer"
gcloud iam service-accounts keys create key.json --iam-account="$SERVICE_ACCOUNT"
# Paste the contents of key.json into the connector form (Service Account Key), then delete the file.
```

The key must belong to the connector's project (a key from another project is rejected as a paste error). It is sealed in DevPlus's managed secret store under the connector; the connector document and every API response carry only `gcpHasServiceAccountKey: true`. The executor fetches the unsealed key through the internal API only for the key path. Switching a connector to WIF discards the stored key; sending a key while WIF is enabled is rejected (choose one method).

## Secret Manager as a secret source (optional)

```bash
gcloud services enable secretmanager.googleapis.com --project="$PROJECT_ID"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/secretmanager.viewer"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SERVICE_ACCOUNT" --role="roles/secretmanager.secretAccessor"
```

Then enable the connector as a secret source (project > Secrets) with an allowlist such as `prod-*` (secrets in the connector's project) or `projects/other-project/secrets/shared-*`. Secret references use backend `gcp_sm`, locator `<secretName>` or `projects/<project>/secrets/<secretName>`, version empty (latest) or a version number.

## Discovery

Discovery scans the project through **Cloud Asset Inventory** (`searchAllResources` with `readMask=*`, all supported asset types) as the connector's identity. Resource types are `gcp:<assetType>` (e.g. `gcp:compute.googleapis.com/Instance`); ids are Cloud Asset full resource names (`//compute.googleapis.com/projects/<p>/zones/<z>/instances/<i>`); zones are folded into their region (`us-central1-a` scans as `us-central1`, the zone stays in properties). Firewall rules open to the internet carry `openToInternet: true`; resources whose IAM policy grants `allUsers` or `allAuthenticatedUsers` carry `publiclyAccessible: true` (the IAM pass is best-effort: a denied `searchAllIamPolicies` is recorded, not fatal).

Requirements: `cloudasset.googleapis.com` enabled and `roles/cloudasset.viewer` on the service account.

## Authorize Project to Use Connector

1. Navigate to **Admin > Connectors** and click on the GCP connector
2. In the **Project access** card, click **Authorize project**
3. Select the project(s) that should use this connector
4. When creating deployments with google modules (`google` / `google-beta` providers), select the authorized connector

## Environment Variables

| Variable | Description | Auth Methods |
|----------|-------------|-------------|
| `GOOGLE_PROJECT` | The connector's project id | All |
| `GOOGLE_REGION` / `GOOGLE_ZONE` | Connector defaults, when set | All |
| `GOOGLE_OAUTH_ACCESS_TOKEN` | Impersonated service-account token (about one hour, not renewable) | WIF only |
| `GOOGLE_CREDENTIALS` | The key JSON contents | Key only |

Module doctrine: self-configure `provider "google" {}`; never shell out to `gcloud` (the executor image has none); for GKE credentials use `data.google_client_config.default.access_token` (note the provider persists that token in state; DevPlus encrypts state at rest). An apply longer than the token lifetime fails the same way an AWS STS session or an Azure `ARM_OIDC_TOKEN` does.

## Troubleshooting

### STS: audience mismatch / invalid_target

The connector's project number, pool or provider id does not match what was created. Compare the connector page's **Workload Identity Provider** and **OIDC Audience** with `gcloud iam workload-identity-pools providers describe`. Only set a custom audience on the connector if you configured `--allowed-audiences` on the provider.

### STS: invalid_grant, rejected by the attribute condition

The provider pins another subject. Its condition must be exactly `assertion.sub == 'devplus:<slug>'` for this connector's slug (shown as **OIDC Subject** on the connector page).

### 403 on generateAccessToken

The `roles/iam.workloadIdentityUser` binding is missing on the service account, or names a different principal. Bindings can take about a minute to propagate; retry the test.

### Issuer or JWKS unreachable

Google must fetch the issuer's discovery document and JWKS over public HTTPS, and `iss` must equal the provider's issuer URI byte for byte. `OIDC_ISSUER_URL` (the chart's `publicUrl`) must equal the public host, and the tenant's Cloudflare tunnel must route the anchored paths `^/.well-known/openid-configuration$` and `^/.well-known/jwks.json$` to `oidc-service:8081` (`terraform-cloudflare-ingress`, or `scripts/cf-ingress.sh` in Tilt aks mode). Unanchored rules also capture `/api/v1/.well-known/...`, which the SPA reads through the backend.

### Provider creation denied

The organization policy `constraints/iam.workloadIdentityPoolProviders` restricts issuers; add the DevPlus issuer to its allowlist.

### Discovery fails with SERVICE_DISABLED / PERMISSION_DENIED

Enable `cloudasset.googleapis.com` and grant `roles/cloudasset.viewer`. With an impersonated service account no quota project header is needed; if you point the connector at an existing account in another project, that account's project must have the API enabled too.

### Key creation blocked

`iam.disableServiceAccountKeyCreation` is enforced on the organization; use Workload Identity Federation.

### Adoption import of a GKE cluster fails

Cloud Asset Inventory names zonal clusters under `/zones/`; the google provider imports `google_container_cluster` under `/locations/`. The executor tries both forms. A service account discovered by its numeric unique id cannot be imported without looking up its email.

## Removing a connection

Delete the connector in DevPlus, then its provider and service account in Google Cloud:

```bash
gcloud iam workload-identity-pools providers delete "devplus-<slug>" --location=global --workload-identity-pool=devplus --project="$PROJECT_ID"
gcloud iam service-accounts delete "devplus-<slug>@$PROJECT_ID.iam.gserviceaccount.com" --project="$PROJECT_ID"
```

Keep the shared `devplus` pool: other connectors use it, and deleted pool or provider ids stay reserved for about 30 days.

## References

- [Workload Identity Federation with other identity providers](https://cloud.google.com/iam/docs/workload-identity-federation-with-other-providers)
- [Security Token Service: token exchange](https://cloud.google.com/iam/docs/reference/sts/rest/v1/TopLevel/token)
- [IAM Credentials: generateAccessToken](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/generateAccessToken)
- [Google provider authentication](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference#authentication)
- [Cloud Asset Inventory: searching resources](https://cloud.google.com/asset-inventory/docs/searching-resources)
