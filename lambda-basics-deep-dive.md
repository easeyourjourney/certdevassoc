# AWS Lambda - Basics: Deep Dive

Comprehensive explanation of Lambda fundamentals covering handler, event, context, and execution role.

---

## 1. Handler Function

### What is a Handler?

The **handler** is the entry point of your Lambda function - the method that AWS Lambda calls to start executing your code.

### Handler Format by Runtime

**Python:**
```python
def lambda_handler(event, context):
    # Your code here
    return {
        'statusCode': 200,
        'body': 'Hello World'
    }
```
- Format: `filename.function_name`
- Example: If file is `index.py`, handler is `index.lambda_handler`

**Node.js:**
```javascript
exports.handler = async (event, context) => {
    // Your code here
    return {
        statusCode: 200,
        body: 'Hello World'
    };
};
```
- Format: `filename.exported_function_name`
- Example: If file is `index.js`, handler is `index.handler`

**Java:**
```java
package example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class Handler implements RequestHandler<Map<String,String>, String> {
    public String handleRequest(Map<String,String> event, Context context) {
        return "Hello World";
    }
}
```
- Format: `package.ClassName::methodName`
- Example: `example.Handler::handleRequest`

### Handler Configuration

When you create a Lambda function, you specify:
```bash
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --handler index.lambda_handler \  # This tells Lambda which function to call
  --role arn:aws:iam::123456789012:role/lambda-role \
  --zip-file fileb://function.zip
```

### Key Points

- Handler must be at the **root level** of your deployment package
- Function must be **exported/public**
- Must accept **event** and **context** parameters
- Can be **synchronous** or **asynchronous** (Node.js, Python 3.8+)

### FAQ: Common Handler Questions

**Q: Why is the entry point called a "handler"?**

A: The term "handler" comes from event-driven programming. Lambda functions **handle** incoming events - whether that's an API request, S3 upload, or scheduled task. The naming convention is consistent with other event-driven systems (onClick handlers in JavaScript, HTTP request handlers in web frameworks, etc.). When something triggers your function, the handler is the method that handles that event.

**Q: Can I change the function name from `lambda_handler`?**

A: Yes! The name `lambda_handler` is just a **convention**, not a requirement. You can name your function anything you want - just update the handler configuration to match.

**Examples:**
```python
# index.py
def process_user_data(event, context):  # Custom name
    return {'statusCode': 200}
```

```bash
# Tell Lambda to use your custom function name
--handler index.process_user_data
```

You can even have multiple functions in the same file and switch between them:
```bash
# Switch to a different function without redeploying code
aws lambda update-function-configuration \
  --function-name MyFunction \
  --handler index.new_function_name
```

**Key takeaway:** The handler configuration (`filename.function_name`) tells Lambda which function to call. You control both the filename and function name - just make sure the configuration matches your actual code.

### Exercise: Create Your First Lambda Function

**Objective**: Create and test a simple Lambda function to understand handlers, events, and context.

**What You'll Learn**:
- How to create a Lambda function from scratch
- How handlers receive and process events
- How to use the context object
- How to test Lambda functions

**Step 1: Create the Lambda Function Code**

Create a file called `my_first_lambda.py`:
```python
import json

def my_handler(event, context):
    """
    My first Lambda function that demonstrates handler basics
    """
    # Use context to get runtime information
    print(f"Function name: {context.function_name}")
    print(f"Request ID: {context.aws_request_id}")
    print(f"Memory limit: {context.memory_limit_in_mb} MB")
    print(f"Time remaining: {context.get_remaining_time_in_millis()} ms")

    # Process the event
    name = event.get('name', 'World')
    message = f"Hello, {name}!"

    # Log the event for debugging
    print(f"Received event: {json.dumps(event)}")

    # Return response
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': message,
            'requestId': context.aws_request_id
        })
    }
```

**Step 2: Create IAM Role for Lambda**

```bash
# Create trust policy
aws iam create-role \
  --role-name MyFirstLambdaRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach basic execution policy (for CloudWatch Logs)
aws iam attach-role-policy \
  --role-name MyFirstLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Step 3: Package and Deploy**

```bash
# Create deployment package
zip function.zip my_first_lambda.py

# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create Lambda function
aws lambda create-function \
  --function-name MyFirstLambda \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler my_first_lambda.my_handler \
  --zip-file fileb://function.zip \
  --description "My first Lambda function to learn handlers"
```

**Step 4: Test Your Function**

```bash
# Test with a simple event
aws lambda invoke \
  --function-name MyFirstLambda \
  --payload '{"name": "Student"}' \
  response.json

# View the response
cat response.json

# View the logs
aws logs tail /aws/lambda/MyFirstLambda --follow
```

**Step 5: Experiment with Different Handlers**

Add another function to `my_first_lambda.py`:
```python
def alternative_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'This is a different handler!'})
    }
```

Update the deployment and switch handlers:
```bash
# Update code
zip function.zip my_first_lambda.py
aws lambda update-function-code \
  --function-name MyFirstLambda \
  --zip-file fileb://function.zip

# Switch to the alternative handler
aws lambda update-function-configuration \
  --function-name MyFirstLambda \
  --handler my_first_lambda.alternative_handler

# Test again
aws lambda invoke \
  --function-name MyFirstLambda \
  --payload '{"name": "Student"}' \
  response.json
```

**Expected Results**:
- Function executes successfully with status code 200
- Response includes your custom message
- CloudWatch Logs show context information (function name, request ID, etc.)
- You can switch between handlers without redeploying code

**Cleanup** (when done):
```bash
# Delete function
aws lambda delete-function --function-name MyFirstLambda

# Delete role (detach policy first)
aws iam detach-role-policy \
  --role-name MyFirstLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name MyFirstLambdaRole

# Remove local files
rm function.zip response.json my_first_lambda.py
```

### Additional Tip: Understanding `--cli-binary-format`

When testing Lambda functions, you might encounter this error:
```
Invalid base64: "{"name": "Student"}"
```

**Why This Happens:**

AWS CLI v2 changed how it handles binary parameters like `--payload`. By default, it expects binary data to be base64-encoded.

**The Solution:**

Use the `--cli-binary-format raw-in-base64-out` flag:
```bash
aws lambda invoke \
  --cli-binary-format raw-in-base64-out \
  --payload '{"name": "Student"}' \
  response.json
```

This tells AWS CLI:
- **raw-in**: Accept raw input (plain JSON) - don't require base64
- **base64-out**: Encode output as base64 if needed

**Alternative Approaches:**

**Option 1: Use a file** (recommended for complex payloads):
```bash
echo '{"name": "Student"}' > event.json
aws lambda invoke \
  --function-name MyFirstLambda \
  --payload file://event.json \
  response.json
```

**Option 2: Set it globally** (configure once, use everywhere):
```bash
# Add to ~/.aws/config
[default]
cli_binary_format = raw-in-base64-out
```

**Option 3: Base64 encode manually**:
```bash
aws lambda invoke \
  --function-name MyFirstLambda \
  --payload $(echo '{"name": "Student"}' | base64) \
  response.json
```

**Best Practice:** For Lambda testing, either set `cli_binary_format = raw-in-base64-out` globally in your AWS config or use `file://` to pass payloads from files.

---

## 2. Event Object

### What is the Event?

The **event** is a JSON-formatted document that contains data for the function to process. It varies depending on the **trigger source**.

### Event Structure by Trigger

**API Gateway Event:**
```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer token123"
  },
  "queryStringParameters": {
    "page": "1",
    "limit": "10"
  },
  "body": "{\"name\":\"John\",\"email\":\"john@example.com\"}",
  "requestContext": {
    "requestId": "abc-123",
    "authorizer": {
      "claims": {
        "sub": "user-id-123",
        "email": "john@example.com"
      }
    }
  }
}
```

**S3 Event:**
```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-bucket",
          "arn": "arn:aws:s3:::my-bucket"
        },
        "object": {
          "key": "images/photo.jpg",
          "size": 1024,
          "eTag": "abc123"
        }
      }
    }
  ]
}
```

