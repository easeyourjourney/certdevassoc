# AWS Developer Associate - Complete Concepts Guide

Comprehensive coverage of all exam topics. Study these concepts systematically and check them off in README.md.

---

## Compute Services

### AWS Lambda

**What It Is**: Serverless compute service - run code without managing servers

**Key Concepts**:
- **Handler Function**: Entry point (e.g., `lambda_handler(event, context)` in Python)
- **Event Object**: Input data passed to function
- **Context Object**: Runtime info (request ID, time remaining, memory limit, etc.)
- **Execution Role**: IAM role that grants Lambda permissions to AWS services

**Configuration**:
- **Timeout**: 3 seconds (default) to 900 seconds (15 minutes max)
- **Memory**: 128 MB to 10,240 MB (10 GB) - CPU power scales with memory
- **Environment Variables**: Key-value pairs for configuration
- **Ephemeral Storage (/tmp)**: 512 MB to 10,240 MB - temporary file storage

**Concurrency**:
- **Default Limit**: 1,000 concurrent executions per region
- **Reserved Concurrency**: Guarantee capacity for critical functions (reduces available pool)
- **Provisioned Concurrency**: Keep functions initialized (avoid cold starts)
- **Throttling**: When limit exceeded, returns 429 TooManyRequestsException

**Advanced Features**:
- **Layers**: Share code, libraries, custom runtimes across functions
  - Max 5 layers per function
  - Max 250 MB unzipped (all layers + function)
- **Versions**: Immutable snapshots of function code + configuration
  - $LATEST is always mutable (development version)
  - Numbered versions are immutable (v1, v2, v3...)
- **Aliases**: Pointers to versions (e.g., PROD → v3, DEV → $LATEST)
  - Enable blue/green deployments
  - Can split traffic between versions (canary deployments)
- **Destinations**: Route success/failure to SQS, SNS, EventBridge, or another Lambda
  - Only for asynchronous invocations
- **Dead Letter Queue (DLQ)**: Send failed events to SQS or SNS after retries exhausted

**VPC Integration**:
- Lambda can access resources in VPC (RDS, ElastiCache, internal services)
- Requires subnet IDs and security group IDs
- Lambda creates Elastic Network Interfaces (ENIs) in your VPC
- VPC functions cannot access internet unless:
  - Subnet has NAT Gateway (private subnet) or
  - Subnet is public with Internet Gateway

**Invocation Types**:
1. **Synchronous**: Caller waits for response (API Gateway, ALB, SDK invoke)
   - Immediate error returned if failure
2. **Asynchronous**: Lambda queues event and returns immediately (S3, SNS, EventBridge)
   - Retries twice on failure (3 total attempts)
   - Can configure DLQ or Destination
3. **Event Source Mapping**: Lambda polls the source (SQS, DynamoDB Streams, Kinesis)
   - Lambda manages polling
   - Batch size configurable

**Limits to Remember**:
- Deployment package: 50 MB (zipped), 250 MB (unzipped with layers)
- Max execution time: 15 minutes
- Environment variables: 4 KB total
- Concurrent executions: 1,000 per region (can request increase)

**Common Errors**:
- **Cold Start**: First invocation takes longer (initializing environment)
- **Timeout**: Function exceeds configured timeout
- **Out of Memory**: Function uses more memory than allocated
- **Throttling**: Too many concurrent executions

---

### Amazon EC2 (for Developers)

**What It Is**: Virtual servers in the cloud

**Instance Types**:
- **t3/t2**: Burstable performance (web servers, dev environments)
- **m5/m6**: General purpose (balanced compute/memory/network)
- **c5/c6**: Compute optimized (CPU-intensive workloads)
- **r5/r6**: Memory optimized (databases, caching)
- **Example**: t3.micro, m5.large, c5.xlarge

**User Data**:
- Bootstrap script that runs on instance first launch
- Runs as root user
- Used to install software, start services
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
echo "Hello World" > /var/www/html/index.html
```

**Instance Metadata**:
- Data about instance accessible from within instance
- URL: `http://169.254.169.254/latest/meta-data/`
- Get: instance-id, public-ip, ami-id, instance-type, security-groups
- **IMDSv2** (recommended): More secure, requires token
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

**IAM Roles for EC2**:
- Attach IAM role to instance instead of storing credentials
- Credentials automatically rotated
- Access via instance metadata
- Best practice: NEVER store credentials on EC2

**Security Groups**:
- Virtual firewall for instances
- Stateful (return traffic automatically allowed)
- Default: All inbound DENIED, all outbound ALLOWED
- Rules: Protocol, port range, source/destination

---

### Amazon ECS (Elastic Container Service)

**What It Is**: Run Docker containers on AWS

**Launch Types**:
1. **EC2**: You manage EC2 instances (container hosts)
   - More control, can use Spot instances
   - You handle scaling, patching
2. **Fargate**: Serverless - AWS manages infrastructure
   - No servers to manage
   - Pay per vCPU and memory used

**Components**:
- **Task Definition**: Blueprint for application (JSON/YAML)
  - Docker image
  - CPU and memory
  - Environment variables
  - Port mappings
  - IAM task role
  - Logging configuration
- **Task**: Running instance of task definition
  - One or more containers running together
- **Service**: Maintains desired count of tasks
  - Load balancing
  - Auto-scaling
  - Rolling updates
