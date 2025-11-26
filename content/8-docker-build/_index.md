---
title: "API Containerization"
date: 2024-01-01T00:00:00+07:00
weight: 8
chapter: false
pre: "<b>8. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 9:**

Đóng gói toàn bộ Retail Prediction API (FastAPI + model đã huấn luyện) vào Docker image, sẵn sàng để deploy lên EKS.
→ Đảm bảo môi trường thống nhất, dễ tái sử dụng và sẵn sàng cho EKS deployment.
{{% /notice %}}

## Tổng quan

**API Containerization** là bước quan trọng trong MLOps pipeline, đóng gói ứng dụng dự đoán (Prediction API) thành container image để triển khai trên Kubernetes. Task này tập trung vào việc containerize Retail Prediction API với model đã huấn luyện và tích hợp với ECR registry.

### Thành phần chính

1. **API Application**: FastAPI application với endpoint dự đoán BASKET_PRICE_SENSITIVITY
2. **Model Integration**: Tải model artifacts từ S3 khi container khởi động
3. **Dockerfile**: Multi-stage build để tối ưu kích thước và bảo mật
4. **S3 Connectivity**: Kết nối với model storage thông qua IAM roles
5. **Local Testing**: Validation và testing trước khi deploy lên EKS

---

## 1. Cấu trúc API và Implementation

### 1.1. Cấu trúc thư mục API

```
server/
├── main.py               # FastAPI application
├── model_loader.py       # Model loading từ S3
├── prediction_service.py # Prediction logic
├── health_check.py       # Health check script
├── index.html            # Web UI cho testing
├── Dockerfile            # Multi-stage build
├── requirements.txt      # Python dependencies
└── README.md             # Documentation
```

{{% notice tip %}}
Cấu trúc này đơn giản và dễ maintain. Code thực tế đã được tạo sẵn trong thư mục `retail-price_sensitivy_prediction/server/` và được reference trong Task 6 (ECR Registry).
{{% /notice %}}

### 1.2. FastAPI Application Reference

{{% notice info %}}
**Code đã được implement trong Task 6 - ECR Registry (Section 2.1)**

Các files sau đã được tạo sẵn và có thể tham khảo trong Task 6:
- `server/main.py` (145 dòng): FastAPI app với 6 endpoints
- `server/model_loader.py` (150 dòng): S3 model loading với mock fallback  
- `server/prediction_service.py` (115 dòng): Feature preprocessing và prediction
- `server/health_check.py` (24 dòng): Docker HEALTHCHECK script
- `server/requirements.txt`: Python dependencies
- `server/Dockerfile`: Multi-stage Docker build

Task này tập trung vào **containerization workflow** và **deployment process**.
{{% /notice %}}

**Key Features của API đã implement:**
- ✅ FastAPI với CORS middleware
- ✅ Pydantic validation models
- ✅ Model loading từ S3 (với mock fallback cho testing)
- ✅ Health check endpoints (`/health`, `/ready`)
- ✅ Prediction endpoints (`/predict`, `/predict/batch`)
- ✅ Model info endpoint (`/model/info`)
- ✅ Error handling và logging
## 2. Dockerfile Configuration

**Dockerfile đã được tạo trong Task 6 - Section 2.1:**

Chi tiết cấu hình Docker image với multi-stage build, non-root user, health checks đã được document đầy đủ trong Task 6. File `server/Dockerfile` bao gồm:

- ✅ Multi-stage build (builder + production)
- ✅ Python 3.9-slim base image
- ✅ Non-root user `apiuser` cho security
- ✅ Health check configuration
- ✅ Environment variables cho S3 model loading
- ✅ Port 8000 exposed
- ✅ Uvicorn ASGI server

**Reference:** Xem Task 6, Section 2.1 để biết chi tiết Dockerfile configuration.

## 2. .dockerignore Configuration

**Tạo `server/.dockerignore` - Exclude Unnecessary Files:**

