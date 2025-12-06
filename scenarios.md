# AWS Developer Associate - Real-World Scenarios

Learn how AWS services integrate through practical scenarios. Each scenario shows multiple services working together.

---

## Scenario 1: Serverless REST API

**Use Case**: Build a serverless REST API for a todo application

**Architecture**:
```
Client → API Gateway → Lambda → DynamoDB
                ↓
           CloudWatch Logs
                ↓
              X-Ray
```

**Services Involved**:
- API Gateway (REST API endpoints)
- Lambda (business logic)
- DynamoDB (data storage)
- IAM (permissions)
- CloudWatch (logging and monitoring)
- X-Ray (distributed tracing)

### Implementation Flow

**1. DynamoDB Table**:
```
Table: Todos
Primary Key: userId (partition key) + todoId (sort key)
Attributes: title, description, completed, createdAt
```

**2. Lambda Functions**:
- `createTodo`: PUT /todos
- `getTodos`: GET /todos (list all for user)
- `getTodo`: GET /todos/{id}
- `updateTodo`: PATCH /todos/{id}
- `deleteTodo`: DELETE /todos/{id}

**3. API Gateway Configuration**:
- REST API with resource `/todos`
- Methods: GET, POST, PUT, DELETE
- Lambda proxy integration
- Enable CORS for web apps
- Enable X-Ray tracing

**4. IAM Roles**:
- **Lambda Execution Role**:
  ```json
  {
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:UpdateItem",
      "dynamodb:DeleteItem",
      "dynamodb:Query"
    ],
    "Resource": "arn:aws:dynamodb:region:account:table/Todos"
  }
  ```
  Plus CloudWatch Logs and X-Ray permissions

**5. Authentication**:
- Option A: Cognito User Pool authorizer
  - Users sign up/sign in
  - Get JWT token
  - API Gateway validates token
- Option B: IAM authorization
  - For service-to-service calls
  - Requires Sig v4 signing

### Key Integration Points

**Lambda to DynamoDB**:
```python
import boto3
import json
from uuid import uuid4

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Todos')

def lambda_handler(event, context):
    body = json.loads(event['body'])
    user_id = event['requestContext']['authorizer']['claims']['sub']  # From Cognito
    todo_id = str(uuid4())

    item = {
        'userId': user_id,
        'todoId': todo_id,
        'title': body['title'],
        'completed': False,
        'createdAt': int(time.time())
    }

    table.put_item(Item=item)

    return {
        'statusCode': 201,
        'headers': {
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(item)
    }
```

**API Gateway Response**:
- Lambda proxy integration returns object with:
  - statusCode
  - headers (CORS, content-type)
  - body (JSON string)

### Error Handling

**Lambda Errors**:
- Catch exceptions and return appropriate status codes
- 400 for bad request
- 404 for not found
- 500 for server errors
```python
try:
    response = table.get_item(Key={'userId': user_id, 'todoId': todo_id})
    if 'Item' not in response:
        return {'statusCode': 404, 'body': json.dumps({'error': 'Not found'})}
    return {'statusCode': 200, 'body': json.dumps(response['Item'])}
except Exception as e:
    print(f"Error: {e}")
    return {'statusCode': 500, 'body': json.dumps({'error': 'Internal server error'})}
```

**API Gateway Throttling**:
- Default 10,000 RPS
- Configure usage plans for API keys
- Returns 429 when exceeded

**DynamoDB Throttling**:
- Provisioned: Exceeding RCU/WCU returns 400 ProvisionedThroughputExceededException
- SDK retries with exponential backoff
- Consider on-demand mode for unpredictable traffic

### Monitoring

**CloudWatch Metrics**:
- Lambda: Invocations, Duration, Errors, Throttles
- API Gateway: Count, 4XXError, 5XXError, Latency
- DynamoDB: UserErrors, SystemErrors, ConsumedReadCapacityUnits

