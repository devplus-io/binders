---
id: 0731873a-1607-42d9-87ec-90e993e5eb22
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/AWS_OIDC_SETUP.md

Source: DevPlusInc/devplus-platform/docs/AWS_OIDC_SETUP.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/AWS_OIDC_SETUP.md).

# AWS OIDC Setup Guide for DevPlus

This guide explains how to set up AWS OIDC (OpenID Connect) integration for DevPlus connectors, enabling secure AWS credential access using Assume Role with Web Identity.

## Overview

DevPlus supports AWS authentication via OIDC, which allows deployments to assume IAM roles using JWT tokens instead of storing long-lived access keys. This provides:

- **Enhanced Security**: No need to store AWS access keys in the database
- **Temporary Credentials**: Short-lived credentials that automatically expire
- **Fine-grained Access Control**: Use IAM roles with specific permissions per project/deployment
- **One Role per Connection**: Each connector gets its own IAM role and its own OIDC identity, so several DevPlus connections can live in the same AWS account without colliding

## Architecture

1. **OIDC Service**: Generates JWT tokens for projects/deployments, scoped to the connector they were minted for
2. **AWS Connector**: Configured with the AWS account ID and the OIDC audience; the IAM role ARN is inferred from the account ID and the connector's role name
3. **Executor Service**: Retrieves JWT token, exchanges it for AWS credentials via STS AssumeRoleWithWebIdentity
4. **Terraform**: Uses AWS credentials as environment variables

Discovery scans and AWS Secrets Manager reads assume the same role: the job pod asks the backend's internal `GET /api/v1/internal/connectors/{id}/aws-credentials` endpoint, which performs the same exchange.

## One Connection, One Role

Every connector carries a **slug** — a short, stable identifier DevPlus assigns when the connector is created. The slug is what stops two DevPlus connections in the *same* AWS account from colliding, so it appears in the name of the resources you create for a connection and in the token DevPlus mints for it:

| Thing | Value | Scope |
|---|---|---|
| IAM role name | `DevPlus-<slug>` (e.g. `DevPlus-7f3a91c2`) | **per connector**. Derived from the slug alone, so renaming the connector never moves it; the connector's display name goes on the role description |
| OIDC subject (`sub`) | `devplus:<slug>` | per connector: the trust policy pins it |
| OIDC identity provider | keyed by issuer URL | **shared** — create once per AWS account, reuse for every connector |

> **The connector form generates the exact commands for you.** Open the AWS connector in DevPlus, enable OIDC, and copy the setup commands — they already carry this connection's slug, role name and trust policy. The form is the source of truth for the values; this document explains the model behind them.

