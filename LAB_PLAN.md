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

## Phase 3 implementation results — complete and verified

Phase 3 was implemented with explicit user approval and verified on October 22, 2026. This section records those verification results; updating this document did not make any AWS API calls or change AWS resources.

### Trail and event configuration

- Trail name: `security-monitoring-lab`.
- Trail ARN: `arn:aws:cloudtrail:us-east-2:374786579110:trail/security-monitoring-lab`.
- Region: **us-east-2 (Ohio)**.
- Single-Region trail: `IsMultiRegionTrail=false`.
- Management events: `IncludeManagementEvents=true`, `ReadWriteType=All` (read and write), with no management event-source exclusions.
- Log file integrity validation: **enabled**, confirmed by `LogFileValidationEnabled=true`. Digest generation is enabled; cryptographic validation of a delivered digest chain was not performed during this verification.
- Logging: **active**, confirmed by `IsLogging=true`; started at **2026-10-22 05:57:58 UTC**.

### Delivery verification

- **CloudWatch Logs delivery confirmed** by actual management events in `/aws/cloudtrail/security-monitoring-lab`, including `StartLogging` and `GetTrailStatus`.
- CloudTrail used the existing `SecurityMonitoringLab-CloudTrailLogsRole` and automatically created streams matching `374786579110_CloudTrail_us-east-2*`.
- The first observed successful CloudWatch delivery was at **06:00:06 UTC**; another delivery was reported at **06:02:17 UTC**. The existing log group retains events for **7 days**.
- **S3 delivery confirmed** to the existing bucket `security-monitoring-lab-374786579110-us-east-2`, with no optional S3 prefix.
- CloudTrail reported successful S3 delivery at **06:02:47 UTC**. A nonempty compressed audit file was present under `AWSLogs/374786579110/CloudTrail/us-east-2/2026/10/22/` with a size of **5,613 bytes** and last-modified time **06:02:48 UTC**.
- An object metadata read confirmed **SSE-S3 encryption** (`ServerSideEncryption=AES256`) on that delivered audit file.
- The delivered object also carried an expiration header referencing `ExpireLabAuditLogsAfter30Days`, confirming application of the existing **30-day lifecycle rule**. Expiration is asynchronous and is not an exact deletion deadline.
- Initial S3 delivery took approximately **five minutes** after logging started. Zero-byte folder markers observed earlier were not treated as proof of audit-log delivery.
- **No delivery errors were reported** during verification. Both S3 and CloudWatch delivery were confirmed before Phase 3 was declared complete.

### Excluded features and phase boundary

- Global service events: disabled (`IncludeGlobalServiceEvents=false`). The Ohio-only trail does not provide global IAM activity or complete console sign-in coverage.
- Organization trail: disabled (`IsOrganizationTrail=false`).
- Data events: excluded (`DataResources=[]`).
- Network activity events: not configured.
- CloudTrail Insights: not enabled (`HasInsightSelectors=false`).
- No multi-Region logging, optional S3 prefix, KMS key, or SNS topic configured on the trail.
- Existing Phase 2 bucket and IAM policies were unchanged during Phase 3.
- **At completion of Phase 3, no Phase 4 resources had been created.** That verification returned no metric filters for the lab log group, no CloudWatch alarms, and no SNS topics in us-east-2. No KMS keys or other later-phase resources were created by Phase 3. Phase 4 was subsequently completed as recorded below.
- Further AWS changes require separate explicit user approval.

## Phase 4 implementation results — complete and verified

Phase 4 was implemented with explicit user approval and verified on October 22, 2026. All resources are in **us-east-2 (Ohio)** and use the existing `/aws/cloudtrail/security-monitoring-lab` log group. This section records the completed verification; this documentation update made no AWS API calls or AWS resource changes.

### Created filters, custom metrics, and alarms

Each row represents one metric filter, its generated custom metric in namespace `SecurityMonitoringLab`, and one CloudWatch alarm. The filter and alarm share the same name.

