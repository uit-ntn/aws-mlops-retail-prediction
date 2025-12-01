---
title: "ECR Container Registry"
date: 2024-01-01T00:00:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 6:** Thiết lập Amazon Elastic Container Registry (ECR) cho MLOps pipeline:
1. **Tạo ECR Repository**: Repository cho API container
2. **Cấu hình Security**: Image scanning, IAM policy, lifecycle rules  
3. **Build & Push Image**: Upload FastAPI container lên ECR
4. **Manual Build & Push**: Hướng dẫn build/push bằng script (CLI / PowerShell)
{{% /notice %}}

📥 **Input từ các Task trước:**
- **Task 2 (IAM Roles & Audit):** IAM roles, policies và permissions cho ECR/EKS/S3 access
- **Task 5 (Production VPC):** VPC endpoints, networking và security groups để cho phép EKS pull images từ ECR

📦 **Output:**
- **Inference Container**: `server/` code → FastAPI API serving predictions trong EKS

## Tổng quan

**Amazon ECR (Elastic Container Registry)** là dịch vụ Docker container registry được quản lý hoàn toàn bởi AWS, tích hợp sâu với EKS và CI/CD pipeline. ECR cung cấp khả năng lưu trữ, quản lý và triển khai container images một cách an toàn cho MLOps workflow.

## 1. ECR Repositories Setup

### 1.1. Create ECR Repositories

1. **Navigate to ECR Console:**
   - Đăng nhập AWS Console
   - Navigate to Amazon ECR service
   - Region: ap-southeast-1
   - Chọn "Create repository"

![](/images/06-ecr-registry/01.png)

2. **API Repository Configuration:**

![](/images/06-ecr-registry/02.png)

3. **Repository Created Successfully:**
   
   Sau khi tạo repository, bạn sẽ thấy giao diện như hình dưới với thông tin:
   
   - Repository name: `mlops/retail-api`
   - Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
   - Status: "No active images" (chưa có image nào được push)
   - Các tab: Summary, Images, Permissions, Lifecycle policy, Repository tags

![](/images/06-ecr-registry/03.1.png)

4. **Repository Setup Complete:**
   
   API repository đã sẵn sàng cho containerized FastAPI application.

5. **Repository Management Interface:**
   
   Trong giao diện quản lý repository, bạn có thể:
   - **Images tab**: Xem danh sách images, filter theo tags
   - **View push commands**: Lệnh để push image lên repository  
   - **Copy URI**: Copy repository URI để sử dụng
   - **Scan**: Quét vulnerability cho images
   - **Delete**: Xóa repository khi không cần

![](/images/06-ecr-registry/04.png)

### 1.2. Lifecycle Policy Setup

1. **API Repository Lifecycle Policy:**
   - Chọn repository `mlops/retail-api`
   - Click tab "Lifecycle policy" 
   - Click "Create rule" để tạo lifecycle policy

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
   - Chọn repository từ danh sách
   - Kiểm tra "Scan on push" đã được enabled
   - Review enhanced scanning options nếu cần

2. **View Push Commands:**
   - Click nút "View push commands" trong giao diện repository
   - AWS sẽ hiển thị các lệnh để authenticate và push image
   - Copy các lệnh này để sử dụng từ local machine hoặc CI/CD pipeline

![](/images/06-ecr-registry/09.1.png)

![](/images/06-ecr-registry/09.2.png)

{{% notice success %}}
**🎯 ECR Repositories Setup Complete!**

**Created Repository:**

- ✅ `mlops/retail-api`: FastAPI prediction service container
- ✅ Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
- ✅ Private repository với tag immutability enabled
- ✅ Image scanning enabled on push
- ✅ Lifecycle policies configured for cost optimization
- ✅ Push commands available trong console
- ✅ IAM access policies for EKS integration
{{% /notice %}}

## 2. API Containerization Workflow

### 2.1. Dockerfile Configuration

**Tạo `server/Dockerfile` - Multi-stage build:**

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

**Tạo `server/.dockerignore`:**

```
# Development files
.git
.gitignore
__pycache__/
*.pyc
.env
*.log

# Editor files  
.idea/
.vscode/

# Large files (downloaded at runtime)
*.joblib
*.pkl
model/
```

### 2.2. Local Build & Test

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

### 2.3. View Push Commands từ AWS Console

1. **Trong ECR Console:**
   - Chọn repository `mlops/retail-api`
   - Click nút **"View push commands"**
   - AWS sẽ hiển thị các lệnh để build và push

2. **Các lệnh push commands sẽ như (Windows PowerShell):**

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

   **Hoặc sử dụng AWS CLI:**

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

### 2.2. Verify ECR Push Success

**Kiểm tra trong AWS Console:**

1. **Navigate to ECR Console:**
   - Vào AWS Console → ECR service
   - Chọn repository `mlops/retail-api`
   - Check tab "Images" để xem image đã được push

2. **Expected Result:**
   - Image với tag `latest` xuất hiện trong danh sách
   - Image size hiển thị (~927MB)
   - Vulnerability scan status (if enabled)
   - Push timestamp

![](/images/06-ecr-registry/10.png)

**Kiểm tra bằng CLI:**

![](/images/06-ecr-registry/12.png)

![](/images/06-ecr-registry/13.png)

**Kiểm tra bằng console:**

![](/images/06-ecr-registry/14.png)

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
- Local container test for retail-api :

![](/images/06-ecr-registry/11.png)

**Hoàn thành!** 🎉

ECR registry đã được thiết lập và tích hợp với EKS cluster `mlops-retail-cluster`. Docker image của retail API đã sẵn sàng để deploy trên Kubernetes trong Task 10.

## Kết quả Task 6

✅ **ECR Repository** - mlops/retail-api repository  
✅ **Container Image** - FastAPI prediction service  
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month  

{{% notice success %}}
**🎯 Task 6 Complete - ECR Registry + API Containerization!**

**✅ ECR Setup**: Repository với lifecycle policies & image scanning  
**✅ Dockerfile**: Multi-stage build, non-root user, health checks  
**✅ Build & Push**: Local build → ECR push workflow  
**✅ Testing**: Container verification & API validation  
**✅ Ready**: Sẵn sàng cho EKS deployment trong Task 7  
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 7**: EKS cluster deployment với ECR integration
- **Task 8**: Deploy API container lên EKS với ALB
- **Task 9**: Load balancing và scaling configuration

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
