---
id: eede8529-e196-428a-b258-06777074d0fb
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevOps | devplus-core-infra/artifacts/drata-topology-core.md

Source: DevPlusInc/devplus-core-infra/artifacts/drata-topology-core.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-core-infra, artifacts/drata-topology-core.md).

# DevPlus core-infra topology inventory for Drata / SOC 2 evidence

This inventory extracts network and dataflow facts from the `devplus-core-infra`
Terraform repo. It is documentation-only evidence; it does not query live Azure,
MongoDB Atlas, or Cloudflare state.

Related diagram sources:

- `artifacts/drata-network-core.mmd`
- `artifacts/drata-dataflow-core.mmd`

## Source-of-truth files reviewed

| Area | Terraform source |
|---|---|
| Environments, regions, CIDRs, AKS sizing | `environments/*.tfvars`, `variables.tf`, `locals.tf` |
| Regional VNets, subnets, NSGs, NAT, AKS | `modules/region/*.tf`, `main.tf` |
| Private DNS zones, hub Log Analytics | `dns.tf` |
| AI Foundry, AI Services, model deployments, private endpoints | `ai-foundry.tf`, `ai-foundry-models.tf` |
| MongoDB Atlas project, clusters, OIDC, NAT allowlists | `mongodb.tf` |
| Cloudflare ingress, tunnels, WAF, R2 evidence bucket | `modules/cloudflare-ingress-core/main.tf`, `github-callback.tf`, `auth-service.tf`, `cloudflare-security.tf`, `cloudflare-r2.tf` |
| Monitoring and alerting | `modules/region/monitoring.tf`, `defender.tf`, `otel-collector.tf`, `activity-log-alerts.tf`, `metric-alerts.tf` |

## Important scope notes

- This repo defines the environment spokes and references hub-owned shared
  services. The hub repo (`devplus-hub-infra`) owns hub VNet peering, private DNS
  VNet links, centralized Log Analytics/Sentinel onboarding, hub network
  watchers, the hub NAT public IP, and central per-environment Key Vaults.
- Spokes are described in this repo as hub-spoke with no spoke-to-spoke transit.
  That matters for private endpoint DNS: env-wide singleton resources only allow
  the primary-region private endpoint to write the shared private DNS record.
- Public inbound traffic for core services is through Cloudflare Tunnel; this
  repo does not create Azure public load balancers for `auth-service` or
  `github-callback`.

## Environment inventory

| Environment | Subscription ID | Root/global RG | Primary region | Regional spokes | Public core hostnames |
|---|---:|---|---|---|---|
| dev | `53a006e0-623b-44f1-84ce-5889e9d8e8da` | `devplus-dev` | `eastus2` | `devplus-dev-eus2` (`10.2.0.0/16`) | `auth.dev.devplus.io`, `connect.dev.devplus.io` |
| stage | `61e4e1a9-ceef-4637-a794-aaee7d480a9e` | `devplus-stage` | `eastus2` | `devplus-stage-eus2` (`10.10.0.0/16`), `devplus-stage-cus` (`10.11.0.0/16`) | `auth.stage.devplus.io`, `connect.stage.devplus.io` |
| prod | `a7242621-0d31-4681-9154-d1d0e9d7a4e6` | `devplus-prod` | `eastus2` | `devplus-prod-eus2` (`10.20.0.0/16`), `devplus-prod-cus` (`10.21.0.0/16`) | `auth.devplus.io`, `connect.devplus.io` |

Default shared settings:

- Naming prefix: `devplus`.
- Hub resource group: `devplus-hub-eus2`.
- Hub location: `eastus2` / short name `eus2`.
- Hub Log Analytics workspace: `log-devplus-hub-eus2` in `devplus-hub-eus2`.
- Hub VNet CIDR: `10.100.0.0/16` (`devplus-hub-infra/variables.tf`).
- Central per-environment Key Vault data source:
  - name: `kv-devplus-<env>`
  - RG: `devplus-keyvault-<env>`

## Regional networking inventory

Each regional module creates:

- Resource group: `devplus-<env>-<region_short>`.
- VNet: `vnet-devplus-<env>-<region_short>`.
- Subnets:
  - `snet-devplus-aks-nodes`: `cidrsubnet(vnet, 6, 0)` (`/22` at offset 0).
  - `snet-devplus-general`: `cidrsubnet(vnet, 8, 4)` (`/24` at offset 4).
  - `snet-devplus-private-endpoints`: `cidrsubnet(vnet, 8, 8)` (`/24` at offset 8).