- **Cluster**: Logical grouping of tasks and services

**IAM Roles**:
- **Task Role**: Permissions for the application inside container
  - What the app can do (access S3, DynamoDB, etc.)
- **Task Execution Role**: Permissions for ECS agent
  - Pull images from ECR
  - Send logs to CloudWatch
  - Retrieve secrets from Secrets Manager

**Container Registry**:
- **ECR (Elastic Container Registry)**: AWS Docker registry
  - Private Docker images
  - Integrated with IAM
  - Image scanning for vulnerabilities

---

### AWS Elastic Beanstalk

**What It Is**: Platform as a Service (PaaS) - deploy apps without managing infrastructure

**Supported Platforms**:
- Java, .NET, PHP, Node.js, Python, Ruby, Go
- Docker (single or multi-container)

**Components**:
- **Application**: Logical container
- **Environment**: AWS resources running your app version
  - Web server environment (with load balancer)
  - Worker environment (processes background tasks from SQS)
- **Application Version**: Specific iteration of deployable code

**Deployment Policies**:
1. **All at Once**: Deploy to all instances simultaneously
   - Fastest, but causes downtime
   - Good for dev environments
2. **Rolling**: Deploy in batches
   - No downtime, reduced capacity during deployment
   - Batch size configurable
3. **Rolling with Additional Batch**: Launch new batch first
   - Maintain full capacity
   - Slightly slower, small additional cost
4. **Immutable**: Launch full set of new instances
   - New Auto Scaling Group with new instances
   - Zero downtime, quick rollback
   - Highest cost during deployment
5. **Blue/Green**: Deploy to separate environment, swap URLs
   - Complete new environment
   - Instant rollback
   - DNS change required

**Configuration**:
- `.ebextensions/`: Folder with YAML/JSON config files
  - Customize environment (packages, commands, services)
  - Example: `.ebextensions/01-packages.config`
- **Environment Properties**: Key-value configuration
  - Accessible as environment variables in app

---

## Storage & Databases

### Amazon S3

**What It Is**: Object storage service (files, images, videos, backups)

**Structure**:
- **Buckets**: Containers for objects (globally unique name)
- **Objects**: Files with metadata
  - Key: Object name (including path)
  - Value: Content (data)
  - Metadata: Key-value pairs
  - Max size: 5 TB per object

**Storage Classes**:
1. **S3 Standard**: Frequent access, low latency, 99.99% availability
2. **S3 Intelligent-Tiering**: Auto-moves between frequent/infrequent tiers
3. **S3 Standard-IA** (Infrequent Access): Less frequent but rapid access needed
   - Cheaper storage, retrieval fee
4. **S3 One Zone-IA**: Single AZ, 20% cheaper than Standard-IA
5. **S3 Glacier Instant Retrieval**: Archive, millisecond retrieval
6. **S3 Glacier Flexible Retrieval**: Archive, minutes to hours (1-5 hours)
7. **S3 Glacier Deep Archive**: Lowest cost, 12+ hours retrieval

**Versioning**:
- Keep multiple versions of objects
- Enabled at bucket level
- Protects from accidental deletion (creates delete marker, not deleted)
- Restores previous versions
- Once enabled, can only suspend (not disable)

**Lifecycle Policies**:
- Automate transitioning objects to different storage classes
- Automate expiration/deletion
- Examples:
  - Transition to IA after 30 days
  - Move to Glacier after 90 days
  - Delete after 365 days

**Encryption**:
1. **SSE-S3** (Server-Side Encryption with S3-managed keys):
   - AWS manages encryption keys
   - AES-256 encryption
   - Header: `x-amz-server-side-encryption: AES256`
2. **SSE-KMS** (Server-Side Encryption with KMS):
   - AWS KMS manages keys
   - Audit trail in CloudTrail
   - Control over key access
   - Header: `x-amz-server-side-encryption: aws:kms`
3. **SSE-C** (Server-Side Encryption with Customer-provided keys):
   - You manage keys
   - AWS encrypts/decrypts, doesn't store key
   - Must provide key with every request
4. **Client-Side Encryption**:
   - Encrypt before uploading
   - You manage encryption/decryption

**Presigned URLs**:
- Temporary access to private objects
- Generated using SDK with credentials
- Includes expiration time
- Inherits permissions of the user who created it
```python
import boto3
s3 = boto3.client('s3')
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'mybucket', 'Key': 'myfile.pdf'},
    ExpiresIn=3600  # 1 hour
)
```

**CORS (Cross-Origin Resource Sharing)**:
- Enable web apps from one domain to access S3 bucket in another domain
- Configure allowed origins, methods, headers
```json
{
  "AllowedOrigins": ["https://example.com"],
  "AllowedMethods": ["GET", "PUT"],
  "AllowedHeaders": ["*"]
}
```

**S3 Event Notifications**:
- Trigger actions when objects created/removed
- Destinations: Lambda, SQS, SNS
- Event types: s3:ObjectCreated:*, s3:ObjectRemoved:*
- Filter by prefix/suffix

**Static Website Hosting**:
- Host static websites (HTML, CSS, JS)
- Enable in bucket properties
- Specify index document (index.html) and error document
- Bucket must be publicly accessible

**Important**: S3 provides **strong read-after-write consistency** for all operations

---

### Amazon DynamoDB

