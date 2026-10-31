---
id: da3b9ff0-b5a7-4aed-8679-145c33f8fabe
binder: binder-mvvle32a-1
---
DevPlus cost baselines

## 2026-09-28-baseline-3months.md
# DevPlus cloud spend baseline — June, July, August 2026

Goal: Cheaper, faster DevPlus infrastructure (id 23ace132). Board item 6bf48986.
Measured 2026-09-28 by DevPlus Ops. Source: Azure Cost Management `ActualCost` API
(`Microsoft.CostManagement/query`, api-version 2023-11-01), per subscription, grouped by
ServiceName / ResourceGroupName / Meter. Live resource facts from `az aks`, `az vm`,
`az monitor log-analytics query`, `kubectl` against `~/.kube/cove-sre.yaml`.

**Revised 2026-09-28.** Two of my own earlier conclusions in this document were wrong and are
corrected below, with the evidence: the system-pool SKU swap (withdrawn) and the cause of the
spot pool never provisioning (quota, not architecture).

## 1. Where the money went

Three complete months across all three DevPlus subscriptions.

| Service group | Jun | Jul | Aug | 3-mo |
|---|---:|---:|---:|---:|
| Compute — AKS nodes (VMs) | 994 | 2,702 | 2,565 | **6,260** |
| Security — Microsoft Defender for Cloud | 329 | 860 | 909 | **2,098** |
| Network — VNet + NAT GW + Watcher + DNS | 455 | 712 | 787 | **1,954** |
| Security — Sentinel / Log Analytics | 157 | 17 | 1,533 | **1,707** |
| Data — MongoDB Atlas (billed via Azure SaaS) | 374 | 1,101 | 0 | **1,474** |
| AKS control plane (Standard Uptime SLA) | 173 | 371 | 372 | 917 |
| Storage — managed disks + blobs | 141 | 336 | 391 | 868 |
| Other (ACI, KeyVault, ACR, AI, Cosmos…) | 82 | 50 | 49 | 181 |
| **Total** | **2,705** | **6,149** | **6,605** | **15,460** |

Month on month: July **+127%** vs June, August +7.4% vs July.
By subscription (3-mo): prod $9,428 · stage $3,513 · dev $1,880.

Run-rate, measured daily (`granularity: Daily`, 1–28 September):

| Window | $/day | ≈ $/month |
|---|---:|---:|
| 1–19 Sep | 276.05 | 8,281 |
| 20–28 Sep | 211.11 | 6,333 |

The run-rate fell ~$1,950/month on 20 September. That is **not** a deliberate optimisation —
see §3.

## 2. Compute: the biggest line, and the spot-first design is not working in prod

September VM cost by cluster, with the node makeup measured live:

| Cluster | System pool | Karpenter spot | Karpenter on-demand | Aug VM $ |
|---|---|---|---:|---:|
| prod eus2 | 3 × D4pds_v6 | **0** | 3 × D4ps_v6 | 724 |
| prod cus | 3 × D4pds_v6 | **0** | 3 × B4ps_v2 | 812 |
| stage eus2 | 2 × D4pds_v6 | 1 × D2ps_v6 | 2 × D2ps_v6 | 393 |
| stage cus | 2 × D4pds_v6 | **0** | 1 × D2ps_v6 | 401 |
| dev eus2 | 1 × D2pds_v6 | 1 | 4 × D2 | 235 |

Retail East US 2, Linux, pay-as-you-go vs spot (retail price API):

| SKU | On-demand | Spot | Ratio |
|---|---:|---:|---:|
| D4pds_v6 | $0.1840/hr | $0.0340/hr | 5.4× |
| D4ps_v6 | $0.1400/hr | $0.0259/hr | 5.4× |
| D2pds_v6 | $0.0918/hr | $0.0170/hr | 5.4× |
| D2ps_v6 | $0.0702/hr | $0.0130/hr | 5.4× |
| B4ps_v2 | $0.1530/hr | $0.0269/hr | 5.7× |

