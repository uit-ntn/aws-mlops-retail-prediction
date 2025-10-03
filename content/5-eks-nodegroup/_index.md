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

![Create Node Group Role](../images/05-eks-nodegroup/01-create-nodegroup-role.png)

2. **Select Trusted Entity:**
   ```
   Trusted entity type: AWS service
   Service or use case: EC2
   ```

![Select Trusted Entity](../images/05-eks-nodegroup/02-select-trusted-entity.png)

3. **Attach Required Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy  
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```

![Attach Policies](../images/05-eks-nodegroup/03-attach-policies.png)

4. **Role Configuration:**
   ```
   Role name: mlops-retail-forecast-dev-nodegroup-role
   Description: IAM role for EKS managed node group
   ```

![Role Configuration](../images/05-eks-nodegroup/04-role-configuration.png)

### 1.2. Node Group Creation via Console

1. **Navigate to EKS Cluster:**
   - EKS Console → Clusters
   - Chọn cluster: `mlops-retail-forecast-dev-cluster`
   - Click "Compute" tab
   - Chọn "Add node group"

![Add Node Group](../images/05-eks-nodegroup/05-add-nodegroup.png)

2. **Node Group Configuration:**
   ```
   Name: mlops-retail-forecast-dev-nodegroup
   Node IAM role: mlops-retail-forecast-dev-nodegroup-role
   Kubernetes labels:
     - nodegroup-type: primary
     - environment: dev
   Kubernetes taints: None
   ```

![Node Group Config](../images/05-eks-nodegroup/06-nodegroup-config.png)

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

![Compute Scaling](../images/05-eks-nodegroup/07-compute-scaling.png)

4. **Node Group Networking:**
   ```
   Subnets:
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1a
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1b
   
   Configure SSH access:
   ⬜ Enable SSH access (optional for debugging)
   ```

![Networking Config](../images/05-eks-nodegroup/08-networking-config.png)

5. **Advanced Options:**
   ```
   User data: (Leave empty for standard AMI)
   
   EC2 tags:
   - Name: mlops-retail-forecast-dev-worker-node
   - Environment: dev
   - Project: mlops-retail-forecast
   - NodeGroup: primary
   ```

![Advanced Options](../images/05-eks-nodegroup/09-advanced-options.png)

### 1.3. Node Group Verification

1. **Check Node Group Status:**
   - EKS Console → Cluster → Compute tab
   - Verify status: "Active"
   - Check node count: 2/2 running

![Node Group Status](../images/05-eks-nodegroup/10-nodegroup-status.png)

2. **EC2 Instances Verification:**
   - Navigate to EC2 Console
   - Filter by tag: `aws:eks:cluster-name = mlops-retail-forecast-dev-cluster`
   - Verify 2 instances running

![EC2 Instances](../images/05-eks-nodegroup/11-ec2-instances.png)

3. **Auto Scaling Group:**
   - Navigate to EC2 Auto Scaling
   - Find ASG: `eks-mlops-retail-forecast-dev-nodegroup-*`
   - Verify desired/min/max capacity

![Autoscaling Group](../images/05-eks-nodegroup/12-autoscaling-group.png)

### 1.4. Kubernetes Nodes Verification

1. **Configure kubectl Access:**
   ```bash
   # Update kubeconfig
   aws eks update-kubeconfig --region ap-southeast-1 --name mlops-retail-forecast-dev-cluster
   
   # Verify cluster access
   kubectl cluster-info
   ```

![Kubectl Config](../images/05-eks-nodegroup/13-kubectl-config.png)

2. **Check Node Status:**
   ```bash
   # List all nodes
   kubectl get nodes
   
   # Get detailed node information
   kubectl get nodes -o wide
   
   # Describe specific node
   kubectl describe node <node-name>
   ```

![Kubectl Nodes](../images/05-eks-nodegroup/14-kubectl-nodes.png)

3. **Verify Node Labels and Capacity:**
   ```bash
   # Check node labels
   kubectl get nodes --show-labels
   
   # Check node capacity and allocatable resources
   kubectl describe nodes | grep -A 5 "Capacity\|Allocatable"
   ```

![Node Details](../images/05-eks-nodegroup/15-node-details.png)

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

## 2. Terraform cho Advanced Node Group Strategies

{{% notice info %}}
**💡 Khi nào cần Terraform cho Node Groups:**
- ✅ **Multiple node groups** với different purposes (On-Demand + Spot)
- ✅ **Cost optimization** với mixed instance types và capacity strategies  
- ✅ **Advanced workload isolation** với taints/tolerations
- ✅ **Custom launch templates** với specialized configurations
- ✅ **Production automation** với consistent scaling policies

**Console đủ cho:** Single node group, basic scaling, standard configurations
{{% /notice %}}

### 2.1. Cost-Optimized Multi-Node Group Strategy

**File: `aws/infra/eks-nodegroups-advanced.tf`**

```hcl
# Data sources từ existing infrastructure
data "aws_eks_cluster" "main" {
  name = "${var.project_name}-${var.environment}-cluster"
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

# Reference IAM role từ Task 3 (hoặc Console-created)
data "aws_iam_role" "nodegroup" {
  name = "${var.project_name}-${var.environment}-nodegroup-role"
}

# On-Demand Node Group cho Core Services
resource "aws_eks_node_group" "on_demand" {
  cluster_name    = data.aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-ondemand"
  node_role_arn   = data.aws_iam_role.nodegroup.arn
  subnet_ids      = data.aws_subnets.private.ids

  # On-Demand configuration cho stability
  capacity_type  = "ON_DEMAND"
  instance_types = var.ondemand_instance_types
  disk_size      = 20

  # Conservative scaling cho core workloads
  scaling_config {
    desired_size = var.ondemand_desired_size
    max_size     = var.ondemand_max_size
    min_size     = var.ondemand_min_size
  }

  # Labels cho workload scheduling
  labels = {
    "nodegroup-type" = "on-demand"
    "workload-type"  = "core"
    "environment"    = var.environment
  }

  # Taints để chỉ core services schedule lên đây
  taint {
    key    = "node-type"
    value  = "on-demand"
    effect = "NO_SCHEDULE"
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ondemand-nodegroup"
    Type = "eks-nodegroup-ondemand"
    Purpose = "core-services"
  })
}

