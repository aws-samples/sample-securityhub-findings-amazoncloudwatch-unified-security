# Bring Security Hub Findings into Your CloudWatch On-Call Workflow

Companion resources for the AWS Cloud Operations Blog post. This repository contains the CloudFormation templates, IAM policies, Logs Insights queries, and console walkthroughs referenced in the blog.

## What is in this repository

```
templates/
  security-hub-cloudwatch-foundation.yaml      Deploy once per account/Region: account-wide metric filters and overview dashboard
  security-hub-cloudwatch-per-resource.yaml    Deploy once per monitored resource: per-resource filter, alarms, composite alarm, correlation dashboard
  iam-cloudops-read-only.json                  Read-only IAM policy for CloudOps engineers
queries/
  logs-insights-queries.md        Logs Insights queries for incident investigation and dashboards
LICENSE                           MIT-0 license
```

The CloudFormation is split into two stacks so you can run more than one monitored
resource without collisions. The foundation stack holds the account-wide pieces
that can exist only once per account and Region (the crypto-mining and
production-critical filters, and the Security-Findings-Overview dashboard). The
per-resource stack holds everything keyed to a single resource (the scoped metric
filter, the security-finding alarm, the performance alarm, the composite alarm,
and a correlation dashboard named per instance). Deploy the foundation once, then
the per-resource stack once for each resource you page on.

## Prerequisites

- AWS Security Hub CSPM enabled in your account or organization
- A CloudWatch telemetry enablement rule for Security Hub CSPM configured (see the setup walkthrough below)
- An existing performance alarm on the workload you want to correlate with security findings

## Console walkthroughs

### Setting up the telemetry enablement rule

Security Hub CSPM is the cloud security posture management service previously known as Security Hub; it was renamed when the new Security Hub became generally available in December 2025. CloudWatch ingests Security Hub CSPM findings through a telemetry enablement rule.

1. Open the Amazon CloudWatch console and navigate to **Settings**, then choose **Telemetry enablement rules**.
2. Choose **Create rule**.
3. For **Data source**, select **Security Hub CSPM**. This covers findings from Security Hub CSPM standards checks (AWS Foundational Security Best Practices, CIS Benchmarks), Amazon GuardDuty threat findings forwarded to Security Hub CSPM, and Amazon Inspector vulnerability findings.
4. For **Scope**, choose **Entire organization** (all current and future accounts inherit the rule) or **Specific accounts/OUs**.
5. For **Delivery format**, choose **ASFF** (native schema) or **OCSF** (vendor-neutral). ASFF is recommended for most CloudOps use cases since the metric filter patterns in this repository use ASFF field paths.
6. Choose **Create rule**.

Once the rule is active, findings begin flowing to the managed log group within minutes. Confirm the log group name in your CloudWatch console under **Logs > Log groups**.

### Setting up CloudWatch centralization rules (multi-account)

For organizations that want to centralize findings in a single observability account:

1. From the organization management or delegated administrator account, open the CloudWatch console and choose **Ingestion**, then the centralization configuration for logs.
2. Create a centralization rule and set the destination account and primary destination Region.
3. Scope the rule to the entire organization, specific OUs, or individual accounts, and use log group LIKE / NOT LIKE patterns to include the security findings log groups.
4. Enable **Metrics** in the same centralization rule, not just Logs. The composite alarm correlates the security-finding metric with the performance metric, so both metrics have to exist in the destination account. Metrics and Logs are separate toggles on the rule; a logs-only rule centralizes the findings log group but not the metrics the alarms read.
5. (Optional) Configure a backup Region and a customer managed KMS key for the destination log groups.
6. Create the rule. CloudWatch replicates matching log data into the destination account and begins publishing centralization metrics (for example, IncomingCopiedLogEvents) so you can confirm delivery.

Because centralization physically replicates the data, alarms, metric filters, and dashboards in the destination account operate on local data with no cross-account query federation.

Two things to confirm before you rely on the centralized copies:

- **Not every metric centralizes.** CloudWatch metrics centralization supports custom, Embedded Metric Format (EMF), and OpenTelemetry (OTLP) metrics. AWS vended metrics such as `AWS/EC2` `CPUUtilization` are not centralized. The security-finding metric in this template is a metric-filter custom metric, so it does centralize; the EC2 CPU metric does not. To run the composite in the destination account, publish the performance signal there as a custom metric (for example with the CloudWatch agent, which emits CPU under the `CWAgent` namespace) rather than relying on the vended `AWS/EC2` metric.
- **Centralized metrics gain extra dimensions.** Centralized copies carry added `:@aws.account` and `:@aws.region` dimensions that identify the source account and Region. An alarm written against the workload-account dimensions will not match the centralized copy unless you account for these, so test the alarm dimensions against the centralized metrics before you depend on them.

### Creating a metric filter

Create the same filter once, then repeat for the additional filters in the CloudFormation template:

1. Open the CloudWatch console and choose **Logs**, then **Log groups**.
2. Choose the security findings log group that your enablement rule delivers to.
3. Choose **Actions**, then **Create metric filter**.
4. For **Filter pattern**, enter the pattern from the template (for example, `{ ($.Severity.Label = "CRITICAL") || ($.Severity.Label = "HIGH") }`). Use **Test pattern** with a sample finding to confirm it matches.
5. Choose **Next**, then enter a Filter name.
6. Under **Metric details**, set Metric namespace (for example, SecurityHub/Operational), Metric name, and a Metric value of 1 so the metric increments once per matching finding.
7. (Optional) If you query centralized log groups, use Filter selection criteria to scope by source account or Region.
8. Choose **Next**, review, and choose **Create metric filter**. The metric appears once matching findings arrive.

#### A note on the CryptoMining filter

The account-wide `CryptoMiningMetricFilter` in `templates/security-hub-cloudwatch-foundation.yaml` matches GuardDuty cryptocurrency findings as they appear after the GuardDuty to Security Hub CSPM mapping. GuardDuty's native finding type is rewritten into the ASFF `Types` array, so the value in the log event reads like `TTPs/Command and Control/CryptoCurrency:EC2-BitcoinTool.B`. A single `*CryptoCurrency*` test catches the BitcoinTool findings but misses `CryptoMiner` and `BitcoinDomainRequest`, which do not carry that literal substring, so the pattern matches both `*Crypto*` and `*Bitcoin*` on `Types[0]` to cover the whole family.

The `CryptoCurrency` (BitcoinTool) variant is runtime-verified: a real GuardDuty sample finding delivered through Security Hub CSPM carried `TTPs/Command and Control/CryptoCurrency:EC2-BitcoinTool.B` in `Types[0]`, the filter matched it, and the `CryptoMiningFindings` metric incremented. The `CryptoMiner` and `BitcoinDomainRequest` variants are verified against the published GuardDuty-to-ASFF finding-type mapping and with synthetic ASFF log events, not against live findings, because `aws guardduty create-sample-findings` emits only the BitcoinTool crypto variant and not those two types. A non-crypto CRITICAL finding was confirmed not to match.

To reconfirm in your own account: run `create-sample-findings` to exercise the `CryptoCurrency` path end to end. For the `CryptoMiner` and `BitcoinDomainRequest` family members, either wait for a real detection or inject a synthetic ASFF event with the matching `Types[0]` value, then use **Test pattern** on the metric filter. `Types[0]` assumes the crypto finding type is the first element of the `Types` array, which held for the GuardDuty findings tested but is worth reconfirming against your own sample.

### Creating the composite alarm

1. Ensure both child alarms already exist: the security-finding metric alarm and your existing performance alarm on the same resource.
2. Open the CloudWatch console and choose **Alarms**, then **All alarms**.
3. Select the check box next to both child alarms, then choose **Create composite alarm**.
4. Under **Specify composite alarm conditions**, set the rule expression to require both: `ALARM(SecurityFindingAlarm) AND ALARM(PerformanceAlarm)`. Change the default OR operator to AND.
5. Choose **Next**, then under **Configure actions** select the same SNS topic your performance alarms already use, so the notification lands in the existing on-call path.
6. Enter a name and description (include what to do and what not to do), then create the alarm.