**DynamoDB Stream Event:**
```json
{
  "Records": [
    {
      "eventName": "INSERT",
      "dynamodb": {
        "Keys": {
          "userId": {"S": "user-123"}
        },
        "NewImage": {
          "userId": {"S": "user-123"},
          "name": {"S": "John Doe"},
          "email": {"S": "john@example.com"}
        },
        "SequenceNumber": "111",
        "SizeBytes": 26,
        "StreamViewType": "NEW_AND_OLD_IMAGES"
      }
    }
  ]
}
```

**SQS Event:**
```json
{
  "Records": [
    {
      "messageId": "abc-123",
      "body": "{\"orderId\":\"12345\",\"total\":99.99}",
      "attributes": {
        "ApproximateReceiveCount": "1",
        "SentTimestamp": "1545082649183"
      },
      "messageAttributes": {},
      "md5OfBody": "abc123",
      "eventSource": "aws:sqs"
    }
  ]
}
```

**CloudWatch Scheduled Event:**
```json
{
  "version": "0",
  "id": "abc-123",
  "detail-type": "Scheduled Event",
  "source": "aws.events",
  "time": "2023-11-15T10:30:00Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:events:us-east-1:123456789012:rule/my-schedule"
  ],
  "detail": {}
}
```

### Accessing Event Data

**Python:**
```python
def lambda_handler(event, context):
    # API Gateway
    http_method = event['httpMethod']
    body = json.loads(event['body'])
    user_id = event['requestContext']['authorizer']['claims']['sub']

    # S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # SQS
    for record in event['Records']:
        message_body = json.loads(record['body'])
        order_id = message_body['orderId']

    return {'statusCode': 200}
```

**Node.js:**
```javascript
exports.handler = async (event) => {
    // API Gateway
    const method = event.httpMethod;
    const body = JSON.parse(event.body);

    // S3
    const bucket = event.Records[0].s3.bucket.name;
    const key = event.Records[0].s3.object.key;

    return { statusCode: 200 };
};
```

### Custom Event (Direct Invocation)

You can invoke Lambda directly with custom data:
```bash
aws lambda invoke \
  --function-name MyFunction \
  --payload '{"name":"John","age":30}' \
  output.json
```

Function receives:
```python
def lambda_handler(event, context):
    name = event['name']  # "John"
    age = event['age']    # 30
    return f"Hello {name}, you are {age} years old"
```

---

## 3. Context Object

### What is Context?

The **context** object provides runtime information about the Lambda function execution and environment.

### Context Properties

**Python Example:**
```python
def lambda_handler(event, context):
    print(f"Request ID: {context.aws_request_id}")
    print(f"Function name: {context.function_name}")
    print(f"Function version: {context.function_version}")
    print(f"Memory limit: {context.memory_limit_in_mb} MB")
    print(f"Time remaining: {context.get_remaining_time_in_millis()} ms")
    print(f"Log group: {context.log_group_name}")
    print(f"Log stream: {context.log_stream_name}")

    # Check if running out of time
    if context.get_remaining_time_in_millis() < 1000:
        print("Warning: Less than 1 second remaining!")
        return {'statusCode': 500, 'body': 'Timeout imminent'}

    # Do your work
    return {'statusCode': 200}
```

### Context Properties Reference

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `aws_request_id` | string | Unique request ID for this invocation | `abc-123-def-456` |
| `function_name` | string | Name of Lambda function | `MyFunction` |
| `function_version` | string | Version being executed | `$LATEST` or `1` |
| `invoked_function_arn` | string | ARN used to invoke function | `arn:aws:lambda:us-east-1:123456789012:function:MyFunction` |
| `memory_limit_in_mb` | int | Memory allocated to function | `128` |
| `log_group_name` | string | CloudWatch log group | `/aws/lambda/MyFunction` |
| `log_stream_name` | string | CloudWatch log stream | `2023/11/15/[$LATEST]abc123` |
| `identity` | object | Info about Amazon Cognito identity (mobile apps) | |
| `client_context` | object | Info about client app and device (mobile apps) | |

### Context Methods

**`get_remaining_time_in_millis()`** (Python):
```python
def lambda_handler(event, context):
    start_time = context.get_remaining_time_in_millis()

    # Do some work
    process_data()

    # Check time
    remaining = context.get_remaining_time_in_millis()
    elapsed = start_time - remaining

    print(f"Processing took {elapsed} ms")
    print(f"Remaining time: {remaining} ms")
```

