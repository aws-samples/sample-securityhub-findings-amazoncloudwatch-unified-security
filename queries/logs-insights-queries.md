# Logs Insights Queries for Security Hub Findings

Run these from Log Analytics (the default view under **Logs** in the CloudWatch console). Select the security findings log group, paste the query, set the time range, and choose **Run query**.

Field paths assume the flat ASFF structure delivered by CloudWatch. Validate against your own findings if you ingest differently.

## Recent critical findings by resource

```
fields @timestamp, Severity.Label as severity, ProductName, Resources.0.Id as resource, Title
| filter severity in ["CRITICAL", "HIGH"]
| sort @timestamp desc
| limit 25
```

Use this when an alarm fires and you need immediate context: which resources have active critical findings, who detected them, and what the finding says.

## Findings on a specific resource during an incident

```
fields @timestamp, Severity.Label as severity, Types.0 as findingType, Title
| filter Resources.0.Id like /i-0abc123/
| sort @timestamp desc
| limit 50
```

Replace `i-0abc123` with the actual resource identifier from the alarm. This gives you the full finding history for the resource under investigation.

## Finding volume by source (last 24 hours)

```
stats count(*) as findingCount by ProductName, Severity.Label
| sort findingCount desc
```

Shows which services are generating findings and at what severity. Useful for tuning: if one source dominates, investigate whether it needs scoping or suppression at the Security Hub level.

## Compliance failures grouped by resource type

```
fields @timestamp, Resources.0.Type as resourceType, Resources.0.Id as resource, Title
| filter Compliance.Status = "FAILED"
| stats count(*) as failures by resourceType
| sort failures desc
```

Surfaces which resource types have the most compliance failures, helping prioritize remediation by category rather than chasing individual findings.

## Dashboard widget queries

The four queries below power the unified dashboard widgets defined in `templates/security-hub-cloudwatch.yaml`:

**Findings by severity (bar chart):**
```
stats count(*) as total by Severity.Label
| sort total desc
```

**Findings by source service (pie chart):**
```
stats count(*) as total by ProductName
| sort total desc
```

**Recent critical/high findings (table):**
```
fields @timestamp, Severity.Label as severity, Title, Resources.0.Id as resource
| filter severity in ["CRITICAL", "HIGH"]
| sort @timestamp desc
| limit 10
```

**Finding trend, last 7 days (time series):**
```
fields @timestamp
| filter Severity.Label in ["CRITICAL", "HIGH"]
| stats count() as findings by bin(1h)
```
