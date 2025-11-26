---
title: "API Containerization"
date: 2024-01-01T00:00:00+07:00
weight: 8
chapter: false
pre: "<b>8. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 8:**

Đóng gói toàn bộ Retail Prediction API (FastAPI + model đã huấn luyện) vào Docker image, sẵn sàng để deploy lên EKS.
→ Đảm bảo môi trường thống nhất, dễ tái sử dụng và sẵn sàng cho EKS deployment.
{{% /notice %}}

📥 **Input từ Task trước:**
- **Task 6 (ECR Container Registry):** Source code, Dockerfile and image requirements; reference implementation of the `server/` code
- **Task 2 (IAM Roles & Audit):** IAM policies required for pulling from ECR and accessing S3 at runtime

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

### 1.2. API Components

**API đã được implement với các tính năng cốt lõi:**
- ✅ FastAPI application với prediction endpoints
- ✅ Model loading từ S3 
- ✅ Health check endpoints
- ✅ Error handling và logging
- ✅ Docker containerization ready
## 2. Dockerfile Configuration

**Basic Dockerfile cho API containerization:**

```dockerfile
# Multi-stage build
FROM python:3.9-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Production stage  
FROM python:3.9-slim as production
WORKDIR /app

# Copy dependencies
COPY --from=builder /root/.local /root/.local

# Create non-root user
RUN useradd --create-home --shell /bin/bash apiuser
USER apiuser

# Copy application
COPY . .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Start application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

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

### 3.3. Simple Build & Push Script

**Tạo `scripts/build-push.sh`:**

```bash
#!/bin/bash
# Simple build and push script

# Configuration
REGION="ap-southeast-1"
REPO_NAME="mlops/retail-api"

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME"

echo "Building Docker image..."
cd server/
docker build -t $REPO_NAME:latest .

echo "Testing container..."
docker run -d --name test -p 8000:8000 $REPO_NAME:latest
sleep 10
curl -f http://localhost:8000/health
docker stop test && docker rm test

echo "Pushing to ECR..."
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_URI
docker tag $REPO_NAME:latest $ECR_URI:latest
docker push $ECR_URI:latest

echo "✅ Build and push completed!"
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



## 4. Verification

### 4.1. Local Testing

```bash
# Test container locally
docker run -d --name test-api -p 8000:8000 mlops/retail-api:latest

# Test health endpoint
curl http://localhost:8000/health

# Test prediction endpoint
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"basket_items": {"item1": {"price": 10.0, "quantity": 2}}}'

# Clean up
docker stop test-api && docker rm test-api
```

### 4.2. ECR Push Verification

```bash
# Verify image in ECR
aws ecr list-images --repository-name mlops/retail-api --region ap-southeast-1

# Check image scan results
aws ecr describe-image-scan-findings \
  --repository-name mlops/retail-api \
  --image-id imageTag=latest \
  --region ap-southeast-1
```

## 5. Summary

{{% notice success %}}
**🎯 Task 8 Complete - API Containerization**

✅ **Docker image** ready cho deployment  
✅ **Local testing** passed  
✅ **ECR integration** working  
✅ **Build scripts** available cho automation  

**Container sẵn sàng cho EKS deployment trong Task 9!**
{{% /notice %}}

---

**Next Step**: [Task 9: Deploy API lên EKS](../9-deploy-kubernetes/)