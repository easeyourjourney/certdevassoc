# AWS Certified Developer - Associate Study Guide

Complete study guide with **concept coverage** for exam topics and **scenario-based learning** for practical understanding.

## Exam Information

- **Exam Code**: DVA-C02
- **Duration**: 130 minutes
- **Questions**: 65 (multiple choice and multiple response)
- **Passing Score**: ~720/1000
- **Cost**: $150 USD
- **Valid**: 3 years

---

## Part 1: Concepts Progress Tracker

Track all exam concepts to ensure complete coverage.

### Compute Services
- [x] Lambda - Basics (handler, event, context, execution role)
- [x] Lambda - Configuration (timeout, memory, environment variables)
- [x] Lambda - Advanced (layers, versions, aliases, concurrency)
- [ ] Lambda - VPC and networking
- [ ] EC2 - Basics (instance types, user data, metadata)
- [ ] EC2 - IAM roles for EC2
- [ ] ECS - Task definitions and services
- [ ] ECS - Fargate vs EC2 launch type
- [ ] Elastic Beanstalk - Deployment strategies
- [ ] Elastic Beanstalk - Configuration (.ebextensions)

### Storage & Databases
- [ ] S3 - Basics (buckets, objects, storage classes)
- [ ] S3 - Versioning and lifecycle
- [ ] S3 - Encryption (SSE-S3, SSE-KMS, SSE-C)
- [ ] S3 - Presigned URLs
- [ ] S3 - CORS and static website hosting
- [ ] S3 - Event notifications
- [ ] DynamoDB - Data model (tables, items, attributes)
- [ ] DynamoDB - Primary keys (partition, partition+sort)
- [ ] DynamoDB - Capacity modes (provisioned vs on-demand)
- [ ] DynamoDB - Indexes (GSI vs LSI)
- [ ] DynamoDB - Streams and triggers
- [ ] DynamoDB - TTL, transactions, conditional writes
- [ ] DynamoDB - DAX (caching)
- [ ] RDS - Basics and engine types
- [ ] RDS - Read replicas vs Multi-AZ
- [ ] Aurora - Features and Serverless
- [ ] ElastiCache - Redis vs Memcached

### API & Integration
- [ ] API Gateway - REST API basics
- [ ] API Gateway - Integration types (Lambda, HTTP, AWS services)
- [ ] API Gateway - Request/response transformation
- [ ] API Gateway - Authentication (Cognito, Lambda authorizers, IAM)
- [ ] API Gateway - Throttling and caching
- [ ] API Gateway - CORS and stages
- [ ] SQS - Standard vs FIFO queues
- [ ] SQS - Visibility timeout and dead letter queues
- [ ] SQS - Long polling vs short polling
- [ ] SNS - Topics and subscriptions
- [ ] SNS - Fan-out pattern
- [ ] EventBridge - Event buses and rules
- [ ] Step Functions - State machines and states
- [ ] Step Functions - Error handling

### Security & Identity
- [ ] IAM - Users, groups, roles, policies
- [ ] IAM - Policy structure (Effect, Action, Resource, Condition)
- [ ] IAM - Assume role and cross-account access
- [ ] IAM - Permission boundaries
- [ ] Cognito - User Pools (authentication)
- [ ] Cognito - Identity Pools (AWS credentials)
- [ ] Cognito - Integration with API Gateway
- [ ] KMS - Encryption and key management
- [ ] KMS - Envelope encryption
- [ ] Secrets Manager - Secret storage and rotation
- [ ] Parameter Store - Hierarchical storage
- [ ] Security best practices

### Developer Tools & CI/CD
- [ ] CodeCommit - Git repository basics
- [ ] CodeBuild - buildspec.yml structure
- [ ] CodeBuild - Environment variables and artifacts
- [ ] CodeDeploy - Deployment strategies (in-place, blue/green)
- [ ] CodeDeploy - appspec.yml
- [ ] CodeDeploy - Deployment to Lambda, EC2, ECS
- [ ] CodePipeline - Pipeline stages and actions
- [ ] CodePipeline - Source, build, test, deploy flow
- [ ] CodeArtifact - Package management
- [ ] CloudFormation - Templates (JSON/YAML)
- [ ] CloudFormation - Stacks, change sets, drift detection
- [ ] SAM - Serverless Application Model
- [ ] SAM - Template structure and deployment

