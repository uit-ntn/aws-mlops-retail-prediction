---
title: "ECR Container Registry"
date: 2024-01-01T00:00:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 6:** Thiết lập Amazon Elastic Container Registry (ECR) cho MLOps pipeline:
<<<<<<< HEAD
1. **Tạo ECR Repository**: Repository cho API container
2. **Cấu hình Security**: Image scanning, IAM policy, lifecycle rules  
3. **Build & Push Image**: Upload FastAPI container lên ECR
4. **Manual Build & Push**: Hướng dẫn build/push bằng script (CLI / PowerShell)
{{% /notice %}}

📥 **Input từ các Task trước:**
=======

1. **Tạo ECR Repository**: Repository cho API container
2. **Cấu hình Security**: Image scanning, IAM policy, lifecycle rules
3. **Build & Push Image**: Upload FastAPI container lên ECR
4. **Manual Build & Push**: Hướng dẫn build/push bằng script (CLI / PowerShell)
   {{% /notice %}}

📥 **Input từ các Task trước:**

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- **Task 2 (IAM Roles & Audit):** IAM roles, policies và permissions cho ECR/EKS/S3 access
- **Task 5 (Production VPC):** VPC endpoints, networking và security groups để cho phép EKS pull images từ ECR

📦 **Output:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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
<<<<<<< HEAD
   
   Sau khi tạo repository, bạn sẽ thấy giao diện như hình dưới với thông tin:
   
=======

   Sau khi tạo repository, bạn sẽ thấy giao diện như hình dưới với thông tin:

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
   - Repository name: `mlops/retail-api`
   - Repository URI: `<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api`
   - Status: "No active images" (chưa có image nào được push)
   - Các tab: Summary, Images, Permissions, Lifecycle policy, Repository tags

![](/images/06-ecr-registry/03.1.png)

4. **Repository Setup Complete:**
<<<<<<< HEAD
   
   API repository đã sẵn sàng cho containerized FastAPI application.

5. **Repository Management Interface:**
   
   Trong giao diện quản lý repository, bạn có thể:
   - **Images tab**: Xem danh sách images, filter theo tags
   - **View push commands**: Lệnh để push image lên repository  
=======

   API repository đã sẵn sàng cho containerized FastAPI application.

5. **Repository Management Interface:**

   Trong giao diện quản lý repository, bạn có thể:

   - **Images tab**: Xem danh sách images, filter theo tags
   - **View push commands**: Lệnh để push image lên repository
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
   - **Copy URI**: Copy repository URI để sử dụng
   - **Scan**: Quét vulnerability cho images
   - **Delete**: Xóa repository khi không cần

![](/images/06-ecr-registry/04.png)

{{% notice tip %}}
**Tip:** Bật `tag immutability` cho các tag production (ví dụ `v*`) để tránh accidental overwrite. Sử dụng semantic tags (`v1.2.3`, `commit-<sha>`) giúp dễ rollback và audit.
{{% /notice %}}

### 1.2. Lifecycle Policy Setup

1. **API Repository Lifecycle Policy:**
   - Chọn repository `mlops/retail-api`
<<<<<<< HEAD
   - Click tab "Lifecycle policy" 
=======
   - Click tab "Lifecycle policy"
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

<<<<<<< HEAD

=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
3. **Training Repository Lifecycle Policy:**

![](/images/06-ecr-registry/08.png)

### 1.3. Image Scanning & Push Commands

1. **Check Scan Settings:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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
<<<<<<< HEAD
{{% /notice %}}
=======
  {{% /notice %}}
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

{{% notice tip %}}
**Tip:** Ghi chú lifecycle rule priorities trong docs team và test rules trên non-prod repos trước khi áp dụng production để tránh xóa nhầm images.
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

<<<<<<< HEAD
# Production stage  
=======
# Production stage
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

<<<<<<< HEAD
# Editor files  
=======
# Editor files
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

{{% notice warning %}}
**Warning:** Docker login tokens (ECR auth) có thời hạn; CI agents nên refresh token (`aws ecr get-login-password`) per job. Tránh hardcode credentials in scripts or environment files.
{{% /notice %}}

