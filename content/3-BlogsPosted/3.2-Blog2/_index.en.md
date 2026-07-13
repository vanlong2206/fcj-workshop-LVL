---
title: "Blog 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.2. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# Session Policies in Amazon EKS Pod Identity

Amazon EKS Pod Identity has recently added session policies feature, allowing you to narrow IAM permissions flexibly and precisely for each pod without creating multiple separate IAM roles. This is a significant step toward applying the least privilege principle more effectively in large-scale Kubernetes environments.

Key points to understand:

* Session policy is an inline IAM policy specified when creating or updating a Pod Identity association.
* Effective permissions = intersection between IAM role permissions and session policy → session policy can only narrow, never expand permissions.
* Helps avoid over-permissioning when reusing a shared IAM role for multiple workloads with different needs.
* Supports both same-account and cross-account (via IAM role chaining).
* Significantly reduces the number of IAM roles to manage, avoiding IAM quota limits in large clusters.
* Easy configuration via AWS Management Console, AWS CLI, or AWS SDK when creating associations between Kubernetes ServiceAccount and IAM role.

This feature is especially useful when you have multiple applications running on the same IAM role but need different permission restrictions.

...Images...

...Links...

...Instructions...
