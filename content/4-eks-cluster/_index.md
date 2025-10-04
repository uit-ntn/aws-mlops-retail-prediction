---
title: "EKS Control Plane Setup"
date: 2025-08-30T13:00:00+07:00
weight: 4
chapter: false
pre: "<b>4. </b>"
---

## 🎯 Mục tiêu

Triển khai Amazon EKS Cluster (control plane + managed node groups) trên hạ tầng mạng private/public đã thiết lập ở Task 2.

Đảm bảo cluster chạy ổn định, có thể scale, và sẵn sàng deploy inference API (Task 13).

Sử dụng VPC Endpoints thay cho NAT Gateway để tối ưu chi phí và tăng bảo mật.

## 📥 Input

- **Terraform module** định nghĩa EKS (`aws/infra/eks.tf`)
- **Outputs từ Task 2** (VPC ID, subnet IDs, SG IDs, VPC Endpoints)
- **IAM Roles từ Task 3** (IRSA, Node IAM Role)

## 📌 Các bước chính

1. **Provision EKS Control Plane**
   - Tạo EKS cluster bằng Terraform, liên kết với VPC/subnets (private subnet cho worker node)
   - Enable logging: API, audit, scheduler, controller

2. **Tạo Managed Node Group**
   - EC2 instances (On-Demand hoặc Spot)
   - Cấu hình instance type theo workload (CPU/GPU)
   - Auto-scaling: min/max/desired size, trải đều trên ≥2 AZ
   - Gắn IAM Role cho phép node truy cập ECR, S3, CloudWatch qua VPC Endpoints

3. **Kết nối Cluster**
   - Update kubeconfig: `aws eks update-kubeconfig`
   - Kiểm tra node readiness: `kubectl get nodes`

4. **Cài đặt Core Add-ons**
   - VPC CNI plugin, CoreDNS, kube-proxy
   - Metrics Server (cần cho HPA)
   - Cluster Autoscaler (optional, scale node theo pod demand)

5. **IRSA Setup (IAM Roles for Service Accounts)**
   - Tạo OIDC Identity Provider cho EKS cluster
   - Setup IRSA roles cho S3 và CloudWatch access
   - Configure Service Accounts với proper annotations
   - Test secure pod authentication (no hardcoded credentials)

6. **Integration với VPC Endpoints từ Task 2**
   - **Reference existing VPC Endpoints** đã tạo ở Task 2 (Console) - không tạo mới
   - **ECR API & DKR**: pods pull container images qua VPC Endpoints
   - **S3 Gateway**: nodes/pods access ML data qua Gateway Endpoint (FREE)
   - **CloudWatch Logs**: logging và metrics qua Interface Endpoint

## ✅ Deliverables

- **EKS cluster ACTIVE** với node groups trải trên nhiều AZ
- **IRSA configured** với OIDC provider và Service Account authentication
- **Node có thể pull image** từ ECR qua VPC Endpoint (không cần NAT Gateway)
- **Pods có thể access S3/CloudWatch** qua IRSA (no hardcoded credentials)
- **Logs/metrics** từ pod được gửi lên CloudWatch thành công
- **kubeconfig usable** cho CI/CD pipeline

## 📊 Acceptance Criteria

- ✅ `kubectl get nodes` → tất cả node trạng thái Ready
- ✅ Add-ons (CNI, CoreDNS, kube-proxy, metrics-server) chạy ổn định
- ✅ IRSA OIDC provider created và linked với EKS cluster
- ✅ Pod mẫu deploy lên EKS pull image từ ECR thành công
- ✅ Pod có thể access S3 qua IRSA Service Account
- ✅ CloudWatch hiển thị log và metrics từ EKS pod thông qua VPC Endpoint

## ⚠️ Gotchas

- **Nếu thiếu metrics-server**, HPA sẽ không hoạt động
- **Nếu thiếu ECR VPC Endpoint** (api + dkr) → pod không pull được image
- **Nếu thiếu quyền IAM** cho Node Role → không đọc/ghi S3/CloudWatch
- **Spot instances** có thể bị thu hồi → nên cấu hình mixed instance type hoặc kết hợp On-Demand
- **Do không còn NAT GW** → pod trong private subnet không thể gọi API ngoài AWS (VD: GitHub, PyPI). Nếu cần, phải bổ sung NAT Instance hoặc cho chạy trong public subnet

## Kiến trúc EKS Control Plane

### EKS Architecture Overview

```
EKS Control Plane Architecture
├── AWS Managed Control Plane (Multi-AZ)
│   ├── API Server (Load Balanced)
│   ├── etcd (Distributed Storage)
│   ├── Scheduler (Pod Placement)
│   └── Controller Manager (Resource Management)
├── VPC Integration
│   ├── Private Subnets: 10.0.101.0/24, 10.0.102.0/24
│   ├── Public Subnets: 10.0.1.0/24, 10.0.2.0/24 (for Load Balancers)
│   └── Security Groups: EKS Control Plane SG
├── IAM Integration
│   ├── EKS Cluster Service Role
│   ├── IRSA (IAM Roles for Service Accounts)
│   └── User/Role Access Mapping
└── Networking
    ├── Cluster Endpoint: Public (with restrictions)
    ├── Private Endpoint: Yes (for node communication)
    └── Network Policy: VPC native
```

