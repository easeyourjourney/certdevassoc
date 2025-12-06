# Hands-On Labs

Practical exercises to build real AWS applications. Complete these labs to gain hands-on experience.

---

## Lab 1: Serverless REST API with Authentication

**Objective**: Build a complete serverless API with user authentication

**What You'll Build**:
- Cognito User Pool for authentication
- API Gateway REST API
- Lambda functions for CRUD operations
- DynamoDB table for data storage
- CloudWatch for monitoring

### Step-by-Step Instructions

**1. Create DynamoDB Table**:
```bash
aws dynamodb create-table \
  --table-name Notes \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=noteId,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=noteId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

**2. Create Cognito User Pool**:
```bash
aws cognito-idp create-user-pool \
  --pool-name NotesUserPool \
  --auto-verified-attributes email \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true
    }
  }'

# Note the UserPoolId from output

aws cognito-idp create-user-pool-client \
  --user-pool-id <UserPoolId> \
  --client-name NotesAppClient \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH
```

**3. Create Lambda Function**:

`create_note.py`:
```python
import json
import boto3
from uuid import uuid4
import time

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Notes')

def lambda_handler(event, context):
    try:
        # Get user ID from Cognito authorizer
        user_id = event['requestContext']['authorizer']['claims']['sub']

        # Parse request body
        body = json.loads(event['body'])
        title = body.get('title')
        content = body.get('content')

        if not title or not content:
            return {
                'statusCode': 400,
                'headers': {'Access-Control-Allow-Origin': '*'},
                'body': json.dumps({'error': 'Title and content are required'})
            }

        # Create note
        note = {
            'userId': user_id,
            'noteId': str(uuid4()),
            'title': title,
            'content': content,
            'createdAt': int(time.time())
        }

        table.put_item(Item=note)

        return {
            'statusCode': 201,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps(note)
        }

    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({'error': 'Internal server error'})
        }
```

Create IAM role and deploy:
```bash
# Create execution role (with DynamoDB permissions)
aws iam create-role --role-name LambdaNotesRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach policies
aws iam attach-role-policy \
  --role-name LambdaNotesRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create inline policy for DynamoDB
aws iam put-role-policy \
  --role-name LambdaNotesRole \
  --policy-name DynamoDBAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Query",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/Notes"
    }]
  }'

# Package and deploy Lambda
zip function.zip create_note.py
aws lambda create-function \
  --function-name CreateNote \
  --runtime python3.9 \
  --role arn:aws:iam::<account-id>:role/LambdaNotesRole \
  --handler create_note.lambda_handler \
  --zip-file fileb://function.zip
```

**4. Create API Gateway**:
```bash
# Create REST API
aws apigateway create-rest-api \
  --name NotesAPI \
  --description "Notes API with Cognito auth"

# Create authorizer (Cognito)
aws apigateway create-authorizer \
  --rest-api-id <api-id> \
  --name CognitoAuthorizer \
  --type COGNITO_USER_POOLS \
  --provider-arns arn:aws:cognito-idp:region:<account-id>:userpool/<UserPoolId> \
  --identity-source method.request.header.Authorization
```

Continue configuring resources, methods, and deploy the API.

**5. Test the API**:
```bash
# Sign up user
aws cognito-idp sign-up \
  --client-id <ClientId> \
  --username test@example.com \
  --password TestPassword123!

# Confirm sign-up (use code from email or auto-confirm)
aws cognito-idp admin-confirm-sign-up \
  --user-pool-id <UserPoolId> \
  --username test@example.com

# Sign in
aws cognito-idp initiate-auth \
  --client-id <ClientId> \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=test@example.com,PASSWORD=TestPassword123!

# Get IdToken from output

# Call API
curl -X POST https://<api-id>.execute-api.region.amazonaws.com/prod/notes \
  -H "Authorization: <IdToken>" \
  -H "Content-Type: application/json" \
  -d '{"title": "My First Note", "content": "Hello World"}'
```

### Challenges

1. Add GET, UPDATE, DELETE endpoints
2. Add pagination for list notes
3. Add search functionality
4. Implement rate limiting with usage plans
5. Enable X-Ray tracing and analyze performance

---

## Lab 2: Event-Driven File Processing

**Objective**: Build an automated image processing pipeline

**What You'll Build**:
- S3 bucket for uploads
- Lambda for image processing (resize, thumbnail)
- SNS + SQS for fan-out pattern
- DynamoDB for metadata

### Instructions

**1. Create S3 Buckets**:
```bash
aws s3 mb s3://myapp-uploads-<unique-id>
aws s3 mb s3://myapp-processed-<unique-id>
```

**2. Create SNS Topic and SQS Queues**:
```bash
# Create SNS topic
aws sns create-topic --name ImageProcessing

