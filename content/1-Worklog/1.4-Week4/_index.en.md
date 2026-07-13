---
title: "Worklog Week 4"
date: 2026-06-25
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives:

* Understand and grasp the concepts of Serverless architecture on AWS.
* Get familiar with and practice AWS Lambda along with Serverless database services (DynamoDB, Aurora).
* Explore edge networking and security services: API Gateway, CloudFront, AWS WAF.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources |
| --- | --- | --- | --- | --- |
| Mon | - Learn about Serverless architecture on AWS:<br>+ Serverless concepts.<br>+ Pros/cons compared to traditional servers.<br>+ Explore AWS Lambda. | 25/06/2026 | 25/06/2026 | |
| Tue | - Learn about Serverless databases:<br>+ Amazon DynamoDB.<br>+ Amazon Aurora Serverless.<br>+ Use cases for each DB type. | 26/06/2026 | 26/06/2026 | |
| Wed | - Initialize Amazon Aurora.<br>- Try packaging the entire project into a single Lambda. | 27/06/2026 | 27/06/2026 | |
| Thu | - Learn about edge networking:<br>+ Amazon API Gateway.<br>+ Amazon CloudFront.<br>+ AWS WAF.<br>- Use CloudFormation to set up the edge network layer for the Lambda. | 28/06/2026 | 28/06/2026 | |

### Week 4 Results:

* Clearly understood the Serverless concept, how to optimize costs and reduce server management overhead.
* Grasped the working mechanism of AWS Lambda, trigger configuration, and execution permissions.
* Differentiated between DynamoDB and Aurora Serverless, and when to apply each type for a project.
* Successfully deployed an all-in-one Serverless model.
* Understood the role of edge network services:
  * API Gateway: Acts as the gateway to manage and route API calls to Lambda.
  * CloudFront: Supports content delivery to end users with the lowest latency.
  * AWS WAF: How the web application firewall works to block malicious traffic before it reaches the backend.
* Able to chain services together: `End-user -> CloudFront -> API Gateway -> Lambda -> DynamoDB`.
