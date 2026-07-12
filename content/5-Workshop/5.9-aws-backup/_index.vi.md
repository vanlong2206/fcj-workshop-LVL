---
title: "AWS Backup"
date: 2024-01-01
weight: 9
chapter: false
pre: " <b> 5.9. </b> "
---
#### AWS Backup

#### 5.9.1 Giới thiệu

**AWS Backup** là dịch vụ được quản lý hoàn toàn (fully managed) giúp tập trung hóa việc sao lưu dữ liệu trên nhiều dịch vụ AWS khác nhau. AWS Backup cho phép bạn định nghĩa các chính sách sao lưu (backup policies) tập trung, tự động hóa lịch sao lưu, và quản lý việc khôi phục (restore) dữ liệu một cách thống nhất.

#### 5.9.2 Kiến trúc AWS Backup

- **Backup Vault**: Nơi lưu trữ các bản sao lưu, có thể được mã hóa và quản lý quyền truy cập.
- **Backup Plan**: Định nghĩa lịch sao lưu, cửa sổ bảo trì, và chính sách retention (giữ lại bao lâu).
- **Backup Rule**: Mỗi Backup Plan chứa một hoặc nhiều rules, mỗi rule chỉ định:
  - Lịch schedule (cron expression)
  - Backup Vault đích
  - Lifecycle rules (chuyển sang cold storage và hết hạn)
- **Resource Assignment**: Gán Backup Plan vào các tài nguyên (RDS, DynamoDB, S3, EFS, EC2, ...).

#### 5.9.3 AWS Backup trong dự án FCAJ

Kiến trúc sao lưu sử dụng **AWS Backup** để tự động sao lưu toàn bộ dữ liệu game:

![Backup Architecture](image/_index.vi/backup-architecture.png)

<div align="center"><i>Hình 5.9.1: Sơ đồ kiến trúc AWS Backup.</i></div>

##### Các tài nguyên được sao lưu

| Tài nguyên | Dịch vụ | Mục đích |
|---|---|---|
| **Aurora PostgreSQL** | RDS | Dữ liệu game (người chơi, inventory, giao dịch) |
| **DynamoDB** | DynamoDB | Leaderboard, session data |
| **EFS** | EFS | Media assets, logs |
| **S3** | S3 | Backup files, static assets |

##### Backup Plan

```YAML
BackupPlanName: FCAJ-Backup-Plan
Rules:
  - RuleName: DailyBackup
    ScheduleExpression: cron(0 6 ? * * *)
    StartWindowMinutes: 60
    CompletionWindowMinutes: 480
    Lifecycle:
      DeleteAfterDays: 35
      MoveToColdStorageAfterDays: 7
    TargetBackupVault: FCAJ-Backup-Vault
    CopyActions:
      - DestinationBackupVaultArn: arn:aws:backup:ap-southeast-1:xxx:backup-vault:FCAJ-Backup-Vault-Dr
        Lifecycle:
          DeleteAfterDays: 90
          MoveToColdStorageAfterDays: 30
  - RuleName: WeeklyBackup
    ScheduleExpression: cron(0 8 ? * SUN *)
    StartWindowMinutes: 60
    CompletionWindowMinutes: 720
    Lifecycle:
      DeleteAfterDays: 90
      MoveToColdStorageAfterDays: 30
    TargetBackupVault: FCAJ-Backup-Vault
```

#### 5.9.4 Triển khai AWS Backup với Terraform

##### Khai báo Backup Vault

File: `infrastructure/terraform/modules/backup/main.tf`

```hcl
resource "aws_backup_vault" "main" {
  name        = "FCAJ-Backup-Vault"
  kms_key_arn = aws_kms_key.backup.arn

  tags = {
    Environment = var.environment
    Project     = "FCAJ"
  }
}

resource "aws_backup_vault" "dr" {
  name        = "FCAJ-Backup-Vault-Dr"
  kms_key_arn = aws_kms_key.backup_dr.arn

  tags = {
    Environment = var.environment
    Project     = "FCAJ"
  }
}
```

##### Khai báo Backup Plan

```hcl
resource "aws_backup_plan" "main" {
  name = "FCAJ-Backup-Plan"

  rule {
    rule_name         = "DailyBackup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 6 ? * * *)"

    start_window     = 60
    completion_window = 480

    lifecycle {
      delete_after       = 35
      move_to_cold_storage_after = 7
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle {
        delete_after       = 90
        move_to_cold_storage_after = 30
      }
    }
  }

  rule {
    rule_name         = "WeeklyBackup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 8 ? * SUN *)"

    start_window     = 60
    completion_window = 720

    lifecycle {
      delete_after       = 90
      move_to_cold_storage_after = 30
    }
  }
}
```

