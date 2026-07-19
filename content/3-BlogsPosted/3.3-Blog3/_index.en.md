---
title: "Using Amazon CloudFront to accelerate and optimize user request processing"
date: 2026-07-09
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---
### Using Amazon CloudFront to accelerate and optimize user request processing

![CloudFront CDN](images/cloudfront-cdn.jpg)

Hello everyone, during my research on serverless architecture on AWS, I had the opportunity to learn about **Amazon CloudFront** – a CDN (Content Delivery Network) service that helps accelerate access speed and improve user experience when using applications. After reading the AWS Blog, I would like to share some knowledge I have learned.

#### 3.3.1 Basic operating mechanism

When a user sends a request to the system, the request is routed to Amazon CloudFront first:

* **Cache Hit:** If the content is already stored in the cache, CloudFront will respond immediately from the Edge Location closest to the user, helping to reduce latency and accelerate content loading speed.
* **Cache Miss:** If the data is not yet in the cache, CloudFront will forward the request to the Origin (such as Amazon S3 or API Gateway), retrieve the data, and store it to serve subsequent visits.

#### 3.3.2 Optimizing performance and security

What I find great about CloudFront is that it not only helps improve performance but also reduces the load on the Origin when many users access the same content. By leveraging the caching mechanism at global Edge Locations, the number of requests sent directly to the server is significantly reduced, thereby helping the system operate more stably and optimizing operational costs.

In addition to its acceleration capabilities, Amazon CloudFront can also be combined with **AWS WAF** (Web Application Firewall) to filter invalid requests before they reach the application. This contributes to enhanced security and reduces the risk of common web attacks.

#### 3.3.3 Conclusion

Through this blog post, I have a better understanding of how Amazon CloudFront processes requests, how the caching mechanism works, and why this is one of the commonly used services when deploying applications on AWS. This is quite useful knowledge for me when exploring solutions for building high-performance and scalable systems on the cloud platform.

I hope this short sharing will be useful to everyone. If you have more practical experience using Amazon CloudFront, I look forward to discussing and learning more.

*Author: Lai Van Long*

**Reference:** [Charting the life of an Amazon CloudFront request](https://aws.amazon.com/vi/blogs/networking-and-content-delivery/charting-the-life-of-an-amazon-cloudfront-request/)