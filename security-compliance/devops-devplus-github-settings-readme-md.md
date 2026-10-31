---
id: 257ca48b-5f6a-4553-b396-95fd3f6acc7b
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevOps | devplus-github-settings/README.md

Source: DevPlusInc/devplus-github-settings/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-github-settings, README.md).

# devplus-github-settings: SOC2 controls for the DevPlusInc GitHub org

Terraform (OpenTofu) + documentation that keeps the `DevPlusInc` organization
(GitHub Enterprise Cloud) at SOC2 readiness.

- **`SOC2-CONTROLS.md`**: the requirements, mapped to GitHub settings and
  Trust Services Criteria, marking what Terraform enforces vs. manual steps.
- **`*.tf`**: Terraform enforcing the automatable controls.

## What the Terraform enforces

- **Org settings** (`main.tf`, `github_organization_settings`): base permission
  `read`; members cannot create public repos (private and internal allowed);
  no Pages; no forking of private repos; Advanced Security, secret scanning +
  push protection, Dependabot alerts + security updates and the dependency
  graph on for new repos.
- **Actions policy** (`github_actions_organization_permissions`): Actions on
  for all repos, `allowed_actions = "selected"`: GitHub-owned actions plus the
  explicit `allowed_action_patterns` list in `variables.tf` (verified creators
  are NOT allowed wholesale, `allow_verified_creators = false`). A new
  third-party action needs a PR to that list.
- **Org ruleset `soc2-default-branch-protection`** (enforcement `active`, the
  default branch of every repo):
  - no deletion, no force-push (`non_fast_forward`);
  - signed commits required;
  - linear history NOT required (merge commits are the org standard);
  - pull request: `required_approvals` approvals (2 in `terraform.tfvars`),
    code-owner review, stale reviews dismissed on push,
    `require_last_push_approval`, all review threads resolved;
  - required status checks only if `required_status_checks` is set (it is
    empty, so none are enforced);
  - bypass actors: `OrganizationAdmin` in `pull_request` mode, and the
    `devplus-cicd` GitHub App (id 3985657) in `always` mode for its
    Contents-API image-tag commits.
- **Teams** (`teams.tf`): the `Founders` team (closed), granted the predefined
  `all_repo_admin` org role. Both were adopted with `import` blocks. Team
  membership is not managed here.
- **CODEOWNERS** (`.github/CODEOWNERS`, this repo): `* @DevPlusInc/founders`.
  CODEOWNERS files are per repo and not managed here: as of 2026-10-04, 18 of
  the 33 repos carry the same line, while 15 (e.g. `devplus-cli`,
  `devplus-mcp`, `devplus-docs`, `terraform-cloudflare-ingress`) have none, so
  code-owner review alone does not force a Founders approval there.

In practice a PR to a default branch needs an approval from someone other than
the last pusher (a Founders member where CODEOWNERS applies), with every thread
resolved; any new push dismisses the approval.

## Live drift (checked 2026-10-04)

The live ruleset (id `17097067`, read with
`gh api repos/DevPlusInc/<repo>/rulesets/17097067`) was last edited on
2026-09-01, after the last commit here (2026-06-21), and no longer matches the
code:

| Setting | Code | Live |
|---|---|---|
| Signed commits | required | no `required_signatures` rule |
| Required approvals | 2 (`terraform.tfvars`) | 1 |
| Required reviewers | none | `Founders` team (id 17807541), 1 approval on `*` |
| Extra approval for unattributed changes | not set | on |
| Bypass actors | `OrganizationAdmin` (`pull_request`) + App 3985657 (`always`) | Apps 3985657 and 4067705 (`devplus-development`), both `always`; no human bypass (`current_user_can_bypass: never` for an org owner) |

The Actions allowlist has drifted too: `mikepenz/action-junit-report` runs in
`devplus-platform` but is not in `allowed_action_patterns`. A `tofu apply` from
`main` would push the code's version (signed commits back on, 2 approvals, the
admin bypass, App 4067705 and the junit action dropped), so reconcile the code
with the intended state before applying. The per-repo "Code Quality Copilot review" rulesets (disabled)
are not managed here.

## What is manual (see SOC2-CONTROLS.md)

2FA requirement, SAML SSO (enterprise), audit-log streaming, IP allow list,
default repo visibility, members' repo delete/transfer and visibility-change
rights, read-only default workflow token, team membership. None of these are
managed here.

## Environment: GitHub Enterprise Cloud with personal accounts (not EMU)

The org belongs to the `devplus` enterprise, but members sign in with personal
GitHub accounts (`hfeixas`, `aaronscofield`, `SaamerS`, all three org and
enterprise owners), so identity lives in GitHub: the org-level 2FA requirement
applies (live: on) and there is no SCIM auto-deprovisioning. No org-level SAML
provider is configured; SAML SSO, where enforced, is an enterprise setting. The
provider's enterprise resources are not used: there is no `enterprise.tf`, and
the `enterprise_slug` and `default_branch` variables are unused (the ruleset
targets `~DEFAULT_BRANCH`). See
`SOC2-CONTROLS.md → "Identity: what this means"`.

## Tooling

OpenTofu (`tofu`), `required_version >= 1.6.0`, provider
`integrations/github ~> 6.2`. `.terraform.lock.hcl` is gitignored, so
`tofu init` resolves the newest 6.x.

## Usage

There is no CI workflow: plan and apply run locally, by an org owner.

```sh
az login                                  # state: devplusstateprod/tfstate/github.tfstate (Entra ID auth)
gh auth refresh -h github.com -s admin:org
export GITHUB_TOKEN="$(gh auth token)"    # org-owner token with admin:org; owner comes from org_name

tofu init
tofu plan      # review the diff against the live org BEFORE applying
tofu apply
```

State lives in the azurerm backend (`backend.tf`): resource group
`devplus-tf-base-infra`, account `devplusstateprod`, container `tfstate`, key
`github.tfstate`, prod subscription. `terraform.tfvars` is committed (org slug
`DevPlusInc`, `common_org_name`, `billing_email`, `required_approvals = 2`)
despite its header comment; there is no `terraform.tfvars.example`.

> ⚠️ Review `tofu plan` carefully: the ruleset and base permission changes
> affect every member and repo immediately, and today's plan would also undo
> the live drift above.
