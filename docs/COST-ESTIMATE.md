# AWS Cost Estimate & Infrastructure ROI - SwiftEats

This document presents the detailed financial breakdown of the redesigned AWS infrastructure, comparing steady-state baseline costs to the surge costs incurred during a peak 4-hour World Cup Final window, concluding with a business case ROI analysis.

---

## 1. Baseline Architecture Cost (~100,000 DAU)

The baseline architecture handles typical daily operations with a steady-state fleet. Calculations assume a 30-day month ($720\text{ hours}$).

### EC2 Application Fleet (Node.js API & Workers)
*   **Instance Type:** `t3.medium` (2 vCPU, 4GB RAM)
*   **Hourly Rate:** $\$0.0416$
*   **Quantity:** $4$ instances (highly-available, spread across AZs)
*   **Formula:** $\text{Hourly Rate} \times \text{Hours} \times \text{Quantity}$
*   **Calculation:**
    $$\$0.0416 \times 720\text{ hours} \times 4 = \$119.81\text{ per month}$$

### RDS PostgreSQL Primary
*   **Instance Type:** `db.r6g.large` (2 vCPU, 16GB RAM)
*   **Hourly Rate:** $\$0.182$
*   **Quantity:** $1$ (Primary write instance)
*   **Formula:** $\text{Hourly Rate} \times \text{Hours} \times \text{Quantity}$
*   **Calculation:**
    $$\$0.182 \times 720\text{ hours} \times 1 = \$131.04\text{ per month}$$

### RDS PostgreSQL Read Replicas
*   **Instance Type:** `db.r6g.large` (2 vCPU, 16GB RAM)
*   **Hourly Rate:** $\$0.182$
*   **Quantity:** $2$ replicas (distributing listing & browsing reads)
*   **Formula:** $\text{Hourly Rate} \times \text{Hours} \times \text{Quantity}$
*   **Calculation:**
    $$\$0.182 \times 720\text{ hours} \times 2 = \$262.08\text{ per month}$$

### ElastiCache Redis Cluster
*   **Instance Type:** `cache.r6g.large` (2 vCPU, 13GB RAM)
*   **Hourly Rate:** $\$0.166$
*   **Quantity:** $3$ cluster nodes (1 Primary, 2 Replicas)
*   **Formula:** $\text{Hourly Rate} \times \text{Hours} \times \text{Quantity}$
*   **Calculation:**
    $$\$0.166 \times 720\text{ hours} \times 3 = \$358.56\text{ per month}$$

### Application Load Balancer (ALB)
*   **Base Cost:** $\$16.20$ flat rate per month
*   **LCU Cost:** Estimated average of $5$ Load Balancer Capacity Units (LCUs) per hour ($\$0.008$ per LCU-hour)
*   **Formula:** $\text{Base Cost} + (\text{LCU Rate} \times \text{LCU Count} \times \text{Hours})$
*   **Calculation:**
    $$\$16.20 + (\$0.008 \times 5 \times 720) = \$16.20 + \$40.00 = \$56.20\text{ per month}$$

### Amazon CloudFront CDN (Steady-State Traffic)
*   **Data Transfer Rate:** $\$0.0085\text{ per GB}$ (standard AWS global outbound rate)
*   **Volume:** $10\text{ TB}$ ($10,000\text{ GB}$) outbound transfer per month
*   **Formula:** $\text{Rate per GB} \times \text{GB Transfer Volume}$
*   **Calculation:**
    $$\$0.0085 \times 10,000 = \$85.00\text{ per month}$$

### Amazon SQS (Payment & Event Queue)
*   **Request Rate:** $\$0.40\text{ per million requests}$
*   **Volume:** $\sim 1\text{M messages/day} = 30\text{M messages/month}$
*   **Formula:** $\text{Rate per Million} \times \text{Millions of Messages}$
*   **Calculation:**
    $$\$0.40 \times 30 = \$12.00\text{ per month}$$

