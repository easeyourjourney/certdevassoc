# Quick Reference Guide

Fast lookup for service limits, error codes, and important commands.

---

## Service Limits & Quotas

### AWS Lambda
| Limit | Value |
|-------|-------|
| Maximum execution time | 15 minutes (900 seconds) |
| Memory allocation | 128 MB - 10,240 MB |
| Deployment package size (zipped) | 50 MB (direct upload), 250 MB (from S3) |
| Deployment package size (unzipped) | 250 MB |
| /tmp directory storage | 512 MB - 10,240 MB |
| Environment variables | 4 KB total |
| Concurrent executions per region | 1,000 (soft limit, can increase) |
| Function timeout default | 3 seconds |
| Layers per function | 5 |
| Invocation payload (request/response) | 6 MB (synchronous), 256 KB (asynchronous) |

### Amazon DynamoDB
| Limit | Value |
|-------|-------|
| Maximum item size | 400 KB |
| Maximum partition key length | 2,048 bytes |
| Maximum sort key length | 1,024 bytes |
| Global Secondary Indexes per table | 20 |
| Local Secondary Indexes per table | 5 (created at table creation only) |
| Projected secondary index attributes | 100 |
| BatchGetItem | 100 items, 16 MB |
| BatchWriteItem | 25 items, 16 MB |
| TransactGetItems / TransactWriteItems | 100 items or 4 MB |
| Query result size | 1 MB per request |
| Scan result size | 1 MB per request |
| Table name length | 3-255 characters |
| DynamoDB Streams retention | 24 hours |

### Amazon S3
| Limit | Value |
|-------|-------|
| Maximum object size | 5 TB |
| Maximum upload size (single PUT) | 5 GB |
| Multipart upload | Required for objects > 5 GB |
| Maximum parts in multipart upload | 10,000 |
| Bucket name length | 3-63 characters |
| Bucket policy size | 20 KB |
| Object tags | 10 per object |
| Lifecycle rules per bucket | 1,000 |
| Buckets per account | 100 (soft limit, can increase to 1,000) |
| Presigned URL expiration | Up to 7 days (with IAM credentials) |

### Amazon API Gateway
| Limit | Value |
|-------|-------|
| Integration timeout | 29 seconds |
| Maximum payload size | 10 MB |
| Default throttle limit | 10,000 requests/second |
| Default burst limit | 5,000 requests |
| API keys per account per region | 500 |
| Usage plans per account per region | 300 |
| Stages per API | 10 |
| Resources per API | 300 |
| Cache size | 0.5 GB - 237 GB |
| Cache TTL | 0 - 3600 seconds (default 300) |

### Amazon SQS
| Limit | Value |
|-------|-------|
| Message size | 256 KB |
| Message retention | 1 minute - 14 days (default 4 days) |
| Visibility timeout | 0 seconds - 12 hours (default 30 seconds) |
| Long polling wait time | 1 - 20 seconds |
| Batch size | 10 messages |
| In-flight messages (standard) | 120,000 |
| In-flight messages (FIFO) | 20,000 |
| FIFO throughput | 300 TPS (3,000 with batching) |
| Message delay | 0 seconds - 15 minutes |
| Queue name length | Up to 80 characters |

### Amazon SNS
| Limit | Value |
|-------|-------|
| Message size | 256 KB |
| Topics per account | 100,000 |
| Subscriptions per topic | 12,500,000 |
| SMS message size | 1,600 characters (GSM), 70 (UCS-2) |
| Message attributes | 10 per message |
| FIFO throughput | 300 TPS (3,000 with batching) |
| Filter policies | 5 per subscription |

### Amazon RDS
| Limit | Value |
|-------|-------|
| Read replicas per primary | 15 (Aurora), 5 (other engines) |
| DB instance max storage | 64 TB (Aurora), varies by engine |
| Backup retention | 0 - 35 days |
| Maximum IOPS | 80,000 (io2), 64,000 (io1) |
| Automated backup window | 30 minutes minimum |
| Manual snapshots | No limit |

