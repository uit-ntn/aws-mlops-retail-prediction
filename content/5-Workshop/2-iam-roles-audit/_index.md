---
title: "IAM Roles & Audit For MLops"
date: 2025-08-30T12:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## 🎯 Task 2 Objectives

Set up **access permissions (IAM)** for all AWS services in the pipeline and **enable CloudTrail** to monitor and record all activities in the AWS account.

→ **Ensure security, access control, and evidence of team activities.**

📥 **Input**
- AWS Account with admin rights
- Project naming convention: `mlops-retail-prediction-dev`
- Target region: `ap-southeast-1`
- Multi-region CloudTrail enabled

- **Input from Task 1:** Task 1 (Introduction) — project conventions, naming, and high-level objectives

- **Input from Task 1:** Task 1 (Introduction) — project conventions, naming, and high-level objectives

✅ **Output**
- AWS services have appropriate **Least Privilege** permissions for their roles
- All operations are **recorded by CloudTrail**
- Meets rubric criteria: **security, access control, cloud project management**

💰 **Estimated Cost**
≈ **0.05 USD/month** (CloudTrail + S3 storage for logs)
💰 **Estimated Cost**
≈ **0.05 USD/month** (CloudTrail + S3 storage for logs)

📌 **Main Steps**
1. **CloudTrail Setup** - Set up multi-region audit logging
2. **S3 CloudTrail Bucket** - Centralized log storage
3. **EKS Cluster Service Role** - Role for control plane
4. **EKS Node Group Role** - Role for worker nodes
5. **SageMaker Execution Role** - Role for training & deployment
6. **IRSA Foundation** - Prepare pod-level permissions

✅ **Deliverables**
- CloudTrail multi-region trail with logging to S3
- EKS Cluster Service Role (Console)
- EKS Node Group Role with ECR/S3/CloudWatch permissions
- SageMaker Execution Role with necessary S3 permissions
- Security foundation for IRSA setup

📊 **Acceptance Criteria**
- CloudTrail records all API calls and user activities
- EKS cluster can be created with appropriate service roles
- SageMaker training jobs have permissions to read/write S3
- Node groups can pull images from ECR
- All IAM roles comply with principle of least privilege
- Audit trail ready for compliance and monitoring

## 1. CloudTrail Setup - Audit Foundation

### 1.1. Create S3 Bucket for CloudTrail

**Go to S3 Console:**
AWS Console → S3 → "Create bucket"

**Bucket Configuration:**
   ```
   Bucket name: mlops-cloudtrail-logs-us-east-1-842676018087
   Region: us-east-1 (must be same region as CloudTrail trail)
   Block all public access: ✅ Enabled
   Versioning: ✅ Enabled  
   Default encryption: ✅ AWS KMS
   KMS key: alias/mlops-retail-prediction-dev-cloudtrail-key
   ```
**Lifecycle Policy Configuration:**

**Step 1**. S3 Console → select bucket `mlops-cloudtrail-logs-ap-southeast-1` → Management → Create lifecycle rule.

![S3 lifecycle](/images/2-iam-roles-audit/1.1-cloudtrail-s3-lifecycle-01.png)

**Step 2**. Name the rule (e.g. `CloudTrailLogLifecycle`), Apply to all objects or use Prefix `mlops-logs/`.

**Configuration field details:**

1. **Object tags**: 
   - ❌ No need to add tags since we're using prefix filtering

2. **Object size**: 
   - ❌ No need to specify minimum object size
   - ❌ No need to specify maximum object size
   - CloudTrail logs are typically small and uniform in size

3. **Lifecycle rule actions**:
   - ✅ Transition current versions of objects between storage classes
     - Select to automatically move logs to cheaper storage classes
     - Select to automatically move logs to cheaper storage classes
   - ❌ Transition noncurrent versions of objects between storage classes
     - Not needed as CloudTrail logs don't have many versions
   - ❌ Expire current versions of objects
     - Don't expire as we need to keep logs for audit
   - ❌ Permanently delete noncurrent versions of objects
     - Don't delete as we need to keep log history
   - ✅ Delete expired object delete markers or incomplete multipart uploads
     - Select to clean up failed markers and uploads

![S3 lifecycle](/images/2-iam-roles-audit/1.2-cloudtrail-s3-lifecycle-01.png)

