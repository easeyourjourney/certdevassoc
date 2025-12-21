# AWS Lambda - VPC and Networking: Deep Dive

Comprehensive explanation of Lambda VPC integration, networking concepts, security, and troubleshooting.

---

## 1. Lambda VPC Integration Fundamentals

### What is Lambda VPC Integration?

**VPC Integration** allows your Lambda function to access resources in your Amazon Virtual Private Cloud (VPC), such as RDS databases, ElastiCache clusters, internal APIs, and other private services.

### Theory: The Networking Problem

#### Default Lambda Networking (No VPC)

**By default, Lambda runs in an AWS-managed VPC:**

```
┌─────────────────────────────────────────────┐
│   AWS-Managed VPC (Lambda Service)         │
│                                             │
│   ┌──────────────────────┐                 │
│   │  Lambda Function     │                 │
│   │  (Public Internet    │                 │
│   │   Access Only)       │                 │
│   └──────────────────────┘                 │
│              ↓                              │
└──────────────│──────────────────────────────┘
               ↓
    ┌──────────────────┐
    │  Public Internet │
    └──────────────────┘
         ↓          ↓
    ┌────────┐  ┌────────┐
    │ S3     │  │ DynamoDB│
    │ (Public│  │ (Public │
    │  API)  │  │  API)   │
    └────────┘  └────────┘
```

**What you CAN access:**
- ✅ Public AWS services (S3, DynamoDB, SNS, SQS via public APIs)
- ✅ Public internet endpoints
- ✅ Any publicly accessible API

**What you CANNOT access:**
- ❌ Private RDS database in your VPC
- ❌ ElastiCache cluster in private subnet
- ❌ EC2 instances in private subnet
- ❌ Internal load balancers
- ❌ VPC endpoints (private)

#### With VPC Integration

**Lambda runs with access to your VPC:**

```
┌─────────────────────────────────────────────────────────┐
│   Your VPC (10.0.0.0/16)                                │
│                                                          │
│   ┌──────────────────────────────────────────────┐     │
│   │  Private Subnet (10.0.1.0/24)                │     │
│   │                                               │     │
│   │  ┌──────────────────┐  ┌──────────────────┐ │     │
│   │  │ Lambda Function  │  │ RDS Database     │ │     │
│   │  │ (ENI attached)   │──│ (Private IP)     │ │     │
│   │  └──────────────────┘  └──────────────────┘ │     │
│   │           ↓                                  │     │
│   │  ┌──────────────────┐                       │     │
│   │  │ ElastiCache      │                       │     │
│   │  │ (Private IP)     │                       │     │
│   │  └──────────────────┘                       │     │
│   └──────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**What you CAN access with VPC integration:**
- ✅ Private RDS database
- ✅ ElastiCache cluster
- ✅ EC2 instances in VPC
- ✅ Internal services
- ✅ VPC endpoints
- ⚠️ Public internet (requires NAT Gateway or NAT Instance)
- ⚠️ Public AWS services (requires NAT or VPC endpoints)

### Why Use VPC Integration?

**Use Case 1: Private Database Access**
```
Lambda Function → Private RDS Database
- Database not exposed to internet
- Secure, private connection
- Example: API that queries customer database
```

**Use Case 2: Microservices Communication**
```
Lambda Function → Internal ALB → EC2 Microservices
- Internal load balancer (no public access)
- Service-to-service communication
- Example: Order processing calling inventory service
```

**Use Case 3: Caching Layer Access**
```
Lambda Function → ElastiCache (Redis/Memcached)
- In-memory cache for performance
- Private cluster in VPC
- Example: Session storage, query caching
```

**Use Case 4: Legacy System Integration**
```
Lambda Function → On-Premises Database (via VPN/Direct Connect)
- VPN or Direct Connect to corporate network
- Access internal systems
- Example: Sync data from mainframe
```

**Use Case 5: Compliance Requirements**
```
Lambda Function → Private Subnet → Audit Logging System
- Meet regulatory requirements
- Data never leaves VPC
- Example: HIPAA, PCI-DSS compliance
```

---

## 2. How Lambda VPC Integration Works

### Theory: Elastic Network Interfaces (ENIs)

#### The Old Way (Pre-2019): One ENI Per Concurrent Execution

**Problem: Cold Starts and ENI Limits**

```
Initial State: Function never invoked
  ↓
First Invocation:
  1. Lambda creates ENI in your VPC
  2. Attaches ENI to execution environment
  3. Waits for ENI to be available (30-90 seconds!)
  4. Runs function

Cold Start: 30-90 SECONDS 🐌

Concurrent Invocations:
  Request 1 → ENI 1 (create new ENI - 30s wait)
  Request 2 → ENI 2 (create new ENI - 30s wait)
  Request 3 → ENI 3 (create new ENI - 30s wait)

Problems:
  - Very slow cold starts
  - ENI creation time unpredictable
  - ENI quota limits (350 per subnet)
  - Each function version/alias = separate ENI pool
```

#### The New Way (Post-2019): Hyperplane ENIs

**AWS Hyperplane: Shared Network Infrastructure**

```
AWS Hyperplane (Shared NAT Technology)
  ↓
  Creates ONE ENI per subnet/security group combo
  ↓
  All function executions share these ENIs
  ↓
  Multiplexing traffic through shared ENIs
```

**How it Works:**

```
┌─────────────────────────────────────────────────────┐
│  AWS Lambda Service (Hyperplane Network)            │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Exec Env │  │ Exec Env │  │ Exec Env │         │
│  │    1     │  │    2     │  │    3     │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │             │             │                 │
│       └─────────────┴─────────────┘                │
│                     ↓                               │
│          ┌──────────────────────┐                  │
│          │   Hyperplane NAT     │                  │
│          │  (Traffic Multiplexer)│                 │
│          └──────────┬───────────┘                  │
└─────────────────────│────────────────────────────────┘
                      ↓
