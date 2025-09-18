---
title: "Task 6 — ECR Container Registry"
date: 2024-01-01T00:00:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 6:**

Thiết lập Amazon Elastic Container Registry (ECR) để lưu trữ và quản lý Docker images của ứng dụng inference, đảm bảo bảo mật và khả năng truy cập từ EKS cluster.
{{% /notice %}}

## Tổng quan

**Amazon ECR (Elastic Container Registry)** là dịch vụ Docker container registry được quản lý hoàn toàn bởi AWS, tích hợp sâu với EKS và các dịch vụ AWS khác. ECR cung cấp khả năng lưu trữ, quản lý và triển khai container images một cách an toàn và hiệu quả.

### Kiến trúc ECR trong MLOps Pipeline

{{< mermaid >}}
graph TB
    subgraph "Development Environment"
        DEV[Developer<br/>Local Docker Build]
        CI[CI/CD Pipeline<br/>Jenkins/GitHub Actions]
    end
    
    subgraph "AWS ECR"
        ECR[ECR Repository<br/>retail-forecast]
        SCAN[Vulnerability Scanning<br/>Scan on Push]
        TAGS[Image Tags<br/>latest, git-sha, v1.0.0]
    end
    
    subgraph "EKS Cluster"
        NODES[Worker Nodes<br/>Pull Images]
        PODS[Application Pods<br/>Running Containers]
    end
    
    subgraph "Security & Access"
        IAM[IAM Policies<br/>ECR Permissions]
        IRSA[IRSA Roles<br/>Pod Identity]
    end
    
    DEV --> ECR
    CI --> ECR
    ECR --> SCAN
    ECR --> TAGS
    ECR --> NODES
    NODES --> PODS
    IAM --> ECR
    IRSA --> NODES
{{< /mermaid >}}

### Thành phần chính

1. **ECR Repository**: Kho lưu trữ Docker images
2. **Image Scanning**: Phát hiện lỗ hổng bảo mật
3. **Lifecycle Policies**: Quản lý vòng đời images
4. **IAM Integration**: Kiểm soát truy cập
5. **Replication**: Sao chép multi-region (optional)

---

## 1. Alternative: AWS Console Implementation

### 1.1. ECR Repository Creation

1. **Navigate to ECR Console:**
   - Đăng nhập AWS Console
   - Navigate to ECR service
   - Region: ap-southeast-1
   - Chọn "Create repository"

{{< imgborder src="/images/06-ecr-registry/01-create-repository.png" title="Navigate to ECR Console" >}}

2. **Repository Configuration:**
   ```
   Visibility settings: Private
   Repository name: retail-forecast
   Tag immutability: Disabled (Allow mutable tags)
   Image scan settings: ✅ Scan on push
   ```

{{< imgborder src="/images/06-ecr-registry/02-repository-config.png" title="Configure ECR repository settings" >}}

3. **Advanced Configuration:**
   ```
   Encryption settings: AES-256 (Default)
   
   Optional: KMS encryption
   ✅ Use KMS encryption
   KMS key: alias/aws/ecr (AWS managed)
   ```

{{< imgborder src="/images/06-ecr-registry/03-encryption-settings.png" title="Configure encryption settings" >}}

4. **Repository Creation Complete:**
   - Verify repository appears in ECR console
   - Note repository URI: `123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast`

{{< imgborder src="/images/06-ecr-registry/04-repository-created.png" title="ECR repository created successfully" >}}

### 1.2. Repository Policies Configuration

1. **Access Repository Permissions:**
   - Chọn repository `retail-forecast`
   - Click "Permissions" tab
   - Chọn "Edit policy JSON"

{{< imgborder src="/images/06-ecr-registry/05-repository-permissions.png" title="Access repository permissions" >}}

