---
title: "Worklog Week 7"
date: 2026-06-15
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives:

* Build an overall system architecture diagram for the practical project.
* Review, monitor, and cross-evaluate peer assignments/projects to extract optimal solutions for the team project.
* Survey, sketch, and analyze pros/cons of various architecture variants: Traditional Server, Serverless, and Hybrid architectures.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources |
| --- | --- | --- | --- | --- |
| Mon | - Review peer project evaluations:<br>+ Analyze architecture models used by other groups.<br>+ Team discussion to select suitable ideas for the shared project. | 15/06/2026 | 15/06/2026 | |
| Tue | - Design Traditional Server architecture:<br>+ Draw diagram: EC2 + RDS + Load Balancer.<br>+ Evaluate scalability and operational costs. | 16/06/2026 | 16/06/2026 | |
| Wed | - Design Hybrid architecture:<br>+ Model 1: EC2 + Supabase + Supabase Pooling.<br>+ Model 2: EC2 + RDS + RDS Proxy + Load Balancer. | 17/06/2026 | 17/06/2026 | |
| Thu | - Design Pure Serverless architecture:<br>+ Model 1: Lambda + AuroraDB + RDS Proxy.<br>+ Model 2: Lambda + AuroraDB + ElastiCache + RDS Proxy. | 18/06/2026 | 18/06/2026 | |
| Fri | - Synthesize and finalize the practical project diagram:<br>+ Compare latency, cost, and maintainability across variants.<br>+ Finalize the architecture diagram for the team project. | 19/06/2026 | 19/06/2026 | |

### Week 7 Results:

* Completed the peer review and cross-evaluation process, gaining fresh perspectives on resource allocation and data flow handling to consider for the team's practical project.
* Finished sketching and visualizing 5 infrastructure architecture variants:
  * Traditional architecture: `EC2 + RDS + Load Balancer` - Easy to approach and visualize, but incurs fixed maintenance costs and requires careful Auto Scaling setup.
  * Hybrid architecture 1: `EC2 + Supabase + Supabase Pooling` - Combines traditional EC2 computing with Backend-as-a-Service (Supabase), leveraging Supabase's built-in pooling to reduce direct connections to Postgres.
  * Hybrid architecture 2: `EC2 + RDS + RDS Proxy + Load Balancer` - Solves the database connection bottleneck of the traditional model when EC2 scales out by using RDS Proxy as a buffer layer.
  * Serverless architecture 1: `Lambda + AuroraDB + RDS Proxy` - Optimizes pay-per-use costs. RDS Proxy plays a critical role here in maintaining the connection pool, preventing Lambda from creating too many new connections that could crash AuroraDB during traffic spikes.
  * Serverless architecture 2: `Lambda + AuroraDB + ElastiCache + RDS Proxy` - The most comprehensive model. Adding ElastiCache as a cache layer significantly reduces query rounds to the DB, minimizing response latency. However, the cost exceeds what is necessary and is not suitable for the current project scale.
* Sketching multiple variants gave the team the clearest picture of the trade-offs between Maintainability, Performance, and Cost. Based on these drafts, the team had sufficient basis to finalize the overall system architecture for the project.
