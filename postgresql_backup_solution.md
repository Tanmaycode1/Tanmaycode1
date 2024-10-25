 # PostgreSQL Backup Automation with S3 and Cron for Production Environment

This document outlines a production-grade solution for automating PostgreSQL database backups in AWS EC2 environment. The solution includes a robust shell script for database backups, AWS S3 integration for redundant remote storage, and cron job scheduling with comprehensive monitoring.

## Table of Contents
1. [Shell Script Implementation](#1-shell-script-implementation)
2. [Cron Job Setup](#2-cron-job-setup)
3. [AWS IAM Configuration](#3-aws-iam-configuration)
4. [S3 Bucket Setup](#4-s3-bucket-setup)
5. [Monitoring and Alerting](#5-monitoring-and-alerting)
6. [Testing and Verification](#6-testing-and-verification)
7. [Backup Recovery Procedure](#7-backup-recovery-procedure)
8. [Maintenance and Troubleshooting](#8-maintenance-and-troubleshooting)

## 1. Shell Script Implementation

### backup_postgres.sh

```bash
#!/bin/bash

# Exit on any error
set -e

# Configuration variables
DB_NAME="your_db_name"
DB_USER="your_db_user"
DB_HOST="localhost"
DB_PORT="5432"
BACKUP_DIR="/path/to/backup/dir"
RETENTION_DAYS=7  # Local backup retention
DATE=$(date +"%Y%m%d%H%M")
BACKUP_FILE="$BACKUP_DIR/$DB_NAME-backup-$DATE.sql.gz"
S3_BUCKETS=("s3://bucket-1" "s3://bucket-2" "s3://bucket-3" "s3://bucket-4")
LOG_FILE="/var/log/pg_backup.log"

# Function for logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Function to send notification (replace with your preferred method)
notify() {
    if [ "$2" = "error" ]; then
        aws sns publish --topic-arn "your-sns-topic" --message "$1" || true
    fi
}

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Cleanup old local backups
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Check available disk space
SPACE_AVAILABLE=$(df -P "$BACKUP_DIR" | awk 'NR==2 {print $4}')
if [ "$SPACE_AVAILABLE" -lt 5242880 ]; then  # 5GB in KB
    log "ERROR: Insufficient disk space"
    notify "Insufficient disk space for PostgreSQL backup" "error"
    exit 1
fi

# Start backup
log "Starting backup of database $DB_NAME..."

# Get database size for progress monitoring
DB_SIZE=$(psql -U "$DB_USER" -h "$DB_HOST" -p "$DB_PORT" -t -c "SELECT pg_database_size('$DB_NAME')")

# Perform backup with compression and progress monitoring
pg_dump -U "$DB_USER" -h "$DB_HOST" -p "$DB_PORT" \
    --format=plain \
    --no-owner \
    --no-acl \
    "$DB_NAME" 2>> "$LOG_FILE" | gzip > "$BACKUP_FILE"

if [ ${PIPESTATUS[0]} -eq 0 ] && [ ${PIPESTATUS[1]} -eq 0 ]; then
    log "Backup created successfully: $BACKUP_FILE"
    
    # Verify backup integrity
    if gunzip -t "$BACKUP_FILE" 2>/dev/null; then
        log "Backup file integrity verified"
    else
        log "ERROR: Backup file is corrupted"
        notify "PostgreSQL backup file is corrupted" "error"
        exit 1
    fi
else
    log "ERROR: Failed to create backup"
    notify "PostgreSQL backup creation failed" "error"
    exit 1
fi

# Upload to S3 with verification
for BUCKET in "${S3_BUCKETS[@]}"; do
    log "Uploading backup to $BUCKET..."
    
    # Upload with metadata
    aws s3 cp "$BACKUP_FILE" "$BUCKET/$(basename "$BACKUP_FILE")" \
        --metadata "database=$DB_NAME,timestamp=$DATE,size=$DB_SIZE" \
        --storage-class STANDARD_IA \
        --retry 3

    # Verify upload
    if aws s3 ls "$BUCKET/$(basename "$BACKUP_FILE")" >/dev/null 2>&1; then
        log "Successfully uploaded to $BUCKET"
    else
        log "ERROR: Failed to verify upload to $BUCKET"
        notify "Failed to upload PostgreSQL backup to $BUCKET" "error"
        exit 1
    fi
done

# Keep last 2 local backups and remove older ones
cd "$BACKUP_DIR" && ls -t *.sql.gz | tail -n +3 | xargs -r rm --

log "Backup process completed successfully"
```

### Script Features

- Automated compression using gzip
- Comprehensive error handling and logging
- Backup integrity verification
- Disk space checking
- S3 upload verification
- Local backup rotation
- SNS notifications for failures
- Metadata tagging for S3 objects

## 2. Cron Job Setup

### Installation

1. Make the script executable:
```bash
chmod +x /path/to/backup_postgres.sh
```

2. Open crontab editor:
```bash
crontab -e
```

3. Add the backup schedule (runs at 2 AM daily):
```bash
0 2 * * * /path/to/backup_postgres.sh
```

### Alternative Schedules

- Hourly: `0 * * * *`
- Twice daily: `0 */12 * * *`
- Weekly: `0 2 * * 0`
- Monthly: `0 2 1 * *`

## 3. AWS IAM Configuration

### Required IAM Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-1/*",
                "arn:aws:s3:::bucket-2/*",
                "arn:aws:s3:::bucket-3/*",
                "arn:aws:s3:::bucket-4/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": "arn:aws:sns:region:account-id:topic-name"
        }
    ]
}
```

### EC2 Instance Role Setup

1. Create an IAM role for EC2
2. Attach the above policy
3. Assign role to EC2 instance

## 4. S3 Bucket Setup

### Bucket Configuration

For each backup bucket:

1. Enable versioning:
```bash
aws s3api put-bucket-versioning \
    --bucket your-bucket-name \
    --versioning-configuration Status=Enabled
```

2. Apply lifecycle rules:
```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket your-bucket-name \
    --lifecycle-configuration file://lifecycle.json
```

### Lifecycle Configuration (lifecycle.json)

```json
{
    "Rules": [
        {
            "ID": "Move old backups to Glacier",
            "Status": "Enabled",
            "Filter": {
                "Prefix": ""
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "GLACIER"
                }
            ]
        }
    ]
}
```

## 5. Monitoring and Alerting

### CloudWatch Alarm Setup

1. Create metric filter for backup failures:
```bash
aws logs create-metric-filter \
    --log-group-name /var/log/pg_backup.log \
    --filter-name backup-errors \
    --filter-pattern "ERROR" \
    --metric-transformations \
        metricName=BackupErrors,metricNamespace=Database,metricValue=1
```

2. Create alarm:
```bash
aws cloudwatch put-metric-alarm \
    --alarm-name postgresql-backup-failure \
    --metric-name BackupErrors \
    --namespace Database \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --period 300 \
    --statistic Sum \
    --alarm-actions arn:aws:sns:region:account-id:topic-name
```

## 6. Testing and Verification

### Manual Testing Procedure

1. Run backup script manually:
```bash
./backup_postgres.sh
```

2. Verify backup file creation:
```bash
ls -l $BACKUP_DIR
```

3. Check S3 uploads:
```bash
for bucket in "${S3_BUCKETS[@]}"; do
    aws s3 ls "$bucket" --recursive
done
```

### Backup Verification

Test restore to a temporary database:

```bash
# Create test database
createdb test_restore

# Restore backup
gunzip -c latest-backup.sql.gz | psql -d test_restore

# Verify data
psql -d test_restore -c "SELECT count(*) FROM your_table;"
```

## 7. Backup Recovery Procedure

### Restore from Local Backup

```bash
# Stop application
systemctl stop your-app

# Create clean database
dropdb your_database
createdb your_database

# Restore backup
gunzip -c backup-file.sql.gz | psql -d your_database

# Start application
systemctl start your-app
```

### Restore from S3

```bash
# Download from S3
aws s3 cp s3://your-bucket/backup-file.sql.gz .

# Restore using same procedure as local backup
gunzip -c backup-file.sql.gz | psql -d your_database
```

## 8. Maintenance and Troubleshooting

### Common Issues and Solutions

1. **Insufficient Disk Space**
   - Check disk usage: `df -h`
   - Clean old backups: `find $BACKUP_DIR -mtime +7 -delete`

2. **S3 Upload Failures**
   - Verify IAM roles: `aws sts get-caller-identity`
   - Check S3 bucket permissions
   - Verify network connectivity

3. **PostgreSQL Connection Issues**
   - Check PostgreSQL service: `systemctl status postgresql`
   - Verify credentials in `pg_hba.conf`
   - Test connection: `psql -U $DB_USER -d $DB_NAME`

### Maintenance Checklist

Weekly:
- Review backup logs
- Verify backup sizes
- Check S3 storage usage

Monthly:
- Test backup restoration
- Update IAM roles if needed
- Review and optimize retention policies

### Performance Optimization

1. **Backup Window**
   - Schedule during low-traffic periods
   - Monitor backup duration
   - Adjust compression levels if needed

2. **Resource Usage**
   - Monitor CPU/Memory during backups
   - Adjust nice level if needed
   - Consider IO scheduling

3. **Storage Optimization**
   - Use STANDARD_IA for cost savings
   - Implement lifecycle policies
   - Monitor storage costs

---

This solution provides a robust, production-ready backup strategy for PostgreSQL databases running on AWS EC2 instances. The implementation includes comprehensive error handling, monitoring, and verification steps to ensure reliable backups.