- NAT gateway: `natgw-devplus-<env>-<region_short>`.
- NAT public IP: `pip-nat-devplus-<env>-<region_short>`.
- NSGs:
  - `nsg-devplus-<env>-<region_short>-aks-nodes`
  - `nsg-devplus-<env>-<region_short>-general`
  - `nsg-devplus-<env>-<region_short>-private-endpoints`
- Flow-log storage: `devplusfl<env><region_short>`.

| Env | Azure region | RG | VNet / CIDR | AKS nodes subnet | General subnet | Private endpoint subnet | NAT resources |
|---|---|---|---|---|---|---|---|
| dev | `eastus2` | `devplus-dev-eus2` | `vnet-devplus-dev-eus2` / `10.2.0.0/16` | `10.2.0.0/22` | `10.2.4.0/24` | `10.2.8.0/24` | `natgw-devplus-dev-eus2`, `pip-nat-devplus-dev-eus2` |
| stage | `eastus2` | `devplus-stage-eus2` | `vnet-devplus-stage-eus2` / `10.10.0.0/16` | `10.10.0.0/22` | `10.10.4.0/24` | `10.10.8.0/24` | `natgw-devplus-stage-eus2`, `pip-nat-devplus-stage-eus2` |
| stage | `centralus` | `devplus-stage-cus` | `vnet-devplus-stage-cus` / `10.11.0.0/16` | `10.11.0.0/22` | `10.11.4.0/24` | `10.11.8.0/24` | `natgw-devplus-stage-cus`, `pip-nat-devplus-stage-cus` |
| prod | `eastus2` | `devplus-prod-eus2` | `vnet-devplus-prod-eus2` / `10.20.0.0/16` | `10.20.0.0/22` | `10.20.4.0/24` | `10.20.8.0/24` | `natgw-devplus-prod-eus2`, `pip-nat-devplus-prod-eus2` |
| prod | `centralus` | `devplus-prod-cus` | `vnet-devplus-prod-cus` / `10.21.0.0/16` | `10.21.0.0/22` | `10.21.4.0/24` | `10.21.8.0/24` | `natgw-devplus-prod-cus`, `pip-nat-devplus-prod-cus` |

### NSG baseline

- `aks_nodes` and `private_endpoints` NSGs:
  - allow inbound from `VirtualNetwork`.
  - deny all other inbound at priority `4096`.
- `general` NSG:
  - deny all inbound at priority `4096`.
- Subnets are associated with their matching NSGs.

### NAT and egress

- All three subnets in every region are associated with the regional NAT gateway.
- AKS uses `outbound_type = "userAssignedNATGateway"`.
- MongoDB Atlas IP access list entries are created for every regional NAT public
  IP (`<env>-<region>-nat` comments).
- Internet egress for AKS pods to external SaaS/API endpoints goes through the
  regional NAT public IP unless the destination is reached over a private
  endpoint/private DNS path.

## AKS inventory

| Env | Region | AKS cluster | RG | Private API | Network profile | System nodes |
|---|---|---|---|---|---|---|
| dev | `eastus2` | `aks-devplus-dev-eus2` | `devplus-dev-eus2` | Enabled, private DNS zone `privatelink.eastus2.azmk8s.io` | Azure CNI overlay, Cilium dataplane/policy, service CIDR `10.1.0.0/16`, pod CIDR `10.244.0.0/16` | 1 x `Standard_D2pds_v6` |
| stage | `eastus2` | `aks-devplus-stage-eus2` | `devplus-stage-eus2` | Enabled, private DNS zone `privatelink.eastus2.azmk8s.io` | Same | 2 x `Standard_D4pds_v6`, zones `1`, `3` |
| stage | `centralus` | `aks-devplus-stage-cus` | `devplus-stage-cus` | Enabled, private DNS zone `privatelink.centralus.azmk8s.io` | Same | 2 x `Standard_D4pds_v6`, zones `1`, `2`, `3` (env default) |
| prod | `eastus2` | `aks-devplus-prod-eus2` | `devplus-prod-eus2` | Enabled, private DNS zone `privatelink.eastus2.azmk8s.io` | Same | 3 x `Standard_D4pds_v6`, zones `2`, `3` |
| prod | `centralus` | `aks-devplus-prod-cus` | `devplus-prod-cus` | Enabled, private DNS zone `privatelink.centralus.azmk8s.io` | Same | 3 x `Standard_D4pds_v6`, zones `1`, `2`, `3` (env default) |

AKS security/identity facts:

- Local Kubernetes admin account disabled.
- Azure RBAC enabled with Entra tenant.
- OIDC issuer and Azure Workload Identity enabled.
- SKU tier: dev `Free`, stage and prod `Standard`.
- User-assigned identity: `id-aks-devplus-<env>-<region_short>`.
- Disk encryption: Microsoft-managed platform keys. The customer-managed key
  setup (disk encryption sets and CMK Key Vaults, `cmk.tf`) was removed on
  2026-09-20. `host_encryption_enabled = true` on the system pool covers the
  temp disk. `disk_encryption_set_id` is in `ignore_changes`, so a cluster
  created before the switch keeps its existing disk encryption set until it
  is rebuilt.
- `prevent_destroy` blocks any plan that would replace a cluster.
- Cluster bootstrap installs Karpenter/NAP resources, storage class,
  overprovisioning, External Secrets, Reflector, Reloader, and Kyverno image
  verification in each AKS-bearing region. Kyverno runs in `Audit` mode in all
  three environments (`image_verification_failure_action` in the tfvars).
- Karpenter general pool is spot-first in dev and stage and on-demand only in
  prod; an on-demand fallback pool exists in every environment.

## Private DNS and private endpoints

Hub-owned private DNS zones referenced by this repo:

| Private DNS zone | Used for |
|---|---|
| `privatelink.blob.core.windows.net` | Flow-log storage blob endpoints and AI Foundry storage blob endpoint |
| `privatelink.queue.core.windows.net` | Flow-log storage queue endpoints |
| `privatelink.cognitiveservices.azure.com` | AI Services private endpoint |
| `privatelink.<region>.azmk8s.io` | Private AKS API endpoints |

Private endpoints created by this repo:

| Endpoint | Env/region placement | Target | DNS registration |
|---|---|---|---|
| `pe-flowlogs-devplus-<env>-<region_short>` | Every regional PE subnet | Regional flow-log storage blob subresource | `privatelink.blob.core.windows.net` |
| `pe-flowlogs-q-devplus-<env>-<region_short>` | Every regional PE subnet | Regional flow-log storage queue subresource | `privatelink.queue.core.windows.net` |
| `pe-ais-devplus-<env>-<region_short>` | Every regional PE subnet | Env AI Services account `ais-devplus-<env>` | Only primary region attaches DNS zone group |
| `pe-aifstg-devplus-<env>-<region_short>` | Every regional PE subnet | Env AI Foundry storage `devplusaif<env>` blob subresource | Only primary region attaches DNS zone group |

The AI Services and AI Foundry storage private endpoints are provisioned in
every region, but only the primary region registers the shared DNS A record. The
Terraform comments explain that all regional private endpoints would otherwise
write the same FQDN record, causing the last writer to point primary-region
workloads at a secondary-region private IP that is not reachable without
spoke-to-spoke transit.

## Data stores and storage

### MongoDB Atlas

| Env | Atlas project / cluster | Azure-backed region(s) | Network access | Auth |
|---|---|---|---|---|
| dev | `devplus-dev` | `eastus2` (`US_EAST_2`) | Atlas IP access list includes the dev regional NAT public IP | AKS workload identity OIDC IdP per cluster |
| stage | `devplus-stage` | `eastus2` (`US_EAST_2`) | Atlas IP access list includes stage `eus2` and `cus` NAT public IPs | AKS workload identity OIDC IdP per cluster |
| prod | `devplus-prod` | `eastus2` (`US_EAST_2`) | Atlas IP access list includes prod `eus2` and `cus` NAT public IPs | AKS workload identity OIDC IdP per cluster |

Additional Atlas controls:

- Dedicated M10 replica set clusters.
- Backups enabled and termination protection enabled.
- Stage/prod backup schedule with weekly copy to `centralus`.
- Atlas auditing enabled for authentication, authorization failures, user/role
  changes, and DDL/index events.
- `auth-service` gets a custom role limited to `auth.workos` and authenticates
  as an OIDC database user with subject
  `system:serviceaccount:auth-service:auth-service`.
- The org-level federation config (`dataAccessIdentityProviderIds`) is managed
  only by the prod workspace and lists every environment's AKS IdPs.
- Stage/prod alerts: CPU, replication lag, connections, and snapshot
  fallback/behind events.

### Azure storage

| Storage account | RG | Region | Purpose | Network posture |
|---|---|---|---|---|
| `devplusfl<env><region_short>` | `devplus-<env>-<region_short>` | Matching regional location | VNet flow-log storage | Public network access disabled, network default deny, shared-key auth disabled, private endpoints for blob and queue, GRS, Microsoft-managed keys |
| `devplusaif<env>` | `devplus-<env>` | Primary region `eastus2` | AI Foundry backing storage | Public network access disabled, network default deny, private endpoint for blob, GRS, Microsoft-managed keys |
| `devplus-edge-logs` | Cloudflare R2, prod only | Cloudflare account | Cloudflare Logpush exports for HTTP requests, firewall events, account audit logs | R2 lifecycle deletes after 365 days |