### Monitoring & Troubleshooting
- [ ] CloudWatch - Metrics (standard vs custom)
- [ ] CloudWatch - Logs and log groups
- [ ] CloudWatch - Alarms and actions
- [ ] CloudWatch - Dashboards
- [ ] CloudWatch - Log Insights queries
- [ ] CloudTrail - API logging and audit
- [ ] X-Ray - Distributed tracing
- [ ] X-Ray - Segments, subsegments, annotations
- [ ] X-Ray - Service map and trace analysis
- [ ] X-Ray - Integration with Lambda and API Gateway
- [ ] Debugging Lambda functions
- [ ] Debugging application errors

### SDK & Development
- [ ] AWS SDK - Basics (Boto3, JavaScript SDK)
- [ ] SDK - Credential configuration
- [ ] SDK - Error handling and retries
- [ ] SDK - Exponential backoff
- [ ] SDK - Pagination for large result sets
- [ ] AWS CLI - Common commands
- [ ] CLI - Profiles and configuration
- [ ] Environment variables for configuration

---

## Part 2: Scenarios Progress Tracker

Apply concepts through real-world integration scenarios.

### Core Integration Scenarios
- [ ] **Scenario 1**: Serverless REST API
  - API Gateway + Lambda + DynamoDB + IAM
- [ ] **Scenario 2**: Event-Driven File Processing
  - S3 Events + Lambda + SNS + SQS
- [ ] **Scenario 3**: Asynchronous Workflow
  - Step Functions + Lambda + DynamoDB
- [ ] **Scenario 4**: Containerized Web App
  - ECS Fargate + ECR + ALB + RDS
- [ ] **Scenario 5**: User Authentication System
  - Cognito User Pools + API Gateway + Lambda
- [ ] **Scenario 6**: Real-time Data Processing
  - DynamoDB Streams + Lambda + ElastiCache
- [ ] **Scenario 7**: CI/CD Pipeline
  - CodeCommit + CodeBuild + CodeDeploy + CodePipeline
- [ ] **Scenario 8**: Microservices Architecture
  - Multiple Lambda functions + API Gateway + EventBridge

### Advanced Patterns
- [ ] Caching strategies (CloudFront, API Gateway, DAX, ElastiCache)
- [ ] Error handling patterns (DLQ, retries, circuit breaker)
- [ ] Security patterns (least privilege, encryption, secrets management)
- [ ] Performance optimization patterns
- [ ] Cost optimization strategies

---

## Study Materials

### Main Guides
- **[Concepts Guide](./concepts.md)** - Complete concept coverage organized by service
- **[Scenarios Guide](./scenarios.md)** - Real-world integration patterns
- **[Hands-On Labs](./labs.md)** - Step-by-step practical exercises
- **[Exam Preparation](./exam-prep.md)** - Practice questions and exam tips
- **[Quick Reference](./quick-reference.md)** - Service limits, error codes, CLI commands

---

## How to Use This Guide

### For Comprehensive Coverage (Concepts)
1. Work through the **Concepts Progress Tracker** systematically
2. Study each topic in `concepts.md`
3. Ensure you understand each checkbox item
4. Check off items as you master them

### For Practical Understanding (Scenarios)
1. Work through **Scenarios Progress Tracker**
2. Study how services integrate in `scenarios.md`
3. Do the hands-on labs
4. Build the architectures yourself

### Recommended Approach
- **Alternate**: Study concepts, then practice with scenarios
- **Example**: Study Lambda concepts → Build serverless API scenario
- **Review**: Revisit concepts when working on scenarios

---

## Study Timeline

**Week 1-2**: Compute + Storage Concepts + Scenarios 1-2
**Week 3**: API/Integration Concepts + Scenarios 3-4
**Week 4**: Security + CI/CD Concepts + Scenarios 5-7
**Week 5**: Monitoring Concepts + Scenario 8 + Advanced Patterns
**Week 6**: Hands-on Labs + Practice Exams + Review

---

## Quick Tips

- **Check both trackers** - Ensure you're covering concepts AND understanding integration
- **Hands-on is critical** - Create AWS Free Tier account and build
- **Focus on integration** - Exam tests how services work together
- **Know the limits** - Service quotas are frequently tested
- **Understand errors** - Know common error codes and solutions

Start with `concepts.md` for systematic learning or `scenarios.md` for practical integration!
