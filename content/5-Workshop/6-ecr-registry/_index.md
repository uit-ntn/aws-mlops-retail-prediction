---
title: "Amazon ECR Container Registry (MLOps)"
date: 2024-01-01T00:00:00Z
weight: 7
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}

**🎯 Task 6 Objectives:** Set up Amazon Elastic Container Registry (ECR) for MLOps pipeline:
1. **Create ECR Repository**: Repository for API container
2. **Security Configuration**: Image scanning, IAM policy, lifecycle rules  
3. **Build & Push Image**: Upload FastAPI container to ECR
4. **Manual Build & Push**: Guide for build/push using script (CLI / PowerShell)
{{% /notice %}}

📥 **Input from Previous Tasks:**
- **Task 2 (IAM Roles & Audit):** IAM roles, policies and permissions for ECR/EKS/S3 access
- **Task 5 (Production VPC):** VPC endpoints, networking and security groups to allow EKS pull images from ECR

📦 **Output:**
- **Inference Container**: `server/` code → FastAPI API serving predictions in EKS

## Overview

**Amazon ECR (Elastic Container Registry)** is a fully managed Docker container registry service by AWS, deeply integrated with EKS and CI/CD pipeline. ECR provides secure storage, management, and deployment capabilities for container images in MLOps workflow.

**🎯 Task 6 Goal:** Create a private Amazon ECR repository for the Retail API image, enforce image hygiene (immutability, scan-on-push, lifecycle), and publish a production-ready FastAPI container image to ECR.
{{% /notice %}}

## 0) Inputs from previous tasks

- Production AWS Region: **ap-southeast-1**
- AWS Account ID: **842676018087**
- Source folder: `server/` (FastAPI inference API)
- Target ECR repository: `mlops/retail-api`

---

## 1) Create the ECR repository (recommended settings)


### 1.1 Create repo (CLI)

```bash
export AWS_REGION="ap-southeast-1"
export REPO_NAME="mlops/retail-api"


1. **Navigate to ECR Console:**
   - Login to AWS Console
   - Navigate to Amazon ECR service
   - Region: ap-southeast-1
   - Select "Create repository"

aws ecr create-repository   --region "$AWS_REGION"   --repository-name "$REPO_NAME"   --image-tag-mutability IMMUTABLE   --image-scanning-configuration scanOnPush=true
```


> If the repo already exists, the command will fail. In that case, just update settings in the console.

### 1.2 Enable scan-on-push and immutability (Console)

ECR → Repositories → `mlops/retail-api` → **Edit**:


3. **Repository Created Successfully:**
   
   After creating the repository, you will see the interface as shown below with information:
   
   - Repository name: `mlops/retail-api`
   - Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
   - Status: "No active images" (no images have been pushed yet)
   - Tabs: Summary, Images, Permissions, Lifecycle policy, Repository tags

- **Image tag mutability:** Immutable
- **Scan on push:** Enabled


---


4. **Repository Setup Complete:**
   
   API repository is ready for containerized FastAPI application.

5. **Repository Management Interface:**
   
   In the repository management interface, you can:
   - **Images tab**: View list of images, filter by tags
   - **View push commands**: Commands to push image to repository  
   - **Copy URI**: Copy repository URI for use
   - **Scan**: Scan vulnerabilities for images
   - **Delete**: Delete repository when not needed

## 2) Add lifecycle policy (keep prod/dev tags, expire untagged)

Create `ecr-lifecycle.json`:


```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 images for prod/dev tag prefixes",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod", "dev", "staging", "latest"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}
```


{{% notice tip %}}
**Tip:** Enable `tag immutability` for production tags (e.g., `v*`) to avoid accidental overwrite. Use semantic tags (`v1.2.3`, `commit-<sha>`) to help with rollback and audit.
{{% /notice %}}

Apply it:


```bash
aws ecr put-lifecycle-policy   --region "$AWS_REGION"   --repository-name "$REPO_NAME"   --lifecycle-policy-text file://ecr-lifecycle.json
```


1. **API Repository Lifecycle Policy:**
   - Select repository `mlops/retail-api`
   - Click tab "Lifecycle policy" 
   - Click "Create rule" to create lifecycle policy

---


## 3) Build a production-ready FastAPI container image