### Control Plane Benefits

- **🚀 High Availability**: AWS managed multi-AZ control plane
- **🔒 Security**: Private API endpoint với public access control
- **📊 Monitoring**: Native CloudWatch integration
- **🔄 Auto-scaling**: Control plane scales automatically
- **🛡️ Updates**: AWS managed Kubernetes version updates

{{% notice success %}}
**🎯 Architecture Decision:** Hybrid endpoint access (Public + Private)

**Production Benefits:**
- ✅ **Developer access** qua public endpoint với IP restrictions
- ✅ **Worker nodes** communicate qua private endpoint
- ✅ **Security controls** với fine-grained access policies
- ✅ **Cost optimization** với right-sized control plane
{{% /notice %}}

## 1. Alternative: AWS Console Implementation

### 1.1. EKS Cluster Creation via Console

1. **Navigate to EKS Console:**
   - Đăng nhập AWS Console
   - Navigate to EKS service
   - Chọn "Create cluster"

![Create EKS Cluster](../images/04-eks-cluster/01-create-eks-cluster.png)

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

![Cluster Basic Config](../images/04-eks-cluster/02-cluster-basic-config.png)

3. **Networking Configuration:**
   ```
   VPC: mlops-retail-forecast-dev-vpc
   Subnets: 
     - mlops-retail-forecast-dev-private-ap-southeast-1a
     - mlops-retail-forecast-dev-private-ap-southeast-1b
     - mlops-retail-forecast-dev-public-ap-southeast-1a
     - mlops-retail-forecast-dev-public-ap-southeast-1b
   Security groups: mlops-retail-forecast-dev-eks-control-plane-sg
   ```

![Networking Config](../images/04-eks-cluster/03-networking-config.png)

4. **Cluster Endpoint Access:**
   ```
   Endpoint private access: Enabled
   Endpoint public access: Enabled
   Public access source: Specific CIDR blocks (your IP)
   ```

![Endpoint Access](../images/04-eks-cluster/04-endpoint-access.png)

5. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

![Logging Config](../images/04-eks-cluster/05-logging-config.png)

6. **Encryption Configuration:**
   ```
   Secrets encryption: Enabled
   KMS key: Create new key or use existing
   Key alias: alias/mlops-retail-forecast-dev-eks
   ```

![Encryption Config](../images/04-eks-cluster/06-encryption-config.png)

### 1.2. Add-ons Installation via Console

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

![Addons Overview](../images/04-eks-cluster/07-addons-overview.png)

2. **Install Essential Add-ons:**
   
   **CoreDNS:**
   ```
   Name: coredns
   Version: v1.10.1-eksbuild.5
   Configuration: Default
   ```

   **kube-proxy:**
   ```
   Name: kube-proxy
   Version: v1.28.2-eksbuild.2
   Configuration: Default
   ```

   **VPC CNI:**
   ```
   Name: vpc-cni
   Version: v1.15.4-eksbuild.1
   Configuration: Default
   ```

   **EBS CSI Driver:**
   ```
   Name: aws-ebs-csi-driver
   Version: v1.24.1-eksbuild.1
   Service account role: Create new IAM role
   ```

![Install Addons](../images/04-eks-cluster/08-install-addons.png)

3. **AWS Load Balancer Controller:**
   ```
   Name: aws-load-balancer-controller
   Version: v2.6.3-eksbuild.1
   Service account role: mlops-retail-forecast-dev-alb-controller
   ```

![ALB Controller Addon](../images/04-eks-cluster/09-alb-controller-addon.png)

### 1.3. Cluster Verification via Console

1. **Cluster Overview:**
   - Check cluster status: "Active"
   - Verify Kubernetes version: 1.28
   - Confirm endpoint access configuration

![Cluster Overview](../images/04-eks-cluster/10-cluster-overview.png)

2. **Add-ons Status:**
   ```
   ✅ coredns: Active
   ✅ kube-proxy: Active
   ✅ vpc-cni: Active
   ✅ aws-ebs-csi-driver: Active
   ✅ aws-load-balancer-controller: Active
   ```

![Addons Status](../images/04-eks-cluster/11-addons-status.png)

3. **CloudWatch Logs:**
   - Navigate to CloudWatch → Log groups
   - Verify log group: `/aws/eks/mlops-retail-forecast-dev-cluster/cluster`
   - Check log streams cho control plane components

![CloudWatch Logs](../images/04-eks-cluster/12-cloudwatch-logs.png)

{{% notice success %}}
**🎯 Console Implementation Complete!**

EKS cluster đã được tạo thành công với:
- ✅ Multi-AZ control plane
- ✅ Proper IAM integration
- ✅ Essential add-ons installed
- ✅ CloudWatch logging enabled
- ✅ Secrets encryption configured
{{% /notice %}}

{{% notice info %}}
**💡 Console vs Terraform:**

**Console Advantages:**
- ✅ Visual cluster creation wizard
- ✅ Real-time status updates
- ✅ Easy add-ons management
- ✅ Integrated validation