# Spot Node Group cho Batch Workloads (70% cost savings)
resource "aws_eks_node_group" "spot" {
  cluster_name    = data.aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-spot"
  node_role_arn   = data.aws_iam_role.nodegroup.arn
  subnet_ids      = data.aws_subnets.private.ids

  # Spot configuration cho cost savings
  capacity_type  = "SPOT"
  instance_types = var.spot_instance_types  # Multiple types cho availability
  disk_size      = 20

  # Aggressive scaling cho batch workloads
  scaling_config {
    desired_size = var.spot_desired_size
    max_size     = var.spot_max_size
    min_size     = var.spot_min_size
  }

  # Update configuration cho spot interruptions
  update_config {
    max_unavailable_percentage = 50  # Higher tolerance
  }

  # Labels cho batch workloads
  labels = {
    "nodegroup-type" = "spot"
    "workload-type"  = "batch"
    "environment"    = var.environment
  }

  # Taints cho spot-tolerant workloads
  taint {
    key    = "node-type"
    value  = "spot"
    effect = "NO_SCHEDULE"
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-spot-nodegroup"
    Type = "eks-nodegroup-spot"
    Purpose = "batch-workloads"
  })
}
```

### 2.2. Cost Optimization Variables

**File: `aws/infra/variables.tf`:**

```hcl
# On-Demand Node Group (Core Services)
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

# Spot Node Group (Batch Workloads - 70% cost savings)
variable "spot_instance_types" {
  description = "Multiple instance types for Spot availability"
  type        = list(string)
  default     = ["t3.medium", "t3.large", "m5.large"]
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
  default     = 0  # Can scale to zero
}
```

### 2.3. Production Configuration

**File: `aws/terraform.tfvars`:**

```hcl
# Cost-optimized node group strategy
ondemand_instance_types = ["t3.medium"]
ondemand_desired_size   = 1
ondemand_max_size       = 2
ondemand_min_size       = 1

