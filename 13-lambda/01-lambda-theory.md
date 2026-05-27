# AWS Lambda — Complete Guide

## Table of Contents
1. [What is AWS Lambda?](#1-what-is-aws-lambda)
2. [Lambda Architecture](#2-lambda-architecture)
3. [Lambda Execution Model](#3-lambda-execution-model)
4. [Lambda Configuration](#4-lambda-configuration)
5. [Lambda Triggers and Event Sources](#5-lambda-triggers-and-event-sources)
6. [Lambda Layers](#6-lambda-layers)
7. [Lambda Destinations and Error Handling](#7-lambda-destinations-and-error-handling)
8. [Lambda Concurrency](#8-lambda-concurrency)
9. [Lambda VPC Integration](#9-lambda-vpc-integration)
10. [Lambda with API Gateway](#10-lambda-with-api-gateway)
11. [Lambda Pricing](#11-lambda-pricing)
12. [Lambda Best Practices](#12-lambda-best-practices)
13. [Real-World Use Cases](#13-real-world-use-cases)
14. [Detailed Workflows](#14-detailed-workflows)
15. [Common Interview Questions](#15-common-interview-questions)

---

## 1. What is AWS Lambda?

**AWS Lambda** is a serverless compute service that runs your code in response to events without provisioning or managing servers. You pay only for the compute time consumed.

```
Traditional Server Model            Lambda (Serverless)
──────────────────────────────      ──────────────────────────────
Provision EC2 instance              No server management
Pay 24/7 even when idle             Pay only when code runs
Scale manually or with ASG          Auto-scales automatically
OS patching & maintenance           AWS manages everything
Always-on process                   Event-driven execution
```

**Key characteristics:**
- **Event-driven**: Code runs in response to triggers
- **Stateless**: Each invocation is independent
- **Ephemeral**: Execution environment exists only during invocation
- **Auto-scaling**: Scales from 0 to thousands of concurrent executions
- **Managed runtime**: AWS handles OS, patching, security

**Supported Runtimes:**
| Runtime | Versions |
|---------|----------|
| Python | 3.8, 3.9, 3.10, 3.11, 3.12 |
| Node.js | 18.x, 20.x |
| Java | 8, 11, 17, 21 |
| Go | 1.x |
| Ruby | 3.2 |
| .NET | 6, 8 |
| Custom Runtime | Any (via Lambda Runtime API) |

---

## 2. Lambda Architecture

```
                    ┌─────────────────────────────────────────┐
                    │            AWS Lambda Service             │
                    │                                           │
  Event Source ─────┤──► Event ──► Lambda Function Handler     │
  (S3, API GW,      │              │                           │
   DynamoDB, etc.)  │              ▼                           │
                    │         Execution Environment             │
                    │         ┌─────────────────┐             │
                    │         │ Init Phase       │             │
                    │         │ • Download code  │             │
                    │         │ • Start runtime  │             │
                    │         │ • Run init code  │             │
                    │         └────────┬────────┘             │
                    │                  │                        │
                    │         ┌────────▼────────┐             │
                    │         │ Invoke Phase     │             │
                    │         │ • Run handler()  │             │
                    │         │ • Return result  │             │
                    │         └────────┬────────┘             │
                    │                  │                        │
                    │         ┌────────▼────────┐             │
                    │         │ Shutdown Phase   │             │
                    │         │ • Cleanup        │             │
                    │         │ • Freeze env     │             │
                    │         └─────────────────┘             │
                    └─────────────────────────────────────────┘
```

### Lambda Function Anatomy

```python
import json
import boto3

# INIT PHASE — runs once per cold start
# This code runs outside the handler
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('MyTable')

# HANDLER — runs on every invocation
def lambda_handler(event, context):
    """
    event  - dict with trigger data
    context - runtime info (request_id, remaining_time, etc.)
    """
    print(f"Event: {json.dumps(event)}")
    print(f"Request ID: {context.aws_request_id}")
    print(f"Remaining time: {context.get_remaining_time_in_millis()}ms")
    
    # Business logic
    item = table.get_item(Key={'id': event['id']})
    
    # Return value (for sync invocations)
    return {
        'statusCode': 200,
        'body': json.dumps({'data': item.get('Item', {})})
    }
```

---

## 3. Lambda Execution Model

### Cold Start vs Warm Start

```
COLD START (first invocation or after idle period)
──────────────────────────────────────────────────
1. Download function code from S3         ~100-500ms
2. Start execution environment (MicroVM)  ~50-100ms
3. Start language runtime (JVM, etc.)     ~50-1000ms
4. Run initialization code (imports)      ~10-500ms
5. Run handler function                   your code time
───────────────────────────────────────────────────
Total: 200ms to 2+ seconds (Java worst)

WARM START (subsequent invocations, env reused)
───────────────────────────────────────────────
1. Run handler function only              your code time
───────────────────────────────────────────────
Total: milliseconds
```

**Reducing Cold Starts:**
- Use Provisioned Concurrency (pre-warm environments)
- Keep functions small (less to download)
- Use interpreted languages (Python/Node.js) over JVM
- Minimize imports — only import what you need
- Use Lambda SnapStart (Java 11+)

### Invocation Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Synchronous** | Caller waits for response | API Gateway, CLI |
| **Asynchronous** | Lambda queues, no wait | S3 events, SNS |
| **Event source mapping** | Lambda polls for records | SQS, DynamoDB Streams, Kinesis |

---

## 4. Lambda Configuration

### Core Settings

| Setting | Description | Limit |
|---------|-------------|-------|
| **Memory** | RAM allocation (also scales CPU) | 128MB – 10,240MB |
| **Timeout** | Max execution time | 1s – 900s (15 min) |
| **Ephemeral Storage** | /tmp directory size | 512MB – 10,240MB |
| **Concurrency** | Simultaneous executions | 1,000 per region (default) |
| **Package size** | Deployment zip (unzipped) | 50MB zip / 250MB unzipped |
| **Container image** | Container-based Lambda | Up to 10GB |

### Environment Variables

```bash
# Set via Console, CLI, or IaC
aws lambda update-function-configuration \
  --function-name my-function \
  --environment "Variables={DB_HOST=mydb.example.com,LOG_LEVEL=INFO}"

# Access in code
import os
db_host = os.environ['DB_HOST']
log_level = os.environ.get('LOG_LEVEL', 'INFO')
```

**Encrypting env vars:** Use KMS CMK for sensitive values (DB passwords, API keys)

### IAM Execution Role

Every Lambda function needs an **execution role** defining what AWS services it can access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/MyTable"
    }
  ]
}
```

---

## 5. Lambda Triggers and Event Sources

### Event Source Overview

```
                        ┌─────────────────────┐
  ┌──────────────┐      │                     │      ┌──────────────┐
  │  API Gateway │─────►│                     │─────►│   DynamoDB   │
  └──────────────┘      │                     │      └──────────────┘
  ┌──────────────┐      │   Lambda Function   │      ┌──────────────┐
  │  S3 Events   │─────►│                     │─────►│     SQS      │
  └──────────────┘      │                     │      └──────────────┘
  ┌──────────────┐      │                     │      ┌──────────────┐
  │  CloudWatch  │─────►│                     │─────►│     SNS      │
  │  Events/Sched│      │                     │      └──────────────┘
  └──────────────┘      └─────────────────────┘
  ┌──────────────┐
  │  SQS Queue   │ (Event Source Mapping — Lambda polls)
  └──────────────┘
  ┌──────────────┐
  │  Kinesis     │ (Event Source Mapping — Lambda polls)
  └──────────────┘
  ┌──────────────┐
  │  DynamoDB    │ (Event Source Mapping — Lambda polls streams)
  │  Streams     │
  └──────────────┘
```

### S3 Trigger Event Example

```json
{
  "Records": [
    {
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-bucket",
          "arn": "arn:aws:s3:::my-bucket"
        },
        "object": {
          "key": "uploads/photo.jpg",
          "size": 1024,
          "eTag": "abc123"
        }
      }
    }
  ]
}
```

### SQS Event Source Mapping

```python
def lambda_handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        message_id = record['messageId']
        
        try:
            process_message(body)
            # SQS message auto-deleted on success
        except Exception as e:
            # Raise exception = message goes back to queue
            # After maxReceiveCount, goes to DLQ
            raise e
    
    # Return batch item failures for partial success
    return {
        'batchItemFailures': []  # empty = all succeeded
    }
```

### EventBridge (CloudWatch Events) Scheduled Trigger

```json
{
  "source": ["aws.events"],
  "detail-type": ["Scheduled Event"],
  "resources": ["arn:aws:events:us-east-1:123456789:rule/my-schedule"]
}
```

Cron expression examples:
```
rate(5 minutes)          # Every 5 minutes
rate(1 day)              # Every day
cron(0 12 * * ? *)       # Every day at noon UTC
cron(0 8 ? * MON-FRI *)  # 8AM UTC weekdays
```

---

## 6. Lambda Layers

Layers allow sharing code and dependencies across multiple functions:

```
Without Layers:               With Layers:
─────────────────             ─────────────────────────────────
Function A (50MB)             Function A (5MB) ─► Layer (45MB)
  ├── my_code.py                ├── my_code.py
  ├── pandas/                 Function B (5MB) ─► Layer (45MB)
  ├── numpy/                    ├── my_code.py
  └── requests/              Layer contains:
                                ├── pandas/
Function B (50MB)               ├── numpy/
  ├── my_code.py               └── requests/
  ├── pandas/
  ├── numpy/
  └── requests/
```

**Creating a Layer:**
```bash
# Create layer zip
mkdir python && pip install pandas -t python/
zip -r layer.zip python/

# Publish layer
aws lambda publish-layer-version \
  --layer-name pandas-layer \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11 python3.12

# Attach to function
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:123456789:layer:pandas-layer:1
```

---

## 7. Lambda Destinations and Error Handling

### Asynchronous Invocation Error Handling

```
Invocation ──► Lambda ──► Success ──► Destination (OnSuccess)
                     │
                     └──► Failure ──► Retry (2 attempts) ──► DLQ or Destination (OnFailure)
```

```bash
# Configure destinations
aws lambda put-function-event-invoke-config \
  --function-name my-function \
  --destination-config '{
    "OnSuccess": {"Destination": "arn:aws:sqs:us-east-1:123:success-queue"},
    "OnFailure": {"Destination": "arn:aws:sqs:us-east-1:123:failure-queue"}
  }'
```

### Error Handling Patterns

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    try:
        result = process(event)
        return {'statusCode': 200, 'body': json.dumps(result)}
    
    except ValueError as e:
        # Client error — don't retry
        logger.error(f"Validation error: {e}")
        return {'statusCode': 400, 'body': str(e)}
    
    except Exception as e:
        # Server error — Lambda will retry for async
        logger.exception(f"Unexpected error: {e}")
        raise  # Re-raise to trigger retry/DLQ
```

---

## 8. Lambda Concurrency

```
Total Account Limit: 1,000 concurrent executions per region
                         │
        ┌────────────────┼────────────────┐
        │                │                │
Function A          Function B        Function C
(Reserved: 200)   (Reserved: 100)   (Unreserved pool)
        │                │                │
Max 200 concurrent  Max 100 concurrent  Shares remaining
executions          executions          700 with others
```

### Concurrency Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Unreserved** | Shares account pool | Dev/test functions |
| **Reserved** | Guaranteed allocation | Critical production functions |
| **Provisioned** | Pre-warmed environments | Latency-sensitive APIs |

```bash
# Set reserved concurrency
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100

# Set provisioned concurrency (eliminates cold starts)
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier PROD \
  --provisioned-concurrent-executions 10
```

---

## 9. Lambda VPC Integration

By default Lambda runs **outside** your VPC. To access RDS, ElastiCache, private resources:

```
Without VPC:                    With VPC:
─────────────────               ─────────────────────────────
Lambda ──► Internet ──► RDS?    Lambda ──► Private Subnet
(RDS must be public!)             └──► RDS (private, secure)

VPC Lambda Setup:
┌─────────────────────────────────────────────────────┐
│                    Your VPC                          │
│  ┌──────────────────┐    ┌──────────────────────┐  │
│  │  Private Subnet  │    │   Private Subnet     │  │
│  │  ┌────────────┐  │    │  ┌────────────────┐  │  │
│  │  │   Lambda   │──┼────┼─►│  RDS / Redis   │  │  │
│  │  │  (ENI)     │  │    │  └────────────────┘  │  │
│  │  └────────────┘  │    └──────────────────────┘  │
│  └──────────────────┘                               │
│                                                     │
│  NAT Gateway (needed for Lambda to reach internet)  │
└─────────────────────────────────────────────────────┘
```

**VPC Configuration:**
```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-xyz789
```

**Important:** VPC Lambda needs a NAT Gateway to reach the internet or AWS services (unless using VPC endpoints).

---

## 10. Lambda with API Gateway

The most common serverless pattern — REST APIs without servers:

```
Client ──► API Gateway ──► Lambda ──► DynamoDB/RDS
              │
         (Auth, Rate limiting,
          SSL termination,
          Request validation)
```

### API Gateway Proxy Integration Event

```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJ..."
  },
  "queryStringParameters": {
    "page": "1"
  },
  "body": "{\"name\": \"John\", \"email\": \"john@example.com\"}",
  "requestContext": {
    "authorizer": {
      "claims": {
        "sub": "user-id-123"
      }
    }
  }
}
```

### Lambda Response for API Gateway

```python
def lambda_handler(event, context):
    method = event['httpMethod']
    path = event['path']
    body = json.loads(event.get('body', '{}'))
    
    if method == 'POST' and path == '/users':
        user_id = create_user(body)
        return {
            'statusCode': 201,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'  # CORS
            },
            'body': json.dumps({'userId': user_id})
        }
    
    return {
        'statusCode': 404,
        'body': json.dumps({'error': 'Not found'})
    }
```

---

## 11. Lambda Pricing

```
Pricing Components:
1. Requests:    $0.20 per 1 million requests
                First 1M requests/month FREE

2. Duration:    $0.0000166667 per GB-second
                Calculated as: (memory GB) × (duration seconds)
                First 400,000 GB-seconds/month FREE

Example:
Function: 512MB memory, runs 200ms, called 10M times/month

Duration cost:
  0.5 GB × 0.2s × 10,000,000 = 1,000,000 GB-seconds
  1,000,000 - 400,000 (free) = 600,000 GB-seconds
  600,000 × $0.0000166667 = $10.00

Request cost:
  10,000,000 - 1,000,000 (free) = 9,000,000
  9 × $0.20 = $1.80

Total: $11.80/month
```

---

## 12. Lambda Best Practices

### Code Best Practices

```python
# ✅ GOOD: Initialize outside handler (reused across warm invocations)
import boto3
s3_client = boto3.client('s3')
db_connection = create_db_connection()

def lambda_handler(event, context):
    # Handler is called on every invocation
    data = s3_client.get_object(...)  # Reuses client

# ❌ BAD: Initialize inside handler (re-created every invocation)
def lambda_handler(event, context):
    s3_client = boto3.client('s3')    # New client every time!
    db_connection = create_db_connection()  # New connection every time!
```

```python
# ✅ GOOD: Return early on validation failure
def lambda_handler(event, context):
    if not event.get('user_id'):
        return {'statusCode': 400, 'body': 'user_id required'}
    
    # Continue processing...
```

### Security Best Practices

- **Least privilege**: Only grant permissions the function needs
- **No hardcoded credentials**: Use IAM roles, not access keys
- **Secrets Manager**: For DB passwords, API keys
- **Encrypt env vars**: Use KMS for sensitive environment variables
- **VPC**: Put Lambda in VPC when accessing private resources
- **Function URL auth**: Use `AWS_IAM` auth, not `NONE`

### Performance Best Practices

- **Right-size memory**: More memory = more CPU = faster execution
- **Reuse connections**: DB connections, HTTP clients outside handler
- **Minimize package size**: Smaller deployment = faster cold starts
- **Use Layers**: Separate dependencies from function code
- **Provisioned Concurrency**: For latency-critical APIs

---

## 13. Real-World Use Cases

### Use Case 1: Image Processing Pipeline

```
S3 Upload (photo.jpg)
      │
      ▼
Lambda (image-processor)
  • Resize to thumbnails (100px, 300px, 800px)
  • Extract EXIF metadata
  • Run Rekognition for content moderation
  • Store thumbnails back to S3
  • Save metadata to DynamoDB
      │
      ▼
SQS Queue ──► Lambda (notification-sender)
  • Send email notification via SES
  • Update user record
```

### Use Case 2: Real-Time Data Processing

```
IoT Devices ──► Kinesis Data Stream
                      │
                      ▼
            Lambda (stream-processor)
              • Parse sensor readings
              • Calculate aggregates
              • Detect anomalies
              • Store to DynamoDB
              • Alert via SNS if threshold exceeded
```

### Use Case 3: Serverless API Backend

```
Mobile App ──► API Gateway ──► Lambda (CRUD handlers)
                                    │
                          ┌─────────┼─────────┐
                          ▼         ▼         ▼
                       DynamoDB  S3 (files)  SES (email)
```

### Use Case 4: Automated Cost Optimization

```
EventBridge Rule (daily at midnight)
      │
      ▼
Lambda (cost-optimizer)
  • List all EC2 instances
  • Check if tagged with "Environment: dev"
  • Stop all dev instances
  • Send Slack notification
  • Log action to CloudWatch
```

---

## 14. Detailed Workflows

### Workflow: Deploy Lambda with CI/CD

```
Developer pushes code
        │
        ▼
GitHub Actions / CodePipeline
  1. Run tests (pytest/jest)
  2. Package function (zip)
  3. Deploy to staging Lambda alias
  4. Run integration tests
  5. If pass: Deploy to production alias (blue/green)
        │
        ▼
Lambda Aliases (Traffic Shifting)
  PROD alias: 90% → v3 (current), 10% → v4 (canary)
        │
        ▼
CloudWatch Alarms
  If error rate > 1% ──► CodeDeploy auto-rollback
```

### Workflow: Lambda Alias & Version Strategy

```bash
# Publish a new version
aws lambda publish-version --function-name my-function

# Create/update alias pointing to version
aws lambda update-alias \
  --function-name my-function \
  --name PROD \
  --function-version 5 \
  --routing-config AdditionalVersionWeights={"4"=0.1}
# 90% to v5, 10% to v4 (canary)

# After validation, shift 100% to v5
aws lambda update-alias \
  --function-name my-function \
  --name PROD \
  --function-version 5
```

### Workflow: SQS Dead Letter Queue Pattern

```
SQS Queue ──► Lambda ──► Success ──► Message deleted
   ▲               │
   │               └──► Failure
   │                      │
   │              Retry (up to maxReceiveCount=3)
   │                      │
   └──── DLQ ◄── Move to Dead Letter Queue
                          │
                          ▼
               Lambda (dlq-processor)
                 • Alert on-call engineer
                 • Log to S3 for analysis
                 • Attempt manual reprocessing
```

---

## 15. Common Interview Questions

### Conceptual Questions

**Q1: What is the difference between Lambda and EC2?**
> Lambda is serverless, event-driven, auto-scales, pay per invocation, max 15-min execution. EC2 is a virtual machine you manage, always running, manual/ASG scaling, pay per hour. Lambda is ideal for short-duration event-driven tasks; EC2 for long-running processes, complex networking, custom OS.

**Q2: What is a cold start and how do you reduce it?**
> Cold start is the latency added when Lambda creates a new execution environment: download code, start runtime, run init code. Reduce it by: Provisioned Concurrency (pre-warms envs), smaller packages, interpreted languages (Python/Node over Java), Lambda SnapStart (Java), minimizing imports.

**Q3: How does Lambda handle errors differently for sync vs async invocations?**
> Synchronous: error returned directly to caller, no retry. Asynchronous: Lambda retries twice with backoff, then sends to DLQ or OnFailure destination. Event source mapping (SQS): message stays in queue, retried until maxReceiveCount, then goes to DLQ.

**Q4: What is the maximum timeout for a Lambda function?**
> 900 seconds (15 minutes). For longer processes, use Step Functions, ECS Tasks, or Fargate.

**Q5: How does Lambda scale?**
> Lambda scales by running multiple concurrent instances. Each request gets its own execution environment. Default account limit is 1,000 concurrent executions per region (can request increase). Scaling is automatic and near-instant (burst limit: 500-3,000 new instances/minute depending on region).

**Q6: What is the difference between Reserved Concurrency and Provisioned Concurrency?**
> Reserved Concurrency: Guarantees X executions for this function (throttles others). Prevents one function from consuming all account concurrency. Provisioned Concurrency: Pre-warms N execution environments to eliminate cold starts. Costs more but essential for latency-sensitive workloads.

**Q7: Can Lambda access resources in a private VPC? What's needed?**
> Yes. Configure Lambda with VPC subnet IDs and security group IDs. Lambda creates ENIs in your subnet. For internet access from VPC Lambda, you need a NAT Gateway in a public subnet with route table configured. For AWS services, use VPC endpoints.

**Q8: What are Lambda Layers and why use them?**
> Layers are ZIP archives containing libraries/dependencies attached to multiple functions. Benefits: Smaller deployment packages, shared dependencies across functions, separation of concerns, can be versioned independently. Maximum 5 layers per function, 250MB total unzipped.

**Q9: What is Lambda function URL?**
> A dedicated HTTPS endpoint for a Lambda function without API Gateway. Supports IAM auth or no auth. Cheaper than API Gateway but fewer features (no WAF, no custom domain without CloudFront, no throttling rules).

**Q10: How do you pass sensitive data to Lambda?**
> Options: (1) AWS Secrets Manager — retrieve at runtime, cache with rotation support. (2) SSM Parameter Store — fetch SecureString params at runtime. (3) Encrypted environment variables with KMS. Never hardcode credentials. Always use IAM roles for AWS service access.

### Scenario-Based Questions

**Q11: Your Lambda function times out connecting to RDS. What do you check?**
> 1. Is Lambda in the same VPC as RDS? 2. Do security groups allow Lambda's SG to reach RDS on port 3306/5432? 3. Is the Lambda timeout long enough? 4. Connection pool exhaustion? (Lambda creates a new connection per cold start — use RDS Proxy) 5. Lambda in private subnet with NAT if needed?

**Q12: How would you implement idempotency in Lambda?**
> Lambda may be invoked multiple times (retries). Use: (1) Check/set idempotency key in DynamoDB before processing. (2) AWS Powertools for Lambda idempotency decorator. (3) Make operations naturally idempotent (S3 PutObject is idempotent). (4) Use SQS FIFO with message deduplication IDs.

**Q13: Lambda function is running slow. How do you diagnose?**
> 1. CloudWatch Logs — check duration and init duration (cold starts). 2. X-Ray tracing — identify slow subsegments (DB queries, external APIs). 3. Memory — increase memory (also increases CPU). 4. Check if connections are being reused (outside handler). 5. Lambda Power Tuning tool to find optimal memory.

**Q14: How would you implement a serverless REST API?**
> API Gateway (HTTP API or REST API) → Lambda (handlers per route or single proxy) → DynamoDB/RDS Proxy. Use Lambda Proxy Integration. Implement JWT auth via Cognito or custom Lambda Authorizer. Deploy with SAM or Serverless Framework. Use aliases for blue/green deployments.

**Q15: What happens when Lambda exceeds concurrency limits?**
> Lambda returns 429 TooManyRequestsException (throttling). For synchronous invocations: error returned to caller. For async: Lambda queues for up to 6 hours, retries with backoff. For event source mappings: messages stay in queue. Solutions: increase reserved concurrency, request account limit increase, implement exponential backoff in callers.

---

*Next: [Lambda Assignments](02-lambda-assignments.md)*
