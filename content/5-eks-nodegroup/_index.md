---
title: "EKS Managed Node Group"
date: 2024-01-01T00:00:00+07:00
weight: 5
chapter: false
pre: "<b>5. </b>"
---

## 🎯 Mục tiêu

Triển khai EKS Managed Node Group để cung cấp EC2 worker nodes cho EKS control plane (Task 4).

Đảm bảo node group có khả năng auto-scaling, chạy workload inference API.

Có thể kết hợp On-Demand + Spot Instances để tối ưu chi phí.

Node group trải trên ≥2 AZ để tăng độ sẵn sàng.

## 📥 Input

- **Outputs từ Task 2** (VPC/Subnet/SG)
- **Outputs từ Task 3** (IAM Role cho node group)
- **Cluster EKS từ Task 4**

## 📌 Các bước chính

1. **Tạo Managed Node Group**
   - Sử dụng Terraform để định nghĩa node group
   - Chọn instance type phù hợp (t3.medium/m5.large cho dev, GPU instance cho training/inference đặc thù)
   - Cấu hình auto-scaling: min, max, desired capacity

2. **Triển khai trên nhiều AZ**
   - Node group phải trải ít nhất 2 AZ trong VPC để tăng tính sẵn sàng
   - Private subnet thường được chọn làm placement cho node group

3. **Tích hợp IAM Role**
   - Node IAM Role với quyền pull image từ ECR, đọc/ghi S3, gửi log lên CloudWatch (qua VPC Endpoints nếu Task 2 đã tối ưu)

4. **Auto-Scaling & Spot Optimization**
   - Cho phép node group dùng Spot Instance để giảm chi phí (kèm fallback On-Demand)
   - Có thể triển khai nhiều node group (VD: 1 On-Demand group cho core services, 1 Spot group cho workload batch)

5. **Kiểm thử kết nối**
   - Sau khi node group active, update kubeconfig
   - Chạy lệnh `kubectl get nodes` để xác nhận node ở trạng thái Ready

## ✅ Deliverables

- **EKS Managed Node Group** chạy ổn định, trải trên nhiều AZ
- **Node group có auto-scaling policy** hoạt động
- **IAM Role gắn kèm**, đảm bảo pod có thể truy cập dịch vụ AWS (qua IRSA)

## 📊 Acceptance Criteria

- ✅ Node group trạng thái ACTIVE trên AWS Console
- ✅ Lệnh `kubectl get nodes` hiển thị danh sách node Ready
- ✅ Auto-scaling kích hoạt khi có workload tăng (nếu test HPA)
- ✅ Node group có thể pull image từ ECR và gửi log lên CloudWatch

## ⚠️ Gotchas

- **Spot instance** có thể bị thu hồi bất kỳ lúc nào → cần mixed instance type hoặc fallback On-Demand
- **Nếu thiếu quyền IAM** → node không pull được image từ ECR hoặc không ghi được log
- **Quên cài metrics-server** trong cluster → HPA và autoscaler không hoạt động
- **Nếu private subnet không có VPC Endpoint** → node không truy cập được ECR/S3/CloudWatch

## Tổng quan

**EC2 Managed Node Group** là tập hợp các EC2 instances được AWS EKS quản lý tự động, đóng vai trò worker nodes trong Kubernetes cluster. Đây là tài nguyên tính toán thực tế nơi các pods và services sẽ được triển khai.

### Kiến trúc Node Group

{{< mermaid >}}
graph TB
    subgraph "EKS Cluster"
        CP[Control Plane<br/>Managed by AWS]
    end
    
    subgraph "VPC"
        subgraph "Private Subnet 1a"
            N1[Worker Node 1<br/>t3.medium]
        end
        subgraph "Private Subnet 1b"
            N2[Worker Node 2<br/>t3.medium]
        end
    end
    
    subgraph "AWS Services"
        ECR[ECR Registry<br/>Container Images]
        CW[CloudWatch<br/>Logs & Metrics]
        S3[S3 Bucket<br/>Artifacts]
    end
    
    CP --> N1
    CP --> N2
    N1 --> ECR
    N2 --> ECR
    N1 --> CW
    N2 --> CW
    N1 --> S3
    N2 --> S3
{{< /mermaid >}}

### Thành phần chính

