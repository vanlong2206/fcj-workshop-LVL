---
title: "Building a safer multi-tenant AI system with Amazon Bedrock"
date: 2026-07-07
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---
### Building a safer multi-tenant AI system with Amazon Bedrock

![Multi-tenant system architecture with Amazon Bedrock](images/bedrock-architecture_1.jpg)

![Multi-tenant system architecture with Amazon Bedrock](images/bedrock-architecture_2.jpg)

Hello everyone, today I want to share a great article by AWS on how PAR Technology built an AI system using Amazon Bedrock while ensuring that each user can only see the exact data they are authorized to access.

PAR is a company providing technology solutions for various restaurant chains. They developed a system that allows users to ask questions in natural language, and then the AI automatically generates SQL queries to retrieve data and answer. The idea sounds quite simple, but when implemented in reality, a very important problem arises: data security.

#### 3.1.1 The Problem

In PAR's system, many customers share the same platform. For example, a store owner can only view their own store's revenue, while a senior manager can view the revenue of the entire chain.

If the AI generates an incorrect query or fetches the wrong data, users might see information they are not permitted to access. This is a very serious issue for enterprise systems.

#### 3.1.2 PAR's Solution

Instead of letting the AI decide everything, PAR built a system with multiple layers of protection:

* **Identity Verification:** All requests are authenticated to ensure the sender is valid.
* **Request Moderation:** The system checks if the user's question is clear and appropriate before passing it to the AI for processing.
* **Data Filtering by Permissions:** Most importantly, data is filtered according to each user's access permissions *before* the AI generates the SQL query. This ensures the AI only works on authorized data, rather than the entire database.

#### 3.1.3 Why is this approach effective?

What I find interesting is that AWS and PAR do not place complete trust in the LLM. The AI is only responsible for understanding the question and assisting in SQL generation, while access control remains handled by the system. Thanks to this, even if the AI generates an inaccurate query or is influenced by unexpected prompts, data outside the permitted scope cannot be accessed.

#### 3.1.4 Results Achieved

According to the article, this architecture has been used in a real-world environment and processed over **50,000 queries** without any data leakage between users. Beyond increasing security, this design approach helps businesses feel more confident when deploying AI applications on critical data.

#### 3.1.5 Personal Thoughts & Conclusion

In my opinion, this is quite an interesting lesson when building AI applications. I used to think that just choosing a good LLM model was enough, but in reality, designing the architecture and controlling access rights are even more important.

Amazon Bedrock makes building AI applications convenient, but for the system to operate securely, it must be combined with authentication, authorization, and data filtering mechanisms right from the start. Through this article, I realize that AI should not be the sole component deciding data access. By combining Amazon Bedrock with appropriate security layers, businesses can build AI systems that are smarter, safer, and more reliable.

*Author: Lai Van Long*

**Reference:** [Multi-tenant LLM analytics with row-level security: How we built a secure agent on AWS](https://aws.amazon.com/vi/blogs/machine-learning/multi-tenant-llm-analytics-with-row-level-security-how-we-built-a-secure-agent-on-aws/)