### Adding a vertical annotation to a metric widget

The annotation is a property of the metric widget itself, so you add it to the widget JSON rather than creating a separate resource:

1. Open the dashboard, choose the CPU (or other performance) widget, and choose **Edit**.
2. Switch to the **Source** view to see the widget JSON.
3. Inside `properties`, add an `annotations` object with a `vertical` array, then add an entry with the finding timestamp as `value`, a `label`, a `color`, and `fill` set to `after`.
4. Choose **Update widget**, then **Save dashboard**. The vertical line appears on the graph at the finding timestamp.

A single metric widget can carry one or more vertical annotations, but it cannot have both an alarm and annotations on the same widget. To keep this current without manual edits, automate step 3 with a Lambda function triggered by the Critical/High findings alarm that calls the CloudWatch PutDashboard API to inject the annotation at the finding timestamp.

### Building Logs Insights widgets on the dashboard

1. Open your dashboard and choose **Add widget**.
2. Pick the visualization (bar, pie, table, or time series).
3. Select **Logs Insights** as the data source.
4. Choose the security findings log group.
5. Paste the matching query from `queries/logs-insights-queries.md`.
6. Run it to confirm results, then choose **Create widget**.
7. Repeat for each query.
8. Choose **Save dashboard**.

These queries are also built into the dashboard definitions in the templates. The three account-wide findings views are in the Security-Findings-Overview dashboard in `templates/security-hub-cloudwatch-foundation.yaml`; the resource-scoped findings table is in the correlation dashboard in `templates/security-hub-cloudwatch-per-resource.yaml`.

## Deploying the CloudFormation template

Deploy the foundation stack once per account and Region:

```bash
aws cloudformation deploy \
  --template-file templates/security-hub-cloudwatch-foundation.yaml \
  --stack-name security-hub-cloudwatch-foundation \
  --parameter-overrides FindingsLogGroupName=/aws/securityhub/findings
```

Then deploy the per-resource stack once for each resource you page on, giving each deployment its own stack name and resource parameters:

```bash
aws cloudformation deploy \
  --template-file templates/security-hub-cloudwatch-per-resource.yaml \
  --stack-name security-hub-cloudwatch-i-0abc123 \
  --parameter-overrides \
      FindingsLogGroupName=/aws/securityhub/findings \
      MonitoredResourceArn=arn:aws:ec2:us-east-1:111122223333:instance/i-0abc123 \
      MonitoredInstanceId=i-0abc123 \
      CpuThreshold=90 \
      P1EscalationSNSTopicArn=arn:aws:sns:us-east-1:111122223333:oncall-p1
```

Neither stack creates IAM resources, so no capability flag is needed. The per-resource stack names its composite alarm and correlation dashboard after the instance ID, so repeat deployments for different resources never collide. The foundation stack uses fixed names and is meant to exist only once per account and Region.

Before applying `templates/iam-cloudops-read-only.json`, replace the `{region}`, `{account_id}`, and `{findings_log_group_name}` placeholders in the `Resource` ARNs with your own Region, account ID, and the name of the log group your enablement rule delivers findings to. The `logs:DescribeLogGroups` and `logs:StopQuery` actions do not support resource-level permissions, so they sit in a separate statement scoped to all log groups in the account and Region; the remaining actions stay scoped to the findings log group.

## Disclaimer

This repository contains sample code provided for illustration purposes only. The templates, policies, queries, and walkthroughs are examples and are not intended for production use as is. Review, test, and validate them in a non-production environment first, and work with your own security, compliance, and legal teams to meet your organization's requirements before you deploy any of it. The example thresholds, resource names, account IDs, and findings in this repository are illustrative and do not represent a real environment.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