┌─────────────────────│────────────────────────────────┐
│  Your VPC           ↓                                 │
│          ┌──────────────────────┐                    │
│          │  Shared ENI          │                    │
│          │  (One per subnet +   │                    │
│          │   security group)    │                    │
│          └──────────┬───────────┘                    │
│                     ↓                                 │
│          ┌──────────────────────┐                    │
│          │  Your Resources      │                    │
│          │  (RDS, ElastiCache)  │                    │
│          └──────────────────────┘                    │
└───────────────────────────────────────────────────────┘
```

**Benefits of Hyperplane:**
1. **Fast cold starts**: ENI already exists (< 1 second)
2. **No ENI limits**: One ENI serves many executions
3. **Predictable performance**: No ENI creation delays
4. **Automatic scaling**: Handles thousands of concurrent executions
5. **Shared across aliases/versions**: More efficient

**First Invocation with Hyperplane:**
```
First Invocation:
  1. Lambda checks: Does ENI exist for this subnet + SG?
     - Yes → Use existing ENI (< 1 second)
     - No → Create ENI (one-time, 10-30 seconds)
  2. Route traffic through Hyperplane
  3. Execute function

Cold Start: < 1 SECOND ⚡ (after ENI exists)
```

### VPC Configuration Components

#### 1. Subnets

**You must specify subnets where Lambda will connect:**

```yaml
# Lambda VPC Configuration
VpcConfig:
  SubnetIds:
    - subnet-abc123  # Private Subnet 1 (AZ us-east-1a)
    - subnet-def456  # Private Subnet 2 (AZ us-east-1b)
```

**Best Practices:**
- ✅ Use **private subnets** (not public)
- ✅ Use **multiple subnets** in different AZs (high availability)
- ✅ Use **dedicated subnets** for Lambda (easier to manage)
- ❌ Don't use public subnets (Lambda doesn't get public IP)

**Example Subnet Architecture:**

```
VPC: 10.0.0.0/16

├── Public Subnet 1 (10.0.1.0/24) - us-east-1a
│   ├── NAT Gateway
│   └── Internet Gateway
│
├── Public Subnet 2 (10.0.2.0/24) - us-east-1b
│   └── NAT Gateway
│
├── Private Subnet 1 (10.0.10.0/24) - us-east-1a
│   ├── Lambda Functions (ENIs here)
│   └── RDS Primary
│
└── Private Subnet 2 (10.0.20.0/24) - us-east-1b
    ├── Lambda Functions (ENIs here)
    └── RDS Standby
```

#### 2. Security Groups

**Security Groups act as virtual firewalls for your Lambda function:**

```yaml
VpcConfig:
  SecurityGroupIds:
    - sg-123abc  # Lambda Security Group
```

**Security Group Rules Example:**

```
Lambda Security Group (sg-lambda-123):

OUTBOUND RULES (What Lambda can access):
  ┌─────────────────────────────────────────────────┐
  │ Type        Protocol  Port    Destination       │
  ├─────────────────────────────────────────────────┤
  │ MYSQL/Aurora  TCP     3306    sg-rds-456       │ ← RDS access
  │ Redis         TCP     6379    sg-redis-789     │ ← ElastiCache
  │ HTTPS         TCP     443     0.0.0.0/0        │ ← Internet (via NAT)
  │ HTTP          TCP     80      0.0.0.0/0        │ ← Internet (via NAT)
  └─────────────────────────────────────────────────┘

INBOUND RULES (What can access Lambda):
  ┌─────────────────────────────────────────────────┐
  │ Usually NONE - Lambda initiates connections     │
  │ (unless Lambda itself is a server, rare)        │
  └─────────────────────────────────────────────────┘
```

**RDS Security Group (receives connections from Lambda):**

```
RDS Security Group (sg-rds-456):

INBOUND RULES:
  ┌─────────────────────────────────────────────────┐
  │ Type        Protocol  Port    Source            │
  ├─────────────────────────────────────────────────┤
  │ MYSQL/Aurora  TCP     3306    sg-lambda-123    │ ← Allow Lambda
  └─────────────────────────────────────────────────┘

OUTBOUND RULES:
  ┌─────────────────────────────────────────────────┐
  │ All Traffic   All     All     0.0.0.0/0         │
  └─────────────────────────────────────────────────┘
```

#### 3. IAM Permissions for VPC

**Lambda needs IAM permissions to manage ENIs in your VPC:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    }
  ]
}
```

**AWS Managed Policy (easier):**
```
arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

This policy is **required** for VPC-enabled Lambda functions.

---

## 3. Internet Access from VPC Lambda Functions

### The Internet Access Problem

**⚠️ IMPORTANT: Lambda in VPC does NOT have internet access by default!**

```
Lambda Function in VPC (No NAT):
  ↓
  Tries to call api.example.com
  ↓
  ❌ TIMEOUT - No route to internet!
