---
title: "Dọn dẹp tài nguyên"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---
## 5.11.1 Disable tất cả SQS Event Source Mappings

Để tránh Lambda bị invoke trong lúc đang xoá:

```bash
for uuid in $(aws lambda list-event-source-mappings --region ap-southeast-1 \
  --query 'EventSourceMappings[?starts_with(FunctionArn, `arn:aws:lambda:ap-southeast-1:<account-id>:function:gameapi-sqs-consumer`)].UUID' \
  --output text); do
  aws lambda update-event-source-mapping --uuid "$uuid" --no-enabled --region ap-southeast-1
done
```

## 5.11.2 Xoá SQS Consumer Lambda stacks (5 stacks)

```bash
BASE="<project-path>/services"

for dir in sqs-consumer-economy sqs-consumer-inventory sqs-consumer-giftcode \
           sqs-consumer-stats sqs-consumer-save-data; do
  (cd "$BASE/$dir" && npx serverless remove --stage dev) &
done
wait
```

## 5.11.3 Xoá Individual Lambda stacks (6 stacks)

Các stack này cần biến môi trường `JWT_SECRET` để resolve serverless.yml.

```bash
export JWT_SECRET="<your-jwt-secret>"
BASE="<project-path>/services"

for dir in lambda-auth lambda-economy lambda-inventory lambda-transaction \
           lambda-progression-world lambda-loot-reward; do
  (cd "$BASE/$dir" && JWT_SECRET="$JWT_SECRET" npx serverless remove --stage dev) &
done
wait
```

## 5.11.4 Xoá SQS Infrastructure

```bash
cd services/sqs-infrastructure
npx serverless remove --stage dev
```

Xoá luôn các queue còn sót (nếu có):

```bash
aws sqs list-queues --region ap-southeast-1 \
  --query 'QueueUrls[?contains(@, `game-`)]' --output text | tr '\t' '\n' |
while read q; do
  aws sqs delete-queue --queue-url "$q" --region ap-southeast-1
done
```

## 5.11.5 Xoá EventBridge, Maintenance, API Gateway

```bash
# EventBridge
cd services/eventbridge-infrastructure
npx serverless remove --stage dev

# Maintenance (cần API_GATEWAY_ID)
cd services/lambda-maintenance
API_GATEWAY_ID="dummy" npx serverless remove --stage dev

# API Gateway
cd api-gateway
npx serverless remove --stage dev
```

## 5.11.6 Xoá Root Serverless Stack (game-backend-core)

Stack này chứa Lambda wrapper cho monolith. Cần biến môi trường:

```bash
cd <project-path>
JWT_SECRET="<your-jwt-secret>" \
  JWT_ISSUER="<your-issuer>" \
  JWT_AUDIENCE="<your-audience>" \
  ADMIN_SECRET="<your-admin-secret>" \
  npx serverless remove --stage dev
```

Nếu lỗi `DELETE_FAILED: ServerlessDeploymentBucket`:

```bash
# Tìm bucket
BUCKET=$(aws s3api list-buckets --query \
  'Buckets[?contains(Name, `game-backend-core`) || contains(Name, `game-backend-api`)].Name' \
  --output text)

# Empty bucket
aws s3 rm "s3://$BUCKET" --recursive

# Retry remove
npx serverless remove --stage dev
```

## 5.11.7 Xoá Aurora PostgreSQL Cluster

Kiểm tra cluster:

```bash
aws rds describe-db-clusters --region ap-southeast-1 \
  --query 'DBClusters[].{id:DBClusterIdentifier, instances:DBClusterMembers[*].DBInstanceIdentifier}'
```

Xoá DB instance trước:

```bash
aws rds delete-db-instance \
  --db-instance-identifier <instance-name> \
  --skip-final-snapshot \
  --region ap-southeast-1
```

Đợi instance deleted (~2-5 phút), sau đó xoá cluster:

```bash
while aws rds describe-db-instances --region ap-southeast-1 \
  --query 'DBInstances[?DBInstanceIdentifier==`<instance-name>`].DBInstanceStatus' \
  --output text 2>/dev/null | grep -q .; do sleep 30; done

aws rds delete-db-cluster \
  --db-cluster-identifier <cluster-name> \
  --skip-final-snapshot \
  --region ap-southeast-1
```

