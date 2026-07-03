---
title : "Compute Sync — Real-Time Processing with Lambda"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

#### Synchronous Compute with AWS Lambda

In this section, you will create **an AWS Lambda function** that processes requests **synchronously** via **Amazon API Gateway**. When a client sends a request, API Gateway invokes the Lambda function directly and waits for the response before replying to the client. This pattern is ideal for real-time workloads such as REST APIs, CRUD operations, and lightweight data transformations.

The Lambda function will interact with **Amazon S3** and **Aurora Database** to perform business logic, demonstrating how serverless compute can replace traditional EC2-based backends.

#### Content

- [Create Lambda function](5.6.1-create-function/)
- [Configure API Gateway integration](5.6.2-configure-integration/)
- [Test synchronous invocation](5.6.3-test-invocation/)