1. **EC2 Instances**: Worker nodes chạy Kubernetes kubelet
2. **Auto Scaling Group**: Quản lý số lượng nodes tự động
3. **Launch Template**: Template cấu hình cho EC2 instances
4. **IAM Roles**: Quyền truy cập ECR, CloudWatch, VPC
5. **Security Groups**: Kiểm soát network traffic

---

## 1. Alternative: AWS Console Implementation

### 1.1. Node Group IAM Role Creation

1. **Navigate to IAM Console:**
   - Đăng nhập AWS Console
   - Navigate to IAM → Roles
   - Chọn "Create role"

{{< imgborder src="/images/05-eks-nodegroup/01-create-nodegroup-role.png" >}}

2. **Select Trusted Entity:**
   ```
   Trusted entity type: AWS service
   Service or use case: EC2
   ```

{{< imgborder src="/images/05-eks-nodegroup/02-select-trusted-entity.png" >}}

3. **Attach Required Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy  
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```

{{< imgborder src="/images/05-eks-nodegroup/03-attach-policies.png" >}}

4. **Role Configuration:**
   ```
   Role name: mlops-retail-forecast-dev-nodegroup-role
   Description: IAM role for EKS managed node group
   ```

{{< imgborder src="/images/05-eks-nodegroup/04-role-configuration.png" >}}

### 1.2. Node Group Creation via Console

1. **Navigate to EKS Cluster:**
   - EKS Console → Clusters
   - Chọn cluster: `mlops-retail-forecast-dev-cluster`
   - Click "Compute" tab
   - Chọn "Add node group"

{{< imgborder src="/images/05-eks-nodegroup/05-add-nodegroup.png" >}}

2. **Node Group Configuration:**
   ```
   Name: mlops-retail-forecast-dev-nodegroup
   Node IAM role: mlops-retail-forecast-dev-nodegroup-role
   Kubernetes labels:
     - nodegroup-type: primary
     - environment: dev
   Kubernetes taints: None
   ```

{{< imgborder src="/images/05-eks-nodegroup/06-nodegroup-config.png" >}}

3. **Compute and Scaling Configuration:**
   ```
   AMI type: Amazon Linux 2 (AL2_x86_64)
   Capacity type: On-Demand
   Instance types: t3.medium
   Disk size: 20 GB
   
   Scaling configuration:
   - Desired size: 2
   - Minimum size: 1  
   - Maximum size: 4
   ```

{{< imgborder src="/images/05-eks-nodegroup/07-compute-scaling.png" >}}

4. **Node Group Networking:**
   ```
   Subnets:
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1a
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1b
   
   Configure SSH access:
   ⬜ Enable SSH access (optional for debugging)
   ```

{{< imgborder src="/images/05-eks-nodegroup/08-networking-config.png" >}}

5. **Advanced Options:**
   ```
   User data: (Leave empty for standard AMI)
   
   EC2 tags:
   - Name: mlops-retail-forecast-dev-worker-node
   - Environment: dev
   - Project: mlops-retail-forecast
   - NodeGroup: primary
   ```

{{< imgborder src="/images/05-eks-nodegroup/09-advanced-options.png" >}}

### 1.3. Node Group Verification

1. **Check Node Group Status:**
   - EKS Console → Cluster → Compute tab
   - Verify status: "Active"
   - Check node count: 2/2 running

{{< imgborder src="/images/05-eks-nodegroup/10-nodegroup-status.png" >}}

2. **EC2 Instances Verification:**
   - Navigate to EC2 Console
   - Filter by tag: `aws:eks:cluster-name = mlops-retail-forecast-dev-cluster`
   - Verify 2 instances running

{{< imgborder src="/images/05-eks-nodegroup/11-ec2-instances.png" >}}

3. **Auto Scaling Group:**
   - Navigate to EC2 Auto Scaling
   - Find ASG: `eks-mlops-retail-forecast-dev-nodegroup-*`
   - Verify desired/min/max capacity

{{< imgborder src="/images/05-eks-nodegroup/12-autoscaling-group.png" >}}

### 1.4. Kubernetes Nodes Verification

1. **Configure kubectl Access:**
   ```bash
   # Update kubeconfig
   aws eks update-kubeconfig --region ap-southeast-1 --name mlops-retail-forecast-dev-cluster
   
   # Verify cluster access
   kubectl cluster-info
   ```

{{< imgborder src="/images/05-eks-nodegroup/13-kubectl-config.png" >}}

2. **Check Node Status:**
   ```bash
   # List all nodes
   kubectl get nodes
   
   # Get detailed node information
   kubectl get nodes -o wide
   
   # Describe specific node
   kubectl describe node <node-name>
   ```

{{< imgborder src="/images/05-eks-nodegroup/14-kubectl-nodes.png" >}}

3. **Verify Node Labels and Capacity:**
   ```bash
   # Check node labels
   kubectl get nodes --show-labels
   
   # Check node capacity and allocatable resources
   kubectl describe nodes | grep -A 5 "Capacity\|Allocatable"
   ```

{{< imgborder src="/images/05-eks-nodegroup/15-node-details.png" >}}

{{% notice success %}}
**🎯 Console Implementation Complete!**

EKS Managed Node Group đã được tạo thành công với:
- ✅ 2 worker nodes ở trạng thái Ready
- ✅ IAM roles configured properly
- ✅ Private subnet deployment
- ✅ Auto scaling configured (1-4 nodes)
- ✅ Proper tagging và labeling
{{% /notice %}}

{{% notice info %}}
**💡 Console vs Terraform:**

**Console Advantages:**
- ✅ Visual node group creation wizard
- ✅ Real-time scaling adjustments
- ✅ Easy instance type changes
- ✅ Integrated health monitoring

**Terraform Advantages:**
- ✅ Infrastructure as Code
- ✅ Consistent deployments
- ✅ Version-controlled scaling policies
- ✅ Automated launch template updates

Khuyến nghị: Console cho learning, Terraform cho production.
{{% /notice %}}

---

## 2. EKS Node Group Terraform Configuration

### 2.1. Main Node Group Resource

**Tạo file `aws/infra/eks-nodegroup.tf`:**

```hcl
# Data source for EKS cluster
data "aws_eks_cluster" "main" {
  name = aws_eks_cluster.main.name
}