**What It Is**: Fully managed NoSQL database (key-value and document)

**Data Model**:
- **Table**: Collection of items
- **Item**: Collection of attributes (similar to row)
- **Attribute**: Key-value pair (similar to column)
- Schema-less: Each item can have different attributes

**Primary Keys**:
1. **Partition Key Only** (Simple Primary Key):
   - Must be unique
   - DynamoDB uses partition key value to distribute data
   - Example: UserID
2. **Partition Key + Sort Key** (Composite Primary Key):
   - Partition key doesn't have to be unique alone
   - Combination must be unique
   - Items with same partition key stored together, sorted by sort key
   - Example: UserID (partition) + OrderDate (sort)
   - Enables range queries on sort key

**Capacity Modes**:
1. **Provisioned**:
   - Specify Read Capacity Units (RCU) and Write Capacity Units (WCU)
   - Predictable workload
   - Can enable auto-scaling
   - **1 RCU** = 1 strongly consistent read of 4 KB per second
     - OR 2 eventually consistent reads of 4 KB per second
   - **1 WCU** = 1 write of 1 KB per second
2. **On-Demand**:
   - Pay per request
   - Unpredictable workload
   - No capacity planning
   - More expensive per request

**Read Consistency**:
- **Eventually Consistent** (default): Might not reflect recent write
- **Strongly Consistent**: Always reflects recent write (uses more RCU)

**Indexes**:
1. **Global Secondary Index (GSI)**:
   - Different partition key and/or sort key from table
   - Query on non-primary-key attributes
   - Has own RCU/WCU (provisioned mode)
   - Can create after table creation
   - Eventually consistent reads only
   - Example: Table key is UserID, GSI key is Email
2. **Local Secondary Index (LSI)**:
   - Same partition key as table, different sort key
   - Must create at table creation time
   - Shares RCU/WCU with table
   - Supports both eventually and strongly consistent reads
   - Max 5 LSIs per table

**DynamoDB Streams**:
- Ordered stream of item-level modifications (create, update, delete)
- Retention: 24 hours
- Trigger Lambda functions on changes
- View types:
  - KEYS_ONLY: Only key attributes
  - NEW_IMAGE: Entire item after modification
  - OLD_IMAGE: Entire item before modification
  - NEW_AND_OLD_IMAGES: Both new and old

