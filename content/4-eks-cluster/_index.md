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

5. **Tích hợp với VPC Endpoints**
   - ECR API & DKR: để pod pull container image từ private subnet
   - S3 Gateway Endpoint: để node/pod đọc/ghi dữ liệu ML (dataset, model artifacts)
   - CloudWatch Logs/Monitoring: Fluent Bit trong pod gửi log qua Interface Endpoint

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

{{< imgborder src="/images/04-eks-cluster/01-create-eks-cluster.png" >}}

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

{{< imgborder src="/images/04-eks-cluster/02-cluster-basic-config.png" >}}

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

{{< imgborder src="/images/04-eks-cluster/03-networking-config.png" >}}

4. **Cluster Endpoint Access:**
   ```
   Endpoint private access: Enabled
   Endpoint public access: Enabled
   Public access source: Specific CIDR blocks (your IP)
   ```

{{< imgborder src="/images/04-eks-cluster/04-endpoint-access.png" >}}

5. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

{{< imgborder src="/images/04-eks-cluster/05-logging-config.png" >}}

6. **Encryption Configuration:**
   ```
   Secrets encryption: Enabled
   KMS key: Create new key or use existing
   Key alias: alias/mlops-retail-forecast-dev-eks
   ```

{{< imgborder src="/images/04-eks-cluster/06-encryption-config.png" >}}

### 1.2. Add-ons Installation via Console

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

{{< imgborder src="/images/04-eks-cluster/07-addons-overview.png" >}}

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

{{< imgborder src="/images/04-eks-cluster/08-install-addons.png" >}}

3. **AWS Load Balancer Controller:**
   ```
   Name: aws-load-balancer-controller
   Version: v2.6.3-eksbuild.1
   Service account role: mlops-retail-forecast-dev-alb-controller
   ```

{{< imgborder src="/images/04-eks-cluster/09-alb-controller-addon.png" >}}

### 1.3. Cluster Verification via Console

1. **Cluster Overview:**
   - Check cluster status: "Active"
   - Verify Kubernetes version: 1.28
   - Confirm endpoint access configuration

{{< imgborder src="/images/04-eks-cluster/10-cluster-overview.png" >}}

2. **Add-ons Status:**
   ```
   ✅ coredns: Active
   ✅ kube-proxy: Active
   ✅ vpc-cni: Active
   ✅ aws-ebs-csi-driver: Active
   ✅ aws-load-balancer-controller: Active
   ```

{{< imgborder src="/images/04-eks-cluster/11-addons-status.png" >}}

3. **CloudWatch Logs:**
   - Navigate to CloudWatch → Log groups
   - Verify log group: `/aws/eks/mlops-retail-forecast-dev-cluster/cluster`
   - Check log streams cho control plane components

{{< imgborder src="/images/04-eks-cluster/12-cloudwatch-logs.png" >}}

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

## 2. EKS Managed Node Groups Configuration

### 2.1. Node Group Terraform Configuration

**File: `aws/infra/eks-nodegroup.tf`**

```hcl
# EKS Managed Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-nodes"
  node_role_arn   = aws_iam_role.eks_node_group_role.arn
  subnet_ids      = aws_subnet.private[*].id

  # Instance configuration
  instance_types = var.node_group_instance_types
  capacity_type  = var.node_group_capacity_type
  disk_size      = var.node_group_disk_size

  # Scaling configuration
  scaling_config {
    desired_size = var.node_group_desired_size
    max_size     = var.node_group_max_size
    min_size     = var.node_group_min_size
  }

  # Update configuration
  update_config {
    max_unavailable_percentage = 25
  }

  # Remote access (optional - for debugging)
  remote_access {
    ec2_ssh_key = var.node_group_key_pair
    source_security_group_ids = [aws_security_group.eks_nodes.id]
  }

  # Launch template for advanced configuration
  launch_template {
    id      = aws_launch_template.eks_nodes.id
    version = aws_launch_template.eks_nodes.latest_version
  }

  # Ensure proper ordering of resource creation
  depends_on = [
    aws_iam_role_policy_attachment.eks_node_group_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.eks_node_group_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.eks_node_group_AmazonEC2ContainerRegistryReadOnly,
    aws_iam_role_policy_attachment.eks_node_group_CloudWatchAgentServerPolicy,
    aws_iam_role_policy_attachment.eks_node_group_S3Access
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-node-group"
    Type = "eks-node-group"
  })
}

# Launch Template for Node Group
resource "aws_launch_template" "eks_nodes" {
  name_prefix   = "${var.project_name}-${var.environment}-node-"
  image_id      = data.aws_ami.eks_worker.id
  instance_type = var.node_group_instance_types[0]

  vpc_security_group_ids = [aws_security_group.eks_nodes.id]

  user_data = base64encode(templatefile("${path.module}/templates/userdata.sh", {
    cluster_name        = aws_eks_cluster.main.name
    cluster_endpoint    = aws_eks_cluster.main.endpoint
    cluster_ca          = aws_eks_cluster.main.certificate_authority[0].data
    bootstrap_arguments = var.node_group_bootstrap_arguments
  }))

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = var.node_group_disk_size
      volume_type          = "gp3"
      iops                 = 3000
      throughput           = 125
      encrypted            = true
      delete_on_termination = true
    }
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
    instance_metadata_tags      = "enabled"
  }

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.project_name}-${var.environment}-eks-node"
      Type = "eks-worker-node"
    })
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-node-template"
    Type = "launch-template"
  })
}

# Data source for EKS optimized AMI
data "aws_ami" "eks_worker" {
  filter {
    name   = "name"
    values = ["amazon-eks-node-${var.kubernetes_version}-v*"]
  }

  most_recent = true
  owners      = ["602401143452"] # Amazon EKS AMI Account ID
}
```

