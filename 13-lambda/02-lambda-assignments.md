# AWS Lambda — Assignments

## Assignment 1: Your First Lambda Function

**Goal:** Create and invoke a Lambda function via console and CLI

**Tasks:**
1. Create a Lambda function in Python 3.11 named `hello-lambda`
2. Write handler that returns: `{"name": your_name, "timestamp": <current_time>}`
3. Test it from the AWS Console
4. Invoke it with AWS CLI: `aws lambda invoke --function-name hello-lambda output.json`
5. View execution logs in CloudWatch Logs

**Expected output:**
```json
{
  "statusCode": 200,
  "body": "{\"name\": \"DevOps\", \"timestamp\": \"2024-01-15T10:00:00\"}"
}
```

---

## Assignment 2: S3 Trigger — Image Thumbnail Generator

**Goal:** Process S3 uploads with Lambda

**Architecture:**
```
User uploads image → S3 bucket (source) → Lambda trigger → Resize → S3 bucket (thumbnails)
```

**Tasks:**
1. Create two S3 buckets: `source-images-<your-name>` and `thumbnail-images-<your-name>`
2. Create Lambda function with Python that:
   - Reads image from source bucket
   - Resizes to 150x150 pixels using Pillow
   - Saves to thumbnails bucket with prefix `thumb_`
3. Create Lambda Layer with Pillow dependency
4. Configure S3 trigger on source bucket for `ObjectCreated` events
5. Configure IAM role with permissions for both buckets
6. Test by uploading an image and verifying thumbnail creation

**Pillow as a Layer:**
```bash
mkdir python && pip install Pillow -t python/
zip -r pillow-layer.zip python/
aws lambda publish-layer-version --layer-name pillow-layer --zip-file fileb://pillow-layer.zip --compatible-runtimes python3.11
```

---

## Assignment 3: Serverless API with API Gateway

**Goal:** Build a REST API for a TODO list

**API Endpoints:**
```
GET    /todos         → List all TODOs
GET    /todos/{id}    → Get specific TODO
POST   /todos         → Create TODO
PUT    /todos/{id}    → Update TODO
DELETE /todos/{id}    → Delete TODO
```

**Tasks:**
1. Create DynamoDB table `todos` with `id` as partition key
2. Create Lambda function that handles all methods (proxy integration)
3. Create HTTP API in API Gateway with Lambda proxy integration
4. Test each endpoint with curl:
```bash
# Create
curl -X POST https://api-id.execute-api.us-east-1.amazonaws.com/todos \
     -H "Content-Type: application/json" \
     -d '{"title": "Learn Lambda", "completed": false}'

# List
curl https://api-id.execute-api.us-east-1.amazonaws.com/todos

# Update
curl -X PUT .../todos/123 -d '{"completed": true}'
```

---

## Assignment 4: Scheduled Lambda — Daily Report

**Goal:** Send automated daily reports

**Tasks:**
1. Create Lambda that:
   - Lists all EC2 instances in your account
   - Counts running vs stopped
   - Calculates estimated monthly cost
   - Formats into a report
2. Configure EventBridge rule to trigger daily at 8 AM UTC
3. Test by manually invoking
4. Add SES email sending (set up SES sandbox first)
5. Monitor execution in CloudWatch

**Bonus:** Add to SNS topic instead of SES for multiple subscribers

---

## Assignment 5: Error Handling and Dead Letter Queue

**Goal:** Implement robust error handling

**Tasks:**
1. Create SQS queue `order-processor` and DLQ `order-processor-dlq`
2. Set `maxReceiveCount = 3` on main queue, DLQ target = dlq
3. Create Lambda that:
   - Processes order messages
   - Succeeds for 70% of messages
   - Fails for messages containing `"fail": true`
4. Send 10 test messages (2 with fail=true)
5. Verify failed messages appear in DLQ after 3 retries
6. Create second Lambda triggered by DLQ to log and alert
7. Configure Lambda destinations for async invocations

---

## Assignment 6: VPC Lambda + RDS

**Goal:** Connect Lambda to private RDS instance

**Tasks:**
1. Create VPC with private subnets in 2 AZs
2. Create RDS MySQL in private subnets
3. Create Lambda in private subnets
4. Configure security groups:
   - Lambda SG → RDS SG on port 3306
5. Lambda function that:
   - Connects to RDS
   - Creates a table if not exists
   - Inserts a record
   - Returns all records
6. Add RDS Proxy to manage connection pooling (bonus)

**Note:** Lambda needs NAT Gateway to download packages or call AWS APIs outside VPC

---

## Interview Assignment 1: Lambda Performance Optimization

**Scenario:** Your Lambda function processes 1M requests/day. Average duration: 3 seconds, memory: 512MB. Users complain about slow response times.

**Tasks:**
1. Calculate current monthly cost
2. Identify at least 3 causes of slow response
3. Implement and measure improvements:
   - Move DB connection outside handler
   - Increase memory to 1024MB (time the difference)
   - Add X-Ray tracing to identify bottleneck
   - Add Provisioned Concurrency for peak hours
4. Calculate new monthly cost after optimization
5. Document findings in a report

---

## Interview Assignment 2: Design Serverless Architecture

**Scenario:** Design a serverless notification system for an e-commerce platform.

**Requirements:**
- Order placed → SMS + email notification
- Daily digest email to users
- Retry failed notifications up to 3 times
- Store notification history
- Handle 10,000 orders/hour

**Deliverables:**
1. Architecture diagram with all Lambda functions, triggers, and data stores
2. IAM permissions for each Lambda function
3. Error handling strategy (what happens when SES is down?)
4. Cost estimate for 10,000 orders/hour
5. Monitoring and alerting plan

---

## Workflow Practice: Lambda CI/CD Pipeline

**Build this complete workflow:**

```
1. Developer pushes code to GitHub
2. GitHub Actions workflow triggers
3. Run unit tests (pytest)
4. Package Lambda function
5. Deploy to dev Lambda alias
6. Run integration tests
7. If tests pass: deploy to staging alias
8. Manual approval for production
9. Deploy to production alias with 10% canary
10. Monitor CloudWatch error rate
11. If errors > 1%: auto-rollback
```

**Implement using:**
- GitHub Actions for CI/CD
- AWS SAM for Lambda packaging
- Lambda aliases for blue/green deployment
- CloudWatch alarms for rollback trigger

---

## Cheat Sheet

```bash
# List functions
aws lambda list-functions --query 'Functions[*].[FunctionName,Runtime,MemorySize]' --output table

# Invoke function
aws lambda invoke \
    --function-name my-function \
    --cli-binary-format raw-in-base64-out \
    --payload '{"key": "value"}' \
    output.json

# Update function code
aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://function.zip

# Get function logs (last 5 minutes)
aws logs filter-log-events \
    --log-group-name /aws/lambda/my-function \
    --start-time $(date -d '5 minutes ago' +%s000)

# Test Lambda locally with SAM
sam local invoke MyFunction --event event.json
sam local start-api --port 3000
```