### 3.1 Multi-stage Dockerfile (non-root + healthcheck)


   **Rule 1 - Keep Latest Production Images:**

   ```
   Rule priority: 1
   Description: Keep latest 10 production images
   Image status: Tagged (wildcard matching)
   Image tag filters: v*

   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 10

   Action: expire
   ```

   **Rule 2 - Keep Latest Development Images:**

   ```
   Rule priority: 2
   Description: Keep latest 5 development images
   Image status: Tagged (wildcard matching)
   Image tag filters: dev*, feature*, main*

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
   - Days since image created: 1

   Action: expire
   ```


3. **Training Repository Lifecycle Policy:**

![](/images/06-ecr-registry/08.png)

### 1.3. Image Scanning & Push Commands

1. **Check Scan Settings:**
   - Select repository from list
   - Check "Scan on push" is enabled
   - Review enhanced scanning options if needed

2. **View Push Commands:**
   - Click "View push commands" button in repository interface
   - AWS will display commands to authenticate and push image
   - Copy these commands for use from local machine or CI/CD pipeline

![](/images/06-ecr-registry/09.1.png)

![](/images/06-ecr-registry/09.2.png)

{{% notice success %}}
**🎯 ECR Repositories Setup Complete!**

**Created Repository:**

- ✅ `mlops/retail-api`: FastAPI prediction service container
- ✅ Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
- ✅ Private repository with tag immutability enabled
- ✅ Image scanning enabled on push
- ✅ Lifecycle policies configured for cost optimization
- ✅ Push commands available in console
- ✅ IAM access policies for EKS integration
{{% /notice %}}

{{% notice tip %}}
**Tip:** Document lifecycle rule priorities in team docs and test rules on non-prod repos before applying to production to avoid accidentally deleting images.
{{% /notice %}}

## 2. API Containerization Workflow

### 2.1. Dockerfile Configuration

**Create `server/Dockerfile` - Multi-stage build:**

Create `server/Dockerfile`:


```dockerfile
# ---- builder ----
FROM python:3.11-slim AS builder
WORKDIR /app

ENV PIP_DISABLE_PIP_VERSION_CHECK=1     PIP_NO_CACHE_DIR=1     PYTHONDONTWRITEBYTECODE=1     PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends     build-essential  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN python -m venv /opt/venv  && /opt/venv/bin/pip install --upgrade pip  && /opt/venv/bin/pip install -r requirements.txt

# ---- runtime ----
FROM python:3.11-slim AS runtime
WORKDIR /app

ENV PATH="/opt/venv/bin:$PATH"     PYTHONDONTWRITEBYTECODE=1     PYTHONUNBUFFERED=1

# Create non-root user
RUN addgroup --system app && adduser --system --ingroup app app

COPY --from=builder /opt/venv /opt/venv
COPY . .

# Healthcheck endpoint should exist in your FastAPI app
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3   CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()" || exit 1

EXPOSE 8000
USER app

# Uvicorn entrypoint
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```


**Create `server/.dockerignore`:**

Create `server/.dockerignore`:


```gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
*.log
.env
.venv/
venv/
tests/
dist/
build/
.git/
.github/
*.ipynb
```

---

## 4) Authenticate and push to ECR

### 4.1 Bash (Linux/macOS) script

Create `scripts/ecr_push.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

AWS_REGION="ap-southeast-1"
AWS_ACCOUNT_ID="842676018087"
REPO_NAME="mlops/retail-api"
IMAGE_TAG="${1:-latest}"

ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"

echo "[1/4] ECR login..."
aws ecr get-login-password --region "$AWS_REGION"   | docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

echo "[2/4] Build image..."
docker build -t "${REPO_NAME}:${IMAGE_TAG}" -f server/Dockerfile server/

echo "[3/4] Tag image..."
docker tag "${REPO_NAME}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"

echo "[4/4] Push image..."
docker push "${ECR_URI}:${IMAGE_TAG}"

echo "✅ Pushed: ${ECR_URI}:${IMAGE_TAG}"
```


{{% notice warning %}}
**Warning:** Docker login tokens (ECR auth) have expiration; CI agents should refresh token (`aws ecr get-login-password`) per job. Avoid hardcoding credentials in scripts or environment files.
{{% /notice %}}


### 2.3. View Push Commands from AWS Console

