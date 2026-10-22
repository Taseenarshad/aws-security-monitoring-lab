# Deployment and Validation Evidence

These screenshots accompany the [project overview](../../README.md) and [detailed technical record](../../LAB_PLAN.md). Deployment views show configuration and state at capture time; historical views show controlled Phase 6 tests from October 22, 2026. They are not evidence of a real security incident or continuous monitoring health.

## Deployment State at Capture

### 1. CloudTrail active logging

[View CloudTrail evidence](01-cloudtrail-active-logging.png)

The deployed `security-monitoring-lab` trail shows logging active, multi-Region logging disabled, log file validation enabled, delivery timestamps, and the configured CloudWatch Logs destination. The console's disabled SNS delivery field refers to CloudTrail's own delivery notifications, not the separate CloudWatch alarm notifications. Disabled SSE-KMS does not mean the S3 objects are unencrypted: SSE-S3 was verified separately in the technical record.

### 2. Three healthy security alarms

[View alarm overview](02-security-alarms-healthy.png)

All three lab alarms are `OK` with actions enabled. Their last state-update timestamps align with the recorded natural recoveries. This view establishes the state at capture time; thresholds, action destinations, and pipeline health require the separate configuration and delivery checks documented in LAB_PLAN.md.

### 3. Deployed authorization metric filter

[View authorization filter configuration](03-authorization-filter-configuration.png)

The representative authorization filter shows the documented Ohio-only condition and both wildcard matches: `*UnauthorizedOperation` and `AccessDenied*`. It also shows the custom metric namespace/name, value 1, default 0, Count unit, no dimensions, and associated alarm. This is deployed-configuration evidence, not a screenshot of the 12 synthetic test results.

## Historical Controlled-Test Evidence

### 4. Controlled alarm history and recovery

[View controlled alarm history](04-controlled-alarm-history.png)

The UnauthorizedAPICalls history shows `OK → ALARM` at 06:51:38 UTC, successful execution of the SNS action, and `ALARM → OK` at 06:52:24 UTC, approximately 46 seconds later. The technical record establishes that the transition used `SetAlarmState` and that recovery occurred through normal metric evaluation. The screenshot's collapsed rows do not display the controlled-test reason; the received email below does. Personal identity and browser details have been cropped out, and the project component of the SNS ARN is masked.

### 5. Received SNS test email

[View received SNS test email](05-sns-test-email-received.png)

The received UnauthorizedAPICalls message identifies Ohio, the `OK → ALARM` transition, the explicit controlled-test reason, and the configured threshold and metric. Its 06:51:38 UTC test timestamp corresponds to the history above. It provides representative inbox-delivery evidence; receipt of all three test emails was personally confirmed and recorded in LAB_PLAN.md. Personal recipient details are not exposed, the project ID and ARN project component are masked, and the console link is covered. The public AWS sender address is not a personal email address. The stale description saying “No notification or remediation actions” remains visible and unresolved; it does not reflect the enabled SNS action.

## Interpretation and Publication Limits

- All 12 synthetic cases passed using `TestMetricFilter`, without inserting sample events into the live log group or publishing artificial metric datapoints. The screenshots do not independently establish those 12 results; see LAB_PLAN.md.
- Notification tests temporarily changed alarm states. No real unauthorized activity, CloudTrail configuration changes, or security-group changes were performed. Filter matching and alarm-to-SNS-to-email delivery were validated separately.
- No capture timestamps are inferred from file modification times. Visible delivery/state timestamps and the technical record provide the relevant chronology.
- All five final images were visually reviewed and are appropriate for public portfolio use: no complete live project IDs, personal email addresses, sensitive console URLs, subscription links, session identifiers, credentials, or unnecessary personal identity details were visible. Generic service domains, lab resource names, masked identifiers, and technical timestamps remain as context. This review covers the screenshots, not sanitization of the entire repository or its Git history.
- The main README embeds only the healthy-alarm overview and representative controlled-test history. All five images are linked here, with the remaining images linked directly from the main README. Screenshots were not modified during this review.
