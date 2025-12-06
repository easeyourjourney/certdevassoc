# Exam Preparation & Practice Questions

Test your knowledge and prepare for the AWS Certified Developer - Associate exam.

---

## Exam Strategy

### Time Management
- 130 minutes for 65 questions = 2 minutes per question
- Flag difficult questions and return later
- Don't spend more than 3 minutes on any question initially
- Review flagged questions if time permits

### Question Approach
1. **Read carefully**: Every word matters
2. **Identify key requirements**: "Most cost-effective", "Least operational overhead", "Real-time", "Serverless"
3. **Eliminate wrong answers**: Usually can eliminate 2 options immediately
4. **Look for AWS best practices**: Least privilege, automation, serverless when possible
5. **Watch for tricks**: Similar-sounding options, missing requirements

### Common Keywords
- **Most cost-effective**: Serverless, on-demand, auto-scaling, caching
- **Least operational overhead**: Managed services, serverless, automation
- **Real-time**: Streams, WebSockets, EventBridge
- **High availability**: Multi-AZ, multiple regions, auto-scaling
- **Scalable**: Auto-scaling, serverless, load balancing
- **Secure**: Encryption, least privilege, MFA, VPC

---

## Practice Questions by Domain

### Domain 1: Development with AWS Services

**Question 1**:
A developer is building a serverless application that processes uploaded images. The processing takes 20 minutes. Which service should be used?

A) Lambda with 20-minute timeout
B) ECS Fargate task triggered by S3 event
C) EC2 instance with Auto Scaling
D) Elastic Beanstalk worker environment

**Answer**: B

**Explanation**: Lambda has a maximum timeout of 15 minutes, so it cannot handle 20-minute tasks. ECS Fargate is serverless and can run tasks of any duration. C and D work but have operational overhead.

---

**Question 2**:
An application stores user session data. Sessions are accessed frequently but can be recreated if lost. Which solution is most appropriate?

A) DynamoDB with on-demand capacity
B) ElastiCache Redis with persistence
C) ElastiCache Memcached
D) S3 Standard

**Answer**: C

**Explanation**: Memcached is perfect for caching session data that can be recreated. It's simpler and cheaper than Redis when persistence isn't needed. DynamoDB and S3 are more expensive for frequent access patterns.

---

**Question 3**:
A Lambda function needs to process items from a DynamoDB table. The function must process each item exactly once. How should the function be triggered?

A) DynamoDB Streams with Lambda event source mapping
B) CloudWatch Events scheduled every minute to scan DynamoDB
C) S3 event notification when DynamoDB exports to S3
D) SNS notification when items are added

**Answer**: A

**Explanation**: DynamoDB Streams with Lambda provides exactly-once processing guarantees for each table item modification. The event source mapping manages the polling and checkpointing.

---

**Question 4**:
A developer needs to store objects in S3 that will be accessed frequently for 30 days, then rarely for 90 days, then archived for 7 years. What's the most cost-effective solution?

A) S3 Standard with manual transition
B) S3 Intelligent-Tiering
C) S3 Lifecycle policy: Standard → IA after 30 days → Glacier after 120 days
D) S3 One Zone-IA

**Answer**: C

**Explanation**: Lifecycle policies automate transitions based on age, reducing storage costs. Standard for frequent access, IA for infrequent, Glacier for archive. Intelligent-Tiering works but is more expensive for predictable patterns.

---

**Question 5**:
An application needs to query a DynamoDB table by email address, but email is not the primary key. What should be created?

A) Local Secondary Index (LSI) with email as sort key
B) Global Secondary Index (GSI) with email as partition key
C) DynamoDB Stream to sync data to another table
D) Scan the table with FilterExpression

**Answer**: B

**Explanation**: GSI allows querying on non-primary-key attributes. LSI requires the same partition key as the table. Scanning is inefficient. Streams don't help with querying.

---

### Domain 2: Security

**Question 6**:
A Lambda function needs to read database credentials without hardcoding them. The credentials should be rotated automatically. What's the best solution?

A) Environment variables
B) Systems Manager Parameter Store
C) AWS Secrets Manager
D) Store in S3 with encryption

**Answer**: C

