---
title: "Worklog Week 8"
date: 2026-06-22
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Week 8 Objectives:

* Deep dive into relational database (PostgreSQL/Aurora) maintenance tasks using `VACUUM` and `ANALYZE` commands.
* Master the technique of intervening in the API Gateway layer to flexibly open/close communication channels using Stage Variables.
* Identify and design an automated System Maintenance flow as a special feature for the project.
* Finalize and reconcile the complete system diagram.
* Review the architecture and establish a concrete deployment plan, assigning functional group tasks to members.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources |
| --- | --- | --- | --- | --- |
| Mon | - Learn DB Maintenance:<br>+ `VACUUM` command: clean up dead tuples.<br>+ `ANALYZE` command: update statistics for the query planner.<br>- Learn API Gateway: How to use SDK to update `stageVariables`, route traffic to a maintenance page or return 503 error when needed. | 22/06/2026 | 22/06/2026 | |
| Tue | - Finalize, unify, and review the architecture diagram.<br>- Plan: Break down the system into small modules, form deployment groups, and assign specific tasks to each member. | 23/06/2026 | 23/06/2026 | |
| Wed - Sun | - Deploy and write step-by-step reports for:<br>+ Base infrastructure: IAM Role, policies, real-world cost estimation for worst-case scenarios, set up cost management limits with AWS Budget.<br>+ Edge network layer: quick deployment of CloudFront and WAF using CloudFormation (in US region).<br>+ Database: AuroraSQL, quick deployment of AWS Backup with AWS S3 using CloudFormation. | 24/06/2026 | 28/06/2026 | |

### Week 8 Results:

* Mastered database maintenance mechanisms (Aurora/PostgreSQL) using `VACUUM` to clean up stale data and `ANALYZE` to optimize the query planner.
* Learned how to use the SDK to intervene in API Gateway, flexibly updating `stageVariables` to automatically route traffic to a maintenance flow or disconnect to protect the system.
* Finalized, reviewed, and unified the final overall system architecture diagram, ensuring tight cohesion between components.
* Successfully established a detailed deployment plan: decomposed the system into small modules, formed groups, and clearly assigned tasks to each project member.
* Completed actual deployment and wrote guide reports for foundational items:
  * Base infrastructure: Successfully created IAM Roles, security policies, calculated real cost risks for worst-case scenarios, and configured safe spending alerts via AWS Budget.
  * Edge network layer: Applied CloudFormation to quickly deploy CloudFront distribution and WAF firewall in the US region.
  * Database & Storage: Initialized AuroraSQL and successfully set up automated data backup to Amazon S3 via AWS Backup using CloudFormation scripts.