# IAM Role for EKS Node Group
resource "aws_iam_role" "eks_nodegroup" {
  name = "${var.project_name}-${var.environment}-nodegroup-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-nodegroup-role"
    Type = "iam-role"
    Service = "eks-nodegroup"
  })
}

# IAM Role Policy Attachments for Node Group
resource "aws_iam_role_policy_attachment" "eks_nodegroup_worker_node_policy" {
  role       = aws_iam_role.eks_nodegroup.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_nodegroup_cni_policy" {
  role       = aws_iam_role.eks_nodegroup.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "eks_nodegroup_ecr_readonly_policy" {
  role       = aws_iam_role.eks_nodegroup.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "eks_nodegroup_cloudwatch_policy" {
  role       = aws_iam_role.eks_nodegroup.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# EKS Managed Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-nodegroup"
  node_role_arn   = aws_iam_role.eks_nodegroup.arn
  subnet_ids      = [
    aws_subnet.private["ap-southeast-1a"].id,
    aws_subnet.private["ap-southeast-1b"].id
  ]

  # Instance configuration
  capacity_type  = var.nodegroup_capacity_type
  instance_types = var.nodegroup_instance_types
  disk_size      = var.nodegroup_disk_size

  # Scaling configuration
  scaling_config {
    desired_size = var.nodegroup_desired_size
    max_size     = var.nodegroup_max_size
    min_size     = var.nodegroup_min_size
  }

  # Update configuration
  update_config {
    max_unavailable = var.nodegroup_max_unavailable
  }

  # Labels
  labels = var.nodegroup_labels

  # Taints (if any)
  dynamic "taint" {
    for_each = var.nodegroup_taints
    content {
      key    = taint.value.key
      value  = taint.value.value
      effect = taint.value.effect
    }
  }

  # Launch template (optional)
  dynamic "launch_template" {
    for_each = var.enable_launch_template ? [1] : []
    content {
      id      = aws_launch_template.eks_nodegroup[0].id
      version = aws_launch_template.eks_nodegroup[0].latest_version
    }
  }

  # Ensure dependencies are met
  depends_on = [
    aws_iam_role_policy_attachment.eks_nodegroup_worker_node_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cni_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_ecr_readonly_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cloudwatch_policy,
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-nodegroup"
    Type = "eks-nodegroup"
  })
}

# Launch Template for advanced configuration (optional)
resource "aws_launch_template" "eks_nodegroup" {
  count = var.enable_launch_template ? 1 : 0
  
  name_prefix   = "${var.project_name}-${var.environment}-nodegroup-"
  image_id      = var.nodegroup_ami_id
  instance_type = var.nodegroup_instance_types[0]

  vpc_security_group_ids = [aws_security_group.eks_nodes.id]

  user_data = base64encode(templatefile("${path.module}/userdata.sh", {
    cluster_name        = aws_eks_cluster.main.name
    cluster_endpoint    = aws_eks_cluster.main.endpoint
    cluster_ca          = aws_eks_cluster.main.certificate_authority[0].data
    bootstrap_arguments = var.nodegroup_bootstrap_arguments
  }))

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.project_name}-${var.environment}-worker-node"
      Type = "eks-worker-node"
    })
  }

  tag_specifications {
    resource_type = "volume"
    tags = merge(var.common_tags, {
      Name = "${var.project_name}-${var.environment}-worker-node-volume"
      Type = "ebs-volume"
    })
  }

  lifecycle {
    create_before_destroy = true
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-nodegroup-lt"
    Type = "launch-template"
  })
}