**Explanation**: Secrets Manager provides automatic rotation for RDS/Redshift credentials. Parameter Store requires manual rotation (with Lambda). Environment variables and S3 don't support automatic rotation.

---

**Question 7**:
An application running on EC2 needs to access S3. What's the most secure way to provide credentials?

A) Store access keys in EC2 user data
B) Store access keys in environment variables
C) Attach an IAM role to the EC2 instance
D) Store access keys in ~/.aws/credentials file

**Answer**: C

**Explanation**: IAM roles provide temporary, automatically rotated credentials without storing long-term keys. All other options involve storing static credentials which is insecure.

---

**Question 8**:
A developer wants to grant temporary AWS credentials to mobile app users. Which service should be used?

A) Cognito User Pools
B) Cognito Identity Pools
C) IAM users
D) STS AssumeRole

**Answer**: B

**Explanation**: Cognito Identity Pools (Federated Identities) exchange authentication tokens for temporary AWS credentials. User Pools only handle authentication. IAM users are for long-term access. STS is used by Identity Pools behind the scenes.

---

**Question 9**:
An application encrypts data client-side before storing in S3. The encryption keys should be managed by AWS. Which approach should be used?

A) SSE-S3
B) SSE-KMS with GenerateDataKey API
C) SSE-C
D) Client-side encryption with custom keys

**Answer**: B

**Explanation**: For client-side encryption with AWS-managed keys, use KMS GenerateDataKey to get a data encryption key, encrypt data client-side, then store encrypted data + encrypted key. SSE-* are server-side encryption options.

---

**Question 10**:
A Lambda function should only be invoked by a specific API Gateway. How can this be enforced?

A) IAM policy on Lambda
B) Resource-based policy on Lambda allowing API Gateway
C) Lambda authorizer
D) API key requirement

**Answer**: B

**Explanation**: Resource-based policies on Lambda specify which services/accounts can invoke it. When API Gateway is added as a trigger, AWS automatically creates this policy. IAM policies don't apply to service invocations.

---

### Domain 3: Deployment

**Question 11**:
A developer wants to deploy a Lambda function update with 10% of traffic going to the new version for 10 minutes before full deployment. What deployment type should be used?

A) CodeDeploy with Linear10PercentEvery10Minutes
B) CodeDeploy with Canary10Percent10Minutes
C) CodeDeploy with AllAtOnce
D) Lambda alias with weighted routing

**Answer**: B

**Explanation**: Canary deployment shifts specified percentage (10%) and waits for specified time (10 minutes) before shifting remaining traffic. Linear shifts gradually over time at regular intervals. Weighted routing is manual, not automated.

---

**Question 12**:
A CodeBuild project fails because it cannot access a private package repository. Where should repository credentials be stored?

A) Hardcoded in buildspec.yml
B) Environment variables in CodeBuild (plaintext)
C) Systems Manager Parameter Store (SecureString) referenced in CodeBuild
D) S3 bucket

**Answer**: C

**Explanation**: Parameter Store SecureString encrypts credentials. CodeBuild can reference them as environment variables. Never hardcode credentials. Plaintext environment variables are visible in console.

---

**Question 13**:
An application deployed with Elastic Beanstalk must maintain full capacity during deployments. Which deployment policy should be used?

A) All at once
B) Rolling
C) Rolling with additional batch
D) Immutable

**Answer**: C or D

**Explanation**: Both C and D maintain full capacity. "Rolling with additional batch" launches new instances before terminating old ones. "Immutable" is even safer (full new environment) but costs more during deployment. C is "most cost-effective while maintaining capacity".

---

**Question 14**:
A developer needs to deploy the same Lambda function to dev, staging, and prod with different configuration. What's the best approach?

A) Three separate Lambda functions
B) Environment variables + Lambda aliases
C) Three separate AWS accounts
D) Different regions

**Answer**: B

**Explanation**: Use environment variables for configuration and aliases (dev, staging, prod) pointing to different versions. Keeps code DRY. Separate functions create maintenance overhead.

---

**Question 15**:
A CloudFormation stack update failed halfway through. What happened to the resources?

A) All changes are kept, stack is in UPDATE_FAILED state
B) Automatic rollback to previous state
C) Stack is deleted
D) Manual intervention required

