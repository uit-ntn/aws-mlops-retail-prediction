---
title: "VPC / Networking"
date: 2025-08-30T11:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## 🎯 Mục tiêu

Thiết lập hạ tầng mạng multi-AZ VPC trên AWS phục vụ EKS, SageMaker và các dịch vụ phụ trợ, đảm bảo:

- **Cách ly rõ ràng** public subnet (cho ALB, bastion, NAT) và private subnet (cho EKS worker nodes, SageMaker jobs)
- **Bảo mật và kiểm soát** luồng traffic ra/vào với Security Groups theo nguyên tắc least privilege
- **Hỗ trợ private subnet** có thể truy cập Internet khi cần (download package, pull image từ ECR...)
- **Infrastructure as Code** với Terraform modules có thể tái sử dụng

## 📥 Input

- **Terraform module** định nghĩa VPC, subnet, NAT, security group
- **CIDR block** dự kiến (10.0.0.0/16 hoặc tùy chọn khác)
- **Chính sách bảo mật mạng** từ tổ chức (SG, NACL baseline)
- **Environment requirements** (dev/staging/prod) để tối ưu chi phí

## 📌 Các bước chính

1. **Tạo VPC** với CIDR không trùng với on-prem hoặc VPN
2. **Public subnet**: đặt ở ≥2 AZ, chứa ALB, NAT Gateway hoặc NAT Instance
3. **Private subnet**: đặt ở ≥2 AZ, chạy EKS worker node, SageMaker
4. **Routing**:
   - Public subnet → Internet Gateway
   - Private subnet → NAT Gateway (hoặc NAT Instance, tùy lựa chọn)
5. **Security Group**:
   - ALB SG: inbound HTTP/HTTPS, outbound Internet
   - Node SG: inbound từ ALB SG, outbound đến S3/ECR/CloudWatch
6. **Outputs**: VPC ID, subnet ID, security group ID để các task sau (EKS, SageMaker, ALB) có thể dùng

## ✅ Deliverables

- **VPC hoạt động** với public/private subnet phân tách rõ ràng
- **Networking diagram** thể hiện Internet Gateway, NAT, routing
- **Output Terraform** để các module khác tái sử dụng
- **Cost optimization strategy** cho từng environment

## 📊 Acceptance Criteria

- ✅ VPC có ít nhất 2 AZ với cả public và private subnet
- ✅ Private subnet có thể truy cập Internet (ECR, S3, CloudWatch)
- ✅ Security group theo nguyên tắc least privilege
- ✅ CIDR không bị overlap với existing networks
- ✅ Terraform outputs đầy đủ cho integration với tasks khác

## ⚠️ Gotchas

- **CIDR trùng** với mạng nội bộ gây lỗi khi kết nối VPN/DirectConnect
- **NAT Gateway chi phí cao** (~$32/tháng/cái + phí traffic)
- **Multi-AZ mặc định** cần 1 NAT Gateway/AZ → tăng chi phí gấp 2–3 lần
- **Security Group limits** (60 rules/SG, 5 SGs/ENI)
- **Route table limits** (50 routes/table)

## Kiến trúc VPC Networking

### Network Design Overview

```
VPC: 10.0.0.0/16 (65,536 IPs)
├── ap-southeast-1a (AZ-1)
│   ├── Public Subnet: 10.0.1.0/24 (256 IPs)
│   │   ├── NAT Gateway-1
│   │   └── Load Balancer (future)
│   └── Private Subnet: 10.0.101.0/24 (256 IPs)
│       ├── EKS Worker Nodes
│       └── SageMaker Instances
└── ap-southeast-1b (AZ-2)
    ├── Public Subnet: 10.0.2.0/24 (256 IPs)
    │   ├── NAT Gateway-2
    │   └── Load Balancer (future)
    └── Private Subnet: 10.0.102.0/24 (256 IPs)
        ├── EKS Worker Nodes
        └── SageMaker Instances
```

### Multi-AZ High Availability Benefits

- **🚀 High Availability**: EKS nodes distributed across 2 AZs
- **📈 Auto Scaling**: Load balanced traffic across AZs  
- **💾 Data Resilience**: Multi-AZ deployment for databases
- **🔧 Maintenance**: Rolling updates without downtime
- **🌐 Load Distribution**: Even traffic distribution

{{% notice success %}}
**🎯 Architecture Decision:** Multi-AZ design với separate NAT Gateways

**Production Benefits:**
- ✅ **99.99% availability** với multi-AZ design
- ✅ **Auto failover** giữa availability zones
- ✅ **Cost optimization** với right-sized subnets
- ✅ **Security layers** với public/private separation
{{% /notice %}}

## 1. Terraform Infrastructure Setup

### 1.1. Project Structure

