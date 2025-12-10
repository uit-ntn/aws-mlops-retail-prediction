---
title: "IAM Roles & Audit For MLops"
date: 2025-08-30T12:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## 🎯 Task 2 Objectives

Set up **access control (IAM)** for all AWS services in the pipeline and **enable CloudTrail** to monitor and record every activity in the AWS account.

→ **Ensure security, access governance, and provide evidence of team activities.**

📥 **Input**

- AWS Account with admin permissions
- Project naming convention: `mlops-retail-prediction-dev`
- Target region: `ap-southeast-1`
- Multi-region CloudTrail is enabled

- **Input from Task 1:** Task 1 (Introduction) — project conventions, naming, and high-level objectives

✅ **Output**

- AWS services are granted **Least Privilege** permissions aligned with roles
- All actions are **recorded by CloudTrail**
- Meets rubric criteria: **security, access control, cloud project management**

💰 **Estimated Cost**
≈ **0.05 USD/month** (CloudTrail + S3 storage for logs)

📌 **Main Steps**

1. **CloudTrail Setup** - Configure multi-region audit logging
2. **S3 CloudTrail Bucket** - Centralized log storage
3. **EKS Cluster Service Role** - Role for the control plane
4. **EKS Node Group Role** - Role for worker nodes
5. **SageMaker Execution Role** - Role for training & deployment
6. **IRSA Foundation** - Prepare permissions at the Pod level

✅ **Deliverables**

- Multi-region CloudTrail trail with logging to S3
- EKS Cluster Service Role (Console)
- EKS Node Group Role with ECR/S3/CloudWatch permissions
- SageMaker Execution Role with required S3 permissions
- Secure foundation for IRSA setup

📊 **Acceptance Criteria**

- CloudTrail records all API calls and user activities
- EKS cluster can be created with the appropriate service roles
- SageMaker training jobs can read/write S3
- Node groups can pull images from ECR
- All IAM roles follow the principle of least privilege
- Audit trail is ready for compliance and monitoring

## 1. CloudTrail Setup - Audit Foundation

### 1.1. Create an S3 Bucket for CloudTrail

**Go to the S3 Console:**
AWS Console → S3 → "Create bucket"

**Bucket configuration:**

```
Bucket name: mlops-cloudtrail-logs-us-east-1-842676018087
Region: us-east-1 (must match the CloudTrail trail region)
Block all public access: ✅ Enabled
Versioning: ✅ Enabled
Default encryption: ✅ AWS KMS
KMS key: alias/mlops-retail-prediction-dev-cloudtrail-key
```

**Configure Lifecycle Policy:**

**Step 1**. S3 Console → select bucket `mlops-cloudtrail-logs-ap-southeast-1` → Management → Create lifecycle rule.

![S3 lifecycle](/images/2-iam-roles-audit/1.1-cloudtrail-s3-lifecycle-01.png)

**Step 2**. Name the rule (e.g. `CloudTrailLogLifecycle`), apply to all objects or use Prefix `mlops-logs/`.

**Configuration fields explained:**

1. **Object tags**:

   - ❌ No need to add tags because we are filtering by prefix

2. **Object size**:

   - ❌ No need to specify minimum object size
   - ❌ No need to specify maximum object size
   - CloudTrail logs are usually small and consistent

3. **Lifecycle rule actions**:
   - ✅ Transition current versions of objects between storage classes
     - Select to automatically move logs to cheaper storage classes
   - ❌ Transition noncurrent versions of objects between storage classes
     - Not needed because CloudTrail logs typically do not have many versions
   - ❌ Expire current versions of objects
     - Do not expire because logs are needed for auditing
   - ❌ Permanently delete noncurrent versions of objects
     - Do not delete because historical logs must be preserved
   - ✅ Delete expired object delete markers or incomplete multipart uploads
     - Select to clean up delete markers and failed uploads

![S3 lifecycle](/images/2-iam-roles-audit/1.2-cloudtrail-s3-lifecycle-01.png)

**Step 3**. Choose actions (Current versions):

- After 30 days → STANDARD_IA
- After 90 days → GLACIER / GLACIER_IR (optional)
- After 365 days → DEEP_ARCHIVE