| Metric filter and alarm name | Custom metric name | Alarm threshold | Evaluation | Verified alarm state |
| --- | --- | --- | --- | --- |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | `UnauthorizedAPICalls` | `Sum >= 3` | One 300-second period; 1 of 1 datapoints | `OK` |
| `SecurityMonitoringLab-CloudTrailChanges` | `CloudTrailChanges` | `Sum >= 1` | One 300-second period; 1 of 1 datapoints | `OK` |
| `SecurityMonitoringLab-SecurityGroupChanges` | `SecurityGroupChanges` | `Sum >= 1` | One 300-second period; 1 of 1 datapoints | `OK` |

All three alarms use `ComparisonOperator=GreaterThanOrEqualToThreshold`, `Statistic=Sum`, `Period=300`, `EvaluationPeriods=1`, `DatapointsToAlarm=1`, and `TreatMissingData=notBreaching`.

Each filter publishes `metricValue=1`, `defaultValue=0`, and `unit=Count`, with **no dimensions**. Default zero applies when logs arrive without matches; no ingestion can still mean missing data. An `OK` alarm is not proof of logging pipeline health.

### Exact metric filter patterns

**Authorization failures** — matches Ohio API authorization errors; isolated mistakes and managed-policy restrictions can also match.

```text
{ ($.awsRegion = "us-east-2") && (($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*")) }
```

**CloudTrail configuration changes** — matches successful changes that could alter audit visibility; authorized administration also matches.

```text
{ ($.awsRegion = "us-east-2") && ($.eventSource = "cloudtrail.amazonaws.com") && (($.eventName = "StopLogging") || ($.eventName = "DeleteTrail") || ($.eventName = "UpdateTrail") || ($.eventName = "PutEventSelectors") || ($.eventName = "PutInsightSelectors")) && ($.errorCode NOT EXISTS) }
```

**Security-group rule changes** — matches successful ingress and egress rule changes, including legitimate changes that tighten access.

```text
{ ($.awsRegion = "us-east-2") && ($.eventSource = "ec2.amazonaws.com") && (($.eventName = "AuthorizeSecurityGroupIngress") || ($.eventName = "AuthorizeSecurityGroupEgress") || ($.eventName = "RevokeSecurityGroupIngress") || ($.eventName = "RevokeSecurityGroupEgress") || ($.eventName = "ModifySecurityGroupRules")) && ($.errorCode NOT EXISTS) }
```

### Verification results and limits

- All **three metric filters and three alarms** were read back and their configurations matched the approved plan, including the exact patterns, metric transformations, thresholds, evaluation settings, and action settings.
- The patterns had been validated with positive and negative sample events using the read-only `TestMetricFilter` API during planning.
- **Metric filtering and custom metric publication are working:** `GetMetricStatistics` returned a `Sum=0`, `Unit=Count` datapoint at **2026-10-22 06:12:00 UTC** for each of the three generated metrics after log ingestion.
- **All three alarms were in `OK` state at the final Phase 4 check.** Two alarms initially showed `INSUFFICIENT_DATA` during initialization and subsequently transitioned to `OK` without manual state changes.
- Zero-valued datapoints verify publication, not a positive security-event match or a complete alarm-trigger test. No synthetic audit events were injected, no alarm states were forced, and no security settings were weakened for testing. Filters process newly ingested logs and do not backfill historical events.
- **CloudTrail remains active** (`IsLogging=true`) with its Phase 3 configuration unchanged. Its status reported successful S3, CloudWatch Logs, and digest delivery, with no reported delivery errors during the Phase 4 checks.
- **At completion of Phase 4, no SNS notifications, topics, subscriptions, or automated remediation had been created for this lab.** At that check, all alarms had `ActionsEnabled=false` and empty `AlarmActions`, `OKActions`, and `InsufficientDataActions` lists, and the SNS topic inventory was empty. Phase 5 subsequently enabled notifications as recorded below.
- No KMS keys or other Phase 5 or later resources were created during Phase 4.
- The approved planning estimate for three custom metrics and three standard alarms was approximately **$1.20/month before allowances or credits**, plus existing log storage and ingestion costs. This is an estimate, not measured spending or a hard cap; the existing $2 monthly budget safeguard remains unchanged.

