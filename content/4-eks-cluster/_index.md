---
title: "EKS Control Plane Setup"
date: 2025-08-30T13:00:00+07:00
weight: 4
chapter: false
pre: "<b>4. </b>"
---

## Mục tiêu Task 4

Triển khai EKS Control Plane để quản lý Kubernetes cluster - thành phần trung tâm chịu trách nhiệm điều phối các pod, service và workload của hệ thống MLOps:

1. **EKS Cluster Creation** - Khởi tạo managed Kubernetes control plane
2. **Multi-AZ High Availability** - Control plane distributed across availability zones
3. **VPC Integration** - Kết nối cluster với networking infrastructure đã tạo
4. **IAM Integration** - Sử dụng IAM roles cho secure cluster management
5. **kubectl Configuration** - Setup local access để manage cluster

{{% notice info %}}
**📋 Scope Task 4: EKS Control Plane Foundation**

Task này tạo Kubernetes control plane cho MLOps platform:
- ✅ EKS cluster với managed control plane (multi-AZ)
- ✅ Integration với VPC và security groups từ Task 2
- ✅ IAM roles integration từ Task 3
- ✅ Cluster logging và monitoring setup
- ✅ kubectl access configuration
{{% /notice %}}

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

{{< imgborder src="/images/04-eks-cluster/01-create-eks-cluster.png" title="Tạo EKS cluster qua AWS Console" >}}

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

{{< imgborder src="/images/04-eks-cluster/02-cluster-basic-config.png" title="Cấu hình cơ bản EKS cluster" >}}

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

{{< imgborder src="/images/04-eks-cluster/03-networking-config.png" title="Cấu hình networking cho EKS cluster" >}}

4. **Cluster Endpoint Access:**
   ```
   Endpoint private access: Enabled
   Endpoint public access: Enabled
   Public access source: Specific CIDR blocks (your IP)
   ```

{{< imgborder src="/images/04-eks-cluster/04-endpoint-access.png" title="Cấu hình cluster endpoint access" >}}

5. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

{{< imgborder src="/images/04-eks-cluster/05-logging-config.png" title="Enable EKS control plane logging" >}}

6. **Encryption Configuration:**
   ```
   Secrets encryption: Enabled
   KMS key: Create new key or use existing
   Key alias: alias/mlops-retail-forecast-dev-eks
   ```

{{< imgborder src="/images/04-eks-cluster/06-encryption-config.png" title="Configure EKS secrets encryption" >}}

### 1.2. Add-ons Installation via Console

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

{{< imgborder src="/images/04-eks-cluster/07-addons-overview.png" title="EKS Add-ons management interface" >}}

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

{{< imgborder src="/images/04-eks-cluster/08-install-addons.png" title="Install essential EKS add-ons" >}}

3. **AWS Load Balancer Controller:**
   ```
   Name: aws-load-balancer-controller
   Version: v2.6.3-eksbuild.1
   Service account role: mlops-retail-forecast-dev-alb-controller
   ```

{{< imgborder src="/images/04-eks-cluster/09-alb-controller-addon.png" title="Install AWS Load Balancer Controller" >}}

### 1.3. Cluster Verification via Console

1. **Cluster Overview:**
   - Check cluster status: "Active"
   - Verify Kubernetes version: 1.28
   - Confirm endpoint access configuration

{{< imgborder src="/images/04-eks-cluster/10-cluster-overview.png" title="EKS cluster overview và status" >}}

2. **Add-ons Status:**
   ```
   ✅ coredns: Active
   ✅ kube-proxy: Active
   ✅ vpc-cni: Active
   ✅ aws-ebs-csi-driver: Active
   ✅ aws-load-balancer-controller: Active
   ```

{{< imgborder src="/images/04-eks-cluster/11-addons-status.png" title="Verify add-ons status" >}}

3. **CloudWatch Logs:**
   - Navigate to CloudWatch → Log groups
   - Verify log group: `/aws/eks/mlops-retail-forecast-dev-cluster/cluster`
   - Check log streams cho control plane components

