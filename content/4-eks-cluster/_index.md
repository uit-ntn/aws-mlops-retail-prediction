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

5. **Sử dụng VPC Endpoints từ Task 2**
   - ECR API & DKR: sử dụng endpoints đã tạo ở Task 2 để pod pull container image
   - S3 Gateway Endpoint: sử dụng endpoint đã tạo ở Task 2 để node/pod đọc/ghi dữ liệu ML
   - CloudWatch Logs/Monitoring: sử dụng endpoints đã tạo ở Task 2 cho logging và metrics

## ✅ Deliverables

- **EKS cluster ACTIVE** với node groups trải trên nhiều AZ
- **Node có thể pull image** từ ECR qua VPC Endpoint (không cần NAT Gateway)
- **Logs/metrics** từ pod được gửi lên CloudWatch thành công
- **kubeconfig usable** cho CI/CD pipeline

## 📊 Acceptance Criteria

- ✅ `kubectl get nodes` → tất cả node trạng thái Ready
- ✅ Add-ons (CNI, CoreDNS, kube-proxy, metrics-server) chạy ổn định
- ✅ Pod mẫu deploy lên EKS pull image từ ECR thành công
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

### 2.1. EKS Cluster với VPC Integration

**File: `aws/infra/eks-cluster.tf`**

```hcl
# Data sources from Task 2 VPC outputs
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

# EKS Cluster với integration từ Task 2-3
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}-cluster"
  role_arn = data.aws_iam_role.eks_cluster.arn  # From Task 3
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

  # Dependencies từ previous tasks
  depends_on = [
    data.aws_vpc_endpoints.s3,      # Task 2 VPC Endpoints
    data.aws_vpc_endpoints.ecr_api,
    data.aws_vpc_endpoints.ecr_dkr,
    data.aws_iam_role.eks_cluster   # Task 3 IAM
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cluster"
    Type = "eks-cluster"
    CreatedBy = "terraform"
  })
}

# Reference IAM role từ Task 3
data "aws_iam_role" "eks_cluster" {
  name = "${var.project_name}-${var.environment}-eks-cluster-role"
}

# Reference Security Group từ Task 2
data "aws_security_group" "eks_control_plane" {
  filter {
    name   = "tag:Name"
    values = ["${var.project_name}-${var.environment}-eks-control-plane-sg"]
  }
}

# Reference KMS Key từ Task 3
data "aws_kms_key" "eks" {
  key_id = "alias/${var.project_name}-${var.environment}-eks"
}
```

### 2.2. Essential Variables cho Terraform

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

**File: `aws/infra/eks-addons.tf`**

```hcl
# Essential EKS Add-ons cho production (automated management)
locals {
  essential_addons = {
    coredns = {
      addon_version               = "v1.10.1-eksbuild.5"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
    }
    kube-proxy = {
      addon_version               = "v1.28.2-eksbuild.2"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
    }
    vpc-cni = {
      addon_version               = "v1.15.4-eksbuild.1"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
    }
    aws-ebs-csi-driver = {
      addon_version               = "v1.24.1-eksbuild.1"
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "PRESERVE"
      service_account_role_arn    = data.aws_iam_role.ebs_csi_driver.arn
    }
  }
}

# EKS Add-ons automated installation
resource "aws_eks_addon" "essential" {
  for_each = local.essential_addons

  cluster_name             = aws_eks_cluster.main.name
  addon_name               = each.key
  addon_version            = each.value.addon_version
  resolve_conflicts        = each.value.resolve_conflicts_on_update
  service_account_role_arn = lookup(each.value, "service_account_role_arn", null)

  depends_on = [aws_eks_cluster.main]

  tags = merge(var.common_tags, {
    Name      = "${var.project_name}-${var.environment}-${each.key}"
    Type      = "eks-addon"
    CreatedBy = "terraform"
  })
}

# Reference EBS CSI Driver IAM role từ Task 3
data "aws_iam_role" "ebs_csi_driver" {
  name = "${var.project_name}-${var.environment}-ebs-csi-driver-role"
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

## 3. Kubectl Access Configuration

### 3.1. EKS Cluster Creation via Console

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

### 3.2. Add-ons Installation

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

![Addons Overview](../images/04-eks-cluster/06-addons-overview.png)

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

![Install Addons](../images/04-eks-cluster/07-install-addons.png)

## 4. Integration với VPC Endpoints từ Task 2

### 4.1. VPC Endpoints Benefits cho EKS

**EKS sử dụng VPC Endpoints đã được tạo ở Task 2** để giảm chi phí và tăng bảo mật:

```
✅ VPC Endpoints từ Task 2 (đã có sẵn):
├── S3 Gateway Endpoint: FREE
├── ECR API Interface: $7.2/month
├── ECR DKR Interface: $7.2/month
├── CloudWatch Logs: $7.2/month
└── Total: $21.6/month (vs $71/month NAT Gateway)