**CloudWatch Logs**:
- Lambda automatically logs to CloudWatch
- Log group: `/aws/lambda/functionName`
- Use print() or logging module

**X-Ray Tracing**:
- Enable on API Gateway and Lambda
- See complete request flow
- Identify bottlenecks (slow DynamoDB queries)

### Cost Optimization

- Use DynamoDB on-demand for low traffic
- Use API Gateway caching for frequently requested data
- Monitor Lambda memory usage (right-sizing)
- Consider DynamoDB DAX for read-heavy workloads

---

## Scenario 2: Event-Driven File Processing

**Use Case**: Process uploaded files automatically (resize images, extract metadata, generate thumbnails)

**Architecture**:
```
User uploads to S3
    ↓
S3 Event Notification
    ↓
    SNS Topic
    ↓ (fan-out)
├── SQS Queue 1 → Lambda (resize image)
├── SQS Queue 2 → Lambda (extract metadata)
└── SQS Queue 3 → Lambda (generate thumbnail)
    ↓
Store results in S3 and DynamoDB
```

**Services Involved**:
- S3 (file storage, events)
- SNS (pub/sub messaging)
- SQS (message queuing)
- Lambda (processing)
- DynamoDB (metadata storage)

### Implementation Flow

**1. S3 Buckets**:
- `uploads-bucket`: User uploads
- `processed-bucket`: Processed files

**2. S3 Event Notification**:
- Trigger on `s3:ObjectCreated:*` in uploads-bucket
- Send to SNS topic

**3. SNS Topic**:
- Topic: `file-processing-topic`
- Subscriptions: 3 SQS queues

**4. SQS Queues** (Standard or FIFO):
- `resize-queue`
- `metadata-queue`
- `thumbnail-queue`

**5. Lambda Functions**:
- Poll SQS queues (event source mapping)
- Process messages
- Store results

### Key Integration Points

**S3 to SNS**:
- Configure S3 bucket notification
```json
{
  "LambdaFunctionConfigurations": [],
  "TopicConfigurations": [{
    "TopicArn": "arn:aws:sns:region:account:file-processing-topic",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {
      "Key": {
        "FilterRules": [{
          "Name": "suffix",
          "Value": ".jpg"
        }]
      }
    }
  }]
}
```

**SNS to SQS (Fan-Out)**:
- Subscribe 3 SQS queues to SNS topic
- Same event processed by multiple consumers
- SQS policy allows SNS to send messages

**SQS to Lambda**:
- Event source mapping
- Lambda polls queue
- Batch size: 1-10 messages
- Visibility timeout > Lambda timeout
```python
def lambda_handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        message = json.loads(body['Message'])  # SNS wraps S3 event

        bucket = message['Records'][0]['s3']['bucket']['name']
        key = message['Records'][0]['s3']['object']['key']

        # Download from S3
        s3.download_file(bucket, key, '/tmp/input.jpg')

        # Process file
        resize_image('/tmp/input.jpg', '/tmp/output.jpg')

        # Upload to S3
        s3.upload_file('/tmp/output.jpg', 'processed-bucket', f'resized/{key}')

        # Delete message happens automatically if Lambda succeeds
```

### Error Handling

**SQS Dead Letter Queue**:
- Configure DLQ for each queue
- After 3-5 failed processing attempts, move to DLQ
- Analyze failed messages

**Lambda Retries**:
- For SQS event source, Lambda retries failed batches
- Configure maximum retry attempts
- Use partial batch responses (delete successful messages even if some fail)

**Idempotency**:
- Same message might be processed twice
- Check if result already exists before processing
- Store processing state in DynamoDB

### Monitoring

**SQS Metrics**:
- ApproximateNumberOfMessagesVisible (backlog)
- ApproximateAgeOfOldestMessage (processing lag)
- NumberOfMessagesReceived, NumberOfMessagesDeleted

