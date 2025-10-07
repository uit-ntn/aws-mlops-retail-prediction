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

- **EKS cluster ACTIVE** với node groups (t2.micro FREE tier)
- **IRSA configured** với OIDC provider và Service Account authentication
- **Node có thể pull image** từ ECR qua VPC Endpoint (không cần NAT Gateway)
- **Pods có thể access S3/CloudWatch** qua IRSA (no hardcoded credentials)
- **Cost optimized** với FREE tier instances và VPC Endpoints
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

## 1. EKS Cluster Creation via Console

{{% notice info %}}
**💡 Console Approach cho EKS:**
Console đủ cho basic EKS cluster setup. Terraform chỉ cần cho IRSA và automation.
{{% /notice %}}

### 1.1. Basic EKS Cluster Setup

1. **Navigate to EKS Console:**
   - AWS Console → EKS → "Create cluster"

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

3. **Networking Configuration:**
   ```
   VPC: mlops-retail-forecast-dev-vpc
   Subnets: All 4 subnets (public + private)
   Security groups: mlops-retail-forecast-dev-eks-control-plane-sg
   ```

4. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

### 1.2. Essential Add-ons Installation

**Install qua Console:**
- **CoreDNS**: v1.10.1-eksbuild.5
- **kube-proxy**: v1.28.2-eksbuild.2  
- **VPC CNI**: v1.15.4-eksbuild.1
- **EBS CSI Driver**: v1.24.1-eksbuild.1

{{% notice success %}}
**🎯 Console Setup Complete!**

Basic EKS cluster đã sẵn sàng:
- ✅ Control plane ACTIVE
- ✅ Essential add-ons installed
- ✅ CloudWatch logging enabled
- ✅ Ready cho IRSA setup
{{% /notice %}}

## 2. Cost Optimization - Free Tier EC2 Instances

{{% notice info %}}
**💡 Free Tier Optimization:**
Sử dụng t2.micro instances (free tier) để giảm chi phí cho development/testing.
{{% /notice %}}

### 2.1. Free Tier Instance Types

**Recommended cho Development:**
```
t2.micro (Free Tier):
- 1 vCPU, 1 GB RAM
- 750 hours/month FREE (12 months)
- Perfect cho basic testing và learning

t2.small (Low Cost):
- 1 vCPU, 2 GB RAM  
- $16.06/month
- Better performance cho development
```

### 2.2. Cost Comparison

| Instance Type | vCPU | RAM | Free Tier | Monthly Cost |
|---------------|------|-----|-----------|--------------|
| **t2.micro**  | 1    | 1GB | ✅ 750h    | **$0** (12 months) |
| **t2.small**  | 1    | 2GB | ❌         | $16.06 |
| **t3.medium** | 2    | 4GB | ❌         | $30.40 |

**Savings with Free Tier:**
- **t2.micro**: $0/month (12 months free)
- **Total EKS cost**: ~$73/month (control plane only)
- **Node cost**: $0/month (free tier)

## 3. Terraform cho IRSA Setup (Essential Only)

{{% notice info %}}
**💡 Khi nào cần Terraform cho Task 4:**
- ✅ **IRSA setup** - Complex OIDC integration không thể làm dễ qua Console
- ✅ **Automation** - CI/CD deployment cần reproducible setup

**Console đủ cho:** EKS cluster, add-ons, basic configuration
{{% /notice %}}

### 3.1. IRSA Setup Purpose

{{% notice success %}}
**🎯 Mục đích của Terraform code trong Task 4:**

**Input:** 
- EKS cluster đã tạo qua Console
- VPC infrastructure từ Task 2

**Terraform sẽ làm gì:**
1. **Reference existing EKS cluster** từ Console
2. **Setup IRSA** với OIDC provider và roles
3. **Configure Service Accounts** với proper annotations
4. **Enable secure pod authentication** (no hardcoded credentials)

**Kết quả sau khi chạy:**
- ✅ IRSA OIDC provider created
- ✅ S3/CloudWatch access roles configured
- ✅ Pods có thể access AWS services securely
- ✅ No hardcoded credentials needed
{{% /notice %}}

### 3.2. IRSA Foundation - OIDC Provider

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

### 3.3. IRSA Role for ML Workloads (S3 Access)

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

### 3.4. IRSA Role for CloudWatch Monitoring

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

### 3.5. Kubernetes Service Accounts với IRSA Annotations

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

### 3.6. Test Pod với IRSA Authentication

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

### 3.7. Node Group với t2.micro (Free Tier)

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Create managed node group** với t2.micro instances (FREE tier)
2. **Reference existing EKS cluster** từ Console
3. **Configure auto-scaling** và security settings
4. **Enable cost optimization** với free tier instances

**Kết quả:** EKS cluster có worker nodes để chạy pods
{{% /notice %}}

**File: `aws/infra/eks-nodegroup.tf`**