1. **In ECR Console:**
   - Select repository `mlops/retail-api`
   - Click **"View push commands"** button
   - AWS will display commands to build and push

2. **Push commands will be like (Windows PowerShell):**

### 4.2 PowerShell (Windows) script

Create `scripts/ecr_push.ps1`:


```powershell
param(
  [string]$ImageTag = "latest"
)

$AWS_REGION = "ap-southeast-1"
$AWS_ACCOUNT_ID = "842676018087"
$REPO_NAME = "mlops/retail-api"
$ECR_URI = "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME"

Write-Host "[1/4] ECR login..."
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

Write-Host "[2/4] Build image..."
docker build -t "$REPO_NAME:$ImageTag" -f server/Dockerfile server/

Write-Host "[3/4] Tag image..."
docker tag "$REPO_NAME:$ImageTag" "$ECR_URI:$ImageTag"

Write-Host "[4/4] Push image..."
docker push "$ECR_URI:$ImageTag"

Write-Host "✅ Pushed: $ECR_URI:$ImageTag"
```


   **Or use AWS CLI:**

---

## 5) Verify in console / CLI


```bash
aws ecr describe-repositories --region ap-southeast-1 --query 'repositories[?repositoryName==`mlops/retail-api`].repositoryUri' --output text

aws ecr describe-images   --region ap-southeast-1   --repository-name mlops/retail-api   --max-items 10
```


{{% notice info %}}
**Info:** On Windows/PowerShell, prefer using `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <registry>` in CI to avoid deprecated commands. ECR tokens typically expire after ~12 hours; re-authenticate for long-running sessions.
{{% /notice %}}

---


## 6) Cleanup scripts (optional)


### 2.2. Verify ECR Push Success

**Check in AWS Console:**

1. **Navigate to ECR Console:**
   - Go to AWS Console → ECR service
   - Select repository `mlops/retail-api`
   - Check "Images" tab to see if image has been pushed

2. **Expected Result:**
   - Image with tag `latest` appears in the list
   - Image size displayed (~927MB)
   - Vulnerability scan status (if enabled)
   - Push timestamp

![](/images/06-ecr-registry/10.png)

**Check using CLI:**

![](/images/06-ecr-registry/12.png)

![](/images/06-ecr-registry/13.png)

**Check using console:**

![](/images/06-ecr-registry/14.png)

{{% notice tip %}}
**Tip:** Reduce image size using multi-stage builds and lightweight base images (e.g., `python:3.9-slim` or distroless). Smaller images help with faster push/pull and reduce storage/transfer costs.
{{% /notice %}}

### 2.5. Container Environment & Testing

**Environment Variables:**

### 6.1 Delete local build caches (safe)


```bash
docker image prune -f
docker builder prune -f
```

### 6.2 Delete untagged images in ECR (manual)

```bash
aws ecr list-images   --region ap-southeast-1   --repository-name mlops/retail-api   --filter tagStatus=UNTAGGED   --query 'imageIds[*]'   --output json > untagged.json

aws ecr batch-delete-image   --region ap-southeast-1   --repository-name mlops/retail-api   --image-ids file://untagged.json
```


{{% notice warning %}}
**Warning (Local):** When running image on local machine, avoid mounting secrets or AWS credentials into container. Use environment variables only for non-sensitive values and prefer IAM roles for production environment.
{{% /notice %}}

---


## 7) Rough cost notes (ap-southeast-1)


![](/images/06-ecr-registry/11.png)

**Complete!** 🎉

ECR registry has been set up and integrated with EKS cluster `mlops-retail-cluster`. Docker image for retail API is ready to deploy on Kubernetes in Task 10.

## Task 6 Results

✅ **ECR Repository** - mlops/retail-api repository  
✅ **Container Image** - FastAPI prediction service  
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month  

- **ECR storage** is billed per GB-month; lifecycle policies keep it low.
- **Image scanning** is enabled; costs depend on usage (keep scans on push, but avoid pushing too frequently).
- **Data transfer to EKS** is typically intra-region; cross-region pulls cost more (avoid by keeping everything in ap-southeast-1).


{{% notice success %}}
**✅ Task 6 Complete (ECR):**