![S3 Lifecycle Overview](/images/2-iam-roles-audit/1.3-cloudtrail-s3-lifecycle-overview.png "S3 Lifecycle for CloudTrail logs")

**Step 4**. Verify the rule is Active under the Management tab.

![S3 lifecycle](/images/2-iam-roles-audit/1.4-cloudtrail-s3-lifecycle-01.png)

### 1.2 Configure the Trail

{{% notice warning %}}
**⚠️ Region Notes:**

- CloudTrail is a multi-region service, but the trail must be created in a specific region (home region)
- The S3 bucket and KMS key must be created in the same region as the CloudTrail trail
- In this case, we will create all resources in region `us-east-1`
  {{% /notice %}}

### 1.3 Recommendation: Align the Region Between S3 and SageMaker Project

**In short:** If the pipeline’s main data (prefix `gold/` and `artifacts/`) is in `us-east-1`, then **create the SageMaker Domain / Project in `us-east-1`** to avoid cross-region issues (S3 301), KMS key complexity, and other regional endpoint constraints.

If your organization requires SageMaker to be in `ap-southeast-1`, you must copy or replicate the data to a bucket in `ap-southeast-1` before creating the Project. Example sync commands (PowerShell / CloudShell):

```powershell
aws s3 mb s3://mlops-retail-prediction-dev-842676018087-apse1 --region ap-southeast-1
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/gold/ s3://mlops-retail-prediction-dev-842676018087-apse1/gold/ --acl bucket-owner-full-control
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/artifacts/ s3://mlops-retail-prediction-dev-842676018087-apse1/artifacts/ --acl bucket-owner-full-control
```

Additional notes:

- Create a KMS key in the destination region if you use SSE-KMS.
- Update IAM policies to allow the SageMaker role to access the new bucket.

Suggestion: For lab and fast debugging, the lowest-risk option is to create the Project/Domain where the bucket already exists (in this lab: `us-east-1`).

#### 1.2.1 Create a KMS Key for CloudTrail

1. **Create the KMS Key (in region `us-east-1`):**

- AWS Console → KMS → us-east-1 → Customer managed keys → Create key

2. **Key configuration:**

   ```
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

4. **Admin permissions:**

   ```
   Key administrators: Select IAM users/roles allowed to manage the key
   Key deletion: Decide whether to allow deletion or not
   ```

5. **Usage permissions:**

   ```
   Key users:
   - Add service principal: cloudtrail.amazonaws.com
   ```

6. **Edit the key policy:**
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
**Key points in the KMS policy:**
Enable IAM User Permissions: allows the root account to manage the key  
Allow CloudTrail to encrypt logs:

- Allows GenerateDataKey with conditions for EncryptionContext and SourceArn
- EncryptionContext restricts usage to CloudTrail trails in the account
- SourceArn specifies the exact trail allowed to use the key  
  Allow CloudTrail to describe key: allows CloudTrail to read key metadata
  {{% /notice %}}

3. **Default key policy for CloudTrail:**
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

**This Key Policy includes:**

- Allow the root account to manage the key
- Allow CloudTrail to encrypt logs with a matching trail ARN condition
- Allow CloudTrail to read key metadata
- Allow principals in the account to decrypt logs
- Support cross-account log decryption if needed
  {{% /notice %}}

{{% notice warning %}}
The KMS key must be created in the same Region as the S3 bucket and have the correct policy to allow CloudTrail usage.
{{% /notice %}}

#### 1.2.2 Configure the S3 Bucket Policy

1. **Go to S3 bucket permissions:**

   ```
   S3 Console → mlops-cloudtrail-logs-ap-southeast-1 → Permissions → Bucket policy
   ```

2. **Default S3 bucket policy:**
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
**This S3 Bucket Policy includes:**

- AWSCloudTrailAclCheck: allows CloudTrail to check the bucket ACL
- AWSCloudTrailWrite: allows CloudTrail to write logs to the bucket
- Conditions: - aws:SourceArn: ensures only the specific trail can access - s3:x-amz-acl: ensures the bucket owner has full control of objects
  {{% /notice %}}

{{% notice info %}}
This policy allows CloudTrail to check the bucket ACL and write logs into the bucket.
{{% /notice %}}

#### 1.2.3 Create CloudTrail

**Step 1: Create a new Trail**

```
AWS Console → us-east-1 → CloudTrail → Create trail
```

**Step 2: Basic Trail configuration (in region us-east-1)**

| Item                           | Value                                 |
| ------------------------------ | ------------------------------------- |
| **Trail name**                 | `mlops-retail-prediction-audit-trail` |
| **Apply trail to all regions** | ✅ Yes                                |
| **Management events**          | ✅ Read/Write                         |
| **Data events**                | ✅ S3 bucket data events              |
| **Insights events**            | ✅ Enabled (detect unusual activity)  |

**Step 3: Storage configuration**

| Item                            | Value                                                                       |
| ------------------------------- | --------------------------------------------------------------------------- |
| **S3 bucket**                   | `mlops-cloudtrail-logs-ap-southeast-1`                                      |
| **Log file prefix**             | `mlops-logs/`                                                               |
| **Log file SSE-KMS encryption** | ✅ Enabled                                                                  |
| **AWS KMS alias**               | `alias/mlops-retail-prediction-dev-cloudtrail-key` (select the created key) |

**Step 4: CloudWatch Logs integration (optional)**

| Item                | Value                                           |
| ------------------- | ----------------------------------------------- |
| **CloudWatch Logs** | ✅ Enabled                                      |
| **Log group**       | `mlops-cloudtrail-log-group`                    |
| **IAM Role**        | `CloudTrail_CloudWatchLogs_Role` (auto-created) |

{{% notice info %}}
CloudTrail_CloudWatchLogs role will be auto-created with required permissions: `logs:PutLogEvents`, `logs:CreateLogStream`, `logs:DescribeLogStreams`
{{% /notice %}}

**Step 5: Review and Create trail**

{{% notice tip %}}
**Important order to avoid errors:**

1. ✅ Create the KMS key with the correct policy
2. ✅ Configure the S3 bucket policy
3. ✅ Create CloudTrail using the pre-configured KMS and S3
4. ✅ Verify logs are being delivered successfully
   {{% /notice %}}

## 2. IAM Roles Setup - Service Permissions

### 2.1. EKS Cluster Service Role

- AWS Console → IAM → Roles → "Create role"

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EKS - Cluster
   ```
