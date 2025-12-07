# AWS Lambda - Advanced: Deep Dive

Comprehensive explanation of Lambda advanced features covering layers, versions, aliases, and concurrency.

---

## 1. Lambda Layers

### What are Lambda Layers?

**Lambda Layers** are a distribution mechanism for libraries, custom runtimes, and other function dependencies. They allow you to manage your function's dependencies separately from your function code.

### Theory: The Dependency Problem

#### Without Layers

**Traditional approach:**
```
Your Function Package:
├── index.py (your code - 5 KB)
├── requests/ (HTTP library - 500 KB)
├── pandas/ (data processing - 50 MB)
├── numpy/ (numerical computing - 30 MB)
└── other dependencies...

Total: ~80 MB deployment package
```

**Problems:**
1. **Large deployment packages**: Slower uploads, slower cold starts
2. **Duplicate dependencies**: 10 functions using pandas = 10 copies of pandas (500 MB!)
3. **Harder updates**: Update library = redeploy ALL functions
4. **Package size limits**: Max 50 MB zipped, 250 MB unzipped
5. **Version conflicts**: Different functions need different library versions

#### With Layers

**Layer-based approach:**
```
Layer 1 (shared across all functions):
├── requests/ (500 KB)
├── pandas/ (50 MB)
└── numpy/ (30 MB)
Total: ~80 MB (uploaded ONCE)

Function 1 Package:
└── function1.py (5 KB only!)

Function 2 Package:
└── function2.py (8 KB only!)

Function 3 Package:
└── function3.py (3 KB only!)
```

**Benefits:**
1. **Small deployment packages**: Only your code, not dependencies
2. **Shared dependencies**: One layer used by multiple functions
3. **Easy updates**: Update layer = all functions get new version
4. **Separation of concerns**: Dev focuses on code, ops manages dependencies
5. **Faster deployments**: Upload 5 KB vs 80 MB

### Theory: How Layers Work

#### Layer Architecture

When Lambda runs your function:

```
┌─────────────────────────────────────────┐
│   Lambda Execution Environment          │
│                                          │
│  /opt/  ← Layers mounted here           │
│    ├── python/                           │
│    │   └── lib/python3.9/site-packages/ │
│    │       ├── requests/ (Layer 1)       │
│    │       └── pandas/ (Layer 2)         │
│    └── nodejs/node_modules/             │
│        └── axios/ (Layer 3)              │
│                                          │
│  /var/task/  ← Your function code here  │
│    └── index.py                          │
│                                          │
└─────────────────────────────────────────┘
```

**Loading Order:**
1. Lambda creates execution environment
2. Mounts layers to `/opt/` (in order you specify)
3. Extracts your function code to `/var/task/`
4. Runs your function (imports from both locations)

#### Layer Path Resolution

**Python example:**
```python
# Your function code
import requests  # Looks in:
                 # 1. /var/task/ (your code directory) - NOT FOUND
                 # 2. /opt/python/lib/python3.9/site-packages/ (layers) - FOUND!

# Import works because layer provides it
response = requests.get('https://api.example.com')
```

**Node.js example:**
```javascript
// Your function code
const axios = require('axios');  // Looks in:
                                  // 1. /var/task/node_modules/ - NOT FOUND
                                  // 2. /opt/nodejs/node_modules/ (layers) - FOUND!

// Import works because layer provides it
const response = await axios.get('https://api.example.com');
```

### Layer Limits and Rules

| Limit | Value |
|-------|-------|
| **Max layers per function** | 5 |
| **Total unzipped size** | 250 MB (function + all layers) |
| **Layer versions** | 100 per layer (can delete old versions) |
| **Layer size** | 50 MB (zipped), 250 MB (unzipped) |