**Step 3**. Choose actions (Current versions):
   - After 30 days → STANDARD_IA
   - After 90 days → GLACIER / GLACIER_IR (optional)
   - After 365 days → DEEP_ARCHIVE

   ![S3 Lifecycle Overview](/images/2-iam-roles-audit/1.3-cloudtrail-s3-lifecycle-overview.png "S3 Lifecycle for CloudTrail logs")
 
**Step 4**. Verify the rule is Active in the Management tab.

![S3 lifecycle](/images/2-iam-roles-audit/1.4-cloudtrail-s3-lifecycle-01.png)
### 1.2 Configure Trail

{{% notice warning %}}
**⚠️ Region Note:**
- CloudTrail is a multi-region service but the trail must be created in a specific region (home region)
- S3 bucket and KMS key must be created in the same region as the CloudTrail trail
- In this case, we will create all resources in the `us-east-1` region
{{% /notice %}}

### 1.3. Recommendation: Synchronize region between S3 and SageMaker Project

**In short:** If the main pipeline data (prefix `gold/` and `artifacts/`) is in `us-east-1`, **create SageMaker Domain / Project in `us-east-1`** to avoid cross-region errors (S3 301), KMS key complexity, and other endpoint issues.

If your organization requires SageMaker to be in `ap-southeast-1`, you need to copy or replicate data to a bucket in `ap-southeast-1` before creating the Project. Example sync commands (PowerShell / CloudShell):

```powershell
aws s3 mb s3://mlops-retail-prediction-dev-842676018087-apse1 --region ap-southeast-1
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/gold/ s3://mlops-retail-prediction-dev-842676018087-apse1/gold/ --acl bucket-owner-full-control
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/artifacts/ s3://mlops-retail-prediction-dev-842676018087-apse1/artifacts/ --acl bucket-owner-full-control
```

Along with:
- Create KMS key in the destination region if using SSE-KMS.
- Update IAM policies to allow SageMaker role access to the new bucket.

Suggestion: For labs and quick debugging, the least risky approach is to create Project/Domain where the bucket currently exists (in this lab: `us-east-1`).

#### 1.2.1 Create KMS Key for CloudTrail

1. **Create KMS Key (in region `us-east-1`):**

- AWS Console → KMS → us-east-1 → Customer managed keys → Create key

2. **Key Configuration:**
   ```
   Key type: ✅ Symmetric (encrypt and decrypt data)
   Key type: ✅ Symmetric (encrypt and decrypt data)
   Key usage: ✅ Encrypt and decrypt
   ```

3. **Add labels:**
   ```
   Alias: alias/mlops-retail-prediction-dev-cloudtrail-key
   Description (optional): KMS key for CloudTrail logs encryption
   Tags (optional):
     - Key: Project
     - Value: MLOps-Retail-Prediction
   ```

4. **Configure administrative permissions:**
   ```
   Key administrators: Select IAM users/roles allowed to manage key
   Key deletion: Whether to allow key deletion
   ```

5. **Configure usage permissions:**
   ```
   Key users: 
   - Add service principal: cloudtrail.amazonaws.com
   ```

6. **Edit key policy:**
   ```json
   {
     "Version": "2012-10-17",
     "Id": "Key policy created by CloudTrail",
     "Statement": [
       {
         "Sid": "Enable IAM User Permissions",
         "Effect": "Allow",
         "Principal": {
           "AWS": ["arn:aws:iam::842676018087:root"]
         },
         "Action": "kms:*",
         "Resource": "*"
       },
       {
         "Sid": "Allow CloudTrail to encrypt logs",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "kms:GenerateDataKey*",
         "Resource": "*",
         "Condition": {
           "StringLike": {
             "kms:EncryptionContext:aws:cloudtrail:arn": [
               "arn:aws:cloudtrail:*:842676018087:trail/*"
             ]
           },
           "StringEquals": {
             "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:842676018087:trail/mlops-retail-prediction-audit-trail"
           }
         }
       },
       {
         "Sid": "Allow CloudTrail to describe key",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "kms:DescribeKey",
         "Resource": "*"
       }
     ]
   }
   ```

{{% notice warning %}}
**Important points in KMS policy:**
Enable IAM User Permissions: Allow root account to manage key
Allow CloudTrail to encrypt logs:
   - Allow generateDataKey with EncryptionContext and SourceArn conditions
   - EncryptionContext limits to CloudTrail trails in account
   - SourceArn specifies exactly which trail is allowed to use
Allow CloudTrail to describe key: Allow CloudTrail to view key information
{{% /notice %}}