**Answer**: B

**Explanation**: CloudFormation automatically rolls back to the previous working state if update fails (unless disabled). This ensures stack is always in a consistent state.

---

### Domain 4: Troubleshooting and Optimization

**Question 16**:
A Lambda function is timing out when connecting to RDS in a VPC. What could be the issue?

A) Lambda doesn't have IAM permissions for RDS
B) Lambda is not configured with VPC settings
C) Lambda is in public subnet without NAT Gateway
D) RDS security group doesn't allow Lambda security group

**Answer**: D (most likely) or B

**Explanation**: For Lambda to access RDS in VPC: (1) Lambda must be configured with VPC subnets/security groups, (2) RDS security group must allow inbound from Lambda security group. If Lambda isn't in VPC at all, it can't connect (B). IAM permissions aren't needed for network connectivity.

---

**Question 17**:
An application is experiencing throttling errors from DynamoDB. What's the most cost-effective solution?

A) Switch to on-demand capacity mode
B) Increase provisioned capacity
C) Enable DynamoDB auto-scaling
D) Implement exponential backoff in the application

**Answer**: C or D

**Explanation**: Best answer depends on whether throttling is predictable. For variable load, auto-scaling (C) automatically adjusts capacity cost-effectively. Application-level retry with backoff (D) is free but doesn't prevent throttling. On-demand (A) works but is more expensive. Manual provisioning (B) requires monitoring.

---

**Question 18**:
A developer needs to identify which downstream service is causing high latency in a microservices application. Which service should be used?

A) CloudWatch Logs
B) CloudWatch Metrics
C) AWS X-Ray
D) CloudTrail

**Answer**: C

**Explanation**: X-Ray provides distributed tracing with service maps showing latency of each service. CloudWatch Logs/Metrics show individual service performance but not the flow. CloudTrail is for API auditing.

---

**Question 19**:
A Lambda function is being throttled (429 errors). What could fix this?

A) Increase memory allocation
B) Increase timeout
C) Request concurrency limit increase
D) Use provisioned concurrency

**Answer**: C or D

**Explanation**: Throttling happens when concurrent executions exceed limit. Request limit increase (C) or reserve/provision concurrency for this function (D). Memory and timeout don't affect concurrency limits.

---

**Question 20**:
An application's CloudWatch dashboard shows high API Gateway 5XX errors but Lambda shows no errors. What's the likely cause?

A) Lambda function is correct, issue is in API Gateway configuration
B) Lambda is timing out
C) API Gateway throttling
D) Lambda insufficient permissions

**Answer**: B

**Explanation**: If Lambda times out, API Gateway returns 504 (gateway timeout) which counts as 5XX. Lambda metrics won't show error because function didn't throw exception, just exceeded timeout. Other 5XX errors would show in Lambda metrics.

---

## Common Exam Traps

### Trap 1: Lambda Timeout vs Memory
**Trap**: "Lambda function is slow, increase memory"
**Truth**: Only increase memory if function needs more memory or you want faster execution (CPU scales with memory). For timeout errors, increase timeout setting.

### Trap 2: S3 Consistency
**Trap**: "Use S3 because it's eventually consistent"
**Truth**: S3 is strongly consistent for all operations now (changed in December 2020). Questions may reference old behavior.

### Trap 3: DynamoDB Indexes
**Trap**: Confusing GSI vs LSI
**Key Differences**:
- LSI: Same partition key, must create at table creation, shares capacity
- GSI: Different partition key, can create anytime, separate capacity

### Trap 4: API Gateway vs ALB for Lambda
**Trap**: "Always use API Gateway for HTTP APIs"
**Truth**: ALB can invoke Lambda and is cheaper for high-volume internal APIs. API Gateway has more features (throttling, caching, usage plans).

### Trap 5: Secrets Manager vs Parameter Store
**Trap**: "Always use Secrets Manager for secrets"
**Truth**: Parameter Store works for most cases and is cheaper. Use Secrets Manager when automatic rotation is needed.

### Trap 6: RDS Read Replicas vs Multi-AZ
**Trap**: Confusing their purposes
**Truth**: Read Replicas scale reads (async replication), Multi-AZ provides HA (sync replication, auto-failover)