```

**Why?**
- Private subnets have no direct internet route
- Lambda in VPC uses private subnet
- No public IP assigned to Lambda ENI

### Solution 1: NAT Gateway (Recommended)

**Architecture:**

```
┌────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Public Subnet (10.0.1.0/24)                      │ │
│  │                                                   │ │
│  │   ┌───────────────────┐    ┌──────────────────┐ │ │
│  │   │  NAT Gateway      │────│ Internet Gateway │─┼─┼──→ Internet
│  │   │  (Public IP)      │    │                  │ │ │
│  │   └───────────────────┘    └──────────────────┘ │ │
│  └──────────────┬───────────────────────────────────┘ │
│                 ↑                                      │
│  ┌──────────────│───────────────────────────────────┐ │
│  │ Private Subnet (10.0.10.0/24)                    │ │
│  │              ↑                                    │ │
│  │  ┌───────────┴──────────┐                        │ │
│  │  │ Lambda Function       │                       │ │
│  │  │ (Private IP only)     │                       │ │
│  │  └───────────┬───────────┘                       │ │
│  │              ↓                                    │ │
│  │  ┌──────────────────────┐                        │ │
│  │  │ RDS Database         │                        │ │
│  │  │ (Private IP)         │                        │ │
│  │  └──────────────────────┘                        │ │
│  └──────────────────────────────────────────────────┘ │
│                                                         │
└────────────────────────────────────────────────────────┘
```

**Route Table for Private Subnet:**

```
Destination        Target
─────────────────────────────────
10.0.0.0/16        local           ← VPC-internal traffic
0.0.0.0/0          nat-0a1b2c3d    ← Internet traffic → NAT
```

**Traffic Flow:**

```
1. Lambda Function calls api.example.com
   ↓
2. Private subnet route table: 0.0.0.0/0 → NAT Gateway
   ↓
3. NAT Gateway (in public subnet) translates private IP to public IP
   ↓
4. Internet Gateway forwards to internet
   ↓
5. Response follows reverse path back to Lambda
```

**Setting up NAT Gateway:**

```bash
# 1. Create NAT Gateway in PUBLIC subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-123 \
  --allocation-id eipalloc-abc123  # Elastic IP

# 2. Update route table for PRIVATE subnet
aws ec2 create-route \
  --route-table-id rtb-private-456 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0a1b2c3d
```

**NAT Gateway Costs:**
- **Hourly charge**: ~$0.045/hour (~$32/month)
- **Data processing**: $0.045 per GB processed
- **High Availability**: Need one NAT per AZ (~$64/month for 2 AZs)

**Pros:**
- ✅ Fully managed by AWS
- ✅ Highly available (within AZ)
- ✅ Scales automatically to 45 Gbps
- ✅ No maintenance

**Cons:**
- ❌ Costs money (even when idle)
- ❌ Need one per AZ for HA
- ❌ Data processing charges

### Solution 2: VPC Endpoints (More Efficient)

**For AWS Services Only - No internet needed!**

**Architecture:**

```
┌────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Private Subnet (10.0.10.0/24)                    │ │
│  │                                                   │ │
│  │  ┌───────────────────┐                           │ │
│  │  │ Lambda Function   │                           │ │
│  │  └────────┬──────────┘                           │ │
│  │           │ (Private connection)                 │ │
│  │           ↓                                       │ │
│  │  ┌───────────────────┐                           │ │
│  │  │ VPC Endpoint      │                           │ │
│  │  │ (for S3, DynamoDB)│──────────────────────────┼─┼──→ AWS Service
│  │  └───────────────────┘   (Private AWS network)  │ │   (No internet!)
│  │                                                   │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

**Types of VPC Endpoints:**

**1. Gateway Endpoints (Free!):**
- **S3** and **DynamoDB** only
- No hourly charge
- No data processing charge
- Add route to route table

**2. Interface Endpoints (Paid):**
- Most other AWS services (SNS, SQS, API Gateway, etc.)
- **Cost**: ~$0.01/hour per AZ (~$7/month per AZ)
- **Data processing**: $0.01 per GB
- Elastic Network Interface in your subnet

**Creating S3 Gateway Endpoint:**

```bash
# Create S3 VPC Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private-456
```

**Route Table after S3 Endpoint:**

```
Destination               Target
────────────────────────────────────────
10.0.0.0/16              local
pl-63a5400a (S3 prefix)  vpce-s3-123    ← S3 traffic → VPC Endpoint
```

**Traffic Flow:**

```
Lambda Function → s3.amazonaws.com
  ↓
VPC Endpoint (private connection)
  ↓
S3 Service (via AWS private network)
  ↓
Response back to Lambda

No internet traversal! 🔒 Faster & Cheaper!
```

**When to Use VPC Endpoints:**

```
Lambda only needs to access AWS services?
  ↓
  Use VPC Endpoints (cheaper, faster, more secure)

Lambda needs internet access (third-party APIs)?
  ↓
  Use NAT Gateway + VPC Endpoints (hybrid)
```

### Solution 3: NAT Instance (Legacy, Not Recommended)

**EC2 instance configured as NAT:**