3. **Default Key Policy for CloudTrail:**
   ```json
   {
     "Version": "2012-10-17",
     "Id": "Key policy created by CloudTrail",
     "Statement": [
       {
         "Sid": "Enable IAM User Permissions",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::842676018087:root"
         },
         "Action": "kms:*",
         "Resource": "*"
       },
       {
         "Sid": "Allow CloudTrail to encrypt logs",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "kms:GenerateDataKey*",
         "Resource": "*",
         "Condition": {
           "StringEquals": {
             "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:842676018087:trail/mlops-retail-prediction-audit-trail"
           },
           "StringLike": {
             "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:842676018087:trail/*"
           }
         }
       },
       {
         "Sid": "Allow CloudTrail to describe key",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "kms:DescribeKey",
         "Resource": "*"
       },
       {
         "Sid": "Allow principals in the account to decrypt log files",
         "Effect": "Allow",
         "Principal": {
           "AWS": "*"
         },
         "Action": ["kms:Decrypt", "kms:ReEncryptFrom"],
         "Resource": "*",
         "Condition": {
           "StringEquals": {
             "kms:CallerAccount": "842676018087"
           },
           "StringLike": {
             "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:842676018087:trail/*"
           }
         }
       },
       {
         "Sid": "Enable cross account log decryption",
         "Effect": "Allow",
         "Principal": {
           "AWS": "*"
         },
         "Action": ["kms:Decrypt", "kms:ReEncryptFrom"],
         "Resource": "*",
         "Condition": {
           "StringEquals": {
             "kms:CallerAccount": "842676018087"
           },
           "StringLike": {
             "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:842676018087:trail/*"
           }
         }
       }
     ]
   }
   ```

{{% notice info %}}

**Key Policy includes:**
   - Allow root account to manage key
   - Allow CloudTrail to encrypt logs with condition that trail ARN matches
   - Allow CloudTrail to view key information
   - Allow principals in account to decrypt logs
   - Support cross-account log decryption if needed
   {{% /notice %}}

{{% notice warning %}}
KMS key must be created in the same region as the S3 bucket and include a policy that allows CloudTrail usage.
{{% /notice %}}

#### 1.2.2 Configure S3 Bucket Policy

1. **Go to S3 bucket permissions:**
   ```
   S3 Console → mlops-cloudtrail-logs-ap-southeast-1 → Permissions → Bucket policy
   ```