**Important Rules:**
1. Layers are **versioned** and **immutable** (can't change published layer)
2. Layers are **region-specific** (must publish per region)
3. Compatible runtimes must be specified
4. Later layers override earlier layers (if same file path)

### Creating a Lambda Layer

#### Layer Directory Structure

**Python Layer:**
```
my-layer/
└── python/
    └── lib/
        └── python3.9/
            └── site-packages/
                ├── requests/
                ├── urllib3/
                └── certifi/
```

**Node.js Layer:**
```
my-layer/
└── nodejs/
    └── node_modules/
        ├── axios/
        ├── lodash/
        └── moment/
```

**Custom binaries/executables:**
```
my-layer/
└── bin/
    ├── ffmpeg
    └── imagemagick
```

#### Step-by-Step: Create Python Layer

**Step 1: Create directory structure**
```bash
mkdir -p my-layer/python/lib/python3.9/site-packages
cd my-layer
```

**Step 2: Install dependencies into layer**
```bash
# Install packages into the layer directory
pip install requests -t python/lib/python3.9/site-packages/

# Or from requirements.txt
pip install -r requirements.txt -t python/lib/python3.9/site-packages/
```

**Step 3: Package layer**
```bash
# Create zip (from my-layer directory)
zip -r my-layer.zip python/
```

**Step 4: Publish layer**
```bash
aws lambda publish-layer-version \
  --layer-name my-dependencies-layer \
  --description "Requests library for HTTP calls" \
  --zip-file fileb://my-layer.zip \
  --compatible-runtimes python3.9 python3.10 python3.11
```

**Response:**
```json
{
    "LayerArn": "arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer",
    "LayerVersionArn": "arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer:1",
    "Version": 1,
    "CompatibleRuntimes": ["python3.9", "python3.10", "python3.11"]
}
```

### Using Layers in Functions

**Method 1: AWS CLI**
```bash
# Attach layer when creating function
aws lambda create-function \
  --function-name MyFunction \
  --runtime python3.9 \
  --handler index.lambda_handler \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --zip-file fileb://function.zip \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer:1

# Add layer to existing function
aws lambda update-function-configuration \
  --function-name MyFunction \
  --layers \
    arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer:1 \
    arn:aws:lambda:us-east-1:123456789012:layer:another-layer:2
```

**Method 2: SAM Template**
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.lambda_handler
      Runtime: python3.9
      Layers:
        - arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer:1
        - arn:aws:lambda:us-east-1:123456789012:layer:another-layer:2
```

**Your function code:**
```python
import json
import requests  # Provided by layer!

def lambda_handler(event, context):
    # Use library from layer
    response = requests.get('https://api.example.com/data')
    return {
        'statusCode': 200,
        'body': json.dumps(response.json())
    }
```

### Layer Versioning and Updates

**Publishing new version:**
```bash
# Update dependencies
pip install requests==2.28.0 -t python/lib/python3.9/site-packages/

# Create new zip
zip -r my-layer-v2.zip python/

# Publish NEW VERSION (version 2)
aws lambda publish-layer-version \
  --layer-name my-dependencies-layer \
  --zip-file fileb://my-layer-v2.zip \
  --compatible-runtimes python3.9
```

**Result:**
- Version 1: Still exists, unchanged
- Version 2: New version created
- Functions using v1: Still use v1 (no automatic update!)
- Functions using v2: Get new dependencies

**Updating functions to new layer version:**
```bash
# Update function to use layer version 2
aws lambda update-function-configuration \
  --function-name MyFunction \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies-layer:2
```

### Common Layer Use Cases

**1. Shared Dependencies Layer**
```bash
# One layer, many functions
Layer: common-dependencies
  - requests
  - boto3 (updated version)
  - custom-auth-lib

Used by:
  - UserServiceFunction
  - OrderServiceFunction
  - PaymentServiceFunction
```

**2. Custom Runtime Layer**
```bash
# Add runtime not natively supported
Layer: ruby-2.7-runtime
  - Ruby 2.7 binaries
  - Ruby standard library

Function: Uses custom Ruby runtime
```

**3. Binary/Tool Layer**
```bash
# Add executables
Layer: image-processing-tools
  - ffmpeg (video processing)
  - imagemagick (image manipulation)
  - pngquant (compression)

Function: Process uploaded media files
```

**4. Configuration/Secrets Layer**
```bash
# Share configuration files (NOT recommended for secrets - use Secrets Manager!)
Layer: app-config
  - config.json
  - certificates/
  - data/lookup-tables.csv
```

**5. Monitoring/Observability Layer**
```bash
# Add monitoring SDK to all functions
Layer: observability
  - datadog-lambda-python
  - aws-xray-sdk

Used by: ALL production functions
```

### AWS-Provided Layers

AWS maintains public layers you can use:

**AWS X-Ray SDK:**
```
arn:aws:lambda:us-east-1:580247275435:layer:LambdaInsightsExtension:14
```

**AWS Parameters and Secrets Extension:**
```
arn:aws:lambda:us-east-1:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:2
```

**Usage:**
```bash
aws lambda update-function-configuration \
  --function-name MyFunction \
  --layers arn:aws:lambda:us-east-1:580247275435:layer:LambdaInsightsExtension:14
```

### Best Practices for Layers

**1. Keep Layers Focused**
```bash
# ✅ GOOD: Separate layers by purpose
- layer-http-clients (requests, urllib3)
- layer-data-processing (pandas, numpy)
- layer-aws-sdk (boto3, botocore)

# ❌ BAD: One giant layer with everything
- layer-everything (requests, pandas, numpy, boto3, PIL, ...)
```

**2. Version Management**
```bash
# Use semantic versioning in layer names
- my-dependencies-layer-v1  (breaking changes)
- my-dependencies-layer-v1-1 (minor updates)
- my-dependencies-layer-v1-1-1 (patches)

# Or use layer versions
- my-dependencies-layer:1
- my-dependencies-layer:2
```

**3. Share Layers Across Accounts**
```bash
# Grant permission to other AWS accounts
aws lambda add-layer-version-permission \
  --layer-name my-dependencies-layer \
  --version-number 1 \
  --principal 123456789012 \
  --statement-id AllowAccount123 \
  --action lambda:GetLayerVersion

# Make layer public (accessible to all AWS accounts)
aws lambda add-layer-version-permission \
  --layer-name my-dependencies-layer \
  --version-number 1 \
  --principal '*' \
  --statement-id PublicAccess \
  --action lambda:GetLayerVersion
```

**4. Monitor Layer Size**
```bash
# Check unzipped size before publishing
du -sh python/
# Should be < 250 MB (including function code)

# If too large, split into multiple layers
```

**5. Test Layers Before Production**
```bash
# Create test function with layer
# Verify imports work
# Check cold start time
# Then roll out to production
```

### Exercise: Create and Use a Lambda Layer

**Objective**: Create a layer with dependencies and use it in a Lambda function.

**Step 1: Create Layer with Dependencies**

```bash
# Create layer directory structure
mkdir -p my-requests-layer/python/lib/python3.9/site-packages
cd my-requests-layer

# Install requests library into layer
pip install requests -t python/lib/python3.9/site-packages/

# Verify structure
ls python/lib/python3.9/site-packages/
# Should show: requests/ urllib3/ certifi/ charset_normalizer/ idna/

# Create zip
zip -r ../my-requests-layer.zip python/
cd ..
```

**Step 2: Publish Layer**

```bash
# Publish layer to AWS
aws lambda publish-layer-version \
  --layer-name requests-layer \
  --description "Requests library for HTTP calls" \
  --zip-file fileb://my-requests-layer.zip \
  --compatible-runtimes python3.9 python3.10 python3.11

# Save the LayerVersionArn from output
```

**Step 3: Create Function Using Layer**

Create `function.py`:
```python
import json
import requests  # This comes from the layer!

def lambda_handler(event, context):
    """
    Test function using requests library from layer
    """
    try:
        # Make HTTP request using library from layer
        response = requests.get('https://httpbin.org/get')

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Successfully used layer!',
                'response_code': response.status_code,
                'layer_test': 'PASSED'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': str(e),
                'layer_test': 'FAILED'
            })
        }