## 5.11.8 Xoá AWS Backup Infrastructure (nếu có)

### Xoá Recovery Points

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name gameapi-aurora-vault \
  --region ap-southeast-1 \
  --query 'RecoveryPoints[].RecoveryPointArn' --output text |
while read arn; do
  aws backup delete-recovery-point \
    --backup-vault-name gameapi-aurora-vault \
    --recovery-point-arn "$arn" \
    --region ap-southeast-1
done
```

### Xoá CloudWatch Alarms

```bash
aws cloudwatch delete-alarms \
  --alarm-names gameapi-backup-job-failed gameapi-restore-job-failed \
  --region ap-southeast-1
```

### Xoá SNS Topic + Subscription

```bash
SNS_ARN=$(aws sns list-topics --region ap-southeast-1 \
  --query 'Topics[?contains(TopicArn, `gameapi-backup-notifications`)].TopicArn' \
  --output text)

aws sns list-subscriptions-by-topic --topic-arn "$SNS_ARN" --region ap-southeast-1 \
  --query 'Subscriptions[].SubscriptionArn' --output text |
while read sub; do
  aws sns unsubscribe --subscription-arn "$sub" --region ap-southeast-1
done

aws sns delete-topic --topic-arn "$SNS_ARN" --region ap-southeast-1
```

### Xoá Backup Plan + Backup Vault

```bash
PLAN_ID=$(aws backup list-backup-plans --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi`)].BackupPlanId' \
  --output text)

aws backup delete-backup-plan --backup-plan-id "$PLAN_ID" --region ap-southeast-1
aws backup delete-backup-vault --backup-vault-name gameapi-aurora-vault --region ap-southeast-1
```

### Xoá KMS Key (tùy chọn)

```bash
KEY_ID=$(aws kms list-aliases --region ap-southeast-1 \
  --query 'Aliases[?AliasName==`alias/gameapi-aurora-backup`].TargetKeyId' \
  --output text)

aws kms schedule-key-deletion --key-id "$KEY_ID" --pending-window-in-days 7 --region ap-southeast-1
```

### Cách nhanh: xoá toàn bộ stack backup

```bash
cd services/aws-backup-infrastructure
npx serverless remove --stage dev
```

Nếu vault còn recovery point, lệnh trên sẽ fail. Phải xoá recovery points trước (bước 8.1).

## 5.11.9 Xoá CloudWatch Log Groups

```bash
for lg in $(aws logs describe-log-groups --region ap-southeast-1 \
  --query 'logGroups[?contains(logGroupName, `gameapi`) || contains(logGroupName, `game-backend`)].logGroupName' \
  --output text); do
  aws logs delete-log-group --log-group-name "$lg" --region ap-southeast-1
done
```

## 5.11.10 Kiểm tra không còn tài nguyên nào sót

```bash
echo "=== CloudFormation stacks ==="
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE \
  --region ap-southeast-1 \
  --query 'StackSummaries[?contains(StackName, `gameapi`) || contains(StackName, `game-backend`)].{name:StackName, status:StackStatus}'

echo ""
echo "=== Lambda functions ==="
aws lambda list-functions --region ap-southeast-1 \
  --query 'Functions[?contains(FunctionName, `gameapi`) || contains(FunctionName, `game-backend`)].FunctionName'

echo ""
echo "=== SQS queues ==="
aws sqs list-queues --region ap-southeast-1 \
  --query 'QueueUrls[?contains(@, `game-`)]'

echo ""
echo "=== RDS clusters ==="
aws rds describe-db-clusters --region ap-southeast-1 \
  --query 'DBClusters[].DBClusterIdentifier'

echo ""
echo "=== API Gateways ==="
aws apigateway get-rest-apis --region ap-southeast-1 \
  --query 'items[?contains(name, `gameapi`) || contains(name, `game-backend`)].name'

echo ""
echo "=== S3 buckets ==="
aws s3api list-buckets --region ap-southeast-1 \
  --query 'Buckets[?contains(Name, `game-backend`) || contains(Name, `gameapi`)].Name'

echo ""
echo "=== Backup vaults ==="
aws backup list-backup-vaults --region ap-southeast-1 \
  --query 'BackupVaults[?contains(BackupVaultName, `gameapi`)].BackupVaultName'
```