2. **Configure Repository Policy:**
   ```json
   {
     "Version": "2008-10-17",
     "Statement": [
       {
         "Sid": "AllowEKSNodeGroupAccess",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-nodegroup-role"
         },
         "Action": [
           "ecr:BatchCheckLayerAvailability",
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage"
         ]
       },
       {
         "Sid": "AllowCICDPushAccess", 
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-cicd-role"
         },
         "Action": [
           "ecr:BatchCheckLayerAvailability",
           "ecr:GetDownloadUrlForLayer", 
           "ecr:BatchGetImage",
           "ecr:PutImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload"
         ]
       }
     ]
   }
   ```

{{< imgborder src="/images/06-ecr-registry/06-repository-policy.png" title="Configure repository access policy" >}}

### 1.3. Lifecycle Policy Setup

1. **Navigate to Lifecycle Policy:**
   - Repository → "Lifecycle policy" tab
   - Chọn "Create rule"

{{< imgborder src="/images/06-ecr-registry/07-lifecycle-policy.png" title="Create lifecycle policy" >}}

2. **Configure Lifecycle Rules:**
   
   **Rule 1 - Keep Latest Images:**
   ```
   Rule priority: 1
   Description: Keep latest 10 images
   Image status: Any
   
   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 10
   
   Action: expire
   ```

   **Rule 2 - Remove Old Untagged Images:**
   ```
   Rule priority: 2  
   Description: Delete untagged images after 1 day
   Image status: Untagged
   
   Match criteria:
   - Count type: sinceImagePushed
   - Count number: 1
   - Count unit: days
   
   Action: expire
   ```

{{< imgborder src="/images/06-ecr-registry/08-lifecycle-rules.png" title="Configure lifecycle policy rules" >}}

### 1.4. Vulnerability Scanning Verification

1. **Check Scan Settings:**
   - Repository → "Image scan settings" tab
   - Verify "Scan on push" is enabled
   - Review scan results after image push

{{< imgborder src="/images/06-ecr-registry/09-scan-settings.png" title="Verify image scanning configuration" >}}

2. **Manual Scan Trigger:**
   - Repository → Images tab
   - Select image → Actions → "Scan"
   - View scan results

{{< imgborder src="/images/06-ecr-registry/10-manual-scan.png" title="Trigger manual vulnerability scan" >}}

### 1.5. Docker Authentication & Push

1. **Get ECR Login Token:**
   ```bash
   # Get login command
   aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com
   ```

{{< imgborder src="/images/06-ecr-registry/11-docker-login.png" title="Authenticate Docker with ECR" >}}

2. **Build and Tag Image:**
   ```bash
   # Build Docker image
   docker build -t retail-forecast .
   
   # Tag for ECR
   docker tag retail-forecast:latest 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:latest
   docker tag retail-forecast:latest 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:$(git rev-parse --short HEAD)
   ```

{{< imgborder src="/images/06-ecr-registry/12-docker-build-tag.png" title="Build and tag Docker image" >}}

3. **Push Image to ECR:**
   ```bash
   # Push images
   docker push 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:latest
   docker push 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:$(git rev-parse --short HEAD)
   ```

{{< imgborder src="/images/06-ecr-registry/13-docker-push.png" title="Push Docker image to ECR" >}}

### 1.6. Verify Images in ECR

1. **Check Repository Images:**
   - ECR Console → retail-forecast repository
   - Verify images appear with correct tags
   - Check image sizes and push timestamps

{{< imgborder src="/images/06-ecr-registry/14-verify-images.png" title="Verify images in ECR repository" >}}

2. **Review Scan Results:**
   - Click on image digest
   - Review vulnerability scan results
   - Check severity levels (Critical, High, Medium, Low)

{{< imgborder src="/images/06-ecr-registry/15-scan-results.png" title="Review vulnerability scan results" >}}

{{% notice success %}}
**🎯 Console Implementation Complete!**

