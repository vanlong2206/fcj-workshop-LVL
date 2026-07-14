---
title: "Event 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.3. </b> "
---

# Report on FCAJ Community Day

| Info        | Details                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------ |
| Date        | 27/06/2026                                                                                 |
| Location | 26th Floor, Bitexco Building, 02 Hai Trieu, Ben Nghe Ward, District 1, Ho Chi Minh City |
| Role        | Attendee                                                                                |

## 4.3.1 Purpose of the Event

The sharing session focused on introducing AI application solutions in the enterprise environment, particularly in infrastructure operations, cloud system optimization, and data security. The content also covered practical AI deployment models on AWS to improve operational efficiency and process automation.

## 4.3.2 Speaker List

* **Mr. Steve Tran (Cloud Thinker):** Topic *Cloud Infrastructure Operations in the Age of Agentic Platform*.
* **Mr. Hieu Nghi & Mr. Kiet & Mr. Trung:** Topic *Building a Voice AI Assistant for Enterprises*.
* **Ms. Bao & Mr. Nguyen Nguyen (Cloud Kinetics):** Topic *Deploying DevOps AI Agent on AWS*.
* **Mr. Truong & Ms. Minh Anh (Noventic):** Topic *Automating HR Processes with Amazon Q*.
* **Mr. Toan Nguyen & Mr. Hieu Nghi (Renova Cloud):** Topic *Private Security Model for Amazon Q and MCP Server*.

## 4.3.3 Notable Content

### Cloud Infrastructure Operations in the Age of Agentic Platform

The presentation introduced the **Agentic Platform** model aimed at supporting smarter system operations through AI. This platform focuses on solving four common problems: incident investigation, code review, infrastructure cost optimization (FinOps), and security enhancement.

The speaker also compared two AI Agent deployment models:

* **Single Agent** can effectively handle most tasks if provided with sufficient context and workflow.
* **Multi-Agent** divides work among multiple specialized agents to optimize processing capability, context management, and clearer permission segregation.

This approach helps engineering teams reduce analysis and incident response time in real-world operational environments.

### Building a Voice AI Assistant for Enterprises

The next session presented the architecture for building a voice AI assistant using three main components:

* **Speech-to-Text (STT)** converts speech into text.
* **Large Language Model (LLM)** processes content and context.
* **Text-to-Speech (TTS)** generates voice responses.

Using text as an intermediary layer makes it easier for enterprises to apply Guardrails for content control as well as integrate real-time API calling capabilities.

In a Production environment, the system needs additional mechanisms such as Streaming to reduce latency, support for multiple voice regions, gender recognition, handling user interruptions, and call escalation to human agents when necessary.

### DevOps AI Agent on AWS

This topic introduced a solution using AI Agents to support DevOps processes, especially for automated incident investigation after system alerts from CloudWatch.

The solution is built on six key components:

* Context Learning
* Control
* Integration
* Collaboration
* Convenient
* Cost Effective

The processing workflow includes receiving alerts, classifying errors, performing Root Cause Analysis, proposing immediate remediation, and suggesting long-term improvements. The AI Agent only plays a supportive and advisory role, not directly making changes to the system.

### Applying Amazon Q in HR Processes

The session introduced Amazon Q as an internal AI assistant for enterprises with the ability to ensure higher data security compared to public AI platforms.

Amazon Q can integrate with various systems such as Google Workspace, Microsoft Workspace, Jira, Salesforce, and GitHub via MCP.

In the recruitment process, the system can automatically:

* Analyze recruitment guideline documents.
* Extract content from CVs, including OCR.
* Evaluate and classify candidates.
* Suggest salary levels and generate reports through no-code applications.

### Private Security Model for Amazon Q and MCP Server

The final topic focused on the architecture for deploying Amazon Q in a Private environment to ensure information security.

The solution is built on the **Zero Trust** principle, helping to mitigate risks such as DDoS or Man-in-the-Middle attacks.

The proposed architecture includes:

* Placing the MCP Server in a Private Subnet within the VPC.
* Amazon Q accessing via Interface Endpoint and VPC Connection.
* Encrypting traffic with TLS through Internal ALB combined with ACM and Amazon Cognito.
* Using Route 53 Resolver to hide internal DNS from the Internet.

## 4.3.4 Application to Work

* View AI as a supporting tool to enhance work efficiency rather than completely replacing human roles.
* When deploying AI in enterprises, prioritize Data Governance, Guardrails, and Private security models before putting the system into Production.
* Apply the Agentic Platform and Multi-Agent mindset to support system monitoring, incident investigation, and operational cost optimization in projects using AWS.

![](blob:https://www.facebook.com/ebfcc44c-603b-4bb2-a1ce-9575e430ad4c)![1784013303527](image/_index.vi/1784013303527.png)