```
┌────────────────────────────────────────────────────────┐
│  VPC                                                    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Public Subnet                                     │ │
│  │                                                   │ │
│  │   ┌───────────────────┐    ┌──────────────────┐ │ │
│  │   │  NAT Instance     │────│ Internet Gateway │─┼─┼──→ Internet
│  │   │  (EC2, t3.micro)  │    │                  │ │ │
│  │   └───────────────────┘    └──────────────────┘ │ │
│  └──────────────┬───────────────────────────────────┘ │
│                 ↑                                      │
│  ┌──────────────│───────────────────────────────────┐ │
│  │ Private Subnet                                    │ │
│  │              ↑                                    │ │
│  │  ┌───────────┴──────────┐                        │ │
│  │  │ Lambda Function       │                       │ │
│  │  └───────────────────────┘                       │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Cheaper than NAT Gateway (EC2 costs only)
- ✅ Can use Spot Instances

**Cons:**
- ❌ You manage EC2 instance (patching, monitoring)
- ❌ Single point of failure (unless HA setup)
- ❌ Limited bandwidth (instance type dependent)
- ❌ More complex to configure

**Not recommended** unless very specific cost requirements.

---

## 4. Common VPC Lambda Patterns

### Pattern 1: Lambda → RDS (No Internet)

**Use Case:** API that queries database

```
┌────────────────────────────────────────┐
│  VPC                                   │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │ Private Subnet                   │ │
│  │                                  │ │
│  │  ┌──────────────┐               │ │
│  │  │ Lambda       │               │ │
│  │  │ (sg-lambda)  │               │ │
│  │  └──────┬───────┘               │ │
│  │         │ Port 3306             │ │
│  │         ↓                        │ │
│  │  ┌──────────────┐               │ │
│  │  │ RDS MySQL    │               │ │
│  │  │ (sg-rds)     │               │ │
│  │  └──────────────┘               │ │
│  └──────────────────────────────────┘ │
└────────────────────────────────────────┘
```

**Configuration:**
- Lambda in private subnet
- Lambda SG: Outbound rule → RDS SG, port 3306
- RDS SG: Inbound rule ← Lambda SG, port 3306
- **No NAT needed** (VPC-internal only)

**Example Code:**

```python
import pymysql
import os

def lambda_handler(event, context):
    # Connect to RDS
    connection = pymysql.connect(
        host=os.environ['DB_HOST'],        # RDS endpoint
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME']
    )

    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM users WHERE id = %s", (event['user_id'],))
        result = cursor.fetchone()

    connection.close()
    return {'user': result}
```

### Pattern 2: Lambda → RDS + S3 (Hybrid)

**Use Case:** Process data from S3, store in database

```
┌──────────────────────────────────────────────────────┐
│  VPC                                                  │
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │ Private Subnet                                  │ │
│  │                                                 │ │
│  │  ┌──────────────┐                              │ │
│  │  │ Lambda       │──────────┐                   │ │
│  │  └──────┬───────┘          │                   │ │
│  │         │                  │                   │ │
│  │         ↓                  ↓                   │ │
│  │  ┌──────────────┐   ┌─────────────┐           │ │
│  │  │ RDS          │   │ VPC Endpoint│────────────┼─┼──→ S3
│  │  │              │   │ (S3)        │            │ │   (Private)
│  │  └──────────────┘   └─────────────┘           │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**Configuration:**
- Lambda in private subnet
- S3 Gateway VPC Endpoint (free!)
- Lambda → RDS (private)
- Lambda → S3 (via VPC Endpoint)
- **No NAT needed** (VPC Endpoint for S3)

### Pattern 3: Lambda → RDS + Third-party API

**Use Case:** Save payment to database, call Stripe API

```
┌────────────────────────────────────────────────────────┐
│  VPC                                                    │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Public Subnet                                     │ │
│  │   ┌───────────────┐                              │ │
│  │   │ NAT Gateway   │──→ Internet Gateway ──→ Internet
│  │   └───────────────┘                              │ │
│  └────────────┬─────────────────────────────────────┘ │
│               ↑                                        │
│  ┌────────────┴─────────────────────────────────────┐ │
│  │ Private Subnet                                    │ │
│  │                                                   │ │
│  │  ┌──────────────┐                                │ │
│  │  │ Lambda       │────┐                           │ │
│  │  └──────┬───────┘    │ (via NAT)                 │ │
│  │         │            │                           │ │
│  │         ↓            ↓                           │ │
│  │  ┌──────────────┐  Internet                     │ │
│  │  │ RDS          │   (Stripe API)                │ │
│  │  └──────────────┘                                │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

**Configuration:**
- Lambda in private subnet
- NAT Gateway in public subnet
- Private subnet route: 0.0.0.0/0 → NAT
- Lambda → RDS (private)
- Lambda → Internet (via NAT)

### Pattern 4: Multi-AZ High Availability

**Use Case:** Production-grade HA setup

```
┌─────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                           │
│                                                              │
│  ┌─────────────────────┐       ┌─────────────────────┐     │
│  │ Public Subnet AZ-A  │       │ Public Subnet AZ-B  │     │
│  │ 10.0.1.0/24         │       │ 10.0.2.0/24         │     │
│  │  ┌──────────────┐   │       │  ┌──────────────┐   │     │
│  │  │ NAT Gateway A│   │       │  │ NAT Gateway B│   │     │
│  │  └──────────────┘   │       │  └──────────────┘   │     │
│  └──────────┬──────────┘       └──────────┬──────────┘     │
│             ↑                              ↑                │
│  ┌──────────┴──────────┐       ┌──────────┴──────────┐     │
│  │ Private Subnet AZ-A │       │ Private Subnet AZ-B │     │
│  │ 10.0.10.0/24        │       │ 10.0.20.0/24        │     │
│  │  ┌──────────────┐   │       │  ┌──────────────┐   │     │
│  │  │ Lambda ENI   │   │       │  │ Lambda ENI   │   │     │
│  │  └──────┬───────┘   │       │  └──────┬───────┘   │     │
│  │         ↓            │       │         ↓            │     │
│  │  ┌──────────────┐   │       │  ┌──────────────┐   │     │
│  │  │ RDS Primary  │←──┼───────┼──│ RDS Standby  │   │     │
│  │  │              │   │       │  │ (Sync repl.) │   │     │
│  │  └──────────────┘   │       │  └──────────────┘   │     │
│  └─────────────────────┘       └─────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**Configuration:**
- **2 Public subnets**: NAT Gateways (one per AZ)
- **2 Private subnets**: Lambda ENIs (one per AZ)
- **RDS Multi-AZ**: Primary in AZ-A, Standby in AZ-B
- **Route tables**: Each private subnet routes to NAT in same AZ