**Finding A — prod runs entirely on the expensive fallback. The cause is a spot quota of 3
vCPUs, and our startup programme has blocked raising it.** The cluster-bootstrap chart
(`k8s/charts/cluster-bootstrap/values.yaml`) makes the `general` pool spot-only (weight 100)
and the `ondemand` pool the weight-10 fallback. Measured live, prod has **zero spot nodes** in
both clusters; all application capacity sits on the on-demand pool, and Karpenter logs
`NoCompatibleInstanceTypes` on `nodepool/general` continuously (84×/hour in prod eus2,
1,036×/hour in dev).

Root cause, verified 2026-09-28. **My first hypothesis — that arm64 spot capacity was
unavailable, so the pool filtered everything out — was wrong, and the fix I first proposed
with it is withdrawn:**

- The subscription's regional **spot** quota, *Total Regional Low-priority vCPUs*
  (`lowPriorityCores`), is **3** everywhere:

  | subscription / region | used / limit |
  |---|---|
  | prod eastus2 | 0 / 3 |
  | prod centralus | 0 / 3 |
  | stage eastus2 | 2 / 3 |
  | dev eastus2 | 2 / 3 |

  No 4-vCPU spot VM fits under a limit of 3, so prod can never place a single spot node.
- The Azure activity log agrees. In prod's eus2 node resource group over the last 30 days, 22
  `virtualMachines/write` operations failed with
  `OperationNotAllowed … exceeding approved LowPriorityCores quota … Current Limit: 3,
  Current Usage: 0, Additional Required: 4` (the rest are the Started/Succeeded pairs of the
  same attempts).
- Karpenter's reaction makes the pool look empty rather than under-quota: on a low-priority
  quota error it marks **every** spot offering (every SKU, every architecture) unavailable for
  one hour, so each pass logs `NoCompatibleInstanceTypes` and pods land on `ondemand`.
- This also explains stage's single 2-vCPU spot node: it fits in the 2 of 3 used, and nothing
  larger ever can. The architecture is a red herring — the quota gates all spot, so widening
  the pool to amd64 would fail identically.

**The programme constraint.** All three subscriptions are on the `Sponsored_2016-01-01` offer
(Azure Sponsorship / Microsoft for Startups). We have already tried to raise spot quota and
Azure refused, on the grounds that we are on the startup programme with credits allocated.
So this is **not** a Terraform change and not a routine quota bump — it needs a supported
request with the cost argument spelled out. See §6.

**Finding B — the system pools carry more nodes than they need, but only one region can lose
one.** 6 × D4pds_v6 in prod and 4 × D4pds_v6 in stage run at **3–5% actual CPU** and 14–28%
memory. CPU *requests* are the constraint that matters for scheduling, and per node they
measure 26–47% of D4pds_v6 allocatable. `enableAutoScaling` is `false` on every pool and
`az aks nodepool list` reports a single `system` pool per cluster, so these counts are fixed
and unmanaged.

**Corrected sizing — a straight D4pds_v6 → D2pds_v6 swap is NOT safe.** Measured requests are
1,020–1,599m on prod system nodes and 1,247–1,827m on stage. Against a D2pds_v6 (~1,900m
allocatable) that is **80% (prod eus2) to 96% (stage cus)** of allocatable — no headroom for a
new daemonset pod, and scheduling/eviction failures in exactly the place you least want them.
The earlier draft of this document proposed the swap and the ~$670/mo it implies; that was
wrong and is withdrawn.

**Corrected count — eus2 only.** The defensible change is *fewer nodes, same SKU*: prod
**eus2** 3 → 2, prod **cus stays at 3**. Total eus2 requests are 3,648m, which is 47% of a
two-node D4pds_v6 pool — comfortable. That saves 1 × $134/mo = **$134/mo**.

