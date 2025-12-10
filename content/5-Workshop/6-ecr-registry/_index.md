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

## 1. ECR Repositories Setup

### 1.1. Create ECR Repositories

1. **Navigate to ECR Console:**
   - Login to AWS Console
   - Navigate to Amazon ECR service
   - Region: ap-southeast-1
   - Select "Create repository"

![](/images/06-ecr-registry/01.png)

2. **API Repository Configuration:**

![](/images/06-ecr-registry/02.png)

3. **Repository Created Successfully:**
   
   After creating the repository, you will see the interface as shown below with information:
   
   - Repository name: `mlops/retail-api`
   - Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
   - Status: "No active images" (no images have been pushed yet)
   - Tabs: Summary, Images, Permissions, Lifecycle policy, Repository tags

![](/images/06-ecr-registry/03.1.png)

4. **Repository Setup Complete:**
   
   API repository is ready for containerized FastAPI application.

5. **Repository Management Interface:**
   
   In the repository management interface, you can:
   - **Images tab**: View list of images, filter by tags
   - **View push commands**: Commands to push image to repository  
   - **Copy URI**: Copy repository URI for use
   - **Scan**: Scan vulnerabilities for images
   - **Delete**: Delete repository when not needed

![](/images/06-ecr-registry/04.png)

{{% notice tip %}}
**Tip:** Enable `tag immutability` for production tags (e.g., `v*`) to avoid accidental overwrite. Use semantic tags (`v1.2.3`, `commit-<sha>`) to help with rollback and audit.
{{% /notice %}}

### 1.2. Lifecycle Policy Setup

1. **API Repository Lifecycle Policy:**
   - Select repository `mlops/retail-api`
   - Click tab "Lifecycle policy" 
   - Click "Create rule" to create lifecycle policy

![](/images/06-ecr-registry/07.png)

2. **Configure API Lifecycle Rules:**

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
# Navigate to server directory
cd retail-price-sensitivity-prediction/server

# Build Docker image
docker build -t mlops/retail-api:latest .

# Test locally
docker run -d --name test -p 8000:8000 mlops/retail-api:latest
curl http://localhost:8000/health
docker stop test && docker rm test
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

```powershell
# 1. Retrieve an authentication token and authenticate Docker client
(Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com

# 2. Build your Docker image
docker build -t mlops/retail-api .

# 3. Tag your image
docker tag mlops/retail-api:latest 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# 4. Push image to ECR
docker push 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
```

   **Or use AWS CLI:**

```bash
# 1. Retrieve an authentication token and authenticate Docker client
aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com

# 2. Build your Docker image
docker build -t mlops/retail-api .

# 3. Tag your image  
docker tag mlops/retail-api:latest 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# 4. Push image to ECR
docker push 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
```

{{% notice info %}}
**Info:** On Windows/PowerShell, prefer using `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <registry>` in CI to avoid deprecated commands. ECR tokens typically expire after ~12 hours; re-authenticate for long-running sessions.
{{% /notice %}}


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

```bash
# Basic configuration
AWS_DEFAULT_REGION=ap-southeast-1
MODEL_BUCKET=mlops-retail-forecast-models
LOG_LEVEL=INFO
PORT=8000
```

**Test Docker Image Locally:**

```bash
# Test API container locally
docker run -d \
    --name retail-api-test \
    -p 8000:8000 \
    -e AWS_DEFAULT_REGION=ap-southeast-1 \
    -e MODEL_BUCKET=mlops-retail-prediction-dev-842676018087 \
    842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# Test health endpoint
curl http://localhost:8000/health

# Test API documentation
open http://localhost:8000/docs

# Clean up
docker stop retail-api-test && docker rm retail-api-test
```

{{% notice warning %}}
**Warning (Local):** When running image on local machine, avoid mounting secrets or AWS credentials into container. Use environment variables only for non-sensitive values and prefer IAM roles for production environment.
{{% /notice %}}

- Local container test for retail-api :

![](/images/06-ecr-registry/11.png)

**Complete!** 🎉

ECR registry has been set up and integrated with EKS cluster `mlops-retail-cluster`. Docker image for retail API is ready to deploy on Kubernetes in Task 10.

## Task 6 Results

✅ **ECR Repository** - mlops/retail-api repository  
✅ **Container Image** - FastAPI prediction service  
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month  

{{% notice success %}}
**🎯 Task 6 Complete - ECR Registry + API Containerization!**

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
