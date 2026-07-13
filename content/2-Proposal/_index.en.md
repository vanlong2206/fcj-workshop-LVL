---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
## AWS Serverless Solution for This Project: Game Backend API

### 2.1 Executive Summary

The Game Backend API project is for a 2D RPG game with a Node.js + TypeScript + Express 5 backend and PostgreSQL/Aurora RDS database. The AWS Serverless solution migrates from a monolith to **microservices serverless** with 6 Lambda APIs + 5 SQS Consumers + 1 Maintenance Lambda, reducing costs, improving scalability, and enabling flexible deployment.

### 2.2 Problem Statement

**Operational Cost:**
The current monolith architecture runs on a server 24/7, even when no players are active. This wastes significant compute resources. The server must be provisioned for peak load but runs idle during off-peak hours, leading to suboptimal operational costs.

**Scalability:**
When the number of players spikes (game events, ads, holiday seasons), the monolith cannot scale individual parts independently. For example, only Auth or Economy features are overloaded, but the entire system must be scaled. This is both expensive and inefficient. The monolith architecture also does not support flexible auto-scaling per domain.

**Deployment and Maintenance:**
Every feature update or bug fix requires redeploying the entire application. A minor bug in the Auth module can bring down the whole game. CI/CD pipelines are more complex and deployment takes longer due to rebuilding and restarting the entire system. Independent rollback per feature is not possible.

**Async Processing:**
Critical tasks like currency updates (earn/spend), inventory changes (add/remove item), gift code redemption, and save data persistence are currently handled synchronously within the request-response cycle. If the server crashes mid-operation, data may be lost or become inconsistent. There is no retry mechanism or queue to guarantee successful processing.

**Backup and Disaster Recovery:**
Game data — accounts, currency, inventory, save data — is the most valuable asset. There is currently no automated backup strategy or clear disaster recovery (DR) plan. If the database encounters issues (disk failure, replication errors, attacks), player data could be permanently lost.

### 2.3 Solution Architecture

![1783961736805](image/_index.vi/1783961736805.png)

**Data Flow Overview:**

Client (game app) sends requests through API Gateway, which routes to the corresponding Lambda per domain. Lambda APIs handle business logic and write data to Aurora RDS PostgreSQL. For critical tasks requiring data integrity (earn/spend currency, add/remove item, redeem gift code, distribute stats, upload save data), Lambda APIs send messages to SQS FIFO queues. SQS Consumer Lambdas process messages from queues and write to the database, ensuring async processing with retry capability.

**Component Details:**

API Gateway serves as the single entry point receiving client requests, handling authentication (JWT), rate limiting, and routing to the correct Lambda. 6 Lambda APIs are deployed independently, each wrapping an Express app with `@vendia/serverless-express`, running on Node.js 20, handling a specific game domain. These Lambdas scale independently based on request volume per domain.

5 SQS FIFO queues ensure messages are processed in order (FIFO) and not lost. Each queue has its own DLQ (Dead Letter Queue) to collect failed messages after 3 retries, with CloudWatch alarms for timely alerts. SQS Consumer Lambdas process messages in batches (max 10 messages/batch) with partial failure handling (SGDBatchResponse).

Aurora RDS Serverless v2 PostgreSQL is the primary database, auto-scaling ACUs based on load, supporting IAM authentication for security. TypeORM auto-creates/updates schemas on startup (synchronize: true). There are 18 entities divided into 5 domains (User, Forum, GiftCode, Game, System).

EventBridge acts as a scheduler with 4 rules: enable maintenance mode (Monday 10:00 UTC), disable maintenance mode (Monday 12:00 UTC), vacuum analyze and reindex all tables (3:00 AM daily), daily reset (midnight — reset stamina, clean expired codes, logs, stale data). The Maintenance Lambda receives events from EventBridge and executes corresponding tasks.

AWS Backup handles automatic daily database backups (5:00 UTC, 14-day retention) and weekly backups (Sunday 5:00 UTC, 56-day retention).

Shared code is packaged via npm workspace `@gameapi/shared`, including: 18 TypeORM entities, 3 middlewares (auth, admin, maintenance), SQS producer with 8 static methods, utilities (JwtHelper, PasswordHasher, TimeHelper, ItemGenerationHelper, logger), services (GiftCodeService, GameLogicValidator), and CloudWatch metrics helper.

### 2.4 Technical Implementation

**6 Lambda APIs** divided by domain — Auth (register/login/dashboard), Economy (balance/earn/spend), Inventory (item CRUD + storage), Transaction (shop + gift code), Progression (stats + farming), Loot-Reward (leaderboard + forum + save data + game data).

**5 SQS FIFO queues** — economy, inventory, giftcode, stats, save-data. Each queue has its own DLQ with CloudWatch alarm. Messages are processed asynchronously by corresponding consumer Lambdas.

**Database** — Aurora RDS PostgreSQL with TypeORM (synchronize: true). 18 entities across 5 domains. Supports IAM authentication for production, password for local dev.

**Security** — JWT for APIs, admin secret for admin endpoints, rate limiting, IAM authentication.

**Shared code** — npm workspace `@gameapi/shared` containing models, middlewares, utils, SQS producer, shared services.

**Maintenance** — 4 EventBridge rules: enable/disable maintenance (Monday), vacuum analyze (3:00 AM), daily reset (midnight).

**Deployment** — Docker Compose for local dev, Serverless Framework + esbuild for AWS production.

### 2.5 Roadmap

