---
title : "Compute Async — Asynchronous Processing with SQS"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

#### Asynchronous Processing with Amazon SQS

In this section, you will set up **an Amazon SQS queue** to decouple microservices and handle **asynchronous workloads**. When a request is submitted, the producer sends a message to the queue and immediately returns a confirmation. A consumer (Lambda function or EC2 instance) polls the queue and processes the message independently.

This pattern is ideal for background jobs, batch processing, task offloading, and building resilient, loosely-coupled architectures.

#### Content

- [Create SQS queue](5.7.1-create-queue/)
- [Configure Lambda consumer](5.7.2-configure-consumer/)
- [Test asynchronous processing](5.7.3-test-async/)
