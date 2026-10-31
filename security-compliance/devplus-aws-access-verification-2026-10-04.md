---
id: 0f71aa59-667f-4d99-aa48-c3a0f1477dee
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevPlus AWS access verification — 2026-10-04
Read-only checks completed at 04:27 EDT on 2026-10-04.
Acceptance: authenticate into Console Home for each of the four supplied DevPlus account IDs, verify identity/role, report browser versus CLI and MFA blockers, and make no configuration changes.
Method: recovered targeted prior setup conversation 35483971; searched DevPlus context and local AWS config for allowlisted SSO profiles. No DevPlus CLI SSO profiles were found. Verified DevPlus Entra identity helder@devplus.io in https://myapps.microsoft.com/ and launched DevPlus AWS IAM Identity Center.
Portal: https://ssoins-668408f10ecd2d51.portal.us-east-2.app.aws/#/
Entra launch: https://launcher.myapps.microsoft.com/api/signin/0dbc5a95-5f85-4e89-932c-96be747a5ebb?tenantId=002e69ff-b9d4-424e-a5a0-9d81b41114fb
Actual authenticated Console Home access verified for management 002512087693, dev 971671216986, stage 029533636608, prod 174765207070. All showed AdministratorAccess/helder@devplus.io. No new MFA challenge; existing Entra session sufficed. This is Helder's DevPlus federated session, not an independent agent identity.
Single-session limitation: switching accounts signs out older console tabs. Concurrent switches produced collisions; management and stage were reverified sequentially after prod. Each account was accessible, not simultaneously signed in.
Stage naming discrepancy: portal lists 029533636608 as DevPlus Stage / aws-stage@devplus.io, but Console Home and its account menu show Account name DevPlus Prod, Account ID 0295-3363-6608, Federated user AWSReservedSSO_AdministratorAccess_77273b81b2a253dd/helder@devplus.io. Do not infer environments from display names. No rename performed.
AWS CLI installed but access not verified or configured. No credentials retrieved or exposed. Personal AWS account was not accessed or inspected. Warp remained Connected. No billing, resources, production, identity or permission changes.
Sources: live DevPlus Entra My Apps, AWS access portal, authenticated Console Home pages and stage account identity menu.