### 2.2. Node Group Variables

**Thêm vào `aws/infra/variables.tf`:**

```hcl
# EKS Node Group Configuration
variable "node_group_instance_types" {
  description = "List of instance types for EKS node group"
  type        = list(string)
  default     = ["t3.medium", "t3.large"]
}

variable "node_group_capacity_type" {
  description = "Type of capacity associated with the EKS Node Group. Valid values: ON_DEMAND, SPOT"
  type        = string
  default     = "ON_DEMAND"
}

variable "node_group_disk_size" {
  description = "Disk size in GiB for worker nodes"
  type        = number
  default     = 50
}

variable "node_group_desired_size" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 2
}

variable "node_group_max_size" {
  description = "Maximum number of worker nodes"
  type        = number
  default     = 4
}

variable "node_group_min_size" {
  description = "Minimum number of worker nodes"
  type        = number
  default     = 1
}

variable "node_group_key_pair" {
  description = "EC2 Key Pair name for SSH access to nodes"
  type        = string
  default     = null
}

variable "node_group_bootstrap_arguments" {
  description = "Additional arguments for the EKS bootstrap script"
  type        = string
  default     = ""
}
```

### 2.3. Node Group IAM Role

```hcl
# IAM Role for EKS Node Group
resource "aws_iam_role" "eks_node_group_role" {
  name = "${var.project_name}-${var.environment}-eks-node-group"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-node-group"
    Type = "iam-role"
  })
}

# Required IAM policies for EKS Node Group
resource "aws_iam_role_policy_attachment" "eks_node_group_AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_group_role.name
}

resource "aws_iam_role_policy_attachment" "eks_node_group_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_node_group_role.name
}

resource "aws_iam_role_policy_attachment" "eks_node_group_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_node_group_role.name
}

# CloudWatch access for logging and monitoring
resource "aws_iam_role_policy_attachment" "eks_node_group_CloudWatchAgentServerPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
  role       = aws_iam_role.eks_node_group_role.name
}

# S3 access for ML data and model artifacts
resource "aws_iam_role_policy_attachment" "eks_node_group_S3Access" {
  policy_arn = aws_iam_policy.eks_node_s3_access.arn
  role       = aws_iam_role.eks_node_group_role.name
}

# Custom S3 access policy for ML workloads
resource "aws_iam_policy" "eks_node_s3_access" {
  name        = "${var.project_name}-${var.environment}-eks-node-s3-access"
  description = "S3 access policy for EKS nodes"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-*",
          "arn:aws:s3:::${var.project_name}-${var.environment}-*/*"
        ]
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-node-s3-policy"
    Type = "iam-policy"
  })
}
```

### 2.4. User Data Template

**File: `aws/infra/templates/userdata.sh`**

```bash
#!/bin/bash
set -o xtrace

# Bootstrap the node to join the EKS cluster
/etc/eks/bootstrap.sh ${cluster_name} ${bootstrap_arguments}

# Install additional packages for ML workloads
yum update -y
yum install -y amazon-cloudwatch-agent

# Configure CloudWatch agent
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/aws/eks/${cluster_name}/system",
            "log_stream_name": "{instance_id}/messages"
          },
          {
            "file_path": "/var/log/dmesg",
            "log_group_name": "/aws/eks/${cluster_name}/system",
            "log_stream_name": "{instance_id}/dmesg"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "EKS/Node",
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "diskio": {
        "measurement": [
          "io_time"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF

# Start CloudWatch agent
systemctl enable amazon-cloudwatch-agent
systemctl start amazon-cloudwatch-agent

# Signal that the instance is ready
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource NodeGroup --region ${AWS::Region}
```

## 3. EKS Add-ons Terraform Configuration

### 3.1. Essential EKS Add-ons

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

{{< imgborder src="/images/04-eks-cluster/01-create-eks-cluster.png" >}}

2. **Basic Configuration:**
   ```
   Cluster name: mlops-retail-forecast-dev-cluster
   Kubernetes version: 1.28
   Cluster service role: mlops-retail-forecast-dev-eks-cluster-role
   ```

{{< imgborder src="/images/04-eks-cluster/02-cluster-basic-config.png" >}}

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

{{< imgborder src="/images/04-eks-cluster/03-networking-config.png" >}}

4. **Cluster Endpoint Access:**
   ```
   Endpoint private access: Enabled
   Endpoint public access: Enabled
   Public access source: Specific CIDR blocks (your IP)
   ```