### Key Vault

| Vault | RG | Region | Purpose | Network posture |
|---|---|---|---|---|
| `kv-devplus-<env>` | `devplus-keyvault-<env>` | Hub-managed | Central per-env secrets consumed by deploy and AI Foundry | Hub-owned; referenced by data source |

This repo no longer creates Key Vaults: the CMK vaults were removed with the
switch to Microsoft-managed keys on 2026-09-20.

## AI Foundry and AI Services

| Resource | Name | RG | Region | Network posture |
|---|---|---|---|---|
| AI Services account | `ais-devplus-<env>` | `devplus-<env>` | Primary `eastus2` | Public network access disabled, local auth disabled, private endpoints in each region |
| AI Foundry hub | `aif-devplus-<env>` | `devplus-<env>` | Primary `eastus2` | Public network access disabled, managed network `AllowInternetOutbound` |
| AI Foundry project | `devplus-<env>` | Associated with hub | Primary `eastus2` | Inherits hub networking |
| AI Foundry storage | `devplusaif<env>` | `devplus-<env>` | Primary `eastus2` | Public network disabled, blob PE in each region |

Model deployments on the AI Services account:

- `gpt-5.4` (`GlobalStandard`, capacity `200`).
- `gpt-5.4-mini` (`GlobalStandard`, capacity `100`).
- `text-embedding-3-small` (`GlobalStandard`, capacity `100`).
- A catalogue of 86 further pay-per-token `GlobalStandard` deployments in every
  environment (`ai-foundry-models.tf`); Azure Marketplace models are excluded
  and a `marketplace-spend-<env>` budget alerts on any Marketplace charge.
- `FW-GLM-5.3-Flash` (Fireworks, `GlobalStandard`) in dev only
  (`fireworks_enabled`). Requests to it are served by Fireworks AI, outside
  the EU Data Boundary.

Role assignments:

- Foundry hub identity has `Cognitive Services OpenAI Contributor` on the AI
  Services account and `Key Vault Secrets User` on the central Key Vault.
- Foundry project identity has `Cognitive Services OpenAI User` on the AI
  Services account.
- The admins group and the Foundry project identity have `Azure AI Developer`
  on the AI Services account (needed to call the non-OpenAI models).

## Public ingress and in-cluster core services

Cloudflare controls:

- Prod workspace owns the zone-level custom firewall and managed WAF rulesets.
- Custom rules block non-WorkOS IPs on `/webhooks/workos`, block unused HTTP
  methods (`TRACE`, `TRACK`, `CONNECT`), and challenge Tor traffic except
  webhooks.
- Managed WAF executes Cloudflare Managed Ruleset and OWASP Core Ruleset.

Cloudflare Tunnel pattern:

1. Cloudflare DNS creates a proxied CNAME:
   `<dns_name>` -> `<tunnel_id>.cfargotunnel.com`.
2. Cloudflare Zero Trust tunnel config routes the hostname to
   `http://<service_name>:<service_port>`.
3. A `cloudflared` deployment runs in the same Kubernetes namespace as the
   service, using a Kubernetes secret for the tunnel token.
4. Service exposure inside AKS remains `ClusterIP`.

| Service | Public hostname | Kubernetes namespace | Service | Cloudflare tunnel name | Main downstreams |
|---|---|---|---|---|---|
| `auth-service` | dev: `auth.dev.devplus.io`; stage: `auth.stage.devplus.io`; prod: `auth.devplus.io` | `auth-service` | `auth-service` `ClusterIP:80` -> pod `:8080` | `devplus-core-auth-service-<env>` | MongoDB Atlas over NAT, WorkOS API over NAT, OTLP to collector |
| `github-callback` | dev: `connect.dev.devplus.io`; stage: `connect.stage.devplus.io`; prod: `connect.devplus.io` | `github-callback` | `github-callback` `ClusterIP:80` -> pod `:8080` | `devplus-core-github-callback-<env>` | Tenant callback redirect, OTLP to collector |

## Monitoring and audit flows