```

**Step 4: Deploy Function with Layer**

```bash
# Package function (just your code, no dependencies!)
zip function.zip function.py

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Get layer ARN (replace with your actual ARN from step 2)
LAYER_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:layer:requests-layer:1"

# Create function with layer
aws lambda create-function \
  --function-name LayerTestFunction \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler function.lambda_handler \
  --zip-file fileb://function.zip \
  --layers $LAYER_ARN
```

**Step 5: Test Function**

```bash
# Invoke function
aws lambda invoke \
  --function-name LayerTestFunction \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json && cat response.json
```

Expected output:
```json
{
    "statusCode": 200,
    "body": "{\"message\": \"Successfully used layer!\", \"response_code\": 200, \"layer_test\": \"PASSED\"}"
}
```

**Step 6: Update Layer and Test**

```bash
# Install newer version of requests
pip install requests==2.31.0 -t my-requests-layer/python/lib/python3.9/site-packages/

# Create new zip
cd my-requests-layer
zip -r ../my-requests-layer-v2.zip python/
cd ..

# Publish version 2
aws lambda publish-layer-version \
  --layer-name requests-layer \
  --zip-file fileb://my-requests-layer-v2.zip \
  --compatible-runtimes python3.9

# Update function to use version 2
LAYER_ARN_V2="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:layer:requests-layer:2"

aws lambda update-function-configuration \
  --function-name LayerTestFunction \
  --layers $LAYER_ARN_V2

# Test again
sleep 5
aws lambda invoke \
  --function-name LayerTestFunction \
  --payload '{}' \
  response.json && cat response.json
```

**Cleanup:**
```bash
# Delete function
aws lambda delete-function --function-name LayerTestFunction

# Delete layer versions
aws lambda delete-layer-version --layer-name requests-layer --version-number 1
aws lambda delete-layer-version --layer-name requests-layer --version-number 2

# Remove local files
rm -rf my-requests-layer/ my-requests-layer.zip my-requests-layer-v2.zip function.zip response.json function.py
```

**Key Learnings:**
1. Layers separate dependencies from function code
2. Small function packages = faster deployments
3. One layer can be shared across multiple functions
4. Layer versions are immutable
5. Functions must be updated to use new layer versions

---

## 2. Lambda Versions

### What are Lambda Versions?

**Lambda Versions** are immutable snapshots of your function code and configuration. Once published, a version cannot be changed.

### Theory: The Deployment Problem

#### Without Versions

**Traditional deployment:**
```
Function: MyFunction
Code: v1.0 (current)

Deploy v1.1 → Overwrites v1.0
Everyone using MyFunction now gets v1.1

Problem:
- Can't test v1.1 before everyone uses it
- Can't rollback if v1.1 has bugs
- No way to gradually migrate users
- Production uses same code as development
```

#### With Versions

**Version-based deployment:**
```
Function: MyFunction

$LATEST → Development (mutable, always latest code)
Version 1 → v1.0 (immutable, stable)
Version 2 → v1.1 (immutable, new features)
Version 3 → v1.2 (immutable, bug fixes)

Benefits:
- Test new versions before promoting
- Rollback = point to older version
- Gradual migration (canary deployments)
- Separate dev/staging/prod versions
```

### Theory: How Versions Work

#### Version Lifecycle

```
1. Working on code
   ↓
   $LATEST (mutable, changes frequently)

2. Ready to publish
   ↓
   aws lambda publish-version
   ↓
   Version 1 created (immutable snapshot of $LATEST)
   ↓
   $LATEST continues to be mutable

3. More changes
   ↓
   $LATEST updated (Version 1 unchanged!)

4. Publish again
   ↓
   Version 2 created (new immutable snapshot)

Result:
- $LATEST: Latest development code
- Version 1: Stable v1.0
- Version 2: Stable v1.1
```

### $LATEST vs Numbered Versions

| Aspect | $LATEST | Numbered Versions (1, 2, 3...) |
|--------|---------|-------------------------------|
| **Mutability** | Mutable (can update) | Immutable (cannot change) |
| **Use Case** | Development, testing | Production, staging |
| **ARN** | `...function:MyFunction` or `:$LATEST` | `...function:MyFunction:1` |
| **Invocation** | Always latest code | Always specific snapshot |
| **Configuration** | Can change timeout, memory, etc. | Fixed configuration |
| **Code** | Can update code | Cannot update code |
| **Deletion** | Cannot delete | Can delete (if not in use) |

**Example ARNs:**
```
$LATEST:
arn:aws:lambda:us-east-1:123456789012:function:MyFunction
arn:aws:lambda:us-east-1:123456789012:function:MyFunction:$LATEST

Version 1:
arn:aws:lambda:us-east-1:123456789012:function:MyFunction:1

Version 2:
arn:aws:lambda:us-east-1:123456789012:function:MyFunction:2
```

### Creating Versions

**Method 1: AWS CLI**
```bash
# Update function code first
aws lambda update-function-code \
  --function-name MyFunction \
  --zip-file fileb://new-code.zip

# Publish current $LATEST as new version
aws lambda publish-version \
  --function-name MyFunction \
  --description "Version 1.0 - Initial release"
```

**Response:**
```json
{
    "FunctionName": "MyFunction",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction:1",
    "Version": "1",
    "Description": "Version 1.0 - Initial release",
    "CodeSize": 1024,
    "MemorySize": 512,
    "Timeout": 30
}
```

**Method 2: AWS Console**
1. Go to Lambda function
2. Versions tab
3. Click "Publish new version"
4. Add description
5. Publish

### Invoking Specific Versions

**Invoke $LATEST (default):**
```bash
aws lambda invoke \
  --function-name MyFunction \
  response.json

# Or explicitly
aws lambda invoke \
  --function-name MyFunction:$LATEST \
  response.json
```

**Invoke specific version:**
```bash
aws lambda invoke \
  --function-name MyFunction:1 \
  response.json

