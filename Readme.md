# AWS_Discovery_Filtered – Infoblox Discovery Role with Account Filtering

This repository provides CloudFormation templates to deploy the **`infoblox_discovery`**
IAM role across your AWS Organization using a **SERVICE_MANAGED StackSet** with
support for account-level filtering.

Two variants are available — deploy **one or the other**, not both:

| Template | Use case |
|----------|----------|
| **Read-Only** (recommended) | Discovery only — no changes to DNS |
| **Read + Route53 Write** | Discovery + Infoblox manages Route53 DNS records |

> **Upgrading from Read-Only to Route53 Write?** No need to delete anything.
> Simply update your existing bootstrap stack with the R53Write template —
> CloudFormation pushes the expanded policy to all member accounts automatically.

This version includes:

- Least-privilege inline policy scoped to asset and network discovery
- EC2, VPC, IPAM, S3, Route53, Route53Resolver, ELB, DirectConnect, CloudWatch permissions
- Account filtering: deploy to specific accounts (INTERSECTION) or exclude accounts (DIFFERENCE)
- Input validation for OU IDs and Account IDs
- Delegated administrator support (`CallAs` parameter)
- Concurrent deployment across accounts (`ManagedExecution`, `MaxConcurrentPercentage: 100`)

---

## Prerequisites

Before deploying, confirm the following in your **AWS Organizations management account**:

1. **All features enabled** — AWS Organizations must run in *All features* mode (not Consolidated Billing only).
   - Console: AWS Organizations → Settings → *All features*
   - Without this, SERVICE_MANAGED StackSets cannot be created.

2. **Trusted access activated** — authorizes CloudFormation StackSets to manage roles across the org.
   - Console: CloudFormation → StackSets → *Enable trusted access* (banner on first visit)
   - CLI: `aws organizations enable-aws-service-access --service-principal stacksets.cloudformation.amazonaws.com`
   - Only needs to be done once per organization.

3. **Deployer has sufficient permissions** — the IAM principal creating the bootstrap stack needs:
   - `cloudformation:*` (stack and StackSet operations)
   - `organizations:DescribeOrganization`, `organizations:ListRoots`, `organizations:ListOrganizationalUnitsForParent`, `organizations:ListAccountsForParent`
   - `s3:GetObject` on the template bucket (if using an S3 URL)
   - Note: IAM operations in member accounts (`CreateRole`, `PutRolePolicy`) are performed by the auto-created `stacksets-exec-*` execution role, not by the deployer directly.

---

## Security Model

The role:

- Uses cross-account trust with **ExternalId enforcement**
- Restricts assumption to a specific Infoblox AWS account
- Is deployed consistently via CloudFormation StackSet (no manual per-account steps)
- Uses least-privilege inline policy (not broad ReadOnlyAccess)
- Should NOT be manually modified in target accounts — update via StackSet only

---

## Org-Wide Deployment

### Option A — Read-Only Discovery (Recommended)

Deploy across your AWS Organization using a CloudFormation **StackSet**.

#### Steps

1. Log in to your **AWS Organizations Management Account** (or a registered delegated admin account).
2. Ensure prerequisites above are met.
3. Click the button below.
4. Override **`ExternalId`** with your tenant's actual External ID from the Infoblox portal.
5. Enter your target OU IDs (or root ID `r-xxxx`) in **`TargetOUsCsv`**.
6. Set `CallAs` to `SELF` (management account) or `DELEGATED_ADMIN` (delegated admin account).
7. Choose **one region** (IAM is global — `us-east-1` is sufficient).
8. Optionally configure account filtering (`AccountFilterMode` and `AccountFilterCsv`).
9. Click **Create stack**.

[![Deploy Read-Only][deploy-ro-badge]][deploy-ro-link]