**`getRemainingTimeInMillis()`** (Node.js):
```javascript
exports.handler = async (event, context) => {
    console.log(`Time remaining: ${context.getRemainingTimeInMillis()} ms`);

    // Do work
    await processData();

    console.log(`Time remaining after processing: ${context.getRemainingTimeInMillis()} ms`);
};
```

### Practical Use Cases for Context

**1. Graceful Timeout Handling:**
```python
def lambda_handler(event, context):
    results = []

    for item in large_dataset:
        # Check if we have at least 10 seconds left
        if context.get_remaining_time_in_millis() < 10000:
            print(f"Timeout approaching. Processed {len(results)} items")
            # Save progress to DynamoDB
            save_checkpoint(results)
            return {'statusCode': 206, 'processed': len(results)}

        results.append(process_item(item))

    return {'statusCode': 200, 'processed': len(results)}
```

**2. Logging and Debugging:**
```python
import json

def lambda_handler(event, context):
    # Log execution details
    log_entry = {
        'request_id': context.aws_request_id,
        'function_name': context.function_name,
        'function_version': context.function_version,
        'memory_limit': context.memory_limit_in_mb,
        'event': event
    }
    print(json.dumps(log_entry))

    # Your logic
    result = do_something(event)

    return result
```

**3. Version-Specific Logic:**
```python
def lambda_handler(event, context):
    if context.function_version == '$LATEST':
        # Development behavior
        enable_debug_logging()
    else:
        # Production behavior
        enable_standard_logging()

    return process_request(event)
```

---

## 4. Execution Role (IAM Role)

### What is an Execution Role?

The **execution role** is an IAM role that grants the Lambda function **permissions** to access AWS services and resources.

### Why is it Needed?

Lambda functions run in AWS's managed infrastructure. To access other AWS services (S3, DynamoDB, SNS, etc.), Lambda needs permissions. The execution role provides these permissions.

### Basic Execution Role Structure

**Trust Policy** (who can assume the role):
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
This allows Lambda service to assume the role.

**Permissions Policy** (what the role can do):
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
    }
  ]
}
```

### Common Managed Policies

**1. AWSLambdaBasicExecutionRole** (CloudWatch Logs):
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
      "Resource": "*"
    }
  ]
}
```
Use when: Function only needs to write logs

**2. AWSLambdaVPCAccessExecutionRole** (VPC Access):
Includes BasicExecutionRole + VPC network interface permissions
Use when: Function needs to access resources in VPC (RDS, ElastiCache)

**3. AWSLambdaDynamoDBExecutionRole** (DynamoDB Streams):
Includes BasicExecutionRole + DynamoDB Streams read permissions
Use when: Function is triggered by DynamoDB Streams

### Creating Custom Execution Role

**Step 1: Create Trust Policy** (`trust-policy.json`):
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

**Step 2: Create Role:**
```bash
aws iam create-role \
  --role-name MyLambdaRole \
  --assume-role-policy-document file://trust-policy.json
```

**Step 3: Attach Basic Execution Policy:**
```bash
aws iam attach-role-policy \
  --role-name MyLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Step 4: Create Custom Policy for Your Needs** (`my-policy.json`):
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
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:MyTopic"
    }
  ]
}
```

**Step 5: Attach Custom Policy:**
```bash
aws iam put-role-policy \
  --role-name MyLambdaRole \
  --policy-name MyLambdaPolicy \
  --policy-document file://my-policy.json
```

**Step 6: Use Role with Lambda:**
```bash
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/MyLambdaRole \
  --handler index.lambda_handler \
  --zip-file fileb://function.zip
```

### Using AWS SDK with Execution Role

**Python:**
```python
import boto3

def lambda_handler(event, context):
    # AWS SDK automatically uses the execution role
    # No credentials needed in code!

    # Access DynamoDB
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('MyTable')
    table.put_item(Item={'id': '123', 'name': 'John'})

    # Access S3
    s3 = boto3.client('s3')
    s3.put_object(Bucket='my-bucket', Key='file.txt', Body='Hello')

    # Access SNS
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:MyTopic',
        Message='Hello from Lambda'
    )

    return {'statusCode': 200}
```