```hcl
# Reference existing EKS cluster từ Console
data "aws_eks_cluster" "main" {
  name = "${var.project_name}-${var.environment}-cluster"
}

# Reference IAM role cho node group từ Task 3
data "aws_iam_role" "node_group" {
  name = "${var.project_name}-${var.environment}-eks-node-group-role"
}

# Reference VPC và subnets từ Task 2
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

# Create managed node group với t2.micro (FREE tier)
resource "aws_eks_node_group" "main" {
  cluster_name    = data.aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-node-group"
  node_role_arn   = data.aws_iam_role.node_group.arn
  subnet_ids      = data.aws_subnets.private.ids

  # FREE Tier instance configuration
  instance_types = ["t2.micro"]  # FREE tier instances
  capacity_type  = "ON_DEMAND"   # On-demand để đảm bảo free tier

  # Scaling configuration
  scaling_config {
    desired_size = 2  # 2 nodes cho high availability
    max_size     = 4  # Scale up khi cần
    min_size     = 1  # Scale down để tiết kiệm
  }

  # Update configuration
  update_config {
    max_unavailable_percentage = 50
  }

  # Launch template để tối ưu cho free tier
  launch_template {
    name    = aws_launch_template.node_group.name
    version = aws_launch_template.node_group.latest_version
  }

  depends_on = [aws_launch_template.node_group]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-node-group"
    Type = "eks-node-group"
    InstanceType = "t2.micro"
    FreeTier = "true"
  })
}

# Launch template cho node group optimization
resource "aws_launch_template" "node_group" {
  name_prefix   = "${var.project_name}-${var.environment}-node-"
  description   = "Launch template for EKS node group with t2.micro optimization"
  image_id      = data.aws_ssm_parameter.eks_ami.value  # Latest EKS optimized AMI
  instance_type = "t2.micro"

  # User data để optimize cho free tier
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    cluster_name = data.aws_eks_cluster.main.name
    cluster_endpoint = data.aws_eks_cluster.main.endpoint
    cluster_ca = data.aws_eks_cluster.main.certificate_authority[0].data
  }))

  vpc_security_group_ids = [aws_security_group.node_group.id]

  # Block device mapping cho free tier storage
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 20  # 20GB (within 30GB free tier)
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.project_name}-${var.environment}-node"
      Type = "eks-node"
      InstanceType = "t2.micro"
      FreeTier = "true"
    })
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-launch-template"
    Type = "launch-template"
  })
}

# Security group cho node group
resource "aws_security_group" "node_group" {
  name_prefix = "${var.project_name}-${var.environment}-node-group-"
  vpc_id      = data.aws_vpc.main.id

  # Allow all traffic within VPC
  ingress {
    from_port = 0
    to_port   = 65535
    protocol  = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-node-group-sg"
    Type = "security-group"
  })
}

# Get latest EKS optimized AMI
data "aws_ssm_parameter" "eks_ami" {
  name = "/aws/service/eks/optimized-ami/1.28/amazon-linux-2023/x86_64/amazon-eks-node-al2023-x86_64-v20231213/recommended/image_id"
}

# User data script cho node initialization
# File: aws/infra/user_data.sh
locals {
  user_data_script = <<-EOF
#!/bin/bash
/etc/eks/bootstrap.sh ${data.aws_eks_cluster.main.name}
EOF
}
```

## 4. Terraform Deployment (IRSA Only)

### 4.1. Essential Variables

**File: `aws/infra/variables.tf`:**

```hcl
variable "project_name" {
  description = "Name of the MLOps project"
  type        = string
  default     = "mlops-retail-forecast"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Project     = "mlops-retail-forecast"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}
```

## 5. Terraform Deployment (IRSA Only)

### 5.1. IRSA Deployment Process

{{% notice success %}}
**🚀 IRSA + Node Group Deployment Process:**

**Bước 1:** Reference EKS cluster từ Console  
**Bước 2:** Setup IRSA với OIDC provider và roles  
**Bước 3:** Create Node Group với t2.micro (FREE tier)
**Bước 4:** Configure Service Accounts với proper annotations  
**Bước 5:** Test secure pod authentication  

**Time required:** ~10-15 phút
{{% /notice %}}

```bash
# BƯỚC 1: Navigate to infrastructure directory
cd aws/infra

# BƯỚC 2: Plan IRSA setup và Node Group (xem Terraform sẽ làm gì)
terraform plan -target=aws_iam_openid_connect_provider.eks_oidc \
               -target=aws_iam_role.irsa_s3_access \
               -target=aws_iam_role.irsa_cloudwatch_access \
               -target=aws_eks_node_group.main

# BƯỚC 3: Apply IRSA configuration và Node Group
terraform apply -target=aws_iam_openid_connect_provider.eks_oidc \
                -target=aws_iam_role.irsa_s3_access \
                -target=aws_iam_role.irsa_cloudwatch_access \
                -target=aws_eks_node_group.main
```

