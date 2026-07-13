---
title: "Worklog Week 5"
date: 2026-06-01
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives:

* Learn resource optimization techniques and system load management on AWS.
* Practice distributed system design thinking: break down the project into independent AWS Lambda functions.
* Understand the working mechanisms of supporting optimization services: Amazon RDS Proxy, Amazon ElastiCache, Amazon SQS, and Elastic Load Balancing.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources / Notes |
| --- | --- | --- | --- | --- |
| Mon | - Analyze project structure and design Microservices:<br>+ Break down application logic into independent AWS Lambda functions.<br>+ Define boundaries and how functions communicate with each other. | 01/06/2026 | 01/06/2026 | |
| Tue | - Learn about Database connection optimization:<br>+ Connection exhaustion issue when using Lambda with RDS.<br>+ Connection Pooling solution using Amazon RDS Proxy. | 02/06/2026 | 02/06/2026 | |
| Wed | - Learn about read speed optimization:<br>+ How Amazon ElastiCache works.<br>+ Basic differences between Redis and Memcached.<br>+ Apply caching to reduce Database load. | 03/06/2026 | 03/06/2026 | Theory differs greatly from practice; applying Cache for asynchronous processing is nearly infeasible and impractical for current needs. |
| Thu | - Learn about asynchronous processing and Decoupling:<br>+ Amazon SQS.<br>+ How to use SQS as a buffer between services to avoid system overload. | 04/06/2026 | 04/06/2026 | |
| Fri | - Learn about Load Balancing (for EC2):<br>+ Elastic Load Balancing basics.<br>+ Distinguish between Application Load Balancer and Network Load Balancer. | 05/06/2026 | 05/06/2026 | |

### Week 5 Results:

* Developed the mindset of transitioning from Monolithic to Serverless Microservices architecture by decomposing the project into multiple specialized AWS Lambda functions.
* Clearly understood the root cause of database connection bottlenecks when Lambda scales up suddenly, and how RDS Proxy maintains and reuses connections to protect the database.
* Understood how to improve application response speed and reduce primary database load by storing frequently accessed data in Amazon ElastiCache.
* Grasped the concept of decoupling system components using Amazon SQS. Learned how to design a system that withstands sudden traffic spikes by offloading heavy processing tasks into a queue for asynchronous handling.
* Differentiated the network layers of ALB and NLB, enabling appropriate Load Balancer selection for distributing traffic evenly across targets.
