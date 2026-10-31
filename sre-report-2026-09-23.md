---
id: 91f2640f-f497-422c-bb91-10f57d0e7499
binder: binder-mvvle32a-1
---
SRE report · 2026-09-23

**Infrastructure · Wednesday, 23 September 2026** — Healthy on the five clusters: 0 critical, 0 unhealthy pods, 0 pending, all deployments at full replicas. **But Cloudflare and MongoDB were not checked** (no Vault credentials yet), so this is not a complete picture — 2 of 7 systems are unknown.

| System | Status | The number that matters |
|---|---|---|
| Prod eus2 | Healthy | 6/6 nodes ready · 0 unhealthy pods |
| Prod cus | Healthy | 6/6 nodes ready · 0 unhealthy pods |
| Stage eus2 | Healthy | 5/5 nodes ready · 1 node memory 103% |
| Stage cus | Healthy | 4/4 nodes ready · metrics-server rolled 2/2 |
| Dev eus2 | Healthy with pressure | 5/5 nodes ready · 2 nodes memory 101/105% |
| Cloudflare | **Not checked** | no Vault entry `cloudflare` |
| MongoDB Atlas | **Not checked** | no Vault entry `mongodb-atlas` |

All five API servers reachable through WARP (Connected). Azure subscription states all Enabled. No namespaces deleted, nothing changed — read-only run.

**Needs a decision**

1. **Cloudflare and Atlas credentials.** Two systems I'm supposed to watch were unchecked, not healthy. Add a Vault entry named `cloudflare` (API token in the secret field) and one named `mongodb-atlas` (service account client id as username, secret as secret). Until then the daily report covers AKS only, and I can't see 5xx rates, tunnel health or Atlas alerts.
2. **Karpenter can't add nodes** — `nodepool/general` throws *NoCompatibleInstanceTypes* in **all five clusters** (109–1163 events/hour). Requirements are arm64 + spot + SKU families D/B; Karpenter says that filters out every available instance type. Nothing is down, because the older `ondemand` node pool carries today's load and zero pods are pending — the risk is a burst that needs a new node and can't get one. This is a definition fix (NodePool/NodeClass in code), not a live patch, so I have not touched it. Say the word and I'll investigate the exact SKU requirement and propose the change on an approval card.

**Watching**

- **Node memory above allocatable** on 3 spot nodes: dev `aks-ondemand-8wv9k` 101%, dev `aks-ondemand-m4prb` 105%, stage-eus2 `aks-ondemand-vf87d` 103%. All three nodes are Ready with no MemoryPressure and no evictions; the dev node is at 94% of memory *requests*, mostly `ama-logs`. Why it matters: a working set that crosses allocatable can start evicting. I'll watch for MemoryPressure or evictions; if they appear, the smallest fix is trimming ama-logs memory or letting Karpenter move the pods.
- **Kyverno `verify-ghcr-images` violations in dev** (456/hour) — unsigned local-build images (`ghcr.io/devplusinc/executor:tilt-…`) in the `helder` namespace only. Expected for local dev via Tilt; worth confirming it never reaches stage or prod.
- **Event noise on stage** — `ama-logs` 160× and `microsoft-defender-publisher-ds` 54× FailedMount per hour on stage-cus, plus NetworkNotReady. Both pods are 2/2 Running now, so this is churn/noise rather than an outage. Worth an owner cleanup so real problems aren't buried.
- **UnbalancedIRQs** node events on prod-cus and stage-cus (info only, no impact seen).

**Resolved since the previous run** (this is the first report; these cleared during it)
- stage-cus node `aks-ondemand-jsw99` not ready → now ready (4/4).
- stage-cus `cilium` FailedMount → gone.
- stage-cus `metrics-server` 1/2 available → **2/2**. This was the only *new* warning: the deployment was mid-rollout when the check ran (readiness probes returning 500 for ~1 min). It had already settled by the time I looked. No action needed.

**Housekeeping**
- **Kubernetes upgrade available on all five clusters**: 1.35.7 → 1.35.8 (patch) or 1.36.x (minor). Patch is the safe move; a minor bump deserves a plan. Your call when to schedule — nothing urgent.
- Cloudflare certificate expiry could not be checked (no token).

*No report from yesterday to compare against — this is the first run.*
