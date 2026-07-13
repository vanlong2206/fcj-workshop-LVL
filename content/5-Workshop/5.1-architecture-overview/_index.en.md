---
title : "Introduction"
date : 2026-06-18
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Introduction to Game Backend Server Architecture

#### 5.1.1 Deployment Model Overview

Edge & Security Layer:
   * AWS Route 53: Domain name resolution and traffic routing.
   * Amazon CloudFront: Content Delivery Network (CDN) for optimizing transmission speed and reducing latency for clients.
   * AWS WAF: Attached directly to CloudFront to filter malicious requests, prevent DDoS, Botnet, and external attacks.

API & Sync Compute Layer:
   * Amazon API Gateway: Receives and routes RESTful endpoints to logic handler functions.
   * Compute Sync: A group of real-time processing functions that respond directly to clients, including core modules: Authentication, Inventory/Chest, Transaction...

Async Processing Layer:
   * Compute Async: A group of background processing functions for resource-intensive tasks such as World Progression and Reward System. After computation, data is pushed to the processing queue.
   * Amazon SQS FIFO: A queue that ensures the order of important data write events, preventing data state conflicts.
   * Amazon SQS DLQ: Captures and isolates failed error messages for later processing.
   * Amazon CloudWatch: Monitors DLQ status to send immediate alerts when exceptions or crashes occur.

Database Layer:
   * Amazon Aurora DB: Primary database management system with auto-scaling capability.
   * AWS Backup & Amazon S3: Configures automated snapshot and periodic data backup processes.

Automation & Maintenance Layer:
   * Amazon EventBridge: Triggers scheduled or event-based Lambda functions:
     * API Infrastructure: Calls `start_maintenance` and `stop_maintenance` functions to update system status via API Gateway stage variables using AWS SDK.
     * Database Maintenance: Periodically triggers `vacuum & analyze` function to optimize DB performance and `reset_daily` function to refresh world progression daily.

![overview](/images/diagram.png)
<div align="center"><i>Figure 5.1.1: Overall system architecture diagram.</i></div>