Why not cus: its three system nodes sit one per zone across all three AZs, so dropping one
trades AZ coverage 3 → 2 for $134/mo. Not worth it. In eus2 the zones are 2 and 3 only and the
third node merely doubles up zone 2, so 2 nodes keep the same 2-AZ coverage.

Note two operational caveats: the count sits inside `ignore_changes`
(`modules/region/aks.tf`), so the Terraform change alone does not resize a live cluster — it
needs one deliberate `az aks nodepool scale`; and `system_node_count` is a single top-level
variable shared by both prod regions, so the per-region override added in PR #84 is what makes
"eus2 only" possible.

Sizing the pools down to D2pds_v6 needs a memory/daemonset headroom pass first, and merits its
own change.

**Correction to an earlier draft of this document:** an earlier version described the Karpenter
on-demand nodes as "sitting idle". That is wrong and the measured data contradicts it — those
nodes carry 19–25 pods each at **79–83% CPU requests** (`kubectl describe node`). They are the
nodes doing the work. The waste is not idle nodes; it is paying on-demand rates for capacity the
platform was designed to buy at spot.

## 3. Sentinel: an 11× step change, then a silent stop — an incident, not a saving

`modules/region/monitoring.tf` ships AKS control-plane `kube-audit-admin` + `guard` to the hub
workspace `log-devplus-hub-eus2` for all five clusters.

- **15 Aug**: commit `00fcf60` "Collect AKS control-plane audit logs into the hub workspace"
  switched this on. Ingest went from ~0 to **16.7 GB/day** across the five clusters, starting
  15 Aug 22:00 UTC. August Sentinel cost $1,533 (vs $17 in July).
- **20 Sep**: four of five clusters stopped sending. Per-cluster `kube-audit-admin` GB/day:

| Cluster | 1–19 Sep | 20–28 Sep | Last control-plane row |
|---|---:|---:|---|
| aks-devplus-dev-eus2 | 3.7 | 3.7 | 2026-09-28 13:45 (live) |
| aks-devplus-prod-eus2 | 3.1 | **0** | 2026-09-20 05:20 |
| aks-devplus-prod-cus | 3.2 | **0** | 2026-09-20 05:20 |
| aks-devplus-stage-eus2 | 3.2 | **0** | 2026-09-20 03:56 |
| aks-devplus-stage-cus | 3.3 | **0** | 2026-09-20 03:58 |

Both categories stopped: `guard` last seen 05:14 (prod eus2), `kube-audit-admin` 05:20.
The diagnostic setting on all four clusters still reports both categories **enabled**, pointing
at the hub workspace. `search "*"` over the workspace finds no rows for those clusters in any
table, and the two `DefaultWorkspace-*` workspaces hold no AKS data. Prod NSG flow logs are
still arriving (2,176 rows in the last hour), so the workspace and the pipeline are fine — only
the AKS control-plane feed broke.

Impact: Kubernetes API audit and Entra `guard` evidence for **both production clusters and both
stage clusters has been dark for 8 days**. DevPlus runs an active SOC 2 / Drata programme
(commits reference `DCF-777` and Drata controls), so this is a compliance exposure, not just a
monitoring gap.

Cost implication: full five-cluster shipment is the $8,281/mo run-rate; dev-only is $6,333/mo.
Restoring prod + stage is ≈ **$1,950/month** — which is why the September "drop" must not be
recorded as a saving.

## 4. Other confirmed items

- **Atlas**: bills through Azure as a Marketplace SaaS meter in `devplus-hub-eus2`:
  Jun $374, Jul $1,101, **Aug $0**. Atlas's own billing was **not checked** — the Vault has no
  `mongodb-atlas` entry and the `atlas` CLI session is expired. Not counted as healthy.
- **Cloudflare**: not checked — Vault has no `cloudflare` entry (`vault_list` shows only
  `cove-github-app`).
- **p95 API latency**: no baseline. Per-env Application Insights exists (`defender.tf`) but is not
  readable from the hub workspace, which holds only AzureDiagnostics, NTANetAnalytics,
  AzureMetrics and Sentinel tables. The goal's second measure is still unmeasured.
