---
id: 5b50c963-25fb-424c-915e-999cd0a9a704
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
SRE report · 2026-09-25-ama-memory-review

# Azure Monitor AMA memory review · 2026-09-25

## Summary

The fresh hourly diff at 05:15Z reported `ama-logs-rs` OOMKilled restarts in dev-eus2 (5) and prod-eus2 (6), plus 5 OOMKilling node events in dev. Follow-up at 05:16Z showed no new findings: all five AKS clusters reachable, 26/26 nodes Ready, 0 unhealthy pods, and 0 deployments short. Stage-cus mount/network warnings occurred on newly provisioned nodes and cleared without any deployment remaining short.

The leading cause for AMA restarts is confirmed OOM at the container limit. The current agent image in dev, stage-eus2 and prod-eus2 is `mcr.microsoft.com/azuremonitor/containerinsights/ciprod:3.8.0-ci-prod-09-03-2026-fd42f68c`; `ama-logs-rs` has a 1536Mi limit and 250Mi request. The image's support/release status for this exact CI tag was not verified in Microsoft documentation.

## Read-only findings

- Agent's supported configuration ConfigMap is `kube-system/container-azm-ms-agentconfig`, schema v1: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-data-collection-configmap and template https://aka.ms/container-azm-ms-agentconfig.
- Supported collection/memory-relevant settings include stdout/stderr enablement and namespace exclusions, `agent_settings.resource_optimization`, documented Fluent Bit buffer/flush settings, metadata-cache TTL and chunk sizing. These may reduce collection work but require measurement; none guarantees a fix for these OOMs.
- The ConfigMap is absent from dev, stage-eus2, and prod-eus2. Pods mount it as optional. Applying the documented ConfigMap causes a rolling restart of AMA pods, so even a dev-only trial is consequential and must be approved.
- Microsoft's documented supported path to increase AMA DaemonSet resource limits is to contact Microsoft Support when defaults do not meet log-scale requirements: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-high-scale.
- ConfigMap does not expose pod resource requests/limits. Direct `kubectl` upgrade/delete/redeploy of the addon-managed agent is unsupported: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-manage-agent.
- Agent is auto-upgraded on AKS. Manual upgrade requires disabling/re-enabling monitoring; not proposed due to blast radius. Official docs: same manage-agent URL.
- Related docs: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-cost and https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-troubleshoot.

## Access limits and open questions

- Azure CLI sees stage and prod subscriptions, but inventory's dev subscription `53a006e0-623b-44f1-84ce-5889e8d8e8da` is absent from the logged-in account. Dev Azure-side addon profile/settings were therefore not checked. Kubernetes reads did confirm the dev pod image, ConfigMap absence, limits and restarts.
- Current configs in accessible stage/prod show omsagent addon enabled, workspace attached, metrics extension disabled. Container Insights `azureMonitorProfile` metrics disabled. No supported resource-limit field surfaced in these addon profiles.
- Need Microsoft Support to confirm the exact CI-tag status and increase-limit process for these clusters, then trend AMA memory vs log volume before choosing collection filters.

## Recommended next step

1. Open a Microsoft Support request asking if the observed CI image tag is supported/current and whether the AMA `ama-logs-rs` memory cap can be raised (current 1536Mi) for AKS addon-managed monitoring.
2. If a low-risk data-reduction test is still desired, separately review a dev-only `container-azm-ms-agentconfig` trial. That change causes an AMA rolling restart and must have a separately approved Cove card specifying exact settings, commands, blast radius, verification, and rollback. No such change is authorized by this research approval.
3. Do not directly patch addon-managed DaemonSets/Deployments. Do not apply or resubmit the rejected Karpenter `consolidateAfter 1m → 10m` change.

No infrastructure changes were made during this review.
