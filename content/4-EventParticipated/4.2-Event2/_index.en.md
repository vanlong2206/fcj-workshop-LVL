---
title: "Event 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.2. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# Report on "GenAI-powered App-DB Modernization workshop"

### Purpose of the Event

- Share best practices in modern application design
- Introduce DDD methodology and event-driven architecture
- Guide selection of appropriate compute services
- Introduce AI tools supporting the development lifecycle

### Speaker List

- **Jignesh Shah** - Director, Open Source Databases
- **Erica Liu** - Sr. GTM Specialist, AppMod
- **Fabrianne Effendi** - Assc. Specialist SA, Serverless Amazon Web Services

### Key Content

#### Negative Impacts of Legacy Application Architecture

- Long product release time → Lost revenue/missed opportunities
- Poor operational efficiency → Lost productivity, high costs
- Non-compliance with security regulations → Security risks, reputational damage

#### Transitioning to Microservice Architecture

Transform into a modular system – each function is an **independent service** communicating through **events** with 3 core pillars:

- **Queue Management**: Asynchronous task processing
- **Caching Strategy:** Performance optimization
- **Message Handling:** Flexible communication between services

#### Domain-Driven Design (DDD)

- **4-step method**: Identify domain events → arrange timeline → identify actors → determine bounded contexts
- **Bookstore case study**: Illustrates practical DDD application
- **Context mapping**: 7 patterns for integrating bounded contexts

#### Event-Driven Architecture

- **3 integration patterns**: Publish/Subscribe, Point-to-point, Streaming
- **Benefits**: Loose coupling, scalability, resilience
- **Sync vs async comparison**: Understanding trade-offs

#### Compute Evolution

- **Shared Responsibility Model**: From EC2 → ECS → Fargate → Lambda
- **Serverless benefits**: No server management, auto-scaling, pay-for-value
- **Functions vs Containers**: Selection criteria

#### Amazon Q Developer

- **SDLC automation**: From planning to maintenance
- **Code transformation**: Java upgrade, .NET modernization
- **AWS Transform agents**: VMware, Mainframe, .NET migration

### What Was Learned

#### Design Thinking

- **Business-first approach**: Always start from business domain, not technology
- **Ubiquitous language**: Importance of common vocabulary between business and tech teams
- **Bounded contexts**: How to identify and manage complexity in large systems

#### Technical Architecture

- **Event storming technique**: Practical method for modeling business processes
- Using **Event-driven communication** instead of synchronous calls
- **Integration patterns**: Understanding when to use sync, async, pub/sub, streaming
- **Compute spectrum**: Criteria for choosing from VM → containers → serverless

#### Modernization Strategy

- **Phased approach**: Don't rush, need clear roadmap
- **7Rs framework**: Multiple paths depending on application characteristics
- **ROI measurement**: Cost reduction + business agility

### Application to Work

- **Apply DDD** to current project: Event storming sessions with business team
- **Refactor microservices**: Use bounded contexts to identify service boundaries
- **Implement event-driven patterns**: Replace some sync calls with async messaging
- **Serverless adoption**: Pilot AWS Lambda for suitable use cases
- **Try Amazon Q Developer**: Integrate into development workflow to boost productivity

### Event Experience

Participating in the **"GenAI-powered App-DB Modernization"** workshop was a very rewarding experience, providing a comprehensive view of modernizing applications and databases using modern methods and tools.

#### Learning from Highly Skilled Speakers
- Speakers from AWS and major technology organizations shared **best practices** in modern application design.
- Through real case studies, I gained a deeper understanding of applying **Domain-Driven Design (DDD)** and **Event-Driven Architecture** to large projects.

#### Hands-on Technical Experience
- Participating in **event storming** sessions helped me visualize how to **model business processes** into domain events.
- Learned how to **decompose microservices** and identify **bounded contexts** to manage large system complexity.
- Understood the trade-offs between **synchronous and asynchronous communication** as well as integration patterns like **pub/sub, point-to-point, streaming**.

#### Modern Tool Application
- Directly explored **Amazon Q Developer**, an AI tool supporting SDLC from planning to maintenance.
- Learned how to **automate code transformation** and pilot serverless with **AWS Lambda**, enhancing development productivity.

#### Networking and Exchange
- The workshop created opportunities for direct exchange with experts, colleagues, and business teams, helping **improve ubiquitous language** between business and tech.
- Through real examples, I recognized the importance of the **business-first approach**, always starting from business needs rather than focusing only on technology.

#### Key Takeaways
- Applying DDD and event-driven patterns helps reduce **coupling**, increase **scalability** and **resilience** for the system.
- Modernization strategy needs a **phased approach** and **ROI measurement**, avoiding rushing full system transformation.
- AI tools like Amazon Q Developer can **boost productivity** when integrated into existing development workflows.

#### Event Photos
* Add your photos here
> Overall, the event not only provided technical knowledge but also changed my way of thinking about application design, system modernization, and effective team collaboration.
