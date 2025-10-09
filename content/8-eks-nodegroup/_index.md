---
title: "EKS Node Management"
date: 2024-01-01T00:00:00+07:00
weight: 8
chapter: false
pre: "<b>8. </b>"
---

# 🧩 Task 8 – Node Management (Managed Node Group)

## 🎯 Mục tiêu

Tạo và quản lý EKS Managed Node Group để cung cấp tài nguyên compute (EC2) cho các workload chạy trên Kubernetes như:
- API Retail Prediction (FastAPI)
- Batch job / training job nhỏ (nếu cần)

→ Đảm bảo cụm ổn định, tự động scale, tối ưu chi phí, và phù hợp với điều kiện Free Tier hoặc Spot Instance.

## 📥 Input

- **EKS Cluster** từ Task 7 (mlops-retail-cluster)
- **Hybrid VPC Architecture** từ Task 5 (private workload subnets)
- **IAM Role cho Node Group** từ Task 2 (IAM Roles)
- **ECR Repositories** từ Task 6 (mlops/retail-api container image)

## 📌 Kiến trúc Node Group

{{< mermaid >}}
graph TB
    subgraph "EKS Cluster"
        CP[Control Plane<br/>Managed by AWS]
    end
    
    subgraph "Hybrid VPC (Private Subnets)"
        subgraph "Free Tier Option"
            direction TB
            N1[Worker Node 1<br/>t2.micro<br/>FREE]
            N2[Worker Node 2<br/>t2.micro<br/>FREE]
        end
        
        subgraph "Production Option"
            direction TB
            S1[Worker Node 1<br/>t3.small (Spot)<br/>$0.012/h]
            S2[Worker Node 2<br/>t3.small (Spot)<br/>$0.012/h]
        end
    end
    
    subgraph "AWS Services"
        ECR[ECR Registry<br/>Container Images]
        CW[CloudWatch<br/>Logs & Metrics]
        S3[S3 Bucket<br/>Model Artifacts]
    end
    
    CP --> N1 & N2
    CP --> S1 & S2
    
    N1 & N2 & S1 & S2 --> ECR
    N1 & N2 & S1 & S2 --> CW
    N1 & N2 & S1 & S2 --> S3
{{< /mermaid >}}

## 1. Tạo Node Group

### 1.1. Node Group Options Comparison

| Parameter | Free Tier Option | Production Option |
|-----------|------------------|-------------------|
| **Instance Type** | t2.micro | t3.small (Spot) |
| **vCPU** | 1 vCPU | 2 vCPU |
| **Memory** | 1 GB RAM | 2 GB RAM |
| **Cost** | FREE (750h/month) | ~$0.012 USD/h (70% savings) |
| **Use Case** | Demo, test, video rubric | Production API, real workloads |
| **Auto-scaling** | 1-4 nodes | 1-10 nodes |
| **Node Storage** | 20GB gp3 | 20GB gp3 |

### 1.2. Create Node Group via Console

1. **Navigate to EKS Console:**
   - AWS Console → EKS → mlops-retail-cluster → Compute → Add node group

2. **Name & IAM Role:**
   ```
   Name: mlops-retail-nodegroup
   Node IAM role: mlops-eks-node-role (từ Task 2)
   ```

3. **Compute Configuration (Free Tier):**
   ```
   AMI type: Amazon Linux 2
   Capacity type: On-Demand
   Instance type: t2.micro
   Disk size: 20GB
   ```

   **Compute Configuration (Production):**
   ```
   AMI type: Amazon Linux 2
   Capacity type: Spot
   Instance type: t3.small
   Disk size: 20GB
   ```

4. **Scaling Configuration:**
   ```
   Desired size: 2
   Minimum size: 1
   Maximum size: 10
   ```

5. **Networking Configuration:**
   ```
   Subnets: Select private workload subnets from Task 5
   ```

6. **Review & Create Node Group**

### 1.3. Create Node Group via eksctl (CLI)

**Create file `scripts/create-nodegroup.sh`:**