**Terraform Advantages:**
- ✅ Infrastructure as Code
- ✅ Version control
- ✅ Reproducible deployments
- ✅ Automation-ready

Khuyến nghị: Console cho learning, Terraform cho production.
{{% /notice %}}

## 2. Terraform cho Advanced EKS Configuration

{{% notice info %}}
**💡 Khi nào cần Terraform cho EKS:**
- ✅ **Integration phức tạp** với existing VPC/IAM từ Task 2-3  
- ✅ **Automated CI/CD deployment** với consistent configuration  
- ✅ **Advanced security** như KMS encryption, fine-grained IAM  
- ✅ **Production workloads** cần reproducible infrastructure  

**Console đủ cho:** Basic EKS cluster tạo một lần, learning, testing
{{% /notice %}}

### 2.0. Terraform Code Purpose & Expected Results

{{% notice success %}}
**🎯 Mục đích của Terraform code trong Task 4:**

**Input:** 
- VPC infrastructure từ Task 2 (VPC, subnets, Security Groups, VPC Endpoints)
- IAM roles từ Task 3 (EKS cluster role, node group role)

**Terraform sẽ làm gì:**
1. **Reference existing infrastructure** từ Task 2-3 (không tạo mới)
2. **Create EKS cluster** với proper integration
3. **Install essential add-ons** automatically
4. **Configure security** với KMS encryption
5. **Enable logging** cho production monitoring

**Kết quả sau khi chạy:**
- ✅ EKS cluster ACTIVE và ready to use
- ✅ kubectl có thể connect được
- ✅ Pods có thể pull images từ ECR qua VPC Endpoints
- ✅ Logs được gửi lên CloudWatch
- ✅ Add-ons (CoreDNS, kube-proxy, VPC CNI) hoạt động
- ✅ Cost optimized: sử dụng VPC Endpoints thay vì NAT Gateway
{{% /notice %}}

### 2.1. EKS Cluster với VPC Integration

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Tìm VPC và subnets** đã tạo ở Task 2
2. **Tìm IAM roles** đã tạo ở Task 3  
3. **Tạo EKS cluster** kết nối với infrastructure có sẵn
4. **Enable logging và encryption** cho production

**Kết quả:** EKS cluster hoạt động trong VPC đã có, sử dụng VPC Endpoints để tiết kiệm chi phí
{{% /notice %}}

**File: `aws/infra/eks-cluster.tf`**

```hcl
# BƯỚC 1: Tìm VPC infrastructure từ Task 2 (không tạo mới)
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["${var.project_name}-${var.environment}-vpc"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["private-subnet"]
  }
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["public-subnet"]
  }
}

# BƯỚC 2: Tạo EKS cluster với integration từ Task 2-3
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}-cluster"
  role_arn = data.aws_iam_role.eks_cluster.arn  # IAM role từ Task 3
  version  = var.kubernetes_version

  # Networking: sử dụng VPC từ Task 2
  vpc_config {
    subnet_ids              = concat(data.aws_subnets.private.ids, data.aws_subnets.public.ids)
    endpoint_private_access = true   # Worker nodes connect privately
    endpoint_public_access  = true   # Developers can access from outside
    public_access_cidrs     = var.cluster_endpoint_public_access_cidrs
    security_group_ids      = [data.aws_security_group.eks_control_plane.id]
  }

  # Production logging (gửi lên CloudWatch)
  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  # Encryption: bảo mật secrets với KMS
  encryption_config {
    provider {
      key_arn = data.aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  # Dependencies: đảm bảo infrastructure từ Task 2-3 đã sẵn sàng
  depends_on = [
    data.aws_vpc_endpoint.s3,      # VPC Endpoints từ Task 2 (Console)
    data.aws_vpc_endpoint.ecr_api,
    data.aws_vpc_endpoint.ecr_dkr,
    data.aws_vpc_endpoint.logs,
    data.aws_iam_role.eks_cluster   # IAM từ Task 3
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cluster"
    Type = "eks-cluster"
    CreatedBy = "terraform"
  })
}

# BƯỚC 3: Reference resources từ previous tasks (không tạo mới)
data "aws_iam_role" "eks_cluster" {
  name = "${var.project_name}-${var.environment}-eks-cluster-role"  # Từ Task 3
}

data "aws_security_group" "eks_control_plane" {
  filter {
    name   = "tag:Name"
    values = ["${var.project_name}-${var.environment}-eks-control-plane-sg"]  # Từ Task 2
  }
}

data "aws_kms_key" "eks" {
  key_id = "alias/${var.project_name}-${var.environment}-eks"  # Từ Task 3
}
```

## 3. IRSA Setup (IAM Roles for Service Accounts)

{{% notice info %}}
**💡 IRSA Setup trong Task 4:**
Sau khi EKS cluster đã được tạo, chúng ta có thể setup IRSA để pods có thể access AWS services securely mà không cần hardcoded credentials.
{{% /notice %}}

### 3.1. IRSA Foundation - OIDC Provider

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Tìm EKS cluster** vừa tạo để lấy OIDC issuer URL
2. **Get SSL certificate** từ EKS OIDC endpoint cho security validation
3. **Create OIDC Identity Provider** trong AWS IAM để trust EKS cluster
4. **Enable IRSA authentication** cho Kubernetes Service Accounts

