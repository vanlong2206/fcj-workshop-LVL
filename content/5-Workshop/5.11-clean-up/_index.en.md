---
title: "Resource Clean Up"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---
#### 5.11.1 Delete Aurora PostgreSQL Restore Test Cluster

Check if the test cluster still exists:

```bash
aws rds describe-db-clusters \
  --region ap-southeast-1 \
  --query 'DBClusters[?contains(DBClusterIdentifier, `restore-test`)].DBClusterIdentifier' \
  --output table
```

If exists, delete the cluster:

```bash
aws rds delete-db-cluster \
  --db-cluster-identifier quan-restore-test \
  --skip-final-snapshot \
  --region ap-southeast-1
```

> If the cluster has a standalone instance, delete the instance first:
>
> ```bash
> aws rds delete-db-instance \
>   --db-instance-identifier <instance-name> \
>   --skip-final-snapshot \
>   --region ap-southeast-1
> ```

#### 5.11.2 Delete Recovery Points in Backup Vault

The backup vault cannot be deleted if it still contains recovery points. Recovery points must be deleted first.

List recovery points:

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name gameapi-aurora-vault \
  --region ap-southeast-1 \
  --query 'RecoveryPoints[].RecoveryPointArn' \
  --output text
```

Delete each recovery point:

```bash
aws backup delete-recovery-point \
  --backup-vault-name gameapi-aurora-vault \
  --recovery-point-arn <replace-with-arn-from-above> \
  --region ap-southeast-1
```

Repeat until the vault has no remaining recovery points.

#### 5.11.3 Delete CloudWatch Alarms

Delete the 2 alarms that were created:

```bash
aws cloudwatch delete-alarms \
  --alarm-names gameapi-backup-job-failed gameapi-restore-job-failed \
  --region ap-southeast-1
```

#### 5.11.4 Delete SNS Topic and Email Subscription

List subscriptions:

```bash
SNS_ARN=$(aws sns list-topics \
  --region ap-southeast-1 \
  --query 'Topics[?contains(TopicArn, `gameapi-backup-notifications`)].TopicArn' \
  --output text)

aws sns list-subscriptions-by-topic \
  --topic-arn "$SNS_ARN" \
  --region ap-southeast-1
```

Unsubscribe:

```bash
aws sns unsubscribe \
  --subscription-arn <SubscriptionArn> \
  --region ap-southeast-1
```

Delete the SNS topic:

```bash
aws sns delete-topic \
  --topic-arn "$SNS_ARN" \
  --region ap-southeast-1
```

#### 5.11.5 Delete Backup Plan and Backup Vault

List backup plan ID:

```bash
aws backup list-backup-plans \
  --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi-aurora-backup-plan`)].BackupPlanId' \
  --output text
```

Delete backup plan (requires `--deletion-window-in-days` parameter):

```bash
aws backup delete-backup-plan \
  --backup-plan-id <BackupPlanId> \
  --region ap-southeast-1
```

Delete backup vault (only possible if the vault has no recovery points):

```bash
aws backup delete-backup-vault \
  --backup-vault-name gameapi-aurora-vault \
  --region ap-southeast-1
```

#### 5.11.6 Delete KMS Key

If you want to delete the KMS key as well (note: this is irreversible):

```bash
# Get Key ID from alias
KEY_ID=$(aws kms list-aliases \
  --region ap-southeast-1 \
  --query 'Aliases[?AliasName==`alias/gameapi-aurora-backup`].TargetKeyId' \
  --output text)

# Schedule key deletion after 7 days (default)
aws kms schedule-key-deletion \
  --key-id "$KEY_ID" \
  --pending-window-in-days 7 \
  --region ap-southeast-1
```

#### 5.11.7 Remove All Stacks with Serverless Framework (Fastest Way)

This method automates all the steps above (except deleting the test cluster):

```bash
cd services/aws-backup-infrastructure
npx serverless remove --stage dev
```

> **Note:** If the vault still has recovery points, this command will fail. Step 2 must be done first.

#### 5.11.8 Verify No Resources Remain

```bash
# Check vault
aws backup list-backup-vaults --region ap-southeast-1 \
  --query 'BackupVaults[?contains(BackupVaultName, `gameapi`)].BackupVaultName'

# Check plan
aws backup list-backup-plans --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi`)].BackupPlanName'

# Check SNS
aws sns list-topics --region ap-southeast-1 \
  --query 'Topics[?contains(TopicArn, `gameapi-backup`)].TopicArn'

# Check CloudFormation stack
aws cloudformation describe-stacks --stack-name gameapi-aws-backup-dev --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' --output text 2>/dev/null || echo "Stack deleted"
```