**Advanced Features**:
- **TTL (Time To Live)**:
  - Automatically delete expired items
  - Specify attribute name containing Unix timestamp
  - Deletion happens within 48 hours of expiration
  - Free (doesn't use WCUs)
- **Conditional Writes**:
  - Write only if condition is met
  - Prevents race conditions
  - Example: Update only if version number matches
- **Transactions**:
  - ACID transactions across multiple items/tables
  - All-or-nothing operations
  - Max 100 items or 4 MB per transaction
- **Batch Operations**:
  - **BatchGetItem**: Retrieve up to 100 items, 16 MB
  - **BatchWriteItem**: Put/delete up to 25 items, 16 MB
- **DAX (DynamoDB Accelerator)**:
  - In-memory cache for DynamoDB
  - Microsecond latency
  - Compatible with existing DynamoDB API calls
  - Cluster of nodes
  - Use for read-heavy workloads

**PartiQL Support**:
- SQL-compatible query language for DynamoDB
- SELECT, INSERT, UPDATE, DELETE statements

**Best Practices**:
- Design partition key for even distribution (avoid hot partitions)
- Use sparse indexes (only items with attribute are indexed)
- Compress large attribute values
- Use batch operations for efficiency

---

### Amazon RDS (Relational Database Service)

**What It Is**: Managed relational database service

**Supported Engines**:
- MySQL, PostgreSQL, MariaDB
- Oracle, Microsoft SQL Server
- Amazon Aurora (MySQL and PostgreSQL compatible)

**Benefits**:
- Automated backups (point-in-time recovery)
- Automated patching
- Automatic failover (Multi-AZ)
- Vertical and horizontal scaling

**Read Replicas**:
- Asynchronous replication
- Scale read workloads (up to 15 read replicas)
- Can be in different regions
- Can be promoted to standalone database
- Use cases: Reporting, analytics (offload reads from primary)

**Multi-AZ Deployment**:
- Synchronous replication to standby instance in different AZ
- Automatic failover on failure
- Increases availability, not performance
- Used for disaster recovery
- DNS name stays same (RDS handles failover)

**Security**:
- **Encryption at rest**: KMS encryption (must enable at creation)
- **Encryption in transit**: SSL/TLS connections
- **IAM Database Authentication**: Use IAM roles instead of passwords
  - For MySQL and PostgreSQL
  - Generates auth token valid for 15 minutes
- **Security Groups**: Control network access

---

### Amazon Aurora

**What It Is**: AWS-built, cloud-native relational database

**Features**:
- MySQL and PostgreSQL compatible
- **5x faster** than MySQL, **3x faster** than PostgreSQL
- Storage automatically grows (10 GB to 128 TB)
- Up to 15 read replicas
- Replication lag < 10 ms

**High Availability**:
- 6 copies of data across 3 AZs
- Self-healing (corrupt blocks auto-repaired)
- Automated backups to S3

**Aurora Serverless**:
- Auto-scaling database
- Pay per second
- Good for infrequent, intermittent, unpredictable workloads
- No capacity planning

**Aurora Global Database**:
- Cross-region replication (< 1 second lag)
- Up to 5 secondary regions
- Recovery Time Objective (RTO) < 1 minute

---

### Amazon ElastiCache

**What It Is**: Managed in-memory caching service

**Engines**:
1. **Redis**:
   - Advanced data structures (lists, sets, sorted sets, hashes)
   - Persistence (snapshots, AOF)
   - Replication and Multi-AZ
   - Pub/Sub messaging
   - Transactions
   - Backup and restore
   - Cluster mode (sharding)
2. **Memcached**:
   - Simple key-value store
   - Multi-threaded
   - No persistence
   - Easy horizontal scaling (add nodes)
   - No replication

**Use Cases**:
- Database query result caching
- Session storage
- Reduce database load
- Leaderboards (Redis sorted sets)
- Real-time analytics

**Caching Strategies**:
- **Lazy Loading** (Cache-Aside): Load data into cache only when needed
  - Read from cache → If miss, read from DB and write to cache
- **Write-Through**: Write data to cache whenever DB is written
  - Always in sync, but cache can have unnecessary data

---

## API & Integration Services

### Amazon API Gateway

**What It Is**: Create, publish, maintain, secure REST and WebSocket APIs

**API Types**:
1. **REST API**: HTTP API with full features
2. **HTTP API**: Simpler, cheaper, lower latency
3. **WebSocket API**: Two-way communication

**Integration Types**:
1. **Lambda Function**: Invoke Lambda
2. **HTTP**: Forward to HTTP endpoint
3. **AWS Service**: Invoke AWS service (S3, DynamoDB, etc.)
4. **Mock**: Return response without backend
5. **VPC Link**: Access resources in VPC

**Request/Response Transformation**:
- Use mapping templates (VTL - Velocity Template Language)
- Transform request/response format
- Example: Convert query parameters to JSON

**Authentication & Authorization**:
1. **IAM**: AWS credentials (Sig v4 signing)
   - Good for AWS services, internal APIs
2. **Cognito User Pools**: User authentication
   - User sign-up, sign-in
   - Returns JWT token
3. **Lambda Authorizers** (Custom Authorizers):
   - Custom authentication logic
   - Token-based or request parameter-based
   - Returns IAM policy

**Throttling**:
- Default: 10,000 requests per second (RPS), 5,000 burst
- Prevents API abuse
- Returns 429 Too Many Requests
- Can set per method or per stage

**Caching**:
- Cache responses to reduce backend calls
- TTL: 0 to 3600 seconds (default 300)
- Cached by method and parameters
- Can require authorization for cache key

**Stages**:
- Deployment environments (dev, test, prod)
- Each stage has own URL
- Stage variables: Environment-specific configuration
  - Example: Lambda function ARN per stage

**CORS (Cross-Origin Resource Sharing)**:
- Enable for browser-based apps calling API from different domain
- Configure allowed origins, methods, headers

**Usage Plans & API Keys**:
- Throttle and quota limits per API key
- Monetize APIs
- Track usage per customer

---

### Amazon SQS (Simple Queue Service)

**What It Is**: Fully managed message queue service

**Queue Types**:
1. **Standard Queue**:
   - Unlimited throughput
   - At-least-once delivery (duplicates possible)
   - Best-effort ordering (not guaranteed)
   - Use when throughput is critical, order doesn't matter
2. **FIFO Queue**:
   - Exactly-once processing (no duplicates)
   - Strict ordering (First-In-First-Out)
   - 300 TPS default (3000 with batching)
   - Queue name must end with .fifo
   - Use when order matters

**Key Concepts**:
- **Visibility Timeout**: Time message is invisible after being received
  - Default 30 seconds, max 12 hours
  - Prevents other consumers from processing same message
  - If processing takes longer, extend visibility timeout
  - If not deleted within timeout, message becomes visible again
- **Message Retention**: 1 minute to 14 days (default 4 days)
- **Long Polling**: Wait for messages to arrive (1-20 seconds)
  - Reduces API calls
  - Reduces costs
  - Preferred over short polling
- **Short Polling**: Returns immediately (even if no messages)
- **Dead Letter Queue (DLQ)**:
  - Send messages that fail processing after max retries
  - Analyze failed messages
  - Set maximum receives count

**Message Attributes**:
- Metadata in key-value pairs
- Don't count towards 256 KB message size limit

**Use Cases**:
- Decouple components
- Asynchronous processing
- Load leveling (smooth out traffic spikes)
- Buffer requests

---

### Amazon SNS (Simple Notification Service)

**What It Is**: Pub/Sub messaging service

**Components**:
- **Topic**: Communication channel
- **Publisher**: Sends messages to topic
- **Subscriber**: Receives messages from topic

**Supported Subscribers**:
- SQS queues
- Lambda functions
- HTTP/HTTPS endpoints
- Email, SMS
- Mobile push notifications (iOS, Android)

**Message Filtering**:
- Filter messages based on attributes
- Subscribers receive only relevant messages
- JSON filter policy

**Fan-Out Pattern**:
- SNS topic → Multiple SQS queues
- Process same message in different ways
- Decouple and parallelize processing
- Example: S3 event → SNS → Queue for processing, Queue for logging

**FIFO Topics**:
- Strict ordering and exactly-once delivery
- Must subscribe FIFO queues only
- 300 TPS per topic

---

### Amazon EventBridge (formerly CloudWatch Events)

**What It Is**: Serverless event bus service

**Components**:
- **Event Bus**: Receives events
  - Default event bus (AWS services)
  - Custom event buses
  - Partner event buses (SaaS integrations)
- **Event**: JSON object describing state change
- **Rule**: Match events and route to targets
- **Target**: Where event is sent (Lambda, SQS, SNS, Step Functions, etc.)

**Event Pattern**:
- Filter events to match specific criteria
- JSON pattern matching
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```

**Scheduled Events**:
- Cron or rate expressions
- Example: Run Lambda every 5 minutes

**Use Cases**:
- React to AWS service events
- Schedule automated tasks
- SaaS application integrations

---

### AWS Step Functions

**What It Is**: Orchestrate workflows using state machines

**State Types**:
1. **Task**: Do work (invoke Lambda, run ECS task, publish to SNS, etc.)
2. **Choice**: Branch based on conditions
3. **Parallel**: Execute branches in parallel
4. **Wait**: Delay for specified time
5. **Succeed**: Successful termination
6. **Fail**: Failed termination
7. **Pass**: Pass input to output (transform data)
8. **Map**: Iterate over array of items

**Error Handling**:
- **Retry**: Retry failed tasks
  - Configure max attempts, interval, backoff rate
- **Catch**: Handle errors and transition to different state
  - Error types: States.ALL, States.Timeout, specific errors

**Workflow Types**:
1. **Standard**: Long-running, durable (up to 1 year)
   - Exactly-once execution
   - Full execution history
2. **Express**: Short-lived, high-volume (up to 5 minutes)
   - At-least-once execution
   - Cheaper for high-volume

**Use Cases**:
- Coordinate microservices
- Implement complex workflows
- ETL processes
- Error handling and retries

---

## Security & Identity

### AWS IAM (Identity and Access Management)

**What It Is**: Manage access to AWS services and resources

**Components**:
1. **Users**: Individual people or applications
   - Long-term credentials
2. **Groups**: Collection of users
   - Attach policies to groups
3. **Roles**: Temporary credentials for services or users
   - No password
   - Assumed by services, users, or external identities
4. **Policies**: JSON documents defining permissions

**Policy Structure**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",  // or "Deny"
      "Action": "s3:GetObject",  // or ["s3:GetObject", "s3:PutObject"]
      "Resource": "arn:aws:s3:::mybucket/*",
      "Condition": {
        "IpAddress": {"aws:SourceIp": "192.0.2.0/24"}
      }
    }
  ]
}
```

**Policy Types**:
1. **AWS Managed Policies**: Created and maintained by AWS
2. **Customer Managed Policies**: Created by you, reusable
3. **Inline Policies**: Embedded directly in user/group/role

**Permission Boundaries**:
- Maximum permissions a policy can grant
- Used to delegate policy creation
- Does not grant permissions, only limits them

**IAM Roles**:
- **For EC2**: Instance profile
- **For Lambda**: Execution role
- **For Cross-Account**: Assume role from another account
- **For Federated Users**: Assume role with external identity

**Assume Role**:
- Temporarily assume permissions of a role
- Use STS (Security Token Service) AssumeRole API
- Returns temporary credentials (access key, secret key, session token)
```python
sts = boto3.client('sts')
response = sts.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/MyRole',
    RoleSessionName='session1'
)
credentials = response['Credentials']
```

**Best Practices**:
- **Least Privilege**: Grant only necessary permissions
- **Use Roles** instead of long-term credentials
- **Enable MFA** for privileged users
- **Rotate credentials** regularly
- **Use policy conditions** for additional security

---

### Amazon Cognito

**What It Is**: User authentication and authorization service

**Components**:
1. **User Pools**: User directory
   - Sign-up and sign-in
   - User profile management
   - MFA, password policies
   - Returns JWT tokens (ID token, access token, refresh token)
   - Social login (Google, Facebook, Amazon)
   - SAML and OIDC identity providers

2. **Identity Pools** (Federated Identities):
   - Grant AWS credentials to users
   - Authenticated (logged in) or unauthenticated (guest) users
   - Exchange tokens for temporary AWS credentials
   - Access AWS services (S3, DynamoDB)

**Integration with API Gateway**:
- User authenticates with User Pool → Gets JWT token
- API Gateway validates JWT token
- If valid, allows access to API

**Cognito Sync** (deprecated):
- Synchronize user data across devices
- Replaced by AppSync

---

### AWS KMS (Key Management Service)

**What It Is**: Managed encryption key service

**Key Types**:
1. **AWS Managed Keys**: Created by AWS services (free)
   - Automatic rotation every year
   - Example: aws/s3, aws/rds
2. **Customer Managed Keys (CMK)**: You create and manage
   - $1/month per key
   - Optional automatic rotation (every year)
   - Full control (policies, grants, enable/disable)
3. **AWS Owned Keys**: AWS uses, you don't see

**Operations**:
- **Encrypt**: Encrypt data (max 4 KB)
- **Decrypt**: Decrypt data
- **GenerateDataKey**: Create data key for envelope encryption
- **ReEncrypt**: Decrypt with one key, encrypt with another (server-side)

**Envelope Encryption**:
- Encrypt data with data key
- Encrypt data key with master key (CMK)
- Store encrypted data + encrypted data key together
- To decrypt: Decrypt data key with CMK, then decrypt data with data key
- Allows encrypting large data (CMK Encrypt limited to 4 KB)

**Key Policies**:
- Resource-based policy for CMK
- Defines who can use and manage key
- Default: Root user has full access

**Grants**:
- Programmatic delegation of key use
- Temporary, limited permissions

---

### AWS Secrets Manager

**What It Is**: Store and rotate secrets (passwords, API keys)

**Features**:
- Automatic rotation of secrets
  - Built-in rotation for RDS, Redshift, DocumentDB
  - Custom rotation with Lambda
- Encryption with KMS
- Fine-grained access control (IAM)
- Versioning (stage labels: AWSCURRENT, AWSPREVIOUS)

**vs Parameter Store**:
- Secrets Manager: Paid, automatic rotation, secrets-focused
- Parameter Store: Free tier, manual rotation (with Lambda), config-focused

---

### AWS Systems Manager Parameter Store

**What It Is**: Store configuration data and secrets

**Parameter Types**:
1. **String**: Plain text
2. **StringList**: Comma-separated values
3. **SecureString**: Encrypted with KMS

**Hierarchy**:
- Organize parameters in paths
- Example: /dev/database/password, /prod/database/password
- Allows bulk retrieval by path

**Features**:
- Free (standard parameters, up to 10,000)
- Advanced parameters: Higher throughput, larger size ($0.05/month)
- Integration with other AWS services
- Change notifications (EventBridge)

---

## Developer Tools & CI/CD

### AWS CodeCommit

**What It Is**: Managed Git repository service

**Features**:
- Fully managed, highly available
- Encrypted at rest and in transit
- IAM integration for access control
- Triggers (Lambda, SNS) on repository events

**Authentication**:
- HTTPS: AWS CLI credential helper or Git credentials
- SSH: SSH keys

---

### AWS CodeBuild

**What It Is**: Fully managed build service

**buildspec.yml**:
- Build instructions file
- Placed in root of source code
```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 14
    commands:
      - npm install
  pre_build:
    commands:
      - npm run test
  build:
    commands:
      - npm run build
  post_build:
    commands:
      - echo "Build complete"
artifacts:
  files:
    - '**/*'
  base-directory: dist
```

**Phases**:
1. **install**: Install runtime and dependencies
2. **pre_build**: Commands before build (tests, linting)
3. **build**: Actual build commands
4. **post_build**: Commands after build (package, upload)

**Environment**:
- Managed Docker images (Node.js, Python, Java, etc.)
- Custom Docker images
- Environment variables (plaintext, Parameter Store, Secrets Manager)

**Artifacts**:
- Build outputs
- Stored in S3
- Can be used by CodeDeploy or CodePipeline

**Logs**:
- CloudWatch Logs (build logs)
- S3 (archive logs)

---

### AWS CodeDeploy

**What It Is**: Automated deployment service

**Deployment Targets**:
- EC2 instances
- On-premises servers
- Lambda functions
- ECS services

**Deployment Strategies** (for EC2/On-premises):
1. **In-Place** (Rolling):
   - Deploy to existing instances
   - Instances briefly offline during deployment
   - Can specify deployment groups
2. **Blue/Green**:
   - Deploy to new instances
   - Traffic shifted from old to new
   - Easy rollback
   - More expensive (running two environments)

**Lambda Deployment**:
- **Canary**: Shift traffic in two increments
  - Example: Canary10Percent5Minutes (10% for 5 min, then 100%)
- **Linear**: Shift traffic gradually at regular intervals
  - Example: Linear10PercentEvery1Minute
- **All-at-once**: Immediate shift

**appspec.yml**:
- Deployment instructions
- For EC2/On-premises:
  ```yaml
  version: 0.0
  os: linux
  files:
    - source: /
      destination: /var/www/html
  hooks:
    BeforeInstall:
      - location: scripts/install_dependencies.sh
    AfterInstall:
      - location: scripts/start_server.sh
  ```
- For Lambda:
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
  ```

**Lifecycle Hooks** (EC2):
- **BeforeInstall, AfterInstall**
- **ApplicationStart, ApplicationStop**
- **ValidateService**

---

### AWS CodePipeline

**What It Is**: CI/CD orchestration service

**Structure**:
- **Pipeline**: Workflow
- **Stage**: Major phase (source, build, test, deploy)
- **Action**: Task within stage

**Common Pipeline**:
1. **Source**: CodeCommit, GitHub, S3
2. **Build**: CodeBuild
3. **Test**: CodeBuild, third-party tools
4. **Deploy**: CodeDeploy, Elastic Beanstalk, ECS, S3

**Triggers**:
- Automatic on source change
- Manual approval actions
- CloudWatch Events for scheduled pipelines

**Artifacts**:
- Outputs from one stage passed to next stage
- Stored in S3

---

### AWS CodeArtifact

**What It Is**: Managed artifact repository

**Supports**:
- npm, Maven, Gradle, pip, NuGet

**Features**:
- Store and retrieve packages
- Proxy public repositories (npm, PyPI, Maven Central)
- Share packages across accounts

---

### AWS CloudFormation

**What It Is**: Infrastructure as Code (IaC) service

**Templates**:
- JSON or YAML
- Define AWS resources
```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-unique-bucket-name
```

**Components**:
- **Resources**: AWS resources to create (required)
- **Parameters**: Input values
- **Mappings**: Static variables (key-value)
- **Conditions**: Control resource creation
- **Outputs**: Values to export

**Stack**:
- Collection of resources managed as single unit
- Create, update, delete stack

**Change Sets**:
- Preview changes before applying
- See what will be added, modified, deleted

**Drift Detection**:
- Detect manual changes to resources
- Compare actual config vs template

**Rollback**:
- Automatic rollback on stack creation failure
- Rollback on update failure (configurable)

---

### AWS SAM (Serverless Application Model)

**What It Is**: Framework for building serverless applications

**SAM Template**:
- Extension of CloudFormation
- Simplified syntax for serverless resources
```yaml
Transform: AWS::Serverless-2016-10-31
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: python3.9
      CodeUri: ./src
      Events:
        MyApi:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

**SAM CLI**:
- `sam init`: Create new project
- `sam build`: Build application
- `sam local`: Test locally
- `sam deploy`: Deploy to AWS

**Simplified Resources**:
- `AWS::Serverless::Function` → Lambda + IAM + Event mappings
- `AWS::Serverless::Api` → API Gateway
- `AWS::Serverless::SimpleTable` → DynamoDB table

---

## Monitoring & Troubleshooting

### Amazon CloudWatch

**What It Is**: Monitoring and observability service

**Metrics**:
- Time-ordered data points (CPU usage, request count, etc.)
- **Namespaces**: Container for metrics (AWS/EC2, AWS/Lambda)
- **Dimensions**: Name-value pairs to identify metric (InstanceId=i-123)
- **Standard Metrics**: Provided by AWS services (free)
  - EC2: CPU, Network, Disk (no RAM - requires custom metric)
  - Lambda: Invocations, Duration, Errors, Throttles
- **Custom Metrics**: You publish
  - Use PutMetricData API
  - Standard resolution: 1 minute
  - High resolution: 1 second
- **Detailed Monitoring**: 1-minute intervals (EC2, RDS) - extra cost

**Logs**:
- **Log Groups**: Container for log streams (usually one per application)
- **Log Streams**: Sequence of log events (usually one per instance)
- **Metric Filters**: Extract metrics from logs
  - Example: Count error messages
- **Log Insights**: Query logs with SQL-like syntax
- **Retention**: 1 day to 10 years (default: never expire)
- **Export**: Export to S3 (can take 12 hours)
- **Subscriptions**: Real-time delivery to Lambda, Kinesis, Elasticsearch

**Alarms**:
- Watch metrics and trigger actions
- States: OK, ALARM, INSUFFICIENT_DATA
- **Actions**:
  - SNS notification
  - Auto Scaling action
  - EC2 action (stop, terminate, reboot)
- **Period**: Evaluation period (1 minute, 5 minutes, etc.)
- **Threshold**: Value to compare
- Example: Alarm if CPU > 80% for 5 minutes

**Dashboards**:
- Visual representation of metrics
- Cross-region, cross-account

**EventBridge** (formerly CloudWatch Events):
- Covered in Integration Services section

---

### AWS X-Ray

**What It Is**: Distributed tracing service

**Concepts**:
- **Trace**: End-to-end request journey
- **Segment**: Work done by single service (Lambda function, EC2 instance)
- **Subsegment**: Granular work within segment (DB call, HTTP request)
- **Annotations**: Key-value pairs indexed for searching
  - Searchable, max 50
- **Metadata**: Key-value pairs not indexed
  - Additional context, not searchable
- **Sampling**: Control amount of data recorded
  - Reduce cost
  - Default: 1 request/second + 5% of additional requests

**Service Map**:
- Visual representation of application architecture
- Shows latency, errors, traffic volume

**Integration**:
- **Lambda**: Enable active tracing in Lambda config
  - X-Ray SDK automatically included
- **API Gateway**: Enable X-Ray tracing
- **EC2/ECS**: Install X-Ray daemon
  - Application uses X-Ray SDK to send data to daemon
  - Daemon forwards to X-Ray service

**X-Ray SDK**:
- Available for Java, Node.js, Python, Go, .NET
- Instrument code
- Capture AWS SDK calls, HTTP requests, SQL queries
```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('my_function')
def my_function():
    # Function code
    pass
```

**IAM Permissions**:
- `xray:PutTraceSegments` - Write segment documents
- `xray:PutTelemetryRecords` - Write telemetry data

---

### AWS CloudTrail

**What It Is**: AWS API call logging (audit trail)

**What It Logs**:
- Who made the request (user, role)
- When (timestamp)
- What (service, action, parameters)
- From where (source IP, user agent)
- Response

**Event Types**:
1. **Management Events**: Control plane operations (create, delete, modify resources)
   - Enabled by default
2. **Data Events**: Data plane operations (S3 GetObject, Lambda Invoke)
   - Disabled by default (high volume, extra cost)
3. **Insights Events**: Detect unusual activity
   - Identify anomalies (API call rate spikes)

**Delivery**:
- CloudWatch Logs (real-time monitoring, alarms)
- S3 (long-term storage, analysis)
- Retention: 90 days in Event history (free)
- Trail: Continuous delivery to S3 (as long as configured)

**Use Cases**:
- Security analysis
- Compliance auditing
- Troubleshooting
- Resource change tracking

---

### Debugging Strategies

**Lambda Debugging**:
- CloudWatch Logs: Print statements, errors
- X-Ray: Trace execution, identify bottlenecks
- **Common Issues**:
  - Timeout: Increase timeout or optimize code
  - Out of memory: Increase memory allocation
  - Throttling: Increase concurrency limit or use reserved concurrency
  - Permissions: Check execution role has necessary permissions
  - Cold starts: Use provisioned concurrency

**Application Errors**:
- Check CloudWatch Logs for error messages
- Use X-Ray to trace request flow
- Check CloudTrail for API call failures
- **Common Issues**:
  - 403 Forbidden: IAM permissions issue
  - 404 Not Found: Resource doesn't exist or incorrect path
  - 429 Too Many Requests: Throttling (API Gateway, Lambda, DynamoDB)
  - 500 Internal Server Error: Application error, check logs
  - 503 Service Unavailable: AWS service issue or over capacity

**SDK Error Handling**:
- Retry with exponential backoff
  - Most SDKs have built-in retry logic
- Throttling errors (429): Use backoff, reduce request rate
- Client errors (4xx): Fix request (authentication, validation)
- Server errors (5xx): Retry, might be temporary

---

## AWS SDK and CLI

### AWS SDK Basics

**SDKs Available**:
- Python (Boto3)
- JavaScript (AWS SDK for JavaScript)
- Java, .NET, Go, Ruby, PHP, C++

**Example** (Python Boto3):
```python
import boto3

# Create client
s3 = boto3.client('s3')

# Upload file
s3.upload_file('myfile.txt', 'mybucket', 'myfile.txt')

# Download file
s3.download_file('mybucket', 'myfile.txt', 'downloaded.txt')

# List objects
response = s3.list_objects_v2(Bucket='mybucket')
for obj in response['Contents']:
    print(obj['Key'])
```

---

### Credential Configuration

**Credential Chain** (order of precedence):
1. **Code** (not recommended):
   ```python
   s3 = boto3.client('s3', aws_access_key_id='...', aws_secret_access_key='...')
   ```
2. **Environment Variables**:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_SESSION_TOKEN` (for temporary credentials)
3. **Credentials File** (`~/.aws/credentials`):
   ```
   [default]
   aws_access_key_id = ...
   aws_secret_access_key = ...
   ```
4. **Config File** (`~/.aws/config`):
   ```
   [default]
   region = us-east-1
   ```
5. **IAM Role** (for EC2, Lambda, ECS):
   - Preferred method
   - Automatically rotated
   - Retrieved from instance metadata

**Best Practice**: Use IAM roles for EC2/Lambda/ECS, never hardcode credentials

---

### Error Handling and Retries

**Error Types**:
- **Client Errors (4xx)**: Your fault (invalid parameters, permissions)
  - Don't retry (fix the request)
- **Server Errors (5xx)**: AWS fault or temporary issue
  - Retry with backoff

**Exponential Backoff**:
- Retry with increasing wait time
- Example: Wait 1s, 2s, 4s, 8s, 16s...
- Add jitter (randomness) to avoid thundering herd
```python
import time
import random

for retry in range(5):
    try:
        # Make API call
        response = s3.list_buckets()
        break
    except Exception as e:
        if retry == 4:
            raise
        wait = (2 ** retry) + random.uniform(0, 1)
        time.sleep(wait)
```

**SDK Built-in Retry**:
- Most SDKs automatically retry transient errors
- Configurable (max retries, retry modes)

---

### Pagination

**Problem**: Some API calls return large result sets (S3 ListObjects, DynamoDB Scan)
- Results truncated, need multiple calls

**Manual Pagination**:
```python
paginator = s3.get_paginator('list_objects_v2')
for page in paginator.paginate(Bucket='mybucket'):
    for obj in page['Contents']:
        print(obj['Key'])
```

**DynamoDB Example**:
```python
response = dynamodb.scan(TableName='MyTable')
items = response['Items']

while 'LastEvaluatedKey' in response:
    response = dynamodb.scan(
        TableName='MyTable',
        ExclusiveStartKey=response['LastEvaluatedKey']
    )
    items.extend(response['Items'])
```

---

### AWS CLI

**Common Commands**:
```bash
# S3
aws s3 ls s3://mybucket
aws s3 cp myfile.txt s3://mybucket/
aws s3 sync ./localdir s3://mybucket/dir

# DynamoDB
aws dynamodb put-item --table-name MyTable --item '{"id": {"S": "123"}}'
aws dynamodb get-item --table-name MyTable --key '{"id": {"S": "123"}}'
aws dynamodb scan --table-name MyTable

# Lambda
aws lambda invoke --function-name MyFunction output.json
aws lambda list-functions

# IAM
aws iam list-users
aws iam create-user --user-name john
```

**Profiles**:
- Multiple credential sets
```bash
aws s3 ls --profile myprofile
```
- Config in `~/.aws/credentials`:
```
[myprofile]
aws_access_key_id = ...
aws_secret_access_key = ...
```

**Output Formats**:
- JSON (default), YAML, text, table
```bash
aws s3 ls --output table
```

**Dry Run**:
- Test command without executing
```bash
aws ec2 run-instances --dry-run ...
```

---

This covers all major concepts for the AWS Developer Associate exam. Use the checkboxes in README.md to track your progress!
