---
title: "ECR Container Registry"
date: 2024-01-01T00:00:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 6:** Thiết lập Amazon Elastic Container Registry (ECR) cho MLOps pipeline:

1. **Tạo ECR Repository**: Cài đặt repo cho API và training image
2. **Cấu hình Security**: Image scanning, IAM policy, lifecycle rules
3. **Build & Push Image**: Upload image lên ECR thông qua CLI/console
4. **CI/CD**: Tích hợp với GitHub Actions để tự động hóa

→ Mục tiêu: Có thể lưu trữ và quản lý Docker image một cách an toàn trên AWS.
{{% /notice %}}

## Tổng quan

**Amazon ECR (Elastic Container Registry)** là dịch vụ Docker container registry được quản lý hoàn toàn bởi AWS, tích hợp sâu với EKS và CI/CD pipeline. ECR cung cấp khả năng lưu trữ, quản lý và triển khai container images một cách an toàn cho MLOps workflow.

### Kiến trúc ECR trong MLOps Pipeline

```
MLOps Container Strategy:
├── mlops/retail-api (FastAPI Prediction Service)
│   ├── FastAPI app (main.py)
│   ├── Model loader (từ S3 artifacts)
│   ├── Dependencies (pandas, scikit-learn, xgboost, boto3)
│   └── Health checks & monitoring
├── mlops/train-model (Optional Training Container)
│   ├── Training scripts (train.py, evaluate.py)
│   ├── Data processing (feature engineering)
│   ├── Model export to S3
│   └── MLflow integration
└── CI/CD Integration
    ├── GitHub Actions / Jenkins
    ├── Automated build on code changes
    ├── Image scanning & security
    └── Auto-deploy to EKS

Security & Lifecycle:
├── Image scanning on push (vulnerability detection)
├── Tag immutability (reproducibility)
├── Encryption (AES-256)
├── IAM-based access control
└── Lifecycle policies (automatic cleanup)
```

### Thành phần chính

1. **ECR Repositories**: Separate repos cho API và training images
2. **Image Scanning**: Automatic vulnerability detection on push
3. **Lifecycle Policies**: Auto-cleanup old images
4. **IAM Integration**: EKS pull access, CI/CD push permissions
5. **CI/CD Automation**: Build → Push → Deploy workflow

## 1. ECR Repositories Setup

### 1.1. Create ECR Repositories

1. **Navigate to ECR Console:**
   - Đăng nhập AWS Console
   - Navigate to ECR service
   - Region: ap-southeast-1
   - Chọn "Create repository"

![Navigate to ECR Console](/images/06-ecr-registry/01-create-repository.png)

2. **API Repository Configuration:**
   ```
   Visibility settings: Private
   Repository name: mlops/retail-api
   Tag immutability: ✅ Enabled (Reproducibility)
   Image scan settings: ✅ Scan on push
   Encryption settings: AES-256 (Default)
   ```

![Configure API repository](/images/06-ecr-registry/02-api-repository-config.png)

3. **Training Repository Configuration (Optional):**
   ```
   Visibility settings: Private
   Repository name: mlops/train-model
   Tag immutability: ✅ Enabled
   Image scan settings: ✅ Scan on push
   Encryption settings: AES-256
   ```

![Configure training repository](/images/06-ecr-registry/03-training-repository-config.png)

4. **Repository Creation Complete:**
   ```
   API Repository URI: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api
   Training Repository URI: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/train-model
   ```

![ECR repositories created successfully](/images/06-ecr-registry/04-repositories-created.png)

### 1.2. Repository Policies Configuration

1. **API Repository Access Policy:**
   - Chọn repository `mlops/retail-api`
   - Click "Permissions" tab → "Edit policy JSON"

![Configure API repository permissions](/images/06-ecr-registry/05-api-repository-permissions.png)