**Kết quả:** AWS IAM có thể trust và authenticate Kubernetes Service Accounts
{{% /notice %}}

**File: `aws/infra/eks-irsa.tf`**

```hcl
# BƯỚC 1: Get OIDC issuer certificate từ EKS cluster vừa tạo
data "tls_certificate" "eks_oidc" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer  # EKS OIDC endpoint
}

# BƯỚC 2: Create OIDC Identity Provider trong AWS IAM
resource "aws_iam_openid_connect_provider" "eks_oidc" {
  client_id_list  = ["sts.amazonaws.com"]  # AWS STS service
  thumbprint_list = [data.tls_certificate.eks_oidc.certificates[0].sha1_fingerprint]  # SSL cert validation
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer  # EKS OIDC URL

  # Purpose: Cho phép AWS IAM trust Kubernetes Service Accounts
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-oidc"
    Type = "oidc-provider"
    Purpose = "irsa-authentication"
  })
}
```

### 3.2. IRSA Role for ML Workloads (S3 Access)

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Create IAM role** chỉ có thể được assume bởi specific Kubernetes Service Account
2. **Setup trust policy** với exact namespace và service account matching
3. **Grant S3 permissions** chỉ cho ML data buckets (least privilege)
4. **Enable secure access** từ pods mà không cần hardcoded AWS credentials

**Kết quả:** Pods với Service Account `s3-access-sa` có thể access S3 securely
{{% /notice %}}

```hcl
# BƯỚC 1: Create IRSA Role cho ML workloads access S3
resource "aws_iam_role" "irsa_s3_access" {
  name = "${var.project_name}-${var.environment}-irsa-s3-access"

  # Trust Policy: CHỈ specific Service Account có thể assume role này
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks_oidc.arn  # OIDC provider từ bước trước
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            # QUAN TRỌNG: Exact match namespace và service account name
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:sub" = "system:serviceaccount:mlops-retail-forecast:s3-access-sa"
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-irsa-s3-access"
    Type = "iam-role"
    Service = "irsa-s3"
  })
}

# BƯỚC 2: S3 access policy - LEAST PRIVILEGE cho ML buckets only
resource "aws_iam_role_policy" "irsa_s3_policy" {
  name = "${var.project_name}-${var.environment}-irsa-s3-policy"
  role = aws_iam_role.irsa_s3_access.id

  # Permissions: Chỉ access ML data buckets, không phải tất cả S3
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",     # Read files
          "s3:PutObject",     # Upload files  
          "s3:DeleteObject",  # Delete files
          "s3:ListBucket"     # List bucket contents
        ]
        Resource = [
          # CHỈ access specific ML buckets
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-data",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-data/*",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts/*"
        ]
      }
    ]
  })
}
```

### 3.3. IRSA Role for CloudWatch Monitoring

```hcl
# IRSA Role for CloudWatch monitoring
resource "aws_iam_role" "irsa_cloudwatch_access" {
  name = "${var.project_name}-${var.environment}-irsa-cloudwatch"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks_oidc.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:sub" = "system:serviceaccount:mlops-retail-forecast:cloudwatch-sa"
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-irsa-cloudwatch"
    Type = "iam-role"
    Service = "irsa-cloudwatch"
  })
}

# CloudWatch permissions for IRSA
resource "aws_iam_role_policy_attachment" "irsa_cloudwatch_policy" {
  role       = aws_iam_role.irsa_cloudwatch_access.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# Custom CloudWatch metrics policy
resource "aws_iam_role_policy" "irsa_cloudwatch_custom" {
  name = "${var.project_name}-${var.environment}-irsa-cloudwatch-custom"
  role = aws_iam_role.irsa_cloudwatch_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData",
          "cloudwatch:GetMetricStatistics",
          "cloudwatch:ListMetrics"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudwatch:namespace" = [
              "MLOps/RetailForecast",
              "AWS/EKS",
              "ContainerInsights"
            ]
          }
        }
      }
    ]
  })
}
```

### 3.4. Kubernetes Service Accounts với IRSA Annotations

**File: `aws/k8s/service-accounts.yaml`**

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: mlops-retail-forecast
  labels:
    name: mlops-retail-forecast
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-s3-access
  labels:
    app.kubernetes.io/name: s3-access-service-account
    app.kubernetes.io/component: rbac
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudwatch-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-cloudwatch
  labels:
    app.kubernetes.io/name: cloudwatch-service-account
    app.kubernetes.io/component: monitoring
```

### 3.5. Test Pod với IRSA Authentication

**File: `aws/k8s/test-pod-irsa.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-irsa-s3-access
  namespace: mlops-retail-forecast
spec:
  serviceAccountName: s3-access-sa  # IRSA Service Account
  containers:
  - name: test-s3-access
    image: amazon/aws-cli:latest
    command: ["/bin/bash"]
    args: ["-c", "aws s3 ls && sleep 3600"]
    env:
    - name: AWS_DEFAULT_REGION
      value: "ap-southeast-1"
  restartPolicy: Never