aws lambda invoke \
  --function-name MyFunction:2 \
  response.json
```

**In code (invoking another Lambda):**
```python
import boto3

lambda_client = boto3.client('lambda')

# Invoke $LATEST
response = lambda_client.invoke(
    FunctionName='MyFunction'
)

# Invoke version 1
response = lambda_client.invoke(
    FunctionName='MyFunction:1'
)

# Invoke version 2
response = lambda_client.invoke(
    FunctionName='MyFunction:2'
)
```

### Version Management

**List all versions:**
```bash
aws lambda list-versions-by-function \
  --function-name MyFunction
```

**Get specific version details:**
```bash
aws lambda get-function \
  --function-name MyFunction:1
```

**Delete old versions:**
```bash
# Delete version 1 (if no aliases point to it)
aws lambda delete-function \
  --function-name MyFunction:1

# Note: Cannot delete $LATEST
```

### Versioning Best Practices

**1. Semantic Versioning in Descriptions**
```bash
# Use descriptions to track versions
aws lambda publish-version \
  --function-name MyFunction \
  --description "v1.0.0 - Initial release"

aws lambda publish-version \
  --function-name MyFunction \
  --description "v1.1.0 - Added feature X"

aws lambda publish-version \
  --function-name MyFunction \
  --description "v1.1.1 - Bug fix for feature X"
```

**2. Don't Use Versions Directly in Production**
```bash
# ❌ BAD: Hardcode version number
EventSourceMapping → MyFunction:5

# Problem: To update, must reconfigure event source

# ✅ GOOD: Use alias (explained in next section)
EventSourceMapping → MyFunction:PROD (alias)

# Update: Just point PROD alias to new version
```

**3. Keep Version Count Manageable**
```bash
# Delete old versions no longer needed
# AWS limit: 1000 versions per function (soft limit)

# Strategy: Keep last 5-10 versions, delete older
```

**4. Version Immutability = Audit Trail**
```
Version 1: Deployed Jan 1, 2024 - Still available for investigation
Version 2: Deployed Jan 15, 2024 - Can reproduce bugs from this version
Version 3: Deployed Feb 1, 2024 - Current production
```

### Common Version Patterns

**Pattern 1: Development → Staging → Production**
```
$LATEST → Active development (team makes changes daily)

Version 10 → Staging (testing before prod)
Version 9 → Production (stable)
Version 8 → Previous production (rollback target)
```

**Pattern 2: Canary Deployment (with Aliases)**
```
Version 5 → Old production (90% traffic)
Version 6 → New production (10% traffic)

If Version 6 is stable:
  → Shift to 100% traffic on Version 6
```

**Pattern 3: Multi-Region Deployment**
```
us-east-1:
  Version 3 → Current

eu-west-1:
  Version 2 → One version behind (gradual rollout)

ap-southeast-1:
  Version 2 → One version behind
```

### Exercise: Working with Versions

**Objective**: Publish and manage Lambda function versions.

**Step 1: Create Initial Function**

Create `versioned_function.py`:
```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Version 1.0 - Initial release',
            'version': '1.0'
        })
    }
```

```bash
# Package and deploy
zip function-v1.zip versioned_function.py

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws lambda create-function \
  --function-name VersionedFunction \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler versioned_function.lambda_handler \
  --zip-file fileb://function-v1.zip
```

**Step 2: Test $LATEST**

```bash
# Invoke $LATEST
aws lambda invoke \
  --function-name VersionedFunction \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json && cat response.json
```

Expected: `"message": "Version 1.0 - Initial release"`

**Step 3: Publish Version 1**

```bash
# Publish version 1
aws lambda publish-version \
  --function-name VersionedFunction \
  --description "v1.0.0 - Initial release"

# Note the version number in response (should be 1)
```

**Step 4: Update Code and Publish Version 2**

Update `versioned_function.py`:
```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Version 2.0 - Added new feature',
            'version': '2.0',
            'feature': 'New functionality'
        })
    }
```

```bash
# Package new version
zip function-v2.zip versioned_function.py

# Update $LATEST
aws lambda update-function-code \
  --function-name VersionedFunction \
  --zip-file fileb://function-v2.zip

# Wait for update
sleep 5

# Publish version 2
aws lambda publish-version \
  --function-name VersionedFunction \
  --description "v2.0.0 - Added new feature"
```

**Step 5: Invoke Different Versions**

```bash
# Invoke $LATEST (version 2.0)
echo "=== $LATEST ==="
aws lambda invoke \
  --function-name VersionedFunction \
  --payload '{}' \
  response.json && cat response.json && echo ""

# Invoke Version 1
echo "=== Version 1 ==="
aws lambda invoke \
  --function-name VersionedFunction:1 \
  --payload '{}' \
  response.json && cat response.json && echo ""

# Invoke Version 2
echo "=== Version 2 ==="
aws lambda invoke \
  --function-name VersionedFunction:2 \
  --payload '{}' \
  response.json && cat response.json && echo ""
```

**Expected Results:**
- $LATEST: "Version 2.0 - Added new feature"
- Version 1: "Version 1.0 - Initial release" (unchanged!)
- Version 2: "Version 2.0 - Added new feature"

**Step 6: List All Versions**

```bash
aws lambda list-versions-by-function \
  --function-name VersionedFunction
```

**Cleanup:**
```bash
# Delete versions (in reverse order)
aws lambda delete-function --function-name VersionedFunction:2
aws lambda delete-function --function-name VersionedFunction:1

# Delete function ($LATEST)
aws lambda delete-function --function-name VersionedFunction