**Benefits:**
- ✅ High availability (survives AZ failure)
- ✅ Lambda auto-scales across AZs
- ✅ RDS automatic failover

---

## 5. Configuring Lambda VPC Access

### AWS CLI Configuration

```bash
# Create Lambda function with VPC
aws lambda create-function \
  --function-name MyVPCFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-vpc-role \
  --handler index.lambda_handler \
  --zip-file fileb://function.zip \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-lambda123 \
  --timeout 30
```

**Update existing function to add VPC:**

```bash
aws lambda update-function-configuration \
  --function-name MyFunction \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-lambda123
```

**Remove VPC configuration:**

```bash
aws lambda update-function-configuration \
  --function-name MyFunction \
  --vpc-config SubnetIds=[],SecurityGroupIds=[]
```

### CloudFormation/SAM Configuration

**SAM Template:**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  MyVPCFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./src
      Handler: index.lambda_handler
      Runtime: python3.9
      Timeout: 30
      VpcConfig:
        SubnetIds:
          - subnet-abc123
          - subnet-def456
        SecurityGroupIds:
          - sg-lambda123
      Policies:
        - VPCAccessPolicy: {}  # Adds AWSLambdaVPCAccessExecutionRole
```

**CloudFormation (Native Lambda):**

```yaml
Resources:
  MyVPCFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: MyVPCFunction
      Runtime: python3.9
      Handler: index.lambda_handler
      Code:
        S3Bucket: my-code-bucket
        S3Key: function.zip
      Role: !GetAtt LambdaExecutionRole.Arn
      VpcConfig:
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
      Timeout: 30

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
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Lambda function
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          DestinationSecurityGroupId: !Ref RDSSecurityGroup
```

### Terraform Configuration

```hcl
resource "aws_lambda_function" "vpc_lambda" {
  filename      = "function.zip"
  function_name = "MyVPCFunction"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.lambda_handler"
  runtime       = "python3.9"
  timeout       = 30

  vpc_config {
    subnet_ids         = [
      aws_subnet.private_subnet_1.id,
      aws_subnet.private_subnet_2.id
    ]
    security_group_ids = [aws_security_group.lambda_sg.id]
  }
}

resource "aws_iam_role" "lambda_role" {
  name = "lambda_vpc_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_vpc_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_security_group" "lambda_sg" {
  name        = "lambda-sg"
  description = "Security group for Lambda"
  vpc_id      = aws_vpc.main.id

  egress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.rds_sg.id]
  }
}
```

---

## 6. Troubleshooting VPC Lambda Issues

### Issue 1: Lambda Timeout When Accessing Internet

**Symptoms:**
```
Task timed out after 30.00 seconds
or
[Errno 110] Connection timed out
```

**Cause:** Lambda in VPC without NAT Gateway/VPC Endpoint trying to access internet

**Solution:**

1. **Check if you need internet access:**
   ```python
   # Are you calling external APIs?
   response = requests.get('https://api.stripe.com')  # Needs NAT!

   # Or just AWS services?
   s3.put_object(Bucket='mybucket', Key='file', Body=data)  # Use VPC Endpoint!
   ```

2. **For AWS services → Add VPC Endpoint:**
   ```bash
   # S3 Gateway Endpoint (FREE)
   aws ec2 create-vpc-endpoint \
     --vpc-id vpc-abc123 \
     --service-name com.amazonaws.us-east-1.s3 \
     --route-table-ids rtb-private-456
   ```

3. **For internet access → Add NAT Gateway:**
   ```bash
   # Create NAT Gateway
   aws ec2 create-nat-gateway \
     --subnet-id subnet-public-123 \
     --allocation-id eipalloc-abc123

   # Update private subnet route table
   aws ec2 create-route \
     --route-table-id rtb-private-456 \
     --destination-cidr-block 0.0.0.0/0 \
     --nat-gateway-id nat-0a1b2c3d
   ```

### Issue 2: Cannot Connect to RDS

**Symptoms:**
```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")
or
psycopg2.OperationalError: could not connect to server
```

**Troubleshooting Steps:**

**Step 1: Check Security Groups**

```bash
# Lambda security group - check outbound rules
aws ec2 describe-security-groups --group-ids sg-lambda123

# Should have outbound rule to RDS security group:
# Protocol: TCP, Port: 3306 (MySQL) or 5432 (PostgreSQL)
# Destination: sg-rds-456

# RDS security group - check inbound rules
aws ec2 describe-security-groups --group-ids sg-rds-456

# Should have inbound rule from Lambda security group:
# Protocol: TCP, Port: 3306/5432
# Source: sg-lambda123
```

**Step 2: Check RDS is in same VPC**

```bash
# Get RDS VPC
aws rds describe-db-instances \
  --db-instance-identifier mydb \
  --query 'DBInstances[0].DBSubnetGroup.VpcId'

# Get Lambda VPC
aws lambda get-function-configuration \
  --function-name MyFunction \
  --query 'VpcConfig.VpcId'

# Must match!
```

**Step 3: Check Subnets Have Route to Each Other**

```bash
# Lambda and RDS should be in same VPC
# No need for routing if in same VPC
# Just ensure security groups are correct
```

**Step 4: Test Connection from Lambda**

```python
import socket

def lambda_handler(event, context):
    # Test if RDS endpoint is reachable
    host = 'mydb.abc123.us-east-1.rds.amazonaws.com'
    port = 3306

    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(5)
        result = sock.connect_ex((host, port))
        sock.close()

        if result == 0:
            return {'status': 'Port is open'}
        else:
            return {'status': 'Port is closed', 'error_code': result}
    except Exception as e:
        return {'status': 'Error', 'error': str(e)}
