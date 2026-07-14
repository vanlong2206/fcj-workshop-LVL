---
title: "Workshop"
date: 2026-06-18
weight: 5
chapter: false
pre: " <b> 5. </b> "
---
# Research and Deployment of Server Systems for Online Games

#### Overview

The server system plays a core role in managing state, synchronizing real-time data, and securing player information. For games with continuous interaction frequency, optimizing infrastructure for minimal latency and flexible load capacity is a decisive factor for user experience.

In this topic, we will study how to design, configure, and deploy a comprehensive server architecture on the AWS cloud platform, combining optimal storage solutions to ensure data safety and optimize operational costs.

The system separates data flow and processing into two strategic models depending on the traffic characteristics:

+ **Synchronous Processing (Compute Layer)** — Uses intermediary servers coordinated with a load balancing system to route text-based API queries, or transitions to a serverless architecture to auto-scale according to the number of players in real time.
+ **Multi-tier Storage (Database Layer)** — Hierarchical data through a high-security relational database management system placed in a private network partition, combined with a cache layer to reduce load and speed up response times for repetitive read/write operations.

#### Content

1. [Architecture Overview](5.1-architecture-overview/)
2. [Infrastructure Setup and Management](5.2-management/)
3. [Database Setup and Backup](5.3-database/)
4. [Project Architecture](5.4-architecture-overview/)
5. [Amazon API Gateway Setup](5.5-API-Gateway/)
6. [Amazon SQS FIFO](5.6-sqs-fifo/)
7. [AWS DLQ &amp; AWS CloudWatch](5.7-sqs-dlq/)
8. [AWS EventBridge](5.8-eventbridge/)
9. [AWS Backup](5.9-aws-backup/)
10. [Verification and Testing](5.10-verification-and-testing/)
11. [Resource Clean Up](5.11-clean-up/)
12. [Challenges and Future Development](5.12-challenges-and-future/)
