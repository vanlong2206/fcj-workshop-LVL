---
title: "Resource Clean Up"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---
## 5.11.1 Disable all SQS Event Source Mappings

To prevent Lambda from being invoked during deletion:

```bash
for uuid in $(aws lambda list-event-source-mappings --region ap-southeast-1 \
  --query 'EventSourceMappings[?starts_with(FunctionArn, `arn:aws:lambda:ap-southeast-1:<account-id>:function:gameapi-sqs-consumer`)].UUID' \
  --output text); do
  aws lambda update-event-source-mapping --uuid "$uuid" --no-enabled --region ap-southeast-1
done
```

## 5.11.2 Delete SQS Consumer Lambda stacks (5 stacks)

```bash
BASE="<project-path>/services"

for dir in sqs-consumer-economy sqs-consumer-inventory sqs-consumer-giftcode \
           sqs-consumer-stats sqs-consumer-save-data; do
  (cd "$BASE/$dir" && npx serverless remove --stage dev) &
done
wait
```

## 5.11.3 Delete Individual Lambda stacks (6 stacks)

These stacks require the `JWT_SECRET` environment variable to resolve serverless.yml.

```bash
export JWT_SECRET="<your-jwt-secret>"
BASE="<project-path>/services"

for dir in lambda-auth lambda-economy lambda-inventory lambda-transaction \
           lambda-progression-world lambda-loot-reward; do
  (cd "$BASE/$dir" && JWT_SECRET="$JWT_SECRET" npx serverless remove --stage dev) &
done
wait
```

## 5.11.4 Delete SQS Infrastructure

```bash
cd services/sqs-infrastructure
npx serverless remove --stage dev
```

Delete any remaining queues (if any):

```bash
aws sqs list-queues --region ap-southeast-1 \
  --query 'QueueUrls[?contains(@, `game-`)]' --output text | tr '\t' '\n' |
while read q; do
  aws sqs delete-queue --queue-url "$q" --region ap-southeast-1
done
```

## 5.11.5 Delete EventBridge, Maintenance, API Gateway

```bash
# EventBridge
cd services/eventbridge-infrastructure
npx serverless remove --stage dev

# Maintenance (requires API_GATEWAY_ID)
cd services/lambda-maintenance
API_GATEWAY_ID="dummy" npx serverless remove --stage dev

# API Gateway
cd api-gateway
npx serverless remove --stage dev
```

## 5.11.6 Delete Root Serverless Stack (game-backend-core)

This stack contains the Lambda wrapper for the monolith. Requires environment variables:

```bash
cd <project-path>
JWT_SECRET="<your-jwt-secret>" \
  JWT_ISSUER="<your-issuer>" \
  JWT_AUDIENCE="<your-audience>" \
  ADMIN_SECRET="<your-admin-secret>" \
  npx serverless remove --stage dev
```

If `DELETE_FAILED: ServerlessDeploymentBucket` error occurs:

```bash
# Find the bucket
BUCKET=$(aws s3api list-buckets --query \
  'Buckets[?contains(Name, `game-backend-core`) || contains(Name, `game-backend-api`)].Name' \
  --output text)

# Empty bucket
aws s3 rm "s3://$BUCKET" --recursive

# Retry remove
npx serverless remove --stage dev
```

## 5.11.7 Delete Aurora PostgreSQL Cluster

Check cluster:

```bash
aws rds describe-db-clusters --region ap-southeast-1 \
  --query 'DBClusters[].{id:DBClusterIdentifier, instances:DBClusterMembers[*].DBInstanceIdentifier}'
```

Delete DB instance first:

```bash
aws rds delete-db-instance \
  --db-instance-identifier <instance-name> \
  --skip-final-snapshot \
  --region ap-southeast-1
```

Wait for the instance to be deleted (~2-5 minutes), then delete the cluster:

```bash
while aws rds describe-db-instances --region ap-southeast-1 \
  --query 'DBInstances[?DBInstanceIdentifier==`<instance-name>`].DBInstanceStatus' \
  --output text 2>/dev/null | grep -q .; do sleep 30; done

aws rds delete-db-cluster \
  --db-cluster-identifier <cluster-name> \
  --skip-final-snapshot \
  --region ap-southeast-1
```

## 5.11.8 Delete AWS Backup Infrastructure (if any)

### Delete Recovery Points

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

### Delete CloudWatch Alarms

```bash
aws cloudwatch delete-alarms \
  --alarm-names gameapi-backup-job-failed gameapi-restore-job-failed \
  --region ap-southeast-1
```

### Delete SNS Topic + Subscription

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

### Delete Backup Plan + Backup Vault

```bash
PLAN_ID=$(aws backup list-backup-plans --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi`)].BackupPlanId' \
  --output text)

aws backup delete-backup-plan --backup-plan-id "$PLAN_ID" --region ap-southeast-1
aws backup delete-backup-vault --backup-vault-name gameapi-aurora-vault --region ap-southeast-1
```

### Delete KMS Key (optional)

```bash
KEY_ID=$(aws kms list-aliases --region ap-southeast-1 \
  --query 'Aliases[?AliasName==`alias/gameapi-aurora-backup`].TargetKeyId' \
  --output text)

aws kms schedule-key-deletion --key-id "$KEY_ID" --pending-window-in-days 7 --region ap-southeast-1
```

### Fast way: delete entire backup stack

```bash
cd services/aws-backup-infrastructure
npx serverless remove --stage dev
```

If the vault still has recovery points, the command above will fail. Recovery points must be deleted first (step 8.1).

## 5.11.9 Delete CloudWatch Log Groups

```bash
for lg in $(aws logs describe-log-groups --region ap-southeast-1 \
  --query 'logGroups[?contains(logGroupName, `gameapi`) || contains(logGroupName, `game-backend`)].logGroupName' \
  --output text); do
  aws logs delete-log-group --log-group-name "$lg" --region ap-southeast-1
done
```

## 5.11.10 Verify No Resources Remain

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