💰 Cost Savings: 70% reduction ($49.4/month saved)
```

### 4.2. Terraform Integration với VPC Endpoints

**EKS sử dụng VPC Endpoints được tạo từ Task 2 thông qua data sources:**

```hcl
# Data sources to reference VPC Endpoints from Task 2
data "aws_vpc_endpoints" "s3" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  filter {
    name   = "service-name"
    values = ["com.amazonaws.${var.aws_region}.s3"]
  }
}

data "aws_vpc_endpoints" "ecr_api" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  filter {
    name   = "service-name"
    values = ["com.amazonaws.${var.aws_region}.ecr.api"]
  }
}

data "aws_vpc_endpoints" "ecr_dkr" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  filter {
    name   = "service-name"
    values = ["com.amazonaws.${var.aws_region}.ecr.dkr"]
  }
}

data "aws_vpc_endpoints" "logs" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  filter {
    name   = "service-name"
    values = ["com.amazonaws.${var.aws_region}.logs"]
  }
}
```

**EKS Cluster dependencies với VPC Endpoints:**

```hcl
# EKS Cluster với dependency từ VPC Endpoints
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.cluster_endpoint_public_access_cidrs
    security_group_ids      = [aws_security_group.eks_control_plane.id]
  }

  # EKS Cluster phụ thuộc vào VPC Endpoints để private subnets access AWS services
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy.eks_cluster_cloudwatch,
    aws_cloudwatch_log_group.eks_cluster,
    # VPC Endpoints từ Task 2 phải được tạo trước EKS
    data.aws_vpc_endpoints.s3,
    data.aws_vpc_endpoints.ecr_api,
    data.aws_vpc_endpoints.ecr_dkr,
    data.aws_vpc_endpoints.logs
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cluster"
    Type = "eks-cluster"
  })
}
```

### 4.3. Verification VPC Endpoints Integration

**Test ECR access qua VPC Endpoints từ Task 2:**

```bash
# Verify VPC Endpoints từ Task 2 đã tồn tại
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'VpcEndpoints[*].{Service:ServiceName,State:State,Type:VpcEndpointType}'

# Expected output:
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

## 5. kubectl Configuration

### 5.1. Install kubectl

**Windows (PowerShell):**
```powershell
# Install kubectl using chocolatey
choco install kubernetes-cli

# Or download directly
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
```

**macOS:**
```bash
# Install using Homebrew
brew install kubectl

# Or using curl
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/darwin/amd64/kubectl"
```

**Linux:**
```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 4.2. Configure kubeconfig

```bash
# Configure kubectl to connect to EKS cluster
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name mlops-retail-forecast-dev-cluster

# Verify connection
kubectl get nodes
kubectl get namespaces
kubectl cluster-info
```

### 4.3. Verify Cluster Access

```bash
# Check cluster status
kubectl get componentstatuses

# View cluster information
kubectl cluster-info

# List all resources in kube-system namespace
kubectl get all -n kube-system

# Check add-ons status
kubectl get deployments -n kube-system
```

**Expected Output:**
```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
coredns        2/2     2            2           10m
```

## 5. Terraform Deployment

### 5.1. Plan và Apply

```bash
# Navigate to infrastructure directory
cd aws/infra

# Plan EKS cluster creation
terraform plan -target=aws_eks_cluster.main -var-file="../terraform.tfvars"

# Apply EKS cluster
terraform apply -target=aws_eks_cluster.main -var-file="../terraform.tfvars"
```

**Expected Apply Output:**
```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:
cluster_id = "mlops-retail-forecast-dev-cluster"
cluster_arn = "arn:aws:eks:ap-southeast-1:123456789012:cluster/mlops-retail-forecast-dev-cluster"
cluster_endpoint = "https://A1B2C3D4E5F6G7H8I9J0.gr7.ap-southeast-1.eks.amazonaws.com"
cluster_version = "1.28"
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
- **Managed Node Groups**: EC2 instances trải đều trên ≥2 AZ với auto-scaling
- **VPC Endpoints Integration**: Sử dụng ECR, S3, CloudWatch endpoints từ Task 2
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **kubectl Access**: Local development environment configured và tested
- **Cost Optimization**: 70% giảm chi phí so với NAT Gateway approach

### Architecture Achieved

```
✅ EKS Cluster: mlops-retail-forecast-dev-cluster (Kubernetes 1.28)
✅ Control Plane: Multi-AZ managed service với full logging
✅ Node Groups: 2-4 nodes (t3.medium/large) trong private subnets
✅ VPC Endpoints: Sử dụng từ Task 2 - S3 (FREE) + ECR API/DKR + CloudWatch ($21.6/month)
✅ Security: Least privilege IAM roles + Security Groups
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