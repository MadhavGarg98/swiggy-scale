# SRE Incident Runbook - SwiftEats High-Throughput Outages

This document is an actionable, step-by-step incident response playbook designed for on-call engineers. It provides automated thresholds for detection, a 30-second triage protocol, CLI commands for mitigation, and rollback rules.

---

## STEP 1 - DETECT: Alert Thresholds

The following CloudWatch alerts are configured. Alerts categorized as `CRITICAL` trigger immediate PagerDuty calls.

| Alert Metric | Alarm Condition | Period / Evaluation | Alert Type | Escalation Channel |
| :--- | :--- | :--- | :--- | :--- |
| `ALB 5xx Error Rate` | `HTTPCode_Target_5XX_Count` > 5% | 2 minutes / 1 datapoint | **CRITICAL** | PagerDuty On-Call |
| `Database Connection Count` | `DatabaseConnections` > 80% max | 1 minute / 1 datapoint | **WARNING** | Slack `#db-ops` |
| `EC2 CPU Utilization` | `CPUUtilization` > 80% average | 3 minutes / 3 datapoints | **WARNING** | Slack `#platform-alerts` (Triggers Auto-scaler) |
| `Redis Cache Memory` | `DatabaseMemoryUsagePercentage` > 75% | 5 minutes / 1 datapoint | **WARNING** | Slack `#cache-ops` |
| `SQS Queue Depth` | `ApproximateNumberOfMessagesVisible` > 10,000 | 2 minutes / 2 datapoints | **CRITICAL** | PagerDuty On-Call (Payment Core) |
| `P99 Latency` | `TargetResponseTime` > 2 seconds | 2 minutes / 2 datapoints | **CRITICAL** | PagerDuty On-Call |
| `Promo Budget Remaining` | `PromoBudgetRemaining` < 10% | 1 minute / 1 datapoint | **WARNING** | Slack `#marketing-alerts` |

---

## STEP 2 - TRIAGE: Identify Root Failure (30 Seconds)

When an outage is declared, the on-call engineer must execute these checks sequentially. The first check to return "RED" identifies the root cause.

```
                  [Outage Declared (HTTP 5xx Spiking)]
                                   │
                                   ▼
             Is DB Connections > 80% / Page timeouts?
             ├── YES ──────────────────────────────────► [Go to Step 3a: DB Pool Exhaustion]
             └── NO
                   │
                   ▼
             Is CPU Utilization > 80% on all ECS nodes?
             ├── YES ──────────────────────────────────► [Go to Step 3b: Node.js Compute Saturation]
             └── NO
                   │
                   ▼
             Is Redis Cache Miss Rate > 50%?
             ├── YES ──────────────────────────────────► [Go to Step 3c: Cache Invalidation / Miss Spike]
             └── NO
                   │
                   ▼
             Is SQS ApproximateNumberOfMessages > 50,000?
             ├── YES ──────────────────────────────────► [Go to Step 3d: Payment Queue Backlog]
             └── NO
                   │
                   ▼
             [Escalate to Platform Architect / Run Distributed Tracing analysis]
```

---

## STEP 3 - RESPOND: Component-Specific Actions

Follow the assigned recovery playbook based on the triage step completed above.

### 3a: DB Connection Pool Exhausted
*   **Root Cause:** Traffic spike or runaway connection hold times due to slow queries or connection leaks.
*   **Response Commands:**
    1.  Check current active database connections:
        ```bash
        aws rds-data execute-statement \
            --resource-arn "arn:aws:rds:ap-south-1:123456789012:cluster:swiggy-prod-db" \
            --secret-arn "arn:aws:secretsmanager:ap-south-1:123456789012:secret:db-creds" \
            --sql "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
        ```
    2.  If connections are maxed out, temporarily scale up the primary database instance size:
        ```bash
        aws rds modify-db-instance \
            --db-instance-identifier swiggy-prod-db \
            --db-instance-class db.r6g.4xlarge \
            --apply-immediately
        ```
    3.  If queries are hanging, terminate long-running processes:
        ```bash
        aws rds-data execute-statement \
            --resource-arn "arn:aws:rds:ap-south-1:123456789012:cluster:swiggy-prod-db" \
            --secret-arn "arn:aws:secretsmanager:ap-south-1:123456789012:secret:db-creds" \
            --sql "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'active' AND query_start < now() - interval '30 seconds';"
        ```
*   **Success Criteria:** RDS metric `DatabaseConnections` drops to $< 60\%$ within $5\text{ minutes}$.
*   **Responsibility:** Database Reliability Team | Channel: `#db-ops`

### 3b: Node.js Compute Saturation
*   **Root Cause:** Incoming traffic volumes exceed processing limits of active Node.js processes.
*   **Response Commands:**
    1.  Force scale the ECS task count manually to 20 instances:
        ```bash
        aws ecs update-service \
            --cluster swiggy-prod \
            --service api-service \
            --desired-count 20
        ```
    2.  Check container deployments progress:
        ```bash
        aws ecs describe-services \
            --cluster swiggy-prod \
            --services api-service \
            --query "services[0].deployments"
        ```
