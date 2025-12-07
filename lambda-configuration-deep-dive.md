# AWS Lambda - Configuration: Deep Dive

Comprehensive explanation of Lambda configuration covering timeout, memory allocation, and environment variables.

---

## 1. Timeout Configuration

### What is Lambda Timeout?

The **timeout** is the maximum amount of time AWS Lambda allows your function to run before terminating it. This prevents runaway functions from consuming resources indefinitely.

### Default and Limits

- **Default**: 3 seconds
- **Minimum**: 1 second
- **Maximum**: 15 minutes (900 seconds)

### The Theory Behind Lambda Timeout

#### How Lambda Executes Functions

When a Lambda function is invoked, AWS creates an execution environment and runs your code:

```
1. Invocation Request Received
   ↓
2. Execution Environment Ready (or reused from previous invocation)
   ↓
3. Timeout Timer Starts ⏱️
   ↓
4. Handler Function Called
   ↓
5. Your Code Executes
   ↓
6a. [Success Path] Function Returns → Timer Stops ✅
6b. [Timeout Path] Timer Reaches Limit → Force Terminate ❌
```

#### What Happens During a Timeout?

When timeout is reached:
1. **Immediate Termination**: Lambda forcefully kills the execution environment
2. **No Cleanup**: Your code doesn't get a chance to close connections, save state, or clean up resources
3. **Error Returned**: Caller receives `Task timed out after X.XX seconds`
4. **Partial Work Lost**: Any uncommitted work (database transactions, file writes) is lost
5. **Billing**: You're charged for the full timeout duration (not actual execution time)

**Example of Lost Work:**
```python
def lambda_handler(event, context):
    # Start database transaction
    conn = get_db_connection()
    conn.begin_transaction()

    # Process 1000 items (takes 5 minutes)
    for item in items:  # 999 items processed successfully
        process_item(item)

    # Timeout at 3 minutes (default) ⏱️❌
    # Transaction never committed - ALL 999 items lost!
    conn.commit()  # ← Never reached
```

#### Timeout vs Billed Duration

**Important Distinction:**
- **Timeout**: Maximum allowed execution time
- **Billed Duration**: Actual execution time (rounded up to nearest 1ms)

```
Example:
- Timeout: 60 seconds
- Actual execution: 2.456 seconds
- Billed Duration: 2.456 seconds (NOT 60 seconds)

You only pay for actual execution time!
```

However, if timeout occurs:
```
- Timeout: 60 seconds
- Function runs for: 60 seconds (timeout!)
- Billed Duration: 60 seconds
- Result: Error + Full timeout cost
```

#### The 15-Minute Hard Limit

**Why 15 minutes maximum?**

Lambda is designed for **event-driven, short-lived compute**. The 15-minute limit exists to:

1. **Prevent resource hogging**: Discourage long-running processes
2. **Encourage proper architecture**: Use Step Functions, ECS, or EC2 for long tasks
3. **Cost predictability**: Prevents runaway costs from infinite loops
4. **Fair resource sharing**: Ensures compute resources available to all customers

**For tasks > 15 minutes, AWS recommends:**
- **Step Functions**: Orchestrate multiple Lambda functions
- **ECS Fargate**: Containerized tasks
- **EC2**: Traditional compute for hours-long processing
- **Batch**: Managed batch processing

### Why Timeout Matters

**Problem Without Proper Timeout:**
```python
def lambda_handler(event, context):
    # This could run forever if the API is down
    response = requests.get('https://slow-api.com/data')
    return response.json()
```

**What could go wrong:**
- API is down → waits for default timeout (varies by library)
- Network issue → hangs indefinitely
- Slow response → waits 15 minutes
- Cost: $0.0000166667 per GB-second × 15 minutes × memory

**With a proper timeout (30 seconds):**
- Function fails fast if API doesn't respond
- Cost is contained
- Errors are detected quickly
- Retry logic can kick in

### Theory: Timeout and Synchronous vs Asynchronous Invocations

#### Synchronous Invocations (RequestResponse)

**Caller waits for response:**
```
Client → API Gateway → Lambda (timeout: 60s)
          ↓ (29s limit)
       Client receives timeout from API Gateway
```

**Key Points:**
- API Gateway has **29-second hard timeout**
- Even if Lambda timeout is 60s, API Gateway will timeout at 29s
- Client sees: `{"message": "Endpoint request timed out"}`
- Lambda continues running until its timeout (wasted compute!)

**Best Practice:**
```bash
# For API Gateway-triggered functions
# Set Lambda timeout < 29 seconds
aws lambda update-function-configuration \
  --function-name ApiFunction \
  --timeout 25  # Less than API Gateway's 29s limit
```

#### Asynchronous Invocations (Event)

**Caller doesn't wait:**
```
S3 Event → Lambda (timeout: 900s)
   ↓
Returns 202 Accepted immediately

Lambda runs up to timeout, then:
- Success → Done ✅
- Timeout → Retry (up to 2 times) → Dead Letter Queue
```

**Key Points:**
- Timeout affects retry behavior
- Failed invocations are retried automatically (2 times)
- Each retry gets the full timeout duration
- Total time = timeout × 3 attempts (potentially 45 minutes for 15-min timeout)

### Setting Timeout

**Via AWS CLI:**
```bash
# Create function with 30-second timeout
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler index.lambda_handler \
  --timeout 30 \
  --zip-file fileb://function.zip

# Update existing function timeout
aws lambda update-function-configuration \
  --function-name MyFunction \
  --timeout 60
```

**Via AWS Console:**
1. Go to Lambda function
2. Configuration tab → General configuration
3. Click "Edit"
4. Set timeout value
5. Save

**Via SAM Template:**
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.lambda_handler
      Runtime: python3.9
      Timeout: 30  # seconds
```

**Via Terraform:**
```hcl
resource "aws_lambda_function" "my_function" {
  function_name = "MyFunction"
  runtime       = "python3.9"
  handler       = "index.lambda_handler"
  timeout       = 30
  role          = aws_iam_role.lambda_role.arn
}
```

### How Timeout Works

```
Function starts
    ↓
Timer begins counting
    ↓
Your code executes
    ↓
[If time remaining > 0]
    ↓
Function completes successfully ✅
    ↓
Returns response

[If time reaches timeout limit]
    ↓
Lambda terminates execution ❌
    ↓
Returns: Task timed out after X.XX seconds
```

### Timeout Error Example

```python
def lambda_handler(event, context):
    import time
    time.sleep(10)  # Sleep for 10 seconds
    return {'statusCode': 200, 'body': 'Done'}
```

**If timeout is set to 5 seconds:**
```
Error: Task timed out after 5.00 seconds
```

**CloudWatch Logs:**
```
START RequestId: abc-123 Version: $LATEST
END RequestId: abc-123
REPORT RequestId: abc-123  Duration: 5003.45 ms  Billed Duration: 5000 ms
2023-11-15T10:30:05.123Z abc-123 Task timed out after 5.00 seconds
```

### Using Context to Prevent Timeout

**Problem**: Function times out before saving progress.

**Solution**: Use `context.get_remaining_time_in_millis()` to check time:

```python
import json
import boto3