```
# Git files
.git
.gitignore

# Python artifacts
__pycache__/
*.py[cod]
*.so
.Python
env/
venv/
*.egg-info/
.pytest_cache/

# Editor files
.idea/
.vscode/
*.swp

# OS files
.DS_Store
Thumbs.db

# Development files
.env
*.log
logs/

# Large model files (downloaded at runtime)
*.joblib
*.pkl
model/
ENV PORT=8080

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create non-root user
RUN groupadd -r appuser && useradd --no-log-init -r -g appuser appuser

# Set working directory
WORKDIR /app

# Copy virtual environment from builder stage
COPY --from=builder /opt/venv /opt/venv

# Copy application code
COPY --chown=appuser:appuser ./app /app/app

# Create model directory and set permissions
RUN mkdir -p /app/model /app/logs && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:${PORT}/health || exit 1

# Run application
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 2.2. .dockerignore Configuration

**Tạo `server/.dockerignore` - Exclude Unnecessary Files:**

```
# Git files
.git
.gitignore
.gitattributes

# Python artifacts
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
*.egg-info/
.pytest_cache/
.coverage
htmlcov/
.tox/

# Editor files
.idea/
.vscode/
*.swp
*.swo

# OS generated files
.DS_Store
Thumbs.db

# Development files
.env
.env.*
*.log
logs/
docker-compose.yml
docker-compose.yaml

# Documentation
README.md
docs/
*.md

# Test files
tests/
test_*

# Large model files that should be downloaded at runtime
*.joblib
*.pkl
*.h5
model/
```

## 3. Build & Push Docker Image

## 3. Build & Push Docker Image

### 3.1. Local Build và Test

{{% notice tip %}}  
Code trong `server/` đã sẵn sàng để build. Chỉ cần navigate đến thư mục và build Docker image.
{{% /notice %}}

1. **Navigate to Server Directory:**
   ```bash
   cd retail-price_sensitivy_prediction/server
   cd server/
   ```

2. **Build Docker Image:**
   ```bash
   # Get current git information
   GIT_COMMIT=$(git rev-parse --short HEAD)
   GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
   BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
   
   # Build with build arguments
   docker build \
     --build-arg BUILD_DATE=$BUILD_DATE \
     --build-arg GIT_COMMIT=$GIT_COMMIT \
     --build-arg GIT_BRANCH=$GIT_BRANCH \
     -t retail-prediction-api:$GIT_COMMIT \
     .
   ```

3. **Test Image Locally:**
   ```bash
   # Run container locally
   docker run -d \
     --name retail-api-test \
     -p 8080:8080 \
     retail-prediction-api:$GIT_COMMIT
   
   # Wait for container to start
   sleep 5
   
   # Test health endpoint
   curl http://localhost:8080/health
   
   # Test API endpoint with sample basket
   curl -X POST http://localhost:8080/predict \
     -H "Content-Type: application/json" \
     -d '{
       "basket_items": {
         "P1001": {"product_id": "P1001", "quantity": 2, "price": 10.99, "category": "grocery"},
         "P2002": {"product_id": "P2002", "quantity": 1, "price": 25.50, "category": "electronics"}
       },
       "customer_id": "CUST123"
     }'
   
   # Stop test container
   docker stop retail-api-test
   docker rm retail-api-test
   ```

### 3.2. Push to Amazon ECR

1. **ECR Authentication:**
   ```bash
   # Get AWS account ID
   AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
   AWS_REGION="ap-southeast-1"
   ECR_REPOSITORY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/retail-prediction-api"
   
   # Login to ECR
   aws ecr get-login-password --region $AWS_REGION | \
     docker login --username AWS --password-stdin $ECR_REPOSITORY
   ```

2. **Create ECR Repository (if not exists):**
   ```bash
   # Check if repository exists
   if ! aws ecr describe-repositories --repository-names retail-prediction-api --region $AWS_REGION &> /dev/null; then
     echo "Creating ECR repository retail-prediction-api"
     aws ecr create-repository \
       --repository-name retail-prediction-api \
       --image-scanning-configuration scanOnPush=true \
       --region $AWS_REGION
   fi
   ```

3. **Tag Images:**
   ```bash
   # Tag for ECR with different tag strategies
   docker tag retail-prediction-api:$GIT_COMMIT $ECR_REPOSITORY:latest
   docker tag retail-prediction-api:$GIT_COMMIT $ECR_REPOSITORY:$GIT_COMMIT
   docker tag retail-prediction-api:$GIT_COMMIT $ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT
   ```

4. **Push to ECR:**
   ```bash
   # Push all tags
   docker push $ECR_REPOSITORY:latest
   docker push $ECR_REPOSITORY:$GIT_COMMIT
   docker push $ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT
   
   echo "Images pushed to ECR successfully"
   ```

### 3.3. Create Build & Push Script

**Tạo `scripts/build-push-api.sh` - Automated Build Script:**

```bash
#!/bin/bash