```

## 4. Essential Variables cho Terraform

**File: `aws/infra/variables.tf` (chỉ cần thêm essential vars):**

```hcl
# Core EKS configuration
variable "kubernetes_version" {
  description = "Kubernetes version for EKS cluster"
  type        = string
  default     = "1.28"
}

variable "cluster_endpoint_public_access_cidrs" {
  description = "CIDR blocks for public API access (restrict in production)"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}
```

### 2.3. Production Terraform Configuration

**File: `aws/terraform.tfvars`:**

```hcl
# Production EKS configuration
kubernetes_version = "1.28"
cluster_endpoint_public_access_cidrs = [
  "203.0.113.0/24",  # Your office IP range
  "198.51.100.0/24"  # VPN IP range
]
```

## 3. Add-ons Management (Choose Console or Terraform)

{{% notice tip %}}
**💡 Add-ons Installation Strategy:**
- ✅ **Console**: Quick setup, one-time installation, visual management
- ✅ **Terraform**: Automated updates, version control, CI/CD integration

**Recommendation**: Console cho initial setup, Terraform cho production automation
{{% /notice %}}

### 3.1. Essential Add-ons via Terraform (cho automation)

{{% notice tip %}}
**🔍 Add-ons code này làm gì:**
1. **Định nghĩa essential add-ons** cần thiết cho EKS hoạt động
2. **Tự động install** các add-ons sau khi EKS cluster ready
3. **Manage versions** và conflict resolution
4. **Link với IAM roles** từ Task 3 cho permissions

**Kết quả:** EKS cluster có đầy đủ add-ons để pods có thể chạy, networking hoạt động, storage available
{{% /notice %}}

**File: `aws/infra/eks-addons.tf`**

```hcl
# BƯỚC 1: Định nghĩa essential add-ons và versions
locals {
  essential_addons = {
    # DNS resolution trong cluster
  coredns = {
      addon_version               = "v1.10.1-eksbuild.5"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
  }
    # Network proxy cho pods
  kube-proxy = {
      addon_version               = "v1.28.2-eksbuild.2"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
  }
    # VPC networking cho pods
  vpc-cni = {
      addon_version               = "v1.15.4-eksbuild.1"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
  }
    # EBS storage cho persistent volumes
  aws-ebs-csi-driver = {
      addon_version               = "v1.24.1-eksbuild.1"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
      service_account_role_arn    = data.aws_iam_role.ebs_csi_driver.arn  # IAM từ Task 3
    }
  }
}

# BƯỚC 2: Tự động install tất cả add-ons
resource "aws_eks_addon" "essential" {
  for_each = local.essential_addons

  cluster_name             = aws_eks_cluster.main.name
  addon_name               = each.key
  addon_version            = each.value.addon_version
  resolve_conflicts        = each.value.resolve_conflicts_on_update
  service_account_role_arn = lookup(each.value, "service_account_role_arn", null)

  depends_on = [aws_eks_cluster.main]  # Chờ cluster ready trước

  tags = merge(var.common_tags, {
    Name      = "${var.project_name}-${var.environment}-${each.key}"
    Type      = "eks-addon"
    CreatedBy = "terraform"
  })
}

# BƯỚC 3: Reference IAM role từ Task 3 cho EBS CSI Driver
data "aws_iam_role" "ebs_csi_driver" {
  name = "${var.project_name}-${var.environment}-ebs-csi-driver-role"  # Từ Task 3
}
```

### 3.2. Add-ons Variables

**File: `aws/infra/variables.tf` (minimal addon config):**

```hcl
# EKS Add-ons override versions (optional)
variable "addon_versions" {
  description = "Override default addon versions"
  type        = map(string)
  default     = {}  # Use default versions from locals
}
```

{{% notice warning %}}
**📝 Note về Node Groups:**

EKS Node Groups được cover chi tiết trong **Task 5**, bao gồm:
- Basic node group creation qua Console (đủ cho most cases)
- Advanced multi-node group strategies với Terraform
- Mixed instance types, auto-scaling, cost optimization

Task 4 focus vào EKS Control Plane và integration với existing infrastructure.
{{% /notice %}}

## 4. Integration với VPC Endpoints từ Task 2

{{% notice info %}}
**💡 VPC Endpoints đã sẵn sàng:**
VPC Endpoints đã được tạo qua Console ở Task 2. Task 4 chỉ cần **reference** chúng, không tạo mới.
{{% /notice %}}

### 4.1. VPC Endpoints Benefits cho EKS

**EKS sử dụng VPC Endpoints đã được tạo ở Task 2 (Console)** để giảm chi phí và tăng bảo mật:

```
✅ VPC Endpoints từ Task 2 (Console - đã có sẵn):
├── S3 Gateway Endpoint: FREE
├── ECR API Interface: $7.2/month
├── ECR DKR Interface: $7.2/month
├── CloudWatch Logs: $7.2/month
└── Total: $21.6/month (vs $71/month NAT Gateway)

💰 Cost Savings: 70% reduction ($49.4/month saved)
```

### 4.2. Reference VPC Endpoints từ Task 2 (Console)

**EKS sử dụng VPC Endpoints đã được tạo ở Task 2 qua Console** - không cần tạo lại:

```hcl
# Data sources to reference VPC Endpoints from Task 2 (Console-created)
data "aws_vpc_endpoint" "s3" {
  vpc_id       = data.aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
}

