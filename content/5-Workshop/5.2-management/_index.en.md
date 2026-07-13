---
title : "Infrastructure Setup and Management"
date : 2026-06-18
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

### 5.2.1 Setting up Roles with Minimum Permissions using AWS IAM.

In the system architecture diagram, AWS Lambda functions are divided into three groups with separate permissions to ensure authorization and operational safety.

#### 5.2.1.1 Backend Group: Responsible for processing application business logic and granted read/write permissions on Amazon Aurora PostgreSQL database.

##### Creating IAM Policy (Database Access Policy).

![IAM_RDS](images/IAM_RDS_1.png)

<div align="center"><i>Figure 5.2.1: Create custom policy using JSON.</i></div>

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"rds-db:connect"
			],
			"Resource": [
				"arn:aws:rds-db:<region>:<aws-account-id>:dbuser:cluster-id/database-user-name"
			]
		}
	]
}
```

region: ap-southeast-1 is the database region.

account-id: <aws-account-id> is the AWS account ID allowed to access the database.

cluster-id is the Aurora Cluster ID.

database-user-name is the database name.

Temporarily leave cluster-id and database-user-name as placeholders.

![IAM_RDS](images/IAM_RDS_2.png)

<div align="center"><i>Figure 5.2.2: Name and confirm the policy.</i></div>

##### Creating IAM Role for Backend.

![IAM_RDS](images/IAM_RDS_3.png)

<div align="center"><i>Figure 5.2.3: Select the authorized service for database access.</i></div>

Since the database is accessed through Lambda functions, select Lambda Service Use Case.

![IAM_RDS](images/IAM_RDS_4.png)

<div align="center"><i>Figure 5.2.4: Add policy to this role.</i></div>

Search and select the following 2 policies:

* Backend-Aurora-Connect-Policy: the policy just created to grant database access.
* AWSLambdaBasicExecutionRole: this policy grants permission to write logs to CloudWatch.

![IAM_RDS](images/IAM_RDS_5.png)

<div align="center"><i>Figure 5.2.5: Name and confirm the role.</i></div>

#### 5.2.1.2 Database Maintenance Group: Responsible for periodic database maintenance tasks, including resetting the daily world progression and executing `VACUUM` to clean up unused data, helping maintain system performance.

Similar steps...

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowInvokeMaintenanceLambda",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
			"Resource": [
				"arn:aws:lambda:<region>:<aws-account-id>:function:lambda-function-name"
			]
        }
    ]
}
```

region: ap-southeast-1 is the database region.

account-id: <aws-account-id> is the AWS account ID allowed to access the database.

lambda-function-name: the name of the Lambda function responsible for database maintenance.

Temporarily leave lambda-function-name as a placeholder.

![IAM_Maintenance](images/IAM_Maintenance_1.png)

<div align="center"><i>Figure 5.2.6: Name and confirm the policy.</i></div>

![IAM_Maintenance](images/IAM_Maintenance_2.png)

<div align="center"><i>Figure 5.2.7: Name and confirm the role.</i></div>

#### 5.2.1.3 API Traffic Orchestration: Intervene in Amazon API Gateway environment variables to open or close system access when needed.