```bash
#!/bin/bash

# Choose appropriate node group type by uncommenting ONE of these blocks:

# === FREE TIER OPTION (t2.micro - 0$/month) ===
eksctl create nodegroup \
  --cluster mlops-retail-cluster \
  --name mlops-retail-nodegroup-free \
  --node-type t2.micro \
  --nodes-min 1 --nodes-max 4 --nodes 2 \
  --node-volume-size 20 \
  --node-volume-type gp3 \
  --node-private-networking \
  --managed

# === PRODUCTION OPTION (t3.small Spot - ~$0.012/h) ===
# eksctl create nodegroup \
#   --cluster mlops-retail-cluster \
#   --name mlops-retail-nodegroup \
#   --instance-types t3.small \
#   --nodes-min 1 --nodes-max 10 --nodes 2 \
#   --node-volume-size 20 \
#   --node-volume-type gp3 \
#   --node-private-networking \
#   --managed --spot
```

**Run the script:**
```bash
chmod +x scripts/create-nodegroup.sh
./scripts/create-nodegroup.sh
```

## 2. Cấu hình Auto Scaling

### 2.1. Install Cluster Autoscaler

**Create file `k8s/cluster-autoscaler.yaml`:**

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-cluster-autoscaler-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
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
  resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
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
  resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "patch"]
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create"]
- apiGroups: ["coordination.k8s.io"]
  resourceNames: ["cluster-autoscaler"]
  resources: ["leases"]
  verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
- kind: ServiceAccount
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.23.0
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
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/mlops-retail-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        volumeMounts:
        - name: ssl-certs
          mountPath: /etc/ssl/certs/ca-certificates.crt
          readOnly: true
      volumes:
      - name: ssl-certs
        hostPath:
          path: "/etc/ssl/certs/ca-bundle.crt"
```

**Deploy Cluster Autoscaler:**

```bash
# Replace ACCOUNT_ID with your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed "s/ACCOUNT_ID/$ACCOUNT_ID/g" k8s/cluster-autoscaler.yaml | kubectl apply -f -

# Verify deployment
kubectl get deployment cluster-autoscaler -n kube-system
```

### 2.2. Configure IAM Role for Cluster Autoscaler

**Create file `scripts/create-cluster-autoscaler-role.sh`:**

```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="mlops-retail-cluster"
REGION="ap-southeast-1"
ROLE_NAME="mlops-cluster-autoscaler-role"
POLICY_NAME="mlops-cluster-autoscaler-policy"

# Get OIDC issuer and account ID
OIDC_ISSUER=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --region $REGION \
    --query "cluster.identity.oidc.issuer" \
    --output text | sed 's|https://||')

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "🔧 Creating IAM role for Cluster Autoscaler..."

# Create trust policy
cat > autoscaler-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_ISSUER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_ISSUER}:sub": "system:serviceaccount:kube-system:cluster-autoscaler",
          "${OIDC_ISSUER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
    --role-name $ROLE_NAME \
    --assume-role-policy-document file://autoscaler-trust-policy.json \
    --description "IRSA role for EKS Cluster Autoscaler"

# Create autoscaler policy
cat > autoscaler-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create and attach policy
aws iam create-policy \
    --policy-name $POLICY_NAME \
    --policy-document file://autoscaler-policy.json \
    --description "Cluster Autoscaler permissions"

aws iam attach-role-policy \
    --role-name $ROLE_NAME \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/${POLICY_NAME}

# Clean up
rm autoscaler-trust-policy.json autoscaler-policy.json

echo "✅ Cluster Autoscaler role created successfully!"
echo "Role ARN: arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"
```

**Run the script:**
```bash
chmod +x scripts/create-cluster-autoscaler-role.sh
./scripts/create-cluster-autoscaler-role.sh
```

### 2.3. Configure Horizontal Pod Autoscaler (HPA)

**Create file `k8s/retail-api-hpa.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-api-hpa
  namespace: mlops-retail-forecast
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-api
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
```

**Deploy HPA:**
```bash
kubectl apply -f k8s/retail-api-hpa.yaml