**✅ ECR Setup**: Repository with lifecycle policies & image scanning  
**✅ Dockerfile**: Multi-stage build, non-root user, health checks  
**✅ Build & Push**: Local build → ECR push workflow  
**✅ Testing**: Container verification & API validation  
**✅ Ready**: Ready for EKS deployment in Task 7  
{{% /notice %}}

{{% notice info %}}
**Info (Vulnerability Scanning):** Basic image scanning is free; enhanced scanning (Inspector) may incur charges per image/month. Consider scanning only production tags or integrating scanning into CI with conditions to control costs.
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 7**: EKS cluster deployment with ECR integration
- **Task 8**: Deploy API container to EKS with ALB
- **Task 9**: Load balancing and scaling configuration

{{% /notice %}}

{{% notice info %}}
**Benchmark Results (Production):**

- **Image size**: FastAPI ~500MB (multi-stage optimized)
- **Build time**: ~3-5 minutes (with cache)
- **Storage cost**: ~$0.15/month (total ~1.5GB)
- **Security**: Running non-root, vulnerability scanned
- **Availability**: Multi-tag strategy (latest, commit, branch)
- **CI/CD**: Automated on every commit
{{% /notice %}}

## 3. Clean Up Resources (AWS CLI)

### 3.1. Delete Images from ECR Repository

```bash
# List images in repository
aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1 --query 'imageDetails[*].[imageDigest,imageTags[0],imagePushedAt]' --output table

# Delete specific image tag
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids imageTag=latest \
  --region ap-southeast-1

# Delete all images in repository
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids "$(aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1 --query 'imageDetails[*].{imageDigest:imageDigest}' --output json)" \
  --region ap-southeast-1
```

### 3.2. Delete ECR Repositories

```bash
# Delete repository (must be empty first)
aws ecr delete-repository --repository-name mlops/retail-api --region ap-southeast-1 --force

# Verify repository has been deleted
aws ecr describe-repositories --region ap-southeast-1 --query 'repositories[?repositoryName==`mlops/retail-api`]'
```

### 3.3. Delete Lifecycle Policies

```bash
# Delete lifecycle policy (automatically deleted when repository is deleted)
aws ecr delete-lifecycle-policy --repository-name mlops/retail-api --region ap-southeast-1

# List remaining repositories
aws ecr describe-repositories --region ap-southeast-1 --query 'repositories[*].[repositoryName,repositoryUri]' --output table
```

### 3.4. Clean Up Local Docker Images

```bash
# Remove local Docker images
docker rmi mlops/retail-api:latest
docker rmi 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# Clean up Docker build cache
docker system prune -f

# Remove unused images
docker image prune -a -f
```

### 3.5. ECR Cleanup Helper Script

```bash
#!/bin/bash
# ecr-cleanup.sh

REPOSITORY_NAME="mlops/retail-api"
REGION="ap-southeast-1"

echo "🧹 Cleaning up ECR repository: $REPOSITORY_NAME..."

# 1. Delete all images
echo "Deleting all images..."
IMAGE_IDS=$(aws ecr describe-images --repository-name $REPOSITORY_NAME --region $REGION --query 'imageDetails[*].{imageDigest:imageDigest}' --output json)

if [ "$IMAGE_IDS" != "[]" ]; then
    aws ecr batch-delete-image \
        --repository-name $REPOSITORY_NAME \
        --image-ids "$IMAGE_IDS" \
        --region $REGION
    echo "Images deleted"
else
    echo "No images to delete"
fi

# 2. Delete repository
echo "Deleting repository..."
aws ecr delete-repository \
    --repository-name $REPOSITORY_NAME \
    --region $REGION \
    --force

# 3. Clean up local Docker
echo "Cleaning up local Docker images..."
docker rmi mlops/retail-api:latest 2>/dev/null || true
docker rmi 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/$REPOSITORY_NAME:latest 2>/dev/null || true

echo "✅ ECR cleanup completed"
```

---

## 4. Bảng giá ECR

### 4.1. Chi phí ECR Storage

| Storage Type | Giá (USD/GB/tháng) | Ghi chú |
|--------------|-------------------|---------|
| **ECR Storage** | $0.10 | Compressed image size |
| **Free Tier** | 500MB free | First 12 months |
| **Data Transfer IN** | Free | Push images to ECR |
| **Data Transfer OUT** | $0.12/GB | Pull from Internet |
| **Data Transfer VPC** | Free | Pull via VPC Endpoints |

