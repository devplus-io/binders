---
id: 5df5ee79-6284-4716-b0dd-c36ed5a27ea3
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevOps | devplus-github-settings/SOC2-CONTROLS.md

Source: DevPlusInc/devplus-github-settings/SOC2-CONTROLS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-github-settings, SOC2-CONTROLS.md).

# SOC2 Readiness: GitHub Organization Controls

> **Environment: GitHub Enterprise Cloud, `DevPlusInc` org in the `devplus`
> enterprise, personal accounts (NOT Enterprise Managed Users).**
> Identity controls (2FA, membership, offboarding) therefore live in GitHub
> itself, and their evidence comes from GitHub, not from an IdP. See
> [Identity](#identity-what-this-means) below.

SOC2 maps to the AICPA **Trust Services Criteria** (TSC). For a GitHub org the
relevant criteria are mostly under the **Common Criteria (CC)**. This document
maps each control to (a) the GitHub setting that satisfies it and (b) whether
Terraform in this repo enforces it or it must be done manually.

Legend: ✅ enforced by Terraform here · ⚙️ manual (org UI) · 🏢 enterprise-level setting · 📄 evidence to collect · ⚠️ live state differs from the code or the control is not in place (checked 2026-10-04 via read-only `gh api`)

| # | TSC | Control | GitHub setting | Status |
|---|-----|---------|----------------|--------|
| 1 | CC6.1 | Least-privilege base access | `default_repository_permission = read` | ✅ (all three current members are org owners, so this binds future members only) |
| 2 | CC6.1/6.2 | Strong authentication | Org requires 2FA | ⚙️ (live: on) / 🏢 SAML SSO if enforced at the enterprise (no org-level SAML provider) |
| 3 | CC6.2/6.3 | Privileged access in code | `Founders` team + predefined `all_repo_admin` org role (`teams.tf`) | ✅ (team membership itself is manual) |
| 4 | CC6.3 | Timely deprovisioning | Remove the user from the org and the `Founders` team (no SCIM) | ⚙️ + 📄 quarterly access review of org members and owners |
| 5 | CC6.6 | Restrict repo creation | No public repo creation by members (private/internal allowed) | ✅ |
| 6 | CC6.6/6.7 | Private by default | No Pages; default visibility private | ✅ (Pages) / ⚙️ (default visibility; live: 0 public repos) |
| 7 | CC6.7 | Prevent data exfiltration | Disallow forking private repos | ✅ |
| 8 | CC6.7 | Restrict destructive actions | Members cannot delete/transfer repos or change visibility | ⚙️ ⚠️ not in place: live `members_can_delete_repositories` and `members_can_change_repo_visibility` are `true` |
| 9 | CC7.1 | Integrity of changes | Require signed commits | ✅ (ruleset) ⚠️ live ruleset has no `required_signatures` rule |
| 10 | CC7.2 | Vulnerability detection | Secret scanning + push protection, Dependabot, dependency graph, Advanced Security | ✅ (new repos) / ⚙️ (backfill existing repos; code scanning not managed) |
| 11 | CC7.2/7.3 | Logging & monitoring | Audit log streaming to SIEM/storage | 🏢 enterprise audit log streaming + 📄 retention |
| 11b | CC6.6 | Network restriction | IP allow list | 🏢 enterprise policy (covers all orgs) |
| 12 | CC8.1 | Change management: review | Ruleset PR rule: `required_approvals` (2 in `terraform.tfvars`), CODEOWNERS (`* @DevPlusInc/founders`, per repo: 18 of 33 have it) + code-owner review, stale dismissal, last-push approval, thread resolution | ✅ (ruleset) ⚠️ live requires 1 approval plus a `Founders` required-reviewer rule |
| 13 | CC8.1 | Change management: CI gates | Required status checks before merge | ⚙️ supported (`required_status_checks`) but empty: no checks are required on any repo |
| 14 | CC8.1 | No bypassing controls | No force-push/delete; bypass limited to `OrganizationAdmin` (PR merges only) and the `devplus-cicd` App (id 3985657, Contents-API tag bumps) | ✅ (ruleset) ⚠️ live bypass = Apps 3985657 and 4067705, no humans |
| 15 | CC8.1 | Supply-chain hardening | Actions limited to GitHub-owned + the explicit `allowed_action_patterns` list (no blanket verified creators); read-only default `GITHUB_TOKEN` | ✅ (allowed actions) ⚠️ live allowlist also admits `mikepenz/action-junit-report` / ⚙️ (token default in Org → Actions → General) |
| 16 | CC1.x/2.x | Governance | Documented security policy, ownership (CODEOWNERS), onboarding | 📄 policy docs |

The live ruleset (`soc2-default-branch-protection`, id `17097067`) was last
edited 2026-09-01, after the last commit here, so a `tofu apply` from `main`
would revert the ⚠️ rows 9, 12, 14 and 15 to the code. Decide which side is
intended before applying (README → "Live drift").

## Identity: what this means {#identity-what-this-means}

DevPlusInc is NOT an EMU org: members use personal GitHub accounts, and GitHub
holds the identities. Practical consequences for SOC2:

- **Auth & MFA (CC6.1/6.2):** the org "Require 2FA" setting is the MFA
  control (live: on). Evidence is the org authentication settings page. SAML
  SSO, where enforced, is configured on the `devplus` enterprise.
- **Provisioning/deprovisioning (CC6.3):** there is no SCIM. Members are
  invited and removed by hand, so offboarding evidence is the org audit log
  plus a periodic access review. Privileged grants (`Founders` →
  `all_repo_admin`) are in code (`teams.tf`) and change only by reviewed PR;
  team membership is not.
- **Outside collaboration (CC6.1/6.6):** personal accounts can be added as
  outside collaborators (live: members may invite them; 0 today). Watch the
  collaborator list in access reviews.
- **Enterprise level:** audit-log streaming, the IP allow list and allowed
  repository visibilities are enterprise policies enforced down to the org.
- **Terraform reach:** the `integrations/github` provider is org-scoped. It
  has a few enterprise resources, but this repo uses none (no `enterprise.tf`;
  the `enterprise_slug` variable is unused). SSO/SAML, 2FA and audit-log
  streaming are NOT managed through the provider.

## Controls Terraform does not manage (do in the org or enterprise UI)

1. **2FA requirement** for the org (on today) and **SAML SSO** (enterprise). 🏢
2. **Audit log streaming**: enterprise-level; stream to Splunk/S3/Azure for
   retention evidence (SOC2 typically wants ~1 year). 🏢
3. **Allowed repository visibilities (no public)** and **IP allow list**:
   enterprise policies covering all orgs. 🏢
4. **Members can delete/transfer repos and change visibility = off**: Org →
   Member privileges (both on today, see row 8).
5. **Default `GITHUB_TOKEN` workflow permissions = read-only**: Org → Actions →
   General → Workflow permissions.
6. **Team membership**: `teams.tf` manages the `Founders` team and its role,
   not who is in it.

## Evidence to collect for the audit (📄)

- Screenshot/export of org auth settings showing 2FA required.
- Org membership + owner list and a sample quarterly access review (including
  outside collaborators); org audit-log entries for joins/removals.
- Exported ruleset config: this Terraform state reflects the last apply, so
  also export the live ruleset (`gh api repos/DevPlusInc/<repo>/rulesets/17097067`)
  while it differs from the code.
- Audit-log stream destination + retention policy.
- List of allowed GitHub Actions (`allowed_action_patterns`, plus any live
  additions) and justification for the third-party ones.
- Security policy doc, incident response plan, onboarding/offboarding checklist.

## Backfilling existing repos

`github_organization_settings` security flags apply to **new** repos only.
For existing repos, enable secret scanning / Dependabot per-repo (loop
`github_repository` data + `github_repository_dependabot_security_updates`,
or run a one-time script). Track this as a remediation item.