2. **Default policy for S3 bucket:**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AWSCloudTrailAclCheck",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "s3:GetBucketAcl",
         "Resource": "arn:aws:s3:::mlops-cloudtrail-logs-ap-southeast-1-842676018087",
         "Condition": {
           "StringEquals": {
             "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:842676018087:trail/mlops-retail-prediction-audit-trail"
           }
         }
       },
       {
         "Sid": "AWSCloudTrailWrite",
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudtrail.amazonaws.com"
         },
         "Action": "s3:PutObject",
         "Resource": "arn:aws:s3:::mlops-cloudtrail-logs-ap-southeast-1-842676018087/mlops-logs/AWSLogs/842676018087/*",
         "Condition": {
           "StringEquals": {
             "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:842676018087:trail/mlops-retail-prediction-audit-trail",
             "s3:x-amz-acl": "bucket-owner-full-control"
           }
         }
       }
     ]
   }
   ```

{{% notice info %}}
   **S3 Bucket Policy includes:**
   - AWSCloudTrailAclCheck: Allow CloudTrail to check bucket ACL
   - AWSCloudTrailWrite: Allow CloudTrail to write logs to bucket
   - Conditions:
     - aws:SourceArn: Ensure only specific trail can access
     - s3:x-amz-acl: Ensure bucket owner has full control of objects
{{% /notice %}}

{{% notice info %}}
This policy allows CloudTrail to check bucket ACL and write logs to bucket.
{{% /notice %}}

#### 1.2.3 Create the CloudTrail

**Step 1: Create a new trail**
```
AWS Console → us-east-1 → CloudTrail → Create trail
```

**Step 2: Basic trail configuration (in us-east-1)**

| Item | Value |
|-----|----------|
| **Trail name** | `mlops-retail-prediction-audit-trail` |
| **Apply trail to all regions** | ✅ Yes |
| **Management events** | ✅ Read/Write |
| **Data events** | ✅ S3 bucket data events |
| **Insights events** | ✅ Enabled (detect anomalous behavior) |

**Step 3: Storage configuration**

| Item | Value |
|-----|----------|
| **S3 bucket** | `mlops-cloudtrail-logs-ap-southeast-1` |
| **Log file prefix** | `mlops-logs/` |
| **Log file SSE-KMS encryption** | ✅ Enabled |
| **AWS KMS alias** | `alias/mlops-retail-prediction-dev-cloudtrail-key` (select the created key) |

**Step 4: CloudWatch Logs integration (optional)**

| Item | Value |
|-----|----------|
| **CloudWatch Logs** | ✅ Enabled |
| **Log group** | `mlops-cloudtrail-log-group` |
| **IAM Role** | `CloudTrail_CloudWatchLogs_Role` (auto-created) |

{{% notice info %}}
The `CloudTrail_CloudWatchLogs_Role` will be auto-created with necessary permissions: `logs:PutLogEvents`, `logs:CreateLogStream`, `logs:DescribeLogStreams`.
{{% /notice %}}

**Step 5: Review and create the trail**

{{% notice tip %}}
Correct order to avoid errors:
1. ✅ Create KMS key with proper policy
2. ✅ Configure S3 bucket policy
3. ✅ Create CloudTrail with the configured KMS and S3
4. ✅ Verify logs are being delivered
{{% /notice %}}

## 2. IAM Roles Setup - Service permissions


### 2.1. EKS Cluster Service Role

- AWS Console → IAM → Roles → "Create role"

1. **Trusted entity type:**
	```
	/* Lines 445-447 omitted */
	```
2. **Attach policies:**
	```
	/* Lines 450-451 omitted */
	```
3. **Role details:**
	```
	/* Lines 454-456 omitted */
	```

### 2.2. EKS Node Group Role
- Similar steps

1. **Trusted entity type:**
	```
	/* Lines 463-465 omitted */
	```
2. **Attach policies:**
	```
	/* Lines 468-472 omitted */
	```
3. **Role details:**
	```
	/* Lines 475-478 omitted */
	```
	![EKS Node Group Role - Trust relationship and attached policies](/images/2-iam-roles-audit/03-eks-nodegroup-role-policies.png "EKS Node Group Role - Trust relationship and attached policies")

### 2.3. SageMaker Execution Role

1. **Trusted entity type:**
	```
	/* Lines 484-486 omitted */
	```
2. **Attach policies:**
	```
	/* Lines 489-492 omitted */
	```

3. **Add required inline EC2 policy (REQUIRED for Projects):**
	```
	/* Lines 496-521 omitted */
	```
	4. **Policy name**: `SageMakerEC2Access`

4. **Role details:**
	```
	/* Lines 525-527 omitted */
	```

### 2.4. REQUIRED: Add EC2 Permissions

Because SageMaker Projects are mandatory, EC2 permissions must be added:

1. **IAM Console** → **Roles** → `mlops-retail-prediction-dev-sagemaker-execution`
2. **Permissions** tab → **Add permissions** → **Create inline policy**
3. **JSON** tab → paste policy:

```json
{
  "Version": "2012-10-17",
  /* Lines 540-553 omitted */
  ]
}
```

4. **Review** → **Policy name**: `SageMakerEC2Access`
5. **Create policy**

{{% notice tip %}}
✅ Verify: the role should have these 4 policies attached:
- `AmazonSageMakerFullAccess` (AWS managed)
- `AmazonS3FullAccess` (AWS managed)
- `AmazonS3FullAccess` (AWS managed)
- `CloudWatchLogsFullAccess` (AWS managed)
- `SageMakerEC2Access` (the inline policy created above)
{{% /notice %}}

{{% notice warning %}}
SageMaker Unified Studio (2024+) requirements:
- ✅ **Projects are mandatory**
- ✅ **EC2 permissions are REQUIRED** (add inline policy)
- ✅ Project profile must be configured first

If EC2 permissions are missing:
- ❌ Project creation will fail
- ❌ "Insufficient permissions to describe VPCs" errors
- ❌ Studio notebooks inaccessible

✅ SOLUTION:
1. **Add the inline EC2 policy** above
2. Create the Project with "ML and generative AI model development"
3. Studio should work normally
{{% /notice %}}

## 3. Validation & Security checks

### 3.1. Verify CloudTrail

**Check CloudTrail status:**
AWS Console → CloudTrail → Trails

```
✅ mlops-retail-prediction-audit-trail: Active
✅ Multi-region trail: Enabled
✅ Management events: Read/Write
✅ Data events: S3 configured
✅ CloudWatch Logs: Integrated
```

**Verify S3 logging:**
AWS Console → S3 → mlops-cloudtrail-logs-ap-southeast-1

```
✅ Log files present: /mlops-logs/AWSLogs/[account-id]/CloudTrail/
✅ Encryption: SSE-S3 enabled
✅ Lifecycle policy: Applied
✅ Access logging: Configured
```

### 3.2. IAM Roles summary

**Go to IAM → Roles and verify:**
```
✅ mlops-retail-prediction-dev-eks-cluster-role
✅ mlops-retail-prediction-dev-eks-nodegroup-role
✅ mlops-retail-prediction-dev-sagemaker-execution
✅ CloudTrail_CloudWatchLogs_Role (auto-created)
```

**Check Trust Relationships:**
- Open each role → Trust relationships tab
- Verify trusted entities such as `eks.amazonaws.com` (EKS cluster role)
/* Lines 622-624 omitted */
- `cloudtrail.amazonaws.com` (CloudTrail logging role)

### 3.3. Security checks

**Verify CloudTrail logging:**
1. Make a test API call (e.g., list S3 buckets)
2. Check CloudTrail logs within 5–10 minutes
3. Confirm the event appears in CloudWatch Logs

**Verify IAM permissions:**
```bash
# Test SageMaker role can assume and access S3
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/mlops-retail-prediction-dev-sagemaker-execution --role-session-name test
```

**Test SageMaker role full access:**
```bash
# Test S3 access
aws s3 ls s3://mlops-retail-prediction-dev-ACCOUNT/ --profile sagemaker-test