> **Already deployed?** Do **not** click the button again — it will fail with `AlreadyExistsException`.
> To update permissions on an existing deployment, go to **CloudFormation → Stacks → `Infoblox-Discovery-Role-Filtered` → Update → Replace current template** and paste the S3 URL above.
> See [Updating an Existing StackSet](#updating-an-existing-stackset) for full instructions.

---

### Option B — Read + Route53 Write

Use this variant when Infoblox needs to **create, update, or delete Route53 DNS records**
(e.g. for NIOS-X DNS management). Includes all discovery permissions from Option A plus:

- `route53:ChangeResourceRecordSets` — create/update/delete DNS records
- `route53:CreateHostedZone` / `route53:DeleteHostedZone` — manage zones
- `route53:AssociateVPCWithHostedZone` / `route53:DisassociateVPCFromHostedZone` — private zones
- `route53:UpdateHostedZoneComment`

This variant uses the **same StackSet name** (`Infoblox-Discovery-Role`) as Option A.
**Do not deploy both** — use one or the other.

**Deploying fresh:** use the button below.

**Upgrading from Option A:** update your existing bootstrap stack in CloudFormation
(*Update → Replace current template → use the R53Write S3 URL*). CloudFormation will
push the expanded policy to all member accounts — no deletion, no re-onboarding.

[![Deploy Read + Route53 Write][deploy-r53w-badge]][deploy-r53w-link]

> **Already deployed Option A?** Do **not** click this button — use **Update stack** on your existing `Infoblox-Discovery-Role-Filtered` stack with this template URL instead.
> See [Updating an Existing StackSet](#updating-an-existing-stackset) for full instructions.

---

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ExternalId` | Yes | External ID required for Infoblox role assumption — **override with your tenant's value** |
| `AccountId` | Yes | Infoblox AWS account ID allowed to assume the role |
| `TargetOUsCsv` | Yes | Comma-separated OU IDs or root ID (e.g. `ou-abcd-12345678` or `r-xxxx`) |
| `AccountFilterMode` | No | `NONE` (default), `INTERSECTION`, or `DIFFERENCE` |
| `AccountFilterCsv` | Conditional | Comma-separated 12-digit Account IDs. Required when filter mode is not NONE |
| `Regions` | No | AWS region(s) for deployment. Default: `us-east-1` |
| `AutoDeployNewAccounts` | No | Auto-deploy to new accounts joining target OUs. Default: `true` |
| `CallAs` | No | `SELF` (default, management account) or `DELEGATED_ADMIN` (registered delegated admin member account) |

---

## Account Filtering

| Mode | Behavior |
|------|----------|
| `NONE` | Deploy to **all** accounts in the specified OUs |
| `INTERSECTION` | Deploy **only** to accounts listed in `AccountFilterCsv` within the OUs |
| `DIFFERENCE` | Deploy to **all** accounts in the OUs **except** those listed in `AccountFilterCsv` |

### Examples

**Deploy to all accounts in an OU:**
```
TargetOUsCsv:      ou-abcd-12345678
AccountFilterMode: NONE
```

**Deploy only to specific accounts within an OU:**
```
TargetOUsCsv:      ou-abcd-12345678
AccountFilterMode: INTERSECTION
AccountFilterCsv:  111111111111,222222222222
```

**Deploy to all accounts except certain ones:**
```
TargetOUsCsv:      ou-abcd-12345678
AccountFilterMode: DIFFERENCE
AccountFilterCsv:  333333333333,444444444444
```

**Use org root to scope across all OUs (apply INTERSECTION for specific accounts):**
```
TargetOUsCsv:      r-xxxx
AccountFilterMode: INTERSECTION
AccountFilterCsv:  111111111111,222222222222
```

---

## Permissions Included

### Inline Policy: `InfobloxAssetDiscovery`

Both variants use the same policy name. The R53 Write variant adds the `Route53Write` Sid on top of all read-only statements below.

> This permission set matches the official Infoblox documentation exactly:
> [AWS Least Privilege IAM Permissions](https://docs.infoblox.com/space/BloxOneDDI/2358182069/AWS+Least+Privilege+IAM+Permissions) — 68 read-only actions across 10 Sid groups, maintained by Infoblox engineering.

| Service | Permissions |
|---------|------------|
| EC2 | `DescribeRegions`, `DescribeVpcs`, `DescribeVpcAttribute`, `DescribeSubnets`, `DescribeRouteTables`, `DescribeAvailabilityZones`, `DescribeManagedPrefixLists`, `DescribeInternetGateways`, `DescribeEgressOnlyInternetGateways`, `DescribeNatGateways`, `DescribeTransitGateways`, `DescribeTransitGatewayVpcAttachments`, `DescribeTransitGatewayPeeringAttachments`, `DescribeAddresses`, `DescribeNetworkAcls`, `DescribeNetworkInterfaces`, `DescribeVpcEndpoints`, `DescribeVpcPeeringConnections`, `DescribeVpnConnections`, `DescribeVpnGateways`, `DescribeCustomerGateways`, `DescribeInstances`, `DescribeSecurityGroups`, `DescribeSecurityGroupRules`, `DescribeSecurityGroupReferences`, `DescribeStaleSecurityGroups`, `DescribeVolumes`, `DescribeVolumeStatus`, `DescribeSnapshots`, `DescribeSnapshotAttribute` |
| IPAM | `DescribeIpams`, `DescribeIpamScopes`, `DescribeIpamPools`, `DescribeIpamResourceDiscoveries`, `GetIpamPoolAllocations`, `GetIpamPoolCidrs`, `GetIpamResourceCidrs` |
| Route53 | `GetHostedZone`, `ListHostedZones`, `ListResourceRecordSets`, `ListTagsForResources`, `ListQueryLoggingConfigs`, `GetHealthCheck`, `ListHealthChecks` |
| Route53Resolver | `ListResolverEndpoints`, `ListResolverEndpointIpAddresses`, `ListResolverRules`, `ListResolverRuleAssociations` |
| ELB | `DescribeLoadBalancers`, `DescribeLoadBalancerAttributes`, `DescribeListeners`, `DescribeRules`, `DescribeTargetGroups`, `DescribeTargetHealth` |
| S3 | `ListAllMyBuckets`, `GetBucketLocation`, `GetBucketWebsite`, `GetBucketPublicAccessBlock`, `GetBucketAcl`, `GetBucketPolicy`, `GetBucketPolicyStatus` |
| DirectConnect | `DescribeDirectConnectGateways`, `DescribeDirectConnectGatewayAttachments`, `DescribeConnections`, `DescribeVirtualInterfaces` |
| CloudWatch | `ListMetrics`, `GetMetricStatistics`, `GetMetricData` |

### Additional Sid: `Route53Write` (R53 Write variant only)

| Service | Permissions |
|---------|------------|
| Route53 | `ChangeResourceRecordSets`, `CreateHostedZone`, `DeleteHostedZone`, `AssociateVPCWithHostedZone`, `DisassociateVPCFromHostedZone`, `UpdateHostedZoneComment` |

---

## Updating an Existing StackSet

You do **not** need to delete and recreate the StackSet to update permissions.
Update the bootstrap stack directly — CloudFormation propagates the change to all member accounts automatically.

> **Important:** Always update via the bootstrap stack (`CloudFormation → Stacks`), **not** directly through the StackSets console or CLI `update-stack-instances`. Editing the StackSet directly causes drift and bypasses CloudFormation's state tracking for the bootstrap stack.

**Via Console:**
1. CloudFormation → Stacks → select the bootstrap stack (e.g. `Infoblox-Discovery-Role-Filtered`)
2. **Update** → Replace current template → upload the new template or use the S3 URL
3. Review parameter changes → Submit
4. CloudFormation updates the StackSet and pushes the new role policy to all instances

**Via CLI (keep existing parameters, update template only):**
```bash
# Update to read-only template (permission refresh)
aws cloudformation update-stack \
  --stack-name Infoblox-Discovery-Role-Filtered \
  --template-url https://infoblox-igor.s3.eu-west-1.amazonaws.com/infoblox_discovery_stackset_bootstrap_filtered.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=ExternalId,UsePreviousValue=true \
    ParameterKey=AccountId,UsePreviousValue=true \
    ParameterKey=TargetOUsCsv,UsePreviousValue=true \
    ParameterKey=AccountFilterMode,UsePreviousValue=true \
    ParameterKey=AccountFilterCsv,UsePreviousValue=true \
    ParameterKey=Regions,UsePreviousValue=true \
    ParameterKey=AutoDeployNewAccounts,UsePreviousValue=true \
    ParameterKey=CallAs,UsePreviousValue=true

# Upgrade to Route53 Write (replace template URL only)
aws cloudformation update-stack \
  --stack-name Infoblox-Discovery-Role-Filtered \
  --template-url https://infoblox-igor.s3.eu-west-1.amazonaws.com/infoblox_discovery_stackset_bootstrap_filtered_r53write.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=ExternalId,UsePreviousValue=true \
    ParameterKey=AccountId,UsePreviousValue=true \
    ParameterKey=TargetOUsCsv,UsePreviousValue=true \
    ParameterKey=AccountFilterMode,UsePreviousValue=true \
    ParameterKey=AccountFilterCsv,UsePreviousValue=true \
    ParameterKey=Regions,UsePreviousValue=true \
    ParameterKey=AutoDeployNewAccounts,UsePreviousValue=true \
    ParameterKey=CallAs,UsePreviousValue=true
```

---

## Checking Deployment Status

After deploying or updating, use these commands to see which accounts succeeded or failed.

**All instances and their status:**
```bash
aws cloudformation list-stack-instances \
  --stack-set-name Infoblox-Discovery-Role \
  --region eu-west-1 \
  --query "Summaries[*].{Account:Account,Region:Region,Status:Status,Reason:StatusReason}" \
  --output table
```

Instance `Status` values:
| Status | Meaning |
|--------|---------|
| `CURRENT` | Role deployed successfully and up to date |
| `OUTDATED` | Last operation failed or account not yet updated |
| `INOPERABLE` | Account is suspended or otherwise unreachable |

**Filter to failed accounts only:**
```bash
aws cloudformation list-stack-instances \
  --stack-set-name Infoblox-Discovery-Role \
  --filters Name=DETAILED_STATUS,Values=FAILED \
  --region eu-west-1 \
  --query "Summaries[*].{Account:Account,Region:Region,Status:Status,Reason:StatusReason}" \
  --output table
```

This shows the account ID and the exact failure reason for each failed instance — useful for identifying which accounts need attention and why.

---

## Delegated Administrator Setup

If the deploying account is **not** the management account, it must be registered as a
CloudFormation StackSets delegated administrator. This is a one-time setup done from
the management account:

```bash
# Register the member account as delegated admin (run from management account)
aws organizations register-delegated-administrator \
  --service-principal member.org.stacksets.cloudformation.amazonaws.com \
  --account-id <member-account-id>

# Verify registration
aws organizations list-delegated-administrators \
  --service-principal member.org.stacksets.cloudformation.amazonaws.com
```

Then deploy with `CallAs: DELEGATED_ADMIN`.

---

## Important Notes

- Requires `CAPABILITY_NAMED_IAM`
- IAM is global per account — select only one region (`us-east-1` is sufficient)
- `TargetOUsCsv` is **required** — SERVICE_MANAGED StackSets do not support account-only targeting. Use `AccountFilterMode: INTERSECTION` with `AccountFilterCsv` to target specific accounts within an OU
- **ExternalId must match your tenant** — the default value is a placeholder. Always override with the External ID from your Infoblox portal. A mismatch lets the StackSet deploy successfully but causes silent `AccessDenied` on every `AssumeRole` attempt by Infoblox
- **The management (root) account is never a deployment target** — this is a hard AWS platform rule for SERVICE_MANAGED StackSets (not a Control Tower or config setting). To deploy the role to the management account, run a separate standalone CloudFormation stack directly in that account
- Do not manually edit the role in member accounts — update via StackSet only

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `OrganizationalUnitIds should be specified in DeploymentTargets in [SERVICE_MANAGED] model` | `TargetOUsCsv` was left blank | Provide at least one OU ID or the org root ID (`r-xxxx`) |
| `Stack [Infoblox-Discovery-Role-Filtered] already exists` | The deploy button was clicked again on an already-deployed stack — the button only creates, never updates | Do not redeploy. Go to **CloudFormation → Stacks → `Infoblox-Discovery-Role-Filtered` → Update → Replace current template** to apply changes to an existing deployment |
| `Resource … 'Infoblox-Discovery-Role' already exists` | A previous deployment left an orphan StackSet | Delete the existing StackSet (and its instances) from the CloudFormation StackSets console, then redeploy |
| `AccessDenied` on `sts:AssumeRole` (after successful deploy) | `ExternalId` in the role doesn't match what Infoblox presents | Update the bootstrap stack with the correct `ExternalId` |
| Role missing in management account | AWS never deploys SERVICE_MANAGED stacks to the management account | Deploy a standalone stack in the management account |
| `Trusted access is not enabled` | CloudFormation ↔ Organizations trusted access not activated | Enable from CloudFormation → StackSets console (management account only) |
| `You must be the management account or delegated admin account` on a specific stack instance (not on the bootstrap stack) | A target member account independently deployed the bootstrap template as its own local CloudFormation stack, creating a nested `AWS::CloudFormation::StackSet` resource. Member accounts cannot operate SERVICE_MANAGED StackSets | Log into that member account and delete its local bootstrap stack (`aws cloudformation delete-stack --stack-name <bootstrap-stack-name>`), then retry the StackSet operation from the management account |

---

## Architecture Overview

```
Management Account
  └─ CloudFormation Stack (Bootstrap)
       └─ CloudFormation StackSet (SERVICE_MANAGED)
            └─ Target OUs (with optional account filtering)
                 └─ IAM Role: infoblox_discovery
                      └─ Trust: Infoblox account + ExternalId enforcement
```

Note: The management account itself is excluded from StackSet deployment by AWS design.

[deploy-ro-badge]: https://img.shields.io/badge/Deploy%20Read--Only%20(StackSet)-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white
[deploy-ro-link]: https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Finfoblox-igor.s3.eu-west-1.amazonaws.com%2Finfoblox_discovery_stackset_bootstrap_filtered.yaml&stackName=Infoblox-Discovery-Role-Filtered&param_ExternalId=f90ae844-4072-47a5-a6a3-4f900e8317df&param_AccountId=902917483333&param_TargetOUsCsv=%3CREPLACE_WITH_YOUR_OU_IDS%3E&param_AccountFilterMode=NONE&param_Regions=us-east-1&param_AutoDeployNewAccounts=true&param_CallAs=SELF

[deploy-r53w-badge]: https://img.shields.io/badge/Deploy%20Read%20%2B%20Route53%20Write%20(StackSet)-FF4500?style=for-the-badge&logo=amazon-aws&logoColor=white
[deploy-r53w-link]: https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Finfoblox-igor.s3.eu-west-1.amazonaws.com%2Finfoblox_discovery_stackset_bootstrap_filtered_r53write.yaml&stackName=Infoblox-Discovery-Role-Filtered&param_ExternalId=f90ae844-4072-47a5-a6a3-4f900e8317df&param_AccountId=902917483333&param_TargetOUsCsv=%3CREPLACE_WITH_YOUR_OU_IDS%3E&param_AccountFilterMode=NONE&param_Regions=us-east-1&param_AutoDeployNewAccounts=true&param_CallAs=SELF