##### Gán tài nguyên vào Backup Plan

```hcl
resource "aws_backup_selection" "aurora" {
  plan_id      = aws_backup_plan.main.id
  name         = "Aurora-Backup"
  resources    = [aws_rds_cluster.main.arn]
  iam_role_arn = aws_iam_role.backup.arn
}

resource "aws_backup_selection" "dynamodb" {
  plan_id      = aws_backup_plan.main.id
  name         = "DynamoDB-Backup"
  resources    = [aws_dynamodb_table.leaderboard.arn, aws_dynamodb_table.sessions.arn]
  iam_role_arn = aws_iam_role.backup.arn
}

resource "aws_backup_selection" "efs" {
  plan_id      = aws_backup_plan.main.id
  name         = "EFS-Backup"
  resources    = [aws_efs_file_system.main.arn]
  iam_role_arn = aws_iam_role.backup.arn
}

resource "aws_backup_selection" "s3" {
  plan_id      = aws_backup_plan.main.id
  name         = "S3-Backup"
  resources    = [aws_s3_bucket.backups.arn]
  iam_role_arn = aws_iam_role.backup.arn
}
```

##### IAM Role cho AWS Backup

```hcl
resource "aws_iam_role" "backup" {
  name = "FCAJ-Backup-Role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "backup.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "backup" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"
}
```

#### 5.9.5 Restore dữ liệu

##### Restore Aurora từ Backup

```bash
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:ap-southeast-1:xxx:recovery-point:xxx \
  --metadata '{
    "DBClusterIdentifier": "fcaj-restored",
    "Engine": "aurora-postgresql",
    "VpcSecurityGroupIds": "sg-xxx",
    "DBSubnetGroupName": "fcaj-subnet-group"
  }' \
  --iam-role-arn arn:aws:iam::xxx:role/FCAJ-Backup-Role
```

##### Restore DynamoDB từ Backup

```bash
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:ap-southeast-1:xxx:recovery-point:xxx \
  --metadata '{
    "tableName": "fcaj-restored"
  }' \
  --iam-role-arn arn:aws:iam::xxx:role/FCAJ-Backup-Role
```

#### 5.9.6 Monitoring & Alerting

##### CloudWatch Metrics cho AWS Backup

| Metric | Mô tả |
|---|---|
| `NumberOfBackupJobsCompleted` | Số job backup thành công |
| `NumberOfBackupJobsFailed` | Số job backup thất bại |
| `NumberOfRestoreJobsCompleted` | Số job restore thành công |
| `NumberOfRestoreJobsFailed` | Số job restore thất bại |
| `BackupJobDuration` | Thời gian thực hiện backup |

##### CloudWatch Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "backup_failure" {
  alarm_name          = "FCAJ-Backup-Failure"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "NumberOfBackupJobsFailed"
  namespace           = "AWS/Backup"
  period              = 3600
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Cảnh báo khi có backup job thất bại"
  alarm_actions       = [aws_sns_topic.alarm.arn]
}
```

#### 5.9.7 Kiểm thử

##### Kiểm tra Backup Job

```json
{
  "BackupJobId": "xxx",
  "BackupVaultName": "FCAJ-Backup-Vault",
  "ResourceArn": "arn:aws:rds:ap-southeast-1:xxx:cluster:fcaj-db",
  "State": "COMPLETED",
  "PercentDone": "100.0",
  "BackupSizeInBytes": 1073741824,
  "ExpectedCompletionDate": "2026-07-12T07:00:00Z",
  "CompletionDate": "2026-07-12T06:45:30Z"
}
```

##### Kiểm tra Restore Job

```json
{
  "RestoreJobId": "xxx",
  "RecoveryPointArn": "arn:aws:backup:ap-southeast-1:xxx:recovery-point:xxx",
  "ResourceType": "Aurora",
  "State": "COMPLETED",
  "PercentDone": "100.0",
  "CompletionDate": "2026-07-12T08:00:00Z"
}
```

##### Các bước kiểm tra

1. **Kiểm tra Backup tự động** — Đợi theo schedule hoặc kích hoạt thủ công qua AWS Console
2. **Kiểm tra Backup thành công** — Vào AWS Backup Console > Jobs > kiểm tra trạng thái COMPLETED
3. **Kiểm tra Restore** — Thực hiện restore vào môi trường staging, kiểm tra tính toàn vẹn dữ liệu
4. **Kiểm tra CloudWatch Alarm** — Mô phỏng backup failure để kiểm tra alarm hoạt động
