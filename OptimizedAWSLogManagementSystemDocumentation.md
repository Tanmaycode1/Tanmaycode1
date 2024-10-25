# AWS CloudWatch Log Review System
## Complete Implementation Guide with Functions and Testing

## Table of Contents
1. Core Functions
2. Testing Framework
3. Implementation Steps
4. Sample Test Cases

## 1. Core Functions

### 1.1 Log Retrieval Function
```python
def get_log_events(logs_client, log_group, start_time, end_time):
    """
    Retrieve log events from specified CloudWatch Log group.
    
    Args:
        logs_client: boto3 logs client
        log_group (str): CloudWatch Log group name
        start_time (int): Start timestamp in milliseconds
        end_time (int): End timestamp in milliseconds
    
    Returns:
        dict: Dictionary containing log events and metadata
    """
    try:
        response = logs_client.filter_log_events(
            logGroupName=log_group,
            startTime=start_time,
            endTime=end_time,
            limit=10000  # Adjust based on your needs
        )
        return {
            'status': 'success',
            'events': response.get('events', []),
            'metadata': {
                'count': len(response.get('events', [])),
                'log_group': log_group
            }
        }
    except Exception as e:
        return {
            'status': 'error',
            'error': str(e),
            'log_group': log_group
        }

def get_metric_data(cloudwatch_client, log_group, start_time, end_time):
    """
    Retrieve CloudWatch metrics for the specified log group.
    
    Args:
        cloudwatch_client: boto3 cloudwatch client
        log_group (str): Log group name
        start_time (datetime): Start time
        end_time (datetime): End time
    
    Returns:
        dict: Dictionary containing metric data
    """
    try:
        response = cloudwatch_client.get_metric_statistics(
            Namespace='AWS/Logs',
            MetricName='IncomingLogEvents',
            Dimensions=[{'Name': 'LogGroupName', 'Value': log_group}],
            StartTime=start_time,
            EndTime=end_time,
            Period=86400,
            Statistics=['Sum', 'Average']
        )
        return {
            'status': 'success',
            'metrics': response.get('Datapoints', []),
            'log_group': log_group
        }
    except Exception as e:
        return {
            'status': 'error',
            'error': str(e),
            'log_group': log_group
        }
```

### 1.2 Log Analysis Function
```python
def analyze_log_events(events, patterns=None):
    """
    Analyze log events for patterns and generate statistics.
    
    Args:
        events (list): List of log events
        patterns (dict): Optional dictionary of patterns to search for
    
    Returns:
        dict: Analysis results
    """
    if patterns is None:
        patterns = {
            'error': ['ERROR', 'CRITICAL', 'FATAL'],
            'warning': ['WARN', 'WARNING'],
            'info': ['INFO'],
            'debug': ['DEBUG']
        }
    
    analysis = {
        'total_events': len(events),
        'severity_counts': {k: 0 for k in patterns.keys()},
        'critical_events': [],
        'timestamp_range': {
            'start': None,
            'end': None
        }
    }
    
    for event in events:
        # Update timestamp range
        timestamp = event.get('timestamp', 0)
        if not analysis['timestamp_range']['start']:
            analysis['timestamp_range']['start'] = timestamp
        analysis['timestamp_range']['end'] = timestamp
        
        # Analyze message content
        message = event.get('message', '')
        
        # Check patterns
        for severity, pattern_list in patterns.items():
            if any(pattern in message for pattern in pattern_list):
                analysis['severity_counts'][severity] += 1
                
                # Store critical events for review
                if severity in ['error']:
                    analysis['critical_events'].append({
                        'timestamp': timestamp,
                        'message': message,
                        'log_stream': event.get('logStreamName', '')
                    })
    
    return analysis
```

### 1.3 Report Generation Function
```python
def generate_review_report(analysis_results, metric_data, compliance_info):
    """
    Generate formatted review report.
    
    Args:
        analysis_results (dict): Results from log analysis
        metric_data (dict): CloudWatch metric data
        compliance_info (dict): Compliance-related information
    
    Returns:
        dict: Formatted report
    """
    report = {
        'report_date': datetime.datetime.now().strftime('%Y-%m-%d'),
        'analysis_summary': {
            'total_events': analysis_results['total_events'],
            'error_count': analysis_results['severity_counts']['error'],
            'warning_count': analysis_results['severity_counts']['warning'],
            'time_range': {
                'start': datetime.datetime.fromtimestamp(
                    analysis_results['timestamp_range']['start']/1000
                ).strftime('%Y-%m-%d %H:%M:%S'),
                'end': datetime.datetime.fromtimestamp(
                    analysis_results['timestamp_range']['end']/1000
                ).strftime('%Y-%m-%d %H:%M:%S')
            }
        },
        'critical_events': analysis_results['critical_events'][:10],  # Top 10 critical events
        'metrics_summary': {
            'incoming_events_24h': sum(
                d['Sum'] for d in metric_data.get('metrics', [])
            ),
            'average_events_per_hour': sum(
                d['Average'] for d in metric_data.get('metrics', [])
            ) / 24 if metric_data.get('metrics') else 0
        },
        'compliance_status': compliance_info
    }
    
    return report
```