### 4.2. Image Scanning Cost

| Scan Type | Price (USD) | Notes |
|-----------|-----------|---------|
| **Basic Scanning** | Free | CVE database scanning |
| **Enhanced Scanning** | $0.09/image/month | Inspector integration |
| **OS Package Scanning** | Free | Basic vulnerability detection |
| **Language Package Scanning** | $0.09/image/month | Enhanced scanning only |

### 4.3. Estimated Cost for Task 6

**Container Images:**
- FastAPI image: ~500MB (compressed)
- Total storage: ~0.5GB

**Monthly Costs:**

| Component | Size | Price | Monthly Cost |
|-----------|------|-------|--------------|
| **ECR Storage** | 0.5GB | $0.10/GB | $0.05 |
| **Basic Scanning** | 1 image | Free | $0.00 |
| **VPC Endpoint Transfer** | ~1GB/month | Free | $0.00 |
| **Total** | | | **$0.05** |

### 4.4. Cost Comparison with Alternatives

**ECR vs Docker Hub:**

| Feature | ECR | Docker Hub | Winner |
|---------|-----|------------|--------|
| **Storage (500MB)** | $0.05/month | Free (public) | Docker Hub |
| **Private repos** | ✅ Native | $5/month | **ECR** |
| **AWS Integration** | ✅ Native | Manual setup | **ECR** |
| **VPC Endpoints** | ✅ Free transfer | ❌ Internet only | **ECR** |
| **IAM Integration** | ✅ Native | ❌ Token-based | **ECR** |
| **Vulnerability Scanning** | ✅ Built-in | ❌ Extra cost | **ECR** |

### 4.5. Data Transfer Costs

**ECR Pull Scenarios:**

| Pull Location | Cost | Use Case |
|---------------|------|----------|
| **Same Region (VPC)** | Free | EKS production |
| **Same Region (Internet)** | $0.12/GB | CI/CD outside AWS |
| **Cross Region** | $0.12/GB + transfer | Multi-region deployment |
| **Internet (outside AWS)** | $0.12/GB | Local development |

### 4.6. Lifecycle Policy Cost Savings

**Without Lifecycle Policies:**
- 50 images × 500MB = 25GB storage
- Cost: 25GB × $0.10 = $2.50/month

**With Lifecycle Policies (Task 6):**
- Keep 10 production images = 5GB
- Keep 5 development images = 2.5GB  
- Total: 7.5GB × $0.10 = $0.75/month
- **Savings: $1.75/month (70%)**

### 4.7. Cost Optimization Tips

**Storage Optimization:**
```bash
# Multi-stage builds reduce image size
FROM node:16 as builder
# ... build steps
FROM node:16-alpine as production  # Smaller base image
COPY --from=builder /app/dist ./dist
```

**Registry Management:**
```bash
# Automated cleanup with lifecycle policies
aws ecr put-lifecycle-policy \
  --repository-name mlops/retail-api \
  --lifecycle-policy-text file://lifecycle-policy.json
```

**Free Tier Usage:**
- Use 500MB free tier for development
- Production images in separate repositories
- VPC Endpoints to avoid data transfer charges

{{% notice info %}}
**💰 Cost Summary for Task 6:**
- **Storage:** $0.05/month (500MB images)
- **Scanning:** Free (basic vulnerability detection)
- **Data Transfer:** Free (VPC Endpoints to EKS)
- **Total:** **$0.05/month** (vs $5/month Docker Hub private)
- **Savings:** $4.95/month with ECR + lifecycle policies
{{% /notice %}}

---

{{% notice tip %}}
**Success Tip:** Before deleting repos/images for cleanup, snapshot deployment manifests and CI references if archival is needed. Prefer using lifecycle policies for automated retention management instead of manual deletion to avoid data loss.
{{% /notice %}}

## 🎬 Task 6 Implementation Video

<div style="position: relative; width: 100%; max-width: 800px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=b5OPwgR-rqI&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=5" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 7: EKS Cluster Setup](../7-eks-cluster/) 

- Private repo `mlops/retail-api` created in `ap-southeast-1`
- Scan-on-push + tag immutability enabled
- Lifecycle policy applied
- FastAPI image built (multi-stage, non-root, healthcheck) and pushed to ECR
  {{% /notice %}}