{{< imgborder src="/images/04-eks-cluster/12-cloudwatch-logs.png" title="EKS control plane logs trong CloudWatch" >}}

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

## 2. EKS Cluster Terraform Configuration

### 1.1. EKS Cluster Resource

**File: `aws/infra/eks-cluster.tf`**

```hcl
# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = concat(aws_subnet.private[*].id, aws_subnet.public[*].id)
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.cluster_endpoint_public_access_cidrs
    security_group_ids      = [aws_security_group.eks_control_plane.id]
  }

  # Enable EKS cluster logging
  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  # Encryption for secrets
  encryption_config {
    provider {
      key_arn = aws_kms_key.eks_cluster.arn
    }
    resources = ["secrets"]
  }

  # Ensure proper ordering of resource creation
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy.eks_cluster_cloudwatch,
    aws_cloudwatch_log_group.eks_cluster
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-cluster"
    Type = "eks-cluster"
  })
}

# CloudWatch Log Group for EKS
resource "aws_cloudwatch_log_group" "eks_cluster" {
  name              = "/aws/eks/${var.project_name}-${var.environment}-cluster/cluster"
  retention_in_days = var.cluster_log_retention_days
  kms_key_id        = aws_kms_key.eks_cluster.arn

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-logs"
    Type = "cloudwatch-log-group"
  })
}

# KMS Key for EKS encryption
resource "aws_kms_key" "eks_cluster" {
  description             = "KMS key for EKS cluster encryption"
  deletion_window_in_days = var.kms_key_deletion_window

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-kms"
    Type = "kms-key"
  })
}

resource "aws_kms_alias" "eks_cluster" {
  name          = "alias/${var.project_name}-${var.environment}-eks"
  target_key_id = aws_kms_key.eks_cluster.key_id
}
```

### 1.2. Additional Variables

**Thêm vào `aws/infra/variables.tf`:**

```hcl
# EKS Cluster Configuration
variable "kubernetes_version" {
  description = "Kubernetes version for EKS cluster"
  type        = string
  default     = "1.28"
}

variable "cluster_endpoint_public_access_cidrs" {
  description = "List of CIDR blocks that can access the public API server endpoint"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Restrict this in production
}

variable "cluster_log_retention_days" {
  description = "Number of days to retain EKS cluster logs"
  type        = number
  default     = 7
}

variable "kms_key_deletion_window" {
  description = "KMS key deletion window in days"
  type        = number
  default     = 7
}

# EKS Add-ons
variable "cluster_addons" {
  description = "Map of cluster addon configurations"
  type = map(object({
    addon_version = string
  }))
  default = {
    coredns = {
      addon_version = "v1.10.1-eksbuild.5"
    }
    kube-proxy = {
      addon_version = "v1.28.2-eksbuild.2"
    }
    vpc-cni = {
      addon_version = "v1.15.4-eksbuild.1"
    }
    aws-ebs-csi-driver = {
      addon_version = "v1.24.1-eksbuild.1"
    }
  }
}
```

### 1.3. Environment-specific Configuration

**Cập nhật `aws/terraform.tfvars`:**

```hcl
# EKS cluster configuration
kubernetes_version = "1.28"
cluster_endpoint_public_access_cidrs = [
  "0.0.0.0/0"  # Restrict to your office/VPN CIDR in production
]
cluster_log_retention_days = 7

# EKS add-ons
cluster_addons = {
  coredns = {
    addon_version = "v1.10.1-eksbuild.5"
  }
  kube-proxy = {
    addon_version = "v1.28.2-eksbuild.2"
  }
  vpc-cni = {
    addon_version = "v1.15.4-eksbuild.1"
  }
  aws-ebs-csi-driver = {
    addon_version = "v1.24.1-eksbuild.1"
  }
}
```

## 2. EKS Add-ons Terraform Configuration

### 2.1. Essential EKS Add-ons

