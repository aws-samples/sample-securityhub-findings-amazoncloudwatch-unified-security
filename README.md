# Bring Security Hub Findings into Your CloudWatch On-Call Workflow

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

- AWS Security Hub enabled in your account or organization
- CloudWatch Telemetry enablement rules configured (see the setup walkthrough below)
- An existing performance alarm on the workload you want to correlate with security findings

## Console walkthroughs

### Setting up Telemetry enablement rules

You need two rules: one for Security Hub CSPM findings and one for Security Hub V2 findings.

1. Open the Amazon CloudWatch console and navigate to **Settings**, then choose **Telemetry enablement rules**.
2. Choose **Create rule**.
3. For **Data source**, select **Security Hub CSPM**. This covers findings from Security Hub standards checks (AWS Foundational Security Best Practices, CIS Benchmarks), GuardDuty threat findings forwarded to Security Hub, and Inspector vulnerability findings.
4. For **Scope**, choose **Entire organization** (all current and future accounts inherit the rule) or **Specific accounts/OUs**.
5. For **Delivery format**, choose **ASFF** (native Security Hub schema) or **OCSF** (vendor-neutral). ASFF is recommended for most CloudOps use cases since the metric filter patterns in this repository use ASFF field paths.
6. Choose **Create rule**.
7. Repeat steps 2 through 6, this time selecting **Security Hub V2** as the data source. V2 findings include the newer consolidated control findings format with enhanced resource context and remediation guidance.

Once both rules are active, findings begin flowing to the managed log group within minutes. Confirm the log group name in your CloudWatch console under **Logs > Log groups**.

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
4. For **Filter pattern**, enter the pattern from the template (for example, `{ ($.Severity.Label = "CRITICAL") || ($.Severity.Label = "HIGH") }`). Use **Test pattern** with a sample finding to confirm it matches.
5. Choose **Next**, then enter a Filter name.
6. Under **Metric details**, set Metric namespace (for example, SecurityHub/Operational), Metric name, and a Metric value of 1 so the metric increments once per matching finding.
7. (Optional) If you query centralized log groups, use Filter selection criteria to scope by source account or Region.
8. Choose **Next**, review, and choose **Create metric filter**. The metric appears once matching findings arrive.

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

All four queries are also included in the dashboard definition in `templates/security-hub-cloudwatch.yaml`.

## Deploying the CloudFormation template

```bash
aws cloudformation deploy \
  --template-file templates/security-hub-cloudwatch.yaml \
  --stack-name security-hub-cloudwatch-ops \
  --capabilities CAPABILITY_IAM
```

Update the parameter defaults in the template to match your environment (log group name, service name, API name, SNS topic ARN, performance alarm name).

Before applying `templates/iam-cloudops-read-only.json`, replace the `{region}` and `{account_id}` placeholders in the `Resource` ARNs with your own Region and account ID. If your findings log group uses a different name, update the ARN paths to match.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
