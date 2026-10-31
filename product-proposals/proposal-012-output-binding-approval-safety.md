---
id: 7f284d93-3823-4b06-95da-05c9906194ad
binder: binder-mvvle32a-1
folder: folder-mvw0u8o0
---
Proposal 012: output-binding approval safety

FILING DECLINED. Helder rejected the filing card 76842140-112E-496A-944E-2A4CC0561707; disposition reconciled 26 October 2026. Historical addendum only, not an outstanding approval request. No code or issue created. Do not re-pitch or routinely revise without new owner direction. The non-sensitive-first scope was proposed, not adopted, and Saamer's original feature note is unchanged. Source observations below retain their 25 October verification date. This is an addendum to Cross-Deployment Output Bindings (9EC3476F-D491-4DEB-9BA8-045EFF63A13D), not a duplicate of its whole feature or an edit to Saamer's note.

The existing feature already calls for outputsVersion in the approval hash. The addendum specifies an authoritative state identity returned atomically with its bytes, a per-run resolved reference snapshot, invalidation of old approvals after upstream revision changes, and consistent inputs for plan and saved-plan apply. Coordinate revision identity with the planned state-history feature (98C92623-B961-4171-A05F-CEA267A42BCE) rather than introduce a second history store.

Proposed scope decision: non-sensitive outputs first. Same-project deployment-write permission is not an established sensitive-consumption authorization contract. TF_VAR delivery does not guarantee that provider state or plan artifacts omit secret values. Sensitive consumption should remain disabled until delegation and artifact protection are designed. This recommendation was declined for filing. It is not awaiting a founder decision and is not an accepted release restriction.

Evidence: shared fdf993d models/environments.go:303-334; executor 74743d2 services/deploy_approval.go:62-89 and services/opentofu_executor.go:132-137,188-192; opentofu-server c9b10d6 services/state_service.go:63-75,99-115 and handlers/state_handler.go:23-46; backend b0e3a3c handlers/deployment_output_handler.go:63-95. Running tenants and unpushed branches not inspected.

Canonical exact proposed issue text: ~/Documents/DevPlus Architecture/proposals/012-output-binding-approval-safety.md. Includes API/UI changes, service ownership, rollout, acceptance tests and an incremental estimate of 4 to 6 days after binding validation and state revision prerequisites. No code or issue created. Writing check passed with no findings. Current conversion counts unchanged.