### 1.4 Storage and Notification Function
```python
def store_report(s3_client, report, bucket_name, report_date):
    """
    Store report in S3 bucket.
    
    Args:
        s3_client: boto3 s3 client
        report (dict): Report to store
        bucket_name (str): S3 bucket name
        report_date (str): Report date for key generation
    
    Returns:
        dict: Storage operation result
    """
    try:
        report_key = f"log-reviews/{report_date}/daily-review.json"
        s3_client.put_object(
            Bucket=bucket_name,
            Key=report_key,
            Body=json.dumps(report, indent=2),
            ContentType='application/json'
        )
        return {
            'status': 'success',
            'location': f"s3://{bucket_name}/{report_key}"
        }
    except Exception as e:
        return {
            'status': 'error',
            'error': str(e)
        }

def send_notification(sns_client, report, topic_arn):
    """
    Send notification about report generation.
    
    Args:
        sns_client: boto3 sns client
        report (dict): Generated report
        topic_arn (str): SNS topic ARN
    
    Returns:
        dict: Notification result
    """
    try:
        message = format_report_for_notification(report)
        response = sns_client.publish(
            TopicArn=topic_arn,
            Subject=f"Log Review Report - {report['report_date']}",
            Message=message
        )
        return {
            'status': 'success',
            'message_id': response['MessageId']
        }
    except Exception as e:
        return {
            'status': 'error',
            'error': str(e)
        }
```

## 2. Testing Framework

### 2.1 Test Setup
```python
import unittest
import boto3
from moto import mock_logs, mock_cloudwatch, mock_s3, mock_sns
from datetime import datetime, timedelta

class TestLogReviewSystem(unittest.TestCase):
    def setUp(self):
        """Set up test environment."""
        self.log_group_name = '/test/system/logs'
        self.bucket_name = 'test-log-reports'
        self.topic_arn = 'arn:aws:sns:us-east-1:123456789012:test-topic'
        
        # Initialize mock AWS services
        self.mock_logs = mock_logs()
        self.mock_cloudwatch = mock_cloudwatch()
        self.mock_s3 = mock_s3()
        self.mock_sns = mock_sns()
        
        # Start mocks
        self.mock_logs.start()
        self.mock_cloudwatch.start()
        self.mock_s3.start()
        self.mock_sns.start()
        
        # Create mock clients
        self.logs_client = boto3.client('logs')
        self.cloudwatch_client = boto3.client('cloudwatch')
        self.s3_client = boto3.client('s3')
        self.sns_client = boto3.client('sns')
        
        # Create test resources
        self.logs_client.create_log_group(logGroupName=self.log_group_name)
        self.s3_client.create_bucket(Bucket=self.bucket_name)
        
    def tearDown(self):
        """Clean up test environment."""
        self.mock_logs.stop()
        self.mock_cloudwatch.stop()
        self.mock_s3.stop()
        self.mock_sns.stop()
```

### 2.2 Test Cases
```python
def test_log_retrieval(self):
    """Test log event retrieval."""
    # Create test log stream
    stream_name = 'test-stream'
    self.logs_client.create_log_stream(
        logGroupName=self.log_group_name,
        logStreamName=stream_name
    )
    
    # Put test log events
    self.logs_client.put_log_events(
        logGroupName=self.log_group_name,
        logStreamName=stream_name,
        logEvents=[
            {
                'timestamp': int(datetime.now().timestamp() * 1000),
                'message': 'Test log message'
            }
        ]
    )
    
    # Test retrieval
    start_time = int((datetime.now() - timedelta(days=1)).timestamp() * 1000)
    end_time = int(datetime.now().timestamp() * 1000)
    
    result = get_log_events(
        self.logs_client,
        self.log_group_name,
        start_time,
        end_time
    )
    
    self.assertEqual(result['status'], 'success')
    self.assertEqual(len(result['events']), 1)

def test_log_analysis(self):
    """Test log analysis function."""
    test_events = [
        {
            'timestamp': int(datetime.now().timestamp() * 1000),
            'message': 'ERROR: Test error message',
            'logStreamName': 'test-stream'
        },
        {
            'timestamp': int(datetime.now().timestamp() * 1000),
            'message': 'WARNING: Test warning message',
            'logStreamName': 'test-stream'
        }
    ]
    
    analysis = analyze_log_events(test_events)
    
    self.assertEqual(analysis['total_events'], 2)
    self.assertEqual(analysis['severity_counts']['error'], 1)
    self.assertEqual(analysis['severity_counts']['warning'], 1)

def test_report_generation(self):
    """Test report generation."""
    analysis_results = {
        'total_events': 10,
        'severity_counts': {'error': 2, 'warning': 3, 'info': 5},
        'critical_events': [],
        'timestamp_range': {
            'start': int(datetime.now().timestamp() * 1000),
            'end': int(datetime.now().timestamp() * 1000)
        }
    }
    
    metric_data = {
        'metrics': [{'Sum': 100, 'Average': 4.17}]
    }
    
    compliance_info = {
        'review_completed': True,
        'reviewer': 'test-function',
        'review_timestamp': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    }
    
    report = generate_review_report(
        analysis_results,
        metric_data,
        compliance_info
    )
    
    self.assertIn('report_date', report)
    self.assertIn('analysis_summary', report)
    self.assertIn('metrics_summary', report)
```