# Verify HPA
kubectl get hpa -n mlops-retail-forecast
```

### 2.4. Install Metrics Server (if not already installed)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify metrics server
kubectl get deployment metrics-server -n kube-system
```

## 3. IAM & Quyền truy cập

### 3.1. Node Group IAM Role Permissions

**Required Permissions for Node Group:**

```
- AmazonEKSWorkerNodePolicy
- AmazonEKS_CNI_Policy
- AmazonEC2ContainerRegistryReadOnly
- AmazonS3ReadOnlyAccess
```

**Create file `scripts/update-node-role.sh`:**

```bash
#!/bin/bash

# Configuration
NODE_ROLE_NAME="mlops-eks-node-role"

echo "🔧 Updating node group IAM role permissions..."

# Attach required policies
aws iam attach-role-policy \
    --role-name $NODE_ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
    --role-name $NODE_ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
    --role-name $NODE_ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
    --role-name $NODE_ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

echo "✅ Node role permissions updated!"
```

### 2.1. Cost-Optimized Multi-Node Group Strategy

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Tìm EKS cluster** đã tạo ở Task 4
2. **Tìm private subnets** từ Task 2 để deploy nodes
3. **Tạo 2 node groups** với strategies khác nhau:
   - **On-Demand**: Stable, expensive, cho core services
   - **Spot**: 70% cheaper, có thể bị interrupt, cho batch workloads
4. **Setup workload isolation** với taints/tolerations

**Kết quả:** Cost-optimized node groups với intelligent workload scheduling
{{% /notice %}}

**File: `aws/infra/eks-nodegroups-advanced.tf`**

```hcl
# BƯỚC 1: Tìm existing infrastructure (không tạo mới)
data "aws_eks_cluster" "main" {
  name = "${var.project_name}-${var.environment}-cluster"  # EKS từ Task 4
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["private-subnet"]  # Private subnets từ Task 2
  }
}

# Reference IAM role từ Task 3 (hoặc Console-created)
data "aws_iam_role" "nodegroup" {
  name = "${var.project_name}-${var.environment}-nodegroup-role"  # IAM từ Task 3
}

# BƯỚC 2: On-Demand Node Group cho Core Services (Stable, Expensive)
resource "aws_eks_node_group" "on_demand" {
  cluster_name    = data.aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-ondemand"
  node_role_arn   = data.aws_iam_role.nodegroup.arn
  subnet_ids      = data.aws_subnets.private.ids

  # On-Demand: Stable nhưng expensive, cho critical workloads
  capacity_type  = "ON_DEMAND"
  instance_types = var.ondemand_instance_types
  disk_size      = 20

  # Conservative scaling: ít nodes nhưng stable
  scaling_config {
    desired_size = var.ondemand_desired_size  # Default: 1
    max_size     = var.ondemand_max_size      # Default: 2
    min_size     = var.ondemand_min_size      # Default: 1
  }

  # Labels: Kubernetes scheduler sẽ dùng để place pods
  labels = {
    "nodegroup-type" = "on-demand"
    "workload-type"  = "core"
    "environment"    = var.environment
  }

  # Taints: Chỉ pods có tolerations mới schedule được lên đây
  taint {
    key    = "node-type"
    value  = "on-demand"
    effect = "NO_SCHEDULE"  # Block pods không có toleration
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ondemand-nodegroup"
    Type = "eks-nodegroup-ondemand"
    Purpose = "core-services"
  })
}

# BƯỚC 3: Spot Node Group cho Batch Workloads (70% cost savings)
resource "aws_eks_node_group" "spot" {
  cluster_name    = data.aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-spot"
  node_role_arn   = data.aws_iam_role.nodegroup.arn
  subnet_ids      = data.aws_subnets.private.ids

  # Spot: 70% cheaper nhưng có thể bị AWS interrupt bất kỳ lúc nào
  capacity_type  = "SPOT"
  instance_types = var.spot_instance_types  # Multiple types tăng availability
  disk_size      = 20

  # Aggressive scaling: nhiều nodes cho batch processing
  scaling_config {
    desired_size = var.spot_desired_size  # Default: 2
    max_size     = var.spot_max_size      # Default: 6
    min_size     = var.spot_min_size      # Default: 0 (có thể scale về 0)
  }

  # Update config: tolerate higher disruption cho spot instances
  update_config {
    max_unavailable_percentage = 50  # 50% nodes có thể unavailable cùng lúc
  }

  # Labels cho batch workloads
  labels = {
    "nodegroup-type" = "spot"
    "workload-type"  = "batch"
    "environment"    = var.environment
  }

  # Taints: Chỉ batch workloads có toleration mới chạy ở đây
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

{{% notice tip %}}
**🔍 Workload Scheduling Strategy:**
1. **Core Services** → On-Demand nodes (stable, expensive)
2. **Batch Workloads** → Spot nodes (70% cheaper, interruptible)
3. **Tolerations** → Cho phép pods chạy trên tainted nodes
4. **Node Selectors** → Force pods chạy trên specific node types

**Kết quả:** Intelligent cost optimization với workload isolation
{{% /notice %}}

**Core Services (On-Demand Nodes - Stable, Critical):**

```yaml
# deployment-core.yaml - Chạy trên On-Demand nodes (stable)
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
      # QUAN TRỌNG: Tolerations cho phép pod chạy trên tainted On-Demand nodes
      tolerations:
      - key: "node-type"
        operator: "Equal"
        value: "on-demand"
        effect: "NoSchedule"  # Match với taint trong Terraform
      
      # Force pod chỉ chạy trên On-Demand nodes
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