def lambda_handler(event, context):
    """
    Process items but stop before timeout to save progress
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('ProcessingQueue')

    items = event['items']
    processed = []

    for item in items:
        # Check if we have at least 10 seconds remaining
        if context.get_remaining_time_in_millis() < 10000:
            print(f"Approaching timeout. Processed {len(processed)} of {len(items)} items")

            # Save checkpoint
            table.put_item(Item={
                'jobId': event['jobId'],
                'processed': processed,
                'remaining': items[len(processed):],
                'status': 'partial'
            })

            return {
                'statusCode': 206,  # Partial Content
                'body': json.dumps({
                    'message': 'Partial processing complete',
                    'processed': len(processed),
                    'remaining': len(items) - len(processed)
                })
            }

        # Process item
        result = process_item(item)
        processed.append(result)

    # All items processed
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'All items processed',
            'processed': len(processed)
        })
    }

def process_item(item):
    # Your processing logic
    return {'id': item['id'], 'status': 'completed'}
```

### Choosing the Right Timeout

| Use Case | Recommended Timeout | Reasoning |
|----------|-------------------|-----------|
| API Gateway integration | 3-30 seconds | API Gateway has 29-second limit |
| Simple data processing | 10-60 seconds | Most operations complete quickly |
| File processing (small files) | 1-3 minutes | Depends on file size |
| Batch processing | 5-15 minutes | Use chunking with timeout monitoring |
| Database queries | 10-60 seconds | Add buffer for connection + query time |
| External API calls | 15-60 seconds | Account for network latency |
| Image/video processing | 5-15 minutes | Depends on file size and complexity |

### Theory: Timeout Impact on Downstream Services

#### Cascading Timeouts

When Lambda calls other services, timeout configuration creates a chain:

```
API Gateway (29s)
    ↓
Lambda Function A (25s timeout)
    ↓
calls Lambda Function B (20s timeout)
    ↓
calls DynamoDB (10s connection timeout)
```

**Rule: Each layer should have progressively shorter timeouts**

**Problem - Inverted Timeout:**
```
API Gateway (29s)
    ↓
Lambda (60s timeout)
    ↓
External API (120s timeout)

Result:
- API Gateway times out at 29s
- Lambda keeps running until 60s (wasted compute!)
- External API call might succeed at 40s, but response lost
```

**Solution - Proper Timeout Chain:**
```
API Gateway (29s)
    ↓
Lambda (25s timeout)
    ↓
External API (20s timeout)

Result:
- External API fails at 20s
- Lambda handles error, returns at 21s
- API Gateway receives response (within 29s)
```

### Best Practices

**1. Set Realistic Timeouts**
```python
# ❌ BAD: Default 3 seconds for file processing
# Timeout: 3 seconds
# Function: Processes 100MB files

# ✅ GOOD: Appropriate timeout based on testing
# Timeout: 300 seconds (5 minutes)
# Function: Processes 100MB files, tested average: 180 seconds
```

**2. Add Buffer Time**
```bash
# If average execution is 45 seconds
# Don't set timeout to 45 seconds
# Set to 60-90 seconds to handle variance
aws lambda update-function-configuration \
  --function-name MyFunction \
  --timeout 90
```

**3. Monitor and Adjust**
```python
# Log execution time to CloudWatch
def lambda_handler(event, context):
    start_time = context.get_remaining_time_in_millis()

    # Do work
    result = process_data(event)

    elapsed = start_time - context.get_remaining_time_in_millis()
    print(f"Execution time: {elapsed}ms")

    # Alert if approaching timeout
    if elapsed > (context.memory_limit_in_mb * 800):  # 80% of timeout
        print("WARNING: Function approaching timeout threshold")

    return result
```

**4. Design for Timeout**
- Break large jobs into smaller chunks
- Use Step Functions for multi-step workflows
- Implement checkpointing for long-running tasks
- Consider using SQS for asynchronous processing

### Common Timeout Patterns

**Pattern 1: Retry with Exponential Backoff**
```python
import time

def lambda_handler(event, context):
    max_retries = 3
    retry_count = 0

    while retry_count < max_retries:
        # Check time remaining
        if context.get_remaining_time_in_millis() < 5000:
            return {'statusCode': 408, 'body': 'Request timeout'}

        try:
            result = call_external_api()
            return {'statusCode': 200, 'body': result}
        except TimeoutError:
            retry_count += 1
            wait_time = 2 ** retry_count  # 2, 4, 8 seconds
            time.sleep(wait_time)

    return {'statusCode': 500, 'body': 'Max retries exceeded'}
```

**Pattern 2: Chunked Processing**
```python
def lambda_handler(event, context):
    items = event['items']
    chunk_size = 100

    for i in range(0, len(items), chunk_size):
        # Process chunk
        chunk = items[i:i + chunk_size]
        process_chunk(chunk)

        # Check if we should continue
        if context.get_remaining_time_in_millis() < 30000:
            # Invoke new Lambda for remaining items
            invoke_continuation(items[i + chunk_size:])
            break

    return {'statusCode': 200}
```

### Exercise: Timeout Configuration and Monitoring

**Objective**: Understand timeout behavior and implement graceful handling.

**Step 1: Create a Function That Simulates Long Processing**

Create `timeout_demo.py`:
```python
import json
import time

def lambda_handler(event, context):
    """
    Demonstrates timeout behavior and monitoring
    """
    print(f"Function timeout configured: {context.get_remaining_time_in_millis() / 1000} seconds (approx)")

    # Get processing time from event (default 5 seconds)
    processing_time = event.get('processingTime', 5)

    print(f"Starting processing that will take {processing_time} seconds")

    # Simulate processing in 1-second intervals
    for i in range(processing_time):
        remaining_ms = context.get_remaining_time_in_millis()
        remaining_sec = remaining_ms / 1000

        print(f"Second {i + 1}/{processing_time} - Time remaining: {remaining_sec:.2f}s")

        # Check if we're running out of time
        if remaining_ms < 2000:  # Less than 2 seconds left
            print("⚠️  WARNING: Running out of time! Saving progress...")
            return {
                'statusCode': 206,
                'body': json.dumps({
                    'message': 'Partial completion',
                    'completed': i + 1,
                    'total': processing_time
                })
            }

        time.sleep(1)

    print("✅ Processing completed successfully!")
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Completed successfully',
            'completed': processing_time
        })
    }
```

**Step 2: Deploy with Different Timeouts**

```bash
# Package function
zip timeout-demo.zip timeout_demo.py

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create function with 10-second timeout
aws lambda create-function \
  --function-name TimeoutDemo \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler timeout_demo.lambda_handler \
  --timeout 10 \
  --zip-file fileb://timeout-demo.zip
```

**Step 3: Test Different Scenarios**

**Test 1: Completes within timeout (5 seconds processing, 10-second timeout)**
```bash
aws lambda invoke \
  --function-name TimeoutDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{"processingTime": 5}' \
  response.json && cat response.json
```

Expected: Success (200)

**Test 2: Graceful handling (8 seconds processing, 10-second timeout)**
```bash
aws lambda invoke \
  --function-name TimeoutDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{"processingTime": 8}' \
  response.json && cat response.json
```

Expected: Partial completion (206) because function detects low time remaining

**Test 3: Actual timeout (15 seconds processing, 10-second timeout)**
```bash
aws lambda invoke \
  --function-name TimeoutDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{"processingTime": 15}' \
  response.json && cat response.json
```

Expected: Timeout error after 10 seconds

**Step 4: Update Timeout and Retry**

```bash
# Increase timeout to 20 seconds
aws lambda update-function-configuration \
  --function-name TimeoutDemo \
  --timeout 20

# Wait for update to complete
sleep 5

# Retry the 15-second processing
aws lambda invoke \
  --function-name TimeoutDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{"processingTime": 15}' \
  response.json && cat response.json
```

Expected: Success (200)

**Step 5: View Execution Metrics in CloudWatch**

```bash
# View logs
aws logs tail /aws/lambda/TimeoutDemo --since 10m

# Get function metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=TimeoutDemo \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**Cleanup:**
```bash
aws lambda delete-function --function-name TimeoutDemo
rm timeout-demo.zip response.json timeout_demo.py
```

**Key Learnings:**
1. Timeout is a hard limit - Lambda forcefully terminates execution
2. Use `context.get_remaining_time_in_millis()` for graceful handling
3. Set timeout with buffer above average execution time
4. Monitor Duration metrics to optimize timeout settings
5. Design functions to handle partial completion

---

## 2. Memory Configuration

### What is Memory in Lambda?

Memory is the amount of RAM allocated to your Lambda function. Unlike traditional servers, **memory also determines CPU power** in Lambda.

### The Critical Concept: Memory = CPU Power

**This is the most important thing to understand about Lambda:**

In traditional computing:
- Memory (RAM) and CPU are independent
- 8 GB RAM server could have 2 cores or 16 cores

In Lambda:
- **Memory and CPU are coupled**
- More memory = proportionally more CPU
- This is unique to Lambda!

### Memory and CPU Relationship

**Key Concept**: More memory = More CPU power

- At **1,769 MB**: Function gets 1 full vCPU
- Below 1,769 MB: Fractional vCPU (proportional)
- Above 1,769 MB: More than 1 vCPU (up to 6 vCPUs at 10,240 MB)

**The Formula:**
```
vCPU allocation = (Memory in MB / 1,769 MB)

Examples:
- 128 MB  → 0.07 vCPU  (128/1769 = 0.072)
- 512 MB  → 0.29 vCPU  (512/1769 = 0.289)
- 1,769 MB → 1.00 vCPU  (1769/1769 = 1.0)
- 3,538 MB → 2.00 vCPU  (3538/1769 = 2.0)
- 10,240 MB → 5.79 vCPU (10240/1769 = 5.79, capped at 6)
```

### Theory: How Lambda Allocates Resources

#### The Execution Environment

When Lambda runs your function, it creates an execution environment:

```
┌─────────────────────────────────────┐
│   Lambda Execution Environment      │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Your Function Code           │ │
│  │  + Dependencies               │ │
│  └───────────────────────────────┘ │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Runtime (Python, Node, etc.) │ │
│  └───────────────────────────────┘ │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Memory: XXXXX MB             │ │
│  │  CPU: X.XX vCPU               │ │ ← Determined by memory!
│  │  /tmp: 512 MB - 10 GB         │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
```

**What gets allocated:**
1. **RAM**: Exact amount you configure
2. **CPU**: Proportional to memory (cannot configure separately!)
3. **Network bandwidth**: Also scales with memory
4. **Disk I/O**: Also scales with memory

### Memory Limits

- **Minimum**: 128 MB
- **Maximum**: 10,240 MB (10 GB)
- **Increment**: 1 MB (you can set any value between min and max)

### Theory: Why More Memory Makes Code Faster

**It's not just about RAM - it's about CPU!**

**Example: Processing 1,000 images**

```python
# Same code, different memory allocations

def lambda_handler(event, context):
    # CPU-intensive: resize 1,000 images
    for image in images:
        resize_image(image)  # CPU-bound operation
```

**With 128 MB (0.07 vCPU):**
```
CPU is the bottleneck (only 7% of a vCPU)
Duration: 60 seconds
Actual memory used: 200 MB (over limit!)
Result: OUT OF MEMORY ERROR ❌
```

**With 512 MB (0.29 vCPU):**
```
CPU still bottleneck (29% of vCPU)
Duration: 20 seconds (3x faster than 128 MB!)
Actual memory used: 200 MB (fits comfortably)
Result: Success, but slow ✅
```

**With 1,769 MB (1.0 vCPU):**
```
Full CPU available
Duration: 6 seconds (10x faster than 128 MB!)
Actual memory used: 200 MB (lots of headroom)
Result: Success, fast ✅
Cost: Slightly higher per GB-second, but lower total due to faster execution
```

**Key Insight:**
- Actual memory used was 200 MB in all cases
- Performance difference was due to **CPU power**, not RAM
- More memory gave more CPU → faster execution → potentially lower cost!

### Setting Memory

**Via AWS CLI:**
```bash
# Create function with 512 MB
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler index.lambda_handler \
  --memory-size 512 \
  --zip-file fileb://function.zip

# Update memory
aws lambda update-function-configuration \
  --function-name MyFunction \
  --memory-size 1024
```

**Via SAM Template:**
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.lambda_handler
      Runtime: python3.9
      MemorySize: 1024  # MB
```

### How Memory Affects Performance

**Example: Image Processing Function**

```python
# Same code, different memory allocations

# 128 MB Memory:
# Duration: 8,500 ms
# Billed Duration: 8,500 ms
# Cost: $0.00000208

# 1,024 MB Memory (8x more):
# Duration: 1,200 ms
# Billed Duration: 1,200 ms
# Cost: $0.00000250

# Result: 7x faster, only 20% more expensive!
```

**Why This Happens:**
- More memory = More CPU
- CPU-intensive tasks complete faster
- Less execution time can offset higher memory cost

### Theory: CPU-Bound vs I/O-Bound Workloads

#### CPU-Bound Workloads

**Definition**: Execution time limited by CPU speed.

**Examples:**
- Image/video processing
- Data compression/decompression
- Cryptographic operations
- Mathematical calculations
- Machine learning inference

**Characteristic**: CPU is at 100% utilization

**Memory Impact**: ⭐⭐⭐⭐⭐ (Very High)
- More memory = More CPU = Much faster
- Worth paying for higher memory

#### I/O-Bound Workloads

**Definition**: Execution time limited by waiting for I/O operations.

**Examples:**
- Database queries
- API calls to external services
- S3 reads/writes
- Network requests
- Disk I/O

**Characteristic**: CPU mostly idle, waiting for responses

**Memory Impact**: ⭐⭐ (Low to Moderate)
- More memory helps slightly (faster network)
- Not worth paying for much higher memory
- CPU sits idle waiting for responses

**Example: API Call Function**

```python
import requests

def lambda_handler(event, context):
    # I/O-bound: waiting for external API
    response = requests.get('https://api.example.com/data')
    return response.json()

# With 128 MB (0.07 vCPU):
# Duration: 2,500 ms (2,000 ms waiting for API, 500 ms processing)

# With 1,769 MB (1.0 vCPU):
# Duration: 2,100 ms (2,000 ms still waiting for API, 100 ms processing)

# Result: 14x more expensive memory, only 16% faster!
# Not worth it for I/O-bound workloads!
```

### Memory Allocation Examples

| Memory | vCPU Power | Use Case | Example Workload |
|--------|-----------|----------|------------------|
| 128 MB | ~0.07 vCPU | Simple tasks | JSON validation, simple API routing |
| 256 MB | ~0.15 vCPU | Light processing | DynamoDB reads/writes, S3 metadata |
| 512 MB | ~0.29 vCPU | Moderate tasks | API integrations, data transformation |
| 1,024 MB | ~0.58 vCPU | Standard workload | File parsing, business logic |
| 1,769 MB | 1 vCPU | CPU-bound tasks | Image resizing, data processing |
| 3,008 MB | ~1.7 vCPU | Heavy processing | Video transcoding, ML inference |
| 10,240 MB | 6 vCPU | Maximum performance | Large data processing, complex computations |

### Theory: Network Performance and Memory

**Lesser-known fact**: Network bandwidth also scales with memory!

```
128 MB   → ~1 Gbps network
512 MB   → ~2 Gbps network
1,769 MB → ~5 Gbps network
3,008 MB → ~7 Gbps network
10,240 MB → ~25 Gbps network (estimated)
```

**Impact on Data Transfer:**

Downloading 1 GB file from S3:
```
128 MB memory:
- Network: ~1 Gbps (125 MB/s)
- Time: ~8 seconds

1,769 MB memory:
- Network: ~5 Gbps (625 MB/s)
- Time: ~1.6 seconds
```

**When this matters:**
- Large file downloads from S3
- High-throughput data streaming
- Database bulk operations
- Multi-region data transfers

### Monitoring Memory Usage

**In Your Code:**
```python
import os
import psutil

def lambda_handler(event, context):
    # Check allocated memory
    allocated_mb = context.memory_limit_in_mb
    print(f"Allocated memory: {allocated_mb} MB")

    # Check actual memory usage (requires psutil layer)
    process = psutil.Process(os.getpid())
    memory_info = process.memory_info()
    used_mb = memory_info.rss / 1024 / 1024

    print(f"Memory used: {used_mb:.2f} MB")
    print(f"Memory available: {allocated_mb - used_mb:.2f} MB")
    print(f"Memory utilization: {(used_mb / allocated_mb) * 100:.1f}%")

    # Do work
    result = process_data(event)

    # Check memory after processing
    memory_info_after = process.memory_info()
    used_mb_after = memory_info_after.rss / 1024 / 1024
    print(f"Memory after processing: {used_mb_after:.2f} MB")

    return result
```

**In CloudWatch Logs (REPORT line):**
```
REPORT RequestId: abc-123
Duration: 1234.56 ms
Billed Duration: 1300 ms
Memory Size: 1024 MB
Max Memory Used: 256 MB
```

**Analyzing the REPORT:**
- **Memory Size**: Allocated (1024 MB)
- **Max Memory Used**: Actual usage (256 MB)
- **Waste**: 768 MB unused (75% waste!)

### Memory Optimization Strategy

**Step 1: Start High, Measure, Reduce**

```bash
# Start with 1024 MB
aws lambda create-function \
  --function-name OptimizeMe \
  --memory-size 1024 \
  ...

# Run tests and check CloudWatch logs for "Max Memory Used"
# If Max Memory Used is consistently 200 MB, you can reduce

# Reduce to 256 MB (add small buffer)
aws lambda update-function-configuration \
  --function-name OptimizeMe \
  --memory-size 256
```

**Step 2: Test Different Memory Sizes**

```bash
# Test script to find optimal memory
for memory in 128 256 512 1024 1536 2048 3008
do
  echo "Testing with ${memory} MB memory..."

  # Update memory
  aws lambda update-function-configuration \
    --function-name OptimizeMe \
    --memory-size $memory

  sleep 5  # Wait for update

  # Invoke and measure
  aws lambda invoke \
    --function-name OptimizeMe \
    --payload '{"test": true}' \
    output.json

  # Check logs for duration
  aws logs tail /aws/lambda/OptimizeMe --since 1m | grep "REPORT"
done
```

**Step 3: Use AWS Lambda Power Tuning**

AWS provides an open-source tool for automated memory optimization:
```bash
# Deploy Lambda Power Tuning (Step Functions-based tool)
# https://github.com/alexcasalboni/aws-lambda-power-tuning

# It automatically:
# - Tests different memory configurations
# - Measures execution time and cost
# - Recommends optimal memory setting
```

### Memory and Cost Relationship

**Pricing Formula:**
```
Cost = (Memory in GB) × (Duration in seconds) × (Price per GB-second)
Price per GB-second = $0.0000166667
```

**Example Scenario: 1 million requests/month**

| Memory | Avg Duration | GB-seconds | Cost @ $0.0000166667/GB-sec |
|--------|-------------|------------|---------------------------|
| 128 MB | 5,000 ms | 625 | $10.42 |
| 256 MB | 2,500 ms | 625 | $10.42 |
| 512 MB | 1,500 ms | 750 | $12.50 |
| 1,024 MB | 1,000 ms | 1,000 | $16.67 |

**Calculation for 128 MB:**
```
Memory in GB: 128 MB / 1024 = 0.125 GB
Duration: 5,000 ms / 1000 = 5 seconds
Requests: 1,000,000

GB-seconds = 0.125 GB × 5 sec × 1,000,000 = 625,000 GB-seconds
Cost = 625,000 × $0.0000166667 = $10.42
```

**Insight**: 256 MB is 2x faster than 128 MB with **same cost** because execution time halves!

### Theory: The Sweet Spot - 1,769 MB Threshold

**Why 1,769 MB is Special:**

This is the memory level where you get exactly **1 full vCPU**.

**Mathematical Relationship:**
```
AWS allocates vCPU linearly with memory:
1 vCPU at 1,769 MB

Below 1,769 MB:
vCPU = (Memory / 1,769)

Example at 512 MB:
vCPU = 512 / 1,769 = 0.289 vCPU (28.9% of a CPU)
```

**Performance Jump:**

For CPU-bound workloads:
```
1,024 MB (0.58 vCPU):
- Duration: 10 seconds
- Cost: $0.000170

1,769 MB (1.0 vCPU):
- Duration: 6 seconds (full CPU!)
- Cost: $0.000177

Result: 67% faster for only 4% more cost!
```

**When to use 1,769 MB:**
- CPU-intensive tasks (image processing, compression, etc.)
- You're hitting CPU limits at lower memory
- Need consistent full CPU performance
- Cost-performance sweet spot for compute-heavy workloads

**When NOT to use 1,769 MB:**
- I/O-bound tasks (API calls, database queries)
- Simple data transformation
- Max memory used is < 500 MB (you're wasting memory)

### Common Memory Patterns

**Pattern 1: Low Memory for I/O-Bound Tasks**
```python
# Network calls, database queries
# Not CPU-intensive, mostly waiting

def lambda_handler(event, context):
    # Memory: 128-256 MB is sufficient
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')

    # Simple read/write operations
    response = table.get_item(Key={'userId': event['userId']})
    return response['Item']
```

**Pattern 2: High Memory for CPU-Bound Tasks**
```python
# Image processing, data transformation
# CPU-intensive operations

def lambda_handler(event, context):
    # Memory: 1,769+ MB recommended
    from PIL import Image
    import io

    # Download image from S3
    s3 = boto3.client('s3')
    image_data = s3.get_object(Bucket=bucket, Key=key)

    # Process image (CPU-intensive)
    image = Image.open(io.BytesIO(image_data['Body'].read()))
    resized = image.resize((800, 600))
    compressed = optimize_image(resized)

    return upload_to_s3(compressed)
```

**Pattern 3: Adaptive Memory Based on Input**
```python
def lambda_handler(event, context):
    """
    Function processes different workload sizes
    Deploy with memory based on expected max workload
    """
    memory = context.memory_limit_in_mb
    items = event['items']

    if memory >= 1024:
        # Parallel processing for high memory
        return parallel_process(items)
    else:
        # Sequential for low memory
        return sequential_process(items)
```

### Out of Memory Error

**Error Message:**
```
Runtime.ExitError: RequestId: abc-123
Error: Runtime exited with error: signal: killed
```

**CloudWatch REPORT:**
```
REPORT RequestId: abc-123
Duration: 3456.78 ms
Memory Size: 256 MB
Max Memory Used: 256 MB  ← Used ALL allocated memory!
```

**What happened:**
1. Function tried to allocate more than 256 MB
2. Linux kernel OOM (Out Of Memory) killer terminated the process
3. Function crashes with "signal: killed"

**Solution:**
```bash
# Increase memory
aws lambda update-function-configuration \
  --function-name MyFunction \
  --memory-size 512  # Double the memory
```

**Code-Level Prevention:**
```python
def lambda_handler(event, context):
    import sys

    # Check memory limit
    allocated_mb = context.memory_limit_in_mb

    # Estimate required memory based on input
    items_count = len(event['items'])
    estimated_mb = items_count * 0.5  # 0.5 MB per item

    if estimated_mb > allocated_mb * 0.8:  # Use 80% threshold
        return {
            'statusCode': 413,  # Payload Too Large
            'body': f'Input requires ~{estimated_mb}MB, only {allocated_mb}MB available'
        }

    # Proceed with processing
    return process_items(event['items'])
```

### Exercise: Memory Optimization

**Objective**: Understand how memory affects performance and cost.

**Step 1: Create a CPU-Intensive Function**

Create `memory_test.py`:
```python
import json
import time
import hashlib

def lambda_handler(event, context):
    """
    CPU-intensive function to test memory/performance relationship
    """
    print(f"Allocated memory: {context.memory_limit_in_mb} MB")

    start_time = time.time()

    # CPU-intensive task: compute 100,000 hashes
    iterations = event.get('iterations', 100000)

    for i in range(iterations):
        # Generate hash (CPU-bound operation)
        data = f"iteration-{i}".encode('utf-8')
        hashlib.sha256(data).hexdigest()

    end_time = time.time()
    duration_ms = (end_time - start_time) * 1000

    print(f"Completed {iterations} iterations in {duration_ms:.2f}ms")

    return {
        'statusCode': 200,
        'body': json.dumps({
            'iterations': iterations,
            'durationMs': duration_ms,
            'memoryMB': context.memory_limit_in_mb
        })
    }
```

**Step 2: Deploy Function**

```bash
# Package
zip memory-test.zip memory_test.py

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create function with 128 MB (minimum)
aws lambda create-function \
  --function-name MemoryTest \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler memory_test.lambda_handler \
  --memory-size 128 \
  --timeout 60 \
  --zip-file fileb://memory-test.zip
```

**Step 3: Test Different Memory Configurations**

```bash
# Test with different memory sizes
for memory in 128 256 512 1024 1536 2048 3008
do
  echo "========================================="
  echo "Testing with ${memory} MB memory"
  echo "========================================="

  # Update memory
  aws lambda update-function-configuration \
    --function-name MemoryTest \
    --memory-size $memory > /dev/null

  # Wait for update
  sleep 5

  # Invoke function
  aws lambda invoke \
    --function-name MemoryTest \
    --cli-binary-format raw-in-base64-out \
    --payload '{"iterations": 100000}' \
    response.json

  # Show result
  cat response.json | python3 -m json.tool

  # Get last log entry (REPORT line)
  sleep 2
  aws logs tail /aws/lambda/MemoryTest --since 30s --format short | grep "REPORT" | tail -1

  echo ""
done
```

**Step 4: Analyze Results**

You'll see output like:
```
Testing with 128 MB memory
{
    "statusCode": 200,
    "body": "{\"iterations\": 100000, \"durationMs\": 4523.45, \"memoryMB\": 128}"
}
REPORT Duration: 4567.89 ms  Billed Duration: 4568 ms  Memory Size: 128 MB  Max Memory Used: 38 MB

Testing with 1024 MB memory
{
    "statusCode": 200,
    "body": "{\"iterations\": 100000, \"durationMs\": 892.34, \"memoryMB\": 1024}"
}
REPORT Duration: 934.56 ms  Billed Duration: 935 ms  Memory Size: 1024 MB  Max Memory Used: 39 MB
```

**Analysis:**
- 128 MB: ~4,500ms duration, uses only 38 MB actual memory
- 1,024 MB: ~900ms duration (5x faster!), same 39 MB actual memory
- **More memory = More CPU = Faster execution**
- **Actual memory used is the same** (38-39 MB)

**Step 5: Calculate Cost Comparison**

```bash
# For 1 million invocations:

# 128 MB, 4,500ms average
# GB-seconds: 0.128 GB × 4.5 sec × 1,000,000 = 576,000
# Cost: 576,000 × $0.0000166667 = $9.60

# 1,024 MB, 900ms average
# GB-seconds: 1.024 GB × 0.9 sec × 1,000,000 = 921,600
# Cost: 921,600 × $0.0000166667 = $15.36

# Trade-off: 5x faster but 60% more expensive
```

**Cleanup:**
```bash
aws lambda delete-function --function-name MemoryTest
rm memory-test.zip response.json memory_test.py
```

**Key Learnings:**
1. More memory = More CPU power (not just RAM)
2. CPU-bound tasks benefit significantly from higher memory
3. I/O-bound tasks don't benefit as much from extra memory
4. Monitor "Max Memory Used" to avoid over-provisioning
5. Find balance between performance and cost

### Best Practices for Memory Configuration

**1. Right-Size Memory**
- Start with 512-1024 MB for testing
- Monitor "Max Memory Used" in CloudWatch logs
- Reduce to actual usage + 20% buffer

**2. Consider CPU Requirements**
- CPU-intensive: Use 1,769 MB+ for full vCPU
- I/O-intensive: Use 256-512 MB (waiting doesn't need CPU)

**3. Test and Optimize**
- Use Lambda Power Tuning tool
- Test with realistic workloads
- Monitor both duration and cost

**4. Watch for These Signals**

Over-provisioned (too much memory):
```
Memory Size: 3008 MB
Max Memory Used: 128 MB  ← Using only 4%!
```

Under-provisioned (too little memory):
```
Memory Size: 256 MB
Max Memory Used: 256 MB  ← Using 100%! Risk of OOM
```

Optimal:
```
Memory Size: 512 MB
Max Memory Used: 380 MB  ← Using 74%, good buffer
```

---

## 3. Environment Variables

### What are Environment Variables?

**Environment variables** are key-value pairs that you can configure for your Lambda function. They allow you to pass configuration data to your code **without hardcoding values**.

### Theory: Why Environment Variables Exist

#### The Configuration Problem

In software development, you need different configurations for:
- **Environments**: Dev, staging, production
- **Secrets**: API keys, database passwords
- **Resources**: Bucket names, table names, endpoints
- **Behavior**: Feature flags, debug modes

**Traditional approach (hardcoded):**
```python
# ❌ Problems:
# 1. Need separate code for dev/prod
# 2. Secrets in source code (security risk!)
# 3. Can't change config without redeploying code
# 4. Can't reuse same code across environments

def lambda_handler(event, context):
    bucket = "prod-bucket-12345"  # What about dev?
    api_key = "sk_live_abc123"     # Exposed in code!
    db_host = "prod-db.aws.com"    # Hardcoded!
```

**Solution: Environment Variables:**
```python
# ✅ Benefits:
# 1. Same code for all environments
# 2. Secrets not in source code
# 3. Change config without redeploying
# 4. Reusable, testable code

import os

def lambda_handler(event, context):
    bucket = os.environ['BUCKET_NAME']   # Set per environment
    api_key = os.environ['API_KEY']      # Encrypted, not in code
    db_host = os.environ['DB_HOST']      # Configurable
```

### Theory: How Environment Variables Work in Lambda

#### Loading Process

```
1. Lambda Function Deployed
   ↓
2. Environment Variables Stored (encrypted at rest)
   ↓
3. Function Invoked (first time or cold start)
   ↓
4. Execution Environment Created
   ↓
5. Environment Variables Decrypted and Loaded into Process
   ↓
6. Your Code Runs (os.environ available)
   ↓
7. [Warm invoke] Environment Reused → Variables Already Loaded (fast!)
```

**Key Points:**
- Variables are loaded **once** per execution environment (cold start)
- Warm invocations reuse the same environment → same variables
- Variables are **NOT** re-read on every invocation (for performance)

#### Caching Behavior

```python
import os

# This runs ONCE per execution environment (cold start)
BUCKET_NAME = os.environ['BUCKET_NAME']

def lambda_handler(event, context):
    # This runs on EVERY invocation
    # BUCKET_NAME is already loaded (from cold start)
    return upload_to_bucket(BUCKET_NAME)
```

**Implication:**
- If you update environment variables, existing warm containers keep old values
- New invocations may get new execution environments with new values
- Can take minutes for all environments to refresh

### Why Use Environment Variables?

**Without Environment Variables:**
```python
# ❌ BAD: Hardcoded values
def lambda_handler(event, context):
    bucket_name = "my-production-bucket"  # What about dev/staging?
    api_key = "sk_prod_12345"  # Security risk!
    region = "us-east-1"  # Can't change without redeploying

    s3 = boto3.client('s3', region_name=region)
    return s3.get_object(Bucket=bucket_name, Key='data.json')
```

**With Environment Variables:**
```python
# ✅ GOOD: Configurable via environment variables
import os

def lambda_handler(event, context):
    bucket_name = os.environ['BUCKET_NAME']  # Set per environment
    api_key = os.environ['API_KEY']  # Encrypted at rest
    region = os.environ.get('AWS_REGION', 'us-east-1')  # With default

    s3 = boto3.client('s3', region_name=region)
    return s3.get_object(Bucket=bucket_name, Key='data.json')
```

### Setting Environment Variables

**Via AWS CLI:**
```bash
# Create function with environment variables
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler index.lambda_handler \
  --environment "Variables={BUCKET_NAME=my-bucket,TABLE_NAME=my-table,STAGE=prod}" \
  --zip-file fileb://function.zip

# Update environment variables
aws lambda update-function-configuration \
  --function-name MyFunction \
  --environment "Variables={BUCKET_NAME=new-bucket,TABLE_NAME=my-table,STAGE=prod,DEBUG=true}"
```

**Via AWS Console:**
1. Go to Lambda function
2. Configuration tab → Environment variables
3. Click "Edit"
4. Add key-value pairs
5. Save

**Via SAM Template:**
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.lambda_handler
      Runtime: python3.9
      Environment:
        Variables:
          BUCKET_NAME: my-data-bucket
          TABLE_NAME: users-table
          STAGE: prod
          API_ENDPOINT: https://api.example.com
```

**Via Terraform:**
```hcl
resource "aws_lambda_function" "my_function" {
  function_name = "MyFunction"
  runtime       = "python3.9"
  handler       = "index.lambda_handler"

  environment {
    variables = {
      BUCKET_NAME   = "my-data-bucket"
      TABLE_NAME    = "users-table"
      STAGE         = "prod"
      API_ENDPOINT  = "https://api.example.com"
    }
  }
}
```

### Accessing Environment Variables

**Python:**
```python
import os

def lambda_handler(event, context):
    # Get environment variable (raises KeyError if not set)
    bucket_name = os.environ['BUCKET_NAME']

    # Get with default value (recommended)
    table_name = os.environ.get('TABLE_NAME', 'default-table')

    # Check if variable exists
    if 'DEBUG' in os.environ:
        print(f"Debug mode enabled: {os.environ['DEBUG']}")

    # Get all environment variables
    print(f"All env vars: {dict(os.environ)}")

    return {'statusCode': 200}
```

**Node.js:**
```javascript
exports.handler = async (event) => {
    // Get environment variable
    const bucketName = process.env.BUCKET_NAME;

    // Get with default value
    const tableName = process.env.TABLE_NAME || 'default-table';

    // Check if variable exists
    if (process.env.DEBUG) {
        console.log(`Debug mode: ${process.env.DEBUG}`);
    }

    return { statusCode: 200 };
};
```

**Java:**
```java
public String handleRequest(Map<String, String> event, Context context) {
    // Get environment variable
    String bucketName = System.getenv("BUCKET_NAME");

    // Get with default
    String tableName = System.getenv().getOrDefault("TABLE_NAME", "default-table");

    return "Success";
}
```

### Environment Variable Limits

- **Total size**: 4 KB (all keys + values combined)
- **No limit on number** of variables (as long as total size < 4 KB)
- **Key naming**: Letters, numbers, underscores (case-sensitive)

**Size Calculation:**
```python
# Example environment variables:
BUCKET_NAME=my-production-bucket        # 15 + 21 = 36 bytes
TABLE_NAME=users-dynamodb-table         # 10 + 23 = 33 bytes
API_KEY=sk_live_51H...                   # 7 + 107 = 114 bytes

# Total: 183 bytes (well under 4 KB limit)
```

**What if you exceed 4 KB?**
```bash
# Error when trying to set too many variables:
InvalidParameterValueException:
Environment variables has exceeded the maximum allowed size of 4096 bytes
```

**Solution for large configuration:**
- Use **AWS Systems Manager Parameter Store**
- Use **AWS Secrets Manager**
- Store configuration in **S3** and load at runtime

### Theory: Reserved Environment Variables

AWS automatically sets these (read-only) for every Lambda function:

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `AWS_REGION` | AWS region where function runs | `us-east-1` |
| `AWS_EXECUTION_ENV` | Runtime identifier | `AWS_Lambda_python3.9` |
| `AWS_LAMBDA_FUNCTION_NAME` | Function name | `MyFunction` |
| `AWS_LAMBDA_FUNCTION_VERSION` | Function version | `$LATEST` or `1` |
| `AWS_LAMBDA_FUNCTION_MEMORY_SIZE` | Allocated memory (MB) | `512` |
| `AWS_LAMBDA_LOG_GROUP_NAME` | CloudWatch log group | `/aws/lambda/MyFunction` |
| `AWS_LAMBDA_LOG_STREAM_NAME` | CloudWatch log stream | `2023/11/15/[$LATEST]abc123` |
| `LAMBDA_TASK_ROOT` | Function code directory | `/var/task` |
| `LAMBDA_RUNTIME_DIR` | Runtime directory | `/var/runtime` |
| `TZ` | Timezone | `UTC` (always UTC) |

**Using Reserved Variables:**
```python
import os

def lambda_handler(event, context):
    # These are automatically available
    region = os.environ['AWS_REGION']
    function_name = os.environ['AWS_LAMBDA_FUNCTION_NAME']
    memory = os.environ['AWS_LAMBDA_FUNCTION_MEMORY_SIZE']

    print(f"Running {function_name} in {region} with {memory}MB memory")

    return {'statusCode': 200}
```

### Common Use Cases

**1. Multi-Environment Configuration**
```python
import os
import boto3

def lambda_handler(event, context):
    stage = os.environ.get('STAGE', 'dev')

    # Different resources per environment
    if stage == 'prod':
        bucket = os.environ['PROD_BUCKET']
        table = os.environ['PROD_TABLE']
    elif stage == 'staging':
        bucket = os.environ['STAGING_BUCKET']
        table = os.environ['STAGING_TABLE']
    else:
        bucket = os.environ['DEV_BUCKET']
        table = os.environ['DEV_TABLE']

    # Use environment-specific resources
    dynamodb = boto3.resource('dynamodb')
    db_table = dynamodb.Table(table)

    return {'statusCode': 200, 'environment': stage}
```

**2. Feature Flags**
```python
import os

def lambda_handler(event, context):
    # Enable/disable features without code changes
    enable_new_feature = os.environ.get('ENABLE_NEW_FEATURE', 'false') == 'true'
    enable_debug_logging = os.environ.get('DEBUG', 'false') == 'true'

    if enable_debug_logging:
        print(f"Debug: Processing event {event}")

    if enable_new_feature:
        return process_with_new_logic(event)
    else:
        return process_with_old_logic(event)
```

**3. External Service Configuration**
```python
import os
import requests

def lambda_handler(event, context):
    # External API configuration
    api_endpoint = os.environ['API_ENDPOINT']
    api_key = os.environ['API_KEY']
    timeout = int(os.environ.get('API_TIMEOUT', '10'))

    response = requests.post(
        api_endpoint,
        headers={'Authorization': f'Bearer {api_key}'},
        json=event,
        timeout=timeout
    )

    return {'statusCode': 200, 'data': response.json()}
```

**4. Database Connection Strings**
```python
import os
import psycopg2

def lambda_handler(event, context):
    # Database configuration
    db_host = os.environ['DB_HOST']
    db_port = os.environ.get('DB_PORT', '5432')
    db_name = os.environ['DB_NAME']
    db_user = os.environ['DB_USER']
    db_password = os.environ['DB_PASSWORD']

    conn = psycopg2.connect(
        host=db_host,
        port=db_port,
        database=db_name,
        user=db_user,
        password=db_password
    )

    # Query database
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (event['userId'],))
    result = cursor.fetchone()

    conn.close()
    return {'statusCode': 200, 'user': result}
```

### Theory: Encryption at Rest

**Default Encryption:**

Lambda encrypts all environment variables at rest using **AWS-managed KMS keys**.

```
Your Environment Variables (plaintext when you set them)
    ↓
Lambda Console / API
    ↓
Encrypted with AWS-managed KMS key (automatic)
    ↓
Stored encrypted in Lambda service
    ↓
Decrypted when execution environment starts
    ↓
Available to your code as plaintext (os.environ)
```

**Custom Encryption (Customer-managed KMS key):**

For extra security/control:
```bash
# Create KMS key first
KMS_KEY_ID=$(aws kms create-key --description "Lambda env var encryption" \
  --query 'KeyMetadata.KeyId' --output text)

# Create function with custom KMS key
aws lambda create-function \
  --function-name MyFunction \
  --environment "Variables={API_KEY=secret123}" \
  --kms-key-arn arn:aws:kms:us-east-1:123456789012:key/${KMS_KEY_ID} \
  ...
```

**Benefits of customer-managed keys:**
- Audit who accesses your keys (CloudTrail)
- Control key rotation policy
- Can revoke access to environment variables
- Compliance requirements

**Decrypting in Code (for sensitive values):**

For extra security, encrypt values before setting as env vars:
```python
import os
import boto3
import base64

# Encrypt sensitive value before deployment
kms = boto3.client('kms')

def lambda_handler(event, context):
    # Get encrypted environment variable
    encrypted_api_key = os.environ['ENCRYPTED_API_KEY']

    # Decrypt using KMS
    decrypted = kms.decrypt(
        CiphertextBlob=base64.b64decode(encrypted_api_key)
    )

    api_key = decrypted['Plaintext'].decode('utf-8')

    # Use decrypted value
    return call_api_with_key(api_key)
```

### Theory: Environment Variables vs Other Configuration Methods

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **Environment Variables** | ✅ Fast (no API calls)<br>✅ Simple<br>✅ Built-in | ❌ 4 KB limit<br>❌ Not ideal for secrets<br>❌ Manual updates | Non-secret config, flags, resource names |
| **Secrets Manager** | ✅ Automatic rotation<br>✅ Versioning<br>✅ Audit trail | ❌ API call (latency)<br>❌ Costs money | Database passwords, API keys, certificates |
| **Parameter Store** | ✅ Hierarchical<br>✅ Free tier<br>✅ Versioning | ❌ API call (latency)<br>❌ 8 KB limit (standard) | App config, connection strings |
| **S3** | ✅ Unlimited size<br>✅ Versioning | ❌ API call (latency)<br>❌ More complex | Large config files, JSON/YAML configs |
| **Hardcoded** | ✅ Fastest | ❌ Can't change without redeploy<br>❌ Security risk | Never (avoid!) |

**Recommendation:**
```python
# Use combination based on needs:

import os
import boto3
import json

def lambda_handler(event, context):
    # 1. Environment Variables: Simple config
    stage = os.environ['STAGE']
    bucket_name = os.environ['BUCKET_NAME']

    # 2. Secrets Manager: Sensitive data
    secrets_client = boto3.client('secretsmanager')
    db_secret = secrets_client.get_secret_value(
        SecretId=os.environ['DB_SECRET_NAME']
    )
    db_password = json.loads(db_secret['SecretString'])['password']

    # 3. Parameter Store: App configuration
    ssm = boto3.client('ssm')
    app_config = ssm.get_parameter(
        Name=f'/{stage}/app/config',
        WithDecryption=True
    )
    config = json.loads(app_config['Parameter']['Value'])

    # Use all three together!
    return process_request(bucket_name, db_password, config)
```

### Best Practices

**1. Use AWS Secrets Manager for Secrets**
```python
# ❌ Don't store sensitive data in environment variables
# API_KEY=sk_live_abc123  (visible in console, logs)

# ✅ Use Secrets Manager
import os
import boto3
import json

def lambda_handler(event, context):
    secrets_client = boto3.client('secretsmanager')
    secret_name = os.environ['SECRET_NAME']  # Just the secret name

    # Retrieve actual secret from Secrets Manager
    secret_value = secrets_client.get_secret_value(SecretId=secret_name)
    secret = json.loads(secret_value['SecretString'])

    api_key = secret['api_key']  # Securely retrieved
    return call_api(api_key)
```

**2. Use Parameter Store for Configuration**
```python
import os
import boto3

def lambda_handler(event, context):
    ssm = boto3.client('ssm')

    # Get parameter path from environment variable
    param_path = os.environ['CONFIG_PATH']

    # Retrieve configuration from Parameter Store
    response = ssm.get_parameters_by_path(
        Path=param_path,
        Recursive=True,
        WithDecryption=True
    )

    config = {p['Name']: p['Value'] for p in response['Parameters']}
    return config
```

**3. Naming Conventions**
```bash
# Use descriptive, uppercase names with underscores
GOOD:
  - BUCKET_NAME
  - API_ENDPOINT
  - DATABASE_HOST
  - ENABLE_DEBUG
  - MAX_RETRY_COUNT

BAD:
  - bucket (too generic)
  - apiEndpoint (not conventional for env vars)
  - db (too short)
```

**4. Provide Defaults for Optional Settings**
```python
import os

def lambda_handler(event, context):
    # Required (will raise error if not set)
    bucket_name = os.environ['BUCKET_NAME']

    # Optional with sensible defaults
    max_retries = int(os.environ.get('MAX_RETRIES', '3'))
    timeout_seconds = int(os.environ.get('TIMEOUT', '30'))
    enable_cache = os.environ.get('ENABLE_CACHE', 'true') == 'true'
    log_level = os.environ.get('LOG_LEVEL', 'INFO')

    return process_data(bucket_name, max_retries, timeout_seconds)
```

**5. Validate Environment Variables at Startup**
```python
import os

# Validate required environment variables at module load time
REQUIRED_ENV_VARS = ['BUCKET_NAME', 'TABLE_NAME', 'API_ENDPOINT']

for var in REQUIRED_ENV_VARS:
    if var not in os.environ:
        raise ValueError(f"Missing required environment variable: {var}")

def lambda_handler(event, context):
    # Environment variables validated before handler runs
    bucket = os.environ['BUCKET_NAME']
    return process_request(bucket)
```

### Exercise: Environment Variables Configuration

**Objective**: Learn to use environment variables for configuration management.

**Step 1: Create a Configurable Function**

Create `env_config.py`:
```python
import os
import json
import boto3

def lambda_handler(event, context):
    """
    Demonstrates environment variable usage
    """
    print("=== Environment Configuration ===")

    # Required variables
    try:
        bucket_name = os.environ['BUCKET_NAME']
        table_name = os.environ['TABLE_NAME']
        print(f"✅ Bucket: {bucket_name}")
        print(f"✅ Table: {table_name}")
    except KeyError as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': f'Missing required env var: {e}'})
        }

    # Optional variables with defaults
    stage = os.environ.get('STAGE', 'dev')
    debug = os.environ.get('DEBUG', 'false') == 'true'
    max_items = int(os.environ.get('MAX_ITEMS', '10'))

    print(f"Stage: {stage}")
    print(f"Debug: {debug}")
    print(f"Max items: {max_items}")

    # AWS-provided variables
    region = os.environ['AWS_REGION']
    function_name = os.environ['AWS_LAMBDA_FUNCTION_NAME']
    memory = os.environ['AWS_LAMBDA_FUNCTION_MEMORY_SIZE']

    print(f"\nAWS Environment:")
    print(f"- Region: {region}")
    print(f"- Function: {function_name}")
    print(f"- Memory: {memory}MB")

    # Feature flags
    enable_caching = os.environ.get('ENABLE_CACHING', 'false') == 'true'
    if enable_caching:
        print("🔥 Caching enabled!")

    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Configuration loaded successfully',
            'config': {
                'bucket': bucket_name,
                'table': table_name,
                'stage': stage,
                'debug': debug,
                'maxItems': max_items,
                'region': region,
                'caching': enable_caching
            }
        })
    }
```

**Step 2: Deploy Without Required Variables (to see error)**

```bash
# Package
zip env-config.zip env_config.py

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create function WITHOUT environment variables
aws lambda create-function \
  --function-name EnvConfigDemo \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler env_config.lambda_handler \
  --zip-file fileb://env-config.zip

# Test (will fail - missing env vars)
aws lambda invoke \
  --function-name EnvConfigDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json && cat response.json
```

Expected error:
```json
{
    "statusCode": 500,
    "body": "{\"error\": \"Missing required env var: 'BUCKET_NAME'\"}"
}
```

**Step 3: Add Environment Variables**

```bash
# Update with required environment variables
aws lambda update-function-configuration \
  --function-name EnvConfigDemo \
  --environment "Variables={BUCKET_NAME=my-data-bucket,TABLE_NAME=users-table,STAGE=dev}"

# Wait for update
sleep 5

# Test again (should succeed)
aws lambda invoke \
  --function-name EnvConfigDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json && cat response.json
```

Expected success:
```json
{
    "statusCode": 200,
    "body": "{\"message\": \"Configuration loaded successfully\", \"config\": {...}}"
}
```

**Step 4: Add Optional Variables and Feature Flags**

```bash
# Add more configuration
aws lambda update-function-configuration \
  --function-name EnvConfigDemo \
  --environment "Variables={BUCKET_NAME=my-data-bucket,TABLE_NAME=users-table,STAGE=prod,DEBUG=true,MAX_ITEMS=50,ENABLE_CACHING=true}"

# Wait and test
sleep 5
aws lambda invoke \
  --function-name EnvConfigDemo \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json && cat response.json

# View logs to see all configuration
aws logs tail /aws/lambda/EnvConfigDemo --since 1m
```

**Step 5: Simulate Different Environments**

```bash
# Development environment
aws lambda update-function-configuration \
  --function-name EnvConfigDemo \
  --environment "Variables={BUCKET_NAME=dev-bucket,TABLE_NAME=dev-table,STAGE=dev,DEBUG=true,ENABLE_CACHING=false}"

sleep 5
aws lambda invoke --function-name EnvConfigDemo --payload '{}' response.json
echo "Dev config:" && cat response.json

# Production environment
aws lambda update-function-configuration \
  --function-name EnvConfigDemo \
  --environment "Variables={BUCKET_NAME=prod-bucket,TABLE_NAME=prod-table,STAGE=prod,DEBUG=false,ENABLE_CACHING=true,MAX_ITEMS=100}"

sleep 5
aws lambda invoke --function-name EnvConfigDemo --payload '{}' response.json
echo "Prod config:" && cat response.json
```

**Cleanup:**
```bash
aws lambda delete-function --function-name EnvConfigDemo
rm env-config.zip response.json env_config.py
```

**Key Learnings:**
1. Environment variables enable configuration without code changes
2. Use required vs optional variables with defaults
3. Validate required variables at startup
4. Use feature flags to enable/disable functionality
5. Configure different environments (dev/staging/prod)
6. AWS automatically provides useful environment variables

---

## Summary

### Timeout
- **Default**: 3 seconds | **Max**: 15 minutes
- Controls maximum execution time before forced termination
- Use `context.get_remaining_time_in_millis()` for graceful handling
- Set timeout = average execution + buffer (20-50%)
- Synchronous invocations: Consider API Gateway's 29s limit
- Asynchronous invocations: Affects retry behavior
- Design for timeout with checkpointing and chunked processing

### Memory
- **Range**: 128 MB to 10,240 MB (10 GB)
- **Critical**: More memory = More CPU power (coupled, not independent!)
- **1,769 MB = 1 full vCPU** (sweet spot for CPU-bound tasks)
- CPU-bound workloads benefit greatly from higher memory
- I/O-bound workloads don't benefit much from extra memory
- Network bandwidth also scales with memory
- Monitor "Max Memory Used" in CloudWatch
- Right-size to actual usage + 20% buffer
- Test different memory sizes to find cost-performance balance

### Environment Variables
- **Limit**: 4 KB total size (all keys + values)
- Key-value configuration without hardcoding
- Encrypted at rest by default (AWS-managed or customer-managed KMS)
- Loaded once per execution environment (cold start)
- Use for: config, feature flags, resource names, non-sensitive settings
- For secrets: Use Secrets Manager or Parameter Store
- AWS provides reserved variables (region, function name, memory, etc.)
- Validate required variables at startup
- Use defaults for optional variables
- Naming convention: UPPERCASE_WITH_UNDERSCORES

### Best Practices Summary

**Timeout:**
1. Monitor execution duration and set realistic limits
2. Add 20-50% buffer above average duration
3. Use context for graceful timeout handling
4. Design functions to handle partial completion
5. Chain timeouts properly (each layer shorter than previous)

**Memory:**
1. Identify if workload is CPU-bound or I/O-bound
2. For CPU-intensive tasks: Consider 1,769 MB for full vCPU
3. For I/O-bound tasks: Use minimal memory (256-512 MB)
4. Monitor actual memory usage, reduce over-provisioning
5. Test multiple memory settings to optimize cost vs performance
6. Use Lambda Power Tuning for automated optimization

**Environment Variables:**
1. Never hardcode secrets - use Secrets Manager
2. Use environment variables for non-sensitive configuration
3. Provide sensible defaults for optional settings
4. Validate required variables at module load time
5. Use consistent naming conventions
6. For large config: Use Parameter Store or S3

---

These three configuration options give you fine-grained control over Lambda function behavior, performance, cost, and security!