Never hardcode the names below: derive them from the connector (`Connector.IAMRoleName()`, `Connector.OIDCSubject()` in `devplus-shared`, mirrored in the frontend's `lib/connectorNaming.ts`), which also returns the legacy names for pre-slug connectors.

## Prerequisites

- AWS Account with appropriate permissions
- Kubernetes cluster running DevPlus
- OIDC service publicly reachable over HTTPS at the issuer URL (`OIDC_ISSUER_URL`, which the Helm chart sets from `publicUrl`), with `/.well-known/openid-configuration` and `/.well-known/jwks.json` routed to `oidc-service:8081`. A local/atlas Tilt tenant mints tokens for `https://<tenant>.devplus.io`, which no tunnel routes to the k3d cluster, so AWS cannot trust it: use an aks-mode or Terraform-deployed tenant.

## Step 1: Create OIDC Identity Provider in AWS

> **Once per AWS account.** The identity provider is keyed by issuer URL, not by connector. If another DevPlus connection already added it, skip to Step 2 and reuse its ARN — creating it twice fails with `EntityAlreadyExists`.

1. Go to AWS IAM Console → Identity providers
2. Click "Add provider"
3. Select "OpenID Connect"
4. Configure:
   - **Provider URL**: your DevPlus OIDC issuer URL (publicly reachable over HTTPS; shown in the connector form)
   - **Audience**: `sts.amazonaws.com` (or your custom audience)
5. Click "Add provider"
6. Note the **Provider ARN** (e.g., `arn:aws:iam::123456789012:oidc-provider/oidc.example.com`)

## Step 2: Create the IAM Role for This Connection

> **Important**: one role per connector. Name it `DevPlus-<slug>`: the connector form shows the exact name, and the connector's IAM Role ARN is inferred as `arn:aws:iam::[AccountID]:role/DevPlus-<slug>`.

1. Go to AWS IAM Console → Roles
2. Click "Create role"
3. Select "Web identity"
4. Choose the OIDC provider created in Step 1
5. Set audience to `sts.amazonaws.com` (or your configured audience)
6. **Add the subject condition** — `<issuer-host>:sub` must equal `devplus:<slug>`. This is what scopes the role to one connection; without it, any other DevPlus connector that knows the ARN can assume it.
7. Attach policies with permissions needed for your deployments
8. **Name the role**: `DevPlus-<slug>`, exactly as shown in the connector form
9. Note the **Role ARN** (e.g., `arn:aws:iam::123456789012:role/DevPlus-7f3a91c2`)

Do not add conditions on other token claims (`project_id`, `connector_id`, `deployment_id`): AWS cannot see them (see the note below), so such a policy denies every request.

### Example Trust Policy

Both conditions matter: `aud` proves the token was minted for AWS STS, and `sub` proves it was minted for *this* connector.

> AWS exposes only `aud`, `sub` and a handful of standard claims as trust-policy condition keys for a self-hosted OIDC provider — custom claims are never placed in the request context, so a policy that pins one matches nothing and every `AssumeRoleWithWebIdentity` fails with `AccessDenied`. The connector identity therefore lives in the subject.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/<issuer-host>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<issuer-host>:aud": "sts.amazonaws.com",
          "<issuer-host>:sub": "devplus:<slug>"
        }
      }
    }
  ]
}
```

### AWS CLI

```bash
# Values for THIS connection (shown in the DevPlus connector form)
CONNECTOR_SLUG="<slug>"                       # e.g. 7f3a91c2
ROLE_NAME="DevPlus-<slug>"                    # e.g. DevPlus-7f3a91c2
ISSUER_HOST="<issuer-host>"                   # e.g. oidc.example.com

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$ISSUER_HOST"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$ISSUER_HOST:aud": "sts.amazonaws.com",
          "$ISSUER_HOST:sub": "devplus:$CONNECTOR_SLUG"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role assumed by DevPlus for the '<connector>' connector"

# Attach permissions (adjust to least privilege)
aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

## Step 3: Configure AWS Connector in DevPlus

1. Navigate to **Admin > Connectors**, click Amazon Web Services, then **New connector** (or open an existing AWS connector to edit it)
2. Enable OIDC:
   - **Use OIDC authentication**: Enable the toggle
   - **AWS account ID**: Enter your AWS account ID (e.g., `123456789012`)
   - **IAM Role ARN**: Inferred from the account ID and this connector's role name, as `arn:aws:iam::[AccountID]:role/DevPlus-<slug>` (shown under the account ID field)
   - **OIDC audience**: `sts.amazonaws.com` (default, must match AWS audience)
3. Save the connector
4. **Test connection**: DevPlus assumes the role with a test token and calls `sts:GetCallerIdentity`. The result sets the connector status (`connected` or `error`).

## Step 4: Authorize Project to Use Connector

1. Open the AWS connector under **Admin > Connectors**
2. In the **Project access** card, click **Authorize project**
3. Select the project that should use the connector
4. This creates a Project-Connector authorization record; the backend refuses a deployment whose project is not authorized for its connector

## Step 5: Deploy with OIDC

When you create a deployment:

1. DevPlus generates a JWT token for the deployment (`aud` = the connector's audience, `sub` = `devplus:<slug>`, plus a `connector_id` claim)
2. Executor service retrieves the token
3. Executor calls AWS STS `AssumeRoleWithWebIdentity` with:
   - JWT token
   - IAM Role ARN
   - Session name `devplus-<deploymentId>` (CloudTrail records it, so each deployment stays traceable)
4. AWS returns temporary credentials
5. Terraform uses credentials via environment variables:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_SESSION_TOKEN`

The executor also writes the root `provider "aws"` block, with the connector's region (default `us-east-1`).

## Troubleshooting

### Token Validation Fails

- Verify OIDC provider URL matches in both AWS and DevPlus: the provider must be created for the token's `iss` (the `issuer` in `<issuer>/.well-known/openid-configuration`), not for the browser host. A mismatch fails with `No OpenIDConnect provider found`.
- Check that `<issuer>/.well-known/openid-configuration` and `<issuer>/.well-known/jwks.json` are served publicly by the oidc-service (not the frontend)
- Check that the OIDC service is accessible from the executor

### AssumeRoleWithWebIdentity Fails

- Verify IAM role trust policy allows the OIDC provider
- Check that audience matches (`sts.amazonaws.com`)
- Verify role ARN is correct (should be `arn:aws:iam::[AccountID]:role/DevPlus-<slug>`, as shown in the connector form)
- Ensure the role name matches the connector form exactly — a role created for a *different* connector will not accept this connector's token
- `AccessDenied` with a valid token usually means the trust policy's `<issuer-host>:sub` condition does not match this connector's slug
- Check IAM role permissions

### Credentials Not Working in Terraform

- Verify environment variables are set in executor logs
- Check that credentials haven't expired
- Verify AWS region is configured correctly

## Security Best Practices

1. **Use Least Privilege**: Grant IAM roles only the permissions needed
2. **Token Expiration**: deployment JWT tokens expire after 1 hour (fixed in the oidc-service)
3. **Audit Logging**: Enable CloudTrail to monitor AssumeRoleWithWebIdentity calls
4. **Condition Keys**: Pin `aud` and `sub` only; restrict which projects use a role by authorizing projects on its connector in DevPlus
5. **Key Rotation**: The oidc-service signs with the key in the `oidc-keys` secret, or with a key it generates at startup when that secret is empty. The JWKS `kid` is derived from the key, so AWS fetches the new key after a rotation.

## Advanced: Narrowing a Role Further

Each connector already has its own role, scoped by the `sub` condition. To split permissions *within* a connection, create one connector per boundary (each gets its own slug and role, with its own policies) and authorize each connector only for the projects that need it. Do not add conditions on custom claims such as `project_id`: AWS never places them in the request context, so the role would deny every request.

Several connectors can point at the *same* AWS account — their roles differ by slug, so they never collide, and none of them can assume another's role.

## Existing Connections

Connectors created before per-connection naming have no slug. They keep the old role name `DevPlusDeploymentRole` and a trust policy that conditions on `aud` only, and they keep working exactly as they are. **Nothing needs to be re-created, and nothing on the AWS side needs to change.** DevPlus derives these names from the connector (`Connector.IAMRoleName()`, `Connector.OIDCSubject()`), which falls back to the legacy values when the slug is empty; the slug is never back-filled. Their deploy tokens keep the oidc-service's default subject (the deployment id), since the trust policy does not pin `sub`. The flip side: any token from the same issuer with audience `sts.amazonaws.com` satisfies a legacy role's trust, which is why migrating is worth it.

To move an existing connection onto the per-connection model, migrate it — do not rename its cloud resources:

1. Create a **new** connector in DevPlus (it gets a slug)
2. Run the setup commands from its form (new role, trust policy with the `sub` condition)
3. Re-point the projects at the new connector
4. Delete the old connector, then its old IAM role

> **Never rename the IAM role of a live connector.** Its ARN is persisted on the connector under the legacy role name, and its trust pins only the audience. Renaming the role, or adding a `sub` condition in place, breaks every deployment, discovery scan and secret lookup that connection performs.

## References

- [AWS IAM OIDC Identity Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [AssumeRoleWithWebIdentity API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html)
- [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)