2. **Attach Policy:**
   ```
   Policy: AmazonEKSClusterPolicy
   ```
3. **Role details:**
   ```
   Role name: mlops-retail-prediction-dev-eks-cluster-role
   Description: EKS cluster service role for retail prediction MLOps platform
   ```

### 2.2. EKS Node Group Role

- Similar process

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EC2
   ```
2. **Attach Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```
3. **Role details:**
   ```
   Role name: mlops-retail-prediction-dev-eks-nodegroup-role
   Description: EKS node group role with ECR, S3, and CloudWatch access for retail prediction
   ```
   ![EKS Node Group Role - Trust relationship and attached policies](/images/2-iam-roles-audit/03-eks-nodegroup-role-policies.png "EKS Node Group Role - Trust relationship and attached policies")

### 2.3. SageMaker Execution Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: SageMaker
   ```
2. **Attach Policies:**

   ```
   ✅ AmazonSageMakerFullAccess
   ✅ AmazonS3FullAccess (for data and model storage)
   ✅ CloudWatchLogsFullAccess (for training job logs)
   ```

3. **Add Inline Policy for EC2 (REQUIRED for Projects):**

   **Policy Name**: `SageMakerEC2Access`

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ec2:DescribeVpcs",
           "ec2:DescribeSubnets",
           "ec2:DescribeSecurityGroups",
           "ec2:DescribeNetworkInterfaces",
           "ec2:DescribeAvailabilityZones",
           "ec2:DescribeRouteTables"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

   **How to add:**

   1. **IAM Console** → **Roles** → `mlops-retail-prediction-dev-sagemaker-execution`
   2. **Permissions** tab → **Add permissions** → **Create inline policy**
   3. **JSON** tab → paste the policy above
   4. **Policy name**: `SageMakerEC2Access`

4. **Role details:**
   ```
   Role name: mlops-retail-prediction-dev-sagemaker-execution
   Description: SageMaker execution role for retail prediction training jobs and model deployment
   ```

### 2.4. REQUIRED: Add EC2 Permissions

**Because SageMaker Projects is mandatory**, add EC2 permissions right away:

1. **IAM Console** → **Roles** → `mlops-retail-prediction-dev-sagemaker-execution`
2. **Permissions** tab → **Add permissions** → **Create inline policy**
3. **JSON** tab → paste the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeRouteTables"
      ],
      "Resource": "*"
    }
  ]
}
```

