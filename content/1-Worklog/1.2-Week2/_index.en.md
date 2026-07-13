---
title: "Worklog Week 2"
date: 2026-05-11
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

* Explore AWS services and free modules applicable to the project.
* Deploy a real project on EC2 and RDS.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources |
| --- | --- | --- | --- | --- |
| Mon | - Research AWS services and free modules applicable to the project: <br>&emsp; + EC2 (t3.micro) <br>&emsp; + RDS (db.t4g.micro) <br>&emsp; + Amazon Gamelift <br>&emsp; + ElastiCache (cache.t3.micro) | 11/05/2026 | 11/05/2026 |
| Tue - Thu | - Initialize basic resources: <br>&emsp; + Create EC2 (t3.micro) with Ubuntu Linux. <br>&emsp; + Create RDS (db.t4g.micro) running MySQL. <br>&emsp; + Set up static IP for the server using Elastic IP. | 12/05/2026 | 14/05/2026 |
| Fri - Sat | - Configure and link resources: <br>&emsp; + Link RDS to EC2. <br>&emsp; + Set up inbound rules (launch-wizard-x). <br>&emsp; + Utilize spare disk space for swap memory. | 15/05/2026 | 16/05/2026 |
| Sun | - Deploy project: <br>&emsp; + Use pm2 to deploy directly on EC2. <br>&emsp; + Open ports for external access to the public IP. | 17/05/2026 | 17/05/2026 |

### Week 2 Results:

* Clearly identified suitable free AWS services/modules for the project: EC2, RDS, Amazon Gamelift, ElastiCache.
* Successfully deployed EC2 (Ubuntu Linux, t3.micro) and RDS (MySQL, db.t4g.micro).
* Successfully set up a static IP for the server using Elastic IP.
* Completed linking RDS to EC2 and configured inbound rules (launch-wizard-x) for connectivity.
* Optimized the virtual machine by utilizing spare disk space as swap memory.
* Used pm2 to deploy the project directly on EC2 and successfully opened ports for public IP access.
