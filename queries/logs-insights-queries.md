# Logs Insights queries for Security Hub findings

Run these from Log Analytics (the default view under **Logs** in the CloudWatch console). Select the security findings log group, paste the query, set the time range, and choose **Run query**.

Field paths assume the flat ASFF structure delivered by CloudWatch. Validate against your own findings if you ingest differently.

Security Hub re-delivers a finding on every update, so each query filters on `Workflow.Status = "NEW"` and `RecordState = "ACTIVE"` to count active, untriaged finding events. A finding updated while still NEW is still re-delivered and still matches; the filter excludes findings that have been resolved, suppressed, or already triaged. Drop those two clauses if you want the full history of a finding, including its updates.

## Recent critical findings by resource

```
fields @timestamp, Severity.Label as severity, ProductName, Resources.0.Id as resource, Title
| filter severity in ["CRITICAL", "HIGH"] and Workflow.Status = "NEW" and RecordState = "ACTIVE"
| sort @timestamp desc
| limit 25
```

Use this when an alarm fires and you need immediate context: which resources have active critical findings, which service detected them, and what each finding says.

## Findings on a specific resource during an incident

```
fields @timestamp, Severity.Label as severity, Types.0 as findingType, Title
| filter Resources.0.Id like /i-0123456789abcdef0/
| sort @timestamp desc
| limit 50
```

Replace `i-0123456789abcdef0` with the actual resource identifier from the alarm. This returns the full finding history for the resource under investigation, including updates, so it is intentionally not filtered to NEW. This is the query behind Pattern 2 (security context as first triage step) and Step 1 of the decision tree.

## Finding volume by source (last 24 hours)

```
filter Workflow.Status = "NEW" and RecordState = "ACTIVE"
| stats count(*) as findingCount by ProductName, Severity.Label
| sort findingCount desc
```

Shows which services are generating new findings and at what severity. Useful for tuning: if one source dominates, investigate whether it needs scoping or suppression at the Security Hub level.

## Compliance failures grouped by resource type

```
fields @timestamp, Resources.0.Type as resourceType, Resources.0.Id as resource, Title
| filter Compliance.Status = "FAILED" and RecordState = "ACTIVE"
| stats count(*) as failures by resourceType
| sort failures desc
```

Surfaces which resource types have the most active compliance failures, so you can prioritize remediation by category rather than chasing individual findings.

## Dashboard widget queries

The three Logs Insights queries below power the log widgets in the Security-Findings-Overview-<region> dashboard defined in `templates/security-hub-cloudwatch-foundation.yaml`. The per-resource correlation dashboard in `templates/security-hub-cloudwatch-per-resource.yaml` adds a metric widget that overlays the per-resource finding metric on the CPU graph (not a Logs Insights query) and a resource-scoped version of the recent-findings table.

**Findings by severity (bar chart):**
```
filter Workflow.Status = "NEW" and RecordState = "ACTIVE"
| stats count(*) as total by Severity.Label
| sort total desc
```

**Recent new critical/high findings (table):**
```
fields @timestamp, Severity.Label as severity, Title, Resources.0.Id as resource
| filter severity in ["CRITICAL", "HIGH"] and Workflow.Status = "NEW" and RecordState = "ACTIVE"
| sort @timestamp desc
| limit 10
```

**Critical/high finding trend (time series):**
```
filter Severity.Label in ["CRITICAL", "HIGH"] and Workflow.Status = "NEW" and RecordState = "ACTIVE"
| stats count() as findings by bin(1h)
```