- **Defender for Cloud** grew $329 → $909/mo: subscription vCore packs (prod $389, stage $267),
  per-node plans, malware-scanning ingest. Dev $79/mo, stage $266/mo.
- **Network** $1,954/3-mo: six NAT gateways (~$83–100/mo each) plus Network Watcher traffic
  analytics (~$95/mo) and private-endpoint hours.
- **Storage** $868/3-mo: mostly P10/P4 LRS disks in the AKS node resource groups.
- **Foundry/AI**: dev holds a very large model catalogue (60+ deployments) but September spend is
  small (~$318 over 28 days, `gpt-6-astra`, and a `gpt-6-luna` deployment created 22 Sep with
  capacity 10000). Idle-deployment cost is real but currently minor; worth a monthly check.
- No unattached managed disks (Resource Graph, all three subscriptions: 0 rows).
- Reliability at measurement: all 5 clusters reachable; 6/6, 6/6, 5/5, 4/4, 6/6 nodes ready;
  0 unhealthy pods; 0 deployments short. WARP connected.

## 5. Savings, with numbers

| # | Change | Est. saving | Risk |
|---|---|---:|---|
| 1 | Drop the prod **eus2** system pool from 3 to 2 nodes (same D4pds_v6 SKU); cus keeps 3 for 3-AZ coverage. In PR #84 | **$134/mo** | Low — eus2 requests are 47% of a two-node pool and 2-AZ coverage is unchanged. Needs one manual `az aks nodepool scale` (count is in `ignore_changes`) |
| 2 | Raise the prod subscription's regional spot quota (`lowPriorityCores`, 3 → 64) in eastus2 and centralus so the spot-first pool can finally provision | **$520/mo** | Medium — Azure quota request, not Terraform, and the startup programme has refused it before. Prod then genuinely runs on spot and will see evictions; the `ondemand` fallback and overprovisioning cover them exactly as today |
| 3 | Re-tier the AKS audit feed: resource-specific tables + Basic Logs, retention 365 → 180 days, 12-month evidence exported to Blob. See §8 — this is the big one | **$1,500–2,300/mo** | Medium — changes the evidence model; Sentinel analytics rules need the Analytics plan, Auxiliary would drop alerting |
| 4 | Scope Defender plans to production where not contractually required (stage servers + malware scanning on non-prod) | **$150–345/mo** | Medium — reduced non-prod coverage; confirm no compliance requirement |