### Baseline Monthly Total
$$\text{Baseline Cost} = \$119.81 + \$131.04 + \$262.08 + \$358.56 + \$56.20 + \$85.00 + \$12.00 = \$1,024.69\text{ / month}$$

---

## 2. Peak Event Scaling Cost (World Cup Final Surge - 4 Hours)

To handle the surge of $10\text{M}$ concurrent users, we upscale compute and database capacities during a 4-hour peak window.

### EC2 Auto-Scaled Fleet (Peak Surge)
*   **Surge Instance Type:** `t3.2xlarge` (8 vCPU, 32GB RAM) to process complex logic and routing
*   **Hourly Rate:** $\$0.3328$
*   **Quantity:** $20$ additional instances added
*   **Window:** $4\text{ hours}$
*   **Formula:** $\text{Hourly Rate} \times \text{Hours} \times \text{Quantity}$
*   **Calculation:**
    $$\$0.3328 \times 4\text{ hours} \times 20 = \$26.62\text{ extra}$$

### RDS Primary Upgrade (Surge Window)
*   **Surge Database Type:** `db.r6g.4xlarge` (16 vCPU, 128GB RAM) to sustain order placement transaction volume
*   **Hourly Rate:** $\$1.027$
*   **Quantity:** $1$ database upgraded from `db.r6g.large`
*   **Window:** $4\text{ hours}$
*   **Formula:** $\text{Hourly Rate} \times \text{Hours}$
*   **Calculation:**
    $$\$1.027 \times 4\text{ hours} = \$4.11\text{ extra}$$

### CloudFront Surge Data Transfer
*   **Surge Volume:** $50\text{ TB}$ ($50,000\text{ GB}$) of static content served in 4 hours
*   **Rate:** $\$0.0085\text{ per GB}$
*   **Formula:** $\text{Rate per GB} \times \text{GB Surge Volume}$
*   **Calculation:**
    $$\$0.0085 \times 50,000 = \$425.00\text{ extra}$$

### Peak Night Extra Cost (4-Hour Surge Total)
$$\text{Peak Surge Cost} = \$26.62 + \$4.11 + \$425.00 = \$455.73\text{ extra}$$

---

## 3. Business Justification & ROI

When validating infrastructure expenditures, SRE leaders compare operational maintenance costs to the financial impact of database downtime.

### Outage Financial Metrics
*   **Lost Orders Rate:** $₹4.2\text{ crore}$ ($\$504,000\text{ USD}$) in lost orders per minute.
*   **Average Mean Time to Recovery (MTTR) for Monolith Collapse:** $45\text{ minutes}$ (triage, DB restart, connection flush).
*   **Total Revenue Loss for 45-Minute Outage:**
    $$\text{Outage Cost} = 45\text{ minutes} \times ₹4.2\text{ crore} = ₹189\text{ crore} \approx \$22,700,000\text{ USD}$$

### Return on Investment (ROI) Calculation
We calculate the ROI of investing in the baseline system for a full year plus running the World Cup Final scale event:

*   **Total Annual Infrastructure Cost:**
    $$\text{Annual Cost} = (\text{Baseline Cost} \times 12\text{ months}) + \text{Peak Surge Cost}$$
    $$\text{Annual Cost} = (\$1,024.69 \times 12) + \$455.73 = \$12,296.28 + \$455.73 = \$12,752.01\text{ USD} \approx ₹10.6\text{ Lakhs}$$

$$\text{ROI} = \frac{\text{Averted Losses} - \text{Annual Infrastructure Cost}}{\text{Annual Infrastructure Cost}}$$
$$\text{ROI} = \frac{\$22,700,000 - \$12,752.01}{\$12,752.01} \approx 1,779\times\text{ or } 177,900\%$$

> [!IMPORTANT]
> The financial loss from just **1.5 seconds** of database downtime ($₹10.5\text{ Lakhs}$) completely exceeds the cost of running this highly available, auto-scaled, multi-tier system for an **entire year** ($₹10.6\text{ Lakhs}$). Investing in this architecture is a fundamental business necessity, protecting brand value, user retention (App Store rating), and millions in transaction revenues.