# Test SageMaker training job permissions
aws sagemaker list-training-jobs --region us-east-1 --profile sagemaker-test

# Test EC2 permissions (if added)
# Test EC2 permissions (if added)
aws ec2 describe-vpcs --region us-east-1 --profile sagemaker-test
```

``` bash
# Test EKS roles readiness for cluster creation
aws eks describe-cluster --name test-cluster --region ap-southeast-1
```

## 4. Cost optimization & compliance
### 4.1 CloudTrail cost management — comparison table

| Item | Price | Notes / Assumptions | Example estimate |
|---|---:|---|---:|
| S3 — Standard | $0.023 / GB‑month | Hot logs (day 0–30) | 1 GB → $0.023 |
| S3 — Standard‑IA | $0.0125 / GB‑month | After 30 days (infrequent access) | 1 GB → $0.0125 |
| S3 — Glacier | $0.004 / GB‑month | Long-term retention (90–365 days) | 1 GB → $0.004 |
| S3 — Deep Archive | $0.00099 / GB‑month | Retention >365 days | 1 GB → $0.00099 |
| CloudTrail — Management events | Free (first copy) | Management API calls | — |
| CloudTrail — Data events | $0.10 / 100,000 events | S3 object-level, Lambda, etc. | 100k events → $0.10 |
| CloudTrail — Insights | $0.35 / 100,000 events | Optional anomaly detection | 100k events → $0.35 |

Sample monthly scenarios
- Minimal (small project): 0.5 GB storage (mostly Standard‑IA) + 10k data events  
  → S3 ≈ 0.5 * $0.0125 = $0.0063 ; Data events ≈ (10k/100k)*$0.10 = $0.01  
  → Total ≈ $0.016
- Typical (tens of GB, tens of thousands of events): 5 GB mixed classes + 50k data events  
  → S3 (mix) ≈ $0.02–$0.04 ; Data events ≈ $0.05 ; Insights may add $0.00–$0.35  
  → Total ~ $0.02–$0.05 (common for small projects)

Notes:
- Lifecycle transitions to IA/Glacier/Deep Archive are key for long-term cost reduction.
- Data events and Insights scale with event count — optimize sampling and only log what’s necessary.
- Verify actual spend using Billing / Cost Explorer to adjust assumptions.

## 5. Clean Up Resources (CLI guide)

> Warning: commands below will delete real resources. Confirm names (bucket, role, trail, key) before running.

### 5.1 Delete CloudTrail

PowerShell (AWS CLI):

```powershell
# Delete the trail (if name matches)
aws cloudtrail delete-trail --name mlops-retail-prediction-audit-trail