# Remove files
rm function-v*.zip response.json versioned_function.py
```

**Key Learnings:**
1. Versions are immutable snapshots
2. $LATEST is always mutable
3. Can invoke specific versions
4. Old versions remain unchanged when publishing new versions
5. Versions enable rollback and testing

---

## 3. Lambda Aliases

### What are Lambda Aliases?

**Lambda Aliases** are pointers to specific Lambda function versions. They provide a stable reference that can be updated to point to different versions without changing the caller's configuration.

### Theory: The Version Reference Problem

#### Without Aliases

**Direct version references:**
```
API Gateway → MyFunction:5
Step Function → MyFunction:5
Event Source → MyFunction:5

To update to Version 6:
  - Update API Gateway configuration
  - Update Step Function definition
  - Update Event Source mapping

Problem: Many places to update, error-prone!
```

#### With Aliases

**Alias-based references:**
```
API Gateway → MyFunction:PROD (alias)
Step Function → MyFunction:PROD (alias)
Event Source → MyFunction:PROD (alias)

PROD alias → Version 5

To update to Version 6:
  - Update PROD alias to point to Version 6
  - That's it! All callers automatically use Version 6

Benefit: Update once, affects all callers!
```

### Theory: How Aliases Work

#### Alias Architecture

```
Alias: PROD
  ↓ (points to)
Version 5

Invocation:
MyFunction:PROD
  ↓ (resolves to)
MyFunction:5
  ↓ (executes)
Function Code Version 5
```

**Updating alias:**
```
1. Initial state:
   PROD → Version 5

2. Publish Version 6
   PROD → Version 5
   (Version 6 exists but unused)

3. Update alias:
   PROD → Version 6 (atomic switch!)

4. New invocations:
   PROD → Version 6
   (Version 5 still exists, can rollback)
```

### Alias Features

**1. Mutable Pointers**
- Alias name is fixed ("PROD", "STAGING", "DEV")
- Can update which version it points to
- Callers use alias name, don't need to know version

**2. Traffic Splitting (Weighted Routing)**
- Route percentage of traffic to two versions
- Enables canary deployments
- Gradual rollout of new versions

**3. Stable ARNs**
```
Alias ARN: arn:aws:lambda:us-east-1:123456789012:function:MyFunction:PROD
(Never changes, even when pointing to different versions)

Version ARN: arn:aws:lambda:us-east-1:123456789012:function:MyFunction:5
(Specific, immutable)
```

### Creating and Using Aliases

**Create alias:**
```bash
# Create PROD alias pointing to version 1
aws lambda create-alias \
  --function-name MyFunction \
  --name PROD \
  --function-version 1 \
  --description "Production environment"
```

**Update alias to new version:**
```bash
# Point PROD to version 2
aws lambda update-alias \
  --function-name MyFunction \
  --name PROD \
  --function-version 2
```

**Invoke via alias:**
```bash
# Invoke PROD alias
aws lambda invoke \
  --function-name MyFunction:PROD \
  response.json
```

### Common Alias Naming Patterns

**Environment-based:**
```
DEV → $LATEST (always latest development code)
STAGING → Version 5 (testing before production)
PROD → Version 4 (stable production)
```

**Release-based:**
```
LATEST-STABLE → Version 10
PREVIOUS-STABLE → Version 9 (rollback target)
BETA → Version 11 (beta testing)
```

**Feature-based:**
```
FEATURE-AUTH → Version 6 (new authentication feature)
FEATURE-PAYMENT → Version 7 (new payment integration)
MAIN → Version 5 (main branch)
```

### Traffic Splitting (Weighted Aliases)

#### Theory: Canary Deployments

**Canary deployment** = Gradually shift traffic from old version to new version.

```
Initial State:
PROD → Version 1 (100%)

Canary Phase 1: (10% on new version)
PROD → Version 1 (90%)
    → Version 2 (10%)

Monitor for errors...

Canary Phase 2: (50% on new version)
PROD → Version 1 (50%)
    → Version 2 (50%)

Monitor for errors...

Final State: (100% on new version)
PROD → Version 2 (100%)
```

**Create alias with traffic splitting:**
```bash
# PROD: 90% on Version 1, 10% on Version 2
aws lambda create-alias \
  --function-name MyFunction \
  --name PROD \
  --function-version 1 \
  --routing-config AdditionalVersionWeights={"2"=0.1}
```

**Update traffic weights:**
```bash
# Increase to 50/50 split
aws lambda update-alias \
  --function-name MyFunction \
  --name PROD \
  --routing-config AdditionalVersionWeights={"2"=0.5}

# Shift to 100% Version 2
aws lambda update-alias \
  --function-name MyFunction \
  --name PROD \
  --function-version 2
  # (omit routing-config for 100% on single version)