Week 1-2: Survey and solution design.
Week 3-4: Set up AWS infrastructure (RDS, API Gateway, SQS, EventBridge, IAM).
Week 5-6: Implement Lambda Auth + Economy + corresponding consumers.
Week 7-8: Implement Lambda Inventory + Transaction + corresponding consumers.
Week 9-10: Implement Lambda Progression + Loot-Reward + corresponding consumers.
Week 11: Implement Maintenance Lambda, backup, monitoring.
Week 12: Integration testing, load testing, cutover, go-live.

### 2.6 Budget Estimation

Monthly AWS service costs (estimated for ~1000 players):

- **AWS Lambda (12 functions):** $50-200/month — based on request count and execution time. 6 API Lambdas + 5 Consumer Lambdas + 1 Maintenance Lambda. 1 million requests/month free, then $0.20/million requests.
- **API Gateway (REST API):** $30-100/month — based on request count. 1 million requests/month free for first 12 months.
- **Aurora RDS Serverless v2 (PostgreSQL):** $50-150/month — based on ACU (Aurora Capacity Unit). Starts at 2 ACU, auto-scales.
- **SQS FIFO (5 queues + 5 DLQs):** $5-15/month — FIFO $0.50/million requests, DLQ storage $0.023/GB/month.
- **EventBridge (4 rules):** $5-10/month — $1/million events, 4 daily schedule rules.
- **CloudWatch Logs + Metrics:** $10-30/month — Lambda logs, custom metrics, DLQ alarms.
- **AWS Backup (daily + weekly):** $10-20/month — database backup storage + 14-56 day retention.

**Total: $160-525/month.** For small scale under 1000 players, costs could be around ~$200/month.

### 2.7 Risk Assessment

**Lambda Cold Start:**
When no requests come for an extended period, Lambda resources are reclaimed. The first subsequent request will be slow due to runtime initialization and database connection setup (cold start). Medium impact — players may experience slight lag on the first request. Mitigated by using Provisioned Concurrency for critical Lambdas (Auth and GameData) to keep execution environments warm.

**FIFO Queue Throughput Limit:**
SQS FIFO limits throughput to 3000 TPS — exceeding this causes message throttling. Low impact since for under 1000 concurrent players, this throughput is sufficient. For future expansion, more queues and account-based sharding can be added.

**Data Loss or Duplicate Messages:**
SQS FIFO guarantees exactly-once processing, but the consumer may crash after processing but before acknowledging, leading to duplicates. Low impact — handlers should be designed idempotent. DLQ with maxReceiveCount=3 ensures no messages are ever lost, and operations teams are alerted via CloudWatch alarm when DLQ has messages.

**Database Connection Pool Exhaustion:**
Each Lambda instance creates its own connection pool to the database. If many Lambda instances run concurrently, connections may exceed RDS limits. Medium-high impact. Mitigated by limiting pool size per Lambda (max 2-5 connections), using RDS Proxy for centralized connection pool management, and leveraging IAM authentication to avoid storing passwords.

**Sudden AWS Cost Spikes:**
When a game goes viral or has a major event, request spikes drive up Lambda, API Gateway, and database costs. High impact — could result in unexpected AWS bills. Mitigated by setting up AWS Budget Alerts at 3 thresholds (50%, 80%, 100%), API Gateway usage plans, rate limiting, and CloudWatch dashboard monitoring.

**Lambda Timeout with Large Data:**
Player save data can be very large (many items, plots, transaction history), causing Lambda timeout (default 30s, max 900s). Medium impact. Mitigated by increasing Lambda timeout appropriately, splitting save data into multiple parts, or handling heavy tasks async via SQS.

**AWS Dependency:**
The entire system runs on AWS — if AWS has a regional outage (ap-southeast-1), the game stops completely. High impact but low probability. Mitigated by multi-AZ RDS configuration, cross-region DR backup plan, and detailed recovery documentation.

### 2.8 Expected Results

**Reduced Operational Costs:**
With Lambda pay-per-use, compute costs only incur when there are player requests. No more wasted resources from running a server 24/7. Expected 40-60% cost savings compared to a fixed-server monolith, especially during early stages with low player counts.

**Flexible Scalability:**
Each domain scales independently based on actual load. Auth Lambda can scale to 1000 instances while Inventory Lambda needs only 10. API Gateway auto-scales by request volume. Aurora Serverless v2 automatically adjusts ACUs based on database load. No more system-wide bottlenecks.

**Fast and Safe Deployment:**
Deploy each Lambda independently without affecting other domains. Deployment time drops from 10-15 minutes (monolith) to 1-2 minutes (per Lambda). Independent rollback if issues arise. Simpler CI/CD with Serverless Framework.

**Reliable Async Processing:**
FIFO queues guarantee ordered processing with no message loss. DLQ + CloudWatch alarm enables timely detection and handling of failed messages. Automatic retry up to 3 times. Idempotent handlers prevent duplicates caused by retries.

**Production Ready:**
AWS Backup automatic (daily + weekly) with clear retention policies. CloudWatch monitoring and alarms for all components (DLQ, Lambda errors, API Gateway 5xx, database connections). Multi-layer security: JWT, admin secret, rate limiting, IAM authentication. Maintenance mode for controlled maintenance without affecting data.

**Smooth Migration:**
Monolith can run in parallel during cutover. Domains can be migrated gradually from monolith to Lambda without stopping the game. Quick rollback by pointing API Gateway back to the monolith if issues are detected.