## Phase 5 implementation results — complete and verified

Phase 5 was implemented with explicit user approval. The email subscription was verified as confirmed before modifying any CloudWatch alarms, and the subscription status and all three alarm configurations were verified again afterward. This section records those completed checks; this local documentation update made no AWS API calls or AWS resource changes.

### SNS topic and confirmed subscription

- Topic name: `SecurityMonitoringLab-Alerts`.
- Topic ARN: `arn:aws:sns:us-east-2:374786579110:SecurityMonitoringLab-Alerts`.
- Region: **us-east-2 (Ohio)**.
- Topic type: **Standard**.
- One email subscription: `taseenarshad1991@gmail.com`, using protocol `email`.
- The user confirmed the subscription through the AWS SNS confirmation email. `GetSubscriptionAttributes` verified `PendingConfirmation=false` both before and after the alarm updates.
- The topic policy permits `cloudwatch.amazonaws.com` to publish only for project `374786579110` and the exact three lab alarm ARNs. It also denies insecure client transport for the configured topic-level actions.
- No KMS encryption was configured on the SNS topic, as approved for this budget-focused lab. No new IAM roles or KMS keys were created.

### Enabled alarm notifications

Each existing alarm has `ActionsEnabled=true` and exactly one `AlarmActions` entry: the `SecurityMonitoringLab-Alerts` topic ARN above.

| Existing alarm | ALARM notification destination | State at final Phase 5 verification |
| --- | --- | --- |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | `SecurityMonitoringLab-Alerts` | `OK` |
| `SecurityMonitoringLab-CloudTrailChanges` | `SecurityMonitoringLab-Alerts` | `OK` |
| `SecurityMonitoringLab-SecurityGroupChanges` | `SecurityMonitoringLab-Alerts` | `OK` |

- Read-back comparison confirmed that **all existing thresholds, metrics, dimensions, statistics, units, periods, evaluation settings, comparison operators, and missing-data settings were preserved** for all three alarms.
- **`OKActions=[]` and `InsufficientDataActions=[]` remain empty** on all three alarms.
- Notifications are configured for transitions into `ALARM`; they are not periodic reminders while an alarm remains in `ALARM`.
- **No automated remediation was configured.** The only enabled alarm action is publishing to the existing SNS topic.
- No additional topics, subscriptions, alarms, metric filters, IAM roles, KMS keys, or other resources were created while enabling the alarm actions.
- All alarm update and verification calls succeeded; no errors were reported during that completion step.

### Verification limits and phase boundary

- **At completion of Phase 5, no test notification had been sent.** Subscription confirmation and notification configuration had been verified, but SNS-to-inbox test delivery and a complete detection-to-email test had not been performed. Phase 6 subsequently validated notification delivery as recorded below; a complete real-event detection-to-email test remains outside the validated scope.
- No alarm states were forced and no synthetic security events were generated during Phase 5.
- **At completion of Phase 5, Phase 6 had not started.** Phase 6 was subsequently approved, completed, and verified as recorded below.

## Phase 6 validation results — completed and verified

Phase 6 was performed with explicit user approval in **us-east-2 (Ohio)** on **October 22, 2026**. Its purpose was to validate the existing metric filter matching logic and the alarm-to-SNS-to-email notification paths through controlled tests, then verify natural alarm recovery and preservation of the environment. It created no infrastructure and changed no detection or notification configuration.

### Step 1: baseline verification