```
retail-forecast/
├── aws/
│   ├── infra/
│   │   ├── main.tf              # Main infrastructure config
│   │   ├── variables.tf         # Input variables
│   │   ├── output.tf            # Output values
│   ├── k8s/                     # Kubernetes manifests
│   │   ├── namespace.yaml       # Kubernetes namespace
│   │   ├── service.yaml         # Service definition
│   │   └── hpa.yaml             # Horizontal Pod Autoscaler
│   ├── script/                  # Python automation scripts
│   │   ├── create_training_job.py    # SageMaker training job
│   │   ├── register_model.py         # Model registry script
│   │   ├── deploy_endpoint.py        # Model deployment
│   │   └── autoscaling_endpoint.py   # Auto-scaling setup
│   ├── Jenkinsfile              # Jenkins CI/CD pipeline
│   ├── .travis.yml              # Travis CI configuration
│   └── terraform.tfvars         # Environment-specific values
```

{{% notice info %}}
**📁 Project Structure Components:**
{{% /notice %}}

**Infrastructure (infra/):**
- ✅ **main.tf**: Core VPC infrastructure với subnets, NAT gateways, security groups
- ✅ **variables.tf**: Input variables cho environment configuration  
- ✅ **output.tf**: Export values cho other modules

**Kubernetes (k8s/):**
- ✅ **namespace.yaml**: Isolated namespace cho MLOps workloads
- ✅ **service.yaml**: Service exposure cho inference API
- ✅ **hpa.yaml**: Horizontal Pod Autoscaler cho dynamic scaling

**Automation (script/):**
- ✅ **create_training_job.py**: SageMaker training job automation
- ✅ **register_model.py**: Model registry và versioning
- ✅ **deploy_endpoint.py**: Model deployment automation
- ✅ **autoscaling_endpoint.py**: Endpoint auto-scaling configuration

**CI/CD:**
- ✅ **Jenkinsfile**: Jenkins pipeline cho automated deployment
- ✅ **.travis.yml**: Travis CI alternative configuration


### 1.2. Variables Configuration

**File: `aws/infra/variables.tf`**

```hcl
# Project configuration
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

# VPC configuration
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "azs" {
  description = "Availability zones"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}

variable "public_subnets" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Tagging
variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Project     = "MLOpsRetailForecast"
    Environment = "dev"
    ManagedBy   = "Terraform"
    Component   = "networking"
  }
}
```

**File: `aws/terraform.tfvars`**

```hcl
# Environment-specific configuration
project_name = "mlops-retail-forecast"
environment  = "dev"

# VPC networking
vpc_cidr        = "10.0.0.0/16"
azs            = ["ap-southeast-1a", "ap-southeast-1b"]
public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]

# Resource tagging
common_tags = {
  Project     = "MLOpsRetailForecast"
  Environment = "dev"
  ManagedBy   = "Terraform"
  Component   = "networking"
  CostCenter  = "ML-Platform"
}
```

### 1.3. Kubernetes Manifests Preview

**File: `aws/k8s/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mlops-retail-forecast
  labels:
    name: mlops-retail-forecast
    environment: dev
```

**File: `aws/k8s/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-api
  namespace: mlops-retail-forecast
spec:
  selector:
    app: retail-forecast
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

**File: `aws/k8s/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-forecast-hpa
  namespace: mlops-retail-forecast
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-forecast-api
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

{{% notice tip %}}
**🚀 Kubernetes Integration:**

VPC infrastructure được thiết kế để support Kubernetes deployment:
- ✅ **Private Subnets**: EKS worker nodes isolated từ Internet
- ✅ **Security Groups**: Proper network access control
- ✅ **Multi-AZ**: High availability cho Kubernetes pods
- ✅ **Load Balancer Tags**: EKS integration với AWS Load Balancer Controller
{{% /notice %}}

## 2. VPC Infrastructure Implementation

### 2.1. Main Infrastructure

**File: `aws/infra/main.tf`**

```hcl
# Configure AWS Provider
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
  
  default_tags {
    tags = var.common_tags
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-vpc"
    Type = "vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-igw"
    Type = "internet-gateway"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnets)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-${var.azs[count.index]}"
    Type = "public-subnet"
    "kubernetes.io/role/elb" = "1"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnets)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-${var.azs[count.index]}"
    Type = "private-subnet"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count = length(var.public_subnets)

  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
    Type = "elastic-ip"
  })
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = length(var.public_subnets)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-nat-${var.azs[count.index]}"
    Type = "nat-gateway"
  })

  depends_on = [aws_internet_gateway.main]
}

# Route Table for Public Subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-public-rt"
    Type = "route-table"
  })
}

# Route Table for Private Subnets
resource "aws_route_table" "private" {
  count = length(var.private_subnets)

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-private-rt-${count.index + 1}"
    Type = "route-table"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.public_subnets)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(var.private_subnets)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### 2.2. Security Groups

**Thêm vào `aws/infra/main.tf`:**

```hcl
# Security Group for EKS Control Plane
resource "aws_security_group" "eks_control_plane" {
  name_prefix = "${var.project_name}-${var.environment}-eks-control-plane"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-control-plane-sg"
    Type = "security-group"
  })
}

# Security Group for EKS Worker Nodes
resource "aws_security_group" "eks_nodes" {
  name_prefix = "${var.project_name}-${var.environment}-eks-nodes"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "All traffic from control plane"
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.eks_control_plane.id]
  }

  ingress {
    description = "All traffic from same security group"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true
  }

  egress {
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

# Security Group for Load Balancer
resource "aws_security_group" "alb" {
  name_prefix = "${var.project_name}-${var.environment}-alb"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-alb-sg"
    Type = "security-group"
  })
}

