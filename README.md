# Bring Security Hub findings into your CloudWatch on-call workflow

Companion resources for the AWS Cloud Operations Blog post. This repository contains the CloudFormation templates, IAM policies, Logs Insights queries, and console walkthroughs referenced in the blog.

## What is in this repository

```
templates/
  security-hub-cloudwatch.yaml    Full CloudFormation: metric filters, alarm, composite alarm, dashboard
  iam-cloudops-read-only.json     Read-only IAM policy for CloudOps engineers
queries/
  logs-insights-queries.md        Logs Insights queries for incident investigation and dashboards
LICENSE                           MIT-0 license
```

## Prerequisites

- AWS Security Hub CSPM enabled in your account or organization
- A CloudWatch telemetry enablement rule for Security Hub CSPM configured (see the setup walkthrough below)
- The ASFF resource identifier for the resource you want to correlate on, exactly as it appears in the finding's `Resources[0].Id` field, and the short EC2 instance ID of that same resource. Both values must point at the same resource; the template creates the performance alarm for you, so you do not need one to already exist.

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
4. (Optional) Configure a backup Region and a customer managed KMS key for the destination log groups.
5. Create the rule. CloudWatch replicates matching log data into the destination account and begins publishing centralization metrics (for example, IncomingCopiedLogEvents) so you can confirm delivery.

Because centralization physically replicates the data, alarms, metric filters, and dashboards in the destination account operate on local data with no cross-account query federation.

### Creating a metric filter

Create the same filter once, then repeat for the additional filters in the CloudFormation template:

1. Open the CloudWatch console and choose **Logs**, then **Log groups**.
2. Choose the security findings log group that your enablement rule delivers to.
3. Choose **Actions**, then **Create metric filter**.
4. For **Filter pattern**, enter the per-resource pattern from the template:
   `{ ($.Severity.Label = "CRITICAL" || $.Severity.Label = "HIGH") && $.Workflow.Status = "NEW" && $.RecordState = "ACTIVE" }`. Use **Test pattern** with a sample finding to confirm it matches.
5. Choose **Next**, then enter a Filter name.
6. Under **Metric details**, set Metric namespace (for example, SecurityHub/Operational), Metric name, a Metric value of 1, and a dimension of `ResourceId` with value `$.Resources[0].Id` so one metric is published per distinct resource. A dimensioned metric filter cannot also set a default value, so the metric only reports when a matching finding arrives; the alarm compensates with `TreatMissingData: notBreaching`.
7. (Optional) If you query centralized log groups, use Filter selection criteria to scope by source account or Region.
8. Choose **Next**, review, and choose **Create metric filter**. The metric appears once matching findings arrive.

`ResourceId` is an ARN, which makes it high-cardinality: CloudWatch Logs disables a metric filter once it generates more than 1,000 distinct dimension name and value pairs in a short window, and each distinct value bills as a separate custom metric. Scope this filter to the specific production resources you page on, not every resource that can produce a finding.

### Creating the composite alarm

The template creates the security-finding alarm, the performance alarm, and the composite alarm together, in the same account and Region. A composite alarm requires its child alarms to be co-located with it, so there is no console step to link a pre-existing performance alarm; deploying the stack creates all three as a set. If you prefer to build this by hand instead of deploying the template:

1. Ensure both child alarms exist in the same account and Region as each other: the security-finding metric alarm and the performance alarm on the same resource.
2. Open the CloudWatch console and choose **Alarms**, then **All alarms**.
3. Select the check box next to both child alarms, then choose **Create composite alarm**.
4. Under **Specify composite alarm conditions**, set the rule expression to require both: `ALARM(SecurityFindingAlarm) AND ALARM(PerformanceAlarm)`. Change the default OR operator to AND.
5. Choose **Next**, then under **Configure actions** select the SNS topic used for P1 escalation.
6. Enter a name and description (include what to do and what not to do), then create the alarm.

### Correlating findings with the CPU graph

The dashboard overlays the per-resource finding metric on the CPU graph's right Y axis, instead of relying on a static vertical annotation. A finding lands on the same timeline as CPU as soon as the metric reports, so the overlay stays current with no extra automation and works for any number of findings, not just the one that happened to be present when an annotation was added.

### Building Logs Insights widgets on the dashboard

The dashboard has three Logs Insights log widgets (findings by severity, recent new critical/high findings, and the critical/high finding trend) plus the metric overlay widget described above; there is no pie chart widget.

1. Open your dashboard and choose **Add widget**.
2. Pick the visualization (bar, table, or time series, matching the query).
3. Select **Logs Insights** as the data source.
4. Choose the security findings log group.
5. Paste the matching query from `queries/logs-insights-queries.md`.
6. Run it to confirm results, then choose **Create widget**.
7. Repeat for each of the three queries.
8. Choose **Save dashboard**.

All three Logs Insights queries are also included in the dashboard definition in `templates/security-hub-cloudwatch.yaml`; the fourth widget on that dashboard is the CPU and security findings metric overlay, not a Logs Insights query.

## Deploying the CloudFormation template

```bash
aws cloudformation deploy \
  --template-file templates/security-hub-cloudwatch.yaml \
  --stack-name security-hub-cloudwatch-ops \
  --parameter-overrides \
    FindingsLogGroupName=/aws/securityhub/findings \
    MonitoredResourceArn=arn:aws:ec2:us-east-1:111122223333:instance/i-0abc123 \
    MonitoredInstanceId=i-0abc123 \
    CpuThreshold=90 \
    P1EscalationSNSTopicArn=arn:aws:sns:us-east-1:111122223333:p1-escalation
```

This stack creates metric filters, alarms, a composite alarm, and a dashboard; it does not create IAM resources, so no capability flag is needed.

Parameters:

- `FindingsLogGroupName`: name of the CloudWatch Logs log group where Security Hub CSPM findings are delivered. Confirm the actual name in your CloudWatch console before deploying; it defaults to `/aws/securityhub/findings`.
- `MonitoredResourceArn`: the ASFF resource identifier to correlate on, exactly as it appears in the finding's `Resources[0].Id` field. This is the dimension value for the security-finding metric.
- `MonitoredInstanceId`: the same resource in the form the performance metric uses, which for EC2 CPU is the short instance ID. ASFF keys the resource by ARN while the AWS/EC2 namespace keys it by InstanceId, so `MonitoredResourceArn` and `MonitoredInstanceId` are separate parameters that must refer to the same instance.
- `CpuThreshold`: CPU utilization percent that trips the performance alarm. Defaults to 90.
- `P1EscalationSNSTopicArn`: ARN of the SNS topic for the composite (P1) escalation. Must be in the same Region as the stack.

Before applying `templates/iam-cloudops-read-only.json`, replace the `{region}` and `{account_id}` placeholders in the `Resource` ARNs with your own Region and account ID. If your findings log group uses a different name, update the ARN paths to match.

### Account placement

A composite alarm and its child alarms must all live in the same account and Region, so this template deploys the security-finding alarm, the performance alarm, and the composite alarm together in the workload account where the monitored instance runs and its findings are delivered. If you centralize findings in a separate observability account, replicate both the finding metric and the performance metric into that account and deploy this stack there instead.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