# Security Group for EKS Worker Nodes
resource "aws_security_group" "eks_nodes" {
  name_prefix = "${var.project_name}-${var.environment}-eks-nodes-"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Worker node port range"
    from_port   = 1025
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  ingress {
    description = "Allow HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-nodes-sg"
    Type = "security-group"
  })
}

# Security Group Rule: Allow cluster control plane to communicate with nodes
resource "aws_security_group_rule" "cluster_to_nodes" {
  type                     = "ingress"
  from_port                = 1025
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.eks_control_plane.id
  security_group_id        = aws_security_group.eks_nodes.id
}

# Security Group Rule: Allow nodes to communicate with cluster control plane
resource "aws_security_group_rule" "nodes_to_cluster" {
  type                     = "egress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.eks_control_plane.id
  security_group_id        = aws_security_group.eks_nodes.id
}
```

### 2.2. Additional Variables

**Thêm vào `aws/infra/variables.tf`:**

```hcl
# EKS Node Group Configuration
variable "nodegroup_capacity_type" {
  description = "Type of capacity associated with the EKS Node Group. Valid values: ON_DEMAND, SPOT"
  type        = string
  default     = "ON_DEMAND"
}

variable "nodegroup_instance_types" {
  description = "List of instance types associated with the EKS Node Group"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "nodegroup_disk_size" {
  description = "Disk size in GiB for worker nodes"
  type        = number
  default     = 20
}

variable "nodegroup_desired_size" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 2
}

variable "nodegroup_max_size" {
  description = "Maximum number of worker nodes"
  type        = number
  default     = 4
}

variable "nodegroup_min_size" {
  description = "Minimum number of worker nodes"
  type        = number
  default     = 1
}

variable "nodegroup_max_unavailable" {
  description = "Maximum number of nodes unavailable at once during version update"
  type        = number
  default     = 1
}

variable "nodegroup_labels" {
  description = "Key-value map of Kubernetes labels for nodes"
  type        = map(string)
  default = {
    "nodegroup-type" = "primary"
    "environment"    = "dev"
  }
}

variable "nodegroup_taints" {
  description = "List of Kubernetes taints to apply to nodes"
  type = list(object({
    key    = string
    value  = string
    effect = string
  }))
  default = []
}

# Launch Template Options
variable "enable_launch_template" {
  description = "Enable launch template for advanced node configuration"
  type        = bool
  default     = false
}

variable "nodegroup_ami_id" {
  description = "AMI ID for EKS worker nodes (if using launch template)"
  type        = string
  default     = ""
}

variable "nodegroup_bootstrap_arguments" {
  description = "Additional arguments for EKS bootstrap script"
  type        = string
  default     = ""
}
```

### 2.3. Environment-specific Configuration

**Cập nhật `aws/terraform.tfvars`:**

```hcl
# EKS Node Group configuration
nodegroup_capacity_type     = "ON_DEMAND"
nodegroup_instance_types    = ["t3.medium"]
nodegroup_disk_size        = 20
nodegroup_desired_size     = 2
nodegroup_max_size         = 4
nodegroup_min_size         = 1
nodegroup_max_unavailable  = 1