| Signal | Source | Destination | Evidence resources |
|---|---|---|---|
| VNet flow logs | Each regional VNet | Regional flow-log storage, optional Traffic Analytics to hub workspace | `azapi_resource.flow_log`, `devplusfl<env><region_short>` |
| NSG events/rule counters | Regional NSGs | Hub Log Analytics workspace | `diag-nsg-*` |
| NAT metrics and NAT public IP DDoS logs | Regional NAT gateway/public IP | Hub Log Analytics workspace | `diag-nat-gateway`, `diag-nat-public-ip` |
| VNet metrics and VM protection alerts | Regional VNet | Hub Log Analytics workspace | `diag-vnet` |
| AKS control plane writes/authz | Private AKS clusters | Hub Log Analytics workspace, resource-specific tables `AKSAuditAdmin` and `AKSControlPlane` | `diag-aks-control-plane-v2` categories `kube-audit-admin`, `guard` |
| AI Services audit/request logs | `ais-devplus-<env>` | Hub Log Analytics workspace | `diag-ai-services` |
| AI Foundry hub/project logs | Foundry hub and project | Hub Log Analytics workspace | `diag-ai-foundry-hub`, `diag-ai-foundry-project` |
| App traces/metrics | `auth-service`, `github-callback` | OTLP -> in-cluster OTel collector -> App Insights `appi-devplus-<env>` -> hub workspace | `otel-collector.tf`, `defender.tf` |
| Cloudflare edge logs | Cloudflare Logpush jobs (dashboard-managed) | Cloudflare R2 bucket `devplus-edge-logs` (prod) | `cloudflare-r2.tf` |
| Subscription governance | Azure Activity Log | Security action group `ag-security-<env>` | Policy assignment, NSG, public IP, SQL firewall, security solution alerts |
| Node CPU saturation | AKS clusters (prod also node VMSS/VMs) | Security action group | `alert-aks-node-cpu-*`, prod `alert-aks-vmss-cpu-*`, `alert-aks-vm-cpu-*` (> 90% for 15 min) |
| Atlas audit log | Atlas project | Atlas audit log (downloadable ~30 days) | `mongodbatlas_auditing.main` |

Microsoft Defender for Cloud plans enabled by this repo:

- Storage Accounts (`DefenderForStorageV2`) with malware scanning and sensitive
  data discovery.
- Containers with agentless VM scanning, Kubernetes discovery, registry
  vulnerability assessments, and container sensor.
- Key Vaults, ARM, and AI services (AI services is `Free` in dev:
  `defender_ai_services_enabled = false`).

## Traffic and dataflow summary

### Public inbound: internet to Cloudflare to AKS

1. Internet clients, GitHub App install redirects, and WorkOS webhooks connect to
   `auth*` or `connect*` hostnames.
2. Cloudflare DNS/WAF/Zero Trust receives traffic. Prod zone WAF rules apply at
   this layer.
3. Traffic enters a Cloudflare Tunnel and is delivered to the `cloudflared` pod
   running inside the relevant eastus2 AKS namespace.
4. `cloudflared` forwards to the in-cluster `ClusterIP` service.
5. The service pod processes the request. There is no Azure public load balancer
   for these core services in this repo.

### Private Azure service access

1. AKS workloads resolve Azure service FQDNs through hub-owned private DNS zones
   linked to the spokes by hub infrastructure. The hub excludes the `azmk8s`
   zones from its spoke links; each cluster uses its region's zone as its
   custom private DNS zone.
2. Blob, queue, AKS API, and AI Services names resolve to private endpoint
   IPs in the spoke private endpoint subnets.
3. Primary-region private endpoints own DNS records for env-wide singleton
   AI Services and AI Foundry storage FQDNs.
4. Flow-log storage, private AKS API, and regional private endpoint traffic
   stays on private IP space.

### Data store access

1. `auth-service` uses Azure Workload Identity and Atlas workload OIDC; no static
   MongoDB password is defined here.
2. Atlas network access allows only the regional NAT public IPs created by the
   spokes.
3. Connections from AKS pods to Atlas leave the spoke through the regional NAT
   gateway and reach Atlas public endpoints constrained by the IP access list.

### AI Foundry / AI Services dataflow

1. Consumers authenticate to AI Services with Entra ID/AAD; local shared-key auth
   is disabled on `ais-devplus-<env>`.
2. AI Services public network access is disabled; access is through private
   endpoints.
3. The AI Foundry hub uses `devplusaif<env>` storage and central Key Vault.
4. AI Services, Foundry hub, and Foundry project diagnostics flow to the hub Log
   Analytics workspace.

### Observability and audit flow

1. In-cluster services emit OTLP traces/metrics to
   `otel-collector.observability:4317`.
2. The OTel collector exports to Application Insights `appi-devplus-<env>`,
   which is workspace-based on the hub Log Analytics workspace.
3. Network diagnostics and flow logs are sent to regional private storage and/or
   hub Log Analytics.
4. Cloudflare edge/audit logs are exported to the prod R2 evidence bucket with a
   365-day lifecycle.
