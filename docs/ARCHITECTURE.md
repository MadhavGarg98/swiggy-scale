# Architectural Redesign - SwiftEats High-Throughput System

This document outlines the existing structural bottlenecks of the SwiftEats monolith and details a multi-tier, auto-scaled, and decoupled architecture capable of handling $720,000\text{ RPS}$ under peak load without failure.

---

## 1. Current Monolith Architecture Diagram

Below is the representation of our single-instance monolith annotated with the specific failures from the cascade analysis:

```
                                  10M Users
                                      │
                                      ▼ (Direct IP hit - No Load Balancer)
         ┌──────────────────────────────────────────────────────────┐
         │  Single Node.js Express Server                           │
         │  1 process · 1 CPU · 4GB RAM                             │
         │                                                          │
         │  - Serves static assets/images ──────────────────────────┼──► [⚠️ Failure 5: NIC Saturation]
         │                                                          │    (10Gbps bandwidth choked by images)
         │  - Menu API (restaurant listings)                        │
         │                                                          │
         │  - Promo Validation (TOCTOU checks) ─────────────────────┼──► [⚠️ Failure 4: Promo TOCTOU Race]
         │                                                          │    (No locking; oversells budget)
         │  - Sync Payment Processing (Razorpay/PayU) ──────────────┼──► [⚠️ Failure 2: Payment Amplification]
         │                                                          │    (DB connection held for 800ms)
         │  - Event Loop (Queues requests in memory) ───────────────┼──► [⚠️ Failure 3: Event Loop Saturation]
         │                                                          │    (CPU at 100%, OOM crash @ 12K RPS)
         └──────────────────────────────────────────────────────────┘
                                      │
                                      ▼ (Sync PostgreSQL client queries)
         ┌──────────────────────────────────────────────────────────┐
         │  Single PostgreSQL Database                              │
         │  max_connections = 100                                   │
         │  No Replicas · No Indexes                                │
         │                                                          │
         │  - Connections exhausted in 3s ──────────────────────────┼──► [⚠️ Failure 1: Pool Exhaustion]
         │                                                          │    (Blocked at 394 RPS)
         └──────────────────────────────────────────────────────────┘
```

---

## 2. Redesigned Multi-Tier Architecture Diagram

The new system removes single points of failure, edge-caches static content, decouples write paths from read paths, offloads database traffic using Redis, multiplexes connections, and processes payments asynchronously.

```
                                      10M Users
                                          │
                                          ▼
     ┌────────────────────────────────────────────────────────────────────────┐
     │                             CloudFront CDN                             │
     │   - Caches restaurant images & static menus at edge (TTL: 5m)          │
     │   - Offloads 99% of media requests (40TB/min -> 0TB to origin)         │
     └────────────────────────────────────────────────────────────────────────┘
                                          │ (Dynamic API REST Traffic Only)
                                          ▼
     ┌────────────────────────────────────────────────────────────────────────┐
     │                      Application Load Balancer (ALB)                   │
     │   - SSL Termination, HTTP/2 Support, Health Checks                     │
     │   - AWS WAF rate-limiting rules (e.g., max 100 reqs/minute per IP)    │
     └────────────────────────────────────────────────────────────────────────┘
                │                        │                        │
                ▼                        ▼                        ▼
     ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
     │  Node Instance [1] │   │  Node Instance [2] │   │  Node Instance [N] │
     │  (ECS / Auto-scale)│   │  (ECS / Auto-scale)│   │  (ECS / Auto-scale)│
     └────────────────────┘   └────────────────────┘   └────────────────────┘
           │            │                │            │           │            │
           │            └──────┐  ┌──────┘            │           │            │
           │                   ▼  ▼                   │           │            │
           │         ┌─────────────────────────┐      │           │            │
           │         │   Redis Cache Cluster   │◄─────┘           │            │
           │         │   - cache.r6g.large x3  │                  │            │
           │         │   - Menu caches (TTL 5m)│                  │            │
           │         │   - Atomic Promo Lock   │                  │            │
           │         └─────────────────────────┘                  │            │
           ▼                                                      ▼            ▼
     ┌────────────────────────────────────────────────────────────────────────┐
     │                       PgBouncer Connection Pooler                      │
     │   - Multiplexes 5000+ app connections to 100 physical DB connections   │
     └────────────────────────────────────────────────────────────────────────┘
                │                                                 │
                ▼ (Writes Only)                                   ▼ (Reads Only)
     ┌────────────────────┐                            ┌────────────────────┐
     │ PostgreSQL Primary │ ─── (Streaming Repl) ───►  │ PostgreSQL Read    │
     │ (writes only)      │                            │ Replica Cluster    │
     └────────────────────┘                            │ (r6g.large x2)     │
                │                                      └────────────────────┘
                ▼ (Async Payments)
     ┌────────────────────────────────────────────────────────────────────────┐
     │                       AWS SQS Payment Queue                            │
     │   - Message published in 1ms; orders saved in status "PENDING"        │
     └────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
     ┌────────────────────────────────────────────────────────────────────────┐
     │                       Payment Worker Fleet                             │
     │   - Pulls from SQS, executes external payment gateways asynchronously  │
     │   - Dead Letter Queue (DLQ) captures failed attempts for retries      │
     └────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Justification Table

Every component added to this architecture directly mitigates a specific failure point in the cascade.

| Component | Failure It Prevents | How It Prevents It |
| :--- | :--- | :--- |
| **CloudFront CDN** | **Failure 5:** Static Asset NIC Saturation | Intercepts static assets and images at edge locations close to users. Eliminates 40TB of image transfer requests, meaning they never hit the application servers. |
| **Application Load Balancer (ALB)** | Single Point of Failure (SPOF) & Compute Saturation | Distributes incoming traffic across auto-scaled instances. If a Node.js process crashes, ALB immediately halts routing to it (5-second health checks) and forwards requests to healthy containers. WAF protects against DDoS surges. |
| **Auto-scaling Node.js Fleet (ECS)** | **Failure 3:** Node.js Event Loop Saturation / Heap OOM | Spreads workload across 4 to 20 instances. When CPU exceeds 70% or request volume spikes, auto-scaling provisions extra instances, ensuring total capacity remains above peak RPS. |
| **Redis Cache Cluster (ElastiCache)** | **Failure 1:** DB Pool Exhaustion (on read queries) | Caches restaurant menu lists (`GET /restaurants` and `GET /restaurant/:id`) with a 5-minute TTL. Offloads 80%+ of database reads, dropping read query load on PostgreSQL from 350,000+ to nearly zero. |
| **Redis Atomic Locks (`SETNX`/`DECR`)** | **Failure 4:** Promo Code TOCTOU Race Condition | Ensures atomic checks and decrements on promo counters using Redis keys. Concurrent request threads see decrement results in real-time, preventing double-application and budget overruns. |
| **PgBouncer** | **Failure 1:** DB Pool Exhaustion (on write queries) | Acts as a connection proxy. Reuses and multiplexes a pool of 100 physical DB connections to support thousands of active Node.js application clients, avoiding PostgreSQL's connection rejection overhead. |
| **PostgreSQL Read Replicas** | Write/Read Database Contention | Isolates transactional writes from analytical or browsing reads. Order insertions (writes) go to the Primary instance; menu listings and order history (reads) scale out across two dedicated Read Replicas. |
| **AWS SQS Payment Queue & Workers** | **Failure 2:** Synchronous Payment Call Amplification | Converts synchronous payment gateway interactions into asynchronous tasks. When an order is placed, a message is published to SQS in 1ms and the DB connection is released. Payment workers poll SQS and execute payment logic out-of-band. |