## 3. Implementation Steps

1. Set up the AWS environment:
```bash
# Create log groups
aws logs create-log-group --log-group-name /aws/systemadmin/logs
aws logs create-log-group --log-group-name /aws/operator/logs

# Create S3 bucket
aws s3 mb s3://your-log-reports-bucket

# Create SNS topic
aws sns create-topic --name log-review-notifications
```

2. Create Lambda deployment package:
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install boto3

# Create deployment package
zip -r deployment.zip lambda_function.py
```

3. Deploy Lambda function:
```bash
aws lambda create-function \
    --function-name LogReviewSystem \
    --runtime python3.9 \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::YOUR_ACCOUNT_ID:role/LogReviewRole \
    --zip-file fileb://deployment.zip
```

4. Set up CloudWatch Events trigger:
```bash
aws events put-rule \
    --name DailyLogReview \
    --schedule-expression "cron(0 0 * * ? *)"

aws events put-targets \
    --rule DailyLogReview \
    --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT:function:LogReviewSystem"
```

## 4. Testing Instructions

### 4.1 Local Testing
```bash
# Run unit tests
python -m unittest test_log_review.py

# Run integration tests
python -m unittest test_integration.py
```

### 4.2 AWS Testing
1. Generate test logs:
```python
import boto3
import time

logs = boto3.client('logs')

# Create test log events
log_group = '/aws/systemadmin/logs'
stream_name = f"test-stream-{int(time.time())}"

logs.create_log_stream(
    logGroupName=log_group,
    logStreamName=stream_name
)

logs.put_log_events(
    logGroupName=log_group,
    logStreamName=stream_name,
    logEvents=[
        {
            'timestamp': int(time.time() * 1000),
            'message': 'ERROR: Test error message'
        },
        {
            'timestamp': int(time.time() * 1000),
            'message': 'WARNING: Test warning message'
        }
    ]
)
```

2. Trigger Lambda function:
```bash
aws lambda invoke \
    --function-name LogReviewSystem \
    --payload '{"test": true}' \
    response.json
```


3. Verify results:
```bash
# Check S3 for report
aws s3 ls s3://your-log-reports-bucket/log-reviews/

# Check CloudWatch Logs for Lambda execution
aws logs get-log-events \
    --log-group-name /aws/lambda/LogReviewSystem \
    --log-stream-name $(aws logs describe-log-streams \
        --log-group-name /aws/lambda/LogReviewSystem \
        --order-by LastEventTime \
        --descending \
        --limit 1 \
        --query 'logStreams[0].logStreamName' \
        --output text)