### Trap 7: SQS Visibility Timeout
**Trap**: "Message disappears after visibility timeout"
**Truth**: Message becomes visible again in queue if not deleted. Delete message after successful processing.

### Trap 8: Lambda Cold Starts
**Trap**: "Lambda is always slow"
**Truth**: Only first invocation (cold start) is slow. Use provisioned concurrency for consistently low latency.

---

## Key Numbers to Remember

### Lambda
- Max timeout: 15 minutes (900 seconds)
- Max memory: 10,240 MB (10 GB)
- Deployment package: 50 MB (zipped), 250 MB (unzipped)
- /tmp storage: 512 MB to 10 GB
- Concurrent executions: 1,000 per region (default)
- Environment variables: 4 KB total

### DynamoDB
- Item size: 400 KB max
- GSI: 20 per table (default)
- LSI: 5 per table
- Batch operations: 25 items (write), 100 items (read)
- Transaction: 100 items or 4 MB
- 1 RCU = 1 strongly consistent read of 4 KB/sec
- 1 WCU = 1 write of 1 KB/sec

### S3
- Object size: 5 TB max
- Single PUT: 5 GB max
- Multipart upload: Required for >5 GB
- Bucket policies: 20 KB max
- Lifecycle rules: 1,000 per bucket

### API Gateway
- Timeout: 29 seconds
- Payload: 10 MB
- Throttle: 10,000 RPS, 5,000 burst (default)

### SQS
- Message retention: 1 minute to 14 days (default 4 days)
- Message size: 256 KB
- Visibility timeout: 0 seconds to 12 hours (default 30 seconds)
- Batch size: 10 messages
- Long polling: 1-20 seconds

### CloudWatch
- Metric retention: 15 months
- Logs retention: 1 day to 10 years (or indefinite)
- Custom metrics: Standard (1 minute), High-resolution (1 second)

---

## Must-Know AWS CLI Commands

### S3
```bash
aws s3 ls s3://bucket/
aws s3 cp file.txt s3://bucket/
aws s3 sync ./local s3://bucket/
aws s3 presign s3://bucket/file.txt --expires-in 3600
```

### Lambda
```bash
aws lambda invoke --function-name MyFunc output.json
aws lambda update-function-code --function-name MyFunc --zip-file fileb://function.zip
aws lambda list-functions
aws lambda get-function --function-name MyFunc
```

### DynamoDB
```bash
aws dynamodb put-item --table-name Table --item file://item.json
aws dynamodb get-item --table-name Table --key '{"id":{"S":"123"}}'
aws dynamodb scan --table-name Table
aws dynamodb query --table-name Table --key-condition-expression "id = :id"
```

### Cognito
```bash
aws cognito-idp sign-up --client-id abc123 --username user --password pass
aws cognito-idp initiate-auth --client-id abc123 --auth-flow USER_PASSWORD_AUTH
```

### CloudWatch
```bash
aws cloudwatch put-metric-data --namespace MyApp --metric-name Count --value 1
aws logs tail /aws/lambda/MyFunc --follow
```

---

## Final Week Checklist

- [ ] Review all service limits and quotas
- [ ] Practice 3+ full-length practice exams
- [ ] Review incorrect answers and understand why
- [ ] Re-read FAQ for: Lambda, DynamoDB, S3, API Gateway
- [ ] Memorize IAM policy structure
- [ ] Understand all deployment strategies
- [ ] Know all caching strategies
- [ ] Understand error codes (400 vs 500, 429, 503)
- [ ] Review X-Ray, CloudWatch, CloudTrail differences
- [ ] Understand VPC basics (security groups, subnets, NAT)

## Day Before Exam

- [ ] Light review (don't cram)
- [ ] Review this document's key numbers
- [ ] Get good sleep
- [ ] Prepare ID and confirmation number
- [ ] Know testing center location/online setup

## Good Luck!

You've got this! Trust your preparation and remember:
- Read questions carefully
- Eliminate obvious wrong answers
- Think about AWS best practices
- Don't overthink

**Remember**: The exam tests practical knowledge. If you've done the hands-on labs and understand the concepts, you'll do great!