### Best Practices for Execution Roles

**1. Principle of Least Privilege:**
```json
{
  "Effect": "Allow",
  "Action": "dynamodb:GetItem",  // Only GetItem, not PutItem/DeleteItem
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"  // Specific table, not *
}
```

**2. One Role Per Function (or logical grouping):**
- Don't reuse roles across unrelated functions
- Makes permissions easier to audit and manage

**3. Use Conditions for Additional Security:**
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": "10.0.0.0/16"
    }
  }
}
```

**4. Avoid Using AWS Managed Keys in Production:**
- Create customer-managed policies for production
- Easier to audit and control

### Common Permission Issues

**Error: AccessDeniedException**
```
botocore.exceptions.ClientError: An error occurred (AccessDeniedException)
when calling the PutItem operation: User: arn:aws:sts::123:assumed-role/MyLambdaRole
is not authorized to perform: dynamodb:PutItem on resource: arn:aws:dynamodb:us-east-1:123:table/MyTable
```

**Solution:** Add permission to execution role:
```json
{
  "Effect": "Allow",
  "Action": "dynamodb:PutItem",
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
}
```

---

## Complete Example: Putting It All Together

```python
import json
import boto3
import os
from datetime import datetime

# Initialize AWS SDK clients (uses execution role automatically)
dynamodb = boto3.resource('dynamodb')
s3 = boto3.client('s3')

def lambda_handler(event, context):
    """
    Lambda function that processes user registrations
    - Receives user data from API Gateway
    - Stores in DynamoDB
    - Uploads profile to S3
    """

    # ===== CONTEXT USAGE =====
    print(f"Request ID: {context.aws_request_id}")
    print(f"Function: {context.function_name}")
    print(f"Time remaining: {context.get_remaining_time_in_millis()} ms")

    # ===== EVENT USAGE =====
    # Parse API Gateway event
    try:
        body = json.loads(event['body'])
        user_id = event['requestContext']['authorizer']['claims']['sub']
        email = body['email']
        name = body['name']
    except (KeyError, json.JSONDecodeError) as e:
        print(f"Invalid event: {e}")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Invalid request'})
        }

    # ===== EXECUTION ROLE IN ACTION =====
    try:
        # Store in DynamoDB (requires dynamodb:PutItem permission)
        table = dynamodb.Table(os.environ['TABLE_NAME'])
        table.put_item(Item={
            'userId': user_id,
            'email': email,
            'name': name,
            'createdAt': datetime.utcnow().isoformat()
        })

        # Upload to S3 (requires s3:PutObject permission)
        s3.put_object(
            Bucket=os.environ['BUCKET_NAME'],
            Key=f'users/{user_id}/profile.json',
            Body=json.dumps({'email': email, 'name': name}),
            ContentType='application/json'
        )

        print(f"Successfully registered user {user_id}")

        return {
            'statusCode': 201,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({
                'message': 'User registered successfully',
                'userId': user_id
            })
        }

    except Exception as e:
        # Log error (requires logs:PutLogEvents permission)
        print(f"Error: {str(e)}")

        return {
            'statusCode': 500,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({'error': 'Internal server error'})
        }
```

**Required Execution Role:**
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
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UsersTable"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-users-bucket/users/*"
    }
  ]
}
```

---

## Summary

### Handler
- Entry point of your Lambda function
- Format: `filename.function_name`
- Must accept `event` and `context` parameters

### Event
- JSON data passed to your function
- Structure depends on trigger source (API Gateway, S3, SQS, etc.)
- Access with `event['key']` or `event.key`

### Context
- Runtime information about the execution
- Provides request ID, function name, memory limit, time remaining
- Use `context.get_remaining_time_in_millis()` to avoid timeouts

### Execution Role
- IAM role that grants permissions to access AWS services
- Consists of trust policy (who can assume) and permissions policy (what can do)
- Follow least privilege principle
- AWS SDK automatically uses the role - no credentials in code!

---

These are the fundamental building blocks of every Lambda function. Master these concepts and you'll understand 80% of Lambda development!