**Batch Workloads (Spot Nodes - 70% Cheaper, Interruptible):**

```yaml
# deployment-batch.yaml - Chạy trên Spot nodes (70% cheaper)
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
      # QUAN TRỌNG: Tolerations cho phép pod chạy trên tainted Spot nodes
      tolerations:
      - key: "node-type"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"  # Match với taint trong Terraform
      
      # Force pod chỉ chạy trên Spot nodes (70% cheaper)
      nodeSelector:
        nodegroup-type: "spot"
      
      # Graceful shutdown: Spot instances có thể bị AWS interrupt với 2 phút notice
      terminationGracePeriodSeconds: 120
      
      containers:
      - name: processor
        image: ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/batch-processor:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
        
        # Handle spot interruptions gracefully
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Grace period
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

{{% notice success %}}
**🚀 Advanced Deployment Process:**

**Bước 1:** Terraform tìm EKS cluster từ Task 4  
**Bước 2:** Tạo multiple node groups với cost optimization  
**Bước 3:** Configure workload isolation với taints/tolerations  
**Bước 4:** Verify nodes và test workload scheduling  

**Time required:** ~10-15 phút cho multiple node groups
{{% /notice %}}

```bash
# BƯỚC 1: Deploy advanced multi-node group strategy
cd aws/infra

# BƯỚC 2: Plan deployment (xem Terraform sẽ tạo gì)
terraform plan -target=aws_eks_node_group.on_demand \
               -target=aws_eks_node_group.spot \
               -var-file="terraform.tfvars"

# BƯỚC 3: Apply node groups
terraform apply -target=aws_eks_node_group.on_demand \
                -target=aws_eks_node_group.spot \
                -var-file="terraform.tfvars"

# BƯỚC 4: Verify multiple node groups created
kubectl get nodes --show-labels | grep nodegroup-type

# Expected output:
# node1    Ready    <none>   5m    v1.28.3   nodegroup-type=on-demand,workload-type=core
# node2    Ready    <none>   5m    v1.28.3   nodegroup-type=spot,workload-type=batch
# node3    Ready    <none>   5m    v1.28.3   nodegroup-type=spot,workload-type=batch
```

**Expected Apply Output:**
```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Resources Created:
✅ aws_eks_node_group.on_demand (1 node, t3.medium, On-Demand)
✅ aws_eks_node_group.spot (2 nodes, t3.medium, Spot - 70% cheaper)

Cost Optimization:
💰 On-Demand: $30/month (stable, core services)
💰 Spot: $18/month (vs $60 On-Demand - 70% savings)
💰 Total: $48/month (vs $90 all On-Demand - 47% savings)
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