# Security Group for SageMaker
resource "aws_security_group" "sagemaker" {
  name_prefix = "${var.project_name}-${var.environment}-sagemaker"
  vpc_id      = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-sagemaker-sg"
    Type = "security-group"
  })
}
```

### 2.3. Outputs Configuration

**File: `aws/infra/outputs.tf`**

```hcl
# VPC Outputs
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

# Subnet Outputs
output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

# Gateway Outputs
output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  description = "IDs of the NAT Gateways"
  value       = aws_nat_gateway.main[*].id
}

# Security Group Outputs
output "eks_control_plane_security_group_id" {
  description = "Security group ID for EKS control plane"
  value       = aws_security_group.eks_control_plane.id
}

output "eks_nodes_security_group_id" {
  description = "Security group ID for EKS worker nodes"
  value       = aws_security_group.eks_nodes.id
}

output "alb_security_group_id" {
  description = "Security group ID for Application Load Balancer"
  value       = aws_security_group.alb.id
}

output "sagemaker_security_group_id" {
  description = "Security group ID for SageMaker"
  value       = aws_security_group.sagemaker.id
}

# Availability Zones
output "availability_zones" {
  description = "List of availability zones"
  value       = var.azs
}
```

## 3. Alternative: AWS Console Implementation

Ngoài Terraform, bạn cũng có thể tạo VPC infrastructure qua AWS Console để hiểu rõ hơn về từng component.

### 3.1. Tạo VPC qua Console

1. **Truy cập VPC Dashboard:**
   - Đăng nhập AWS Console
   - Navigate to VPC service
   - Chọn "Create VPC"

![Create VPC Console](../images/02-vpc-networking/01-create-vpc-console.png)

2. **VPC Configuration:**
   ```
   VPC Name: mlops-retail-forecast-dev-vpc
   IPv4 CIDR: 10.0.0.0/16
   IPv6 CIDR: No IPv6 CIDR block
   Tenancy: Default
   ```

![VPC Configuration](../images/02-vpc-networking/02-vpc-configuration.png)

### 3.2. Tạo Subnets

1. **Public Subnets:**
   - Navigate to "Subnets" → "Create subnet"
   - Chọn VPC vừa tạo

   **Subnet 1 (ap-southeast-1a):**
   ```
   Name: mlops-retail-forecast-dev-public-ap-southeast-1a
   Availability Zone: ap-southeast-1a
   IPv4 CIDR: 10.0.1.0/24
   ```

   **Subnet 2 (ap-southeast-1b):**
   ```
   Name: mlops-retail-forecast-dev-public-ap-southeast-1b
   Availability Zone: ap-southeast-1b
   IPv4 CIDR: 10.0.2.0/24
   ```

![Create Subnets 1](../images/02-vpc-networking/03.1-create-subnets.png)

![Create Subnets 2](../images/02-vpc-networking/03.2-create-subnets.png)


2. **Private Subnets:**
   
   **Subnet 3 (ap-southeast-1a):**
   ```
   Name: mlops-retail-forecast-dev-private-ap-southeast-1a
   Availability Zone: ap-southeast-1a
   IPv4 CIDR: 10.0.101.0/24
   ```


   **Subnet 4 (ap-southeast-1b):**
   ```
   Name: mlops-retail-forecast-dev-private-ap-southeast-1b
   Availability Zone: ap-southeast-1b
   IPv4 CIDR: 10.0.102.0/24
   ```

3. **Hoàn thành tạo subnet**

![Create Subnets 3](../images/02-vpc-networking/03.3-create-subnets.png)



### 3.3. Internet Gateway Setup

1. **Tạo Internet Gateway:**
   - Navigate to "Internet Gateways" → "Create internet gateway"
   ```
   Name: mlops-retail-forecast-dev-igw
   ```

![Internet Gateway 1](../images/02-vpc-networking/04.1-internet-gateway.png)

![Internet Gateway 2](../images/02-vpc-networking/04.2-internet-gateway.png)


2. **Attach to VPC:**
   - Select Internet Gateway → "Actions" → "Attach to VPC"
   - Chọn VPC đã tạo


![Internet Gateway 3](../images/02-vpc-networking/04.3-internet-gateway.png)

![Internet Gateway 4](../images/02-vpc-networking/04.4-internet-gateway.png)

![Internet Gateway 5](../images/02-vpc-networking/04.5-internet-gateway.png)




### 3.4. Route Tables Configuration

#### 3.4.1. Tạo Public Route Table

1. **Navigate to Route Tables:**
   - Trong VPC Dashboard → "Route Tables" → "Create route table"

![Public Route Table 1](../images/02-vpc-networking/07.1-public-route-table.png)

2. **Create Public Route Table:**
   ```
   Name: mlops-retail-forecast-dev-public-rt
   VPC: vpc-01f887abcfc9a090e (mlops-retail-forecast-dev-vpc)
   ```

![Public Route Table 2](../images/02-vpc-networking/07.2-public-route-table.png)


3. **Add Internet Gateway Route:**
   - Sau khi tạo route table → Tab "Routes" → "Edit routes"
   - Add route: `0.0.0.0/0` → Internet Gateway

![Public Route Table 3](../images/02-vpc-networking/07.3-public-route-table.png)



4. **Associate Public Subnets:**
   - Tab "Subnet associations" → "Edit subnet associations"
   - Chọn 2 public subnets (ap-southeast-1a và ap-southeast-1b)

![Public Route Table 4](../images/02-vpc-networking/07.4-public-route-table.png)



![Public Route Table 5](../images/02-vpc-networking/07.5-public-route-table.png)


#### 3.4.2. Tạo Private Route Table (VPC Endpoints Approach)

**Recommended:** Sử dụng **VPC Endpoints** thay vì NAT Gateway để tiết kiệm chi phí (~70% cost reduction).

**Bước 1: Tạo Private Route Table**
1. "Create route table" → Name: `mlops-retail-forecast-dev-private-rt`
2. Chọn VPC: `mlops-retail-forecast-dev-vpc`
3. "Create route table"

**Bước 2: Giữ Default Routes**
- **Không cần add thêm routes** - chỉ giữ local route (10.0.0.0/16 → local)
- VPC Endpoints sẽ tự động handle routing đến AWS services

**Bước 3: Associate Both Private Subnets**
1. Tab "Subnet associations" → "Edit subnet associations"
2. Chọn **cả 2 private subnets** (ap-southeast-1a và ap-southeast-1b)
3. "Save associations"

**Bước 4: Tạo VPC Endpoints (quan trọng)**
- S3 Gateway Endpoint (FREE)
- ECR API Interface Endpoint
- ECR DKR Interface Endpoint  
- CloudWatch Logs Interface Endpoint
- (Xem chi tiết ở section 3.7.1)

**Tại sao approach này tốt hơn?**
- **VPC Endpoints** handle AWS services (S3, ECR, CloudWatch) directly
- **No Internet access needed** cho private subnets
- **Cost savings**: $21.6/month vs $71/month với NAT Gateway
- **Better security**: Traffic không đi qua Internet

![Private Route Tables](../images/02-vpc-networking/08-private-route-tables.png)

#### 3.4.3. Verification Route Tables (VPC Endpoints Approach)

Sau khi hoàn thành, bạn sẽ có:
```
✅ 1 Public Route Table:
   - mlops-retail-forecast-dev-public-rt
   - Routes: 0.0.0.0/0 → Internet Gateway
   - Associated: 2 public subnets

