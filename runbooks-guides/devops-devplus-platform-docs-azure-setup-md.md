---
id: 543d1212-fd3d-41ba-9172-cc8298616919
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/docs/AZURE_SETUP.md

Source: DevPlusInc/devplus-platform/docs/AZURE_SETUP.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/AZURE_SETUP.md).

# Azure Setup Guide for DevPlus

This guide explains how to set up Azure integration for DevPlus connectors, supporting three authentication methods: Service Principal (Client Credentials), OIDC (Workload Identity Federation), and Managed Identity.

## Overview

DevPlus supports Azure authentication via three methods:

- **Service Principal**: Traditional client ID + client secret. Simple to set up, requires secret rotation.
- **OIDC (Workload Identity Federation)**: Uses JWT tokens from DevPlus's OIDC service to obtain Azure tokens without storing secrets. Most secure option.
- **Managed Identity**: Uses the identity assigned to the VM/container running DevPlus. Zero credential management, but only works when DevPlus runs on Azure.

## Required Roles

Every method assigns the connector's identity the same two roles:

| Role | Why |
|---|---|
| **Owner** | DevPlus deploys whatever services a design calls for, and designs create role assignments: a cluster's `AcrPull` on its registry, a workload identity's Key Vault access, the `Key Vault Secrets User` self-grant for Key Vault secret sources, the `Cognitive Services OpenAI User` grant of the DevPlus Tenant provisioning item. Contributor cannot assign roles. |
| **Azure Kubernetes Service RBAC Cluster Admin** | Owner carries no Kubernetes data-plane rights. Deployments reach AKS with an Entra token (`KUBE_TOKEN`), which the cluster's Azure RBAC authorizes. |

This is the Azure equivalent of the `AdministratorAccess` policy the AWS setup attaches: a tool that deploys any service needs it. DevPlus compensates with its own controls: governance (Cedar) policies decide which actions are allowed, and the Architecture Principles decide what a design may contain. To confine a connection, assign both roles at a resource group instead of the subscription.

Connections set up earlier with Contributor plus the constrained "Role Based Access Control Administrator" (ABAC-limited to Key Vault Secrets User) keep working for what that allowed; designs that assign other roles, or deploy onto AKS, need the two roles above.

## Architecture

### Service Principal
1. **Azure Connector**: Configured with Tenant ID, Client ID, Client Secret, and Subscription ID
2. **Executor Service**: Passes credentials as `ARM_*` environment variables, plus an Entra token for AKS as `KUBE_TOKEN`
3. **Terraform/OpenTofu**: Uses credentials to provision Azure resources

### OIDC (Workload Identity Federation)
1. **OIDC Service**: Generates JWT tokens for projects/deployments (`sub` = the connector's subject, `aud` = the connector's audience)
2. **Azure Connector**: Configured with Tenant ID, Client ID, Subscription ID, and OIDC audience
3. **Executor Service**: Retrieves JWT token and passes it as `ARM_OIDC_TOKEN`; it also redeems it as a client assertion for the AKS token (`KUBE_TOKEN`)
4. **Terraform/OpenTofu**: The azurerm provider exchanges `ARM_OIDC_TOKEN` for an Azure access token via the federated credential

### Managed Identity
1. **Azure Connector**: Configured with Subscription ID only
2. **Executor Service**: Sets `ARM_USE_MSI=true` environment variable
3. **Terraform/OpenTofu**: Uses Managed Identity to authenticate

## One Connection, One App Registration

Every connector carries a **slug** — a short, stable identifier DevPlus assigns when the connector is created. It is what keeps two DevPlus connections in the *same* Azure tenant apart, so it appears in every Azure object you create for a connection and in the token DevPlus mints for it:

| Thing | Value |
|---|---|
| App registration display name | `DevPlus <connector> (<slug>)` (e.g. `DevPlus prod-azure (7f3a91c2)`) |
| Federated credential name | `devplus-<slug>` |
| Subject (`sub`) | `devplus:<slug>` |

> **Why this changed:** the old flow named every app `DevPlus Deployment OIDC` (or `DevPlus Deployment`) and looked it up with `az ad app list --display-name "…" --query "[0].appId"`. That query returns whichever matching app Azure happens to list **first**, so setting up a second DevPlus connection in the same tenant would silently attach its federated credential to the **first** connection's app registration — and hand you that app's client ID. Under Service Principal auth it was worse: `az ad app credential reset` would rotate the other connection's secret and break it. The constant subject `devplus` had the same flaw — two connectors had identical (issuer, subject) trust, so either could use the other's Azure identity. Per-connector names and a per-connector subject remove both.