### Amazon ElastiCache
| Limit | Value |
|-------|-------|
| Nodes per cluster | 90 (Memcached), 500 (Redis) |
| Parameter groups | 150 |
| Subnet groups | 150 |
| Redis snapshot retention | 0 - 35 days |
| Node type | cache.t2.micro - cache.r6g.16xlarge |

### Amazon ECS/Fargate
| Limit | Value |
|-------|-------|
| Tasks per service | No limit |
| Containers per task | 10 |
| Fargate task CPU | 0.25 vCPU - 16 vCPU |
| Fargate task memory | 0.5 GB - 120 GB |
| Task definition size | 64 KB (total) |
| Task definition revisions | No limit (keep active < 1 million) |

### CloudWatch
| Limit | Value |
|-------|-------|
| Metric data retention | 15 months |
| Custom metrics per region | 10,000 (can increase) |
| Alarms per region | 5,000 |
| PutMetricData | 40 KB per POST |
| Log event size | 256 KB |
| Log group retention | 1 day - 10 years (or infinite) |
| GetLogEvents | 10 MB or 10,000 events |

---

## Common HTTP Status Codes

### Client Errors (4xx)

| Code | Name | Meaning | Common Causes |
|------|------|---------|---------------|
| 400 | Bad Request | Malformed request | Invalid JSON, missing required parameters |
| 401 | Unauthorized | Authentication required | Missing or invalid credentials/token |
| 403 | Forbidden | No permission | IAM policy denies access, resource policy |
| 404 | Not Found | Resource doesn't exist | Wrong URL, resource deleted |
| 408 | Request Timeout | Request took too long | Client didn't send data in time |
| 409 | Conflict | Resource conflict | Concurrent modification, duplicate resource |
| 413 | Payload Too Large | Request body too large | Exceeds service limit |
| 429 | Too Many Requests | Rate limit exceeded | Throttling, exceeded quota |

### Server Errors (5xx)

| Code | Name | Meaning | Common Causes |
|------|------|---------|---------------|
| 500 | Internal Server Error | Unhandled exception | Application error, uncaught exception |
| 502 | Bad Gateway | Invalid response from backend | Lambda error, backend timeout |
| 503 | Service Unavailable | Service temporarily unavailable | AWS service issue, overload |
| 504 | Gateway Timeout | Backend didn't respond in time | Lambda timeout, integration timeout |

---

## AWS-Specific Error Codes

### DynamoDB

| Error | Meaning | Solution |
|-------|---------|----------|
| ProvisionedThroughputExceededException | Exceeded RCU/WCU | Increase capacity, enable auto-scaling, or switch to on-demand |
| ConditionalCheckFailedException | Condition in write operation not met | Expected condition failed (version mismatch, etc.) |
| ValidationException | Invalid request | Check parameter types, required fields |
| ResourceNotFoundException | Table/item doesn't exist | Verify table name, check region |
| ItemCollectionSizeLimitExceededException | LSI collection > 10 GB | Redesign data model, use GSI instead |

### Lambda

| Error | Meaning | Solution |
|-------|---------|----------|
| TooManyRequestsException (429) | Throttled | Increase concurrency limit, use reserved concurrency |
| ResourceNotFoundException | Function doesn't exist | Check function name, region, account |
| InvalidParameterValueException | Invalid parameter | Check function configuration (memory, timeout, etc.) |
| RequestEntityTooLargeException | Payload too large | Reduce payload size (6 MB sync, 256 KB async) |

### S3

| Error | Meaning | Solution |
|-------|---------|----------|
| NoSuchBucket | Bucket doesn't exist | Create bucket, check bucket name and region |
| NoSuchKey | Object doesn't exist | Verify object key (path), check if deleted |
| AccessDenied | No permission | Check bucket policy, IAM policy, ACL |
| EntityTooLarge | Object > 5 GB in single PUT | Use multipart upload |
| SlowDown | Too many requests | Reduce request rate, implement backoff |

### SQS

| Error | Meaning | Solution |
|-------|---------|----------|
| OverLimit | Too many in-flight messages | Process messages faster, delete after processing |
| QueueDoesNotExist | Queue not found | Create queue, check queue URL/name |
| MessageNotInflight | Can't delete non-visible message | Message already deleted or not received |