set -e

# Script configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
SERVER_DIR="$PROJECT_ROOT/server"
AWS_REGION="ap-southeast-1"
ECR_REPO_NAME="retail-prediction-api"

# Colors for output
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Print colored messages
info() { echo -e "${BLUE}INFO: $1${NC}"; }
success() { echo -e "${GREEN}SUCCESS: $1${NC}"; }
warn() { echo -e "${YELLOW}WARNING: $1${NC}"; }
error() { echo -e "${RED}ERROR: $1${NC}"; exit 1; }

# Validate requirements
command -v docker >/dev/null 2>&1 || error "Docker is required but not installed"
command -v aws >/dev/null 2>&1 || error "AWS CLI is required but not installed"

# Get git information
GIT_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')

info "Building API container image"
info "Git commit: $GIT_COMMIT"
info "Git branch: $GIT_BRANCH"

# Validate server directory
[ -d "$SERVER_DIR" ] || error "Server directory not found: $SERVER_DIR"

# Navigate to server directory  
cd "$SERVER_DIR"

# Build Docker image
info "Building Docker image..."
docker build \
  --build-arg BUILD_DATE="$BUILD_DATE" \
  --build-arg GIT_COMMIT="$GIT_COMMIT" \
  --build-arg GIT_BRANCH="$GIT_BRANCH" \
  -t "$ECR_REPO_NAME:$GIT_COMMIT" \
  .

success "Docker image built successfully"

# Test the image
info "Running container for testing..."
CONTAINER_ID=$(docker run -d -p 8000:8000 "$ECR_REPO_NAME:$GIT_COMMIT")
  --build-arg BUILD_DATE="$BUILD_DATE" \
  --build-arg GIT_COMMIT="$GIT_COMMIT" \
  --build-arg GIT_BRANCH="$GIT_BRANCH" \
  -t "$ECR_REPO_NAME:$GIT_COMMIT" \
  .

success "Docker image built successfully"

# Test the image
info "Running container for testing..."
CONTAINER_ID=$(docker run -d -p 8080:8080 "$ECR_REPO_NAME:$GIT_COMMIT")

info "Waiting for container to start..."
sleep 10

# Test health endpoint
if curl -s -f http://localhost:8080/health > /dev/null; then
  success "Health check passed"
else
  warn "Health check failed, but continuing with push"
fi

# Stop and remove the test container
docker stop "$CONTAINER_ID" > /dev/null
docker rm "$CONTAINER_ID" > /dev/null

# Get AWS account ID
info "Getting AWS account ID..."
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
[ -z "$AWS_ACCOUNT_ID" ] && error "Failed to get AWS account ID"

ECR_REPOSITORY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME"

# Create ECR repository if it doesn't exist
info "Checking ECR repository..."
if ! aws ecr describe-repositories --repository-names "$ECR_REPO_NAME" --region "$AWS_REGION" &> /dev/null; then
  info "Creating ECR repository: $ECR_REPO_NAME"
  aws ecr create-repository \
    --repository-name "$ECR_REPO_NAME" \
    --image-scanning-configuration scanOnPush=true \
    --region "$AWS_REGION"
fi

# Login to ECR
info "Logging in to ECR..."
aws ecr get-login-password --region "$AWS_REGION" | \
  docker login --username AWS --password-stdin "$ECR_REPOSITORY"