> **The connector form generates the exact commands for you.** Open the Azure connector in DevPlus and copy the setup commands — they already carry this connection's slug, app display name, federated credential name and subject. The form is the source of truth for the values; this document explains the model behind them.

Never hardcode the names below: derive them from the connector (`Connector.AzureAppDisplayName()`, `Connector.AzureFederatedCredentialName()`, `Connector.OIDCSubject()` in `devplus-shared`), which also returns the legacy names for pre-slug connectors.

## Prerequisites

- Azure Subscription with appropriate permissions
- Azure CLI installed (`az` command)
- Kubernetes cluster running DevPlus
- For OIDC: OIDC service deployed and publicly accessible (Entra ID fetches `<issuer>/.well-known/openid-configuration` and the JWKS; a local/atlas Tilt tenant's issuer `https://<tenant>.devplus.io` is not routed to the k3d cluster, so use an aks-mode or Terraform-deployed tenant)

## Method 1: Service Principal Setup

### Step 1: Create an App Registration

```bash
# Login to Azure
az login

# Set your subscription
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

# The app registration name for THIS connection (shown in the DevPlus connector form)
APP_NAME="DevPlus <connector> (<slug>)"   # e.g. DevPlus prod-azure (7f3a91c2)

# Create the App Registration and read its appId straight from the create call.
# Never `az ad app list --display-name ... --query "[0].appId"`: it can return a
# DIFFERENT DevPlus connection's app, and the credential reset below would then
# rotate that connection's secret and break it.
APP_ID=$(az ad app create --display-name "$APP_NAME" --query appId -o tsv)
echo "Client ID: $APP_ID"
```

### Step 2: Create a Service Principal

```bash
# Create a Service Principal for the App Registration
az ad sp create --id $APP_ID

# Get the Service Principal Object ID
SP_OBJECT_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
echo "Service Principal Object ID: $SP_OBJECT_ID"
```

### Step 3: Create a Client Secret

```bash
# Create a client secret (valid for 1 year)
CLIENT_SECRET=$(az ad app credential reset \
  --id $APP_ID \
  --append \
  --years 1 \
  --query "password" -o tsv)

echo "Client Secret: $CLIENT_SECRET"

# Get your Tenant ID
TENANT_ID=$(az account show --query "tenantId" -o tsv)
echo "Tenant ID: $TENANT_ID"

# Get your Subscription ID
SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"
```

### Step 4: Assign Permissions

```bash
# Owner: create any resource and the role assignments it needs.
az role assignment create \
  --assignee $APP_ID \
  --role "Owner" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Kubernetes resources and Helm charts on AKS clusters with Azure RBAC.
az role assignment create \
  --assignee $APP_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# To confine DevPlus to one resource group, give both roles the scope
#   "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/<RESOURCE_GROUP>"
```

### Step 5: Configure DevPlus Connector

1. Navigate to **Admin > Connectors** in DevPlus
2. Click Microsoft Azure, then **New connector**
3. Select **Service Principal** authentication
4. Enter:
   - **Tenant ID**: Your Azure AD Tenant ID
   - **Client ID**: The Application (Client) ID
   - **Client Secret**: The secret created above
   - **Subscription ID**: Your Azure Subscription ID
5. Save the connector

## Method 2: OIDC (Workload Identity Federation) Setup

### Step 1: Create an App Registration

Do **not** reuse another connection's app registration, and do not look one up by a name that is not unique — each connector gets its own.

```bash
# Values for THIS connection (shown in the DevPlus connector form)
CONNECTOR_SLUG="<slug>"                   # e.g. 7f3a91c2
APP_NAME="DevPlus <connector> (<slug>)"   # e.g. DevPlus prod-azure (7f3a91c2)

# Create the App Registration and read its appId straight from the create call —
# never `az ad app list --display-name ... --query "[0].appId"`, which can return a
# DIFFERENT DevPlus connection's app and attach your credential to it.
APP_ID=$(az ad app create --display-name "$APP_NAME" --query appId -o tsv)
echo "Client ID: $APP_ID"

# Create a Service Principal
az ad sp create --id $APP_ID
```

### Step 2: Add Federated Credential

The credential is named after the connector and matches *only* that connector's subject, so a token minted for another DevPlus connection cannot satisfy it. Azure compares `subject` as a literal string — it must be `devplus:<slug>` exactly.

```bash
# Get your DevPlus OIDC issuer URL
# The connector form's setup commands fill it in (the `issuer` of
# <tenant-url>/.well-known/openid-configuration):
ISSUER_URL="https://your-oidc-issuer.example.com"

# Create federated credential for THIS DevPlus connection
az ad app federated-credential create --id $APP_ID --parameters "{
  \"name\": \"devplus-$CONNECTOR_SLUG\",
  \"issuer\": \"$ISSUER_URL\",
  \"subject\": \"devplus:$CONNECTOR_SLUG\",
  \"audiences\": [\"api://AzureADTokenExchange\"],
  \"description\": \"DevPlus OIDC Workload Identity Federation\"
}"

# Verify (name devplus-<slug>, subject devplus:<slug>)
az ad app federated-credential list --id $APP_ID
```

### Step 3: Assign Permissions

```bash
SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)

# Owner: create any resource and the role assignments it needs.
az role assignment create \
  --assignee $APP_ID \
  --role "Owner" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Kubernetes resources and Helm charts on AKS clusters with Azure RBAC.
az role assignment create \
  --assignee $APP_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### Step 4: Configure DevPlus Connector

1. Navigate to **Admin > Connectors** in DevPlus
2. Click Microsoft Azure, then **New connector**
3. Select **OIDC** authentication
4. Enter:
   - **Tenant ID**: Your Azure AD Tenant ID
   - **Client ID**: The Application (Client) ID
   - **Subscription ID**: Your Azure Subscription ID
   - **OIDC Audience**: `api://AzureADTokenExchange` (default)
5. Save the connector

## Method 3: Managed Identity Setup

> **Note**: Managed Identity only works when DevPlus runs on Azure (AKS, Azure VM, Azure Container Instances, etc.)

### Step 1: Enable Managed Identity

```bash
# For AKS - enable workload identity
az aks update \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME> \
  --enable-oidc-issuer \
  --enable-workload-identity

# For VMs - assign a system-managed identity
az vm identity assign \
  --resource-group <RESOURCE_GROUP> \
  --name <VM_NAME>
```

### Step 2: Assign Permissions

```bash
SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)

# Get the Managed Identity Principal ID
# For AKS workload identity:
IDENTITY_ID=$(az identity show \
  --resource-group <RESOURCE_GROUP> \
  --name <IDENTITY_NAME> \
  --query "principalId" -o tsv)

# Owner: create any resource and the role assignments it needs.
az role assignment create \
  --assignee $IDENTITY_ID \
  --role "Owner" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Kubernetes resources and Helm charts on AKS clusters with Azure RBAC.
az role assignment create \
  --assignee $IDENTITY_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### Step 3: Configure DevPlus Connector

1. Navigate to **Admin > Connectors** in DevPlus
2. Click Microsoft Azure, then **New connector**
3. Select **Managed Identity** authentication
4. Enter:
   - **Subscription ID**: Your Azure Subscription ID
5. Save the connector

## Authorize Project to Use Connector

After creating the connector:

1. Navigate to **Admin > Connectors** and click on the Azure connector
2. In the **Project access** card, click **Authorize project**
3. Select the project(s) that should use this connector
4. When creating deployments with Azure modules, select the authorized connector

## Environment Variables

When DevPlus runs a deployment with an Azure connector, it sets these environment variables for Terraform/OpenTofu:

| Variable | Description | Auth Methods |
|----------|-------------|-------------|
| `ARM_TENANT_ID` | Azure AD Tenant ID | SP, OIDC |
| `ARM_CLIENT_ID` | Application Client ID | SP, OIDC |
| `ARM_CLIENT_SECRET` | Client Secret | SP only |
| `ARM_SUBSCRIPTION_ID` | Azure Subscription ID | All |
| `ARM_USE_OIDC` | Enable OIDC auth | OIDC only |
| `ARM_OIDC_TOKEN` | OIDC JWT token | OIDC only |
| `ARM_USE_MSI` | Enable Managed Identity | MI only |
| `ARM_ENVIRONMENT` | Azure cloud environment | All (if non-default) |
| `KUBE_TOKEN` | Entra token for the AKS server app (`6dae42f8-4368-4678-94ff-3960e28e3630`), minted as the connector identity; the `kubernetes` and `helm` providers read it | All |

A module that deploys onto AKS sets only `host` and `cluster_ca_certificate` in its `kubernetes` / `helm` provider blocks (from `data.azurerm_kubernetes_cluster`). It must not use `kube_admin_config` (clusters with local accounts disabled have none) or an `exec` block (the job image has no `kubelogin`). The token lasts about an hour.

For a module that requires `hashicorp/azurerm`, the executor writes a root `provider "azurerm" { features {} }` block, so the module may leave azurerm unconfigured; a module that configures azurerm itself keeps its own block.

## Troubleshooting

### Service Principal Authentication Fails

- Verify Tenant ID, Client ID, and Client Secret are correct
- Confirm the Client ID belongs to *this* connection's app registration: `az ad app show --id $APP_ID --query displayName` should print `DevPlus <connector> (<slug>)`. If it names a different connector, the setup was run against the wrong app — and a `credential reset` on it will have broken that connector too.
- Check that the client secret hasn't expired
- Ensure the Service Principal has the required role assignments
- Verify the Subscription ID matches

### OIDC Token Exchange Fails

- Verify the OIDC issuer URL is publicly accessible over HTTPS
- Check that the federated credential's issuer URL matches exactly
- Check that its **subject** is `devplus:<slug>` for this connector's slug — a subject mismatch (e.g. the credential was added to another connection's app) is the usual cause of `AADSTS70021: No matching federated identity record found`
- Verify the audience is set to `api://AzureADTokenExchange`
- Ensure the OIDC service is responding at `/.well-known/openid-configuration`

### Managed Identity Not Working

- Verify DevPlus is running on Azure (AKS, VM, etc.)
- Check that Managed Identity is enabled on the compute resource
- Verify role assignments are in place for the identity
- Check that the Subscription ID is correct

### Kubernetes Provider Unauthorized or Forbidden

- `Unauthorized`: no `KUBE_TOKEN` reached the plan (the executor logs `Warning: no AKS token for connector ...`), or the module sets its own `token` / `client_certificate`, which beats the environment
- `Forbidden` (`User "<object id>" cannot ...`): the identity lacks **Azure Kubernetes Service RBAC Cluster Admin** on the cluster, or the cluster uses Kubernetes RBAC instead of Azure RBAC (then bind the identity's object id in the cluster)
- A cluster with a private API server must be reachable from the DevPlus job pods
- A `kubernetes` / `helm` provider with no `host` fails: the executor removes the job pod's in-cluster Kubernetes variables (`KUBERNETES_SERVICE_HOST` / `_PORT`) from the OpenTofu environment, so an unconfigured provider can never reach the DevPlus platform cluster

### Terraform Provider Errors

- Verify `ARM_*` environment variables are set in executor logs
- `The argument "features" is required`: the root block is written only when the module's required providers include `azurerm` from `hashicorp/azurerm` (or with no source); for any other source, configure azurerm in the module
- Check that the assigned role has sufficient permissions for the resources being provisioned

## Existing Connections

Connectors created before per-connection naming have no slug. They keep the old app display name `DevPlus Deployment OIDC`, the old federated credential name `devplus-oidc` and the old subject `devplus`, and they keep working exactly as they are. **Nothing needs to be re-created, and nothing on the Azure side needs to change.** DevPlus derives these names from the connector (`Connector.AzureAppDisplayName()`, `Connector.AzureFederatedCredentialName()`, `Connector.OIDCSubject()`), which falls back to the legacy values when the slug is empty — the slug is never back-filled.

To move an existing connection onto the per-connection model, migrate it — do not rename its Azure objects:

1. Create a **new** connector in DevPlus (it gets a slug)
2. Run the setup commands from its form (new app registration, new federated credential with subject `devplus:<slug>`, role assignments)
3. Re-point the projects at the new connector
4. Delete the old connector, then its old app registration

> **Never rename the app registration or federated credential of a live connector.** Its trust is established against the legacy subject `devplus` — changing the subject or the credential in place breaks every deployment, discovery scan and secret lookup that connection performs.

## Security Best Practices

1. **One Identity per Connection**: Give each connector its own app registration (`DevPlus <connector> (<slug>)`) — never share one across connections, and never look an app up by a name that is not unique
2. **Guardrails, Not a Narrower Role**: Like the AWS setup's AdministratorAccess, the connection holds Owner and AKS RBAC Cluster Admin (see [Required Roles](#required-roles)); governance policies and Architecture Principles decide what gets deployed
3. **Prefer OIDC or Managed Identity**: Avoid storing long-lived client secrets when possible
4. **Rotate Secrets**: If using Service Principal, rotate client secrets regularly
5. **Scope Permissions**: To confine a connection, assign both roles at a resource group instead of the subscription
6. **Audit Logging**: Enable Azure Activity Log to monitor provisioning actions
7. **Network Restrictions**: Use Azure AD Conditional Access to restrict where tokens can be used

## References

- [Azure Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)
- [Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation)
- [Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- [Terraform AzureRM Provider Authentication](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs#authenticating-to-azure)