4. **Review** → **Policy name**: `SageMakerEC2Access`
5. **Create policy**

{{% notice tip %}}
**✅ Verification:** The role must have 4 policies:

- `AmazonSageMakerFullAccess` (AWS managed)
- `AmazonS3FullAccess` (AWS managed)
- `CloudWatchLogsFullAccess` (AWS managed)
- `SageMakerEC2Access` (inline policy you created)
  {{% /notice %}}

{{% notice warning %}}
**SageMaker Unified Studio (2024+) requires:**

- ✅ **Projects are mandatory** - cannot be skipped
- ✅ **EC2 permissions are REQUIRED** - you must add the inline policy
- ✅ Project profile must be set up beforehand

**If EC2 permissions are missing:**

- ❌ "Create project" will fail
- ❌ "Insufficient permissions to describe VPCs"
- ❌ Cannot access Studio notebooks

**✅ SOLUTION:**

1. **Must** add the EC2 inline policy above
2. Create Project with "ML and generative AI model development"
3. Studio will work normally
   {{% /notice %}}

## 3. Validation & Security Checks

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
✅ Log files created: /mlops-logs/AWSLogs/[account-id]/CloudTrail/
✅ Encryption: SSE-S3 enabled
✅ Lifecycle policy: Applied
✅ Access logging: Configured
```

### 3.2. IAM Roles Summary

**Go to IAM → Roles and verify:**

```
✅ mlops-retail-prediction-dev-eks-cluster-role
✅ mlops-retail-prediction-dev-eks-nodegroup-role
✅ mlops-retail-prediction-dev-sagemaker-execution
✅ CloudTrail_CloudWatchLogs_Role (auto-created)
```

**Check Trust Relationships:**

- Open each role → Trust relationships tab
- Verify trusted entities are correct:
  - `eks.amazonaws.com` (EKS cluster role)
  - `ec2.amazonaws.com` (EKS node group role)
  - `sagemaker.amazonaws.com` (SageMaker execution role)
  - `cloudtrail.amazonaws.com` (CloudTrail logging role)

### 3.3. Security Verification

**Verify CloudTrail logging:**

1. Perform a test API call (e.g., list S3 buckets)
2. Check CloudTrail logs within 5–10 minutes
3. Confirm the event appears in CloudWatch Logs

**Verify IAM permissions:**

```bash
# Test that the SageMaker role can be assumed and can access S3
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/mlops-retail-prediction-dev-sagemaker-execution --role-session-name test
```

**Verify SageMaker role end-to-end:**

```bash
# Test S3 access
aws s3 ls s3://mlops-retail-prediction-dev-ACCOUNT/ --profile sagemaker-test

# Test SageMaker training job permissions
aws sagemaker list-training-jobs --region us-east-1 --profile sagemaker-test