Similar steps...

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowUpdateAPIValues",
            "Effect": "Allow",
            "Action": [
                "apigateway:PATCH"
            ],
            "Resource": [
                "arn:aws:apigateway:ap-southeast-1::/restapis/api-id/stages/*"
            ]
        }
    ]
}
```

api-id: The API Gateway ID.

Temporarily leave api-id as a placeholder.

![IAM_API_Traffic](images/IAM_API_Traffic_1.png)

<div align="center"><i>Figure 5.2.8: Name and confirm the policy.</i></div>

![IAM_API_Traffic](images/IAM_API_Traffic_2.png)

<div align="center"><i>Figure 5.2.9: Name and confirm the role.</i></div>

---

### 5.2.2 Cost Management with AWS Budgets.

#### 5.2.2.1 Operational Cost Estimation.

Assumed Load and System Parameters

* Number of users: 500 daily active users.
* Total requests: 5,000,000 requests/month through API Gateway.
* Average data size: 34 KB per request/response.
* Network traffic: ~50 GB Data Transfer Out per month.
* AWS Lambda:
  * Memory: 512 MB RAM.
  * Average execution time: 100 ms/request.
  * Architecture: ARM.
* Amazon SQS: ~2,000,000 async processing requests, generating 6,000,000 total SQS operations. DLQ error rate estimated at <0.1%.
* Amazon S3: ~10 GB.
* Amazon CloudWatch Logs: ~10 GB ingested and stored per month.
* Amazon Aurora PostgreSQL & RDS Proxy: average 2 ACU running 24/7.

| Service Name                           |           Monthly Cost (USD)           | Annual Cost (USD) |
| :------------------------------------- | :-------------------------------------: | :---------------: |
| Amazon Aurora PostgreSQL-Compatible DB |                  74.68                  |      896.16      |
| Amazon RDS Proxy                       |                  26.28                  |      315.36      |
| AWS Web Application Firewall (WAF)     |                  11.00                  |      132.00      |
| Amazon CloudFront                      |                  10.25                  |      123.00      |
| Amazon CloudWatch                      |                  7.05                  |       84.54       |
| Amazon API Gateway                     |                  6.25                  |       75.00       |
| AWS Lambda                             |                  4.33                  |       51.96       |
| Amazon Simple Queue Service (SQS)      |                  3.00                  |       36.00       |
| Amazon Route 53                        |                  0.90                  |       10.80       |
| S3 Standard                            |                  0.51                  |       6.12       |
| Data Transfer                          |                  0.00                  |       0.00       |
| **Total**                        | **$134.30** | **$1,611.58** |                  |

![Cost_Calculator](images/Cost_Calculator_1.png)

<div align="center"><i>Figure 5.2.10: Service cost ratio chart.</i></div>

#### 5.2.2.2 Setting Budget Limits for AWS Budgets.

##### Budget 1: Calculating total operational cost of the entire system.

![Budget](images/Budget_1.png)

<div align="center"><i>Figure 5.2.11: Configure Cost Budget.</i></div>

![Budget](images/Budget_2.png)

<div align="center"><i>Figure 5.2.12: Set Cost Budget parameters.</i></div>

Fixed monthly limit of 140 USD.

![Budget](images/Budget_3.png)

<div align="center"><i>Figure 5.2.13: Set early warning (forecast).</i></div>

![Budget](images/Budget_4.png)

<div align="center"><i>Figure 5.2.14: Set actual threshold warning (near limit).</i></div>

![Budget](images/Budget_5.png)

<div align="center"><i>Figure 5.2.15: Set emergency warning (exceeded limit).</i></div>

![Budget](images/Budget_6.png)

<div align="center"><i>Figure 5.2.16: Confirm and complete Cost Budget creation.</i></div>

##### Budget 2: Monitor Aurora DB.

![Budget](images/Budget_7.png)

<div align="center"><i>Figure 5.2.17: Configure Usage Budget.</i></div>

![Budget](images/Budget_8.png)

<div align="center"><i>Figure 5.2.18: Set Usage Budget parameters.</i></div>

System runs 2 ACU 24/7, requiring approximately 1460 ACU-Hrs per month.

![Budget](images/Budget_9.png)

<div align="center"><i>Figure 5.2.19: Set actual threshold warning.</i></div>

![Budget](images/Budget_10.png)

<div align="center"><i>Figure 5.2.20: Set emergency actual threshold warning.</i></div>

![Budget](images/Budget_11.png)

<div align="center"><i>Figure 5.2.21: Confirm and complete Usage Budget creation.</i></div>

### 5.2.3 Creating the Network Edge Layer.

This layer includes: CloudFront, Route 53, and WAF.

Create a `.yaml` file `network-security-stack.yaml` with the following content:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Edge Layer: CloudFront + WAF'

Parameters:
  ApiGatewayDomainName:
    Type: String
    Description: Endpoint URL của API Gateway

Resources:
# Initialize WAF
  ApiWafWebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: GameApiWaf
      Scope: CLOUDFRONT
      DefaultAction:
        Allow: {}
      VisibilityConfig:
        CloudWatchMetricsEnabled: true
        MetricName: GameApiWafMetrics
        SampledRequestsEnabled: true
      Rules:
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 1
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet
          VisibilityConfig:
            CloudWatchMetricsEnabled: true
            MetricName: AWSCommonRulesMetrics
            SampledRequestsEnabled: true

# Initialize CloudFront
  ApiCloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Comment: "CDN API Gateway"
        WebACLId: !GetAtt ApiWafWebACL.Arn
        Origins:
          - Id: ServerlessApiGatewayOrigin
            DomainName: !Ref ApiGatewayDomainName
            CustomOriginConfig:
              HTTPSPort: 443
              OriginProtocolPolicy: https-only
        DefaultCacheBehavior:
          TargetOriginId: ServerlessApiGatewayOrigin
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods: 
            - GET
            - HEAD
            - OPTIONS
            - PUT
            - PATCH
            - POST
            - DELETE
          CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
          OriginRequestPolicyId: b689b0a8-53d0-40ab-b3f3-19cb4105c74c

Outputs:
  CloudFrontUrl:
    Description: "URL for Client"
    Value: !GetAtt ApiCloudFrontDistribution.DomainName
```

Note: Although the core system is in ap-southeast-1, the SSL certificates and WAF protecting CloudFront are located in us-east-1, so the network edge layer will be deployed in us-east-1.

![Network_Security_Stack](images/Network_Security_Stack_1.png)

<div align="center"><i>Figure 5.2.22: Create CloudFormation Stack and upload the network-security-stack.yaml file.</i></div>

![Network_Security_Stack](images/Network_Security_Stack_2.png)

<div align="center"><i>Figure 5.2.23: Name the stack and provide the endpoint.</i></div>

Currently there is no API Gateway domain name for communication, so temporarily use any domain name `aws.amazon.com` to bypass this step; it will be updated later when the actual domain is available.

![Network_Security_Stack](images/Network_Security_Stack_3.png)

<div align="center"><i>Figure 5.2.24: Confirm and Submit.</i></div>

![Network_Security_Stack](images/Network_Security_Stack_4.png)

<div align="center"><i>Figure 5.2.25: Verify the created components.</i></div>
