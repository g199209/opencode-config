---
name: lightsail-traffic-query
description: Query AWS Lightsail instance traffic usage, monthly transfer allowance, outbound/inbound month-to-date metrics, and remaining bandwidth using AWS CLI. Use this skill whenever the user asks about Lightsail bandwidth, traffic usage, transfer package size, remaining monthly transfer, Tokyo Lightsail usage, or current-month Lightsail NetworkIn/NetworkOut, even if they do not specify an instance name.
---

# AWS Lightsail Traffic Query

Use this skill to answer questions about AWS Lightsail instance traffic usage and monthly transfer allowance with AWS CLI.

The common user intent is: "How much traffic has my Lightsail instance used this month?" or "How much transfer is left in my Lightsail package?" Lightsail does not expose a direct month-to-date package-usage API, so query `NetworkIn` and `NetworkOut` metrics and sum them client-side.

## Core Rules

- Use AWS CLI read-only commands only.
- If the user does not specify an instance name, list Lightsail instances in the requested region first.
- If exactly one instance is returned, use it automatically.
- If multiple instances are returned and the user did not specify one, show the list and ask which one to query.
- If the user specifies Tokyo, use region `ap-northeast-1`.
- Tokyo Zone A is `ap-northeast-1a`.
- For monthly package usage, estimate usage as `NetworkIn + NetworkOut`.
- If the user asks only for outbound traffic, query only `NetworkOut`.
- Monthly transfer allowance is read from `instance.networking.monthlyTransfer.gbPerMonthAllocated`.
- Metric values are bytes. Convert bytes to GiB with `bytes / 1073741824`.
- AWS reports `gbPerMonthAllocated` as GB. When computing remaining GiB precisely, convert allowance with `allowanceGB * 1000000000 / 1073741824`.
- In `awk` examples, avoid reserved identifiers such as `in`; pass metric variables as `in_bytes` and `out_bytes` instead.

## AWS Profile Handling

First try AWS CLI without `--profile`.

If AWS returns `NoCredentials` or cannot locate credentials, list configured profiles:

```bash
aws configure list-profiles
```

Then retry with an available profile:

```bash
--profile <profile>
```

In this user's current environment, profile `aws` is known to work. Prefer `--profile aws` when default credentials are missing and that profile exists.

## Region Mapping

| User says | AWS region / AZ |
|---|---|
| Tokyo | `ap-northeast-1` |
| Tokyo Zone A | `ap-northeast-1a` |
| Singapore | `ap-southeast-1` |
| Virginia | `us-east-1` |
| Oregon | `us-west-2` |
| Ireland | `eu-west-1` |
| Frankfurt | `eu-central-1` |

## Workflow

### 1. Resolve Region and Instance

If the user gives a region, map it to an AWS region code. If the user gives only an availability zone such as Tokyo Zone A, use the parent region for API calls: `ap-northeast-1`.

If the instance name is missing, list instances:

```bash
aws lightsail get-instances \
  --region ap-northeast-1 \
  --query 'instances[].{Name:name,AZ:location.availabilityZone,State:state.name}' \
  --output table
```

With an explicit profile:

```bash
aws lightsail get-instances \
  --profile aws \
  --region ap-northeast-1 \
  --query 'instances[].{Name:name,AZ:location.availabilityZone,State:state.name}' \
  --output table
```

### 2. Query Monthly Transfer Allowance

```bash
aws lightsail get-instance \
  --profile aws \
  --region ap-northeast-1 \
  --instance-name Datahub \
  --query 'instance.{Name:name,Bundle:bundleId,MonthlyTransferGB:networking.monthlyTransfer.gbPerMonthAllocated}' \
  --output table
```

Use the returned `MonthlyTransferGB` as the Lightsail package allowance reported by AWS.

### 3. Query Month-to-Date Outbound Traffic

```bash
aws lightsail get-instance-metric-data \
  --profile aws \
  --region ap-northeast-1 \
  --instance-name Datahub \
  --metric-name NetworkOut \
  --period 86400 \
  --start-time "$(date -u +%Y-%m-01T00:00:00Z)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --unit Bytes \
  --statistics Sum \
  --query 'sum(metricData[].sum)' \
  --output text | awk '{printf "%.2f GiB\n", $1/1073741824}'
```

### 4. Query Month-to-Date Inbound Traffic

```bash
aws lightsail get-instance-metric-data \
  --profile aws \
  --region ap-northeast-1 \
  --instance-name Datahub \
  --metric-name NetworkIn \
  --period 86400 \
  --start-time "$(date -u +%Y-%m-01T00:00:00Z)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --unit Bytes \
  --statistics Sum \
  --query 'sum(metricData[].sum)' \
  --output text | awk '{printf "%.2f GiB\n", $1/1073741824}'
```