<<<<<<< HEAD

### 2.3. View Push Commands từ AWS Console

1. **Trong ECR Console:**
=======
### 2.3. View Push Commands từ AWS Console

1. **Trong ECR Console:**

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

<<<<<<< HEAD
   **Hoặc sử dụng AWS CLI:**
=======
**Hoặc sử dụng AWS CLI:**
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```bash
# 1. Retrieve an authentication token and authenticate Docker client
aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com

# 2. Build your Docker image
docker build -t mlops/retail-api .

<<<<<<< HEAD
# 3. Tag your image  
=======
# 3. Tag your image
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
docker tag mlops/retail-api:latest 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# 4. Push image to ECR
docker push 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
```

{{% notice info %}}
**Info:** Trên Windows/PowerShell, ưu tiên dùng `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <registry>` trong CI để tránh các lệnh đã bị deprecated. Token ECR thường hết hạn sau ~12 giờ; xác thực lại cho các phiên chạy dài.
{{% /notice %}}

<<<<<<< HEAD

=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
### 2.2. Verify ECR Push Success

**Kiểm tra trong AWS Console:**

1. **Navigate to ECR Console:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

{{% notice tip %}}
**Tip:** Giảm kích thước image bằng multi-stage builds và base images nhẹ (ví dụ `python:3.9-slim` hoặc distroless). Image nhỏ giúp đẩy/kéo nhanh hơn và giảm chi phí lưu trữ/truyền tải.
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
**Warning (Local):** Khi chạy image trên máy local, tránh mount secrets hoặc AWS credentials vào container. Dùng biến môi trường chỉ cho giá trị không nhạy cảm và ưu tiên IAM roles cho môi trường production.
{{% /notice %}}

- Local container test for retail-api :

![](/images/06-ecr-registry/11.png)

**Hoàn thành!** 🎉

ECR registry đã được thiết lập và tích hợp với EKS cluster `mlops-retail-cluster`. Docker image của retail API đã sẵn sàng để deploy trên Kubernetes trong Task 10.

## Kết quả Task 6

✅ **ECR Repository** - mlops/retail-api repository  
✅ **Container Image** - FastAPI prediction service  
<<<<<<< HEAD
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month  
=======
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

{{% notice success %}}
**🎯 Task 6 Complete - ECR Registry + API Containerization!**

**✅ ECR Setup**: Repository với lifecycle policies & image scanning  
**✅ Dockerfile**: Multi-stage build, non-root user, health checks  
**✅ Build & Push**: Local build → ECR push workflow  
**✅ Testing**: Container verification & API validation  
**✅ Ready**: Sẵn sàng cho EKS deployment trong Task 7  
{{% /notice %}}

{{% notice info %}}
**Info (Quét lỗ hổng):** Quét image cơ bản miễn phí; quét nâng cao (Inspector) có thể phát sinh phí theo image/tháng. Cân nhắc chỉ quét các tag production hoặc tích hợp quét vào CI với điều kiện để kiểm soát chi phí.
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 7**: EKS cluster deployment với ECR integration
- **Task 8**: Deploy API container lên EKS với ALB
- **Task 9**: Load balancing và scaling configuration

{{% /notice %}}

{{% notice info %}}
**Kết quả benchmark (Production):**

- **Kích thước image**: FastAPI ~500MB (đã tối ưu multi-stage)
- **Thời gian build**: ~3-5 phút (với cache)
- **Chi phí lưu trữ**: ~$0.15/tháng (tổng ~1.5GB)
- **Bảo mật**: Chạy non-root, đã quét lỗ hổng
- **Khả dụng**: Multi-tag strategy (latest, commit, branch)
- **CI/CD**: Tự động trên mỗi commit
<<<<<<< HEAD
{{% /notice %}}
=======
  {{% /notice %}}
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

## 3. Clean Up Resources (AWS CLI)

### 3.1. Xóa Images từ ECR Repository