```

### Issue 3: Slow Cold Starts

**Symptoms:**
```
First invocation: 15-30 seconds
Subsequent invocations: < 1 second
```

**Cause:** ENI creation (should be rare with Hyperplane, but can happen)

**Solutions:**

**1. Use Provisioned Concurrency** (eliminates cold starts)

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name MyFunction:PROD \
  --provisioned-concurrent-executions 5
```

**2. Reduce VPC Complexity**

```bash
# Use fewer security groups
# Use fewer subnets (but at least 2 for HA)
```

**3. Keep Functions Warm** (scheduled invocations)

```python
# CloudWatch Events Rule (every 5 minutes)
{
  "source": ["aws.events"],
  "detail-type": ["Scheduled Event"],
  "detail": {}
}

# Lambda handler
def lambda_handler(event, context):
    if event.get('source') == 'aws.events':
        return {'status': 'warmed'}

    # Normal processing
    ...
```

### Issue 4: ENI Limit Exceeded

**Symptoms:**
```
ENILimitReachedException: The provided execution role does not have permissions to call CreateNetworkInterface on EC2
```

**Cause:** (Rare with Hyperplane) Too many ENIs in subnet

**Solutions:**

**1. Check ENI Count**

```bash
# List all ENIs in subnet
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-abc123" \
  --query 'NetworkInterfaces[].NetworkInterfaceId'
```

**2. Request ENI Limit Increase**

```bash
# Default: 350 ENIs per subnet
# Can request increase via AWS Support
```

**3. Use Larger Subnet CIDR**

```bash
# /24 subnet = 256 IPs - 5 (AWS reserved) = 251 available
# /23 subnet = 512 IPs - 5 (AWS reserved) = 507 available
```

### Issue 5: Lambda Cannot Assume Execution Role

**Symptoms:**
```
Lambda is unable to configure VPC access for your function
```

**Cause:** Missing VPC permissions in execution role

**Solution:**

```bash
# Attach VPC access policy
aws iam attach-role-policy \
  --role-name lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

**Or create inline policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 7. Best Practices for Lambda VPC

### 1. Subnet Design

**✅ DO:**
- Use **dedicated subnets** for Lambda (e.g., 10.0.100.0/24, 10.0.101.0/24)
- Use **multiple AZs** (minimum 2 for HA)
- Use **private subnets** (not public)
- Use **/24 or larger** CIDR (enough IPs)

**❌ DON'T:**
- Share subnets with other services (harder to manage)
- Use only one AZ (no redundancy)
- Use public subnets (Lambda won't get public IP anyway)
- Use tiny subnets like /28 (only 16 IPs)

**Example:**

```
VPC: 10.0.0.0/16

Lambda Subnets (Dedicated):
├── Lambda Subnet AZ-A: 10.0.100.0/24 (251 IPs)
└── Lambda Subnet AZ-B: 10.0.101.0/24 (251 IPs)

Application Subnets:
├── App Subnet AZ-A: 10.0.10.0/24
└── App Subnet AZ-B: 10.0.20.0/24

Data Subnets:
├── Data Subnet AZ-A: 10.0.30.0/24
└── Data Subnet AZ-B: 10.0.40.0/24
```

### 2. Security Group Strategy

**✅ DO:**
- Create **dedicated security group** for Lambda
- Use **least privilege** (only allow necessary outbound traffic)
- Reference security groups by ID (not IP ranges)
- Document all rules

**❌ DON'T:**
- Reuse security groups across different services
- Allow all outbound traffic (0.0.0.0/0 on all ports)
- Use IP ranges when SGs can be referenced

**Example:**

```bash
Lambda Security Group (sg-lambda):
  OUTBOUND:
    - RDS: TCP 3306 → sg-rds
    - ElastiCache: TCP 6379 → sg-redis
    - HTTPS: TCP 443 → 0.0.0.0/0 (for external APIs via NAT)

RDS Security Group (sg-rds):
  INBOUND:
    - MySQL: TCP 3306 ← sg-lambda
    - MySQL: TCP 3306 ← sg-app (if needed)
```

### 3. Cost Optimization

**Strategy 1: Use VPC Endpoints Instead of NAT**

```
Cost Comparison (per month):

NAT Gateway:
  - Hourly: $32.40 (1 AZ) or $64.80 (2 AZs)
  - Data: $0.045/GB
  - Total: ~$70-$150/month

S3 Gateway VPC Endpoint:
  - Hourly: $0
  - Data: $0
  - Total: FREE!

DynamoDB Gateway VPC Endpoint:
  - Hourly: $0
  - Data: $0
  - Total: FREE!
```

**When to use what:**

```
Lambda only accesses S3/DynamoDB:
  → Use Gateway VPC Endpoints (FREE!)

Lambda accesses RDS only (no internet):
  → No NAT needed (private VPC traffic)

Lambda needs internet for third-party APIs:
  → NAT Gateway + VPC Endpoints (hybrid)
  → Use VPC Endpoints for AWS services (save $)
```

**Strategy 2: Right-Size NAT Gateways**

```
Development/Staging:
  → 1 NAT Gateway (single AZ is OK)
  → ~$35/month

Production:
  → 2 NAT Gateways (multi-AZ for HA)
  → ~$70/month
```

**Strategy 3: Remove VPC When Not Needed**

```
Lambda needs VPC only for:
  ✅ RDS/Aurora access
  ✅ ElastiCache access
  ✅ Private EC2/ECS services
  ✅ On-premises via VPN/Direct Connect