---

## SDK Error Handling

### Python (Boto3)

```python
import boto3
from botocore.exceptions import ClientError
import time
import random

s3 = boto3.client('s3')

# Error handling
try:
    response = s3.get_object(Bucket='mybucket', Key='mykey')
except ClientError as e:
    error_code = e.response['Error']['Code']
    if error_code == 'NoSuchKey':
        print("Object not found")
    elif error_code == '404':
        print("Not found")
    else:
        print(f"Error: {error_code}")
        raise

# Exponential backoff with jitter
max_retries = 5
for attempt in range(max_retries):
    try:
        response = dynamodb.put_item(TableName='MyTable', Item={'id': {'S': '123'}})
        break
    except ClientError as e:
        if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
            if attempt == max_retries - 1:
                raise
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            print(f"Throttled, waiting {wait_time:.2f}s")
            time.sleep(wait_time)
        else:
            raise
```

### JavaScript (AWS SDK v3)

```javascript
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

const client = new S3Client({ region: "us-east-1" });

// Error handling
try {
  const response = await client.send(
    new GetObjectCommand({ Bucket: "mybucket", Key: "mykey" })
  );
} catch (error) {
  if (error.name === "NoSuchKey") {
    console.log("Object not found");
  } else if (error.$metadata?.httpStatusCode === 404) {
    console.log("Not found");
  } else {
    console.error("Error:", error);
    throw error;
  }
}

// Exponential backoff (SDK has built-in retry)
const clientWithRetry = new S3Client({
  region: "us-east-1",
  maxAttempts: 5,
  retryMode: "adaptive" // or "standard"
});
```

---

## IAM Policy Examples

### Lambda Execution Role (DynamoDB Access)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### S3 Bucket Policy (Public Read)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mybucket/*"
    }
  ]
}
```

### S3 Bucket Policy (User-Specific Folders with Cognito)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::user-uploads/${cognito-identity.amazonaws.com:sub}/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::user-uploads",
      "Condition": {
        "StringLike": {
          "s3:prefix": "${cognito-identity.amazonaws.com:sub}/*"
        }
      }
    }
  ]
}
```

### Assume Role Policy (EC2)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Assume Role Policy (Lambda)
```json
{
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
}
```

---

## CloudFormation Template Examples

### Simple Lambda Function
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple Lambda function

Resources:
  MyLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: MyFunction
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return {'statusCode': 200, 'body': 'Hello World'}
      Timeout: 60
      MemorySize: 128

  LambdaExecutionRole:
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

Outputs:
  FunctionArn:
    Value: !GetAtt MyLambdaFunction.Arn
    Description: Lambda function ARN
```

### DynamoDB Table with GSI
```yaml
Resources:
  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: Users
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
        - AttributeName: email
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
      GlobalSecondaryIndexes:
        - IndexName: EmailIndex
          KeySchema:
            - AttributeName: email
              KeyType: HASH
          Projection:
            ProjectionType: ALL
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
```

---

## SAM Template Example

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless API

Globals:
  Function:
    Timeout: 30
    Runtime: python3.9
    Environment:
      Variables:
        TABLE_NAME: !Ref NotesTable

Resources:
  NotesApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt UserPool.Arn

  CreateNoteFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: create_note.lambda_handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref NotesTable
      Events:
        CreateNote:
          Type: Api
          Properties:
            RestApiId: !Ref NotesApi
            Path: /notes
            Method: post

  NotesTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: noteId
        Type: String

  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: NotesUserPool
      AutoVerifiedAttributes:
        - email
```

---

## Important CLI Commands Cheat Sheet

### Configure AWS CLI
```bash
aws configure
aws configure --profile myprofile
aws configure set region us-east-1
```