```bash
# Liệt kê images trong repository
aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1 --query 'imageDetails[*].[imageDigest,imageTags[0],imagePushedAt]' --output table

# Xóa specific image tag
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids imageTag=latest \
  --region ap-southeast-1

# Xóa tất cả images trong repository
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids "$(aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1 --query 'imageDetails[*].{imageDigest:imageDigest}' --output json)" \
  --region ap-southeast-1
```

### 3.2. Xóa ECR Repositories

```bash
# Xóa repository (phải trống trước)
aws ecr delete-repository --repository-name mlops/retail-api --region ap-southeast-1 --force

# Verify repository đã bị xóa
aws ecr describe-repositories --region ap-southeast-1 --query 'repositories[?repositoryName==`mlops/retail-api`]'
```

### 3.3. Xóa Lifecycle Policies

```bash
# Xóa lifecycle policy (tự động xóa khi xóa repository)
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

<<<<<<< HEAD
| Storage Type | Giá (USD/GB/tháng) | Ghi chú |
|--------------|-------------------|---------|
| **ECR Storage** | $0.10 | Compressed image size |
| **Free Tier** | 500MB free | First 12 months |
| **Data Transfer IN** | Free | Push images to ECR |
| **Data Transfer OUT** | $0.12/GB | Pull từ Internet |
| **Data Transfer VPC** | Free | Pull qua VPC Endpoints |

### 4.2. Chi phí Image Scanning

| Scan Type | Giá (USD) | Ghi chú |
|-----------|-----------|---------|
| **Basic Scanning** | Free | CVE database scanning |
| **Enhanced Scanning** | $0.09/image/month | Inspector integration |
| **OS Package Scanning** | Free | Basic vulnerability detection |
| **Language Package Scanning** | $0.09/image/month | Enhanced scanning only |
=======
| Storage Type          | Giá (USD/GB/tháng) | Ghi chú                |
| --------------------- | ------------------ | ---------------------- |
| **ECR Storage**       | $0.10              | Compressed image size  |
| **Free Tier**         | 500MB free         | First 12 months        |
| **Data Transfer IN**  | Free               | Push images to ECR     |
| **Data Transfer OUT** | $0.12/GB           | Pull từ Internet       |
| **Data Transfer VPC** | Free               | Pull qua VPC Endpoints |

### 4.2. Chi phí Image Scanning

| Scan Type                     | Giá (USD)         | Ghi chú                       |
| ----------------------------- | ----------------- | ----------------------------- |
| **Basic Scanning**            | Free              | CVE database scanning         |
| **Enhanced Scanning**         | $0.09/image/month | Inspector integration         |
| **OS Package Scanning**       | Free              | Basic vulnerability detection |
| **Language Package Scanning** | $0.09/image/month | Enhanced scanning only        |
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 4.3. Ước tính chi phí cho Task 6

**Container Images:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- FastAPI image: ~500MB (compressed)
- Total storage: ~0.5GB

**Monthly Costs:**

<<<<<<< HEAD
| Component | Size | Price | Monthly Cost |
|-----------|------|-------|--------------|
| **ECR Storage** | 0.5GB | $0.10/GB | $0.05 |
| **Basic Scanning** | 1 image | Free | $0.00 |
| **VPC Endpoint Transfer** | ~1GB/month | Free | $0.00 |
| **Total** | | | **$0.05** |
=======
| Component                 | Size       | Price    | Monthly Cost |
| ------------------------- | ---------- | -------- | ------------ |
| **ECR Storage**           | 0.5GB      | $0.10/GB | $0.05        |
| **Basic Scanning**        | 1 image    | Free     | $0.00        |
| **VPC Endpoint Transfer** | ~1GB/month | Free     | $0.00        |
| **Total**                 |            |          | **$0.05**    |
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 4.4. Cost Comparison với Alternatives

**ECR vs Docker Hub:**

<<<<<<< HEAD
| Feature | ECR | Docker Hub | Winner |
|---------|-----|------------|--------|
| **Storage (500MB)** | $0.05/month | Free (public) | Docker Hub |
| **Private repos** | ✅ Native | $5/month | **ECR** |
| **AWS Integration** | ✅ Native | Manual setup | **ECR** |
| **VPC Endpoints** | ✅ Free transfer | ❌ Internet only | **ECR** |
| **IAM Integration** | ✅ Native | ❌ Token-based | **ECR** |
| **Vulnerability Scanning** | ✅ Built-in | ❌ Extra cost | **ECR** |
=======
| Feature                    | ECR              | Docker Hub       | Winner     |
| -------------------------- | ---------------- | ---------------- | ---------- |
| **Storage (500MB)**        | $0.05/month      | Free (public)    | Docker Hub |
| **Private repos**          | ✅ Native        | $5/month         | **ECR**    |
| **AWS Integration**        | ✅ Native        | Manual setup     | **ECR**    |
| **VPC Endpoints**          | ✅ Free transfer | ❌ Internet only | **ECR**    |
| **IAM Integration**        | ✅ Native        | ❌ Token-based   | **ECR**    |
| **Vulnerability Scanning** | ✅ Built-in      | ❌ Extra cost    | **ECR**    |
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 4.5. Data Transfer Costs

**ECR Pull Scenarios:**

<<<<<<< HEAD
| Pull Location | Cost | Use Case |
|---------------|------|----------|
| **Same Region (VPC)** | Free | EKS production |
| **Same Region (Internet)** | $0.12/GB | CI/CD outside AWS |
| **Cross Region** | $0.12/GB + transfer | Multi-region deployment |
| **Internet (outside AWS)** | $0.12/GB | Local development |
=======
| Pull Location              | Cost                | Use Case                |
| -------------------------- | ------------------- | ----------------------- |
| **Same Region (VPC)**      | Free                | EKS production          |
| **Same Region (Internet)** | $0.12/GB            | CI/CD outside AWS       |
| **Cross Region**           | $0.12/GB + transfer | Multi-region deployment |
| **Internet (outside AWS)** | $0.12/GB            | Local development       |
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 4.6. Lifecycle Policy Cost Savings

**Without Lifecycle Policies:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- 50 images × 500MB = 25GB storage
- Cost: 25GB × $0.10 = $2.50/month

**With Lifecycle Policies (Task 6):**
<<<<<<< HEAD
- Keep 10 production images = 5GB
- Keep 5 development images = 2.5GB  
=======

- Keep 10 production images = 5GB
- Keep 5 development images = 2.5GB
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- Total: 7.5GB × $0.10 = $0.75/month
- **Savings: $1.75/month (70%)**

### 4.7. Cost Optimization Tips

**Storage Optimization:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
```bash
# Multi-stage builds giảm image size
FROM node:16 as builder
# ... build steps
FROM node:16-alpine as production  # Smaller base image
COPY --from=builder /app/dist ./dist
```

**Registry Management:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
```bash
# Automated cleanup with lifecycle policies
aws ecr put-lifecycle-policy \
  --repository-name mlops/retail-api \
  --lifecycle-policy-text file://lifecycle-policy.json
