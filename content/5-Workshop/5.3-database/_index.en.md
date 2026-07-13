---
title : "Database Setup and Backup"
date : 2026-06-18
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---
#### 5.3.1 Database Initialization

The Database Layer includes: Aurora PostgreSQL database, RDS Proxy for connection pooling, and AWS Backup integration to automatically push backups to S3 Storage.

Return to the ap-southeast-1 region and build the system core here.

![Database_Create](images/Database_Create_1.png)

<div align="center"><i>Figure 5.3.1: Create database with Aurora engine (PostgreSQL Compatible).</i></div>

![Database_Create](images/Database_Create_2.png)

<div align="center"><i>Figure 5.3.2: Configure basic database parameters.</i></div>

Create a `.yaml` file `backup-s3-stack.yaml` with the following content:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Database Backup Layer: AWS Backup + Amazon S3 Bucket'

Parameters:
  TargetDatabaseClusterArn:
    Type: String
    Description: "ARN of the Aurora Cluster"

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

  # Backup Schedule
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

  # Link to Aurora Cluster
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

<div align="center"><i>Figure 5.3.3: Create CloudFormation Stack and upload the backup-s3-stack.yaml file.</i></div>

![Backup_Stack](images/Backup_Stack_2.png)

<div align="center"><i>Figure 5.3.4: Name the stack and enter the Cluster ARN (instructions in Figure 5.3.5).</i></div>

![Get_Database_Arn](images/Get_Database_Arn.png)

<div align="center"><i>Figure 5.3.5: Get the Cluster ARN from the Configuration tab.</i></div>

![Backup_Stack](images/Backup_Stack_3.png)

<div align="center"><i>Figure 5.3.6: Confirm information and Submit.</i></div>

![Confirm_Backup_Stack](images/Confirm_Backup_Stack.png)

<div align="center"><i>Figure 5.3.7: Result.</i></div>

![IAM_Policy_Update](images/IAM_Policy_Update.png)

<div align="center"><i>Figure 5.3.8: Update cluster-id and database-user-name in the Backend IAM Policy from section 2.1.</i></div>

cluster-id is obtained from the Configuration tab of the database.