data "aws_vpc_endpoint" "ecr_api" {
  vpc_id       = data.aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.ecr.api"
}

data "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id       = data.aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.ecr.dkr"
}

data "aws_vpc_endpoint" "logs" {
  vpc_id       = data.aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.logs"
}
```

**EKS Cluster chỉ cần reference VPC Endpoints từ Task 2:**

```hcl
# EKS Cluster sử dụng VPC Endpoints đã có từ Task 2
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}-cluster"
  role_arn = data.aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = concat(data.aws_subnets.private.ids, data.aws_subnets.public.ids)
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.cluster_endpoint_public_access_cidrs
    security_group_ids      = [data.aws_security_group.eks_control_plane.id]
  }

  # Production logging
  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  # Encryption với KMS từ Task 3
  encryption_config {
    provider {
      key_arn = data.aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  # Dependencies - VPC Endpoints từ Task 2 (Console) đã tồn tại
  depends_on = [
    data.aws_vpc_endpoint.s3,      # Task 2 VPC Endpoints (Console-created)
    data.aws_vpc_endpoint.ecr_api,
    data.aws_vpc_endpoint.ecr_dkr,
    data.aws_vpc_endpoint.logs,
    data.aws_iam_role.eks_cluster   # Task 3 IAM
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cluster"
    Type = "eks-cluster"
    CreatedBy = "terraform"
  })
}
```

### 4.3. Verification VPC Endpoints Integration

**Verify VPC Endpoints từ Task 2 (Console) đã sẵn sàng cho EKS:**

```bash
# Verify VPC Endpoints từ Task 2 (Console) đã tồn tại và available
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'VpcEndpoints[*].{Service:ServiceName,State:State,Type:VpcEndpointType}'

# Expected output (VPC Endpoints từ Task 2 - Console):
# [
#   {
#     "Service": "com.amazonaws.ap-southeast-1.s3",
#     "State": "available",
#     "Type": "Gateway"
#   },
#   {
#     "Service": "com.amazonaws.ap-southeast-1.ecr.api", 
#     "State": "available",
#     "Type": "Interface"
#   },
#   {
#     "Service": "com.amazonaws.ap-southeast-1.ecr.dkr",
#     "State": "available", 
#     "Type": "Interface"
#   },
#   {
#     "Service": "com.amazonaws.ap-southeast-1.logs",
#     "State": "available", 
#     "Type": "Interface"
#   }
# ]
```

**Test ECR access từ EKS pods:**

```bash
# Deploy test pod để verify ECR access qua VPC Endpoints
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-ecr-access
  namespace: default
spec:
  containers:
  - name: test
    image: public.ecr.aws/amazonlinux/amazonlinux:latest
    command: ["/bin/bash"]
    args: ["-c", "while true; do sleep 30; done;"]
  restartPolicy: Always
EOF

# Check pod có thể pull image thành công qua VPC Endpoints
kubectl get pods test-ecr-access
kubectl describe pod test-ecr-access

# Test DNS resolution cho ECR endpoints (should resolve to private IPs)
kubectl exec -it test-ecr-access -- nslookup ${AWS_ACCOUNT_ID}.dkr.ecr.ap-southeast-1.amazonaws.com

# Cleanup
kubectl delete pod test-ecr-access
```

## 5. Terraform Deployment

### 5.1. Step-by-Step Terraform Deployment

{{% notice success %}}
**🚀 Deployment Process:**

**Bước 1:** Terraform tìm infrastructure từ Task 2-3  
**Bước 2:** Tạo EKS cluster với proper integration  
**Bước 3:** Install essential add-ons automatically  
**Bước 4:** Setup IRSA với OIDC provider và roles  
**Bước 5:** Configure kubectl access  
**Bước 6:** Verify cluster, add-ons và IRSA hoạt động  

**Time required:** ~20-25 phút
{{% /notice %}}

```bash
# BƯỚC 1: Navigate to infrastructure directory
cd aws/infra

# BƯỚC 2: Plan EKS cluster creation (xem Terraform sẽ làm gì)
terraform plan -target=aws_eks_cluster.main \
               -target=aws_eks_addon.essential \
               -target=aws_iam_openid_connect_provider.eks_oidc \
               -target=aws_iam_role.irsa_s3_access \
               -target=aws_iam_role.irsa_cloudwatch_access \
               -var-file="terraform.tfvars"

# BƯỚC 3: Apply EKS cluster, add-ons và IRSA setup
terraform apply -target=aws_eks_cluster.main \
                -target=aws_eks_addon.essential \
                -target=aws_iam_openid_connect_provider.eks_oidc \
                -target=aws_iam_role.irsa_s3_access \
                -target=aws_iam_role.irsa_cloudwatch_access \
                -var-file="terraform.tfvars"
```

**Expected Apply Output:**
```
Apply complete! Resources: 11 added, 0 changed, 0 destroyed.