*   **Success Criteria:** ALB target response time drops to $< 100\text{ms}$ and 5xx rate drops to $< 0.1\%$ within $3\text{ minutes}$.
*   **Responsibility:** Platform Engineering Team | Channel: `#platform-alerts`

### 3c: Redis Cache Miss Rate High (Cache Stampede)
*   **Root Cause:** Eviction of menu listings or cluster node crash causing direct database read hit spikes.
*   **Response Commands:**
    1.  Retrieve Redis memory utilization and hits/misses statistics:
        ```bash
        aws elasticache describe-cache-clusters \
            --cache-cluster-id swiggy-redis \
            --show-cache-node-info
        ```
    2.  If cache was cleared/evicted, trigger manual warm-up script from the bastion server:
        ```bash
        aws ecs run-task \
            --cluster swiggy-prod \
            --task-definition swiggy-cache-warmer:latest \
            --network-configuration "awsvpcConfiguration={subnets=[subnet-12345],securityGroups=[sg-12345],assignPublicIp=ENABLED}"
        ```
*   **Success Criteria:** ElastiCache `CacheMisses` drops below $5\%$ and DB read queries drop to baseline levels.
*   **Responsibility:** API Core Team | Channel: `#api-dev`

### 3d: Payment Queue Backed Up
*   **Root Cause:** Payment workers crashed, or external gateway (Razorpay/PayU) latency spiked.
*   **Response Commands:**
    1.  Describe the count of pending messages in the SQS queue:
        ```bash
        aws sqs get-queue-attributes \
            --queue-url https://sqs.ap-south-1.amazonaws.com/123456789012/payment-queue \
            --attribute-names ApproximateNumberOfMessages
        ```
    2.  Scale payment worker task count to accelerate backlog consumption:
        ```bash
        aws ecs update-service \
            --cluster swiggy-prod \
            --service payment-worker-service \
            --desired-count 15
        ```
*   **Success Criteria:** SQS queue depth decreases consistently, target backlog latency under $10\text{s}$ within $10\text{ minutes}$.
*   **Responsibility:** Fintech Integrations Team | Channel: `#fintech-alerts`

---

## STEP 4 - ROLLBACK

Initiate a code rollback immediately if deployment was completed within the last 2 hours AND:
1.  Target HTTP 5xx error rate remains $> 20\%$ for more than 5 minutes.
2.  Compute scaling (Step 3b) does not stabilize response times.
3.  Root cause is isolated to application-level code modifications.

### Rollback Commands
Execute task definition rollback to the previous stable container build:
```bash
aws ecs update-service \
    --cluster swiggy-prod \
    --service api-service \
    --task-definition swiggy-api:PREVIOUS_STABLE_VERSION
```

> [!CAUTION]
> **CRITICAL WARNING:** Never rollback database schemas during an active outage. Rollbacks of migrations can lead to severe data corruption, record mismatch, or orphan order transactions. Always roll back application code to compatibility mode or push a hotfix forwards.

---

## STEP 5 - POSTMORTEM TEMPLATE

This template must be filled out and distributed via email/Slack within 24 hours of incident resolution.

```markdown
# INCIDENT POSTMORTEM - [INCIDENT-ID]

## Incident Summary
*   **Date:** [YYYY-MM-DD]
*   **Duration:** [X] minutes (from [HH:MM] to [HH:MM] UTC)
*   **Impacted Services:** [e.g., Order Placement, Menu browsing]
*   **Financial Loss Estimate:** ₹[X] crore
*   **Severity level:** SEV-1 (Global Outage)

## Timeline (All times in UTC/IST)
*   **[HH:MM:SS]** - [Event 1: Alert triggered, notification sent]
*   **[HH:MM:SS]** - [Event 2: On-call engineer acknowledged alert]
*   **[HH:MM:SS]** - [Event 3: Root cause identified as database pool exhaustion]
*   **[HH:MM:SS]** - [Event 4: Mitigation steps applied (database scaled up)]
*   **[HH:MM:SS]** - [Event 5: Incident resolved, metrics stabilized]

## Root Cause Analysis (5 Whys)
1.  Why was order placement unresponsive? Because the database pool was exhausted.
2.  Why was the database pool exhausted? Because connections were held open for 800ms waiting for payments.
3.  Why were connections held during payments? Because the payment logic was synchronous.
4.  Why was payment logic synchronous? [Identify engineering architecture decision]
5.  Why... [Deepest structural issue]

## What Worked & What Didn't
*   **What Went Well:** [e.g., Runbook triage mapped symptom to cause in under 30 seconds]
*   **What Went Poorly:** [e.g., Lack of autoscaling tasks caused delay in compute restoration]

## Action Items (Follow-up Tasks)
1.  Migrate POST /orders payment routing to asynchronous SQS pipeline | Owner: [Name] | Due: [YYYY-MM-DD]
2.  Optimize database query indexing on orders table | Owner: [Name] | Due: [YYYY-MM-DD]
3.  Configure auto-scaling triggers based on ALB target request volumes | Owner: [Name] | Due: [YYYY-MM-DD]
```
