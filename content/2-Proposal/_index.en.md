---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

In this section, you need to summarize the workshop contents you **plan** to do.

# IoT Weather Platform for Lab Research
## Unified AWS Serverless Solution for Real-time Weather Monitoring

### 1. Executive Summary
The IoT Weather Platform is designed for the *ITea Lab* group in Ho Chi Minh City to enhance weather data collection and analysis capabilities. The platform supports up to 5 weather stations, scalable to 10–15 stations, using Raspberry Pi edge devices with ESP32 sensors communicating via MQTT. The platform leverages AWS Serverless services to provide real-time monitoring, predictive analytics, and cost savings, with limited access for 5 lab members through Amazon Cognito.

### 2. Problem Statement
*Current Problem*
Current weather stations require manual data collection, making management difficult with multiple stations. There is no centralized system for data or real-time analytics, and third-party platforms are often expensive and overly complex.

*Solution*
The platform uses AWS IoT Core for MQTT data ingestion, AWS Lambda and API Gateway for processing, Amazon S3 for storage (including data lake), and AWS Glue Crawlers with ETL jobs to extract, transform, and load data from the S3 data lake to another S3 bucket for analytics. AWS Amplify with Next.js provides the web interface, and Amazon Cognito ensures secure access. Similar to Thingsboard and CoreIoT, users can register new devices and manage connections, but this platform operates at a smaller scale for internal use. Key features include real-time dashboards, trend analysis, and low operational costs.

*Benefits and ROI*
The solution creates a foundational platform for lab members to develop a larger IoT platform, while providing data sources for AI researchers for model training or analysis. The platform reduces manual reporting for each station through a centralized system, simplifies management and maintenance, and improves data reliability. Estimated monthly cost is 0.66 USD (per AWS Pricing Calculator), totaling 7.92 USD for 12 months. All IoT devices are already equipped from the existing weather station system, with no additional development costs. ROI period is 6–12 months due to significant manual operation time savings.

### 3. Solution Architecture
The platform adopts an AWS Serverless architecture to manage data from 5 Raspberry Pi-based stations, scalable to 15 stations. Data is ingested via AWS IoT Core, stored in an S3 data lake, and processed by AWS Glue Crawlers and ETL jobs for transformation and loading into another S3 bucket for analytics. Lambda and API Gateway handle additional processing, while Amplify with Next.js provides a dashboard secured by Cognito.

![IoT Weather Station Architecture](/images/2-Proposal/edge_architecture.jpeg)

![IoT Weather Platform Architecture](/images/2-Proposal/platform_architecture.jpeg)

*AWS Services Used*
- *AWS IoT Core*: Ingests MQTT data from 5 stations, scalable to 15.
- *AWS Lambda*: Processes data and triggers Glue jobs (2 functions).
- *Amazon API Gateway*: Communicates with the web application.
- *Amazon S3*: Stores raw data (data lake) and processed data (2 buckets).
- *AWS Glue*: Crawlers index data, ETL jobs transform and load data.
- *AWS Amplify*: Hosts the Next.js web interface.
- *Amazon Cognito*: Manages access for lab users.

*Component Design*
- *Edge Devices*: Raspberry Pi collects and filters sensor data, sends to IoT Core.
- *Data Ingestion*: AWS IoT Core receives MQTT messages from edge devices.
- *Data Storage*: Raw data stored in S3 data lake; processed data in another S3 bucket.
- *Data Processing*: AWS Glue Crawlers index data; ETL jobs transform for analytics.
- *Web Interface*: AWS Amplify hosts Next.js application for dashboards and real-time analytics.
- *User Management*: Amazon Cognito limits to 5 active accounts.

### 4. Technical Implementation
*Deployment Phases*
The project consists of 2 parts — setting up the edge weather station and building the weather platform — each going through 4 phases:
1. *Research and Architecture Design*: Research Raspberry Pi with ESP32 sensors and design AWS Serverless architecture (1 month before internship).
2. *Cost Estimation and Feasibility Check*: Use AWS Pricing Calculator for estimation and adjustment (Month 1).
3. *Architecture Optimization for Cost/Solution*: Fine-tune (e.g., optimize Lambda with Next.js) to ensure efficiency (Month 2).
4. *Development, Testing, Deployment*: Program Raspberry Pi, AWS services with CDK/SDK and Next.js application, then test and launch (Month 2–3).

*Technical Requirements*
- *Edge Weather Station*: Sensors (temperature, humidity, rainfall, wind speed), ESP32 microcontroller, Raspberry Pi as edge device. Raspberry Pi runs Raspbian, uses Docker for data filtering and sends 1 MB/day/station via MQTT over Wi-Fi.
- *Weather Platform*: Practical knowledge of AWS Amplify (Next.js hosting), Lambda (minimized due to Next.js handling), AWS Glue (ETL), S3 (2 buckets), IoT Core (gateway and rules), and Cognito (5 users). Use AWS CDK/SDK for programming (e.g., IoT Core rules to S3). Next.js helps reduce Lambda load for the fullstack web application.

### 5. Roadmap & Milestones
- *Pre-internship (Month 0)*: 1 month planning and evaluating old station.
- *Internship (Month 1–3)*:
    - Month 1: Learn AWS and upgrade hardware.
    - Month 2: Design and adjust architecture.
    - Month 3: Deploy, test, launch.
- *Post-deployment*: Further research for up to 1 year.

### 6. Budget Estimation
View costs on [AWS Pricing Calculator](https://calculator.aws/#/estimate?id=621f38b12a1ef026842ba2ddfe46ff936ed4ab01)
Or download the [budget estimation file](../attachments/budget_estimation.pdf).

*Infrastructure Costs*
- AWS Lambda: 0.00 USD/month (1,000 requests, 512 MB storage).
- S3 Standard: 0.15 USD/month (6 GB, 2,100 requests, 1 GB scan).
- Data Transfer: 0.02 USD/month (1 GB in, 1 GB out).
- AWS Amplify: 0.35 USD/month (256 MB, 500 ms request).
- Amazon API Gateway: 0.01 USD/month (2,000 requests).
- AWS Glue ETL Jobs: 0.02 USD/month (2 DPU).
- AWS Glue Crawlers: 0.07 USD/month (1 crawler).
- MQTT (IoT Core): 0.08 USD/month (5 devices, 45,000 messages).

*Total*: 0.7 USD/month, 8.40 USD/12 months
- *Hardware*: 265 USD one-time (Raspberry Pi 5 and sensors).

### 7. Risk Assessment
*Risk Matrix*
- Network outage: Medium impact, medium probability.
- Sensor failure: High impact, low probability.
- Budget overrun: Medium impact, low probability.

*Mitigation Strategies*
- Network: Local storage on Raspberry Pi with Docker.
- Sensors: Periodic checks, spare components.
- Costs: AWS budget alerts, service optimization.

*Contingency Plan*
- Revert to manual collection if AWS encounters issues.
- Use CloudFormation to restore cost-related configurations.

### 8. Expected Results
*Technical Improvements*: Real-time data and analytics replacing manual processes. Scalable to 10–15 stations.
*Long-term Value*: 1-year data platform for AI research, reusable for future projects.