Resources Created:
✅ aws_eks_cluster.main
✅ aws_eks_addon.essential["coredns"]
✅ aws_eks_addon.essential["kube-proxy"] 
✅ aws_eks_addon.essential["vpc-cni"]
✅ aws_eks_addon.essential["aws-ebs-csi-driver"]
✅ aws_iam_openid_connect_provider.eks_oidc (IRSA OIDC provider)
✅ aws_iam_role.irsa_s3_access (S3 access role cho ML workloads)
✅ aws_iam_role_policy.irsa_s3_policy (Least privilege S3 permissions)
✅ aws_iam_role.irsa_cloudwatch_access (CloudWatch monitoring role)
✅ aws_iam_role_policy_attachment.irsa_cloudwatch_policy (CloudWatch permissions)
✅ aws_iam_role_policy.irsa_cloudwatch_custom (Custom CloudWatch metrics)

Outputs:
cluster_id = "mlops-retail-forecast-dev-cluster"
cluster_arn = "arn:aws:eks:ap-southeast-1:123456789012:cluster/mlops-retail-forecast-dev-cluster"
cluster_endpoint = "https://A1B2C3D4E5F6G7H8I9J0.gr7.ap-southeast-1.eks.amazonaws.com"
cluster_version = "1.28"
irsa_s3_access_role_arn = "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-irsa-s3-access"
irsa_cloudwatch_role_arn = "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-irsa-cloudwatch"
```

### 5.2. Configure kubectl after Terraform

```bash
# Get cluster name from Terraform output
CLUSTER_NAME=$(terraform output -raw cluster_id)

# Update kubeconfig
aws eks update-kubeconfig --region ap-southeast-1 --name $CLUSTER_NAME

# Verify cluster access
kubectl get svc
```

## 6. Cluster Verification

### 6.1. Control Plane Health Check

```bash
# Check cluster status
kubectl get --raw='/readyz?verbose'

# Check API server health
kubectl get --raw='/healthz'

# View cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 6.2. Add-ons Verification

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check kube-proxy
kubectl get daemonset -n kube-system kube-proxy

# Check VPC CNI
kubectl get daemonset -n kube-system aws-node

# Check EBS CSI driver
kubectl get deployment -n kube-system ebs-csi-controller
```

### 6.3. RBAC and Permissions

```bash
# Check current user permissions
kubectl auth can-i "*" "*"

# List available API resources
kubectl api-resources

# Check cluster roles
kubectl get clusterroles | grep eks
```

### 6.4. IRSA Verification

```bash
# Deploy service accounts với IRSA annotations
kubectl apply -f aws/k8s/service-accounts.yaml

# Deploy test pod với IRSA authentication
kubectl apply -f aws/k8s/test-pod-irsa.yaml

# Verify pod có thể access S3 qua IRSA (no AWS credentials needed!)
kubectl exec -it test-irsa-s3-access -- aws s3 ls

# Check IRSA role annotations
kubectl get serviceaccount s3-access-sa -o yaml

# Verify OIDC provider
aws iam list-open-id-connect-providers

# List IRSA roles
aws iam list-roles --query 'Roles[?contains(RoleName, `irsa`)].{RoleName:RoleName,CreateDate:CreateDate}'

# Test CloudWatch access
kubectl exec -it test-irsa-s3-access -- aws cloudwatch list-metrics --namespace "MLOps/RetailForecast"

# Cleanup test pod
kubectl delete pod test-irsa-s3-access
```

## 7. Monitoring và Logging

### 7.1. CloudWatch Integration

```bash
# Check if cluster logging is enabled
aws eks describe-cluster \
  --name mlops-retail-forecast-dev-cluster \
  --query 'cluster.logging'

# View control plane logs in CloudWatch
aws logs describe-log-groups \
  --log-group-name-prefix /aws/eks/mlops-retail-forecast-dev-cluster
```

### 7.2. Setup CloudWatch Container Insights

**File: `aws/k8s/cloudwatch-insights.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: amazon-cloudwatch
  labels:
    name: amazon-cloudwatch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-cloudwatch
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      name: cloudwatch-agent
  template:
    metadata:
      labels:
        name: cloudwatch-agent
    spec:
      serviceAccountName: cloudwatch-agent
      containers:
      - name: cloudwatch-agent
        image: amazon/cloudwatch-agent:1.300026.2b251814
        env:
        - name: AWS_REGION
          value: ap-southeast-1
        - name: CLUSTER_NAME
          value: mlops-retail-forecast-dev-cluster
        volumeMounts:
        - name: cwagentconfig
          mountPath: /etc/cwagentconfig
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: dockersock
          mountPath: /var/run/docker.sock
          readOnly: true
        - name: varlibdocker
          mountPath: /var/lib/docker
          readOnly: true
      volumes:
      - name: cwagentconfig
        configMap:
          name: cwagentconfig
      - name: rootfs
        hostPath:
          path: /
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: varlibdocker
        hostPath:
          path: /var/lib/docker
```

## 8. Security Hardening

### 8.1. Network Security

```bash
# Verify security groups
aws ec2 describe-security-groups \
  --group-ids $(terraform output -raw eks_control_plane_security_group_id) \
  --query 'SecurityGroups[0].{GroupId:GroupId,IpPermissions:IpPermissions}'

