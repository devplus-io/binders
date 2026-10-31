---
id: 354ffd6a-2653-4e6c-be1b-51315ce49aa0
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
DevPlus AWS guardrails 2026-10-04

## policies/DevPlus-IdentityGuard.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "IdentityCenterAndRolesOnlyNoLongLivedIamIdentities",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateLoginProfile",
        "iam:UpdateLoginProfile",
        "iam:CreateAccessKey"
      ],
      "Resource": "*"
    }
  ]
}

## policies/DevPlus-OrgGuard.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeavingOrClosingTheOrganization",
      "Effect": "Deny",
      "Action": [
        "organizations:LeaveOrganization",
        "organizations:DeleteOrganization",
        "organizations:RemoveAccountFromOrganization",
        "organizations:DeregisterDelegatedAdministrator",
        "account:CloseAccount",
        "account:DisableRegion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisablingAuditTrails",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:DeleteEventDataStore",
        "cloudtrail:UpdateEventDataStore"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisablingThreatDetection",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DeleteMembers",
        "guardduty:DisassociateFromAdministratorAccount",
        "guardduty:DisassociateFromMasterAccount",
        "securityhub:DisableSecurityHub",
        "securityhub:DeleteMembers",
        "access-analyzer:DeleteAnalyzer"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisablingConfig",
      "Effect": "Deny",
      "Action": [
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder",
        "config:DeleteDeliveryChannel"
      ],
      "Resource": "*"
    },
    {
      "Sid": "NeverCreateRootUserAccessKeys",
      "Effect": "Deny",
      "Action": "iam:CreateAccessKey",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