ECR Repository đã được tạo thành công với:
- ✅ Private repository `retail-forecast`
- ✅ Vulnerability scanning enabled
- ✅ Repository policies configured
- ✅ Lifecycle policies for image cleanup
- ✅ Docker images pushed successfully
- ✅ EKS node access configured
{{% /notice %}}

{{% notice info %}}
**💡 Console vs Terraform:**

**Console Advantages:**
- ✅ Visual repository management
- ✅ Real-time scan results viewing
- ✅ Easy policy editing interface
- ✅ Immediate image verification

**Terraform Advantages:**
- ✅ Infrastructure as Code
- ✅ Automated repository setup
- ✅ Version-controlled policies
- ✅ Consistent multi-environment deployment

Khuyến nghị: Console cho learning, Terraform cho production.
{{% /notice %}}

---

## 2. ECR Repository Terraform Configuration

### 2.1. Main ECR Resource

**Tạo file `aws/infra/ecr.tf`:**

```hcl
# ECR Repository for ML inference application
resource "aws_ecr_repository" "retail_forecast" {
  name                 = var.ecr_repository_name
  image_tag_mutability = var.ecr_image_tag_mutability

  image_scanning_configuration {
    scan_on_push = var.ecr_scan_on_push
  }

  encryption_configuration {
    encryption_type = var.ecr_encryption_type
    kms_key         = var.ecr_encryption_type == "KMS" ? aws_kms_key.ecr[0].arn : null
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-repo"
    Type = "ecr-repository"
    Application = "ml-inference"
  })
}

# KMS Key for ECR encryption (optional)
resource "aws_kms_key" "ecr" {
  count = var.ecr_encryption_type == "KMS" ? 1 : 0
  
  description             = "KMS key for ECR repository encryption"
  deletion_window_in_days = var.kms_key_deletion_window

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-kms"
    Type = "kms-key"
    Service = "ecr"
  })
}

resource "aws_kms_alias" "ecr" {
  count = var.ecr_encryption_type == "KMS" ? 1 : 0
  
  name          = "alias/${var.project_name}-${var.environment}-ecr"
  target_key_id = aws_kms_key.ecr[0].key_id
}

# ECR Repository Policy
resource "aws_ecr_repository_policy" "retail_forecast" {
  repository = aws_ecr_repository.retail_forecast.name

  policy = jsonencode({
    Version = "2008-10-17"
    Statement = [
      {
        Sid    = "AllowEKSNodeGroupAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_nodegroup.arn
        }
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
      },
      {
        Sid    = "AllowCICDAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.cicd_role.arn
        }
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
      }
    ]
  })
}

# ECR Lifecycle Policy
resource "aws_ecr_lifecycle_policy" "retail_forecast" {
  repository = aws_ecr_repository.retail_forecast.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last ${var.ecr_lifecycle_policy.keep_last_images} images"
        selection = {
          tagStatus     = "any"
          countType     = "imageCountMoreThan"
          countNumber   = var.ecr_lifecycle_policy.keep_last_images
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Delete untagged images after ${var.ecr_lifecycle_policy.untagged_expire_days} days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = var.ecr_lifecycle_policy.untagged_expire_days
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# CI/CD IAM Role for ECR access
resource "aws_iam_role" "cicd_role" {
  name = "${var.project_name}-${var.environment}-cicd-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = ["codebuild.amazonaws.com", "codepipeline.amazonaws.com"]
        }
        Action = "sts:AssumeRole"
      },
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "aws:RequestedRegion" = var.aws_region
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cicd-role"
    Type = "iam-role"
    Service = "cicd"
  })
}

# CI/CD ECR Policy
resource "aws_iam_policy" "cicd_ecr_policy" {
  name        = "${var.project_name}-${var.environment}-cicd-ecr-policy"
  description = "Policy for CI/CD to access ECR"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:DescribeRepositories",
          "ecr:GetRepositoryPolicy",
          "ecr:ListImages",
          "ecr:DescribeImages",
          "ecr:BatchDeleteImage"
        ]
        Resource = aws_ecr_repository.retail_forecast.arn
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cicd-ecr-policy"
    Type = "iam-policy"
    Service = "cicd"
  })
}

# Attach ECR policy to CI/CD role
resource "aws_iam_role_policy_attachment" "cicd_ecr_policy" {
  role       = aws_iam_role.cicd_role.name
  policy_arn = aws_iam_policy.cicd_ecr_policy.arn
}

# Data source for current AWS account
data "aws_caller_identity" "current" {}
```