```

### Alias Permissions

**Grant invoke permission to alias:**
```bash
# Allow API Gateway to invoke PROD alias
aws lambda add-permission \
  --function-name MyFunction:PROD \
  --statement-id AllowAPIGatewayInvoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn arn:aws:execute-api:us-east-1:123456789012:api-id/*
```

### Best Practices for Aliases

**1. Always Use Aliases for External Integrations**
```bash
# ✅ GOOD
API Gateway → MyFunction:PROD
EventBridge → MyFunction:PROD

# ❌ BAD
API Gateway → MyFunction:5 (hard to update)
EventBridge → MyFunction:$LATEST (unstable)
```

**2. Standard Alias Set**
```bash
# Create consistent aliases across all functions
DEV → $LATEST (development)
STAGING → Version N (staging/testing)
PROD → Version N-1 (production)
```

**3. Use Traffic Splitting for Gradual Rollouts**
```bash
# Start with 5% canary
# Increase gradually: 5% → 10% → 25% → 50% → 100%
# Monitor metrics between each step
```

**4. Combine with CloudWatch Alarms**
```bash
# Monitor errors on canary traffic
# Automatic rollback if error rate spikes
```

### Alias Management

**List aliases:**
```bash
aws lambda list-aliases \
  --function-name MyFunction
```

**Get alias details:**
```bash
aws lambda get-alias \
  --function-name MyFunction \
  --name PROD
```

**Delete alias:**
```bash
aws lambda delete-alias \
  --function-name MyFunction \
  --name STAGING
```

### Exercise: Working with Aliases and Traffic Splitting

**Objective**: Create aliases and perform canary deployment with traffic splitting.

**Step 1: Create Function with Two Versions**

Create `alias_function_v1.py`:
```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Version 1 - Stable',
            'version': 1
        })
    }
```

```bash
# Deploy version 1
zip function-v1.zip alias_function_v1.py

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws lambda create-function \
  --function-name AliasTestFunction \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler alias_function_v1.lambda_handler \
  --zip-file fileb://function-v1.zip

# Publish version 1
aws lambda publish-version \
  --function-name AliasTestFunction \
  --description "Version 1 - Stable"
```

Create `alias_function_v2.py`:
```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Version 2 - New Feature',
            'version': 2,
            'feature': 'Enhanced response'
        })
    }
```

```bash
# Update to version 2
zip function-v2.zip alias_function_v2.py

aws lambda update-function-code \
  --function-name AliasTestFunction \
  --zip-file fileb://function-v2.zip

sleep 5

# Publish version 2
aws lambda publish-version \
  --function-name AliasTestFunction \
  --description "Version 2 - New Feature"
```

**Step 2: Create PROD Alias (100% Version 1)**

```bash
# Create PROD alias pointing to version 1
aws lambda create-alias \
  --function-name AliasTestFunction \
  --name PROD \
  --function-version 1 \
  --description "Production environment"

# Test PROD alias
aws lambda invoke \
  --function-name AliasTestFunction:PROD \
  --payload '{}' \
  response.json && cat response.json
```

Expected: Version 1 response

**Step 3: Canary Deployment (10% on Version 2)**

```bash
# Update PROD: 90% Version 1, 10% Version 2
aws lambda update-alias \
  --function-name AliasTestFunction \
  --name PROD \
  --function-version 1 \
  --routing-config AdditionalVersionWeights={"2"=0.1}

echo "Starting canary deployment: 90% v1, 10% v2"
echo "Invoking 20 times to see traffic distribution..."

# Invoke 20 times and count responses
for i in {1..20}; do
  aws lambda invoke \
    --function-name AliasTestFunction:PROD \
    --payload '{}' \
    response.json > /dev/null 2>&1
  cat response.json | grep -o '"version": [0-9]' | cut -d' ' -f2
done | sort | uniq -c
```

Expected: ~18 invocations on Version 1, ~2 on Version 2

**Step 4: Increase to 50/50 Split**

```bash
# Update to 50/50 split
aws lambda update-alias \
  --function-name AliasTestFunction \
  --name PROD \
  --function-version 1 \
  --routing-config AdditionalVersionWeights={"2"=0.5}

echo "Increased canary: 50% v1, 50% v2"
echo "Invoking 20 times..."

for i in {1..20}; do
  aws lambda invoke \
    --function-name AliasTestFunction:PROD \
    --payload '{}' \
    response.json > /dev/null 2>&1
  cat response.json | grep -o '"version": [0-9]' | cut -d' ' -f2
done | sort | uniq -c
```

Expected: ~10 invocations on Version 1, ~10 on Version 2

**Step 5: Complete Migration (100% Version 2)**

```bash
# Switch to 100% Version 2
aws lambda update-alias \
  --function-name AliasTestFunction \
  --name PROD \
  --function-version 2

echo "Completed migration: 100% v2"
echo "Invoking 10 times..."

for i in {1..10}; do
  aws lambda invoke \
    --function-name AliasTestFunction:PROD \
    --payload '{}' \
    response.json > /dev/null 2>&1
  cat response.json | grep -o '"version": [0-9]' | cut -d' ' -f2
done | sort | uniq -c
```

Expected: All 10 invocations on Version 2

**Step 6: Rollback Test**

```bash
# Simulate issue with Version 2, rollback to Version 1
echo "Simulating rollback to Version 1..."

aws lambda update-alias \
  --function-name AliasTestFunction \
  --name PROD \
  --function-version 1

# Test
aws lambda invoke \
  --function-name AliasTestFunction:PROD \
  --payload '{}' \
  response.json && cat response.json
```

Expected: Back to Version 1 response

**Cleanup:**
```bash
# Delete alias
aws lambda delete-alias \
  --function-name AliasTestFunction \
  --name PROD

# Delete versions
aws lambda delete-function --function-name AliasTestFunction:2
aws lambda delete-function --function-name AliasTestFunction:1

# Delete function
aws lambda delete-function --function-name AliasTestFunction

# Remove files
rm function-v*.zip response.json alias_function_v*.py
```

**Key Learnings:**
1. Aliases provide stable references to versions
2. Traffic splitting enables canary deployments
3. Can gradually shift traffic from old to new version
4. Easy rollback by updating alias
5. Callers don't need to know which version they're using

---

## 4. Concurrency

### What is Lambda Concurrency?

**Concurrency** is the number of function invocations running simultaneously at any given time.

### Theory: Understanding Concurrency

#### How Lambda Scales

When requests arrive:

```
Request 1 → Lambda creates Execution Environment 1 → Runs function
Request 2 → Lambda creates Execution Environment 2 → Runs function
Request 3 → Lambda creates Execution Environment 3 → Runs function

Concurrent executions = 3
```

**If 1000 requests arrive at the same time:**
- Lambda creates up to 1000 execution environments
- All 1000 run concurrently
- Limited by concurrency limits

#### Concurrency vs Throughput

**Concurrency**: How many executions running **at the same time**

**Throughput**: How many executions **per second**

```
Example:
- Function duration: 2 seconds
- Requests per second: 100

Concurrency needed = 100 requests/sec × 2 sec = 200 concurrent executions

If concurrency limit is 100:
  → 100 execute immediately
  → 100 are throttled (rejected or queued)
```

### Theory: Concurrency Limits

#### Account-Level Limit

**Default**: 1,000 concurrent executions per region per account

```
Your AWS Account in us-east-1:
  Total concurrent executions across ALL functions: 1,000

Function A: 300 concurrent executions
Function B: 500 concurrent executions
Function C: 200 concurrent executions
Total: 1,000 (at limit!)

New request to Function D → THROTTLED!
```

**Requesting increase:**
```bash
# Contact AWS Support to increase
# Can be increased to tens of thousands
```

#### Burst Concurrency

**Initial burst**: Lambda can scale quickly to handle sudden traffic spikes.

- **Initial burst capacity**: 500-3000 (region-dependent)
- **Sustained scaling**: +500 executions per minute after burst

```
Time 0: 0 concurrent executions
Time 0 (spike): 3000 requests arrive
  → Lambda handles 3000 immediately (initial burst)

Time 1 min: Still receiving 500 req/min
  → Can sustain 3000 + 500 = 3500 concurrent

Time 2 min: Still receiving 500 req/min
  → Can sustain 3500 + 500 = 4000 concurrent

Continues until account limit reached
```

### Types of Concurrency

#### 1. Unreserved Concurrency (Default)

**All functions share the pool:**
```
Account limit: 1,000

Function A: Uses what it needs (up to 1,000)
Function B: Uses what it needs (up to 1,000)
Function C: Uses what it needs (up to 1,000)

Problem: One function can consume entire pool!
```

**Noisy neighbor problem:**
```
Function A (batch job): Suddenly needs 950 executions
Account pool: 950 used, 50 remaining

Function B (API): Needs 100 executions
Result: 50 succeed, 50 throttled (API degraded!)
```

#### 2. Reserved Concurrency

**Dedicated capacity for a function:**

```
Account limit: 1,000
Function A reserved concurrency: 200

Pool allocation:
  Function A: 200 (dedicated)
  Shared pool: 800 (for all other functions)

Benefits:
  - Function A guaranteed 200 executions
  - Function A cannot exceed 200 (protects others)
  - Other functions cannot steal Function A's capacity
```

**Setting reserved concurrency:**
```bash
aws lambda put-function-concurrency \
  --function-name MyFunction \
  --reserved-concurrent-executions 200
```

**Use cases:**
- Critical functions (APIs, payment processing)
- Protect against noisy neighbors
- Limit expensive functions (prevent runaway costs)

**Important**: Reserved concurrency reduces available pool!

```
Account: 1,000
Function A reserved: 300
Function B reserved: 200
Remaining pool: 500 (for all other functions)
```

#### 3. Provisioned Concurrency

**Pre-initialized execution environments:**

**The Cold Start Problem:**
```
First request arrives
  ↓
Lambda creates execution environment (COLD START)
  - Download function code
  - Initialize runtime
  - Run initialization code
  ↓
Duration: 1-5 seconds (slow!)
  ↓
Execute function
```

**With Provisioned Concurrency:**
```
BEFORE any requests:
  Lambda creates 100 execution environments
  All initialized and ready (WARM)

Request arrives
  ↓
Use pre-initialized environment (NO COLD START!)
  ↓
Duration: < 100ms (fast!)
```

**Setting provisioned concurrency:**
```bash
# Must be set on alias or version, NOT $LATEST
aws lambda put-provisioned-concurrency-config \
  --function-name MyFunction:PROD \
  --provisioned-concurrent-executions 100
```

**Costs:**
- Charged for **provisioned capacity** (even if unused!)
- GB-second pricing for provisioned environments
- More expensive than on-demand

**Use cases:**
- Latency-sensitive applications (< 100ms response)
- Predictable traffic patterns
- Critical APIs with SLAs

### Concurrency Comparison

| Type | Cold Starts | Guaranteed Capacity | Protects Others | Cost | Use Case |
|------|-------------|-------------------|-----------------|------|----------|
| **Unreserved** | Yes | No | No | Lowest | Default, non-critical |
| **Reserved** | Yes | Yes | Yes | Same as unreserved | Critical functions, cost control |
| **Provisioned** | No | Yes | Yes | Highest | Latency-sensitive, SLA requirements |

### Throttling Behavior

**When throttled:**

**Synchronous invocation (RequestResponse):**
```
Request → Lambda (at concurrency limit)
Response: 429 TooManyRequestsException
```

**Asynchronous invocation (Event):**
```
Request → Lambda (at concurrency limit)
Lambda: Queues request (retries for up to 6 hours)
Eventually: Executes when capacity available
  or sends to Dead Letter Queue if retries exhausted
```

**Event source mapping (SQS, DynamoDB Streams, Kinesis):**
```
Lambda polls source → At concurrency limit
Lambda: Slows down polling, backs off
Eventually: Resumes when capacity available
```

### Monitoring Concurrency

**CloudWatch Metrics:**

```bash
# View concurrent executions
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name ConcurrentExecutions \
  --dimensions Name=FunctionName,Value=MyFunction \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --period 300 \
  --statistics Maximum
```

**Key metrics:**
- `ConcurrentExecutions`: Current concurrent executions
- `UnreservedConcurrentExecutions`: Executions in shared pool
- `Throttles`: Number of throttled invocations

### Calculating Required Concurrency

**Formula:**
```
Concurrency = (Requests per second) × (Average duration in seconds)

Example 1:
- 100 requests/sec
- Average duration: 2 sec
- Concurrency needed: 100 × 2 = 200

Example 2:
- 1,000 requests/sec
- Average duration: 0.5 sec
- Concurrency needed: 1,000 × 0.5 = 500

Example 3:
- 10 requests/sec
- Average duration: 10 sec
- Concurrency needed: 10 × 10 = 100
```

### Best Practices for Concurrency

**1. Set Reserved Concurrency for Critical Functions**
```bash
# Reserve 300 for payment API (critical)
aws lambda put-function-concurrency \
  --function-name PaymentAPI \
  --reserved-concurrent-executions 300
```

**2. Use Provisioned Concurrency Strategically**
```bash
# Only for latency-critical, high-traffic functions
# Expensive - use sparingly!

# Example: User-facing API
aws lambda put-provisioned-concurrency-config \
  --function-name UserAPI:PROD \
  --provisioned-concurrent-executions 50
```

**3. Monitor and Adjust**
```bash
# Watch CloudWatch metrics
# Adjust reserved/provisioned based on actual usage
```

**4. Set Reserved Concurrency = 0 to Disable Function**
```bash
# Emergency: Disable function completely
aws lambda put-function-concurrency \
  --function-name BuggyFunction \
  --reserved-concurrent-executions 0

# No executions allowed (effectively disabled)
```

**5. Account for Burst Limits**
```
Initial burst: 3000
Sustained: +500/min

Plan deployments considering burst capacity
```

### Exercise: Understanding Concurrency

**Objective**: Observe concurrency behavior and throttling.

**Step 1: Create Long-Running Function**

Create `concurrency_test.py`:
```python
import json
import time

def lambda_handler(event, context):
    """
    Simulates long-running function to demonstrate concurrency
    """
    duration = event.get('duration', 10)

    print(f"Starting execution, will run for {duration} seconds")
    time.sleep(duration)
    print("Execution complete")

    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': f'Completed after {duration} seconds',
            'requestId': context.aws_request_id
        })
    }
```

```bash
# Deploy function
zip concurrency-test.zip concurrency_test.py

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws lambda create-function \
  --function-name ConcurrencyTest \
  --runtime python3.9 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/MyFirstLambdaRole \
  --handler concurrency_test.lambda_handler \
  --timeout 30 \
  --zip-file fileb://concurrency-test.zip
```

**Step 2: Set Reserved Concurrency**

```bash
# Limit to 3 concurrent executions
aws lambda put-function-concurrency \
  --function-name ConcurrencyTest \
  --reserved-concurrent-executions 3

echo "Reserved concurrency set to 3"
```

**Step 3: Generate Concurrent Requests**

```bash
# Invoke 10 functions in parallel (each runs for 10 seconds)
echo "Invoking 10 functions concurrently..."
echo "Expected: 3 succeed immediately, 7 throttled or queued"

for i in {1..10}; do
  (
    aws lambda invoke \
      --function-name ConcurrencyTest \
      --invocation-type RequestResponse \
      --payload '{"duration": 10}' \
      response-${i}.json 2>&1 | grep -E "(StatusCode|TooManyRequests)" &
  )
done

wait

echo "Checking responses..."
ls response-*.json 2>/dev/null | wc -l
```

**Step 4: Monitor Concurrency**

```bash
# Check concurrent executions
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name ConcurrentExecutions \
  --dimensions Name=FunctionName,Value=ConcurrencyTest \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum

# Check throttles
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Throttles \
  --dimensions Name=FunctionName,Value=ConcurrencyTest \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

**Step 5: Remove Concurrency Limit**

```bash
# Remove reserved concurrency (back to unreserved)
aws lambda delete-function-concurrency \
  --function-name ConcurrencyTest

echo "Concurrency limit removed, now using shared pool"
```

**Cleanup:**
```bash
# Delete function
aws lambda delete-function --function-name ConcurrencyTest

# Remove files
rm concurrency-test.zip concurrency_test.py response-*.json 2>/dev/null
```

**Key Learnings:**
1. Concurrency = simultaneous executions
2. Reserved concurrency provides guaranteed capacity
3. Exceeding limits results in throttling
4. CloudWatch metrics show concurrency usage
5. Reserved concurrency limits function's max executions

---

## Summary

### Lambda Layers
- **Purpose**: Share code, libraries, and dependencies across functions
- **Benefits**: Smaller deployment packages, reusable dependencies, easier updates
- **Limits**: Max 5 layers per function, 250 MB total unzipped size
- **Structure**: `/opt/` mount point, language-specific paths
- **Use cases**: Common libraries, custom runtimes, tools/binaries

### Lambda Versions
- **Purpose**: Immutable snapshots of function code and configuration
- **$LATEST**: Mutable development version
- **Numbered versions**: Immutable (1, 2, 3...)
- **Benefits**: Rollback capability, separate dev/staging/prod, audit trail
- **Use cases**: Production deployments, testing, gradual rollouts

### Lambda Aliases
- **Purpose**: Pointers to specific versions with stable references
- **Benefits**: Easy updates, traffic splitting, canary deployments
- **Traffic splitting**: Route percentage to different versions (10%, 50%, 100%)
- **Use cases**: Environment management (DEV/STAGING/PROD), gradual rollouts, A/B testing

### Concurrency
- **Types**: Unreserved (default), Reserved (dedicated), Provisioned (pre-initialized)
- **Account limit**: 1,000 concurrent executions per region (default)
- **Reserved**: Guarantees capacity, protects from noisy neighbors
- **Provisioned**: Eliminates cold starts, costs more, requires alias/version
- **Formula**: Concurrency = (Requests/sec) × (Duration in seconds)
- **Throttling**: 429 error (sync), queued (async), backoff (event sources)

### Best Practices Summary

**Layers:**
1. Keep layers focused (separate by purpose)
2. Version layers appropriately
3. Share layers across accounts when beneficial
4. Monitor layer size (< 250 MB total)

**Versions:**
1. Use $LATEST for development only
2. Publish versions for staging/production
3. Keep version count manageable (delete old versions)
4. Use semantic versioning in descriptions

**Aliases:**
1. Always use aliases for external integrations
2. Standard alias set: DEV, STAGING, PROD
3. Use traffic splitting for gradual rollouts
4. Combine with CloudWatch alarms for safety

**Concurrency:**
1. Set reserved concurrency for critical functions
2. Use provisioned concurrency only when necessary (expensive!)
3. Monitor metrics and adjust based on actual usage
4. Calculate required concurrency before setting limits
5. Use reserved concurrency = 0 to disable functions

---

These advanced features provide powerful capabilities for managing Lambda functions at scale with proper version control, traffic management, and performance optimization!