- Project identity and Region matched the existing lab: `374786579110`, us-east-2.
- All three alarms were `OK`, with the approved thresholds, metrics, five-minute periods, evaluation settings, and `TreatMissingData=notBreaching`.
- All three alarms had actions enabled, with only `SecurityMonitoringLab-Alerts` in `AlarmActions` and empty `OKActions` and `InsufficientDataActions`.
- All three metric filters matched the approved patterns and transformations. Recent zero-valued datapoints were present for all three custom metrics.
- The SNS topic policy was restricted to the three lab alarm ARNs and the project ID. Its single email subscriber was confirmed (`PendingConfirmation=false`), with no pending subscriptions reported.
- CloudTrail was active with the approved event selection and log file validation settings, successful S3 and CloudWatch Logs delivery, and no reported delivery errors.
- The baseline identified stale alarm descriptions saying "No notification or remediation actions" despite enabled SNS notifications. This metadata discrepancy was reported and left unchanged; the user then explicitly approved the filter tests and each subsequent alarm test.

### Step 2: synthetic filter matching tests

The exact patterns were read from the existing filters on `/aws/cloudtrail/security-monitoring-lab` and tested sequentially using `TestMetricFilter`. **All 12 samples passed: all six positive samples matched and all six negative samples did not match.** No unexpected result occurred.

| Existing filter | Synthetic JSON case | Expected result | Actual result |
| --- | --- | --- | --- |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | Ohio request with `errorCode=AccessDeniedException` | Match | Matched — pass |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | Ohio request with `errorCode=Client.UnauthorizedOperation` | Match | Matched — pass |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | Successful Ohio request with no error code | No match | Did not match — pass |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | `AccessDeniedException` request in us-east-1 | No match | Did not match — pass |
| `SecurityMonitoringLab-CloudTrailChanges` | Successful Ohio `StopLogging` from `cloudtrail.amazonaws.com` | Match | Matched — pass |
| `SecurityMonitoringLab-CloudTrailChanges` | Successful Ohio `PutEventSelectors` from `cloudtrail.amazonaws.com` | Match | Matched — pass |
| `SecurityMonitoringLab-CloudTrailChanges` | Read-only Ohio `GetTrail` from `cloudtrail.amazonaws.com` | No match | Did not match — pass |
| `SecurityMonitoringLab-CloudTrailChanges` | Ohio `UpdateTrail` with `errorCode=AccessDeniedException` | No match | Did not match — pass |
| `SecurityMonitoringLab-SecurityGroupChanges` | Successful Ohio `AuthorizeSecurityGroupIngress` from `ec2.amazonaws.com` | Match | Matched — pass |
| `SecurityMonitoringLab-SecurityGroupChanges` | Successful Ohio `ModifySecurityGroupRules` from `ec2.amazonaws.com` | Match | Matched — pass |
| `SecurityMonitoringLab-SecurityGroupChanges` | Read-only Ohio `DescribeSecurityGroups` from `ec2.amazonaws.com` | No match | Did not match — pass |
| `SecurityMonitoringLab-SecurityGroupChanges` | Ohio `RevokeSecurityGroupEgress` with `errorCode=AccessDeniedException` | No match | Did not match — pass |

The us-east-1 value above existed only inside synthetic sample JSON; no test API was run outside us-east-2. **No synthetic events were ingested into the real log group, and no artificial metric datapoints were published.** The filter tests did not trigger alarms or send SNS notifications.

### Step 3: controlled alarm notification tests and natural recovery

Each alarm was tested individually after separate user approval. Before each test, the target and previously tested alarms were verified as `OK`, the target had only the existing SNS topic as its ALARM action, and the email subscription was confirmed.

`SetAlarmState` temporarily changed only the target alarm's state to `ALARM`, with this exact reason:

```text
PHASE 6 CONTROLLED TEST - notification verification; no security incident
```

Alarm history confirmed each controlled `OK → ALARM` transition and a successful notification action to `arn:aws:sns:us-east-2:374786579110:SecurityMonitoringLab-Alerts`, with no reported action error. **All three notification tests succeeded, and the user personally confirmed receipt of the SNS email for each alarm.**

All timestamps below are UTC on October 22, 2026.