Lambda does NOT need VPC for:
  ❌ S3 (public API)
  ❌ DynamoDB (public API)
  ❌ SNS/SQS (public API)
  ❌ Most AWS services (public APIs)
  ❌ Public internet APIs

Recommendation: Remove VPC if not accessing private resources!
```

### 4. High Availability

**Multi-AZ Configuration:**

```yaml
# SAM Template - Multi-AZ
VpcConfig:
  SubnetIds:
    - !Ref PrivateSubnetAZa    # us-east-1a
    - !Ref PrivateSubnetAZb    # us-east-1b
    - !Ref PrivateSubnetAZc    # us-east-1c (optional)
  SecurityGroupIds:
    - !Ref LambdaSecurityGroup
```

**Benefits:**
- Lambda automatically distributes across AZs
- Survives AZ failure
- No configuration needed (automatic)

### 5. Connection Pooling for Databases

**Problem: Lambda creates new DB connection every invocation**

```python
# ❌ BAD: Creates new connection every time
def lambda_handler(event, context):
    connection = pymysql.connect(host=DB_HOST, user=DB_USER, ...)
    # ... query ...
    connection.close()

# Problem: Overwhelms RDS with connections!
# 100 concurrent Lambdas = 100 DB connections
```

**Solution 1: Reuse Connection Across Invocations**

```python
# ✅ GOOD: Reuse connection
connection = None

def lambda_handler(event, context):
    global connection

    # Reuse existing connection
    if connection is None or not connection.open:
        connection = pymysql.connect(host=DB_HOST, user=DB_USER, ...)

    # ... query ...
    # Don't close! Reuse in next invocation

# Warm container reuses connection
# Only creates new connection on cold start
```

**Solution 2: Use RDS Proxy**

```
┌────────────────────────────────────────────┐
│  Lambda Functions                          │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │
│  │Lambda│ │Lambda│ │Lambda│ │Lambda│      │
│  │  1   │ │  2   │ │  3   │ │  4   │      │
│  └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘      │
└──────│────────│────────│────────│──────────┘
       │        │        │        │
       └────────┴────────┴────────┘
                  ↓
       ┌──────────────────────┐
       │   RDS Proxy          │
       │   (Connection Pool)  │
       └──────────┬───────────┘
                  ↓
       ┌──────────────────────┐
       │   RDS Database       │
       │   (25 connections)   │
       └──────────────────────┘

100 Lambda invocations → 25 DB connections (pooled!)
```

**RDS Proxy Benefits:**
- ✅ Connection pooling (reduces DB connections)
- ✅ Automatic failover
- ✅ IAM authentication
- ✅ Secrets Manager integration

```python
# Using RDS Proxy
connection = pymysql.connect(
    host='my-rds-proxy.proxy-abc123.us-east-1.rds.amazonaws.com',  # Proxy endpoint!
    user=DB_USER,
    password=DB_PASSWORD,
    database=DB_NAME
)
```

### 6. Monitoring VPC Lambda

**CloudWatch Metrics:**

```python
# Key metrics to monitor:
- Invocations: Total invocations
- Duration: How long function runs
- Errors: Failed invocations
- Throttles: Throttled due to concurrency limits

# VPC-specific:
- ConcurrentExecutions: Current concurrent executions
- Duration: Watch for increased latency (VPC overhead)
```

**CloudWatch Logs:**

```python
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info('Starting VPC Lambda function')
    logger.info(f'VPC Config: {context.invoked_function_arn}')

    try:
        # Connect to RDS
        connection = pymysql.connect(...)
        logger.info('Successfully connected to RDS')
    except Exception as e:
        logger.error(f'Failed to connect to RDS: {str(e)}')
        raise
```

**VPC Flow Logs:**

```bash
# Enable VPC Flow Logs to monitor network traffic
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-abc123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

---

## 8. Exercise: Deploy VPC Lambda with RDS

**Objective:** Deploy a Lambda function in VPC that connects to RDS MySQL database.

### Step 1: Create VPC and Subnets

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC Created: $VPC_ID"

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway \
  --vpc-id $VPC_ID \
  --internet-gateway-id $IGW_ID

echo "Internet Gateway: $IGW_ID"

# Create Private Subnets (for Lambda and RDS)
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' \
  --output text)

PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.20.0/24 \
  --availability-zone us-east-1b \
  --query 'Subnet.SubnetId' \
  --output text)

echo "Private Subnets: $PRIVATE_SUBNET_1, $PRIVATE_SUBNET_2"
```

### Step 2: Create Security Groups

```bash
# Lambda Security Group
LAMBDA_SG=$(aws ec2 create-security-group \
  --group-name lambda-sg \
  --description "Security group for Lambda" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Lambda SG: $LAMBDA_SG"

# RDS Security Group
RDS_SG=$(aws ec2 create-security-group \
  --group-name rds-sg \
  --description "Security group for RDS" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "RDS SG: $RDS_SG"

# Lambda SG: Allow outbound to RDS (port 3306)
aws ec2 authorize-security-group-egress \
  --group-id $LAMBDA_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $RDS_SG

# RDS SG: Allow inbound from Lambda (port 3306)
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $LAMBDA_SG

echo "Security group rules configured"
```

### Step 3: Create RDS Instance

```bash
# Create DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "Subnet group for RDS" \
  --subnet-ids $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2

# Create RDS MySQL Instance
aws rds create-db-instance \
  --db-instance-identifier my-test-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MyPassword123! \
  --allocated-storage 20 \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible

echo "RDS instance creating (takes ~10 minutes)..."
echo "Check status: aws rds describe-db-instances --db-instance-identifier my-test-db"

# Wait for RDS to be available
aws rds wait db-instance-available \
  --db-instance-identifier my-test-db

# Get RDS Endpoint
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier my-test-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

echo "RDS Endpoint: $DB_ENDPOINT"
```

### Step 4: Create Lambda Function

**Create `lambda_function.py`:**

```python
import json
import pymysql
import os

def lambda_handler(event, context):
    """
    Lambda function that connects to RDS MySQL in VPC
    """
    db_host = os.environ['DB_HOST']
    db_user = os.environ['DB_USER']
    db_password = os.environ['DB_PASSWORD']
    db_name = os.environ.get('DB_NAME', 'mysql')

    try:
        # Connect to RDS
        connection = pymysql.connect(
            host=db_host,
            user=db_user,
            password=db_password,
            database=db_name,
            connect_timeout=5
        )

        with connection.cursor() as cursor:
            # Simple query to test connection
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()

        connection.close()

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Successfully connected to RDS!',
                'mysql_version': version[0],
                'db_host': db_host
            })
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': 'Failed to connect to RDS',
                'error': str(e)
            })
        }