nodegroup_labels = {
  "nodegroup-type" = "primary"
  "environment"    = "dev"
  "project"        = "mlops-retail-forecast"
}

# Advanced configuration
enable_launch_template = false
nodegroup_bootstrap_arguments = ""
```

### 2.4. User Data Script (Optional)

**Tạo file `aws/infra/userdata.sh`:**

```bash
#!/bin/bash

# Update packages
yum update -y

# Install additional packages
yum install -y amazon-cloudwatch-agent

# Configure CloudWatch agent
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << EOF
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
            "log_group_name": "/aws/eks/${cluster_name}/worker-nodes",
            "log_stream_name": "{instance_id}/messages"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "EKS/WorkerNodes",
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

# Bootstrap EKS worker node
/etc/eks/bootstrap.sh ${cluster_name} ${bootstrap_arguments}
```

## 3. Mixed Instance Types & Spot Optimization

### 3.1. Cost Optimization Strategy

**Mixed Instance Types** cho phép kết hợp On-Demand và Spot instances để tối ưu chi phí:

```
Cost Comparison (Monthly - ap-southeast-1):
├── On-Demand Only (2x t3.medium): $60.00/month
├── Spot Only (2x t3.medium): ~$18.00/month (70% savings)
└── Mixed (1 On-Demand + 1 Spot): ~$39.00/month (35% savings)

💰 Spot Savings: Up to 70% cost reduction
⚠️ Risk: Spot interruption (2-minute notice)
```

### 3.2. Multiple Node Groups Configuration

**File: `aws/infra/eks-nodegroups-mixed.tf`**

```hcl
# On-Demand Node Group (Core Services)
resource "aws_eks_node_group" "on_demand" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-ondemand"
  node_role_arn   = aws_iam_role.eks_nodegroup.arn
  subnet_ids      = aws_subnet.private[*].id

  # On-Demand configuration
  capacity_type  = "ON_DEMAND"
  instance_types = var.ondemand_instance_types
  disk_size      = var.nodegroup_disk_size

  # Conservative scaling for core services
  scaling_config {
    desired_size = var.ondemand_desired_size
    max_size     = var.ondemand_max_size
    min_size     = var.ondemand_min_size
  }

  # Update configuration
  update_config {
    max_unavailable = 1
  }

  # Labels for core services
  labels = {
    "nodegroup-type" = "on-demand"
    "workload-type"  = "core"
    "environment"    = var.environment
  }

  # Taints to ensure only core services run here
  taint {
    key    = "node-type"
    value  = "on-demand"
    effect = "NO_SCHEDULE"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_nodegroup_worker_node_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cni_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_ecr_readonly_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cloudwatch_policy,
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ondemand-nodegroup"
    Type = "eks-nodegroup-ondemand"
  })
}

# Spot Node Group (Batch Workloads)
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-spot"
  node_role_arn   = aws_iam_role.eks_nodegroup.arn
  subnet_ids      = aws_subnet.private[*].id

  # Spot configuration
  capacity_type  = "SPOT"
  instance_types = var.spot_instance_types
  disk_size      = var.nodegroup_disk_size

  # Aggressive scaling for batch workloads
  scaling_config {
    desired_size = var.spot_desired_size
    max_size     = var.spot_max_size
    min_size     = var.spot_min_size
  }

  # Update configuration
  update_config {
    max_unavailable_percentage = 50  # Higher tolerance for spot
  }

  # Labels for batch workloads
  labels = {
    "nodegroup-type" = "spot"
    "workload-type"  = "batch"
    "environment"    = var.environment
  }

  # Taints for spot-tolerant workloads
  taint {
    key    = "node-type"
    value  = "spot"
    effect = "NO_SCHEDULE"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_nodegroup_worker_node_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cni_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_ecr_readonly_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cloudwatch_policy,
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-spot-nodegroup"
    Type = "eks-nodegroup-spot"
  })
}