**CloudWatch Alarms**:
- Alert if queue depth > threshold (backlog building up)
- Alert if message age > threshold (slow processing)

### Benefits of This Architecture

- **Decoupling**: S3 doesn't care about processing, processing doesn't care about upload
- **Scalability**: Each queue can scale independently
- **Reliability**: Failed messages go to DLQ
- **Parallel Processing**: Multiple Lambdas process simultaneously

---

## Scenario 3: Asynchronous Workflow with Step Functions

**Use Case**: E-commerce order processing (validate order → charge payment → update inventory → send notification)

**Architecture**:
```
API Gateway → Lambda (create order)
    ↓
Step Functions State Machine
    ├── Validate Order (Lambda)
    ├── Charge Payment (Lambda)
    ├── Update Inventory (Lambda)
    ├── Send Notification (SNS)
    └── Handle Errors (Catch & Retry)
    ↓
DynamoDB (order status)
```

**Services Involved**:
- API Gateway
- Lambda
- Step Functions
- DynamoDB
- SNS
- SQS (for failed orders)

### State Machine Definition

```json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:validateOrder",
      "Next": "ChargePayment",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "ResultPath": "$.error",
        "Next": "OrderFailed"
      }],
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "MaxAttempts": 2,
        "IntervalSeconds": 2,
        "BackoffRate": 2.0
      }]
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:chargePayment",
      "Next": "UpdateInventory",
      "Catch": [{
        "ErrorEquals": ["PaymentError"],
        "ResultPath": "$.error",
        "Next": "RefundPayment"
      }]
    },
    "UpdateInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:updateInventory",
      "Next": "SendNotification"
    },
    "SendNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:region:account:order-notifications",
        "Message.$": "$.orderDetails"
      },
      "Next": "OrderSucceeded"
    },
    "OrderSucceeded": {
      "Type": "Succeed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order validation failed"
    },
    "RefundPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:refundPayment",
      "Next": "OrderFailed"
    }
  }
}
```

### Key Concepts

**Error Handling**:
- **Retry**: Automatic retry with exponential backoff
  - MaxAttempts, IntervalSeconds, BackoffRate
- **Catch**: Handle specific errors and transition to different state
  - Like try-catch in programming

**State Types**:
- **Task**: Execute work (Lambda, ECS, SNS, SQS, DynamoDB, etc.)
- **Choice**: Branch based on input
- **Parallel**: Execute multiple branches concurrently
- **Wait**: Delay execution
- **Succeed/Fail**: Terminal states

**Data Flow**:
- Input from previous state passed to next state
- **ResultPath**: Where to store task result in input
  - `$.result`: Add to input under "result" key
  - `null`: Discard result, pass original input
- **OutputPath**: Filter what to pass to next state

### Integration with Lambda

**Start Execution**:
```python
import boto3
stepfunctions = boto3.client('stepfunctions')

response = stepfunctions.start_execution(
    stateMachineArn='arn:aws:states:region:account:stateMachine:OrderProcessing',
    input=json.dumps({
        'orderId': '12345',
        'userId': 'user-123',
        'items': [{'sku': 'ABC', 'quantity': 2}],
        'total': 99.99
    })
)
execution_arn = response['executionArn']
```

**Check Status**:
```python
response = stepfunctions.describe_execution(
    executionArn=execution_arn
)
status = response['status']  # RUNNING, SUCCEEDED, FAILED, TIMED_OUT, ABORTED
```

### Use Cases for Step Functions

- Long-running workflows (up to 1 year)
- Coordinating multiple microservices
- ETL processes
- Human approval workflows (with manual approval step)
- Retry and error handling logic

---

## Scenario 4: Containerized Web Application

**Use Case**: Deploy a containerized web application with database

**Architecture**:
```
Route 53 (DNS)
    ↓
Application Load Balancer
    ↓
ECS Fargate Service (multiple tasks)
    ├── Container 1 (web app)
    ├── Container 2 (web app)
    └── Container 3 (web app)
    ↓
RDS (PostgreSQL)
```