# Create SQS queues
aws sqs create-queue --queue-name ImageResizeQueue
aws sqs create-queue --queue-name ImageThumbnailQueue

# Subscribe queues to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:region:account:ImageProcessing \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:region:account:ImageResizeQueue
```

**3. Configure S3 Event Notification**:
```json
{
  "TopicConfigurations": [{
    "TopicArn": "arn:aws:sns:region:account:ImageProcessing",
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

**4. Create Lambda Function**:

`resize_image.py`:
```python
import json
import boto3
from PIL import Image
import io

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ImageMetadata')

def lambda_handler(event, context):
    for record in event['Records']:
        # Parse SNS message containing S3 event
        message = json.loads(record['body'])
        s3_event = json.loads(message['Message'])

        for s3_record in s3_event['Records']:
            bucket = s3_record['s3']['bucket']['name']
            key = s3_record['s3']['object']['key']

            # Download image
            obj = s3.get_object(Bucket=bucket, Key=key)
            img = Image.open(io.BytesIO(obj['Body'].read()))

            # Resize to 800x600
            img_resized = img.resize((800, 600), Image.LANCZOS)

            # Upload resized image
            buffer = io.BytesIO()
            img_resized.save(buffer, 'JPEG')
            buffer.seek(0)

            processed_key = f"resized/{key}"
            s3.put_object(
                Bucket='myapp-processed-<unique-id>',
                Key=processed_key,
                Body=buffer,
                ContentType='image/jpeg'
            )

            # Save metadata to DynamoDB
            table.put_item(Item={
                'imageId': key,
                'originalBucket': bucket,
                'processedBucket': 'myapp-processed-<unique-id>',
                'processedKey': processed_key,
                'width': 800,
                'height': 600
            })

    return {'statusCode': 200}
```

### Challenges

1. Add error handling and DLQ
2. Create thumbnail generator (separate Lambda)
3. Add image format conversion (JPG → PNG)
4. Implement retry logic
5. Monitor queue depth and processing time

---

## Lab 3: CI/CD Pipeline for Lambda

**Objective**: Build automated deployment pipeline

**What You'll Build**:
- CodeCommit repository
- CodeBuild for testing and packaging
- CodeDeploy for canary deployments
- CodePipeline to orchestrate

### Instructions

**1. Create CodeCommit Repository**:
```bash
aws codecommit create-repository --repository-name my-lambda-app

# Clone repository
git clone https://git-codecommit.region.amazonaws.com/v1/repos/my-lambda-app
cd my-lambda-app
```

**2. Create Lambda Function Code**:

`index.py`:
```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Hello from version 1!'
    }
```

`tests/test_handler.py`:
```python
from index import lambda_handler

def test_lambda_handler():
    result = lambda_handler({}, {})
    assert result['statusCode'] == 200
```

`requirements.txt`:
```
pytest
```

**3. Create buildspec.yml**:
```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - pip install -r requirements.txt
      - pip install pytest
  pre_build:
    commands:
      - echo "Running tests"
      - pytest tests/
  build:
    commands:
      - echo "Packaging function"
      - zip -r function.zip index.py
artifacts:
  files:
    - function.zip
    - appspec.yml
```

**4. Create appspec.yml**:
```yaml
version: 0.0
Resources:
  - MyFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: MyLambdaFunction
        Alias: live
        CurrentVersion: 1
        TargetVersion: 2
```

**5. Push to CodeCommit**:
```bash
git add .
git commit -m "Initial commit"
git push origin main
```

**6. Create CodeBuild Project** (via console or CLI)

**7. Create CodeDeploy Application**:
```bash
aws deploy create-application \
  --application-name MyLambdaApp \
  --compute-platform Lambda

aws deploy create-deployment-config \
  --deployment-config-name Canary10Percent5Minutes \
  --compute-platform Lambda \
  --traffic-routing-config '{
    "type": "TimeBasedCanary",
    "timeBasedCanary": {
      "canaryPercentage": 10,
      "canaryInterval": 5
    }
  }'
```

**8. Create CodePipeline** (via console)
- Source: CodeCommit
- Build: CodeBuild
- Deploy: CodeDeploy

### Challenges

1. Add manual approval stage
2. Add environment variables per stage
3. Implement blue/green deployment
4. Add integration tests after deployment
5. Set up CloudWatch alarms for rollback

---

## Lab 4: Distributed Tracing with X-Ray

**Objective**: Implement end-to-end tracing

**What You'll Build**:
- API Gateway with X-Ray enabled
- Lambda functions with X-Ray SDK
- DynamoDB with tracing
- X-Ray service map

### Instructions

**1. Enable X-Ray on API Gateway**:
```bash
aws apigateway update-stage \
  --rest-api-id <api-id> \
  --stage-name prod \
  --patch-operations op=replace,path=/tracingEnabled,value=true
```

**2. Update Lambda with X-Ray SDK**:

`requirements.txt`:
```
aws-xray-sdk
boto3
```

`traced_function.py`:
```python
import json
import boto3
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch all AWS SDK calls
patch_all()

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Items')

@xray_recorder.capture('get_item_from_db')
def get_item(item_id):
    response = table.get_item(Key={'id': item_id})
    return response.get('Item')

def lambda_handler(event, context):
    # Add annotation (indexed, searchable)
    xray_recorder.put_annotation('user_id', event.get('userId', 'unknown'))

    # Add metadata (not indexed)
    xray_recorder.put_metadata('event_data', event)

    # Subsegment for custom logic
    with xray_recorder.capture('business_logic'):
        item_id = event.get('id')
        item = get_item(item_id)

        # Simulate processing
        import time
        time.sleep(0.1)

    return {
        'statusCode': 200,
        'body': json.dumps(item)
    }
```

**3. Enable Active Tracing on Lambda**:
```bash
aws lambda update-function-configuration \
  --function-name MyFunction \
  --tracing-config Mode=Active
```

**4. Update Lambda IAM Role** (add X-Ray permissions):
```json
{
  "Effect": "Allow",
  "Action": [
    "xray:PutTraceSegments",
    "xray:PutTelemetryRecords"
  ],
  "Resource": "*"
}
```

**5. Generate Traffic and View Traces**:
```bash
# Make multiple requests
for i in {1..50}; do
  curl https://<api-id>.execute-api.region.amazonaws.com/prod/items?id=$i
done
```

Go to X-Ray console and view:
- Service map
- Traces
- Analytics

### Challenges

1. Add custom subsegments for different operations
2. Implement error tracking with X-Ray
3. Compare performance before/after optimization
4. Create CloudWatch alarms based on X-Ray insights
5. Trace cross-service calls (Lambda → Lambda)

---

## Lab 5: Containerized Application on ECS

**Objective**: Deploy containerized web app with database

**What You'll Build**:
- Docker container
- ECR repository
- ECS Fargate cluster and service
- RDS database
- Application Load Balancer

### Instructions

**1. Create Application**:

`app.py` (Flask app):
```python
from flask import Flask, jsonify
import os
import psycopg2

app = Flask(__name__)

def get_db_connection():
    return psycopg2.connect(
        host=os.environ['DB_HOST'],
        database=os.environ['DB_NAME'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD']
    )

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/items')
def items():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT * FROM items')
    items = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(items)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

**2. Create Dockerfile**:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 80

CMD ["python", "app.py"]
```

`requirements.txt`:
```
Flask==2.0.1
psycopg2-binary==2.9.1
```

**3. Build and Push to ECR**:
```bash
# Create ECR repository
aws ecr create-repository --repository-name myapp

# Build image
docker build -t myapp .

# Tag
docker tag myapp:latest <account-id>.dkr.ecr.region.amazonaws.com/myapp:latest

# Login
aws ecr get-login-password --region region | docker login --username AWS --password-stdin <account-id>.dkr.ecr.region.amazonaws.com

# Push
docker push <account-id>.dkr.ecr.region.amazonaws.com/myapp:latest
```

**4. Create RDS Database**:
```bash
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password <password> \
  --allocated-storage 20
```

**5. Store Credentials in Secrets Manager**:
```bash
aws secretsmanager create-secret \
  --name myapp/db \
  --secret-string '{
    "username":"admin",
    "password":"<password>",
    "host":"myapp-db.abc123.region.rds.amazonaws.com",
    "port":"5432",
    "dbname":"postgres"
  }'
```

**6. Create ECS Task Definition, Service, and Cluster** (via console or CloudFormation)

### Challenges

1. Add auto-scaling based on CPU
2. Implement health checks
3. Add CloudWatch Container Insights
4. Implement blue/green deployment
5. Add ElastiCache for session storage

---

## Completion Checklist

After completing all labs, you should be able to:

- [ ] Build serverless APIs with authentication
- [ ] Implement event-driven architectures
- [ ] Create CI/CD pipelines
- [ ] Implement distributed tracing
- [ ] Deploy containerized applications
- [ ] Configure monitoring and alarms
- [ ] Handle errors and implement retries
- [ ] Manage secrets securely
- [ ] Implement caching strategies
- [ ] Debug production issues

Congratulations! You now have hands-on experience with all major AWS Developer services.