```hcl
# EKS Add-ons
resource "aws_eks_addon" "cluster_addons" {
  for_each = var.cluster_addons

  cluster_name         = aws_eks_cluster.main.name
  addon_name           = each.key
  addon_version        = each.value.addon_version
  resolve_conflicts    = "OVERWRITE"

  depends_on = [
    aws_eks_cluster.main
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-${each.key}"
    Type = "eks-addon"
  })
}

# AWS Load Balancer Controller add-on (for ALB integration)
resource "aws_eks_addon" "aws_load_balancer_controller" {
  cluster_name         = aws_eks_cluster.main.name
  addon_name           = "aws-load-balancer-controller"
  addon_version        = "v2.6.3-eksbuild.1"
  resolve_conflicts    = "OVERWRITE"
  service_account_role_arn = aws_iam_role.aws_load_balancer_controller.arn

  depends_on = [
    aws_eks_cluster.main,
    aws_iam_role.aws_load_balancer_controller
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-alb-controller"
    Type = "eks-addon"
  })
}
```

### 2.2. AWS Load Balancer Controller IAM Role

```hcl
# IAM Role for AWS Load Balancer Controller
resource "aws_iam_role" "aws_load_balancer_controller" {
  name = "${var.project_name}-${var.environment}-alb-controller"

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
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-alb-controller"
    Type = "iam-role"
    Service = "aws-load-balancer-controller"
  })
}

# Attach AWS Load Balancer Controller IAM Policy
resource "aws_iam_role_policy_attachment" "aws_load_balancer_controller" {
  role       = aws_iam_role.aws_load_balancer_controller.name
  policy_arn = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:policy/AWSLoadBalancerControllerIAMPolicy"
}

# Data source for current AWS account
data "aws_caller_identity" "current" {}
```

## 3. Kubectl Access Configuration

### 3.1. EKS Cluster Creation via Console

1. **Navigate to EKS Console:**
   - Đăng nhập AWS Console
   - Navigate to EKS service
   - Chọn "Create cluster"

{{< imgborder src="/images/04-eks-cluster/01-create-eks-cluster.png" title="Tạo EKS cluster qua AWS Console" >}}

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

{{< imgborder src="/images/04-eks-cluster/02-cluster-basic-config.png" title="Cấu hình cơ bản EKS cluster" >}}

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

{{< imgborder src="/images/04-eks-cluster/03-networking-config.png" title="Cấu hình networking cho EKS cluster" >}}

4. **Cluster Endpoint Access:**
   ```
   Endpoint private access: Enabled
   Endpoint public access: Enabled
   Public access source: Specific CIDR blocks (your IP)
   ```

{{< imgborder src="/images/04-eks-cluster/04-endpoint-access.png" title="Cấu hình cluster endpoint access" >}}

5. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

{{< imgborder src="/images/04-eks-cluster/05-logging-config.png" title="Enable EKS control plane logging" >}}

### 3.2. Add-ons Installation

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

{{< imgborder src="/images/04-eks-cluster/06-addons-overview.png" title="EKS Add-ons management interface" >}}

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

{{< imgborder src="/images/04-eks-cluster/07-install-addons.png" title="Install essential EKS add-ons" >}}

## 4. kubectl Configuration

### 4.1. Install kubectl

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

## Kết quả Task 4

✅ **EKS Control Plane**: Managed Kubernetes cluster với multi-AZ high availability  
✅ **VPC Integration**: Cluster connected với networking infrastructure  
✅ **IAM Integration**: Secure access với service roles và IRSA  
✅ **Add-ons Installation**: Essential Kubernetes add-ons deployed  
✅ **kubectl Access**: Local development environment configured  

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 5**: EKS managed node groups deployment
- **Task 6**: ECR repository setup cho container images
- **Task 7**: Build và push inference API container
{{% /notice %}}

{{% notice warning %}}
**🔐 Security Reminder**: 
- Restrict `cluster_endpoint_public_access_cidrs` to your actual IP ranges in production
- Enable all control plane logging cho security audit
- Regularly update Kubernetes version và add-ons
- Implement proper RBAC policies cho team access
{{% /notice %}}