### S3
```bash
# List buckets
aws s3 ls

# List objects
aws s3 ls s3://mybucket/prefix/

# Upload file
aws s3 cp file.txt s3://mybucket/
aws s3 cp file.txt s3://mybucket/newname.txt

# Download file
aws s3 cp s3://mybucket/file.txt ./

# Sync directory
aws s3 sync ./local s3://mybucket/remote --delete

# Generate presigned URL
aws s3 presign s3://mybucket/file.txt --expires-in 3600

# Remove file
aws s3 rm s3://mybucket/file.txt

# Remove all files
aws s3 rm s3://mybucket/ --recursive
```

### Lambda
```bash
# List functions
aws lambda list-functions

# Invoke function
aws lambda invoke --function-name MyFunction output.json
aws lambda invoke --function-name MyFunction --payload '{"key":"value"}' output.json

# Update function code
aws lambda update-function-code --function-name MyFunction --zip-file fileb://function.zip

# Update function configuration
aws lambda update-function-configuration --function-name MyFunction --timeout 60

# Get function
aws lambda get-function --function-name MyFunction

# Create alias
aws lambda create-alias --function-name MyFunction --name PROD --function-version 1

# Update alias
aws lambda update-alias --function-name MyFunction --name PROD --function-version 2
```

### DynamoDB
```bash
# List tables
aws dynamodb list-tables

# Describe table
aws dynamodb describe-table --table-name MyTable

# Put item
aws dynamodb put-item --table-name MyTable --item '{"id": {"S": "123"}, "name": {"S": "John"}}'

# Get item
aws dynamodb get-item --table-name MyTable --key '{"id": {"S": "123"}}'

# Query
aws dynamodb query --table-name MyTable --key-condition-expression "id = :id" --expression-attribute-values '{":id": {"S": "123"}}'

# Scan
aws dynamodb scan --table-name MyTable

# Delete item
aws dynamodb delete-item --table-name MyTable --key '{"id": {"S": "123"}}'
```

### CloudWatch Logs
```bash
# List log groups
aws logs describe-log-groups

# Tail logs (follow)
aws logs tail /aws/lambda/MyFunction --follow

# Filter logs
aws logs filter-log-events --log-group-name /aws/lambda/MyFunction --filter-pattern "ERROR"

# Get log events
aws logs get-log-events --log-group-name /aws/lambda/MyFunction --log-stream-name 2023/11/15/[LATEST]abc123
```

### IAM
```bash
# List users
aws iam list-users

# Create user
aws iam create-user --user-name john

# List roles
aws iam list-roles

# Get role
aws iam get-role --role-name MyRole

# Attach policy to role
aws iam attach-role-policy --role-name MyRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### CodeDeploy
```bash
# List applications
aws deploy list-applications

# Create deployment
aws deploy create-deployment --application-name MyApp --deployment-group-name MyGroup --s3-location bucket=mybucket,key=app.zip,bundleType=zip

# Get deployment
aws deploy get-deployment --deployment-id d-ABC123
```

---

## Performance Optimization Checklist

### Lambda
- [ ] Right-size memory allocation
- [ ] Minimize deployment package size
- [ ] Use layers for common dependencies
- [ ] Reuse connections (database, HTTP)
- [ ] Use provisioned concurrency for latency-sensitive functions
- [ ] Enable X-Ray for performance analysis
- [ ] Keep functions warm with CloudWatch Events (if needed)

### DynamoDB
- [ ] Use on-demand for unpredictable workloads
- [ ] Use auto-scaling for provisioned mode
- [ ] Design partition key for even distribution
- [ ] Use GSI for alternate query patterns
- [ ] Use batch operations when possible
- [ ] Use DAX for read-heavy workloads
- [ ] Enable TTL for automatic deletion

### S3
- [ ] Use CloudFront for global distribution
- [ ] Enable Transfer Acceleration for large uploads
- [ ] Use multipart upload for large files
- [ ] Choose appropriate storage class
- [ ] Use lifecycle policies for cost optimization
- [ ] Enable versioning only when needed

### API Gateway
- [ ] Enable caching for frequently accessed data
- [ ] Use regional endpoints when clients in one region
- [ ] Implement throttling to protect backend
- [ ] Use HTTP API instead of REST API when possible (cheaper, faster)
- [ ] Enable compression

---

This quick reference should be your go-to resource during study and before the exam!