### 2.2. Additional Variables

**Thêm vào `aws/infra/variables.tf`:**

```hcl
# ECR Configuration
variable "ecr_repository_name" {
  description = "Name of the ECR repository"
  type        = string
  default     = "retail-forecast"
}

variable "ecr_image_tag_mutability" {
  description = "The tag mutability setting for the repository. Must be one of: MUTABLE or IMMUTABLE"
  type        = string
  default     = "MUTABLE"
  
  validation {
    condition     = contains(["MUTABLE", "IMMUTABLE"], var.ecr_image_tag_mutability)
    error_message = "ECR image tag mutability must be MUTABLE or IMMUTABLE."
  }
}

variable "ecr_scan_on_push" {
  description = "Indicates whether images are scanned after being pushed to the repository"
  type        = bool
  default     = true
}

variable "ecr_encryption_type" {
  description = "The encryption type to use for the repository. Valid values: AES256, KMS"
  type        = string
  default     = "AES256"
  
  validation {
    condition     = contains(["AES256", "KMS"], var.ecr_encryption_type)
    error_message = "ECR encryption type must be AES256 or KMS."
  }
}

variable "ecr_lifecycle_policy" {
  description = "ECR lifecycle policy configuration"
  type = object({
    keep_last_images        = number
    untagged_expire_days   = number
  })
  default = {
    keep_last_images      = 10
    untagged_expire_days  = 1
  }
}

# Replication Configuration (Optional)
variable "ecr_replication_destinations" {
  description = "List of replication destinations"
  type = list(object({
    region      = string
    registry_id = string
  }))
  default = []
}

variable "enable_ecr_replication" {
  description = "Enable ECR replication"
  type        = bool
  default     = false
}
```

### 2.3. ECR Replication (Optional)

**Thêm vào `aws/infra/ecr.tf`:**

```hcl
# ECR Replication Configuration (optional)
resource "aws_ecr_replication_configuration" "main" {
  count = var.enable_ecr_replication && length(var.ecr_replication_destinations) > 0 ? 1 : 0

  replication_configuration {
    rule {
      dynamic "destination" {
        for_each = var.ecr_replication_destinations
        content {
          region      = destination.value.region
          registry_id = destination.value.registry_id
        }
      }
      
      repository_filter {
        filter      = var.ecr_repository_name
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}

# ECR Registry Scanning Configuration
resource "aws_ecr_registry_scanning_configuration" "main" {
  scan_type = var.ecr_enhanced_scanning ? "ENHANCED" : "BASIC"

  dynamic "rule" {
    for_each = var.ecr_enhanced_scanning ? [1] : []
    content {
      scan_frequency = "SCAN_ON_PUSH"
      repository_filter {
        filter      = "*"
        filter_type = "WILDCARD"
      }
    }
  }
}
```

### 2.4. Environment-specific Configuration

**Cập nhật `aws/terraform.tfvars`:**

```hcl
# ECR Configuration
ecr_repository_name       = "retail-forecast"
ecr_image_tag_mutability = "MUTABLE"
ecr_scan_on_push         = true
ecr_encryption_type      = "AES256"

ecr_lifecycle_policy = {
  keep_last_images     = 10
  untagged_expire_days = 1
}

# Replication (uncomment for multi-region setup)
# enable_ecr_replication = true
# ecr_replication_destinations = [
#   {
#     region      = "us-west-2"
#     registry_id = "123456789012"
#   }
# ]
```

