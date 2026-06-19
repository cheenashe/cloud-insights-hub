# AWS Cost Spike Investigation — Step by Step

> Your AWS bill jumped. Here's exactly how to find what caused it — in under 30 minutes.

---

## Table of Contents

- [First Response — Don't Panic](#first-response--dont-panic)
- [Step 1 — Identify the Service Responsible](#step-1--identify-the-service-responsible)
- [Step 2 — Narrow Down the Region and Account](#step-2--narrow-down-the-region-and-account)
- [Step 3 — Drill Into the Resource](#step-3--drill-into-the-resource)
- [Common Spike Culprits & Fixes](#common-spike-culprits--fixes)
- [Set Up Alerting to Catch This Earlier](#set-up-alerting-to-catch-this-earlier)
- [Prevention — Tagging Strategy](#prevention--tagging-strategy)
- [Cost Investigation Cheat Sheet](#cost-investigation-cheat-sheet)

---

## First Response — Don't Panic

Before touching anything, answer these three questions:

```
1. When did the spike start? (exact date/time)
2. Which AWS account? (if you have multiple)
3. Is this a one-time spike or sustained increase?
```

Open AWS Cost Explorer:  
`AWS Console → Billing → Cost Explorer → Enable if not already`

Set the time range to **Last 3 months, daily granularity** — you need the baseline to spot the anomaly.

---

## Step 1 — Identify the Service Responsible

```bash
# Using AWS CLI — get cost by service for last 30 days
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' \
  --output table | sort -k3 -rn
```

In the Console:
1. Cost Explorer → **Group by: Service**
2. Sort by cost descending
3. Compare current month vs last month — look for the outlier

**Common services that spike:**

| Service | Common Spike Reason |
|---------|-------------------|
| EC2 | Forgotten instances, NAT Gateway traffic, data transfer |
| S3 | Accidental public bucket, GET/PUT request explosion |
| RDS | Multi-AZ accidental enable, storage autoscaling |
| Lambda | Infinite retry loop, runaway invocations |
| Data Transfer | Cross-region traffic, internet egress |
| CloudWatch | Log ingestion explosion, high-resolution metrics |
| ElastiCache | Node type upgrade not reverted |
| EKS | Node group autoscaled and didn't scale down |

---

## Step 2 — Narrow Down the Region and Account

```bash
# Cost by region
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '14 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=REGION \
  --query 'ResultsByTime[*].{Date:TimePeriod.Start, Groups:Groups[*].{Region:Keys[0],Cost:Metrics.UnblendedCost.Amount}}' \
  --output json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for day in data:
    for g in day['Groups']:
        if float(g['Cost']) > 10:
            print(day['Date'], g['Region'], '\$'+str(round(float(g['Cost']),2)))
"
```

In Console:
- Cost Explorer → **Group by: Region**
- Identify the region where cost jumped

---

## Step 3 — Drill Into the Resource

Once you know the service + region, find the specific resource.

### For EC2 spikes:

```bash
# List all running instances with their type and launch time
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,Launched:LaunchTime,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table | sort -k3

# Look for instances launched around the time of the spike
# Especially large instance types: p3, p4, g4, x1, r6i etc.
```

```bash
# Check data transfer costs specifically
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["AWS Data Transfer"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

### For S3 spikes:

```bash
# Check request metrics per bucket (if enabled)
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name NumberOfObjects \
  --dimensions Name=BucketName,Value=your-bucket Name=StorageType,Value=AllStorageTypes \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T00:00:00Z \
  --period 86400 \
  --statistics Average

# Check if a bucket is public (security + cost risk)
aws s3api get-bucket-policy-status --bucket your-bucket-name
# "IsPublic": true is a red flag
```

### For Lambda spikes:

```bash
# Find functions with highest invocation count
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --start-time $(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 \
  --statistics Sum \
  --dimensions Name=FunctionName,Value=your-function-name

# Check for error loops (errors triggering retries triggering more errors)
aws logs filter-log-events \
  --log-group-name /aws/lambda/your-function \
  --start-time $(date -d '24 hours ago' +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].message' | head -20
```

### For RDS spikes:

```bash
# Check storage usage
aws rds describe-db-instances \
  --query 'DBInstances[*].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Storage:AllocatedStorage,MultiAZ:MultiAZ}' \
  --output table

# Storage autoscaling silently doubles your cost
# Check if it triggered
aws rds describe-db-instances \
  --db-instance-identifier your-db \
  --query 'DBInstances[0].MaxAllocatedStorage'
```

---

## Common Spike Culprits & Fixes

### 1. NAT Gateway data processing charges

**Symptom:** Data Transfer cost spike, not EC2.

```bash
# Check NAT Gateway bytes processed
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesInFromDestination \
  --dimensions Name=NatGatewayId,Value=nat-xxxxxxxx \
  --start-time $(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 \
  --statistics Sum
```

**Fix:** Move S3/DynamoDB traffic to VPC Endpoints (free):
```bash
# Create S3 VPC Endpoint — eliminates NAT Gateway charges for S3 traffic
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-xxxxxxxx
```

---

### 2. CloudWatch Logs explosion

**Symptom:** CloudWatch cost jumped, no new services deployed.

```bash
# Check log group sizes
aws logs describe-log-groups \
  --query 'logGroups[*].{Name:logGroupName,StoredBytes:storedBytes,RetentionDays:retentionInDays}' \
  --output table | sort -k3 -rn

# A log group with no retention = logs stored forever = growing cost
```

**Fix — set retention on all log groups:**
```bash
# Set 30-day retention on all log groups without retention
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].logGroupName' \
  --output text | tr '\t' '\n' | while read group; do
    echo "Setting retention for: $group"
    aws logs put-retention-policy \
      --log-group-name "$group" \
      --retention-in-days 30
done
```

---

### 3. Forgotten resources from a POC or test

```bash
# Find all EC2 instances older than 30 days that aren't tagged as production
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,Launched:LaunchTime,Env:Tags[?Key==`Environment`].Value|[0]}' \
  --output json | python3 -c "
from datetime import datetime, timezone
import json, sys
instances = json.load(sys.stdin)
for i in instances:
    launched = datetime.fromisoformat(i['Launched'].replace('Z','+00:00'))
    age = (datetime.now(timezone.utc) - launched).days
    if age > 30 and i['Env'] not in ['production','prod']:
        print(f'{i[\"ID\"]} | {i[\"Type\"]} | {age} days old | env:{i[\"Env\"]}')
"
```

---

### 4. EKS node group didn't scale down

```bash
# Check node group desired vs actual
aws eks list-nodegroups --cluster-name your-cluster
aws eks describe-nodegroup \
  --cluster-name your-cluster \
  --nodegroup-name your-nodegroup \
  --query 'nodegroup.scalingConfig'

# Manually scale down if cluster autoscaler failed
aws eks update-nodegroup-config \
  --cluster-name your-cluster \
  --nodegroup-name your-nodegroup \
  --scaling-config minSize=1,maxSize=10,desiredSize=2
```

---

## Set Up Alerting to Catch This Earlier

### AWS Cost Anomaly Detection (recommended — free):

```bash
# Create an anomaly monitor for all services
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "AllServicesMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Create an alert subscription (email when anomaly detected)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DailyAnomalyAlert",
    "MonitorArnList": ["arn:aws:ce::123456789:anomalymonitor/xxxxxxxx"],
    "Subscribers": [{"Address": "your-team@yourcompany.com", "Type": "EMAIL"}],
    "Threshold": 20,
    "Frequency": "DAILY"
  }'
```

### Budget alert for absolute threshold:

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "MonthlyBudgetAlert",
    "BudgetLimit": {"Amount": "1000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "your-team@yourcompany.com"}]
  }]'
```

---

## Prevention — Tagging Strategy

Tags are what make cost attribution fast. Without them, every investigation takes 10x longer.

```bash
# Enforce mandatory tags via AWS Config rule
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"Environment\",\"tag2Key\":\"Team\",\"tag3Key\":\"Project\"}"
  }'
```

**Minimum required tags for every resource:**

```
Environment: production | staging | dev | test
Team:        platform | backend | data | frontend
Project:     my-project-name
ManagedBy:   terraform | manual | cloudformation
CostCenter:  eng-001  (for finance chargeback)
```

---

## Cost Investigation Cheat Sheet

```bash
# === Quick 5-minute triage ===

# 1. Cost by service this month vs last month
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '60 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# 2. Daily cost last 14 days (spot the day it spiked)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '14 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY --metrics UnblendedCost

# 3. Cost by tag (who owns the expensive stuff)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Team

# 4. Untagged resources (the black hole)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Tags":{"Key":"Environment","MatchOptions":["ABSENT"]}}'
```

---

## When Cost Investigation Becomes a Monthly Routine

Investigating cost spikes reactively is a tax on your engineering time. The goal is proactive optimization.

> **Sygitech's [Cloud Optimization Services](https://www.sygitech.com/cloud-optimization.html)** provide continuous cost monitoring, right-sizing recommendations, reserved instance planning, and monthly optimization reports — so you catch cost spikes before they hit the bill.
>
> 👉 [See how we reduce cloud costs](https://www.sygitech.com/cloud-optimization.html)

---

*Maintained by [Sygitech](https://www.sygitech.com) — Managed Cloud Services for Engineering Teams*  
*Have a cost spike pattern not covered here? Open a Discussion.*
