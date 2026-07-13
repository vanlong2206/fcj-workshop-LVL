---
title: "Dọn dẹp tài nguyên"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---
#### 5.11.1 Xóa Aurora PostgreSQL Restore Test Cluster

Kiểm tra cluster test còn tồn tại không:

```bash
aws rds describe-db-clusters \
  --region ap-southeast-1 \
  --query 'DBClusters[?contains(DBClusterIdentifier, `restore-test`)].DBClusterIdentifier' \
  --output table
```

Nếu có, xóa cluster:

```bash
aws rds delete-db-cluster \
  --db-cluster-identifier quan-restore-test \
  --skip-final-snapshot \
  --region ap-southeast-1
```

> Nếu cluster có instance riêng, phải xóa instance trước:
>
> ```bash
> aws rds delete-db-instance \
>   --db-instance-identifier <instance-name> \
>   --skip-final-snapshot \
>   --region ap-southeast-1
> ```

#### 5.11.2 Xóa các Recovery Point trong backup vault

Backup vault chứa recovery point sẽ không cho xóa. Phải xóa recovery points trước.

Liệt kê recovery points:

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name gameapi-aurora-vault \
  --region ap-southeast-1 \
  --query 'RecoveryPoints[].RecoveryPointArn' \
  --output text
```

Xóa từng recovery point:

```bash
aws backup delete-recovery-point \
  --backup-vault-name gameapi-aurora-vault \
  --recovery-point-arn <thay-bang-arn-tu-buoc-tren> \
  --region ap-southeast-1
```

Lặp lại cho đến khi vault không còn recovery point nào.

#### 5.11.3 Xóa CloudWatch Alarms

Xóa 2 alarm đã tạo:

```bash
aws cloudwatch delete-alarms \
  --alarm-names gameapi-backup-job-failed gameapi-restore-job-failed \
  --region ap-southeast-1
```

#### 5.11.4 Xóa SNS Topic và Email Subscription

Liệt kê subscription:

```bash
SNS_ARN=$(aws sns list-topics \
  --region ap-southeast-1 \
  --query 'Topics[?contains(TopicArn, `gameapi-backup-notifications`)].TopicArn' \
  --output text)

aws sns list-subscriptions-by-topic \
  --topic-arn "$SNS_ARN" \
  --region ap-southeast-1
```

Hủy subscription:

```bash
aws sns unsubscribe \
  --subscription-arn <SubscriptionArn> \
  --region ap-southeast-1
```

Xóa SNS topic:

```bash
aws sns delete-topic \
  --topic-arn "$SNS_ARN" \
  --region ap-southeast-1
```

#### 5.11.5 Xóa Backup Plan và Backup Vault

Liệt kê backup plan ID:

```bash
aws backup list-backup-plans \
  --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi-aurora-backup-plan`)].BackupPlanId' \
  --output text
```

Xóa backup plan (cần thêm tham số `--deletion-window-in-days`):

```bash
aws backup delete-backup-plan \
  --backup-plan-id <BackupPlanId> \
  --region ap-southeast-1
```

Xóa backup vault (chỉ thực hiện được nếu vault không còn recovery point):

```bash
aws backup delete-backup-vault \
  --backup-vault-name gameapi-aurora-vault \
  --region ap-southeast-1
```

#### 5.11.6 Xóa KMS Key

Nếu muốn xóa luôn KMS key (lưu ý: không thể undo):

```bash
# Lấy Key ID từ alias
KEY_ID=$(aws kms list-aliases \
  --region ap-southeast-1 \
  --query 'Aliases[?AliasName==`alias/gameapi-aurora-backup`].TargetKeyId' \
  --output text)

# Lên lịch xóa key sau 7 ngày (mặc định)
aws kms schedule-key-deletion \
  --key-id "$KEY_ID" \
  --pending-window-in-days 7 \
  --region ap-southeast-1
```

#### 5.11.7 Gỡ toàn bộ stack bằng Serverless Framework (cách nhanh nhất)

Cách này thực hiện tự động tất cả các bước trên (trừ xóa cluster test):

```bash
cd services/aws-backup-infrastructure
npx serverless remove --stage dev
```

> **Lưu ý:** Nếu vault còn recovery point, lệnh này sẽ fail. Phải làm bước 2 trước.

#### 5.11.8 Kiểm tra không còn tài nguyên nào sót

```bash
# Kiểm tra vault
aws backup list-backup-vaults --region ap-southeast-1 \
  --query 'BackupVaults[?contains(BackupVaultName, `gameapi`)].BackupVaultName'

# Kiểm tra plan
aws backup list-backup-plans --region ap-southeast-1 \
  --query 'BackupPlansList[?contains(BackupPlanName, `gameapi`)].BackupPlanName'

# Kiểm tra SNS
aws sns list-topics --region ap-southeast-1 \
  --query 'Topics[?contains(TopicArn, `gameapi-backup`)].TopicArn'

# Kiểm tra CloudFormation stack
aws cloudformation describe-stacks --stack-name gameapi-aws-backup-dev --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' --output text 2>/dev/null || echo " Stack đã xóa"
```