```

**Free Tier Usage:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- Sử dụng 500MB free tier cho development
- Production images trong repositories riêng biệt
- VPC Endpoints để tránh data transfer charges

{{% notice info %}}
**💰 Cost Summary cho Task 6:**
<<<<<<< HEAD
=======

>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- **Storage:** $0.05/month (500MB images)
- **Scanning:** Free (basic vulnerability detection)
- **Data Transfer:** Free (VPC Endpoints to EKS)
- **Total:** **$0.05/month** (vs $5/month Docker Hub private)
- **Savings:** $4.95/month với ECR + lifecycle policies
<<<<<<< HEAD
{{% /notice %}}
=======
  {{% /notice %}}
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

{{% notice tip %}}
**Mẹo thành công:** Trước khi xóa repos/images để dọn dẹp, hãy snapshot deployment manifests và tham chiếu CI nếu cần lưu trữ. Ưu tiên sử dụng lifecycle policies để tự động quản lý retention thay vì xóa thủ công, tránh mất mát dữ liệu.
{{% /notice %}}

## 🎬 Video thực hiện Task 6

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

<<<<<<< HEAD
**Next Step**: [Task 7: EKS Cluster Setup](../7-eks-cluster/) 
=======
**Next Step**: [Task 7: EKS Cluster Setup](../7-eks-cluster/)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
