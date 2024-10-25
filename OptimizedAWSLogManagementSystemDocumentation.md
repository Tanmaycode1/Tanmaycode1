# AWS Log Review System - Complete Documentation

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Prerequisites](#2-prerequisites)
3. [Installation Guide](#3-installation-guide)
4. [Configuration](#4-configuration)
5. [System Components](#5-system-components)
6. [Usage Guide](#6-usage-guide)
7. [Maintenance](#7-maintenance)
8. [Troubleshooting](#8-troubleshooting)
9. [Security Considerations](#9-security-considerations)
10. [Compliance](#10-compliance)
11. [Cost Optimization](#11-cost-optimization)

## 1. System Overview

### 1.1 Purpose
This system provides automated daily log review reports for:
- System Administrator Logs
- Operator Logs

### 1.2 Architecture
```plaintext
CloudWatch Logs → Lambda → S3 (Evidence Storage)
           ↑                    ↓
    Log Sources          Evidence Reports
```

### 1.3 Key Features
- Automated daily log reviews
- Evidence generation and storage
- Configurable alert thresholds
- Compliance-ready reporting
- Secure storage of review evidence

## 2. Prerequisites

### 2.1 AWS Services Required
- AWS CloudWatch Logs
- AWS Lambda
- Amazon S3
- AWS IAM
- Amazon EventBridge

### 2.2 Permissions Required
- CloudWatch Logs access
- S3 bucket creation and management
- Lambda function creation and management
- IAM role creation
- EventBridge rule creation

### 2.3 Resource Requirements
```yaml
Minimum Lambda Configuration:
  Memory: 256 MB
  Timeout: 5 minutes
  Runtime: Python 3.9+
```

## 3. Installation Guide

### 3.1 CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Log Review System Infrastructure'

Resources:
  # Log Groups
  SystemAdminLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /system/admin/logs
      RetentionInDays: 90
      
  OperatorLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /system/operator/logs
      RetentionInDays: 90

  # S3 Bucket for Evidence
  LogReviewEvidenceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-log-review-evidence'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  # Lambda Function Role
  LogReviewLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: LogReviewPermissions
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:StartQuery
                  - logs:GetQueryResults
                Resource:
                  - !GetAtt SystemAdminLogGroup.Arn
                  - !GetAtt OperatorLogGroup.Arn
              - Effect: Allow
                Action:
                  - s3:PutObject
                Resource: !Sub '${LogReviewEvidenceBucket.Arn}/*'

  # Lambda Function
  LogReviewFunction:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.lambda_handler
      Role: !GetAtt LogReviewLambdaRole.Arn
      Code:
        ZipFile: |
          [Lambda Function Code Goes Here]
      Runtime: python3.9
      Timeout: 300
      MemorySize: 256
      Environment:
        Variables:
          EVIDENCE_BUCKET: !Ref LogReviewEvidenceBucket

  # EventBridge Rule
  DailyLogReviewRule:
    Type: AWS::Events::Rule
    Properties:
      Name: daily-log-review-trigger
      Description: "Triggers daily log review Lambda function"
      ScheduleExpression: "cron(0 1 * * ? *)"
      State: ENABLED
      Targets:
        - Arn: !GetAtt LogReviewFunction.Arn
          Id: "DailyLogReviewLambda"
```

### 3.2 Deployment Steps

1. **Prepare Environment**:
```bash
# Create deployment bucket
aws s3 mb s3://log-review-deployment-bucket

# Package Lambda code
zip -r function.zip .
```

2. **Deploy CloudFormation**:
```bash
aws cloudformation create-stack \
  --stack-name log-review-system \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM
```

3. **Verify Deployment**:
```bash
aws cloudformation describe-stacks \
  --stack-name log-review-system
```

## 4. Configuration

### 4.1 Lambda Environment Variables
```plaintext
EVIDENCE_BUCKET: Name of S3 bucket for storing evidence
LOG_LEVEL: INFO (or DEBUG for troubleshooting)
RETENTION_DAYS: 90
```

### 4.2 Log Query Configuration
```python
DEFAULT_QUERY = """
    fields @timestamp, @message
    | filter @message like /error/ 
        or @message like /warning/ 
        or @message like /critical/
    | sort @timestamp desc
"""
```

## 5. System Components

### 5.1 Lambda Function Code
```python
[Previous Lambda Function Code]
```

### 5.2 Evidence Report Format
```plaintext
Daily Log Review Report - {DATE}
==================================================

ADMIN LOGS REVIEW
--------------------
[Findings]

OPERATOR LOGS REVIEW
--------------------
[Findings]

Review Metadata:
- Review Date: {DATE}
- Review Time: {TIME}
- Reviewer: AWS Lambda Automated Review
- Report ID: {UUID}
```

## 6. Usage Guide

### 6.1 Accessing Reports
```bash
# Download latest report
aws s3 cp s3://log-review-evidence/daily-reviews/latest/log-review-evidence.txt .

# List all reports
aws s3 ls s3://log-review-evidence/daily-reviews/ --recursive
```

### 6.2 Manual Trigger
```bash
# Trigger Lambda function manually
aws lambda invoke \
  --function-name log-review-function \
  --payload '{}' \
  response.json
```

## 7. Maintenance

### 7.1 Regular Tasks
1. Review and update retention policies
2. Check S3 bucket size and cleanup if needed
3. Update Lambda function configurations
4. Review IAM permissions

### 7.2 Monitoring
```bash
# Check Lambda execution logs
aws logs get-log-events \
  --log-group-name /aws/lambda/log-review-function \
  --log-stream-name [STREAM_NAME]
```

## 8. Troubleshooting

### 8.1 Common Issues

1. **Lambda Timeouts**
   - Increase Lambda timeout
   - Optimize query performance
   - Check log volume

2. **Missing Logs**
   - Verify log group names
   - Check retention periods
   - Validate IAM permissions

3. **Failed Reports**
   - Check S3 permissions
   - Verify bucket exists
   - Check Lambda role

### 8.2 Debugging Steps
```bash
# Enable DEBUG logging
aws lambda update-function-configuration \
  --function-name log-review-function \
  --environment Variables={LOG_LEVEL=DEBUG}
```

## 9. Security Considerations

### 9.1 Data Protection
- All reports encrypted at rest
- TLS for data in transit
- S3 bucket versioning enabled
- Access logging enabled

### 9.2 Access Control
- Least privilege IAM roles
- S3 bucket policies
- CloudWatch Logs encryption

## 10. Compliance

### 10.1 Report Retention
- Reports retained for 90 days
- Versioning enabled
- Audit trail maintained

### 10.2 Evidence Format
- Timestamp on all entries
- Reviewer identification
- Unique report identifiers

## 11. Cost Optimization

### 11.1 Cost Components
- CloudWatch Logs storage
- Lambda execution
- S3 storage
- Data transfer

### 11.2 Optimization Tips
1. Adjust retention periods
2. Optimize Lambda memory
3. Use S3 lifecycle policies
4. Monitor and adjust based on usage

## 12. API Reference

### 12.1 Lambda Function API
```python
def lambda_handler(event, context):
    """
    Entry point for the Log Review Lambda function
    
    Parameters:
    event (dict): AWS Lambda event object
    context (LambdaContext): AWS Lambda context object
    
    Returns:
    dict: Response object with status code and message
    """
```

### 12.2 CloudWatch Logs Query API
```python
def query_logs(log_group, start_time, end_time):
    """
    Query CloudWatch Logs
    
    Parameters:
    log_group (str): Name of the log group to query
    start_time (int): Start time in Unix timestamp
    end_time (int): End time in Unix timestamp
    
    Returns:
    dict: Query results
    """
```

## 13. Change Management

### 13.1 Version Control
All changes should be tracked in version control:
```bash
git add .
git commit -m "Update log review system"
git tag v1.0.0
```

### 13.2 Deployment Process
1. Test changes in development
2. Update documentation
3. Create CloudFormation change set
4. Review and apply changes
5. Verify deployment
