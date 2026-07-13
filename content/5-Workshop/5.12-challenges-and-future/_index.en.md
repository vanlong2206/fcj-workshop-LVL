---
title: "Challenges and Future Development"
date: 2026-07-13
weight: 12
chapter: false
pre: " <b> 5.12. </b> "
---
#### 5.12.1 Challenges Encountered

* Designing a suitable Serverless architecture across AWS services (API Gateway, Lambda, SQS, Aurora, Backup...) increased system complexity.
* Managing cross-service access permissions (IAM Role, IAM Policy) was complex and prone to `AccessDenied` errors.
* Ensuring data consistency with concurrent update requests — risk of data conflicts, transaction overwrites, or loss of data accuracy in the game system.
* Debugging and error handling in a Serverless environment was difficult due to the lack of fixed servers.

#### 5.12.2 Future Development

* Expand the system with an **Event-Driven Architecture**, using EventBridge to connect microservices instead of direct communication.
* Implement **CI/CD** with GitHub Actions combined with Serverless Framework for automated testing and AWS infrastructure deployment.
* Add **AWS WAF** and **Amazon Cognito** to enhance API Gateway security and user authentication.
* Enhance monitoring with **AWS X-Ray** and build CloudWatch Dashboards to track system-wide performance.
* Set up **Cross-Region Backup**, **Cross-Account Backup**, and **Backup Vault Lock** to improve Disaster Recovery capabilities.
* Optimize costs by applying Backup Lifecycle policies, optimizing Lambda execution time, and using service plans suitable for traffic load.
* Expand the architecture to multiple environments (**Development**, **Staging**, **Production**) with independent configurations and resources for easier testing and operations.