{{< imgborder src="/images/04-eks-cluster/04-endpoint-access.png" >}}

5. **Logging Configuration:**
   ```
   Control plane logging:
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

{{< imgborder src="/images/04-eks-cluster/05-logging-config.png" >}}

### 3.2. Add-ons Installation

1. **Navigate to Add-ons Tab:**
   - Chọn cluster vừa tạo
   - Click "Add-ons" tab
   - Chọn "Add new"

{{< imgborder src="/images/04-eks-cluster/06-addons-overview.png" >}}

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

{{< imgborder src="/images/04-eks-cluster/07-install-addons.png" >}}

## 4. VPC Endpoints Integration for Cost Optimization

### 4.1. VPC Endpoints Benefits cho EKS

**Thay thế NAT Gateway bằng VPC Endpoints** để giảm chi phí và tăng bảo mật:

```
Cost Comparison (Monthly):
├── Traditional NAT Gateway: $71/month
│   ├── 2x NAT Gateway: $64/month
│   ├── 2x Elastic IP: $7.2/month
│   └── Data transfer: $0.045/GB
└── VPC Endpoints: $21.6/month
    ├── S3 Gateway Endpoint: FREE
    ├── ECR API Interface: $7.2/month
    ├── ECR DKR Interface: $7.2/month
    ├── CloudWatch Logs: $7.2/month
    └── Data transfer: $0.01/GB (70% cheaper)

💰 Cost Savings: 70% reduction ($49.4/month saved)
```

### 4.2. Essential VPC Endpoints cho EKS

**File: `aws/infra/vpc-endpoints.tf`**

```hcl
# S3 Gateway Endpoint (FREE - no hourly charges)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.aws_region}.s3"
  
  # Associate với private route tables
  route_table_ids = aws_route_table.private[*].id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket",
          "s3:DeleteObject"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-*",
          "arn:aws:s3:::${var.project_name}-${var.environment}-*/*",
          "arn:aws:s3:::amazon-eks-*",
          "arn:aws:s3:::amazon-eks-*/*"
        ]
      }
    ]
  })
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-s3-endpoint"
    Type = "gateway-endpoint"
  })
}

# ECR API Interface Endpoint - cho Docker registry API calls
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  # Enable DNS resolution
  private_dns_enabled = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:DescribeRepositories",
          "ecr:DescribeImages"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-api-endpoint"
    Type = "interface-endpoint"
  })
}

# ECR DKR Interface Endpoint - cho Docker image pulls
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-dkr-endpoint"
    Type = "interface-endpoint"
  })
}

# CloudWatch Logs Interface Endpoint - cho logging
resource "aws_vpc_endpoint" "logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-logs-endpoint"
    Type = "interface-endpoint"
  })
}

# CloudWatch Monitoring Interface Endpoint - cho metrics
resource "aws_vpc_endpoint" "monitoring" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.monitoring"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-monitoring-endpoint"
    Type = "interface-endpoint"
  })
}

# STS Interface Endpoint - cho IAM role assumptions
resource "aws_vpc_endpoint" "sts" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.sts"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-sts-endpoint"
    Type = "interface-endpoint"
  })
}
```

### 4.3. Verification VPC Endpoints

**Test ECR access qua VPC Endpoints:**

```bash
# Deploy test pod để verify ECR access
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

# Check pod có thể pull image thành công
kubectl get pods test-ecr-access
kubectl describe pod test-ecr-access

# Test S3 access từ trong pod
kubectl exec -it test-ecr-access -- aws s3 ls

# Cleanup
kubectl delete pod test-ecr-access
```

**Verify VPC Endpoints hoạt động:**

```bash
# Check VPC endpoints status
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'VpcEndpoints[*].{Service:ServiceName,State:State,Type:VpcEndpointType}'

# Test DNS resolution cho ECR endpoints từ EKS node
kubectl run test-dns --image=busybox --rm -it --restart=Never -- \
  nslookup ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# Expected: Should resolve to private IP addresses (10.0.x.x)
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

Sau Task 4, bạn sẽ có EKS Cluster production-ready, chạy hoàn toàn trong private subnet + VPC Endpoints, tiết kiệm chi phí NAT Gateway và tăng mức độ bảo mật.

### ✅ Deliverables Completed

- **EKS Control Plane ACTIVE**: Managed Kubernetes cluster với multi-AZ high availability
- **Managed Node Groups**: EC2 instances trải đều trên ≥2 AZ với auto-scaling
- **VPC Endpoints Integration**: ECR, S3, CloudWatch access không cần NAT Gateway
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **kubectl Access**: Local development environment configured và tested
- **Cost Optimization**: 70% giảm chi phí so với NAT Gateway approach

### Architecture Achieved

```
✅ EKS Cluster: mlops-retail-forecast-dev-cluster (Kubernetes 1.28)
✅ Control Plane: Multi-AZ managed service với full logging
✅ Node Groups: 2-4 nodes (t3.medium/large) trong private subnets
✅ VPC Endpoints: S3 (FREE) + ECR API/DKR + CloudWatch ($21.6/month)
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