# Optionally disable CloudWatch Logs integration first
aws cloudtrail update-trail --name mlops-retail-prediction-audit-trail --cloud-watch-logs-log-group-arn "" --cloud-watch-logs-role-arn ""
```

### 5.2 Delete S3 CloudTrail bucket and contents

Note: bucket may be in `us-east-1` per this guide. Verify with `aws s3 ls` / console before deleting.

```powershell
# Remove all objects recursively
aws s3 rm s3://mlops-cloudtrail-logs-ap-southeast-1 --recursive

# Delete the bucket
# Delete the bucket
aws s3api delete-bucket --bucket mlops-cloudtrail-logs-ap-southeast-1 --region us-east-1
```

### 5.3 Schedule KMS Key deletion

KMS keys cannot be immediately deleted if in use. Schedule deletion (e.g., 7 days):

```powershell
# Find KeyId from alias
# Find KeyId from alias
$keyId = aws kms list-aliases --query "Aliases[?AliasName=='alias/mlops-retail-prediction-dev-cloudtrail-key'].TargetKeyId" --output text

# Schedule key deletion (pending window: 7-30 days)
aws kms schedule-key-deletion --key-id $keyId --pending-window-in-days 7
```

### 5.4 Remove IAM Roles & Policies (EKS / SageMaker / CloudTrail)

Safe procedure: 1) Detach managed policies 2) Delete inline policies 3) Delete role.

```powershell
# Example: delete SageMaker execution role
$roleName = 'mlops-retail-prediction-dev-sagemaker-execution'

# 1) List and detach attached managed policies
aws iam list-attached-role-policies --role-name $roleName --query 'AttachedPolicies[].PolicyArn' --output text | ForEach-Object { aws iam detach-role-policy --role-name $roleName --policy-arn $_ }

# 2) Delete inline policies
# 2) Delete inline policies
aws iam list-role-policies --role-name $roleName --query 'PolicyNames' --output text | ForEach-Object { aws iam delete-role-policy --role-name $roleName --policy-name $_ }

# 3) Delete the role
# 3) Delete the role
aws iam delete-role --role-name $roleName

# Repeat for other roles (EKS cluster/nodegroup, CloudTrail_CloudWatchLogs_Role, CI roles, etc.)
```

### 5.5 Remove Container Insights / CloudWatch integration

```powershell
# Delete CloudWatch log group (if exists)
aws logs delete-log-group --log-group-name "/aws/containerinsights/mlops-retail-cluster/application" || Write-Host 'Log group not found'

# Delete CloudTrail log group
aws logs delete-log-group --log-group-name "mlops-cloudtrail-log-group" || Write-Host 'Log group not found'

# Disable Container Insights addon from EKS (if applied)
# Disable Container Insights addon from EKS (if applied)
aws eks delete-addon --cluster-name mlops-retail-cluster --addon-name amazon-cloudwatch-observability
```

### 5.6 Delete ECR images (optional)

```powershell
# Delete images by tag
# Delete images by tag
aws ecr batch-delete-image --repository-name mlops/retail-api --image-ids imageTag=dev,imageTag=staging || Write-Host 'No matching images or already deleted'

# Delete untagged images (caution)
aws ecr describe-images --repository-name mlops/retail-api --filter tagStatus=UNTAGGED --query 'imageDetails[].imageDigest' --output text | ForEach-Object { aws ecr batch-delete-image --repository-name mlops/retail-api --image-ids imageDigest=$_ }
```

### 5.7 Stop / Delete SageMaker training jobs, endpoints, model packages

```powershell
# Stop in-progress training jobs by name pattern
aws sagemaker list-training-jobs --name-contains "retail-" --status-equals InProgress --query 'TrainingJobSummaries[].TrainingJobName' --output text | ForEach-Object { aws sagemaker stop-training-job --training-job-name $_ }

# Delete failed endpoints
aws sagemaker list-endpoints --name-contains "retail-" --query 'Endpoints[?EndpointStatus==`Failed`].EndpointName' --output text | ForEach-Object { aws sagemaker delete-endpoint --endpoint-name $_ }

# Delete pending model packages in model group (careful: keep approved ones)
aws sagemaker list-model-packages --model-package-group-name "retail-forecast-models" --model-approval-status PendingManualApproval --query 'ModelPackageSummaryList[].ModelPackageArn' --output text | ForEach-Object { aws sagemaker delete-model-package --model-package-name $_ }
```

### 5.8 Verification

```powershell
# Check trail deletion
aws cloudtrail describe-trails --query 'trailList[?Name==`mlops-retail-prediction-audit-trail`]' || Write-Host 'Trail removed or not found'

# Check bucket
/* Lines 785-844 omitted */