✅ 1 Private Route Table:
   - mlops-retail-forecast-dev-private-rt
   - Routes: 10.0.0.0/16 → local (default)
   - Associated: 2 private subnets
   - AWS services access via VPC Endpoints

✅ VPC Endpoints (thay thế NAT Gateway):
   - S3 Gateway Endpoint (FREE)
   - ECR API/DKR Interface Endpoints
   - CloudWatch Logs Interface Endpoint
   - Total cost: ~$21.6/month vs $71/month NAT Gateway
```

**Lưu ý quan trọng:**
- Private subnets **không có Internet access** trực tiếp
- Tất cả AWS services access qua VPC Endpoints
- Nếu cần external Internet access → dùng NAT Instance cho dev ($10/month)

### 3.5. Security Groups Setup

Security Groups hoạt động như virtual firewall để kiểm soát inbound và outbound traffic cho AWS resources.

#### 3.5.1. Tạo EKS Control Plane Security Group

1. **Navigate to Security Groups:**
   - Trong VPC Dashboard → "Security Groups" → "Create security group"

![Create Security Group](../images/02-vpc-networking/09.1-create-security-group.png)

2. **Basic Details:**
   - **Security group name**: `mlops-retail-forecast-dev-eks-control-plane-sg`
   - **Description**: `Security group for EKS control plane`
   - **VPC**: Chọn `vpc-01f887abcfc9a090e (mlops-retail-forecast-dev-vpc)`

![EKS Control Plane Basic](../images/02-vpc-networking/09.2-eks-control-plane-basic.png)

3. **Inbound Rules:**
   - Click "Add rule"
   - **Type**: HTTPS
   - **Port range**: 443 (auto-filled)
   - **Source**: 0.0.0.0/0 (Anywhere-IPv4)
   - **Description**: `HTTPS access for EKS API server`

![EKS Control Plane Inbound](../images/02-vpc-networking/09.3-eks-control-plane-inbound.png)

4. **Outbound Rules:**
   - Giữ default rule: All traffic → 0.0.0.0/0
   - **Description**: `All outbound traffic allowed`

5. **Tags (Optional):**
   - **Key**: `Name`
   - **Value**: `mlops-retail-forecast-dev-eks-control-plane-sg`

6. **Create Security Group**

![EKS Control Plane Complete](../images/02-vpc-networking/09.4-eks-control-plane-complete.png)

#### 3.5.2. Tạo EKS Worker Nodes Security Group

1. **Basic Details:**
   - **Security group name**: `mlops-retail-forecast-dev-eks-nodes-sg`
   - **Description**: `Security group for EKS worker nodes`
   - **VPC**: Chọn `vpc-01f887abcfc9a090e (mlops-retail-forecast-dev-vpc)`

![EKS Nodes Basic](../images/02-vpc-networking/09.5-eks-nodes-basic.png)

2. **Inbound Rules:**
   
   **Rule 1 - Traffic từ EKS Control Plane:**
   - Click "Add rule"
   - **Type**: All Traffic
   - **Protocol**: All (auto-filled)
   - **Port range**: All (auto-filled)
   - **Source**: Custom → Chọn Security Group → `mlops-retail-forecast-dev-eks-control-plane-sg`
   - **Description**: `Traffic from EKS control plane`
   
   **Rule 2 - Inter-node Communication:**
   - Click "Add rule"
   - **Type**: All Traffic
   - **Protocol**: All (auto-filled)
   - **Port range**: All (auto-filled)
   - **Source**: Custom → Chọn Security Group → `mlops-retail-forecast-dev-eks-nodes-sg` (self-reference)
   - **Description**: `Inter-node communication`

![EKS Nodes Inbound](../images/02-vpc-networking/09.6-eks-nodes-inbound.png)

3. **Outbound Rules:**
   - Giữ default rule: All traffic → 0.0.0.0/0
   - **Description**: `All outbound traffic for package downloads and AWS API calls`

4. **Tags:**
   - **Key**: `Name`
   - **Value**: `mlops-retail-forecast-dev-eks-nodes-sg`

![EKS Nodes Complete](../images/02-vpc-networking/09.7-eks-nodes-complete.png)

#### 3.5.3. Tạo Application Load Balancer Security Group

1. **Basic Details:**
   - **Security group name**: `mlops-retail-forecast-dev-alb-sg`
   - **Description**: `Security group for Application Load Balancer`
   - **VPC**: Chọn `vpc-01f887abcfc9a090e (mlops-retail-forecast-dev-vpc)`

![ALB Basic](../images/02-vpc-networking/09.8-alb-basic.png)

2. **Inbound Rules:**
   
   **Rule 1 - HTTP Traffic:**
   - Click "Add rule"
   - **Type**: HTTP
   - **Port range**: 80 (auto-filled)
   - **Source**: 0.0.0.0/0 (Anywhere-IPv4)
   - **Description**: `HTTP access from Internet`
   
   **Rule 2 - HTTPS Traffic:**
   - Click "Add rule"
   - **Type**: HTTPS
   - **Port range**: 443 (auto-filled)
   - **Source**: 0.0.0.0/0 (Anywhere-IPv4)
   - **Description**: `HTTPS access from Internet`

![ALB Inbound](../images/02-vpc-networking/09.9-alb-inbound.png)

3. **Outbound Rules:**
   - Giữ default rule: All traffic → 0.0.0.0/0
   - **Description**: `Outbound traffic to EKS nodes`

4. **Tags:**
   - **Key**: `Name`
   - **Value**: `mlops-retail-forecast-dev-alb-sg`

![ALB Complete](../images/02-vpc-networking/09.10-alb-complete.png)

#### 3.5.4. Tạo VPC Endpoints Security Group

1. **Basic Details:**
   - **Security group name**: `mlops-retail-forecast-dev-vpc-endpoints-sg`
   - **Description**: `Security group for VPC endpoints`
   - **VPC**: Chọn `vpc-01f887abcfc9a090e (mlops-retail-forecast-dev-vpc)`

![VPC Endpoints Basic](../images/02-vpc-networking/09.11-vpc-endpoints-basic.png)

2. **Inbound Rules:**
   
   **Rule 1 - HTTPS từ VPC:**
   - Click "Add rule"
   - **Type**: HTTPS
   - **Port range**: 443 (auto-filled)
   - **Source**: Custom → `10.0.0.0/16` (VPC CIDR)
   - **Description**: `HTTPS access from VPC for AWS services`

![VPC Endpoints Inbound](../images/02-vpc-networking/09.12-vpc-endpoints-inbound.png)

3. **Outbound Rules:**
   - Giữ default rule: All traffic → 0.0.0.0/0
   - **Description**: `Outbound traffic to AWS services`

4. **Tags:**
   - **Key**: `Name`
   - **Value**: `mlops-retail-forecast-dev-vpc-endpoints-sg`

![VPC Endpoints Complete](../images/02-vpc-networking/09.13-vpc-endpoints-complete.png)

#### 3.5.5. Security Groups Summary

Sau khi tạo xong, bạn sẽ có 4 Security Groups:

![Security Groups Overview](../images/02-vpc-networking/10-security-groups-overview.png)

### 3.6. Console Verification

1. **VPC Resource Map:**
   - Navigate to VPC Dashboard
   - Chọn VPC đã tạo
   - Xem Resource Map để verify architecture

![VPC Resource Map](../images/02-vpc-networking/11-vpc-resource-map.png)

2. **Network Topology:**
   ```
   ✅ VPC: 10.0.0.0/16 (4 subnets across 2 AZs)
   ✅ Internet Gateway: Attached
   ✅ NAT Gateways: 2 (high availability)
   ✅ Route Tables: 3 (1 public, 2 private)
   ✅ Security Groups: 4 (EKS, ALB, SageMaker)
   ```

{{% notice success %}}
**🎯 Console Implementation Complete!**

Bạn đã tạo thành công VPC infrastructure qua AWS Console. Architecture này tương đương với Terraform implementation và ready cho EKS deployment trong Task 4.
{{% /notice %}}

{{% notice info %}}
**💡 Console vs Terraform:**
{{% /notice %}}

**Console Advantages:**
- ✅ Visual interface dễ hiểu
- ✅ Real-time validation
- ✅ Immediate feedback

**Terraform Advantages:**
- ✅ Infrastructure as Code
- ✅ Version control
- ✅ Reproducible deployments
- ✅ Automation-ready

Khuyến nghị: Học Console để hiểu concepts, dùng Terraform cho production.

### 3.7. NAT Gateway là gì và tại sao tốn kém?

**NAT (Network Address Translation) Gateway** là managed service của AWS cho phép resources trong private subnet truy cập Internet mà không expose public IP. Tuy nhiên, NAT Gateway rất tốn chi phí vì:

1. **Hourly Charges**: $0.045/hour (~$32/month) per NAT Gateway
2. **Data Processing**: $0.045/GB cho mọi data đi qua NAT Gateway
3. **Multi-AZ Requirement**: Cần 1 NAT Gateway/AZ cho high availability
4. **Always Running**: Không thể tắt khi không dùng như EC2

**Tại sao cần NAT Gateway?**
- EKS worker nodes cần pull Docker images từ ECR
- Download packages và updates từ Internet
- Access AWS services (S3, CloudWatch, Parameter Store)
- Outbound HTTPS calls từ applications

**Current Cost Impact (ap-southeast-1):**
- **NAT Gateways**: 2 × $32 = $64/month (hourly charges)
- **Elastic IPs**: 2 × $3.6 = $7.2/month  
- **Data Transfer**: $0.045/GB processed (ECR pulls, updates, API calls)
- **Total Baseline**: ~$71/month + traffic costs

**Typical Data Transfer:**
- ECR image pulls: 5-10GB/month
- Package updates: 2-5GB/month  
- API calls: 1-2GB/month
- **Additional cost**: ~$0.36-0.77/month

#### 3.7.1. VPC Endpoints - Giải pháp tối ưu chi phí

**VPC Endpoints** cho phép private subnets truy cập AWS services mà không cần NAT Gateway, giảm đáng kể chi phí và tăng bảo mật.

##### Tạo VPC Endpoints qua AWS Console

**Bước 1: Navigate to VPC Endpoints**
- Trong VPC Dashboard → "Endpoints" → "Create endpoint"

![Create VPC Endpoint](../images/02-vpc-networking/10.1-create-vpc-endpoint.png)

**Bước 2: Endpoint Settings**

1. **Name tag (optional)**: `mlops-s3-endpoint`
2. **Type**: Chọn **AWS services** (đã được chọn mặc định)

![Endpoint Settings](../images/02-vpc-networking/10.2-endpoint-settings.png)

**Bước 3: Tạo S3 Gateway Endpoint (FREE)**

1. **Services**: Tìm và chọn `com.amazonaws.ap-southeast-1.s3` (Type: Gateway)
2. **VPC**: Chọn VPC đã tạo từ dropdown
3. **Route Tables**: Chọn private route table
4. **Policy**: Full Access (default)

![S3 Gateway Endpoint](../images/02-vpc-networking/10.3-s3-gateway-endpoint.png)

**Bước 4: Tạo ECR API Interface Endpoint**

1. **Name tag**: `mlops-ecr-api-endpoint`
2. **Services**: Tìm và chọn `com.amazonaws.ap-southeast-1.ecr.api` (Type: Interface)
3. **VPC**: Chọn VPC đã tạo từ dropdown "Select a VPC"
4. **Subnets**: Chọn cả 2 private subnets trong Network settings
5. **Security Groups**: Chọn `mlops-retail-forecast-dev-vpc-endpoints-sg`
6. **Policy**: Full Access (default)
7. **Private DNS names enabled**: ✅ Checked

![ECR API Endpoint 1](../images/02-vpc-networking/10.4.1-ecr-api-endpoint.png)
![ECR API Endpoint 2](../images/02-vpc-networking/10.4.2-ecr-api-endpoint.png)
![ECR API Endpoint 3](../images/02-vpc-networking/10.4.3-ecr-api-endpoint.png)



**Bước 5: Tạo ECR DKR Interface Endpoint**

1. **Name tag**: `mlops-ecr-dkr-endpoint`
2. **Services**: Tìm và chọn `com.amazonaws.ap-southeast-1.ecr.dkr` (Type: Interface)
3. **VPC**: Chọn VPC đã tạo từ dropdown "Select a VPC"
4. **Network settings**: 
   - **Subnets**: Chọn cả 2 private subnets
   - **Security Groups**: Chọn VPC endpoints security group
5. **Private DNS names enabled**: ✅ Checked

![ECR DKR Endpoint 1](../images/02-vpc-networking/10.5.1-ecr-dkr-endpoint.png)

![ECR DKR Endpoint 2](../images/02-vpc-networking/10.5.2-ecr-dkr-endpoint.png)

![ECR DKR Endpoint 3](../images/02-vpc-networking/10.5.3-ecr-dkr-endpoint.png)


**Bước 6: Tạo CloudWatch Logs Interface Endpoint**

1. **Name tag**: `mlops-cloudwatch-logs-endpoint`
2. **Services**: Tìm và chọn `com.amazonaws.ap-southeast-1.logs` (Type: Interface)
3. **VPC**: Chọn VPC đã tạo từ dropdown "Select a VPC"
4. **Network settings**:
   - **Subnets**: Chọn cả 2 private subnets
   - **Security Groups**: Chọn VPC endpoints security group
5. **Private DNS names enabled**: ✅ Checked

![CloudWatch Logs Endpoint 1](../images/02-vpc-networking/10.6.1-cloudwatch-logs-endpoint.png)

![CloudWatch Logs Endpoint 2](../images/02-vpc-networking/10.6.2-cloudwatch-logs-endpoint.png)

![CloudWatch Logs Endpoint 3](../images/02-vpc-networking/10.6.3-cloudwatch-logs-endpoint.png)


**Bước 7: Verification**

Sau khi tạo xong, kiểm tra trong VPC Endpoints dashboard:

![VPC Endpoints Overview](../images/02-vpc-networking/10.7-vpc-endpoints-overview.png)

**Lưu ý quan trọng khi tạo VPC Endpoints:**

1. **Gateway vs Interface Endpoints:**
   - **Gateway Endpoint** (S3): FREE, route qua route table
   - **Interface Endpoint** (ECR, CloudWatch): $7.2/month + data transfer

2. **Private DNS Names:**
   - ✅ **Bật** cho Interface Endpoints để applications có thể dùng standard AWS service URLs
   - Ví dụ: `ecr.ap-southeast-1.amazonaws.com` sẽ resolve đến VPC endpoint

3. **Security Groups:**
   - Interface Endpoints cần Security Group cho phép HTTPS (443) từ VPC CIDR
   - Gateway Endpoints không cần Security Group

4. **Subnet Selection:**
   - Interface Endpoints: Chọn private subnets trong cả 2 AZ cho high availability
   - Gateway Endpoints: Chỉ cần associate với route tables

##### Terraform Implementation (Alternative):
```hcl
# S3 Gateway Endpoint (FREE - no hourly charges)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
  
  # Associate với route tables
  route_table_ids = aws_route_table.private[*].id
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-s3-endpoint"
    Type = "gateway-endpoint"
  })
}