# Tag images
info "Tagging images..."
docker tag "$ECR_REPO_NAME:$GIT_COMMIT" "$ECR_REPOSITORY:latest"
docker tag "$ECR_REPO_NAME:$GIT_COMMIT" "$ECR_REPOSITORY:$GIT_COMMIT"
docker tag "$ECR_REPO_NAME:$GIT_COMMIT" "$ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT"

# Push images to ECR
info "Pushing images to ECR..."
docker push "$ECR_REPOSITORY:latest"
docker push "$ECR_REPOSITORY:$GIT_COMMIT"
docker push "$ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT"

success "Images successfully pushed to ECR"
echo "📦 Image URLs:"
echo "  - $ECR_REPOSITORY:latest"
echo "  - $ECR_REPOSITORY:$GIT_COMMIT"
echo "  - $ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT"

# Clean up local images
info "Cleaning up local images..."
docker rmi "$ECR_REPO_NAME:$GIT_COMMIT" "$ECR_REPOSITORY:latest" "$ECR_REPOSITORY:$GIT_COMMIT" "$ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT" > /dev/null 2>&1 || true

success "Build and push completed successfully!"
```

### 3.4. Verify in ECR Console

After pushing the image to ECR, you can verify it in the AWS Management Console:

1. **Navigate to the Amazon ECR service**
2. **Select the `retail-prediction-api` repository**
3. **Verify the pushed images with multiple tags**
4. **Check the vulnerability scanning results**

{{% notice tip %}}
Automated vulnerability scanning is enabled with `scanOnPush=true` when creating the ECR repository. This helps identify security issues in your container image.
{{% /notice %}}

## 4. Model Integration with S3

### 4.1. Model Storage Strategy

When containerizing an ML API, you need a strategy for including the trained model. The most common approaches are:

1. **Embedding the model in the container image:**
   - **Pros:** Self-contained, no external dependencies
   - **Cons:** Large image size, difficult to update models independently

2. **Downloading the model at runtime (recommended):**
   - **Pros:** Smaller container images, models can be updated independently
   - **Cons:** Adds startup latency, requires S3 access

3. **Mounting from a persistent volume:**
   - **Pros:** Good for very large models, rapid container startup
   - **Cons:** More complex Kubernetes setup

For our API, we're using the second approach (download at runtime) as it provides the best balance of flexibility and simplicity.

### 4.2. Model S3 Integration

**Setup for storing and accessing model artifacts from S3:**

1. **Create an S3 bucket for model artifacts:**

```bash
# Create S3 bucket for models
aws s3 mb s3://mlops-retail-models --region ap-southeast-1
```

2. **Upload model artifacts to S3:**

```bash
# Assume we have a trained model in a tar.gz archive
aws s3 cp model.tar.gz s3://mlops-retail-models/artifacts/model-v1/model.tar.gz
```

3. **Configure IAM policy for S3 access:**

Create an IAM policy that allows the API container to access the S3 bucket where models are stored:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mlops-retail-models",
        "arn:aws:s3:::mlops-retail-models/*"
      ]
    }
  ]
}
```

### 4.3. IAM Roles for Service Accounts (IRSA)

For Kubernetes deployments, use IRSA to grant the API container access to S3 without storing AWS credentials:

1. **Create an IAM policy:**

```bash
# Create IAM policy
aws iam create-policy \
  --policy-name RetailAPIModelAccess \
  --policy-document file://model-access-policy.json
```

2. **Associate the policy with a Kubernetes service account:**

```bash
# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster=mlops-eks-cluster \
  --namespace=retail-prediction \
  --name=retail-api \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/RetailAPIModelAccess \
  --approve
```

3. **Reference the service account in your Kubernetes deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-prediction-api
  namespace: retail-prediction
spec:
  selector:
    matchLabels:
      app: retail-prediction-api
  replicas: 2
  template:
    metadata:
      labels:
        app: retail-prediction-api
    spec:
      serviceAccountName: retail-api  # Reference to IRSA-enabled service account
      containers:
      - name: api
        image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/retail-prediction-api:latest
        env:
        - name: MODEL_BUCKET
          value: "mlops-retail-models"
        - name: MODEL_KEY
          value: "artifacts/model-v1/model.tar.gz"