# General Purpose Node Group (Mixed Workloads)
resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-general"
  node_role_arn   = aws_iam_role.eks_nodegroup.arn
  subnet_ids      = aws_subnet.private[*].id

  # Mixed capacity (On-Demand + Spot)
  capacity_type  = var.general_capacity_type
  instance_types = var.general_instance_types
  disk_size      = var.nodegroup_disk_size

  # Balanced scaling
  scaling_config {
    desired_size = var.general_desired_size
    max_size     = var.general_max_size
    min_size     = var.general_min_size
  }

  # Update configuration
  update_config {
    max_unavailable = 1
  }

  # Labels for general workloads
  labels = {
    "nodegroup-type" = "general"
    "workload-type"  = "general"
    "environment"    = var.environment
  }

  # No taints - accepts all workloads

  depends_on = [
    aws_iam_role_policy_attachment.eks_nodegroup_worker_node_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cni_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_ecr_readonly_policy,
    aws_iam_role_policy_attachment.eks_nodegroup_cloudwatch_policy,
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-general-nodegroup"
    Type = "eks-nodegroup-general"
  })
}
```

### 3.3. Mixed Instance Variables

**Thêm vào `aws/infra/variables.tf`:**

```hcl
# On-Demand Node Group
variable "ondemand_instance_types" {
  description = "Instance types for On-Demand node group"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "ondemand_desired_size" {
  description = "Desired number of On-Demand nodes"
  type        = number
  default     = 1
}

variable "ondemand_max_size" {
  description = "Maximum number of On-Demand nodes"
  type        = number
  default     = 2
}

variable "ondemand_min_size" {
  description = "Minimum number of On-Demand nodes"
  type        = number
  default     = 1
}

# Spot Node Group
variable "spot_instance_types" {
  description = "Instance types for Spot node group"
  type        = list(string)
  default     = ["t3.medium", "t3.large", "m5.large"]  # Multiple types for availability
}

variable "spot_desired_size" {
  description = "Desired number of Spot nodes"
  type        = number
  default     = 2
}

variable "spot_max_size" {
  description = "Maximum number of Spot nodes"
  type        = number
  default     = 6
}

variable "spot_min_size" {
  description = "Minimum number of Spot nodes"
  type        = number
  default     = 0
}

# General Purpose Node Group
variable "general_capacity_type" {
  description = "Capacity type for general node group"
  type        = string
  default     = "ON_DEMAND"  # Can be changed to SPOT for cost optimization
}

variable "general_instance_types" {
  description = "Instance types for general node group"
  type        = list(string)
  default     = ["t3.medium", "t3.large"]
}

variable "general_desired_size" {
  description = "Desired number of general nodes"
  type        = number
  default     = 2
}

variable "general_max_size" {
  description = "Maximum number of general nodes"
  type        = number
  default     = 4
}

variable "general_min_size" {
  description = "Minimum number of general nodes"
  type        = number
  default     = 1
}
```

### 3.4. Environment-specific Configuration

**Cập nhật `aws/terraform.tfvars`:**

```hcl
# On-Demand Node Group (Core Services)
ondemand_instance_types = ["t3.medium"]
ondemand_desired_size   = 1
ondemand_max_size       = 2
ondemand_min_size       = 1

# Spot Node Group (Batch Workloads)
spot_instance_types = ["t3.medium", "t3.large", "m5.large"]
spot_desired_size   = 2
spot_max_size       = 6
spot_min_size       = 0

# General Purpose Node Group
general_capacity_type   = "ON_DEMAND"  # Change to "SPOT" for more savings
general_instance_types  = ["t3.medium", "t3.large"]
general_desired_size    = 2
general_max_size        = 4
general_min_size        = 1
```

### 3.5. Workload Deployment Strategies

**Core Services (On-Demand Nodes):**

```yaml
# deployment-core.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-api-core
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inference-api
      tier: core
  template:
    metadata:
      labels:
        app: inference-api
        tier: core
    spec:
      # Tolerate on-demand taint
      tolerations:
      - key: "node-type"
        operator: "Equal"
        value: "on-demand"
        effect: "NoSchedule"
      
      # Prefer on-demand nodes
      nodeSelector:
        nodegroup-type: "on-demand"
      
      containers:
      - name: api
        image: your-account.dkr.ecr.ap-southeast-1.amazonaws.com/inference-api:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

**Batch Workloads (Spot Nodes):**

```yaml
# deployment-batch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: batch-processor
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      # Tolerate spot interruptions
      tolerations:
      - key: "node-type"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      
      # Prefer spot nodes
      nodeSelector:
        nodegroup-type: "spot"
      
      containers:
      - name: processor
        image: your-account.dkr.ecr.ap-southeast-1.amazonaws.com/batch-processor:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
```

## 4. Auto-Scaling & HPA Integration

### 4.1. Cluster Autoscaler Installation

**File: `aws/k8s/cluster-autoscaler.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/mlops-retail-forecast-dev-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: ap-southeast-1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-cluster-autoscaler
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["events", "endpoints"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/status"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["endpoints"]
  resourceNames: ["cluster-autoscaler"]
  verbs: ["get", "update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch", "list", "get", "update"]
- apiGroups: [""]
  resources: ["pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["extensions"]
  resources: ["replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["policy"]
  resources: ["poddisruptionbudgets"]
  verbs: ["watch", "list"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "csinodes"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
- kind: ServiceAccount
  name: cluster-autoscaler
  namespace: kube-system
```

### 4.2. Horizontal Pod Autoscaler (HPA)

**File: `aws/k8s/hpa-inference.yaml`**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-api-hpa
  namespace: mlops-retail-forecast
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
```

## 5. Node Group Deployment

### 3.1. Terraform Deployment

```bash
# Navigate to Terraform directory
cd aws/infra

# Initialize Terraform (if not done)
terraform init

# Plan deployment
terraform plan -var-file="terraform.tfvars"

# Apply configuration
terraform apply -var-file="terraform.tfvars" -auto-approve
```

### 3.2. Verify Deployment

```bash
# Get EKS cluster info
aws eks describe-cluster --name mlops-retail-forecast-dev-cluster --region ap-southeast-1

# Get node group info
aws eks describe-nodegroup \
  --cluster-name mlops-retail-forecast-dev-cluster \
  --nodegroup-name mlops-retail-forecast-dev-nodegroup \
  --region ap-southeast-1

# Update kubectl config
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name mlops-retail-forecast-dev-cluster

# Verify nodes
kubectl get nodes
kubectl get nodes -o wide
```

---

## 4. Node Management và Troubleshooting

### 4.1. Scaling Operations

```bash
# Manual scaling via AWS CLI
aws eks update-nodegroup-config \
  --cluster-name mlops-retail-forecast-dev-cluster \
  --nodegroup-name mlops-retail-forecast-dev-nodegroup \
  --scaling-config minSize=1,maxSize=6,desiredSize=3 \
  --region ap-southeast-1

# Scale via kubectl (temporary)
kubectl scale deployment <deployment-name> --replicas=0
kubectl scale deployment <deployment-name> --replicas=3
```

### 4.2. Node Health Check

```bash
# Check node conditions
kubectl describe nodes | grep -A 10 "Conditions:"

# Check node capacity
kubectl describe nodes | grep -A 5 "Capacity\|Allocatable"

# Get node metrics (requires metrics-server)
kubectl top nodes
```

### 4.3. Troubleshooting Common Issues

**Node NotReady:**
```bash
# Check node events
kubectl describe node <node-name>

# Check kubelet logs
kubectl logs -n kube-system -l k8s-app=aws-node

# Check system pods
kubectl get pods -n kube-system -o wide
```

**Pod Scheduling Issues:**
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check node taints và tolerations
kubectl describe nodes | grep -A 3 "Taints:"

# Check resource constraints
kubectl describe nodes | grep -A 5 "Allocated resources:"
```

---

## 5. Best Practices

### 5.1. Instance Type Selection

```hcl
# Cost-optimized (Development)
nodegroup_instance_types = ["t3.medium", "t3.large"]

# Balanced (Staging)
nodegroup_instance_types = ["m5.large", "m5.xlarge"]

# Performance-optimized (Production)
nodegroup_instance_types = ["c5.xlarge", "c5.2xlarge"]

# Mixed instances (Cost optimization)
nodegroup_instance_types = ["t3.medium", "t3.large", "m5.large"]
```

### 5.2. Scaling Strategy

```hcl
# Conservative scaling
nodegroup_min_size     = 2
nodegroup_desired_size = 2
nodegroup_max_size     = 4

# Aggressive scaling
nodegroup_min_size     = 1
nodegroup_desired_size = 3
nodegroup_max_size     = 10
```

### 5.3. Security Hardening

```hcl
# Use custom AMI with hardening
nodegroup_ami_id = "ami-xxxxxxxxx"  # Custom hardened AMI

# Enable IMDSv2
user_data = base64encode(templatefile("userdata.sh", {
  imds_v2_required = true
}))
```

---

## 6. Monitoring và Alerting

### 6.1. CloudWatch Metrics

```bash
# Node CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/EKS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=mlops-retail-forecast-dev-cluster \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --period 3600 \
  --statistics Average

# Node memory utilization
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=InstanceId,Value=i-xxxxxxxxx \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --period 3600 \
  --statistics Average
```

### 6.2. Kubernetes Monitoring

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Get resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

---

## 👉 Kết quả Task 5

Sau Task 5, bạn sẽ có EKS Node Group production-ready, auto-scaling, bảo mật qua IAM/IRSA, và sẵn sàng để deploy inference API trong Task 10 (Kubernetes Deployment).

### ✅ Deliverables Completed

- **EKS Managed Node Groups**: Multiple node groups (On-Demand, Spot, General) trải trên ≥2 AZ
- **Auto-Scaling Policies**: Cluster Autoscaler + HPA integration hoạt động
- **Cost Optimization**: Mixed instance types với Spot instances (up to 70% savings)
- **IAM Integration**: Node roles với quyền ECR, S3, CloudWatch qua VPC Endpoints
- **Multi-AZ Deployment**: High availability với proper subnet distribution
- **Workload Isolation**: Node taints/tolerations cho core vs batch workloads

### Architecture Achieved

```
✅ Node Groups: 3 groups (On-Demand, Spot, General)
✅ Instance Types: t3.medium, t3.large, m5.large (mixed for availability)
✅ Auto-Scaling: 1-10 nodes total với intelligent scaling policies
✅ Cost Optimization: 35-70% savings với Spot instances
✅ High Availability: Multi-AZ deployment trong private subnets
✅ Security: Least privilege IAM + Security Groups + VPC Endpoints
```

### Cost Summary

| Node Group | Instance Type | Capacity | Monthly Cost | Savings |
|------------|---------------|----------|--------------|---------|
| **On-Demand** | 1x t3.medium | Core Services | $30.00 | - |
| **Spot** | 2x t3.medium | Batch Workloads | $18.00 | 70% |
| **General** | 2x t3.medium | Mixed Workloads | $60.00 | - |
| **Total** | **5 nodes** | **Mixed** | **$108.00** | **$42.00 saved** |

### Verification Commands

```bash
# Check node groups status
aws eks describe-nodegroup --cluster-name mlops-retail-forecast-dev-cluster --nodegroup-name mlops-retail-forecast-dev-ondemand
aws eks describe-nodegroup --cluster-name mlops-retail-forecast-dev-cluster --nodegroup-name mlops-retail-forecast-dev-spot

# Verify nodes in cluster
kubectl get nodes -o wide
kubectl get nodes --show-labels

# Check auto-scaling
kubectl get hpa -A
kubectl get deployment cluster-autoscaler -n kube-system

# Test workload scheduling
kubectl describe nodes | grep -A 5 "Taints\|Labels"
```

{{% notice success %}}
**🎯 Ready for Next Tasks:**

EKS Node Groups foundation đã sẵn sàng để deploy workloads:
- ✅ **Task 6**: ECR repository setup cho container images
- ✅ **Task 7**: Build và push inference API container
- ✅ **Task 8**: S3 data storage integration
- ✅ **Task 10**: Deploy inference API lên EKS cluster với proper node selection
- ✅ **Task 11**: Load balancing và ingress configuration
{{% /notice %}}

{{% notice warning %}}
**🔐 Production Considerations:**

- **Spot Interruptions**: Monitor spot instance interruption notices và implement graceful shutdown
- **Node Taints**: Properly configure tolerations cho workloads để schedule trên correct nodes
- **Resource Limits**: Set appropriate CPU/memory requests và limits cho all pods
- **Pod Disruption Budgets**: Implement PDBs cho critical services để maintain availability
- **Cluster Autoscaler**: Monitor scaling events và adjust policies based on workload patterns
- **Security Updates**: Regularly update node AMIs và Kubernetes versions
- **Monitoring**: Setup comprehensive monitoring cho node health, resource usage, và cost tracking
{{% /notice %}}