### 2.5. Outputs for ECR

**Thêm vào `aws/infra/output.tf`:**

```hcl
# ECR Repository Outputs
output "ecr_repository_url" {
  description = "URL of the ECR repository"
  value       = aws_ecr_repository.retail_forecast.repository_url
}

output "ecr_repository_arn" {
  description = "ARN of the ECR repository"
  value       = aws_ecr_repository.retail_forecast.arn
}

output "ecr_repository_name" {
  description = "Name of the ECR repository"
  value       = aws_ecr_repository.retail_forecast.name
}

output "ecr_repository_registry_id" {
  description = "Registry ID of the ECR repository"
  value       = aws_ecr_repository.retail_forecast.registry_id
}

output "cicd_role_arn" {
  description = "ARN of the CI/CD IAM role"
  value       = aws_iam_role.cicd_role.arn
}
```

---

## 3. Docker Image Build và Push Automation

### 3.1. Dockerfile Example

**Tạo file `server/Dockerfile`:**

```dockerfile
# Multi-stage build for ML inference API
FROM python:3.9-slim as builder

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.9-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Set Python path
ENV PATH=/home/appuser/.local/bin:$PATH

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 3.2. Build Script

**Tạo file `scripts/build-and-push.sh`:**

```bash
#!/bin/bash

set -e

# Configuration
PROJECT_NAME="mlops-retail-forecast"
ENVIRONMENT="dev"
AWS_REGION="ap-southeast-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPOSITORY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/retail-forecast"

# Get Git commit hash
GIT_COMMIT=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Image tags
IMAGE_TAG_LATEST="latest"
IMAGE_TAG_COMMIT="${GIT_COMMIT}"
IMAGE_TAG_BRANCH="${GIT_BRANCH}-${GIT_COMMIT}"

echo "🚀 Building and pushing Docker image to ECR..."
echo "Repository: ${ECR_REPOSITORY}"
echo "Tags: ${IMAGE_TAG_LATEST}, ${IMAGE_TAG_COMMIT}, ${IMAGE_TAG_BRANCH}"

# Login to ECR
echo "🔐 Logging in to ECR..."
aws ecr get-login-password --region ${AWS_REGION} | \
    docker login --username AWS --password-stdin ${ECR_REPOSITORY}

# Build Docker image
echo "🏗️ Building Docker image..."
docker build \
    --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg GIT_COMMIT=${GIT_COMMIT} \
    --build-arg GIT_BRANCH=${GIT_BRANCH} \
    -t retail-forecast:${IMAGE_TAG_COMMIT} \
    -f server/Dockerfile \
    server/

# Tag images
echo "🏷️ Tagging images..."
docker tag retail-forecast:${IMAGE_TAG_COMMIT} ${ECR_REPOSITORY}:${IMAGE_TAG_LATEST}
docker tag retail-forecast:${IMAGE_TAG_COMMIT} ${ECR_REPOSITORY}:${IMAGE_TAG_COMMIT}
docker tag retail-forecast:${IMAGE_TAG_COMMIT} ${ECR_REPOSITORY}:${IMAGE_TAG_BRANCH}

# Push images
echo "📤 Pushing images to ECR..."
docker push ${ECR_REPOSITORY}:${IMAGE_TAG_LATEST}
docker push ${ECR_REPOSITORY}:${IMAGE_TAG_COMMIT}
docker push ${ECR_REPOSITORY}:${IMAGE_TAG_BRANCH}

# Clean up local images
echo "🧹 Cleaning up local images..."
docker rmi retail-forecast:${IMAGE_TAG_COMMIT} || true
docker rmi ${ECR_REPOSITORY}:${IMAGE_TAG_LATEST} || true
docker rmi ${ECR_REPOSITORY}:${IMAGE_TAG_COMMIT} || true
docker rmi ${ECR_REPOSITORY}:${IMAGE_TAG_BRANCH} || true

