---
title: "Worklog Week 6"
date: 2026-06-08
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

* Learn about data storage and backup services on AWS: Amazon S3 and AWS Backup.
* Investigate solutions using AWS Lambda to automate database export for optimizing off-AWS storage costs.
* Pause cloud tasks and return to Node.js environment to solve a core backend game problem: Design an anti-cheat item drop system using deterministic random number generation based on a `master seed`.

### Weekly Tasks:

| Day | Tasks | Start Date | End Date | Resources / Notes |
| --- | --- | --- | --- | --- |
| Mon | - Learn about Amazon S3:<br>+ Storage tiers.<br>+ Versioning and Bucket Policies security.<br>+ Overview of AWS Backup. | 08/06/2026 | 08/06/2026 | |
| Tue | - Investigate custom backup solutions:<br>+ Ideate using scheduled Lambda triggers to export the database to files and push to S3.<br>+ Evaluate cost and feasibility of the solution. | 09/06/2026 | 09/06/2026 | The solution is unnecessary since standard Backup services are already quite cheap, and the text-based data is very lightweight. |
| Wed - Fri | - Learn Deterministic Random techniques.<br>- Build a pseudo-random number generation algorithm based on a `master seed` issued by the server.<br>- Integrate and test:<br>+ Server creates and stores the seed for each session.<br>+ Validate item results sent by the client by replaying the seed sequence on the server. | 10/06/2026 | 12/06/2026 | |

### Week 6 Results:

* Mastered static object management on Amazon S3 and how to set up lifecycle policies to optimize long-term storage costs.
* Completed evaluation of using AWS Lambda to automate database export for private storage.
* Strengthened backend programming skills with Node.js through solving a real-world problem in multiplayer game development.
* Successfully designed and implemented a Deterministic Random mechanism:
  * Understood the anti-cheat principle: The client does not decide item drop outcomes; all random numbers are generated from a `master seed`.
  * Built the sync flow: The server creates a `master seed` at the start of a game session and sends it to the Client. When the Client reports drop results, the Server simply uses that same seed to replay the operation. If the result matches, the item is recorded.
  * This method effectively prevents hacking while ensuring the Server is not overloaded, as it does not need to continuously calculate drop rates for each defeated monster.