**Services Involved**:
- ECS Fargate
- ECR (container registry)
- ALB (Application Load Balancer)
- RDS
- Secrets Manager (database credentials)
- CloudWatch (logs)

### Implementation Flow

**1. Build and Push Docker Image**:
```bash
# Build
docker build -t myapp .

# Tag
docker tag myapp:latest 123456789012.dkr.ecr.region.amazonaws.com/myapp:latest

# Login to ECR
aws ecr get-login-password --region region | docker login --username AWS --password-stdin 123456789012.dkr.ecr.region.amazonaws.com

# Push
docker push 123456789012.dkr.ecr.region.amazonaws.com/myapp:latest
```

**2. Store DB Credentials in Secrets Manager**:
```json
{
  "username": "admin",
  "password": "supersecret",
  "host": "mydb.abc123.region.rds.amazonaws.com",
  "port": 5432,
  "dbname": "myapp"
}
```

**3. ECS Task Definition**:
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "executionRoleArn": "arn:aws:iam::account:role/ecsExecutionRole",
  "containerDefinitions": [{
    "name": "web",
    "image": "123456789012.dkr.ecr.region.amazonaws.com/myapp:latest",
    "portMappings": [{
      "containerPort": 80,
      "protocol": "tcp"
    }],
    "environment": [{
      "name": "ENV",
      "value": "production"
    }],
    "secrets": [{
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:region:account:secret:myapp/db:password::"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp",
        "awslogs-region": "region",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

**4. ECS Service**:
- Desired count: 3 tasks
- Load balancer: ALB target group
- Auto-scaling: Based on CPU or request count

**5. Application Load Balancer**:
- Target group: ECS tasks
- Health check: HTTP GET /health
- Listeners: HTTP:80, HTTPS:443

### IAM Roles

**Task Execution Role** (for ECS):
- Pull images from ECR
- Retrieve secrets from Secrets Manager
- Send logs to CloudWatch

**Task Role** (for application):
- Access S3 buckets
- Call other AWS services

### Database Connection

**From Container**:
```python
import os
import boto3
import json
import psycopg2

# Get secret from Secrets Manager
client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='myapp/db')
secret = json.loads(response['SecretString'])

# Connect to RDS
conn = psycopg2.connect(
    host=secret['host'],
    port=secret['port'],
    user=secret['username'],
    password=secret['password'],
    database=secret['dbname']
)
```

**Security**:
- RDS in private subnet (no internet access)
- ECS tasks in private subnet with NAT Gateway
- Security group rules: ECS → RDS on port 5432

### Deployment Strategies

**Rolling Update**:
- Default ECS deployment
- Gradually replace old tasks with new tasks
- Minimum healthy percent: 100% (no downtime)
- Maximum percent: 200% (can run double during deployment)

**Blue/Green with CodeDeploy**:
- Deploy to new task set
- Shift traffic gradually
- Easy rollback

### Monitoring

**CloudWatch Logs**:
- Container logs automatically sent
- Log group: /ecs/myapp

**CloudWatch Metrics**:
- ECS: CPUUtilization, MemoryUtilization
- ALB: RequestCount, TargetResponseTime, HTTPCode_Target_5XX_Count
- RDS: CPUUtilization, DatabaseConnections, ReadLatency

**Alarms**:
- Alert on high CPU (need to scale)
- Alert on 5XX errors (application issues)
- Alert on high database connections

---

## Scenario 5: User Authentication System

**Use Case**: Add user authentication to your application

**Architecture**:
```
User (Web/Mobile App)
    ↓
Cognito User Pool (sign up/sign in)
    ↓
JWT Tokens (ID token, access token)
    ↓
API Gateway (validates JWT)
    ↓
Lambda (authorized requests)
    ↓
DynamoDB (user data)
```

**Services Involved**:
- Cognito User Pools
- Cognito Identity Pools (for AWS credentials)
- API Gateway
- Lambda
- DynamoDB
- S3 (for user uploads)

### Implementation Flow

**1. Create User Pool**:
- Sign-up attributes: email, phone
- Password policy: min length, require uppercase, numbers
- MFA: optional or required
- Email/SMS verification

**2. Create App Client**:
- Enable auth flows: USER_PASSWORD_AUTH, REFRESH_TOKEN_AUTH
- Generate client ID (and optional client secret)

**3. User Sign-Up**:
```python
import boto3
client = boto3.client('cognito-idp')

response = client.sign_up(
    ClientId='app-client-id',
    Username='user@example.com',
    Password='TempPassword123!',
    UserAttributes=[
        {'Name': 'email', 'Value': 'user@example.com'},
        {'Name': 'name', 'Value': 'John Doe'}
    ]
)
# User receives verification code via email
```

**4. Confirm Sign-Up**:
```python
response = client.confirm_sign_up(
    ClientId='app-client-id',
    Username='user@example.com',
    ConfirmationCode='123456'
)
```

**5. Sign In**:
```python
response = client.initiate_auth(
    ClientId='app-client-id',
    AuthFlow='USER_PASSWORD_AUTH',
    AuthParameters={
        'USERNAME': 'user@example.com',
        'PASSWORD': 'TempPassword123!'
    }
)

id_token = response['AuthenticationResult']['IdToken']
access_token = response['AuthenticationResult']['AccessToken']
refresh_token = response['AuthenticationResult']['RefreshToken']
```

**6. Configure API Gateway Authorizer**:
- Type: Cognito User Pool
- User Pool: Select your user pool
- Token source: Authorization header

**7. Call API with Token**:
```javascript
fetch('https://api.example.com/todos', {
  headers: {
    'Authorization': id_token
  }
})
```

**8. Access User Info in Lambda**:
```python
def lambda_handler(event, context):
    # User info from Cognito
    claims = event['requestContext']['authorizer']['claims']
    user_id = claims['sub']  # Cognito user ID
    email = claims['email']
    name = claims['name']

    # Use user_id for database queries
    table.query(KeyConditionExpression=Key('userId').eq(user_id))
```

### Cognito Identity Pools (for AWS Credentials)

**Use Case**: Give users direct access to S3 (upload profile pictures)

**1. Create Identity Pool**:
- Authentication provider: Cognito User Pool
- Unauthenticated access: Disable (or enable for guest access)

**2. IAM Roles**:
- Authenticated role: Can access S3 with path-based permissions
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject"],
  "Resource": "arn:aws:s3:::user-uploads/${cognito-identity.amazonaws.com:sub}/*"
}
```
- Users can only access their own folder

**3. Get AWS Credentials** (from mobile/web app):
```python
import boto3

# Exchange ID token for AWS credentials
identity = boto3.client('cognito-identity')

identity_id = identity.get_id(
    IdentityPoolId='region:pool-id',
    Logins={
        'cognito-idp.region.amazonaws.com/user-pool-id': id_token
    }
)['IdentityId']

credentials = identity.get_credentials_for_identity(
    IdentityId=identity_id,
    Logins={
        'cognito-idp.region.amazonaws.com/user-pool-id': id_token
    }
)['Credentials']

# Use temporary credentials to access S3
s3 = boto3.client(
    's3',
    aws_access_key_id=credentials['AccessKeyId'],
    aws_secret_access_key=credentials['SecretKey'],
    aws_session_token=credentials['SessionToken']
)

s3.upload_file('profile.jpg', 'user-uploads', f"{identity_id}/profile.jpg")
```

### Social Login

**Configuration**:
- Add identity providers: Google, Facebook, Amazon
- Configure OAuth settings (redirect URLs, scopes)

**Sign In Flow**:
- User clicks "Sign in with Google"
- Redirected to Google login
- Google returns authorization code
- Exchange code for Cognito tokens

---

## Scenario 6: Real-Time Data Processing with DynamoDB Streams

**Use Case**: Real-time analytics on user activity

**Architecture**:
```
User Action → API Gateway → Lambda → DynamoDB
                                        ↓
                              DynamoDB Streams
                                        ↓
                              Lambda (stream processor)
                                        ↓
                      ElastiCache (aggregated data)
```

**Services Involved**:
- API Gateway
- Lambda
- DynamoDB
- DynamoDB Streams
- ElastiCache (Redis)

### Implementation

**1. Enable Streams on DynamoDB Table**:
- View type: NEW_AND_OLD_IMAGES (see before and after)

**2. Create Lambda Function for Stream Processing**:
```python
import boto3
import json

redis_client = boto3.client('elasticache')

def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'INSERT':
            new_image = record['dynamodb']['NewImage']
            user_id = new_image['userId']['S']
            action = new_image['action']['S']

            # Increment counter in Redis
            redis.incr(f"user:{user_id}:actions")
            redis.incr(f"action:{action}:count")

        elif record['eventName'] == 'MODIFY':
            old_image = record['dynamodb']['OldImage']
            new_image = record['dynamodb']['NewImage']

            # Detect changes
            if old_image['status']['S'] != new_image['status']['S']:
                # Status changed, update analytics
                pass
```

**3. Event Source Mapping**:
- Lambda polls DynamoDB Streams
- Batch size: 100 records
- Starting position: LATEST or TRIM_HORIZON

### Use Cases

- Real-time analytics dashboards
- Data replication (sync to Elasticsearch)
- Audit logging
- Trigger notifications on changes
- Maintain materialized views

---

## Scenario 7: CI/CD Pipeline

**Use Case**: Automated deployment pipeline for Lambda function

**Architecture**:
```
Developer pushes code to CodeCommit
    ↓
CodePipeline triggered
    ├── Source: CodeCommit
    ├── Build: CodeBuild (run tests, package)
    ├── Deploy to Dev: CodeDeploy (Lambda canary)
    ├── Manual Approval
    └── Deploy to Prod: CodeDeploy (Lambda canary)
```

**Services Involved**:
- CodeCommit
- CodeBuild
- CodeDeploy
- CodePipeline
- Lambda
- CloudWatch (monitoring deployments)

### buildspec.yml

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
  pre_build:
    commands:
      - echo "Installing dependencies"
      - pip install -r requirements.txt -t .
      - pip install pytest
      - echo "Running tests"
      - pytest tests/
  build:
    commands:
      - echo "Packaging Lambda function"
      - zip -r function.zip . -x "tests/*" "*.git/*"
artifacts:
  files:
    - function.zip
    - appspec.yml
```

### appspec.yml (for Lambda)

```yaml
version: 0.0
Resources:
  - MyFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: myLambdaFunction
        Alias: live
        CurrentVersion: 1
        TargetVersion: 2
Hooks:
  - BeforeAllowTraffic: "BeforeTrafficShiftHook"
  - AfterAllowTraffic: "AfterTrafficShiftHook"
```

### CodeDeploy Configuration

**Deployment Type**: Canary10Percent5Minutes
- Shift 10% of traffic to new version
- Wait 5 minutes (monitor errors)
- If successful, shift remaining 90%
- If errors, automatic rollback

**Hooks**:
- BeforeAllowTraffic: Run validation tests
- AfterAllowTraffic: Run integration tests

### Pipeline Stages

1. **Source**: Detect code changes in CodeCommit
2. **Build**: Run CodeBuild (tests + package)
3. **Deploy Dev**: Deploy to dev environment
4. **Manual Approval**: Review and approve
5. **Deploy Prod**: Deploy to production

### Monitoring Deployment

**CloudWatch Alarms**:
- Monitor Lambda errors during canary deployment
- If errors increase, CodeDeploy auto-rolls back

**X-Ray**:
- Compare traces between old and new versions
- Detect performance regressions

---

## Scenario 8: Microservices with EventBridge

**Use Case**: E-commerce system with decoupled microservices

**Architecture**:
```
Order Service → EventBridge → [Event Bus]
                                  ↓
          ┌───────────────────────┼───────────────────────┐
          ↓                       ↓                       ↓
    Inventory Service      Payment Service      Notification Service
    (Lambda + DynamoDB)    (Lambda + Stripe)    (Lambda + SNS)
```

**Services Involved**:
- EventBridge
- Lambda
- DynamoDB
- SNS

### Event Schema

**OrderCreated Event**:
```json
{
  "source": "com.mycompany.orders",
  "detail-type": "OrderCreated",
  "detail": {
    "orderId": "12345",
    "userId": "user-123",
    "items": [
      {"sku": "ABC", "quantity": 2, "price": 29.99}
    ],
    "total": 59.98,
    "timestamp": "2023-11-15T10:30:00Z"
  }
}
```

### Publishing Events

**Order Service** (Lambda):
```python
import boto3
events = boto3.client('events')

def create_order(order_data):
    # Save order to DynamoDB
    orders_table.put_item(Item=order_data)

    # Publish event
    events.put_events(
        Entries=[{
            'Source': 'com.mycompany.orders',
            'DetailType': 'OrderCreated',
            'Detail': json.dumps(order_data),
            'EventBusName': 'default'
        }]
    )
```

### Subscribing to Events

**EventBridge Rules**:

1. **Inventory Rule**:
```json
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["OrderCreated"]
}
```
Target: Inventory Lambda function

2. **Payment Rule**:
```json
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["OrderCreated"],
  "detail": {
    "total": [{"numeric": [">", 0]}]
  }
}
```
Target: Payment Lambda function

3. **Notification Rule**:
```json
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["OrderCreated", "OrderShipped"]
}
```
Target: Notification Lambda function

### Benefits

- **Decoupling**: Services don't know about each other
- **Scalability**: Each service scales independently
- **Flexibility**: Easy to add new services (just add new rule)
- **Resilience**: If one service fails, others continue

---

## Advanced Patterns

### Caching Strategies

**Multi-Layer Caching**:
```
Client → CloudFront (CDN)
    ↓
API Gateway (cache)
    ↓
Lambda
    ↓
DAX (DynamoDB cache)
    ↓
DynamoDB
```

**When to Use**:
- CloudFront: Static content, globally distributed
- API Gateway: Reduce Lambda invocations
- DAX: Reduce DynamoDB reads, microsecond latency
- ElastiCache: Complex queries, session storage

### Error Handling Patterns

**1. Retry with Exponential Backoff**:
- Built into AWS SDKs
- Configurable max retries

**2. Dead Letter Queue**:
- SQS DLQ for failed messages
- Lambda DLQ for failed async invocations
- Analyze and replay failed events

**3. Circuit Breaker**:
- Stop calling failing service
- Return cached response or error
- Periodically retry

**4. Saga Pattern** (with Step Functions):
- Compensating transactions
- If step fails, undo previous steps

### Security Patterns

**Principle of Least Privilege**:
- Grant minimum necessary permissions
- Use specific resources, not `*`

**Encryption Everywhere**:
- At rest: S3, DynamoDB, RDS with KMS
- In transit: HTTPS, TLS

**Secrets Management**:
- Never hardcode credentials
- Use Secrets Manager or Parameter Store
- Rotate automatically

**Defense in Depth**:
- Network: VPC, security groups, NACLs
- Application: IAM, Cognito
- Data: Encryption, backup

---

These scenarios cover the majority of integration patterns you'll see on the exam. Make sure you understand how services work together, not just in isolation!