# Check Lambda metrics
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Duration \
    --dimensions Name=FunctionName,Value=LogReviewSystem \
    --start-time $(date -u -d "1 hour ago" +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 3600 \
    --statistics Average

# Verify SNS notifications
aws sns list-subscriptions-by-topic \
    --topic-arn arn:aws:sns:REGION:ACCOUNT:log-review-notifications
```

### 4.3 Create Verification Script
```python
def verify_log_review_system():
    """
    Comprehensive verification script for the log review system.
    """
    import boto3
    import json
    from datetime import datetime, timedelta
    
    class SystemVerification:
        def __init__(self):
            self.s3 = boto3.client('s3')
            self.logs = boto3.client('logs')
            self.cloudwatch = boto3.client('cloudwatch')
            self.sns = boto3.client('sns')
            
            self.bucket_name = 'your-log-reports-bucket'
            self.log_group = '/aws/systemadmin/logs'
            self.topic_arn = 'arn:aws:sns:REGION:ACCOUNT:log-review-notifications'
            
        def verify_s3_reports(self):
            """Verify S3 report generation"""
            try:
                # Get today's report
                today = datetime.now().strftime('%Y-%m-%d')
                response = self.s3.get_object(
                    Bucket=self.bucket_name,
                    Key=f'log-reviews/{today}/daily-review.json'
                )
                
                # Validate report content
                report = json.loads(response['Body'].read().decode('utf-8'))
                required_fields = [
                    'report_date',
                    'analysis_summary',
                    'critical_events',
                    'metrics_summary',
                    'compliance_status'
                ]
                
                validation = {
                    'exists': True,
                    'valid_format': all(field in report for field in required_fields),
                    'content_length': len(json.dumps(report))
                }
                
                return {
                    'status': 'success',
                    'validation': validation
                }
                
            except Exception as e:
                return {
                    'status': 'error',
                    'error': str(e)
                }
                
        def verify_log_processing(self):
            """Verify log processing functionality"""
            try:
                # Check recent log events
                end_time = int(datetime.now().timestamp() * 1000)
                start_time = int((datetime.now() - timedelta(hours=1)).timestamp() * 1000)
                
                response = self.logs.filter_log_events(
                    logGroupName=self.log_group,
                    startTime=start_time,
                    endTime=end_time
                )
                
                return {
                    'status': 'success',
                    'events_processed': len(response.get('events', [])),
                    'time_range': {
                        'start': start_time,
                        'end': end_time
                    }
                }
                
            except Exception as e:
                return {
                    'status': 'error',
                    'error': str(e)
                }
                
        def verify_metrics(self):
            """Verify CloudWatch metrics"""
            try:
                # Get Lambda execution metrics
                response = self.cloudwatch.get_metric_statistics(
                    Namespace='AWS/Lambda',
                    MetricName='Duration',
                    Dimensions=[
                        {
                            'Name': 'FunctionName',
                            'Value': 'LogReviewSystem'
                        }
                    ],
                    StartTime=datetime.now() - timedelta(hours=1),
                    EndTime=datetime.now(),
                    Period=3600,
                    Statistics=['Average', 'Maximum']
                )
                
                return {
                    'status': 'success',
                    'metrics': {
                        'datapoints': len(response['Datapoints']),
                        'average_duration': response['Datapoints'][0]['Average'] if response['Datapoints'] else None,
                        'max_duration': response['Datapoints'][0]['Maximum'] if response['Datapoints'] else None
                    }
                }
                
            except Exception as e:
                return {
                    'status': 'error',
                    'error': str(e)
                }
                
        def verify_notifications(self):
            """Verify SNS notifications"""
            try:
                # Check SNS topic subscriptions
                response = self.sns.list_subscriptions_by_topic(
                    TopicArn=self.topic_arn
                )
                
                return {
                    'status': 'success',
                    'subscriptions': len(response.get('Subscriptions', [])),
                    'subscription_details': response.get('Subscriptions', [])
                }
                
            except Exception as e:
                return {
                    'status': 'error',
                    'error': str(e)
                }
                
        def run_full_verification(self):
            """Run complete system verification"""
            results = {
                's3_reports': self.verify_s3_reports(),
                'log_processing': self.verify_log_processing(),
                'metrics': self.verify_metrics(),
                'notifications': self.verify_notifications()
            }
            
            # Calculate overall status
            overall_status = all(
                result['status'] == 'success' 
                for result in results.values()
            )
            
            # Generate verification report
            verification_report = {
                'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                'overall_status': 'PASSED' if overall_status else 'FAILED',
                'component_results': results
            }
            
            return verification_report

def print_verification_report(report):
    """Print formatted verification report"""
    print("\nLog Review System Verification Report")
    print("=====================================")
    print(f"Timestamp: {report['timestamp']}")
    print(f"Overall Status: {report['overall_status']}")
    print("\nComponent Results:")
    
    for component, result in report['component_results'].items():
        print(f"\n{component.upper()}:")
        print(f"Status: {result['status']}")
        if result['status'] == 'success':
            for key, value in result.items():
                if key != 'status':
                    print(f"{key}: {value}")
        else:
            print(f"Error: {result['error']}")

# Run verification
if __name__ == "__main__":
    verifier = SystemVerification()
    verification_report = verifier.run_full_verification()
    print_verification_report(verification_report)
```

### 4.4 Automated Testing Schedule
```bash
# Create CloudWatch Events rule for automated testing
aws events put-rule \
    --name DailyLogReviewTesting \
    --schedule-expression "cron(0 1 * * ? *)" \
    --state ENABLED

# Add target to the rule
aws events put-targets \
    --rule DailyLogReviewTesting \
    --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT:function:LogReviewTestFunction"

# Create IAM role for automated testing
aws iam create-role \
    --role-name LogReviewTestRole \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }'

# Attach necessary permissions
aws iam put-role-policy \
    --role-name LogReviewTestRole \
    --policy-name LogReviewTestPermissions \
    --policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "logs:*",
                    "s3:*",
                    "sns:*",
                    "cloudwatch:*",
                    "lambda:InvokeFunction"
                ],
                "Resource": "*"
            }
        ]
    }'
```