| Alarm | Controlled ALARM transition | Natural return to OK | Approximate recovery time | SNS action and inbox result |
| --- | --- | --- | --- | --- |
| `SecurityMonitoringLab-UnauthorizedAPICalls` | 06:51:38 | 06:52:24 | 46 seconds | Action succeeded; user confirmed email receipt |
| `SecurityMonitoringLab-CloudTrailChanges` | 06:57:31 | 06:58:41 | 69 seconds | Action succeeded; user confirmed email receipt |
| `SecurityMonitoringLab-SecurityGroupChanges` | 07:00:47 | 07:02:13 | 86 seconds | Action succeeded; user confirmed email receipt |

Recovery history showed normal metric evaluation using a sum of zero below the respective thresholds (3 for UnauthorizedAPICalls and 1 for each other alarm). **No alarm was manually forced back to OK.** No thresholds, metrics, periods, filters, topic settings, or subscriptions were changed for the tests.

### Step 4: final environment verification

- **All three alarms were `OK`** at the final check. Alarm configuration update timestamps remained unchanged since Phase 5.
- Existing detection settings were preserved: namespace `SecurityMonitoringLab`, the three original metric names, `Count` unit, no dimensions, `Sum` statistic, thresholds 3/1/1, `Period=300`, `EvaluationPeriods=1`, `DatapointsToAlarm=1`, `GreaterThanOrEqualToThreshold`, and `TreatMissingData=notBreaching`.
- Existing notification settings were preserved: actions enabled, only `SecurityMonitoringLab-Alerts` in each `AlarmActions` list, and empty `OKActions` and `InsufficientDataActions`.
- All three metric filter patterns and transformations remained unchanged: metric value 1, default value 0, unit Count, and no dimensions.
- The single SNS email subscription remained confirmed, and the topic policy remained restricted to the three lab alarms.
- **CloudTrail remained active**, with unchanged event selectors and integrity validation enabled. Its final status reported successful S3 delivery at **07:01:33 UTC** and CloudWatch Logs delivery at **07:02:42 UTC**, with **no reported delivery errors**.
- **No unintended resources were created by Phase 6.** Inventory checks found only the expected lab bucket, trail, log group, role, topic, subscription, and three alarms alongside pre-existing managed roles. No KMS keys, Lambda functions, EC2 instances, CloudFormation stacks, or EventBridge rules were returned in us-east-2; the supplementary us-east-1 log-group inventory was empty.
- **No automated remediation was created or enabled.** Alarm actions contain only SNS publishing.
- Final verification used read-only calls. This documentation update also makes no AWS API calls or AWS resource changes.

### Validation limits and unresolved documentation/metadata debt

- These tests validated **filter matching and alarm-to-SNS-to-email delivery separately**. They did not prove a complete real security-event → CloudTrail → metric filter → threshold evaluation → email sequence.
- No real unauthorized AWS API activity, real CloudTrail configuration changes, or real security-group changes were generated. Controlled alarm state changes did not create real metric breaches.
- The trail remains **Ohio-only**, with global-service events disabled and data events excluded. It does not provide complete global IAM or console sign-in coverage, or data-event coverage.
- **Unresolved documentation/metadata debt:** all three AWS alarm descriptions still say **"No notification or remediation actions"**, even though SNS notifications are enabled. The description text is stale; the actual notification settings are correct and were successfully tested. **The descriptions were not fixed.** Correcting them requires a separately approved AWS configuration change.
- Phase 6 is **completed and verified**. No later phase is defined or started by this update.

## Approval boundary

**AWS resources require explicit user approval before creation or modification.** This document grants no authorization to create, modify, enable, disable, or delete AWS resources. Destructive operations also require explicit approval.

Only this local planning document is authorized by the current request. Any later implementation must remain within the approved phase and must not silently add resources or expand permissions.

## References

- [CloudTrail S3 bucket policy](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/create-s3-bucket-policy-for-cloudtrail.html)
- [S3 Object Ownership and disabled ACLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)
- [CloudTrail CloudWatch Logs role policy](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-required-policy-for-cloudwatch-logs.html)
- [CloudTrail confused deputy prevention](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cross-service-confused-deputy-prevention.html)
- [Supported services for the new AWS experience](https://docs.aws.amazon.com/accounts/latest/reference/supported-services-sign-up-new.html)