```

**Install dependencies:**

```bash
# Create deployment package
mkdir lambda-package
cd lambda-package

# Install pymysql
pip install pymysql -t .

# Copy function code
cp ../lambda_function.py .

# Create zip
zip -r ../lambda-vpc-rds.zip .
cd ..
```

### Step 5: Create IAM Role

```bash
# Create trust policy
cat > trust-policy.json <<EOF
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
EOF

# Create role
ROLE_ARN=$(aws iam create-role \
  --role-name lambda-vpc-rds-role \
  --assume-role-policy-document file://trust-policy.json \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"

# Attach policies
aws iam attach-role-policy \
  --role-name lambda-vpc-rds-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
  --role-name lambda-vpc-rds-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

echo "Waiting for role to propagate..."
sleep 10
```

### Step 6: Deploy Lambda Function

```bash
# Create Lambda function with VPC config
aws lambda create-function \
  --function-name VPCRDSFunction \
  --runtime python3.9 \
  --role $ROLE_ARN \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda-vpc-rds.zip \
  --timeout 30 \
  --vpc-config SubnetIds=$PRIVATE_SUBNET_1,$PRIVATE_SUBNET_2,SecurityGroupIds=$LAMBDA_SG \
  --environment Variables="{DB_HOST=$DB_ENDPOINT,DB_USER=admin,DB_PASSWORD=MyPassword123!}"

echo "Lambda function created!"
```

### Step 7: Test the Function

```bash
# Invoke the function
aws lambda invoke \
  --function-name VPCRDSFunction \
  --payload '{}' \
  response.json

# View response
cat response.json
```

**Expected Output:**

```json
{
  "statusCode": 200,
  "body": "{\"message\": \"Successfully connected to RDS!\", \"mysql_version\": \"8.0.35\", \"db_host\": \"my-test-db.abc123.us-east-1.rds.amazonaws.com\"}"
}
```

### Step 8: Cleanup

```bash
# Delete Lambda function
aws lambda delete-function --function-name VPCRDSFunction

# Delete RDS instance
aws rds delete-db-instance \
  --db-instance-identifier my-test-db \
  --skip-final-snapshot

# Delete DB subnet group (after RDS is deleted)
aws rds delete-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group

# Delete security groups
aws ec2 delete-security-group --group-id $LAMBDA_SG
aws ec2 delete-security-group --group-id $RDS_SG

# Delete subnets
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_1
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_2

# Detach and delete internet gateway
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Delete IAM role
aws iam detach-role-policy \
  --role-name lambda-vpc-rds-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy \
  --role-name lambda-vpc-rds-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

aws iam delete-role --role-name lambda-vpc-rds-role

# Remove local files
rm -rf lambda-package/ lambda-vpc-rds.zip response.json trust-policy.json lambda_function.py
```

---

## Summary

### Key Concepts

**VPC Integration:**
- Lambda runs in AWS-managed VPC by default (public internet access only)
- VPC integration allows access to private resources (RDS, ElastiCache, EC2)
- Uses Elastic Network Interfaces (ENIs) to connect to your VPC
- Hyperplane technology (post-2019) provides fast, scalable VPC access

**Networking:**
- **Subnets**: Lambda creates ENIs in your specified subnets
- **Security Groups**: Control inbound/outbound traffic
- **NAT Gateway**: Required for internet access from VPC Lambda
- **VPC Endpoints**: Private access to AWS services (S3, DynamoDB - free!)

**Best Practices:**
1. Use dedicated private subnets for Lambda
2. Deploy in multiple AZs for high availability
3. Use VPC Endpoints for AWS services (save money!)
4. Use NAT Gateway only when needed (costs money)
5. Create dedicated security groups with least privilege
6. Use RDS Proxy for connection pooling
7. Monitor with CloudWatch Logs and VPC Flow Logs

**Common Issues:**
- Timeout accessing internet → Add NAT Gateway or VPC Endpoint
- Cannot connect to RDS → Check security groups
- Slow cold starts → Use Provisioned Concurrency
- ENI limit exceeded → Use larger subnets or request increase

**Cost Optimization:**
- Use Gateway VPC Endpoints (S3, DynamoDB) - FREE
- Minimize NAT Gateway usage (expensive)
- Remove VPC config if not accessing private resources
- Use single NAT for dev/test, multi-AZ for production

---

This completes the Lambda VPC and Networking deep dive! You now understand how Lambda integrates with VPCs, how to configure networking, troubleshoot issues, and follow best practices.