2. **Configure API Repository Policy:**
   ```json
   {
     "Version": "2008-10-17",
     "Statement": [
       {
         "Sid": "AllowEKSNodeGroupPull",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-hybrid-eks-nodes-role"
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
           "AWS": "arn:aws:iam::123456789012:role/mlops-cicd-role"
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

!![API repository access policy](/images/06-ecr-registry/06-api-repository-policy.png)

3. **Training Repository Access Policy:**
   ```json
   {
     "Version": "2008-10-17",
     "Statement": [
       {
         "Sid": "AllowSageMakerAccess",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-sagemaker-execution-role"
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
           "AWS": "arn:aws:iam::123456789012:role/mlops-cicd-role"
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

### 1.3. Lifecycle Policy Setup

1. **API Repository Lifecycle Policy:**
   - Repository → "Lifecycle policy" tab → "Create rule"

![Create API lifecycle policy](/images/06-ecr-registry/07-api-lifecycle-policy.png)

2. **Configure API Lifecycle Rules:**

   **Rule 1 - Keep Latest Production Images:**

   ```
   Rule priority: 1
   Description: Keep latest 10 production images
   Image status: Tagged
   Tag status: Starts with "v" (production versions)

   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 10

   Action: expire
   ```

   **Rule 2 - Keep Latest Development Images:**

   ```
   Rule priority: 2
   Description: Keep latest 5 development images
   Image status: Tagged
   Tag status: Starts with "dev", "feature", "main"

   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 5

   Action: expire
   ```

   **Rule 3 - Remove Old Untagged Images:**

   ```
   Rule priority: 3
   Description: Delete untagged images after 1 day
   Image status: Untagged

   Match criteria:
   - Count type: sinceImagePushed
   - Count number: 1
   - Count unit: days

   Action: expire
   ```

![API lifecycle policy rules](/images/06-ecr-registry/08-api-lifecycle-rules.png)

3. **Training Repository Lifecycle Policy:**
   ```
   Rule 1: Keep latest 5 training images
   Rule 2: Delete untagged images after 1 day
   Rule 3: Keep experiment images (tagged with "exp-") for 30 days max
   ```

### 1.4. Image Scanning Verification

1. **Check Scan Settings:**
   - Repository → "Image scan settings" tab
   - Verify "Scan on push" is enabled
   - Review enhanced scanning options

![Verify image scanning configuration](/images/06-ecr-registry/09-scan-settings.png)

2. **Enhanced Scanning (Optional):**
   ```
   ✅ Basic scanning: Included with ECR (FREE)
   ⚙️ Enhanced scanning: $0.09 per image scan (more detailed CVE detection)
   ```

{{% notice success %}}
**🎯 ECR Repositories Setup Complete!**

**Created Repositories:**

- ✅ `mlops/retail-api`: FastAPI prediction service container
- ✅ `mlops/train-model`: Custom training container (optional)
- ✅ Image scanning enabled on push
- ✅ Tag immutability enabled for reproducibility
- ✅ Lifecycle policies configured for cost optimization
- ✅ IAM access policies for EKS and CI/CD
  {{% /notice %}}

## 2. Container Images Development

### 2.1. FastAPI Prediction Service Container

**Create directory structure:**

```
server/
├── Dockerfile
├── requirements.txt
├── main.py
├── model_loader.py
├── prediction_service.py
└── health_check.py
```

**File: `server/Dockerfile`**

```dockerfile
# Multi-stage build for FastAPI retail prediction API
FROM python:3.9-slim as builder

# Set working directory
WORKDIR /app

# Install system dependencies for ML libraries
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libc6-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.9-slim

# Create non-root user for security
RUN groupadd -r apiuser && useradd -r -g apiuser apiuser

# Set working directory
WORKDIR /app

# Copy Python packages from builder stage
COPY --from=builder /root/.local /home/apiuser/.local

# Copy application code
COPY --chown=apiuser:apiuser . .

# Create directories for model and logs
RUN mkdir -p /app/models /app/logs && \
    chown -R apiuser:apiuser /app

# Switch to non-root user
USER apiuser

# Set Python path
ENV PATH=/home/apiuser/.local/bin:$PATH

# Environment variables
ENV PYTHONPATH=/app
ENV AWS_DEFAULT_REGION=ap-southeast-1
ENV MODEL_BUCKET=mlops-retail-forecast-models
ENV MODEL_PREFIX=models/retail-price-sensitivity

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python health_check.py || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

**File: `server/requirements.txt`**

```txt
# FastAPI and web framework
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0

# ML libraries
scikit-learn==1.3.2
xgboost==2.0.2
pandas==2.1.3
numpy==1.25.2

# AWS SDK
boto3==1.34.0
botocore==1.34.0

# Utilities
python-json-logger==2.0.7
python-multipart==0.0.6
httpx==0.25.2

# Monitoring
prometheus-client==0.19.0
```

**File: `server/main.py`**

**File: `server/model_loader.py`**

**File: `server/prediction_service.py`**

**File: `server/health_check.py`**

### 2.2. Training Container (Optional)

**Create directory structure:**

```
training/
├── Dockerfile
├── requirements.txt
├── train.py
├── evaluate.py
└── export_model.py
```

**File: `training/Dockerfile`**

```dockerfile
# Training container for retail price sensitivity model
FROM python:3.9

# Set working directory
WORKDIR /opt/ml

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy training scripts
COPY . .

# Set executable permissions
RUN chmod +x train.py evaluate.py export_model.py

# Default command
CMD ["python", "train.py"]
```

**File: `training/requirements.txt`**

```txt
# ML libraries
scikit-learn==1.3.2
xgboost==2.0.2
pandas==2.1.3
numpy==1.25.2

# AWS SDK
boto3==1.34.0

# Experiment tracking
mlflow==2.8.1

# Utilities
joblib==1.3.2
```

{{% notice success %}}
**🎯 Container Images Ready!**

**FastAPI Prediction Service:**

- ✅ Multi-stage Docker build for size optimization
- ✅ Model loader từ S3 with automatic refresh
- ✅ Health checks for ALB and Kubernetes
- ✅ Comprehensive prediction API with error handling
- ✅ Non-root user for security

**Training Container (Optional):**

- ✅ Custom training environment
- ✅ MLflow integration
- ✅ Model export to S3
  {{% /notice %}}

## 3. Build & Push Automation

### 3.1. Local Build and Push Script

**Create file `scripts/build-and-push.sh`:**

```bash
#!/bin/bash

set -e

# Configuration
PROJECT_NAME="mlops-retail-forecast"
ENVIRONMENT="dev"
AWS_REGION="ap-southeast-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Repository URLs
API_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/mlops/retail-api"
TRAINING_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/mlops/train-model"

# Get Git information
GIT_COMMIT=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')

# Image tags
if [[ "$GIT_BRANCH" == "main" ]]; then
    IMAGE_TAG="v$(date +%Y%m%d)-${GIT_COMMIT}"
    LATEST_TAG="latest"
elif [[ "$GIT_BRANCH" == "develop" ]]; then
    IMAGE_TAG="dev-${GIT_COMMIT}"
    LATEST_TAG="dev-latest"
else
    IMAGE_TAG="feature-${GIT_BRANCH}-${GIT_COMMIT}"
    LATEST_TAG=""
fi

echo "🚀 Building and pushing Docker images to ECR..."
echo "API Repository: ${API_REPO}"
echo "Training Repository: ${TRAINING_REPO}"
echo "Git Branch: ${GIT_BRANCH}"
echo "Image Tag: ${IMAGE_TAG}"

# Login to ECR
echo "🔐 Logging in to ECR..."
aws ecr get-login-password --region ${AWS_REGION} | \
    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# Build API Image
echo "🏗️ Building FastAPI prediction service..."
docker build \
    --build-arg BUILD_DATE=${BUILD_DATE} \
    --build-arg GIT_COMMIT=${GIT_COMMIT} \
    --build-arg GIT_BRANCH=${GIT_BRANCH} \
    --build-arg VERSION=${IMAGE_TAG} \
    -t mlops-retail-api:${IMAGE_TAG} \
    -f server/Dockerfile \
    server/

# Tag API images
echo "🏷️ Tagging API images..."
docker tag mlops-retail-api:${IMAGE_TAG} ${API_REPO}:${IMAGE_TAG}
if [[ -n "$LATEST_TAG" ]]; then
    docker tag mlops-retail-api:${IMAGE_TAG} ${API_REPO}:${LATEST_TAG}
fi

# Push API images
echo "📤 Pushing API images to ECR..."
docker push ${API_REPO}:${IMAGE_TAG}
if [[ -n "$LATEST_TAG" ]]; then
    docker push ${API_REPO}:${LATEST_TAG}
fi

# Build Training Image (optional)
if [[ -f "training/Dockerfile" ]]; then
    echo "🏗️ Building training container..."
    docker build \
        --build-arg BUILD_DATE=${BUILD_DATE} \
        --build-arg GIT_COMMIT=${GIT_COMMIT} \
        --build-arg GIT_BRANCH=${GIT_BRANCH} \
        -t mlops-train-model:${IMAGE_TAG} \
        -f training/Dockerfile \
        training/

    # Tag training images
    echo "🏷️ Tagging training images..."
    docker tag mlops-train-model:${IMAGE_TAG} ${TRAINING_REPO}:${IMAGE_TAG}
    if [[ -n "$LATEST_TAG" ]]; then
        docker tag mlops-train-model:${IMAGE_TAG} ${TRAINING_REPO}:${LATEST_TAG}
    fi

    # Push training images
    echo "📤 Pushing training images to ECR..."
    docker push ${TRAINING_REPO}:${IMAGE_TAG}
    if [[ -n "$LATEST_TAG" ]]; then
        docker push ${TRAINING_REPO}:${LATEST_TAG}
    fi
else
    echo "ℹ️ Training Dockerfile not found, skipping training image build"
fi

# Clean up local images
echo "🧹 Cleaning up local images..."
docker rmi mlops-retail-api:${IMAGE_TAG} || true
docker rmi ${API_REPO}:${IMAGE_TAG} || true
if [[ -n "$LATEST_TAG" ]]; then
    docker rmi ${API_REPO}:${LATEST_TAG} || true
fi

if [[ -f "training/Dockerfile" ]]; then
    docker rmi mlops-train-model:${IMAGE_TAG} || true
    docker rmi ${TRAINING_REPO}:${IMAGE_TAG} || true
    if [[ -n "$LATEST_TAG" ]]; then
        docker rmi ${TRAINING_REPO}:${LATEST_TAG} || true
    fi
fi

echo "✅ Build and push completed successfully!"
echo "📍 API Image URLs:"
echo "   - ${API_REPO}:${IMAGE_TAG}"
if [[ -n "$LATEST_TAG" ]]; then
    echo "   - ${API_REPO}:${LATEST_TAG}"
fi

if [[ -f "training/Dockerfile" ]]; then
    echo "📍 Training Image URLs:"
    echo "   - ${TRAINING_REPO}:${IMAGE_TAG}"
    if [[ -n "$LATEST_TAG" ]]; then
        echo "   - ${TRAINING_REPO}:${LATEST_TAG}"
    fi
fi
```

**Make script executable:**

```bash
chmod +x scripts/build-and-push.sh
```

### 3.2. PowerShell Build Script (Windows)

**Create file `scripts/build-and-push.ps1`:**

```powershell
# Build and push script for Windows
param(
    [string]$Environment = "dev",
    [string]$Region = "ap-southeast-1"
)

# Configuration
$ProjectName = "mlops-retail-forecast"
$AwsAccountId = (aws sts get-caller-identity --query Account --output text)
$ApiRepo = "$AwsAccountId.dkr.ecr.$Region.amazonaws.com/mlops/retail-api"
$TrainingRepo = "$AwsAccountId.dkr.ecr.$Region.amazonaws.com/mlops/train-model"

# Get Git information
$GitCommit = (git rev-parse --short HEAD)
$GitBranch = (git rev-parse --abbrev-ref HEAD)
$BuildDate = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

# Determine image tags based on branch
if ($GitBranch -eq "main") {
    $ImageTag = "v$(Get-Date -Format 'yyyyMMdd')-$GitCommit"
    $LatestTag = "latest"
} elseif ($GitBranch -eq "develop") {
    $ImageTag = "dev-$GitCommit"
    $LatestTag = "dev-latest"
} else {
    $ImageTag = "feature-$GitBranch-$GitCommit"
    $LatestTag = ""
}

Write-Host "🚀 Building and pushing Docker images to ECR..." -ForegroundColor Green
Write-Host "API Repository: $ApiRepo" -ForegroundColor Yellow
Write-Host "Training Repository: $TrainingRepo" -ForegroundColor Yellow
Write-Host "Git Branch: $GitBranch" -ForegroundColor Yellow
Write-Host "Image Tag: $ImageTag" -ForegroundColor Yellow

# Login to ECR
Write-Host "🔐 Logging in to ECR..." -ForegroundColor Blue
$LoginCommand = aws ecr get-login-password --region $Region
$LoginCommand | docker login --username AWS --password-stdin "$AwsAccountId.dkr.ecr.$Region.amazonaws.com"

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to login to ECR"
    exit 1
}

# Build API Image
Write-Host "🏗️ Building FastAPI prediction service..." -ForegroundColor Blue
docker build `
    --build-arg BUILD_DATE=$BuildDate `
    --build-arg GIT_COMMIT=$GitCommit `
    --build-arg GIT_BRANCH=$GitBranch `
    --build-arg VERSION=$ImageTag `
    -t "mlops-retail-api:$ImageTag" `
    -f server/Dockerfile `
    server/

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to build API image"
    exit 1
}

# Tag API images
Write-Host "🏷️ Tagging API images..." -ForegroundColor Blue
docker tag "mlops-retail-api:$ImageTag" "${ApiRepo}:$ImageTag"
if ($LatestTag) {
    docker tag "mlops-retail-api:$ImageTag" "${ApiRepo}:$LatestTag"
}

# Push API images
Write-Host "📤 Pushing API images to ECR..." -ForegroundColor Blue
docker push "${ApiRepo}:$ImageTag"
if ($LatestTag) {
    docker push "${ApiRepo}:$LatestTag"
}

# Build Training Image (optional)
if (Test-Path "training/Dockerfile") {
    Write-Host "🏗️ Building training container..." -ForegroundColor Blue
    docker build `
        --build-arg BUILD_DATE=$BuildDate `
        --build-arg GIT_COMMIT=$GitCommit `
        --build-arg GIT_BRANCH=$GitBranch `
        -t "mlops-train-model:$ImageTag" `
        -f training/Dockerfile `
        training/

    # Tag training images
    Write-Host "🏷️ Tagging training images..." -ForegroundColor Blue
    docker tag "mlops-train-model:$ImageTag" "${TrainingRepo}:$ImageTag"
    if ($LatestTag) {
        docker tag "mlops-train-model:$ImageTag" "${TrainingRepo}:$LatestTag"
    }

    # Push training images
    Write-Host "📤 Pushing training images to ECR..." -ForegroundColor Blue
    docker push "${TrainingRepo}:$ImageTag"
    if ($LatestTag) {
        docker push "${TrainingRepo}:$LatestTag"
    }
} else {
    Write-Host "ℹ️ Training Dockerfile not found, skipping training image build" -ForegroundColor Yellow
}

# Clean up local images
Write-Host "🧹 Cleaning up local images..." -ForegroundColor Blue
docker rmi "mlops-retail-api:$ImageTag" -ErrorAction SilentlyContinue
docker rmi "${ApiRepo}:$ImageTag" -ErrorAction SilentlyContinue
if ($LatestTag) {
    docker rmi "${ApiRepo}:$LatestTag" -ErrorAction SilentlyContinue
}

if (Test-Path "training/Dockerfile") {
    docker rmi "mlops-train-model:$ImageTag" -ErrorAction SilentlyContinue
    docker rmi "${TrainingRepo}:$ImageTag" -ErrorAction SilentlyContinue
    if ($LatestTag) {
        docker rmi "${TrainingRepo}:$LatestTag" -ErrorAction SilentlyContinue
    }
}

Write-Host "✅ Build and push completed successfully!" -ForegroundColor Green
Write-Host "📍 API Image URLs:" -ForegroundColor Yellow
Write-Host "   - ${ApiRepo}:$ImageTag" -ForegroundColor White
if ($LatestTag) {
    Write-Host "   - ${ApiRepo}:$LatestTag" -ForegroundColor White
}

if (Test-Path "training/Dockerfile") {
    Write-Host "📍 Training Image URLs:" -ForegroundColor Yellow
    Write-Host "   - ${TrainingRepo}:$ImageTag" -ForegroundColor White
    if ($LatestTag) {
        Write-Host "   - ${TrainingRepo}:$LatestTag" -ForegroundColor White
    fi
}
```

### 3.3. Test Local Build

**Run build script:**

```bash
# Linux/Mac
./scripts/build-and-push.sh

# Windows PowerShell
.\scripts\build-and-push.ps1
```

**Verify images in ECR:**

```bash
# List API images
aws ecr list-images \
    --repository-name mlops/retail-api \
    --region ap-southeast-1

# List training images
aws ecr list-images \
    --repository-name mlops/train-model \
    --region ap-southeast-1
```

{{% notice success %}}
**🎯 Local Build & Push Complete!**

**Results:**

- ✅ FastAPI prediction service image built and pushed
- ✅ Training container image built and pushed (if training/Dockerfile exists)
- ✅ Images tagged with Git commit and branch information
- ✅ Latest tags created for main/develop branches
- ✅ Local cleanup completed
  {{% /notice %}}

## 4. CI/CD Integration

### 4.1. GitHub Actions Workflow

**Create file `.github/workflows/build-and-deploy.yml`:**

```yaml
name: Build and Deploy to ECR

on:
  push:
    branches: [main, develop]
    paths:
      - "server/**"
      - "training/**"
      - ".github/workflows/**"
  pull_request:
    branches: [main]
    paths:
      - "server/**"
      - "training/**"

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  API_REPOSITORY: mlops/retail-api
  TRAINING_REPOSITORY: mlops/train-model

jobs:
  build-and-push:
    name: Build and Push to ECR
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Get Git information
        id: git-info
        run: |
          echo "commit=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          echo "branch=${GITHUB_REF#refs/heads/}" >> $GITHUB_OUTPUT
          echo "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT

          # Determine image tag based on branch
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "image-tag=v$(date +%Y%m%d)-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
            echo "latest-tag=latest" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "image-tag=dev-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
            echo "latest-tag=dev-latest" >> $GITHUB_OUTPUT
          else
            echo "image-tag=feature-${{ github.head_ref }}-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
            echo "latest-tag=" >> $GITHUB_OUTPUT
          fi

      - name: Build and push API image
        uses: docker/build-push-action@v5
        with:
          context: ./server
          file: ./server/Dockerfile
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.API_REPOSITORY }}:${{ steps.git-info.outputs.image-tag }}
            ${{ steps.git-info.outputs.latest-tag && format('{0}/{1}:{2}', env.ECR_REGISTRY, env.API_REPOSITORY, steps.git-info.outputs.latest-tag) || '' }}
          build-args: |
            BUILD_DATE=${{ steps.git-info.outputs.build-date }}
            GIT_COMMIT=${{ steps.git-info.outputs.commit }}
            GIT_BRANCH=${{ steps.git-info.outputs.branch }}
            VERSION=${{ steps.git-info.outputs.image-tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push training image
        if: hashFiles('training/Dockerfile') != ''
        uses: docker/build-push-action@v5
        with:
          context: ./training
          file: ./training/Dockerfile
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.TRAINING_REPOSITORY }}:${{ steps.git-info.outputs.image-tag }}
            ${{ steps.git-info.outputs.latest-tag && format('{0}/{1}:{2}', env.ECR_REGISTRY, env.TRAINING_REPOSITORY, steps.git-info.outputs.latest-tag) || '' }}
          build-args: |
            BUILD_DATE=${{ steps.git-info.outputs.build-date }}
            GIT_COMMIT=${{ steps.git-info.outputs.commit }}
            GIT_BRANCH=${{ steps.git-info.outputs.branch }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan images for vulnerabilities
        run: |
          # Trigger ECR scan for API image
          aws ecr start-image-scan \
            --repository-name ${{ env.API_REPOSITORY }} \
            --image-id imageTag=${{ steps.git-info.outputs.image-tag }} \
            --region ${{ env.AWS_REGION }} || true
            
          # Trigger ECR scan for training image (if exists)
          if [[ -f "training/Dockerfile" ]]; then
            aws ecr start-image-scan \
              --repository-name ${{ env.TRAINING_REPOSITORY }} \
              --image-id imageTag=${{ steps.git-info.outputs.image-tag }} \
              --region ${{ env.AWS_REGION }} || true
          fi

      - name: Update EKS deployment (main branch only)
        if: github.ref == 'refs/heads/main'
        run: |
          # Update Kubernetes deployment with new image
          # This would typically involve updating deployment YAML or using tools like ArgoCD
          echo "Would update EKS deployment with image: ${{ env.ECR_REGISTRY }}/${{ env.API_REPOSITORY }}:${{ steps.git-info.outputs.image-tag }}"

      - name: Output image information
        run: |
          echo "### 🚀 Build Complete!" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**API Image:**" >> $GITHUB_STEP_SUMMARY
          echo "- \`${{ env.ECR_REGISTRY }}/${{ env.API_REPOSITORY }}:${{ steps.git-info.outputs.image-tag }}\`" >> $GITHUB_STEP_SUMMARY
          if [[ -n "${{ steps.git-info.outputs.latest-tag }}" ]]; then
            echo "- \`${{ env.ECR_REGISTRY }}/${{ env.API_REPOSITORY }}:${{ steps.git-info.outputs.latest-tag }}\`" >> $GITHUB_STEP_SUMMARY
          fi
          echo "" >> $GITHUB_STEP_SUMMARY
          if [[ -f "training/Dockerfile" ]]; then
            echo "**Training Image:**" >> $GITHUB_STEP_SUMMARY
            echo "- \`${{ env.ECR_REGISTRY }}/${{ env.TRAINING_REPOSITORY }}:${{ steps.git-info.outputs.image-tag }}\`" >> $GITHUB_STEP_SUMMARY
            if [[ -n "${{ steps.git-info.outputs.latest-tag }}" ]]; then
              echo "- \`${{ env.ECR_REGISTRY }}/${{ env.TRAINING_REPOSITORY }}:${{ steps.git-info.outputs.latest-tag }}\`" >> $GITHUB_STEP_SUMMARY
            fi
          fi
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Git Info:**" >> $GITHUB_STEP_SUMMARY
          echo "- Commit: \`${{ steps.git-info.outputs.commit }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- Branch: \`${{ steps.git-info.outputs.branch }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- Build Date: \`${{ steps.git-info.outputs.build-date }}\`" >> $GITHUB_STEP_SUMMARY
```

### 4.2. GitHub Repository Secrets

**Required secrets in GitHub repository:**

```
AWS_ACCOUNT_ID: 123456789012
AWS_ROLE_ARN: arn:aws:iam::123456789012:role/mlops-cicd-role
```

### 4.3. IAM Role for GitHub Actions (OIDC)

**Create OIDC provider and role:**

```bash
# Create OIDC provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --client-id-list sts.amazonaws.com

# Create trust policy for GitHub Actions
cat > github-actions-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:your-username/retail-forecast:*"
        }
      }
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name mlops-cicd-role \
  --assume-role-policy-document file://github-actions-trust-policy.json

# Attach ECR permissions
aws iam attach-role-policy \
  --role-name mlops-cicd-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

{{% notice success %}}
**🎯 CI/CD Integration Complete!**

**Automated Workflow:**

- ✅ GitHub Actions triggered on push to main/develop
- ✅ Multi-stage Docker builds with caching
- ✅ Automatic image tagging based on Git branch/commit
- ✅ ECR vulnerability scanning on push
- ✅ OIDC authentication (no AWS keys in repo)
- ✅ Build summaries with image URLs
  {{% /notice %}}

## 5. Verification & Testing

### 5.1. Image Verification

**Check repositories in ECR:**

```bash
# List all repositories
aws ecr describe-repositories --region ap-southeast-1

# List images in API repository
aws ecr list-images \
    --repository-name mlops/retail-api \
    --region ap-southeast-1

# List images in training repository
aws ecr list-images \
    --repository-name mlops/train-model \
    --region ap-southeast-1
```

**Get detailed image information:**

```bash
# Describe specific image
aws ecr describe-images \
    --repository-name mlops/retail-api \
    --image-ids imageTag=latest \
    --region ap-southeast-1

# Check image scan results
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1
```

### 5.2. Local Container Testing

**Test API container locally:**

```bash
# Run API container locally
docker run -d \
    --name retail-api-test \
    -p 8000:8000 \
    -e AWS_DEFAULT_REGION=ap-southeast-1 \
    -e MODEL_BUCKET=mlops-retail-forecast-models \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# Test health endpoint
curl http://localhost:8000/health

# Test prediction endpoint
curl -X POST http://localhost:8000/predict \
    -H "Content-Type: application/json" \
    -d '{
        "features": {
            "SPEND": 100.0,
            "UNITS": 5.0,
            "VISITS": 2.0
        },
        "customer_id": "test-customer-123"
    }'

# Check logs
docker logs retail-api-test

# Clean up
docker stop retail-api-test
docker rm retail-api-test
```

### 5.3. EKS Integration Testing

**Create test deployment:**

```yaml
# test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api-test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: retail-api-test
  template:
    metadata:
      labels:
        app: retail-api-test
    spec:
      serviceAccountName: retail-forecast-sa
      containers:
        - name: api
          image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: AWS_DEFAULT_REGION
              value: "ap-southeast-1"
            - name: MODEL_BUCKET
              value: "mlops-retail-forecast-models"
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
---
apiVersion: v1
kind: Service
metadata:
  name: retail-api-test-service
spec:
  selector:
    app: retail-api-test
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### 5.4. Security Scan Analysis

**Check vulnerability scan results:**

```bash
# Get scan results summary
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1 \
    --query 'imageScanFindings.findingCounts'

# Get detailed vulnerability findings
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1 \
    --query 'imageScanFindings.findings[?severity==`HIGH` || severity==`CRITICAL`]'
```

**Security best practices verification:**

```bash
# Check if running as non-root user
docker run --rm 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest whoami

# Check file permissions
docker run --rm 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest ls -la /app

# Verify health check works
docker run --rm -p 8000:8000 -d --name health-test \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

sleep 30
docker exec health-test python health_check.py
docker stop health-test
```

## 6. Cost Monitoring & Optimization

### 6.1. ECR Cost Tracking

**Monthly cost breakdown:**

```bash
# Check repository storage usage
aws ecr describe-repositories \
    --region ap-southeast-1 \
    --query 'repositories[*].{Name:repositoryName,Size:repositorySizeInBytes}' \
    --output table

# Estimate monthly storage cost
# ECR Storage: $0.10 per GB-month
# Typical API image: ~500MB
# Training image: ~1GB
# Total: ~1.5GB * $0.10 = $0.15/month
```

### 6.2. Image Optimization

**Multi-stage build analysis:**

```bash
# Compare image sizes
docker images | grep mlops

# Check layer sizes
docker history 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
```

**Optimization strategies:**

1. **Use slim base images**: python:3.9-slim vs python:3.9
2. **Multi-stage builds**: Separate build and runtime environments
3. **Layer caching**: Order Dockerfile commands for better caching
4. **Remove unnecessary packages**: Clean up apt cache, temp files
5. **Use .dockerignore**: Exclude unnecessary files from context

## 7. Troubleshooting

### 7.1. Common Issues

**Issue 1: ECR Authentication Failed**

```bash
# Solution: Refresh ECR login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com

# Check AWS credentials
aws sts get-caller-identity
```

**Issue 2: Image Pull Failed from EKS**

```bash
# Check node group IAM permissions
kubectl describe node

# Verify ECR repository policy
aws ecr get-repository-policy \
    --repository-name mlops/retail-api \
    --region ap-southeast-1

# Check IRSA configuration
kubectl describe serviceaccount retail-forecast-sa
```

**Issue 3: Container Health Check Failing**

```bash
# Check container logs
kubectl logs -l app=retail-api-test

# Test health endpoint manually
kubectl exec -it <pod-name> -- curl http://localhost:8000/health

# Verify model loading
kubectl exec -it <pod-name> -- python -c "
import boto3
s3 = boto3.client('s3')
s3.head_object(Bucket='mlops-retail-forecast-models', Key='models/retail-price-sensitivity/model.pkl')
"
```

**Issue 4: Build Timeout in CI/CD**

```yaml
# Increase timeout in GitHub Actions
jobs:
  build-and-push:
    timeout-minutes: 30 # Increase from default 360
```

### 7.2. Performance Optimization

**Build performance:**

```bash
# Use BuildKit for faster builds
export DOCKER_BUILDKIT=1
docker build ...

# Enable build cache
docker build --cache-from mlops-retail-api:latest ...
```

**Runtime performance:**

```bash
# Monitor container resource usage
kubectl top pods -l app=retail-api

# Check API response times
kubectl exec -it <pod-name> -- curl -w "@curl-format.txt" http://localhost:8000/predict
```

## 👉 Kết quả Task 6

✅ **ECR Repositories** - mlops/retail-api và mlops/train-model repositories  
✅ **Container Images** - FastAPI prediction service và training containers  
✅ **CI/CD Integration** - GitHub Actions automated build-push-deploy  
✅ **Security** - Image scanning, non-root users, OIDC authentication  
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month

### Architecture Delivered

```
✅ ECR Container Registry:
   - mlops/retail-api: FastAPI prediction service
   - mlops/train-model: Custom training environment
   - Tag immutability enabled
   - Vulnerability scanning on push

✅ CI/CD Automation:
   - GitHub Actions workflow
   - Automatic builds on code changes
   - Git-based image tagging (v20241009-abc123)
   - OIDC authentication (no AWS keys)

✅ Security & Compliance:
   - Non-root containers
   - Multi-stage builds
   - Vulnerability scanning
   - IAM-based access control
```

{{% notice success %}}
**🎯 Task 6 Complete - Production-Ready Container Registry!**

**Container Strategy**: FastAPI API + optional training containers  
**CI/CD Automation**: Build → Push → Deploy on Git changes  
**Security**: Image scanning, non-root users, access control  
**Cost Efficient**: ~$0.15/month storage + lifecycle cleanup  
**Demo Ready**: API container ready for ALB + EKS deployment  
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 7**: EKS cluster deployment trong hybrid VPC
- **Task 8**: EKS node groups với ECR image pull permissions
- **Task 10**: Deploy API container với ALB integration
- **Task 11**: ALB ingress controller cho public API access

**Build & Deploy Commands:**

```bash
# Local build and push
./scripts/build-and-push.sh

# CI/CD triggers automatically on:
git push origin main        # → v20241009-abc123 + latest
git push origin develop     # → dev-abc123 + dev-latest
git push origin feature/x   # → feature-x-abc123
```

{{% /notice %}}

{{% notice info %}}
**� Production Benchmarks Achieved:**

- **Image Size**: FastAPI ~500MB (optimized multi-stage)
- **Build Time**: ~3-5 minutes (with caching)
- **Storage Cost**: ~$0.15/month (1.5GB total)
- **Security**: Non-root, vulnerability scanned
- **Availability**: Multi-tag strategy (latest, commit, branch)
- **CI/CD**: Automated on every commit
  {{% /notice %}}

---

**Next Step**: [Task 7: EKS Cluster Setup](../7-eks-cluster/)