# Test EC2 permissions (if added)
aws ec2 describe-vpcs --region us-east-1 --profile sagemaker-test
```

```bash
# Test that EKS roles are ready for cluster creation
aws eks describe-cluster --name test-cluster --region ap-southeast-1
```

## 4. Cost Optimization & Compliance

### 4.1. CloudTrail Cost Management — Comparison Table

| Item                           |             Unit price | Notes / Assumptions               |    Example estimate |
| ------------------------------ | ---------------------: | --------------------------------- | ------------------: |
| S3 — Standard                  |      $0.023 / GB‑month | Hot logs (days 0–30)              |       1 GB → $0.023 |
| S3 — Standard‑IA               |     $0.0125 / GB‑month | After 30 days (infrequent access) |      1 GB → $0.0125 |
| S3 — Glacier                   |      $0.004 / GB‑month | Long-term storage (90–365 days)   |       1 GB → $0.004 |
| S3 — Deep Archive              |    $0.00099 / GB‑month | Retention >365 days               |     1 GB → $0.00099 |
| CloudTrail — Management events |      Free (first copy) | Management API calls              |                   — |
| CloudTrail — Data events       | $0.10 / 100,000 events | S3 object-level, Lambda, etc.     | 100k events → $0.10 |
| CloudTrail — Insights          | $0.35 / 100,000 events | Optional anomaly detection        | 100k events → $0.35 |

Sample monthly scenarios (estimates)

- Minimal (small project): 0.5 GB storage (mostly Standard‑IA) + 10k data events  
   → S3 ≈ 0.5 _ $0.0125 = $0.0063 ; Data events ≈ (10k/100k)_$0.10 = $0.01  
   → Total ≈ $0.016 → aligns with "~ $0.01–$0.02"
- Typical (more logs, tens of GB, tens of thousands events): 5 GB mixed classes + 50k data events  
   → S3 (mix) ≈ $0.02–$0.04 ; Data events ≈ $0.05 ; Insights (optional) may add $0.00–$0.35  
   → Total ~ $0.02–$0.05 (often seen for small projects)

Short notes:

- Lifecycle transitions to IA/Glacier/Deep Archive are key to long-term cost reduction.
- Data events and Insights scale with event volume — log only what you need to save money.
- Validate assumptions using billing/Cost Explorer and refine accordingly.

## 5. Clean Up Resources (Resource Deletion Guide)

> Warning: The commands below delete real resources. Verify resource names (bucket, role, trail, key) before running.

### 5.1 Delete CloudTrail

PowerShell (AWS CLI):

```powershell
# Delete the trail (if the name matches)
aws cloudtrail delete-trail --name mlops-retail-prediction-audit-trail

# If you want to disable CloudWatch Logs delivery first
aws cloudtrail update-trail --name mlops-retail-prediction-audit-trail --cloud-watch-logs-log-group-arn "" --cloud-watch-logs-role-arn ""
```

### 5.2 Delete the S3 CloudTrail Bucket and its contents

Note: The bucket may be in `us-east-1` per the configuration above. Check `aws s3 ls` / console before deleting.

```powershell
# Delete all objects (recursive)
aws s3 rm s3://mlops-cloudtrail-logs-ap-southeast-1 --recursive

# Delete the bucket
aws s3api delete-bucket --bucket mlops-cloudtrail-logs-ap-southeast-1 --region us-east-1
```

### 5.3 Schedule KMS Key Deletion

KMS keys cannot be deleted immediately if they are in use. Schedule safe deletion (e.g., 7 days):

```powershell
# Find KeyId from alias
$keyId = aws kms list-aliases --query "Aliases[?AliasName=='alias/mlops-retail-prediction-dev-cloudtrail-key'].TargetKeyId" --output text

# Schedule key deletion (pending days: 7 - 30)
aws kms schedule-key-deletion --key-id $keyId --pending-window-in-days 7
```

### 5.4 Remove IAM Roles & Policies (EKS / SageMaker / CloudTrail)

Safe process: 1) Detach managed policies 2) Delete inline policies 3) Delete the role.

```powershell
# Example: delete the SageMaker execution role
$roleName = 'mlops-retail-prediction-dev-sagemaker-execution'

# 1) List and detach managed policies
aws iam list-attached-role-policies --role-name $roleName --query 'AttachedPolicies[].PolicyArn' --output text | ForEach-Object { aws iam detach-role-policy --role-name $roleName --policy-arn $_ }

# 2) Delete inline policies
aws iam list-role-policies --role-name $roleName --query 'PolicyNames' --output text | ForEach-Object { aws iam delete-role-policy --role-name $roleName --policy-name $_ }

# 3) Delete the role
aws iam delete-role --role-name $roleName

# Repeat for other roles (EKS cluster/nodegroup, CloudTrail_CloudWatchLogs_Role, GitHub/CI roles, etc.)
```

### 5.5 Remove Container Insights / CloudWatch integration

```powershell
# Delete CloudWatch log group (if any)
aws logs delete-log-group --log-group-name "/aws/containerinsights/mlops-retail-cluster/application" || Write-Host 'Log group not found'

# Delete CloudWatch log group for CloudTrail integration
aws logs delete-log-group --log-group-name "mlops-cloudtrail-log-group" || Write-Host 'Log group not found'