# ECR API Endpoint - cho Docker registry API calls
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  # Enable DNS resolution
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-api-endpoint"
    Type = "interface-endpoint"
  })
}

# ECR DKR Endpoint - cho Docker image pulls
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-dkr-endpoint"
    Type = "interface-endpoint"
  })
}

# CloudWatch Logs Endpoint - cho logging
resource "aws_vpc_endpoint" "logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-logs-endpoint"
    Type = "interface-endpoint"
  })
}

# Security Group for VPC Endpoints
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${var.project_name}-${var.environment}-vpc-endpoints"
  vpc_id      = aws_vpc.main.id
  description = "Security group for VPC endpoints"

  # Allow HTTPS from VPC
  ingress {
    description = "HTTPS from VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  # Allow all outbound (required for endpoints)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-vpc-endpoints-sg"
    Type = "security-group"
  })
}
```

**VPC Endpoints Cost Breakdown:**
- **S3 Gateway Endpoint**: FREE (no charges)
- **ECR API Interface Endpoint**: $7.2/month + $0.01/GB
- **ECR DKR Interface Endpoint**: $7.2/month + $0.01/GB  
- **CloudWatch Logs Endpoint**: $7.2/month + $0.01/GB
- **Total**: ~$21.6/month + minimal data transfer costs

**Benefits:**
- ✅ **70% cost reduction** so với NAT Gateway ($21.6 vs $71/month)
- ✅ **Better security**: Traffic không đi qua Internet
- ✅ **Lower latency**: Direct connection đến AWS services
- ✅ **No bandwidth limits**: Không bị giới hạn NAT Gateway bandwidth

## 4. Terraform Deployment

### 4.1. Initialize và Plan

```bash
# Navigate to infrastructure directory
cd aws/infra

# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Plan deployment
terraform plan -var-file="../terraform.tfvars"
```

**Expected Plan Output:**
```
Plan: 23 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + vpc_id = (known after apply)
  + public_subnet_ids = [
      + (known after apply),
      + (known after apply),
    ]
  + private_subnet_ids = [
      + (known after apply),
      + (known after apply),
    ]
```

### 4.2. Apply Infrastructure

```bash
# Apply infrastructure
terraform apply -var-file="../terraform.tfvars"
```

**Apply Output:**
```
Apply complete! Resources: 23 added, 0 changed, 0 destroyed.

Outputs:
vpc_id = "vpc-0123456789abcdef0"
public_subnet_ids = [
  "subnet-0123456789abcdef0",
  "subnet-0123456789abcdef1",
]
private_subnet_ids = [
  "subnet-0fedcba9876543210",
  "subnet-0fedcba9876543211",
]
```

## 5. Verification và Testing

### 5.1. AWS Console Verification

1. **VPC Dashboard:**
   - Navigate to VPC console
   - Verify VPC created: `mlops-retail-forecast-dev-vpc`
   - Check CIDR: `10.0.0.0/16`

2. **Subnets Verification:**
   ```
   ✅ mlops-retail-forecast-dev-public-ap-southeast-1a (10.0.1.0/24)
   ✅ mlops-retail-forecast-dev-public-ap-southeast-1b (10.0.2.0/24)
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1a (10.0.101.0/24)
   ✅ mlops-retail-forecast-dev-private-ap-southeast-1b (10.0.102.0/24)
   ```

3. **NAT Gateways:**
   ```
   ✅ mlops-retail-forecast-dev-nat-ap-southeast-1a (public subnet)
   ✅ mlops-retail-forecast-dev-nat-ap-southeast-1b (public subnet)
   ```

4. **Security Groups:**
   ```
   ✅ mlops-retail-forecast-dev-eks-control-plane-sg
   ✅ mlops-retail-forecast-dev-eks-nodes-sg
   ✅ mlops-retail-forecast-dev-alb-sg
   ✅ mlops-retail-forecast-dev-sagemaker-sg
   ```

### 5.2. CLI Verification

```bash
# Get VPC information
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=mlops-retail-forecast-dev-vpc" \
  --query 'Vpcs[0].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}'