```

## 6. Best Practices & Considerations

### 6.1. Security Best Practices

When containerizing ML APIs, implement these security best practices:

1. **Use non-root users:**
   - Always run container processes as a non-root user
   - Create dedicated user/group with minimal permissions
   
2. **Multi-stage builds for smaller attack surface:**
   - Include only necessary dependencies in the final image
   - Avoid exposing build tools in runtime image

3. **Image vulnerability scanning:**
   - Integrate scanning in CI/CD pipelines
   - Enable automatic scanning in ECR

4. **Secrets management:**
   - Never hardcode credentials in Dockerfiles or images
   - Use AWS Secrets Manager or SSM Parameter Store
   - Leverage IAM roles for secure access

5. **Update dependencies:**
   - Regularly update base images and dependencies
   - Use dependabot or similar tools to track vulnerabilities

### 6.2. Performance Optimization

For optimal API container performance:

1. **Profiling:**
   - Monitor container resource usage during load testing
   - Identify memory/CPU bottlenecks

2. **Asynchronous processing:**
   - Use async functions for I/O operations
   - Implement background tasks for model loading

3. **Caching strategies:**
   - Cache model predictions where appropriate
   - Use in-memory caching or Redis for high-traffic scenarios

4. **Resource limits:**
   - Set appropriate memory and CPU limits
   - Configure JVM settings if using frameworks like SparkML

### 6.3. Scaling Considerations

When deploying the containerized API to EKS:

1. **Horizontal vs. vertical scaling:**
   - Prefer horizontal scaling for stateless APIs
   - Consider vertical scaling for memory-intensive ML models

2. **Autoscaling:**
   - Implement Horizontal Pod Autoscaler (HPA)
   - Configure custom metrics based on prediction load

3. **Startup optimization:**
   - Optimize container startup time for faster scaling
   - Consider model loading strategies (preload vs. lazy load)

4. **Load testing:**
   - Simulate production traffic patterns
   - Measure latency under various load conditions

## 7. Summary

In this task, we've successfully containerized the Retail Prediction API, making it ready for deployment on Amazon EKS. We've covered:

1. **API Implementation:** Created a FastAPI application with model prediction endpoints, health checks, and proper error handling.

2. **Docker Containerization:** Implemented a multi-stage Dockerfile optimized for size, security, and performance.

3. **Model Integration:** Set up a flexible strategy for loading ML models from S3 at runtime.

4. **IAM Configuration:** Established secure access to AWS resources using IAM Roles for Service Accounts.

5. **Local Testing & Validation:** Comprehensive testing before EKS deployment.

This containerization approach ensures consistent environments across development and production, simplifies deployment, and integrates well with Kubernetes for scalable, resilient operation.

The containerized API is now ready to be deployed to Amazon EKS, which will be covered in the next task.

---

### 3.1. GitHub Actions Workflow

**Tạo `.github/workflows/docker-build.yml`:**

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main, develop]
    paths:
      - 'server/**'
      - '.github/workflows/docker-build.yml'
  pull_request:
    branches: [main]
    paths:
      - 'server/**'

env:
  AWS_REGION: ap-southeast-1
  ECR_REPOSITORY: retail-forecast

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Extract metadata
      id: meta
      run: |
        echo "git-commit=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
        echo "git-branch=${GITHUB_REF#refs/heads/}" >> $GITHUB_OUTPUT
        echo "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT
        echo "image-tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: ./server
        file: ./server/Dockerfile
        push: true
        tags: |
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.git-branch }}-${{ steps.meta.outputs.image-tag }}
        build-args: |
          BUILD_DATE=${{ steps.meta.outputs.build-date }}
          GIT_COMMIT=${{ steps.meta.outputs.git-commit }}
          GIT_BRANCH=${{ steps.meta.outputs.git-branch }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Test container
      run: |
        # Run container for testing
        docker run -d --name test-container \
          -p 8000:8000 \
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
        
        # Wait for container to start
        sleep 30
        
        # Test health endpoint
        curl -f http://localhost:8000/health || exit 1
        
        # Stop test container
        docker stop test-container
        docker rm test-container

    - name: Output image details
      run: |
        echo "🚀 Successfully built and pushed:"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.git-branch }}-${{ steps.meta.outputs.image-tag }}"
```

