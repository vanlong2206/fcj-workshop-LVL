---
title : "Infrastructure Setup and Management"
date : 2026-06-18
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---
### 5.2.1 Setting up IAM Roles with Least Privilege

In the system architecture, AWS Lambda functions are categorized into three groups with distinct permissions to ensure proper delegation and operational security.

#### 5.2.1.1 Backend Group: Responsible for handling application business logic, granted read/write access to the Amazon Aurora PostgreSQL database.

##### Creating an IAM Policy (Database Access Policy).

![IAM_RDS](images/IAM_RDS_1.png)

<div align="center"><i>Figure 5.2.1: Configuring the policy via JSON.</i></div>

```json
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

* region: ap-southeast-1 is the database deployment region.
* account-id: <aws-account-id> is the AWS account ID authorized to access the database.
* cluster-id is the ID of the Aurora Cluster.
* database-user-name is the name of the database.

![IAM_RDS](images/IAM_RDS_2.png)

<div align="center"><i>Figure 5.2.2: Naming and confirming the policy.</i></div>

##### Creating an IAM Role for the Backend.

![IAM_RDS](images/IAM_RDS_3.png)

<div align="center"><i>Figure 5.2.3: Selecting the service authorized to access the database.</i></div>

Since the database is processed via Lambda functions, select "Lambda" as the Service Use Case.

![IAM_RDS](images/IAM_RDS_4.png)

<div align="center"><i>Figure 5.2.4: Attaching policies to the Role.</i></div>

Find and select the following two policies:
* Backend-Aurora-Connect-Policy: The custom policy created above.
* AWSLambdaBasicExecutionRole: Grants permission to write logs to CloudWatch.

![IAM_RDS](images/IAM_RDS_5.png)

<div align="center"><i>Figure 5.2.5: Naming and confirming the role.</i></div>

#### 5.2.1.2 Database Maintenance Group: Handles periodic database maintenance tasks, including daily world state resets and executing `VACUUM` commands to clean up unused data, ensuring system performance.

```json
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

![IAM_Maintenance](images/IAM_Maintenance_1.png)

<div align="center"><i>Figure 5.2.6: Naming and confirming the policy.</i></div>

![IAM_Maintenance](images/IAM_Maintenance_2.png)

<div align="center"><i>Figure 5.2.7: Naming and confirming the role.</i></div>

#### 5.2.1.3 API Traffic Coordination: Modifies Amazon API Gateway environment variables to open or close system access when necessary.

```json
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

![IAM_API_Traffic](images/IAM_API_Traffic_1.png)

<div align="center"><i>Figure 5.2.8: Naming and confirming the policy.</i></div>

![IAM_API_Traffic](images/IAM_API_Traffic_2.png)

<div align="center"><i>Figure 5.2.9: Naming and confirming the role.</i></div>

---

### 5.2.2 Cost Management with AWS Budgets.

#### 5.2.2.1 Calculating Operational Costs.

Assumed Load and System Parameters:
* Daily Active Users: 500.
* Total Requests: 5,000,000 requests/month via API Gateway.
* Average Data Size: 34 KB per request/response.
* Network Traffic: ~50 GB Data Transfer Out/month.
* AWS Lambda: 512 MB RAM, 100 ms average execution time, ARM architecture.
* Amazon SQS: ~2,000,000 async requests, total 6,000,000 SQS tasks, DLQ error rate <0.1%.
* Amazon S3: ~10 GB.
* Amazon CloudWatch Logs: ~10 GB ingestion/month.
* Amazon Aurora PostgreSQL: 2 ACU average, 24/7.

| Service Name                           | Monthly Cost (USD) | Annual Cost (USD) |
| :------------------------------------- | :----------------: | :---------------: |
| Amazon Aurora PostgreSQL-Compatible DB |       74.68       |      896.16      |
| AWS Web Application Firewall (WAF)     |       11.00       |      132.00      |
| Amazon CloudFront                      |       10.25       |      123.00      |
| Amazon CloudWatch                      |        7.05        |       84.54       |
| Amazon API Gateway                     |        6.25        |       75.00       |
| AWS Lambda                             |        4.33        |       51.96       |
| Amazon Simple Queue Service (SQS)      |        3.00        |       36.00       |
| S3 Standard                            |        0.51        |       6.12       |
| Data Transfer                          |        0.00        |       0.00       |
| **Total**                        | **$117.07** |  **$1,404.78**  |

![Cost_Calculator](images/Cost_Calculator_1.png)

<div align="center"><i>Figure 5.2.10: Service cost distribution chart.</i></div>

#### 5.2.2.2 Setting AWS Budgets Limits.

##### Budget 1: Total System Operational Cost.

![Budget](images/Budget_1.png)
<div align="center"><i>Figure 5.2.11 - 5.2.16: Configuring Cost Budget (140 USD threshold, early warnings, and critical alerts).</i></div>

##### Budget 2: Aurora DB Monitoring.

![Budget](images/Budget_7.png)
<div align="center"><i>Figure 5.2.17 - 5.2.21: Configuring Usage Budget for 2 ACU 24/7 runtime.</i></div>

### 5.2.3 Initializing the Edge Layer.

The edge layer includes CloudFront, Route 53, and WAF. The configuration file `network-security-stack.yaml` is provided below.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Edge Layer: CloudFront + WAF'

Parameters:
  ApiGatewayDomainName:
    Type: String
    Description: API Gateway Endpoint URL

Resources:
# WAF Initialization
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

# CloudFront Initialization
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

Note: While the core system is in ap-southeast-1, the SSL certificate and WAF for CloudFront are managed in us-east-1.

![Network_Security_Stack](images/Network_Security_Stack_1.png)
<div align="center"><i>Figure 5.2.22 - 5.2.25: Creating the CloudFormation Stack and verifying components.</i></div>