echo "✅ Build and push completed successfully!"
echo "📍 Image URLs:"
echo "   - ${ECR_REPOSITORY}:${IMAGE_TAG_LATEST}"
echo "   - ${ECR_REPOSITORY}:${IMAGE_TAG_COMMIT}"
echo "   - ${ECR_REPOSITORY}:${IMAGE_TAG_BRANCH}"
```

### 3.3. PowerShell Build Script (Windows)

**Tạo file `scripts/build-and-push.ps1`:**

```powershell
# Build and push script for Windows
param(
    [string]$Environment = "dev",
    [string]$Region = "ap-southeast-1"
)

# Configuration
$ProjectName = "mlops-retail-forecast"
$AwsAccountId = (aws sts get-caller-identity --query Account --output text)
$EcrRepository = "$AwsAccountId.dkr.ecr.$Region.amazonaws.com/retail-forecast"

# Get Git information
$GitCommit = (git rev-parse --short HEAD)
$GitBranch = (git rev-parse --abbrev-ref HEAD)

# Image tags
$ImageTagLatest = "latest"
$ImageTagCommit = $GitCommit
$ImageTagBranch = "$GitBranch-$GitCommit"

Write-Host "🚀 Building and pushing Docker image to ECR..." -ForegroundColor Green
Write-Host "Repository: $EcrRepository" -ForegroundColor Yellow
Write-Host "Tags: $ImageTagLatest, $ImageTagCommit, $ImageTagBranch" -ForegroundColor Yellow

# Login to ECR
Write-Host "🔐 Logging in to ECR..." -ForegroundColor Blue
$LoginCommand = aws ecr get-login-password --region $Region
$LoginCommand | docker login --username AWS --password-stdin $EcrRepository

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to login to ECR"
    exit 1
}

# Build Docker image
Write-Host "🏗️ Building Docker image..." -ForegroundColor Blue
$BuildDate = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

docker build `
    --build-arg BUILD_DATE=$BuildDate `
    --build-arg GIT_COMMIT=$GitCommit `
    --build-arg GIT_BRANCH=$GitBranch `
    -t "retail-forecast:$ImageTagCommit" `
    -f server/Dockerfile `
    server/

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to build Docker image"
    exit 1
}

# Tag images
Write-Host "🏷️ Tagging images..." -ForegroundColor Blue
docker tag "retail-forecast:$ImageTagCommit" "${EcrRepository}:$ImageTagLatest"
docker tag "retail-forecast:$ImageTagCommit" "${EcrRepository}:$ImageTagCommit"
docker tag "retail-forecast:$ImageTagCommit" "${EcrRepository}:$ImageTagBranch"

# Push images
Write-Host "📤 Pushing images to ECR..." -ForegroundColor Blue
docker push "${EcrRepository}:$ImageTagLatest"
docker push "${EcrRepository}:$ImageTagCommit"
docker push "${EcrRepository}:$ImageTagBranch"

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to push images to ECR"
    exit 1
}

# Clean up local images
Write-Host "🧹 Cleaning up local images..." -ForegroundColor Blue
docker rmi "retail-forecast:$ImageTagCommit" -ErrorAction SilentlyContinue
docker rmi "${EcrRepository}:$ImageTagLatest" -ErrorAction SilentlyContinue
docker rmi "${EcrRepository}:$ImageTagCommit" -ErrorAction SilentlyContinue
docker rmi "${EcrRepository}:$ImageTagBranch" -ErrorAction SilentlyContinue

