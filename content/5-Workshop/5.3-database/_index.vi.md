---
title : "Xây dựng Cơ sở dữ liệu và Sao lưu"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### 5.3.1 Khởi tạo Cơ sở dữ liệu

Lớp CSDL sẽ bao gồm: CSDL Aurora PostgreSQL, RDS Proxy để gom kết nối và tích hợp AWS Backup tự động đẩy bản sao lưu sang S3 Storage.

Quay trở về khu vực ap-southeast-1 và xây dựng core hệ thống tại đây.

![Database_Create](images/Database_Create_1.png)
<div align="center"><i>Hình 5.3.1: Tạo database với engine Aurora (PostgreSQL Compatible).</i></div>

![Database_Create](images/Database_Create_2.png)
<div align="center"><i>Hình 5.3.2: Cấu hình các thông số cơ bản cho database.</i></div>

Tạo file .yaml `backup-s3-stack.yaml` với nội dung như sau:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Database Backup Layer: AWS Backup + Amazon S3 Bucket'

Parameters:
  TargetDatabaseClusterArn:
    Type: String
    Description: "ARN cua Aurora Cluster"

Resources:
  # S3 Bucket
  BackupS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "game-database-backup-storage-${AWS::AccountId}"
      VersioningConfiguration:
        Status: "Enabled"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  # AWS Backup Vault
  GameBackupVault:
    Type: AWS::Backup::BackupVault
    Properties:
      BackupVaultName: "GameDatabaseBackupVault"

  # Lịch trình Backup
  GameBackupPlan:
    Type: AWS::Backup::BackupPlan
    Properties:
      BackupPlan:
        BackupPlanName: "DailyDatabaseBackupPlan"
        BackupPlanRule:
          - RuleName: "DailyBackupRule"
            TargetBackupVault: !Ref GameBackupVault
            ScheduleExpression: "cron(0 22 * * ? *)"
            Lifecycle:
              DeleteAfterDays: 7

  # Liên kết tới Aurora Cluster
  GameBackupSelection:
    Type: AWS::Backup::BackupSelection
    Properties:
      BackupPlanId: !Ref GameBackupPlan
      BackupSelection:
        SelectionName: "AuroraClusterSelection"
        IamRoleArn: !Sub "arn:aws:iam::${AWS::AccountId}:role/service-role/AWSBackupDefaultServiceRole"
        Resources:
          - !Ref TargetDatabaseClusterArn

Outputs:
  S3BackupBucketName:
    Description: "S3 Bucket"
    Value: !Ref BackupS3Bucket
    Export:
      Name: Singapore-Backup-S3BucketName
```

![Backup_Stack](images/Backup_Stack_1.png)
<div align="center"><i>Hình 5.3.3: Tạo Stack CloudFormation và nạp file backup-s3-stack.yaml vào.</i></div>

![Backup_Stack](images/Backup_Stack_2.png)
<div align="center"><i>Hình 5.3.4: Đặt tên và nhập ARN của Cluster (hướng dẫn tại Hình 5.3.5).</i></div>

![Get_Database_Arn](images/Get_Database_Arn.png)
<div align="center"><i>Hình 5.3.5: Lấy ARN của Cluster tại tab Configuration.</i></div>

![Backup_Stack](images/Backup_Stack_3.png)
<div align="center"><i>Hình 5.3.6: Xác nhận thông tin và Submit.</i></div>

![Confirm_Backup_Stack](images/Confirm_Backup_Stack.png)
<div align="center"><i>Hình 5.3.7: Kết quả.</i></div>

![IAM_Policy_Update](images/IAM_Policy_Update.png)
<div align="center"><i>Hình 5.3.8: Cập nhật cluster-id và database-user-name trong IAM Policy của Backend tại phần 2.1.</i></div>

cluster-id lấy tại tab Configuration của database.