### 5. Query Total Used and Remaining Transfer

Use this when the user asks "how much traffic is left this month?"

```bash
INSTANCE="Datahub"
REGION="ap-northeast-1"
PROFILE="aws"

ALLOWANCE_GB=$(aws lightsail get-instance \
  --profile "$PROFILE" \
  --region "$REGION" \
  --instance-name "$INSTANCE" \
  --query 'instance.networking.monthlyTransfer.gbPerMonthAllocated' \
  --output text)

IN_BYTES=$(aws lightsail get-instance-metric-data \
  --profile "$PROFILE" \
  --region "$REGION" \
  --instance-name "$INSTANCE" \
  --metric-name NetworkIn \
  --period 86400 \
  --start-time "$(date -u +%Y-%m-01T00:00:00Z)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --unit Bytes \
  --statistics Sum \
  --query 'sum(metricData[].sum)' \
  --output text)

OUT_BYTES=$(aws lightsail get-instance-metric-data \
  --profile "$PROFILE" \
  --region "$REGION" \
  --instance-name "$INSTANCE" \
  --metric-name NetworkOut \
  --period 86400 \
  --start-time "$(date -u +%Y-%m-01T00:00:00Z)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --unit Bytes \
  --statistics Sum \
  --query 'sum(metricData[].sum)' \
  --output text)

awk -v in_bytes="$IN_BYTES" -v out_bytes="$OUT_BYTES" -v allowance_gb="$ALLOWANCE_GB" '
BEGIN {
  used_bytes = in_bytes + out_bytes
  allowance_bytes = allowance_gb * 1000 * 1000 * 1000
  remaining_bytes = allowance_bytes - used_bytes

  printf "NetworkIn:   %.2f GiB\n", in_bytes / 1073741824
  printf "NetworkOut:  %.2f GiB\n", out_bytes / 1073741824
  printf "Total Used:  %.2f GiB\n", used_bytes / 1073741824
  printf "Allowance:   %.0f GB\n", allowance_gb
  printf "Remaining:   %.2f GiB approx\n", remaining_bytes / 1073741824
  printf "Used:        %.2f%% approx\n", used_bytes / allowance_bytes * 100
}'
```

### 6. Query Historical Monthly Outbound Traffic

Use this pattern when the user asks for monthly outbound traffic by month.

```bash
aws lightsail get-instance-metric-data \
  --profile aws \
  --region ap-northeast-1 \
  --instance-name Datahub \
  --metric-name NetworkOut \
  --period 86400 \
  --start-time 2026-04-01T00:00:00Z \
  --end-time 2026-05-01T00:00:00Z \
  --unit Bytes \
  --statistics Sum \
  --query 'sum(metricData[].sum)' \
  --output text | awk '{printf "2026-04 %.2f GiB\n", $1/1073741824}'
```

For multiple independent months, run month queries in parallel when possible.

## Output Format

For total monthly package usage, use this concise format:

```text
Instance: Datahub
Region: ap-northeast-1 / ap-northeast-1a
Bundle: micro_3_0
Monthly transfer package: 2048 GB

Month-to-date:
NetworkIn: 481.04 GiB
NetworkOut: 484.07 GiB
Total used: 965.11 GiB
Remaining: about 942.24 GiB
```

If the user asked only for outbound traffic:

```text
Datahub month-to-date NetworkOut: 484.07 GiB
```

For historical monthly outbound traffic:

```text
Datahub NetworkOut:
2026-01: 527.90 GiB
2026-02: 368.22 GiB
2026-03: 1947.68 GiB
2026-04: 2607.47 GiB
2026-05 to date: 484.16 GiB
```

## Caveats

- Lightsail does not expose a direct "month-to-date package usage" API.
- The workflow sums `NetworkIn` and `NetworkOut` metrics client-side.
- Use `--unit Bytes`; other units can return empty or unexpected results.
- Use `--statistics Sum` for traffic volume.
- Use `--period 86400`; Lightsail metric period max is 86400 seconds.
- This is near-real-time metric data, not exact billing data.
- Billing and Cost Explorer are the source of truth for charges, but they are delayed and less convenient for per-instance checks.
- For a single instance, this method is usually good enough.
- For multiple instances with the same `bundleId` in the same region, Lightsail may aggregate transfer allowance at the bundle/region level. Do not overcomplicate single-instance answers, but mention this caveat if the user asks about multiple instances or billing accuracy.