### 3.2. Jenkins Pipeline

**Tạo `aws/Jenkinsfile`:**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-southeast-1'
        ECR_REPOSITORY = 'retail-forecast'
        PROJECT_NAME = 'mlops-retail-forecast'
        ENVIRONMENT = 'dev'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
                    env.BUILD_DATE = sh(returnStdout: true, script: 'date -u +%Y-%m-%dT%H:%M:%SZ').trim()
                }
            }
        }
        
        stage('Validate') {
            steps {
                script {
                    // Check required files
                    def requiredFiles = [
                        'server/Dockerfile',
                        'server/main.py', 
                        'server/requirements.txt'
                    ]
                    
                    requiredFiles.each { file ->
                        if (!fileExists(file)) {
                            error("Required file not found: ${file}")
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('server') {
                        def imageTag = "${env.GIT_COMMIT}"
                        def imageName = "${env.ECR_REPOSITORY}:${imageTag}"
                        
                        sh """
                            docker build \
                                --build-arg BUILD_DATE='${env.BUILD_DATE}' \
                                --build-arg GIT_COMMIT='${env.GIT_COMMIT}' \
                                --build-arg GIT_BRANCH='${env.GIT_BRANCH}' \
                                -t ${imageName} \
                                .
                        """
                        
                        env.IMAGE_NAME = imageName
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }
        
        stage('Test Image') {
            steps {
                script {
                    // Run container for testing
                    def containerId = sh(
                        returnStdout: true,
                        script: "docker run -d -p 8001:8000 ${env.IMAGE_NAME}"
                    ).trim()
                    
                    try {
                        // Wait for container to start
                        sleep(30)
                        
                        // Test health endpoint
                        sh "curl -f http://localhost:8001/health"
                        
                        echo "✅ Container health check passed"
                        
                    } finally {
                        // Always clean up
                        sh "docker stop ${containerId} || true"
                        sh "docker rm ${containerId} || true"
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Run Trivy security scan
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v \$PWD:/workspace \
                            aquasec/trivy:latest \
                            image --format json --output /workspace/trivy-report.json \
                            ${env.IMAGE_NAME}
                    """
                    
                    // Archive scan results
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }
        
        stage('Push to ECR') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    // Get AWS account ID
                    def awsAccountId = sh(
                        returnStdout: true,
                        script: 'aws sts get-caller-identity --query Account --output text'
                    ).trim()
                    
                    def ecrRegistry = "${awsAccountId}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    def ecrRepository = "${ecrRegistry}/${env.ECR_REPOSITORY}"
                    
                    // Login to ECR
                    sh """
                        aws ecr get-login-password --region ${env.AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ecrRegistry}
                    """
                    
                    // Tag images
                    def tags = ['latest', env.IMAGE_TAG, "${env.GIT_BRANCH}-${env.IMAGE_TAG}"]
                    
                    tags.each { tag ->
                        sh "docker tag ${env.IMAGE_NAME} ${ecrRepository}:${tag}"
                        sh "docker push ${ecrRepository}:${tag}"
                        echo "✅ Pushed ${ecrRepository}:${tag}"
                    }
                    
                    // Store ECR URLs for later stages
                    env.ECR_REPOSITORY_URL = ecrRepository
                }
            }
        }
        
        stage('Update Deployment') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Update Kubernetes deployment
                    sh """
                        kubectl set image deployment/retail-forecast-api \
                            api=${env.ECR_REPOSITORY_URL}:${env.IMAGE_TAG} \
                            --record
                    """
                    
                    echo "✅ Deployment updated with new image"
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    // Clean up local images
                    sh "docker rmi ${env.IMAGE_NAME} || true"
                    sh "docker system prune -f || true"
                }
            }
        }
    }
    
    post {
        always {
            // Clean up workspace
            cleanWs()
        }
        
        success {
            echo """
            🎉 Pipeline completed successfully!
            
            📍 Image URLs:
            - ${env.ECR_REPOSITORY_URL}:latest
            - ${env.ECR_REPOSITORY_URL}:${env.IMAGE_TAG}
            - ${env.ECR_REPOSITORY_URL}:${env.GIT_BRANCH}-${env.IMAGE_TAG}
            
            🚀 Next steps:
            - Verify deployment in EKS
            - Run integration tests
            - Monitor application metrics
            """
        }
        
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
```

---

## 4. Image Optimization & Best Practices

### 4.1. Multi-stage Build Optimization

**Advanced Dockerfile với optimization:**

```dockerfile
# Build stage với dependency caching
FROM python:3.9-slim as dependencies

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy requirements first for Docker layer caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --user -r requirements.txt

# Production base image
FROM python:3.9-slim as base

# Install only runtime dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Production stage
FROM base as production

ARG BUILD_DATE
ARG GIT_COMMIT  
ARG GIT_BRANCH

# Metadata labels
LABEL maintainer="MLOps Team"
LABEL build_date=$BUILD_DATE
LABEL git_commit=$GIT_COMMIT
LABEL git_branch=$GIT_BRANCH
LABEL description="Retail Forecast ML Inference API"

WORKDIR /app

# Copy dependencies from builder
COPY --from=dependencies /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Create required directories
RUN mkdir -p /app/model /app/logs && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1
ENV AWS_DEFAULT_REGION=ap-southeast-1

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Start application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

### 4.2. .dockerignore Configuration

**Tạo `server/.dockerignore`:**

```
# Git
.git
.gitignore

# Python
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.pytest_cache

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Documentation
README.md
*.md
docs/

# Test files
test_*
*_test.py
tests/

# Development files
docker-compose.yml
Dockerfile.dev

# Logs
logs/
*.log

# Temporary files
tmp/
temp/
.tmp

# Build artifacts
build/
dist/
*.egg-info/

# Model files (large files should be downloaded at runtime)
model/*.pkl
model/*.joblib
*.model
```

### 4.3. Image Size Optimization

**Strategies để giảm image size:**

1. **Use Alpine base images:**
```dockerfile
FROM python:3.9-alpine as builder
# Install build dependencies for Alpine
RUN apk add --no-cache gcc musl-dev libffi-dev
```

2. **Multi-stage builds:**
```dockerfile
# Only copy production dependencies
COPY --from=builder /root/.local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
```

3. **Remove unnecessary packages:**
```dockerfile
RUN apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 4.4. Security Best Practices

**Security hardening:**

```dockerfile
# Use specific version tags
FROM python:3.9.18-slim

# Don't run as root
USER appuser

# Use read-only filesystem
COPY --chown=appuser:appuser --chmod=755 ./scripts/ /app/scripts/

# Set security headers
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Health check with timeout
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

---

## 5. Testing & Validation

### 5.1. Local Container Testing

**Test API container locally:**

```bash
# Run container with environment variables
docker run -d \
    --name retail-api-test \
    -p 8080:8080 \
    -e AWS_DEFAULT_REGION=ap-southeast-1 \
    -e MODEL_BUCKET=mlops-retail-prediction-dev-us-east-1 \
    -e MODEL_PREFIX=artifacts/ \
    retail-prediction-api:latest

# Wait for startup
sleep 10

# Test health endpoint
curl http://localhost:8080/health

# Test prediction endpoint
curl -X POST http://localhost:8080/predict \
    -H "Content-Type: application/json" \
    -d '{
        "basket_items": {
            "item1": {"price": 10.99, "quantity": 2, "category": "food"},
            "item2": {"price": 25.50, "quantity": 1, "category": "electronics"}
        },
        "customer_id": "test-customer-123"
    }'

# Check container logs
docker logs retail-api-test

# Clean up
docker stop retail-api-test
docker rm retail-api-test
```

### 5.2. Image Security Scanning

**Run vulnerability scans:**

```bash
# Scan image with ECR
aws ecr start-image-scan \
    --repository-name retail-prediction-api \
    --image-id imageTag=latest \
    --region ap-southeast-1

# Get scan results
aws ecr describe-image-scan-findings \
    --repository-name retail-prediction-api \
    --image-id imageTag=latest \
    --region ap-southeast-1

# Use Trivy for local scanning (optional)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy:latest image retail-prediction-api:latest
```

### 5.3. EKS Deployment Preparation
  push:
    branches: [ main ]
    paths:
      - 'server/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'server/**'

env:
  AWS_REGION: ap-southeast-1
  ECR_REPOSITORY: retail-prediction-api

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      run: |
        cd server
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }} .
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }} $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }}
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
        echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }}"
    
    - name: Fill in the new image ID in the Amazon EKS deployment
      if: github.ref == 'refs/heads/main'
      run: |
        aws eks update-kubeconfig --name mlops-eks-cluster --region $AWS_REGION
        kubectl set image deployment/retail-prediction-api api=${{ steps.build-image.outputs.image }} -n retail-prediction
        kubectl rollout status deployment/retail-prediction-api -n retail-prediction
```

## 5.2. EKS Deployment

Create a basic Kubernetes deployment manifest for the prediction API:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-prediction-api
  namespace: retail-prediction
spec:
  replicas: 3
  selector:
    matchLabels:
      app: retail-prediction-api
  template:
    metadata:
      labels:
        app: retail-prediction-api
    spec:
      serviceAccountName: retail-api  # IRSA-enabled service account
      containers:
      - name: api
        image: ${AWS_ACCOUNT_ID}.dkr.ecr.ap-southeast-1.amazonaws.com/retail-prediction-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
          limits:
            memory: "2Gi"
            cpu: "1"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 15
        env:
        - name: LOG_LEVEL
          value: "INFO"
        - name: MODEL_BUCKET
          value: "mlops-retail-models"
        - name: MODEL_KEY
          value: "artifacts/model-v1/model.tar.gz"

---
apiVersion: v1
kind: Service
metadata:
  name: retail-prediction-api
  namespace: retail-prediction
spec:
  selector:
    app: retail-prediction-api
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
```
```bash
# Solution: Use pip-tools for dependency management
pip install pip-tools
pip-compile requirements.in  # Generate requirements.txt with locked versions
```

**Issue 2: Large image sizes**
```bash
# Analyze image layers
docker history retail-forecast:latest

# Use dive tool for detailed analysis
dive retail-forecast:latest
```

**Issue 3: ECR push fails with authentication errors**
```bash
# Refresh ECR login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin $ECR_REPOSITORY

# Check AWS credentials
aws sts get-caller-identity
```

**Issue 4: Container startup fails**
```bash
# Debug container startup
docker run -it --entrypoint /bin/bash retail-forecast:latest

# Check application logs
docker logs <container-id>
```

---

{{% notice success %}}
**🎯 Task 9 Complete!**

API Containerization đã được hoàn thành thành công với:

✅ **FastAPI inference application** với health checks và monitoring  
✅ **Multi-stage Dockerfile** optimized cho production  
✅ **Manual build & test scripts** (Linux/macOS + Windows PowerShell)  
✅ **Local testing** và container validation  
✅ **Image optimization** và security best practices  
✅ **ECR integration** với manual push và scanning  
✅ **EKS deployment preparation** với Kubernetes manifests  

**Next Steps:**
- Task 10: Deploy container lên EKS cluster
- Task 11: Configure Load Balancer cho API access
- Task 12: Setup monitoring và logging
{{% /notice %}}

{{% notice tip %}}
**💡 Production Considerations:**

- **Security scanning**: Regular vulnerability scans với ECR/Trivy
- **Image signing**: Use AWS Signer cho supply chain security
- **Multi-architecture builds**: Support ARM64 cho cost optimization
- **Layer caching**: Optimize Dockerfile layer ordering
- **Version management**: Consistent tagging strategy
- **Performance monitoring**: Container resource usage và response times
{{% /notice %}}