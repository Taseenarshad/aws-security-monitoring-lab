# AWS Security Monitoring Lab Plan

## Purpose and architecture

Build a small AWS lab for collecting audit events, detecting selected security-relevant activity, and sending notifications while keeping permissions and costs limited.

Planned event flow:

```text
CloudTrail management events
  ├── Private S3 bucket (audit archive; 30-day object expiration)
  └── CloudWatch Logs (7-day retention)
        └── Metric filters → CloudWatch alarms → SNS notifications
```

CloudTrail will use a dedicated IAM service role for CloudWatch Logs delivery. S3 delivery will use a separate bucket policy granting access to the CloudTrail service principal.

## Project and Region

- Inspected project ID: `374786579110`.
- Selected Region: **us-east-2 (Ohio)**.
- All Regional lab resources must remain in us-east-2. IAM roles are global resources.
- Local AWS profile configuration specifies us-east-2. Confirm the assigned Region in AWS Settings → View all projects → Overview → Additional Info → Region before deployment.
- Follow the repository's AGENTS.md instructions.

## Phase 1 status

Phase 1 was completed according to the user's prior work. The exact Phase 1 checklist was not present in this repository, so its individual deliverables have not all been independently verified.

Read-only inspection on October 22, 2026 confirmed:

- Access to the project above through a temporary assumed-role session.
- An active Free plan with $120 remaining credits at inspection time.
- No S3 buckets; no CloudWatch log groups or alarms in us-east-2.
- No CloudTrail trails, including shadow trails, or CloudTrail event data stores returned in us-east-2.
- No SNS topics, Lambda functions, EC2 instances, or CloudFormation stacks returned in us-east-2.
- Only the default EventBridge bus, with no rules returned.
- The planned IAM role `SecurityMonitoringLab-CloudTrailLogsRole` does not exist.
- No CloudWatch log groups returned in the supplementary us-east-1 inventory check.
- No project-level S3 public-access-block configuration returned. Bucket-level protection must be explicitly configured for the lab bucket.
- GuardDuty inspection was denied by a service control policy. GuardDuty is outside the validated Phase 2 scope and is not a prerequisite for it.

The absence of deployed monitoring resources is consistent with Phase 2 being preparation only; it does not establish that earlier resources were deleted.

## Budget safeguard

The user confirms that a **$2 monthly budget safeguard is already configured**. Its configuration and notification thresholds were not independently verified during this inspection.

A budget is a cost-monitoring safeguard, not a guaranteed hard spending cap. Check spend status in AWS Settings → Billing. Retention settings limit stored history but do not cap ingestion charges. No budget changes are part of this plan.

## Validated Phase 2: destinations and delivery permissions

Phase 2 is technically appropriate for the inspected environment, subject to explicit approval and final selection of the bucket name, future trail name, and optional S3 prefix. Read-only checks do not guarantee that subsequent creation operations will be authorized.

### 1. Private S3 audit bucket

- Create a dedicated general purpose bucket in us-east-2 with a unique available name.
- Explicitly enable all four bucket-level Block Public Access settings.
- Set Object Ownership to **Bucket owner enforced** with ACLs disabled.
- Configure default **SSE-S3** encryption; no customer-managed KMS key in this phase.
- Require HTTPS with a bucket-policy deny for insecure transport, covering the bucket and its objects.
- Permit CloudTrail delivery only for this project and the exact future trail ARN.
- Keep the bucket unversioned for this lab plan.
- Apply lifecycle expiration after 30 days to the audit-log prefix, or to all objects if the bucket is exclusively dedicated to these logs.

### 2. CloudWatch Logs destination

- Create `/aws/cloudtrail/security-monitoring-lab` in us-east-2.
- Set retention to **7 days**.
- Do not create streams manually; CloudTrail will create its streams when delivery is configured in a later phase.

### 3. CloudTrail delivery role

- Create `SecurityMonitoringLab-CloudTrailLogsRole` as a service role, not a role assigned to a team member.
- Trust only `cloudtrail.amazonaws.com` to assume the role.
- Restrict trust using the project ID and the exact future trail ARN through supported source conditions.
- Grant only `logs:CreateLogStream` and `logs:PutLogEvents` on this lab's CloudTrail stream prefix:

```text
arn:aws:logs:us-east-2:374786579110:log-group:/aws/cloudtrail/security-monitoring-lab:log-stream:374786579110_CloudTrail_us-east-2*
```

- Do not grant S3 permissions, log-group creation permissions, or broad CloudWatch Logs access to this role.

### Security corrections and clarifications

1. **Choose the future trail name before writing policies.** Construct its ARN as `arn:aws:cloudtrail:us-east-2:374786579110:trail/<trail-name>`. Policies may reference that ARN before the trail exists. The bucket name and optional delivery prefix also need to be settled before implementation.
2. **Separate S3 bucket permissions from object permissions.** Grant `cloudtrail.amazonaws.com` `s3:GetBucketAcl` on the bucket ARN and `s3:PutObject` only on `arn:aws:s3:::<bucket-name>/[optional-prefix/]AWSLogs/374786579110/*`. Omit the bracketed prefix segment when no prefix is used. Restrict both grants with the exact trail `aws:SourceArn`.
3. **Retain the documented upload ownership condition.** CloudTrail's `s3:x-amz-acl = bucket-owner-full-control` condition is compatible with Bucket owner enforced. Do not enable ACLs or grant `s3:PutObjectAcl` to accommodate it.
4. **Scope Logs permissions to a stream prefix, not one fixed stream.** The trailing wildcard accommodates CloudTrail's stream naming while limiting access to this project, Region, and log group.
5. **Treat lifecycle expiration as eventual deletion eligibility.** It is not an exact deletion deadline or immutable retention. If versioning is introduced later, review noncurrent-version expiration and delete-marker handling.
6. **Apply bucket-level public access protection explicitly.** Do not rely on a project-level configuration that was not found during inspection.
7. **Keep SSE-S3 and the agreed scope.** No KMS keys, trail creation, logging activation, metric filters, alarms, or SNS topics belong in Phase 2.

Phase 2 can validate resource configuration and policy scope, but cannot prove CloudTrail delivery end to end before the later trail phase.

## Planned later phases

These are planning placeholders, not approved implementation instructions. Confirm exact settings, detection patterns, thresholds, and recipients before each phase.

### CloudTrail phase

- Create a single-Region trail in us-east-2 using the agreed name.
- Configure the selected management-event scope and log file validation.
- Connect the prepared S3 bucket, CloudWatch log group, and delivery role.
- Start logging only after approval.
- Verify trail status, delivery health, and arrival of audit events in both destinations.
- Do not add cross-Region logging, organization trails, or paid data-event selectors without a separate scope review and approval.

### Metric filters and alarms phase

- Agree on security-relevant event patterns and expected test activity.
- Add CloudWatch Logs metric filters and corresponding CloudWatch alarms.
- Review thresholds, missing-data behavior, and cost implications.
- Validate detection with explicitly approved test activity.

### SNS notifications phase

- Agree on authorized recipients and notification configuration.
- Prepare an SNS topic and subscriptions, with narrowly scoped publication permissions.
- Connect approved alarms to the topic.
- Verify subscription confirmation and notification delivery using an approved test.

## Approval boundary

**AWS resources require explicit user approval before creation or modification.** This document grants no authorization to create, modify, enable, disable, or delete AWS resources. Destructive operations also require explicit approval.

Only this local planning document is authorized by the current request. Any later implementation must remain within the approved phase and must not silently add resources or expand permissions.

## References

- [CloudTrail S3 bucket policy](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/create-s3-bucket-policy-for-cloudtrail.html)
- [S3 Object Ownership and disabled ACLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)
- [CloudTrail CloudWatch Logs role policy](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-required-policy-for-cloudwatch-logs.html)
- [CloudTrail confused deputy prevention](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cross-service-confused-deputy-prevention.html)
- [Supported services for the new AWS experience](https://docs.aws.amazon.com/accounts/latest/reference/supported-services-sign-up-new.html)