Write-Host "✅ Build and push completed successfully!" -ForegroundColor Green
Write-Host "📍 Image URLs:" -ForegroundColor Yellow
Write-Host "   - ${EcrRepository}:$ImageTagLatest" -ForegroundColor White
Write-Host "   - ${EcrRepository}:$ImageTagCommit" -ForegroundColor White
Write-Host "   - ${EcrRepository}:$ImageTagBranch" -ForegroundColor White
```

---

## 4. ECR Integration với EKS

### 4.1. Image Pull Secrets (if needed)

```yaml
# k8s/image-pull-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ecr-registry-secret
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### 4.2. Deployment Configuration

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-forecast-api
  namespace: default
  labels:
    app: retail-forecast-api
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-forecast-api
  template:
    metadata:
      labels:
        app: retail-forecast-api
        version: v1
    spec:
      serviceAccountName: retail-forecast-sa  # IRSA enabled
      containers:
      - name: api
        image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: AWS_DEFAULT_REGION
          value: "ap-southeast-1"
        - name: AWS_REGION
          value: "ap-southeast-1"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
      # Note: imagePullSecrets not needed with IRSA
      # imagePullSecrets:
      # - name: ecr-registry-secret
```

---

## 5. Monitoring và Troubleshooting

### 5.1. ECR Metrics Monitoring

```bash
# Check repository size
aws ecr describe-repositories \
    --repository-names retail-forecast \
    --region ap-southeast-1

# List images
aws ecr list-images \
    --repository-name retail-forecast \
    --region ap-southeast-1

# Describe specific image
aws ecr describe-images \
    --repository-name retail-forecast \
    --image-ids imageTag=latest \
    --region ap-southeast-1
```

### 5.2. Scan Results Analysis

```bash
# Get scan results
aws ecr describe-image-scan-findings \
    --repository-name retail-forecast \
    --image-id imageTag=latest \
    --region ap-southeast-1

# Start manual scan
aws ecr start-image-scan \
    --repository-name retail-forecast \
    --image-id imageTag=latest \
    --region ap-southeast-1
```

### 5.3. Common Issues và Solutions

**Issue 1: Authentication Failed**
```bash
# Solution: Refresh ECR login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com
```

**Issue 2: Image Pull Failed from EKS**
```bash
# Check node group IAM permissions
kubectl describe pod <pod-name>

# Verify IRSA configuration
kubectl describe serviceaccount retail-forecast-sa
```

**Issue 3: Repository Policy Issues**
```bash
# Verify repository policy
aws ecr get-repository-policy \
    --repository-name retail-forecast \
    --region ap-southeast-1
```

---

## 6. Security Best Practices

### 6.1. Image Scanning Strategy

```hcl
# Enable enhanced scanning (requires additional cost)
variable "ecr_enhanced_scanning" {
  description = "Enable enhanced scanning for ECR"
  type        = bool
  default     = false
}
```

### 6.2. Repository Encryption

```hcl
# Use KMS encryption for sensitive images
ecr_encryption_type = "KMS"
```

### 6.3. Access Control

```json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "RestrictToSpecificRoles",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-nodegroup-role",
          "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-cicd-role"
        ]
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-southeast-1"
        }
      }
    }
  ]
}
```

---

{{% notice success %}}
**🎯 Task 6 Complete!**

ECR Container Registry đã được triển khai thành công với:

✅ **Private repository** `retail-forecast` với security scanning  
✅ **Lifecycle policies** để quản lý image versions  
✅ **IAM integration** với EKS node groups và CI/CD  
✅ **Repository policies** cho fine-grained access control  
✅ **Docker build/push** automation scripts  
✅ **Monitoring** và troubleshooting capabilities  

**Next Steps:**
- Task 7: Docker build automation trong CI/CD
- Task 8: S3 Data Lake setup
- Task 9: SageMaker integration
{{% /notice %}}

{{% notice tip %}}
**💡 Production Considerations:**

- Sử dụng **KMS encryption** cho sensitive container images
- Enable **enhanced scanning** cho comprehensive vulnerability detection  
- Implement **image signing** với AWS Signer cho supply chain security
- Configure **cross-region replication** cho disaster recovery
- Set up **CloudWatch alarms** cho repository metrics monitoring
- Use **immutable tags** trong production environment
{{% /notice %}}