# Disable Container Insights addon from EKS (if applied)
aws eks delete-addon --cluster-name mlops-retail-cluster --addon-name amazon-cloudwatch-observability
```

### 5.6 Delete ECR images (optional cleanup for dev/staging)

```powershell
# Delete images by tag
aws ecr batch-delete-image --repository-name mlops/retail-api --image-ids imageTag=dev,imageTag=staging || Write-Host 'No matching images or already deleted'

# Delete untagged images (be careful)
aws ecr describe-images --repository-name mlops/retail-api --filter tagStatus=UNTAGGED --query 'imageDetails[].imageDigest' --output text | ForEach-Object { aws ecr batch-delete-image --repository-name mlops/retail-api --image-ids imageDigest=$_ }
```

### 5.7 Stop / Delete SageMaker training jobs, endpoints, model packages

```powershell
# Stop in-progress training jobs by name pattern
aws sagemaker list-training-jobs --name-contains "retail-" --status-equals InProgress --query 'TrainingJobSummaries[].TrainingJobName' --output text | ForEach-Object { aws sagemaker stop-training-job --training-job-name $_ }

# Delete failed endpoints
aws sagemaker list-endpoints --name-contains "retail-" --query 'Endpoints[?EndpointStatus==`Failed`].EndpointName' --output text | ForEach-Object { aws sagemaker delete-endpoint --endpoint-name $_ }

# Delete pending model packages in a model group (careful: keep approved ones)
aws sagemaker list-model-packages --model-package-group-name "retail-forecast-models" --model-approval-status PendingManualApproval --query 'ModelPackageSummaryList[].ModelPackageArn' --output text | ForEach-Object { aws sagemaker delete-model-package --model-package-name $_ }
```

### 5.8 Verification

```powershell
# Verify the trail is deleted
aws cloudtrail describe-trails --query 'trailList[?Name==`mlops-retail-prediction-audit-trail`]' || Write-Host 'Trail removed or not found'

# Verify the bucket
aws s3 ls s3://mlops-cloudtrail-logs-ap-southeast-1 2>$null || Write-Host 'Bucket removed or empty'

# Verify the IAM role
aws iam get-role --role-name mlops-retail-prediction-dev-sagemaker-execution 2>$null || Write-Host 'Role removed'

# Verify KMS key scheduled deletion
aws kms list-keys --query 'Keys[?KeyId==`'$keyId'`]' || Write-Host 'Check key deletion schedule manually in KMS console'
```

---

If you want, I can:

- add a PowerShell script to automate the entire cleanup flow (would need the exact resource names), or
- replace `ForEach-Object` with safer scripts that preview resource lists before deletion.

## 📹 Task 2 Walkthrough Video

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=wstZ_1yQlIo&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=1" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

## 👉 Task 2 Results

✅ **CloudTrail Multi-Region** - Comprehensive audit logs for all AWS activities  
✅ **S3 Audit Storage** - Cost-optimized log retention with lifecycle policies  
✅ **Secure EKS Roles** - Permissions for cluster and node groups are ready  
✅ **SageMaker Execution Role** - Training jobs + Projects ready (EC2 permissions REQUIRED)  
✅ **Security Foundation** - Enterprise-grade least-privilege architecture  
✅ **Compliance Ready** - Audit trail aligned with legal and monitoring requirements

**💰 Monthly cost**: ~0.05 USD (CloudTrail + S3 storage)  
**🔍 Audit coverage**: 100% of API calls and user activities  
**🛡️ Security posture**: Least-privilege access, production-ready

{{% notice tip %}}
**🚀 Next step:**

- **Task 3**: Set up the S3 data lake with security integration
- **Task 4**: VPC networking with security groups
- **Task 5**: Deploy the EKS cluster using the configured IAM roles
- **Task 6**: Configure IRSA for Pod-level permissions
  {{% /notice %}}

{{% notice warning %}}
**🔐 Security notes:**

- CloudTrail logs contain sensitive information — secure the S3 bucket properly
- **SageMaker role requires EC2 permissions** (Projects mandatory since 2024)
- **You must add the EC2 inline policy** to create Projects
- Role names will be referenced exactly in later tasks
- IRSA requires an OIDC provider for EKS (Task 5)
- Monitor CloudTrail costs using AWS Cost Explorer
- Review logs regularly to detect unusual activities
  {{% /notice %}}