Savings 1+2 together are ≈ **$654/month** and are pure compute efficiency: same workloads,
same capacity, cheaper purchase. Both were revised down from my first draft as the evidence
came in: the D4pds_v6 → D2pds_v6 swap was withdrawn (measured requests leave 80–96% of a
D2pds_v6's allocatable used), prod cus keeps its third node for 3-AZ coverage, and the spot
saving needs a quota increase rather than a Terraform fix.

Savings 3 and 4 are the next tranche (up to ~$945/mo) and have not been worked up to proposals
yet.

## 6. The Azure startup-programme constraint (blocking saving 2)

All three subscriptions carry `quotaId = Sponsored_2016-01-01`. We have raised spot quota
before (SR **#2606070040000158**) and it was refused on the basis that we are on the startup
programme with credits allocated. Microsoft's own route back, given to us on 15 Sep by Edna
Fogaça (Startup Advisor):

> Help + support → Create a support request → Service and subscription limits (quotas) →
> Compute-VM (cores-vCPUs) Subscription limit increases → Manage Quota. From there, select the
> appropriate Spot Quota and Region, choose Edit, enter the new limit required, and submit the
> request. … Once he submits the Quota request from the Azure portal, please send us your SR
> ticket number. Then, we can escalate it with the Microsoft for Startups Capacity Request team.

The escalation team had earlier asked for exactly this: "please raise a new request for the Spot
Quota requirements." So the new ticket must make the case in the terms that answer the
refusal — this is a request to move workload we already run from on-demand to spot, which
*reduces* the cost Microsoft absorbs, not a request for more capacity or more credit.

Ticket parameters: service `Service and subscription limits (quotas)`
(`06bfd9d3-516b-d5c6-5802-169c800dec89`), problem classification `Compute-VM (cores-vCPUs)
subscription limit increases` (`e12e3d1d-7fa0-af33-c6d0-3c50df9658a3`), resource
`lowPriorityCores`.

**Filed 2026-09-28: support request `2609280040007729`** (status Open, severity Moderate,
service and problem classification as above, for subscription a7242621 and both eastus2 and
centralus, requesting 3 -> 64). Verified by reading the ticket back. The quota limit is still 3,
which is expected until Microsoft approve it. Next step: send the SR number to Edna Fogaça for
escalation by the Microsoft for Startups Capacity Request team.

Why 64 and not less: today's prod load needs 12 vCPUs per region (3 x 4-vCPU nodes), and 64 per
region leaves room for a replacement node plus the 3 x 2 CPU overprovisioning replicas. The
NodePool CPU limit (128 in prod) then becomes the only cap.

## 7. Open decisions

- §3: restore prod + stage control-plane audit logging (~$1,950/mo) or knowingly accept the
  gap. Owner decision. The route is now configuration as code: PR #85 (branch
  `fix/aks-control-plane-diag-recreate`) makes a normal `tofu apply` recreate the setting on
  all four dark clusters, and the CI plans confirm it (prod 4 add / 4 change / 2 destroy,
  prod eus2 and prod cus both "must be replaced"). What is left is Helder's call on the
  cost: merge and apply PR #85, or accept the SOC 2 evidence gap.
- §6: the spot quota request is filed (`2609280040007729`); route the SR number to Edna
  Fogaça for escalation. Approval card up.

## 8. The audit-log bill is a design choice, not a price (measured 2026-09-29)

Helder's challenge — "**1900 a month for logs sounds absurd, is there no more cost optimal
solution?**" — is correct. The measurement:

**What we actually pay.** The hub workspace is Sentinel-onboarded with
`unifiedSentinelBillingOnly: true`, so every byte is billed at Sentinel's pay-as-you-go Analysis
rate, **$4.76/GB** (retail eastus2; Log Analytics' own Analytics ingestion is $2.76/GB). Billed
September 1–28 in `devplus-hub-eus2`: **$1,958.74** on the "Pay-as-you-go Analysis" meter.
Measured volume agrees: ~19 GB/day before the 20 Sep blackout, 443.7 GB over 30 days, giving an
effective **~$4.84/GB** — the Sentinel rate, not the Log Analytics one.

**The three levers, priced at the real volumes.** Four dark clusters = 15.0 GB/day = 450 GB/mo;
all five = 18.0 GB/day = 547 GB/mo.

| Destination / plan | Rate | 4 clusters | all 5 |
|---|---:|---:|---:|
| Sentinel PAYG Analysis (today) | $4.76/GB | $2,142 | $2,604 |
| Log Analytics Analytics tier | $2.76/GB | $1,242 | $1,510 |
| **Basic Logs** | $0.50/GB | **$225** | **$274** |
| Auxiliary / Lake | $0.05/GB | $23 | $27 |
| Blob Cool (evidence only) | $0.01/GB/mo | $4.50 | $5.50 |
| Blob Archive (evidence only) | $0.002/GB/mo | $0.90 | $1.09 |

Plus retention: at 365 days hot, the months beyond the included 31 cost $0.12/GB/mo — roughly
**$720/mo** on top for the five-cluster feed. Our own logging-and-monitoring policy only calls for
**180-day native retention plus a quarterly log-evidence export to Drata for the 12-month
requirement**, so the 365-day hot setting is over-provisioned against our own standard.

**Why the log is barely used.** In the hub workspace there is **one** alert rule (`BuiltInFusion`)
and **39 saved searches, none of which reference the AKS audit data or `AzureDiagnostics`**. We
were paying SIEM rates to store logs that nothing read.

**The reason the cheap tiers are reachable.** The setting currently writes to
`AzureDiagnostics`, the legacy catch-all table, which cannot take Basic or Auxiliary. AKS has
resource-specific tables (`AKSAuditAdmin`, `AKSControlPlane`, `AKSAudit`) already present in the
workspace with no rows, because `log_analytics_destination_type` still defaults to
`AzureDiagnostics`. Pointing the diagnostic setting at `AKSAuditAdmin` is a one-line Terraform
change and is the precondition for every saving below it.

**Recommended (all five clusters, ~$350–450/mo instead of $2,604):**
1. `log_analytics_destination_type = "Dedicated"` so the feed lands in `AKSAuditAdmin`.
2. Set `AKSAuditAdmin` to the **Basic** plan ($0.50/GB) — still queryable in KQL; Sentinel
   analytics rules need Analytics, and Auxiliary would drop alerting entirely, so Basic is the
   floor that keeps incident response.
3. Keep `guard` (1.4 GB/day) in Analytics — it is the security-relevant, alertable slice.
4. Drop workspace retention 365 → **180 days**, matching the documented policy.
5. Export the 12-month evidence to **Blob Cool/Archive** via the diagnostic setting's storage
   destination — cents per GB per month instead of hot retention.

**Cheapest compliant option if we only need evidence, not query** — diagnostic setting
straight to a storage account, no Log Analytics at all: **~$5–10/mo** (Blob Cool), read back
with search jobs if ever needed.

**Caveats, stated plainly.** These are list prices from the retail API, not a re-bill, so treat
them as ±20%. Switching a table plan is limited to one change per table per week. Whether
Sentinel's Fusion rule or any Drata control depends on these tables being *Analytics*-plan needs
confirming before the plan change; that is the one check standing between this analysis and a PR.

### 8.1 Was this to unlock startup credits? Tested 2026-09-29

Helder's question: *"I think we did that to unlock additional azure startup credits. But, do we
need to keep it on?"* Checked, and the evidence does **not** tie the logging to credits:

- The three subscriptions are on `Sponsored_2016-01-01` (Azure Sponsorship / Microsoft for
  Startups) with `spendingLimit: Off`. Sep 1-29 month-to-date actual cost: prod **$4,427.79**,
  stage **$1,576.81**, dev **$1,230.17** — about **$7,235/month** in total, all of which draws on
  the sponsorship.
- The "Milestone Disbursements – 90 days" thread (8 Sep, with Plug and Play) is milestone *cash*
  funding to the bank account, not Azure credits. Its four milestones are Ignite sponsorship, the
  AI server, CRM/first ad campaign, and the design-partner program — none mention logging, SIEM or
  Log Analytics.
- The commit that switched the logging on (`00fcf60`, 15 Aug, Aaron) gives its own reason:
  "Collect AKS control-plane audit logs into the hub workspace", and explicitly excludes full
  `kube-audit` "for ingestion cost". Nothing about credits.
- Edna's correspondence is the spot quota and Marketplace/co-sell. Nothing ties the log pipeline
  to a credit condition.

**What I could not check:** the sponsorship's own terms — credit balance, expiry and any
conditions. The `Microsoft.Billing/credits` and `Consumption/balances` APIs return nothing for
this offer, and the Billing `billingPeriods` list is empty. The source of truth is the Azure
portal (Cost Management + Billing → Credits) and the Microsoft for Startups portal. **So I can
say no evidence links the logging to credits; I cannot say the linkage is impossible.**

**Checked in the Azure portal, 29 Sep (signed in as helder@devplus.io).** The billing picture is
now confirmed:

- Three billing accounts: **"DevPlus Startup Credits"** (Active), "Default Directory" (Active), and
  "DevPlus" (**Disabled**).
- **All three of our subscriptions bill against "DevPlus Startup Credits"** — prod `a7242621`,
  stage `61e4e1a9`, dev `53a006e0` — plus two further subscriptions (`9c92c03d`, `4acad28c`). So
  the sponsorship is the funding mechanism, and our ~$7,235/month of spend draws on it. That is
  the real argument for re-tiering: it is burning credit runway.
- Billing profile `Q6PV-NVSL-BG7-PGB`, currency USD, 0 invoices raised (consistent with
  sponsorship).
- **The credit balance itself is still unread.** The API
  (`Microsoft.Billing/.../billingProfiles/credits/read`) returns **Forbidden** for
  helder@devplus.io — the account is billing-account owner but lacks the read on that action. The
  portal's Credits blade is the remaining route and it should be read there.

**What the portal did settle:** nothing ties the log pipeline to the credit condition. The credits
are the funding source; the logging is SOC 2 evidence. Different things.

**Read in the portal, 29 Sep (billing account "DevPlus Startup Credits" -> Overview):**

| Field | Value |
|---|---|
| Amount due | **$0.00** — "No payment needed" |
| Billing period | 9/1/2026 – 9/30/2026 |
| Pre-tax total so far | **$7,253.48** |
| Current charges / forecast | **$7.25K** / **$7.87K** |
| Invoices over time, total | $0.00 |

Top products by charges this month, straight from the portal:

| Product | This month |
|---|---:|
| **Sentinel — Pay-as-you-go — US East** | **$1.96 K** |
| Virtual Machines Dpdsv6 Series | $692.31 |
| Virtual Machines Dpdsv6 Series (second) | $612.58 |
| Other purchases | $3.99 K |

Two things this settles:

1. **The credits are real and they are absorbing the spend.** $7,253 of charges for September and
   **$0.00 due**. So every dollar of the audit-log bill is drawn from the sponsorship, which is
   precisely why re-tiering matters — it is credit runway being spent, not cash. (The
   `microsoftazuresponsorships.com` legacy portal reports "no active sponsorship", which is
   expected for the MCA-era billing account; the Azure portal is the correct source.)
2. **The Sentinel figure reproduces independently.** The portal shows **$1.96 K** for Sentinel
   this month; my API measurement was **$1,958.74**. Same number, two sources — and it confirms
   Sentinel is the single largest line item in the account.

### 8.2 RETRACTION — there is no retention drift; the 365 days is deliberate

**I got this wrong in an earlier version of this section and withdraw it.** I claimed the
workspace retention (365) had drifted from `devplus-hub-infra/monitoring.tf` (90) and that the
difference was worth ~$720/mo. Both halves were wrong.

- **My checkout was stale** — 9 commits behind, dated 21 Jun. `origin/main` says
  `retention_in_days = 365`; only my old working tree said 90.
- **The change was deliberate.** Commit `85ec360` "Bump Log Analytics workspace retention to 12
  months" (PR #21, merged 15 Aug) raised 90 → 365 to meet the Data Retention Policy's 12-month
  requirement inside the workspace. It is not ad-hoc drift, so it is not mine to reclaim.

Corrected by Coder (PR #30) and verified by me against `origin/main`. The retention of 365 is a
policy decision that costs roughly $720/mo on the five-cluster feed; whether the policy needs the
full window in-workspace, or 180 days in-workspace plus a genuinely-running quarterly export, is
Helder's call — not a mistake to be silently "fixed".

Credit for catching it: Coder, in the PR review, and it flagged the reversal in the PR body as
its own separate commit so a reviewer can drop it.

**The two decisions are separable.** Keeping the audit logs is defensible on its own terms — SOC 2
/ Drata evidence, and the 20 Sep blackout is an open incident (PR #85) precisely because that
evidence went dark. What is not defensible is the *price*: Sentinel pay-as-you-go on the legacy
`AzureDiagnostics` table at 365 days hot. Re-tiering keeps every log and cuts the bill.
