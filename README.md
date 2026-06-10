# SwiftEats - Scale Simulation & Incident Architecture

This repository contains SRE-grade analysis, system design architectures, AWS pricing forecasts, and runbooks prepared for the scale transition of SwiftEats (a Swiggy-like food delivery platform) from a fragile monolith to a highly resilient, multi-tier distributed architecture.

## The Scenario

*   **The Event:** India vs. Pakistan World Cup Final (8:00 PM IST).
*   **The Action:** SwiftEats sends a push notification offering a "50% off orders tonight!" promo code to **180 million users**.
*   **The Load:** An expected **10 million to 14.4 million concurrent users** click the notification within a 60-second window, generating up to **720,000 requests per second (RPS)**.
*   **The Core Monolith:** A single Node.js Express process (1 CPU, 4GB RAM) talking to a single PostgreSQL database instance (`max_connections = 100`) without a caching layer, CDN, or load balancing.
*   **The Impact:** Instantaneous failure cascade resulting in ₹189 crore ($22.7M USD) in lost orders, and a drop in App Store ratings from 4.3 to 1.8.

---

## Repository Documentation Index

| Document | Purpose | Contents |
| :--- | :--- | :--- |
| 📄 [FAILURE-CASCADE.md](file:///c:/Users/Madhav%20Garg/OneDrive/Documents/Projects/swigy/docs/FAILURE-CASCADE.md) | Failure Cascade Analysis | Traffic math, component limits, pool exhaustion formula, triggers, and crash timeline. |
| 📄 [ARCHITECTURE.md](file:///c:/Users/Madhav%20Garg/OneDrive/Documents/Projects/swigy/docs/ARCHITECTURE.md) | System Redesign | Monolith bottlenecks diagram, multi-tier system redesign diagram, and component justification table. |
| 📄 [COST-ESTIMATE.md](file:///c:/Users/Madhav%20Garg/OneDrive/Documents/Projects/swigy/docs/COST-ESTIMATE.md) | AWS Financial Model | Steady-state baseline cost, 4-hour peak event autoscaling surge pricing, and business ROI analysis. |
| 📄 [RUNBOOK.md](file:///c:/Users/Madhav%20Garg/OneDrive/Documents/Projects/swigy/docs/RUNBOOK.md) | On-Call Incident Playbook | 6 SRE alerts, 30s triage decision tree, CLI recovery commands, rollback limits, and postmortem template. |

---

## Key Findings & Surprising Numbers

1.  **DB Pool Exhaustion Trigger:** It takes only **394 RPS** to completely saturate the PostgreSQL connection pool of 100. At 720,000 RPS, the DB pool fails in under 1 second.
2.  **Synchronous Payment Amplification:** Decoupling payment gateways from database locks reduces database connection hold times by **80x** (from an average of 800ms down to 10ms for SQS writes), preventing immediate pool depletion.
3.  **Media Bandwidth Saturation:** Serving restaurant images directly from Node.js requires **5.33 Tbps** of network bandwidth, which chokes the server's 10Gbps network interface (NIC) in under 15 seconds, making the APIs completely unreachable.
4.  **Disproportionate Outage Costs:** The cost to run our fully resilient, multi-tier AWS architecture for an **entire year** ($\approx \$12,752\text{ USD}$ or $₹10.6\text{ Lakhs}$) is less than the revenue lost in just **1.5 seconds of database downtime** during peak hours ($₹10.5\text{ Lakhs}$).

---

## Redesigned Architecture Overview

The redesigned system removes the single point of failure by inserting a **CloudFront CDN** at the edge to serve static assets and cached restaurant menus, offloading 99% of media requests from origin. Dynamic API traffic passes through an **Application Load Balancer (ALB)** with WAF rate limiting to a fleet of **auto-scaled Node.js instances (ECS)**. Database access is scaled out by separating writes (handled by a PostgreSQL Primary instance behind a **PgBouncer connection pooler**) from reads (scaled using two **PostgreSQL Read Replicas**), while **ElastiCache Redis** caches volatile query listings and implements atomic locks to prevent promo race conditions. Payment processing is decoupled using **Amazon SQS asynchronous message queues** polled by specialized payment workers.

## Tech Stack Context

*   **Backend Runtime:** Node.js (Express, stateless REST APIs)
*   **Primary Database:** PostgreSQL (Primary-Replica Cluster)
*   **Connection Pooler:** PgBouncer
*   **Caching & State Locks:** Redis (AWS ElastiCache Cluster)
*   **Traffic Routing:** AWS Application Load Balancer (ALB)
*   **Asset & Edge Caching:** AWS CloudFront CDN
*   **Message Broker:** AWS Simple Queue Service (SQS)