# Check VPC configuration
aws eks describe-cluster \
  --name mlops-retail-forecast-dev-cluster \
  --query 'cluster.resourcesVpcConfig'
```

### 8.2. RBAC Configuration

**File: `aws/k8s/rbac.yaml`**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mlops-admin
  namespace: mlops-retail-forecast
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mlops-retail-forecast
  name: mlops-admin-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mlops-admin-binding
  namespace: mlops-retail-forecast
subjects:
- kind: ServiceAccount
  name: mlops-admin
  namespace: mlops-retail-forecast
roleRef:
  kind: Role
  name: mlops-admin-role
  apiGroup: rbac.authorization.k8s.io
```

## Outputs

**Thêm vào `aws/infra/outputs.tf`:**

```hcl
# EKS Cluster Outputs
output "cluster_id" {
  description = "EKS cluster ID"
  value       = aws_eks_cluster.main.id
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.main.arn
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_version" {
  description = "The Kubernetes version for the cluster"
  value       = aws_eks_cluster.main.version
}

output "cluster_platform_version" {
  description = "Platform version for the cluster"
  value       = aws_eks_cluster.main.platform_version
}

output "cluster_status" {
  description = "Status of the EKS cluster"
  value       = aws_eks_cluster.main.status
}

output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data required to communicate with the cluster"
  value       = aws_eks_cluster.main.certificate_authority[0].data
}

# OIDC Issuer
output "cluster_oidc_issuer_url" {
  description = "The URL on the EKS cluster for the OpenID Connect identity provider"
  value       = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

## 👉 Kết quả Task 4

Sau Task 4, bạn sẽ có EKS Cluster production-ready, chạy hoàn toàn trong private subnet và tích hợp với VPC Endpoints từ Task 2, tiết kiệm chi phí NAT Gateway và tăng mức độ bảo mật.

### ✅ Deliverables Completed

- **EKS Control Plane ACTIVE**: Managed Kubernetes cluster với multi-AZ high availability
- **IRSA Configured**: OIDC provider và Service Account authentication setup
- **Managed Node Groups**: EC2 instances trải đều trên ≥2 AZ với auto-scaling
- **VPC Endpoints Integration**: Sử dụng ECR, S3, CloudWatch endpoints từ Task 2
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **Secure Pod Access**: Pods có thể access S3/CloudWatch qua IRSA (no hardcoded credentials)
- **kubectl Access**: Local development environment configured và tested
- **Cost Optimization**: 70% giảm chi phí so với NAT Gateway approach

### Architecture Achieved

```
✅ EKS Cluster: mlops-retail-forecast-dev-cluster (Kubernetes 1.28)
✅ Control Plane: Multi-AZ managed service với full logging
✅ Node Groups: 2-4 nodes (t3.medium/large) trong private subnets
✅ IRSA: OIDC provider + S3/CloudWatch access roles
✅ VPC Endpoints: Sử dụng từ Task 2 (Console) - S3 (FREE) + ECR API/DKR + CloudWatch ($21.6/month)
✅ Security: Least privilege IAM roles + Security Groups + IRSA authentication
✅ Monitoring: CloudWatch integration với Container Insights
```

### Cost Summary

| Component | Monthly Cost | Savings vs NAT GW |
|-----------|--------------|-------------------|
| **EKS Control Plane** | $73.00 | - |
| **EC2 Nodes (2x t3.medium)** | ~$60.00 | - |
| **VPC Endpoints** | $21.60 | -$49.40 (70%) |
| **EBS Storage (100GB)** | ~$10.00 | - |
| **Total** | **~$164.60** | **$49.40 saved** |

### Verification Commands

```bash
# Check cluster status
kubectl get nodes
kubectl get pods -A
kubectl cluster-info

# Verify VPC endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)"

# Test ECR access
kubectl run test-pod --image=public.ecr.aws/amazonlinux/amazonlinux:latest --rm -it --restart=Never -- /bin/bash

# Check add-ons status
kubectl get deployments -n kube-system

# Test IRSA functionality
kubectl apply -f aws/k8s/service-accounts.yaml
kubectl apply -f aws/k8s/test-pod-irsa.yaml
kubectl exec -it test-irsa-s3-access -- aws s3 ls
```

{{% notice success %}}
**🎯 Ready for Next Tasks:**

EKS cluster foundation đã sẵn sàng để deploy:
- ✅ **Task 5**: EKS node groups scaling và optimization
- ✅ **Task 6**: ECR repository setup cho container images  
- ✅ **Task 7**: Build và push inference API container
- ✅ **Task 8**: S3 data storage integration
- ✅ **Task 13**: Deploy inference API lên EKS cluster
{{% /notice %}}

{{% notice warning %}}
**🔐 Security & Maintenance Reminders:**

- **Public Access**: Restrict `cluster_endpoint_public_access_cidrs` to your actual IP ranges in production
- **Logging**: All control plane logging enabled cho security audit
- **Updates**: Regularly update Kubernetes version và add-ons (quarterly)
- **RBAC**: Implement proper role-based access control cho team members
- **Monitoring**: Setup alerts cho node health, pod failures, và resource usage
- **Backup**: Consider EKS cluster backup strategy cho disaster recovery
{{% /notice %}}