**Expected Apply Output:**
```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Resources Created:
✅ aws_iam_openid_connect_provider.eks_oidc (IRSA OIDC provider)
✅ aws_iam_role.irsa_s3_access (S3 access role cho ML workloads)
✅ aws_iam_role_policy.irsa_s3_policy (Least privilege S3 permissions)
✅ aws_iam_role.irsa_cloudwatch_access (CloudWatch monitoring role)
✅ aws_iam_role_policy_attachment.irsa_cloudwatch_policy (CloudWatch permissions)
✅ aws_eks_node_group.main (t2.micro FREE tier nodes)
✅ aws_launch_template.node_group (Launch template cho optimization)
✅ aws_security_group.node_group (Security group cho nodes)

Outputs:
irsa_s3_access_role_arn = "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-irsa-s3-access"
irsa_cloudwatch_role_arn = "arn:aws:iam::123456789012:role/mlops-retail-forecast-dev-irsa-cloudwatch"
node_group_arn = "arn:aws:eks:ap-southeast-1:123456789012:nodegroup/mlops-retail-forecast-dev-cluster/mlops-retail-forecast-dev-node-group/12345678-1234-1234-1234-123456789012"
```

### 5.2. Configure kubectl (EKS đã tạo qua Console)

```bash
# Update kubeconfig với EKS cluster từ Console
aws eks update-kubeconfig --region ap-southeast-1 --name mlops-retail-forecast-dev-cluster

# Verify cluster access
kubectl get svc
kubectl get nodes
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
# EKS Cluster Outputs (from Console)
output "cluster_id" {
  description = "EKS cluster ID"
  value       = data.aws_eks_cluster.main.id
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = data.aws_eks_cluster.main.arn
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = data.aws_eks_cluster.main.endpoint
}

output "cluster_version" {
  description = "The Kubernetes version for the cluster"
  value       = data.aws_eks_cluster.main.version
}

# Node Group Outputs
output "node_group_arn" {
  description = "EKS node group ARN"
  value       = aws_eks_node_group.main.arn
}

output "node_group_status" {
  description = "Status of the EKS node group"
  value       = aws_eks_node_group.main.status
}

output "node_group_instance_types" {
  description = "Instance types used by the node group"
  value       = aws_eks_node_group.main.instance_types
}

# IRSA Outputs
output "irsa_s3_access_role_arn" {
  description = "IRSA role ARN for S3 access"
  value       = aws_iam_role.irsa_s3_access.arn
}

output "irsa_cloudwatch_role_arn" {
  description = "IRSA role ARN for CloudWatch access"
  value       = aws_iam_role.irsa_cloudwatch_access.arn
}

output "oidc_provider_arn" {
  description = "OIDC provider ARN for IRSA"
  value       = aws_iam_openid_connect_provider.eks_oidc.arn
}
```

## 👉 Kết quả Task 4

Sau Task 4, bạn sẽ có EKS Cluster production-ready, chạy hoàn toàn trong private subnet và tích hợp với VPC Endpoints từ Task 2, tiết kiệm chi phí NAT Gateway và tăng mức độ bảo mật.

### ✅ Deliverables Completed

- **EKS Control Plane ACTIVE**: Managed Kubernetes cluster với multi-AZ high availability
- **IRSA Configured**: OIDC provider và Service Account authentication setup
- **Managed Node Groups**: 2x t2.micro instances (FREE tier) trải đều trên ≥2 AZ
- **VPC Endpoints Integration**: Sử dụng ECR, S3 endpoints từ Task 2 (70% cost savings)
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **Secure Pod Access**: Pods có thể access S3/CloudWatch qua IRSA (no hardcoded credentials)
- **kubectl Access**: Local development environment configured và tested
- **Cost Optimization**: $117.40/month saved với FREE tier + VPC Endpoints

### Architecture Achieved

```
✅ EKS Cluster: mlops-retail-forecast-dev-cluster (Kubernetes 1.28)
✅ Control Plane: Multi-AZ managed service với full logging
✅ Node Groups: 2x t2.micro nodes (FREE Tier) trong private subnets
✅ IRSA: OIDC provider + S3/CloudWatch access roles
✅ VPC Endpoints: Sử dụng từ Task 2 (Console) - S3 (FREE) + ECR API/DKR ($14.4/month)
✅ Security: Least privilege IAM roles + Security Groups + IRSA authentication
✅ Monitoring: CloudWatch integration với Container Insights
✅ Cost Optimization: 70% savings với VPC Endpoints + FREE tier instances
```

### Cost Summary

| Component | Monthly Cost | Free Tier | Savings |
|-----------|--------------|-----------|---------|
| **EKS Control Plane** | $73.00 | ❌ | - |
| **EC2 Nodes (2x t2.micro)** | **$0** | ✅ 12 months | **$60.00 saved** |
| **VPC Endpoints** | $14.4 | ❌ | $49.4 vs NAT GW |
| **EBS Storage (20GB)** | ~$2.00 | ✅ 30GB | **$8.00 saved** |
| **Total** | **~$89.40** | | **$117.40 saved** |

**Free Tier Benefits (12 months):**
- **t2.micro instances**: 750 hours/month FREE
- **EBS storage**: 30GB FREE
- **Total savings**: $68/month (instances + storage)

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