# Spot instances với multiple types cho availability
spot_instance_types = ["t3.medium", "t3.large", "m5.large"]
spot_desired_size   = 2
spot_max_size       = 6
spot_min_size       = 0
```

## 3. Workload Scheduling Strategies

### 3.1. Multi-Node Group Cost Analysis

**Cost Comparison (Monthly - ap-southeast-1):**

```
Node Group Strategy Comparison:
├── Single On-Demand Group (3x t3.medium): $90.00/month
├── Mixed Strategy (1 On-Demand + 2 Spot): $48.00/month (47% savings)
└── Advanced Strategy (Multiple groups): $39.00/month (57% savings)

💰 Advanced Multi-Group Benefits:
- Core services: Stable On-Demand nodes (1x t3.medium = $30/month)
- Batch workloads: Cost-effective Spot nodes (2x t3.medium = $18/month)
- Burst capacity: Auto-scaling based on workload type
```

### 3.2. Kubernetes Workload Deployment Examples

**Core Services (On-Demand Nodes):**

```yaml
# deployment-core.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-api-core
  namespace: mlops-retail-forecast
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
      # Schedule on On-Demand nodes chỉ
      tolerations:
      - key: "node-type"
        operator: "Equal"
        value: "on-demand"
        effect: "NoSchedule"
      
      nodeSelector:
        nodegroup-type: "on-demand"
      
      containers:
      - name: api
        image: ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/inference-api:latest
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
  namespace: mlops-retail-forecast
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
      
      nodeSelector:
        nodegroup-type: "spot"
      
      # Graceful shutdown cho spot interruptions
      terminationGracePeriodSeconds: 120
      
      containers:
      - name: processor
        image: ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/batch-processor:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
```

## 4. Auto-Scaling & HPA Integration

{{% notice tip %}}
**💡 Auto-scaling Strategy:**
- ✅ **Cluster Autoscaler**: Scale nodes based on pod demand (Terraform hoặc kubectl)
- ✅ **HPA**: Scale pods based on CPU/memory (kubectl apply)
- ✅ **VPA**: Optimize resource requests (kubectl addon)

**Best practice**: Use both Cluster Autoscaler + HPA cho complete automation
{{% /notice %}}

### 4.1. Essential Cluster Autoscaler (Simplified)

**File: `aws/k8s/cluster-autoscaler-simple.yaml`**

```bash
# Quick Cluster Autoscaler installation
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Patch untuk EKS cluster name
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict":"false"}}}}}'

# Set cluster name
kubectl set env deployment cluster-autoscaler \
  -n kube-system \
  AWS_REGION=ap-southeast-1 \
  CLUSTER_NAME=mlops-retail-forecast-dev-cluster
```

### 4.2. Simple HPA for Inference API

**File: `aws/k8s/hpa-simple.yaml`**

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
```

### 4.3. Apply Auto-scaling

```bash
# Apply HPA
kubectl apply -f aws/k8s/hpa-simple.yaml

# Verify HPA
kubectl get hpa -n mlops-retail-forecast

# Check scaling events
kubectl describe hpa inference-api-hpa -n mlops-retail-forecast
```

## 5. Quick Deployment & Verification

### 5.1. Console Approach (Recommended for Most Cases)

```bash
# After creating node group via Console (Section 1):
# 1. Configure kubectl
aws eks update-kubeconfig --region ap-southeast-1 --name mlops-retail-forecast-dev-cluster

# 2. Verify nodes
kubectl get nodes
kubectl get nodes --show-labels

# 3. Deploy test workload
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### 5.2. Terraform Approach (Advanced Multi-Node Groups)

```bash
# For advanced strategies (Section 2):
cd aws/infra
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"

# Verify multiple node groups
kubectl get nodes --show-labels | grep nodegroup-type
```

### 5.3. Essential Monitoring

```bash
# Check node health
kubectl top nodes

# Check auto-scaling status
kubectl get hpa -A
kubectl get pods -n kube-system | grep autoscaler

# View node events
kubectl get events --sort-by='.lastTimestamp' | grep Node
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