# List subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'Subnets[*].{SubnetId:SubnetId,CidrBlock:CidrBlock,AZ:AvailabilityZone,Type:Tags[?Key==`Type`].Value|[0]}'

# Check NAT Gateways
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'NatGateways[*].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId}'
```

### 5.3. Connectivity Testing

```bash
# Test Internet connectivity from private subnet (will be used in later tasks)
# This command will be used when EKS nodes are deployed

# For now, verify route tables are properly configured
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'RouteTables[*].{RouteTableId:RouteTableId,Routes:Routes[*].{Destination:DestinationCidrBlock,Target:GatewayId//NatGatewayId}}'
```

## 👉 Kết quả Task 2

Sau Task 2, ta có một hạ tầng mạng chuẩn AWS, phân tách private/public rõ ràng, đủ bảo mật, và có chiến lược giảm chi phí NAT phù hợp với từng môi trường (dev/staging vs prod).

### ✅ Deliverables Completed

- **VPC Infrastructure**: Multi-AZ VPC với proper CIDR design (10.0.0.0/16)
- **Subnets**: Public/Private subnets phân tách rõ ràng trong 2 availability zones
- **Internet Access**: NAT Gateway/Instance cho private subnets truy cập Internet
- **Security Groups**: Least privilege access rules cho EKS, ALB, SageMaker
- **Cost Optimization**: Chiến lược giảm chi phí cho từng environment
- **Terraform Outputs**: Infrastructure as Code với proper outputs cho integration

### Architecture Achieved

```
✅ VPC: 10.0.0.0/16 (4 subnets across 2 AZs)
✅ Internet Gateway: Attached và configured
✅ NAT Solutions: Gateway (prod) / Instance (dev) / VPC Endpoints
✅ Route Tables: 3 (1 public, 2 private) với proper routing
✅ Security Groups: 4 (EKS Control Plane, Nodes, ALB, SageMaker)
✅ Cost Options: $10-71/month tùy environment và requirements
```

### Cost Summary

| Environment | Solution | Monthly Cost | Availability |
|-------------|----------|--------------|--------------|
| **Development** | NAT Instance + VPC Endpoints | ~$31.6 | 99.5% |
| **Staging** | VPC Endpoints + Single NAT Gateway | ~$53.6 | 99.95% |
| **Production** | VPC Endpoints + Single NAT Gateway | ~$53.6 | 99.95% |
| **VPC Endpoints Only** | S3/ECR/CloudWatch Endpoints | ~$21.6 | 99.99% |

**Cost Savings vs Traditional Multi-AZ NAT Gateway:**
- Development: 55% savings ($31.6 vs $71)
- Staging/Production: 25% savings ($53.6 vs $71)

{{% notice success %}}
**🎯 Ready for Next Tasks:**

Network foundation đã sẵn sàng để deploy:
- ✅ **Task 3**: IAM Roles & IRSA configuration
- ✅ **Task 4**: EKS cluster deployment trong VPC infrastructure
- ✅ **Task 5**: EKS managed node groups trong private subnets
- ✅ **Task 6**: Application Load Balancer integration
{{% /notice %}}

{{% notice info %}}
**🔧 Integration Points:**

Terraform outputs từ Task 2 sẽ được sử dụng bởi:
- **EKS Module**: `vpc_id`, `private_subnet_ids`, `eks_*_security_group_id`
- **ALB Module**: `public_subnet_ids`, `alb_security_group_id`
- **SageMaker Module**: `private_subnet_ids`, `sagemaker_security_group_id`
{{% /notice %}}