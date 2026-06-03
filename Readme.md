# AWS_Discovery_Filtered – Infoblox Discovery Role with Account Filtering

This repository provides a CloudFormation template to deploy the **`infoblox_discovery`**
IAM role across your AWS Organization using a **SERVICE_MANAGED StackSet** with
support for account-level filtering.

This version includes:

- Least-privilege inline policy scoped to IPAM/network/asset discovery
- EC2, VPC, IPAM, S3, Route53, Route53Resolver, ELB, DirectConnect, CloudWatch discovery permissions
- Account filtering: deploy to specific accounts (INTERSECTION) or exclude accounts (DIFFERENCE)
- Input validation for OU IDs and Account IDs
- Delegated administrator support (`CallAs` parameter)

---

## Security Model

The role:

- Uses cross-account trust with **ExternalId enforcement**
- Restricts assumption to a specific AWS account
- Is deployed consistently via CloudFormation StackSet
- Uses least-privilege inline policy (not broad ReadOnlyAccess)
- Should NOT be manually modified in target accounts

---

## Org-Wide Deployment

Deploy across your AWS Organization using a CloudFormation **StackSet**.

### Steps

1. Log in to your **AWS Organizations Management Account** (or a registered delegated admin account).
2. Ensure **CloudFormation StackSets trusted access** is enabled.
3. Click the button below.
4. Enter your target OU IDs (or root ID `r-xxxx`).
5. Set `CallAs` to `SELF` (management account) or `DELEGATED_ADMIN` (delegated admin account).
6. Choose **one region** (IAM is global).
7. Optionally configure account filtering.
8. Create the stack.

[![Deploy Org-Wide][deploy-org-badge]][deploy-org-link]

---

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ExternalId` | Yes | External ID required for Infoblox role assumption |
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

---

## Permissions Included

### Inline Policy: `InfobloxIPAMDiscovery`

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

---

## Important Notes

- Requires `CAPABILITY_NAMED_IAM`
- IAM is global per account — select only one region
- `TargetOUsCsv` is **required** — SERVICE_MANAGED StackSets do not support account-only targeting
- If the `infoblox_discovery` role already exists in a target account, the deployment will fail for that account. Delete the existing role first or use `DIFFERENCE` mode to exclude it
- Do not manually edit the role in member accounts — update via StackSet only
- **The management (root) account is never a deployment target** — this is an AWS platform rule for SERVICE_MANAGED StackSets. If the management account needs the role, deploy it as a standalone stack directly in that account
- **Delegated admin:** set `CallAs: DELEGATED_ADMIN` when deploying from a member account registered as a StackSets delegated administrator. The member account must first be registered from the management account: `aws organizations register-delegated-administrator --service-principal=member.org.stacksets.cloudformation.amazonaws.com --account-id=<member-account-id>`

---

## Architecture Overview

```
Management Account
  -> CloudFormation Stack (Bootstrap)
    -> CloudFormation StackSet (SERVICE_MANAGED)
      -> Target OUs (with optional account filtering)
        -> IAM Role: infoblox_discovery
```

Cross-account assumption secured by:
- ExternalId
- Explicit Principal restriction

---

## Related

- [AWS-Discovery-Org-Write](https://github.com/iracic82/AWS-Discovery-Org-Write) — Simpler OU-only version with Route53 write permissions and ReadOnlyAccess managed policy

[deploy-org-badge]: https://img.shields.io/badge/Deploy%20Org--Wide%20(StackSet)-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white
[deploy-org-link]: https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Finfoblox-igor.s3.eu-west-1.amazonaws.com%2Finfoblox_discovery_stackset_bootstrap_filtered.yaml&stackName=Infoblox-Discovery-Role-Filtered&param_ExternalId=f90ae844-4072-47a5-a6a3-4f900e8317df&param_AccountId=902917483333&param_TargetOUsCsv=%3CREPLACE_WITH_YOUR_OU_IDS%3E&param_AccountFilterMode=NONE&param_Regions=us-east-1&param_AutoDeployNewAccounts=true&param_CallAs=SELF
