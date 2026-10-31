---
id: b757bb98-b76b-464b-83ac-6bebd83466b9
binder: binder-mvvle32a-1
---
SRE report · 2026-10-27

Source: ~/Documents/DevPlus Ops/reports/2026-10-27.md (full report, evidence in evidence/2026-10-27/).

Verdict: Healthy with three things to watch. Full health check with --diff at 08:01 EDT, read-only verification to 08:26 EDT. No infrastructure, network or cluster setting changed.

Findings: 0 critical, 5 warning, 16 informational. All five clusters reachable again after yesterday's total API-access failure: 28/28 nodes Ready, 0 unhealthy pods, 224 deployments at desired replicas. Yesterday's five "Cluster API unreachable" findings cleared.

Cloudflare and Atlas checked for the first time since 30 September, through the documented Key Vault fallback (kv-devplus-dev) rather than Pennant Vault: zone active, 17/17 tunnels healthy, earliest certificate expiry 12 December 2026 (*.tools 13 December, *.stage and *.dev 4 January 2027), so the inventory's "earliest expiry 5 November" note is stale. Atlas dev, prod and stage all IDLE, unpaused, 8.0.34, 0 open alerts. Cloudflare 5xx still unverified: that token lacks Zone Analytics Read.

Root causes of the five dev warnings: Karpenter spot node churn on a one-minute consolidation policy, with three nodes cycling during the run, and node memory pressure evicting Defender publisher pods. ama-logs-rs OOM-killed on all five clusters at the same 1536Mi limit (prod eus2 8 restarts, stage eus2 6, prod cus 3, stage cus 1, dev 1); restarts are quick and telemetry resumes. Stage eus2 aks-general-5rcdl at 91-92% memory with 98% of allocatable requested; dev aks-general-fh6dh at 91% with 89% requested. The FailedMount warnings are optional configMap/secret mounts (container-azm-ms-agentconfig and the two secrets exist on none of the clusters) and self-resolve.

No fix applied. Open items unchanged: Vault credentials, telemetry OOM limit, dev Defender placement. New decision for the founder: when to schedule the 1.37.x upgrade, since all five clusters still run 1.36.4.
