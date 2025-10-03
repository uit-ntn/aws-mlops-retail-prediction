---
title: "VPC / Networking"
date: 2025-08-30T11:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## 🎯 Mục tiêu Task 2

Thiết lập **cost-optimized VPC foundation** cho MLOps platform:

1. **Basic VPC Setup** (Console) - VPC, subnets, Internet Gateway, Security Groups
2. **VPC Endpoints** (Terraform) - Thay thế NAT Gateway để tiết kiệm 70% chi phí
3. **Advanced Integration** (Terraform) - Outputs cho EKS, ALB, SageMaker modules

{{% notice info %}}
**💡 Khi nào cần Terraform cho VPC:**
- ✅ **VPC Endpoints** - Cost optimization, không thể setup dễ dàng qua Console
- ✅ **Integration outputs** - Structured data cho other Terraform modules
- ✅ **Environment automation** - Consistent deployment across dev/staging/prod

**Console đủ cho:** VPC, subnets, Internet Gateway, Security Groups, route tables
{{% /notice %}}

📥 **Input**
- AWS Account với VPC permissions
- CIDR planning: `10.0.0.0/16`
- Cost optimization target: 70% savings vs NAT Gateway

📌 **Các bước chính**
1. **Console Setup** - Basic VPC infrastructure (15 phút)
2. **Terraform VPC Endpoints** - Cost optimization (5 phút)
3. **Verification** - Test connectivity và cost savings

✅ **Deliverables**
- Multi-AZ VPC với public/private subnet separation
- VPC Endpoints: S3 (FREE), ECR, CloudWatch ($21.6/month vs $71/month NAT Gateway)
- Terraform outputs cho integration với EKS, ALB, SageMaker

📊 **Acceptance Criteria**
- VPC spans 2 AZ với proper subnet design
- Private subnets access AWS services qua VPC Endpoints
- Cost optimized: $21.6/month vs $71/month traditional approach

⚠️ **Gotchas**
- VPC Endpoints cần specific Security Group rules (HTTPS 443 from VPC CIDR)
- Interface Endpoints cost $7.2/month each, Gateway Endpoints (S3) FREE
- Private DNS must be enabled cho Interface Endpoints
- No external Internet access without NAT Gateway (GitHub, PyPI)

## Kiến trúc VPC Cost-Optimized

### Network Design Overview

```
VPC: 10.0.0.0/16 (Cost-Optimized với VPC Endpoints)
├── ap-southeast-1a (AZ-1)
│   ├── Public Subnet: 10.0.1.0/24
│   │   └── Internet Gateway access
│   └── Private Subnet: 10.0.101.0/24
│       ├── EKS Worker Nodes
│       └── VPC Endpoints access
└── ap-southeast-1b (AZ-2)
    ├── Public Subnet: 10.0.2.0/24
    │   └── Internet Gateway access
    └── Private Subnet: 10.0.102.0/24
        ├── EKS Worker Nodes
        └── VPC Endpoints access

VPC Endpoints (thay thế NAT Gateway):
├── S3 Gateway Endpoint (FREE)
├── ECR API Interface Endpoint ($7.2/month)
├── ECR DKR Interface Endpoint ($7.2/month)
└── CloudWatch Logs Interface Endpoint ($7.2/month)

Total: $21.6/month vs $71/month NAT Gateway (70% savings)
```

{{% notice success %}}
**🎯 Cost Optimization Strategy**

**Traditional Approach:** Multi-AZ NAT Gateway = $71/month
- 2x NAT Gateway: $64/month
- 2x Elastic IP: $7.2/month

**Optimized Approach:** VPC Endpoints = $21.6/month
- S3 Gateway Endpoint: FREE
- 3x Interface Endpoints: $21.6/month
- **Savings: 70% reduction**
{{% /notice %}}

## 1. Console Setup - Basic VPC Infrastructure

{{% notice success %}}
**🎯 Console Approach:** Quick setup cho basic VPC components

**Scope:** Tạo foundation infrastructure qua Console
- VPC với CIDR `10.0.0.0/16`
- Public/Private subnets across 2 AZ
- Internet Gateway và route tables
- Basic Security Groups

**Time required:** ~15 phút
{{% /notice %}}

### 1.1. Tạo VPC qua Console

1. **Truy cập VPC Dashboard:**
   - AWS Console → VPC service → "Create VPC"

![Create VPC Console](../images/02-vpc-networking/01-create-vpc-console.png)

2. **VPC Configuration:**
   ```
   VPC Name: mlops-retail-forecast-dev-vpc
   IPv4 CIDR: 10.0.0.0/16
   IPv6 CIDR: No IPv6 CIDR block
   Tenancy: Default
   ```

![VPC Configuration](../images/02-vpc-networking/02-vpc-configuration.png)

### 1.2. Tạo Subnets

1. **Public Subnets:**
   - Navigate to "Subnets" → "Create subnet"

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

![Create Subnets 3](../images/02-vpc-networking/03.3-create-subnets.png)

### 1.3. Internet Gateway Setup

1. **Tạo Internet Gateway:**
   - "Internet Gateways" → "Create internet gateway"
   ```
   Name: mlops-retail-forecast-dev-igw
   ```

![Internet Gateway 1](../images/02-vpc-networking/04.1-internet-gateway.png)
![Internet Gateway 2](../images/02-vpc-networking/04.2-internet-gateway.png)

2. **Attach to VPC:**
   - Select Internet Gateway → "Actions" → "Attach to VPC"

![Internet Gateway 3](../images/02-vpc-networking/04.3-internet-gateway.png)
![Internet Gateway 4](../images/02-vpc-networking/04.4-internet-gateway.png)
![Internet Gateway 5](../images/02-vpc-networking/04.5-internet-gateway.png)

### 1.4. Route Tables Configuration

#### 1.4.1. Public Route Table

1. **Create Public Route Table:**
   ```
   Name: mlops-retail-forecast-dev-public-rt
   VPC: mlops-retail-forecast-dev-vpc
   ```

![Public Route Table 1](../images/02-vpc-networking/07.1-public-route-table.png)
![Public Route Table 2](../images/02-vpc-networking/07.2-public-route-table.png)

2. **Add Internet Gateway Route:**
   - Add route: `0.0.0.0/0` → Internet Gateway

![Public Route Table 3](../images/02-vpc-networking/07.3-public-route-table.png)

3. **Associate Public Subnets:**
   - Associate 2 public subnets

![Public Route Table 4](../images/02-vpc-networking/07.4-public-route-table.png)
![Public Route Table 5](../images/02-vpc-networking/07.5-public-route-table.png)

#### 1.4.2. Private Route Table (VPC Endpoints Strategy)

**Cost-Optimized Approach:** Sử dụng VPC Endpoints thay vì NAT Gateway

1. **Create Private Route Table:**
   ```
   Name: mlops-retail-forecast-dev-private-rt
   VPC: mlops-retail-forecast-dev-vpc
   ```

2. **Keep Default Routes Only:**
   - Chỉ giữ local route (10.0.0.0/16 → local)
   - VPC Endpoints sẽ handle AWS services access

3. **Associate Both Private Subnets:**
   - Associate cả 2 private subnets

![Private Route Tables](../images/02-vpc-networking/08-private-route-tables.png)

### 1.5. Security Groups Setup

#### 1.5.1. EKS Control Plane Security Group

1. **Basic Details:**
   ```
   Security group name: mlops-retail-forecast-dev-eks-control-plane-sg
   Description: Security group for EKS control plane
   VPC: mlops-retail-forecast-dev-vpc
   ```

![Create Security Group](../images/02-vpc-networking/09.1-create-security-group.png)
![EKS Control Plane Basic](../images/02-vpc-networking/09.2-eks-control-plane-basic.png)

2. **Inbound Rules:**
   - **Type**: HTTPS, **Port**: 443, **Source**: 0.0.0.0/0

![EKS Control Plane Inbound](../images/02-vpc-networking/09.3-eks-control-plane-inbound.png)
![EKS Control Plane Complete](../images/02-vpc-networking/09.4-eks-control-plane-complete.png)

#### 1.5.2. EKS Worker Nodes Security Group

1. **Basic Details:**
   ```
   Security group name: mlops-retail-forecast-dev-eks-nodes-sg
   Description: Security group for EKS worker nodes
   ```

![EKS Nodes Basic](../images/02-vpc-networking/09.5-eks-nodes-basic.png)

2. **Inbound Rules:**
   - **Rule 1**: All Traffic from EKS Control Plane SG
   - **Rule 2**: All Traffic from self (inter-node communication)

![EKS Nodes Inbound](../images/02-vpc-networking/09.6-eks-nodes-inbound.png)
![EKS Nodes Complete](../images/02-vpc-networking/09.7-eks-nodes-complete.png)

#### 1.5.3. Application Load Balancer Security Group

1. **Basic Details:**
   ```
   Security group name: mlops-retail-forecast-dev-alb-sg
   Description: Security group for Application Load Balancer
   ```

![ALB Basic](../images/02-vpc-networking/09.8-alb-basic.png)

2. **Inbound Rules:**
   - **Rule 1**: HTTP (80) from 0.0.0.0/0
   - **Rule 2**: HTTPS (443) from 0.0.0.0/0

![ALB Inbound](../images/02-vpc-networking/09.9-alb-inbound.png)
![ALB Complete](../images/02-vpc-networking/09.10-alb-complete.png)

#### 1.5.4. VPC Endpoints Security Group

1. **Basic Details:**
   ```
   Security group name: mlops-retail-forecast-dev-vpc-endpoints-sg
   Description: Security group for VPC endpoints
   ```

![VPC Endpoints Basic](../images/02-vpc-networking/09.11-vpc-endpoints-basic.png)

2. **Inbound Rules:**
   - **Rule**: HTTPS (443) from VPC CIDR (10.0.0.0/16)

![VPC Endpoints Inbound](../images/02-vpc-networking/09.12-vpc-endpoints-inbound.png)
![VPC Endpoints Complete](../images/02-vpc-networking/09.13-vpc-endpoints-complete.png)

### 1.6. Console Setup Complete

![Security Groups Overview](../images/02-vpc-networking/10-security-groups-overview.png)
![VPC Resource Map](../images/02-vpc-networking/11-vpc-resource-map.png)

{{% notice success %}}
**🎯 Console Setup Complete!**

Basic VPC infrastructure đã sẵn sàng. Tiếp theo sẽ setup VPC Endpoints với Terraform cho cost optimization.
{{% /notice %}}

## 2. Terraform cho Integration Outputs (Optional)

{{% notice info %}}
**💡 Khi nào cần Terraform cho Task 2:**
- ✅ **Integration outputs** - Structured data cho EKS, ALB modules trong subsequent tasks
- ✅ **Environment automation** - Reproducible deployment across dev/staging/prod
- ✅ **Infrastructure as Code** - Version control cho VPC configuration

**Console đủ cho:** VPC, subnets, Security Groups, VPC Endpoints (đã có hướng dẫn chi tiết ở Section 1)
{{% /notice %}}

### 2.0. Terraform Purpose & Expected Results

{{% notice success %}}
**🎯 Mục đích của Terraform code trong Task 2 (Optional):**

**Input:** 
- VPC infrastructure đã tạo qua Console (VPC, subnets, Security Groups, VPC Endpoints)
- Cần integration outputs cho Task 4-5 (EKS cluster, ALB)

**Terraform sẽ làm gì:**
1. **Reference existing infrastructure** từ Console (không tạo mới gì cả)
2. **Create data sources** để query VPC, subnets, Security Groups
3. **Generate outputs** cho subsequent Terraform modules
4. **Enable integration** với EKS, ALB modules

**Kết quả sau khi chạy:**
- ✅ Terraform outputs available: VPC ID, subnet IDs, Security Group IDs
- ✅ Integration ready: Data sources cho Task 4-5 Terraform modules
- ✅ Infrastructure as Code: VPC configuration được track trong Terraform state
- ✅ No new resources created: Chỉ reference existing infrastructure

**Lưu ý:** Nếu bạn chỉ dùng Console cho tất cả tasks, không cần phần Terraform này!
{{% /notice %}}

{{% notice warning %}}
**📝 Important Note:**

VPC Endpoints đã được tạo qua Console ở Section 1. Terraform code dưới đây CHỈ để:
- **Reference existing resources** cho integration
- **Generate outputs** cho subsequent tasks

**KHÔNG tạo VPC Endpoints mới!** Nếu bạn đã tạo qua Console rồi thì skip phần Terraform này.
{{% /notice %}}

### 2.1. Data Sources cho Integration (Reference Only)

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Reference existing VPC infrastructure** đã tạo qua Console
2. **Query VPC Endpoints** đã tạo qua Console (không tạo mới)
3. **Generate outputs** cho Task 4-5 integration
4. **Enable Terraform state tracking** cho existing resources

**Kết quả:** Terraform có thể reference Console-created resources cho subsequent tasks
{{% /notice %}}

**File: `aws/infra/vpc-data-sources.tf`**

```hcl
# BƯỚC 1: Reference VPC infrastructure từ Console (không tạo mới)
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-vpc"]  # VPC từ Console
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Name"
    values = ["*private*"]  # Private subnets từ Console
  }
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Name"
    values = ["*public*"]  # Public subnets từ Console
  }
}

# BƯỚC 2: Reference Security Groups từ Console
data "aws_security_group" "eks_control_plane" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-eks-control-plane-sg"]  # SG từ Console
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}

data "aws_security_group" "eks_nodes" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-eks-nodes-sg"]  # SG từ Console
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}

data "aws_security_group" "alb" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-alb-sg"]  # SG từ Console
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}

# BƯỚC 3: Reference VPC Endpoints từ Console (đã tạo ở Section 1)
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

# BƯỚC 4: Variables cho integration
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
```

### 2.2. Terraform Outputs for Integration

**File: `aws/infra/outputs.tf`**

```hcl
# VPC Information (referenced from Console)
output "vpc_id" {
  description = "ID of the VPC"
  value       = data.aws_vpc.main.id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = data.aws_vpc.main.cidr_block
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = data.aws_subnets.private.ids
}

# VPC Endpoints (Terraform managed)
output "s3_vpc_endpoint_id" {
  description = "ID of the S3 VPC endpoint"
  value       = aws_vpc_endpoint.s3.id
}

output "ecr_api_vpc_endpoint_id" {
  description = "ID of the ECR API VPC endpoint"
  value       = aws_vpc_endpoint.ecr_api.id
}

output "ecr_dkr_vpc_endpoint_id" {
  description = "ID of the ECR DKR VPC endpoint"
  value       = aws_vpc_endpoint.ecr_dkr.id
}

output "logs_vpc_endpoint_id" {
  description = "ID of the CloudWatch Logs VPC endpoint"
  value       = aws_vpc_endpoint.logs.id
}

# Security Groups (referenced from Console)
output "vpc_endpoints_security_group_id" {
  description = "Security group ID for VPC endpoints"
  value       = data.aws_security_group.vpc_endpoints.id
}
```

## 3. Optional: Terraform Integration Setup

### 3.1. Khi nào cần Terraform cho Task 2

{{% notice info %}}
**💡 Chỉ cần Terraform nếu:**
- ✅ Bạn sẽ dùng Terraform cho Task 4-5 (EKS cluster, node groups)
- ✅ Cần integration outputs cho subsequent Terraform modules
- ✅ Muốn Infrastructure as Code cho production environments

**Nếu bạn dùng Console cho tất cả:** Skip phần này hoàn toàn!
{{% /notice %}}

```bash
# CHỈ chạy nếu cần integration với Terraform modules khác
cd aws/infra

# Initialize Terraform (reference existing resources only)
terraform init

# Plan - chỉ tạo data sources, không tạo resources mới
terraform plan -var-file="terraform.tfvars"

# Apply - import existing resources vào Terraform state
terraform apply -var-file="terraform.tfvars"
```

**Expected Output (No new resources created):**
```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Data Sources Created:
✅ VPC, subnets, Security Groups referenced
✅ VPC Endpoints referenced (đã tạo qua Console)
✅ Terraform outputs available cho Task 4-5

Note: Không có resources mới được tạo - chỉ reference existing infrastructure
```

### 3.2. VPC Endpoints via Console (Recommended)

Nếu muốn tạo VPC Endpoints qua Console:

**Bước 1: S3 Gateway Endpoint (FREE)**
1. VPC Dashboard → "Endpoints" → "Create endpoint"
2. Service: `com.amazonaws.ap-southeast-1.s3` (Type: Gateway)
3. Route Tables: Chọn private route table

![Create VPC Endpoint](../images/02-vpc-networking/10.1-create-vpc-endpoint.png)
![Endpoint Settings](../images/02-vpc-networking/10.2-endpoint-settings.png)
![S3 Gateway Endpoint](../images/02-vpc-networking/10.3-s3-gateway-endpoint.png)

**Bước 2: ECR API Interface Endpoint**
1. Service: `com.amazonaws.ap-southeast-1.ecr.api` (Type: Interface)
2. Subnets: Chọn cả 2 private subnets
3. Security Groups: VPC endpoints security group
4. Private DNS: ✅ Enabled

![ECR API Endpoint 1](../images/02-vpc-networking/10.4.1-ecr-api-endpoint.png)
![ECR API Endpoint 2](../images/02-vpc-networking/10.4.2-ecr-api-endpoint.png)
![ECR API Endpoint 3](../images/02-vpc-networking/10.4.3-ecr-api-endpoint.png)

**Bước 3: ECR DKR Interface Endpoint**
1. Service: `com.amazonaws.ap-southeast-1.ecr.dkr` (Type: Interface)
2. Subnets: Chọn cả 2 private subnets
3. Security Groups: VPC endpoints security group
4. Private DNS: ✅ Enabled

![ECR DKR Endpoint 1](../images/02-vpc-networking/10.5.1-ecr-dkr-endpoint.png)
![ECR DKR Endpoint 2](../images/02-vpc-networking/10.5.2-ecr-dkr-endpoint.png)
![ECR DKR Endpoint 3](../images/02-vpc-networking/10.5.3-ecr-dkr-endpoint.png)

**Bước 4: CloudWatch Logs Interface Endpoint**
1. Service: `com.amazonaws.ap-southeast-1.logs` (Type: Interface)
2. Subnets: Chọn cả 2 private subnets
3. Security Groups: VPC endpoints security group
4. Private DNS: ✅ Enabled

![CloudWatch Logs Endpoint 1](../images/02-vpc-networking/10.6.1-cloudwatch-logs-endpoint.png)
![CloudWatch Logs Endpoint 2](../images/02-vpc-networking/10.6.2-cloudwatch-logs-endpoint.png)
![CloudWatch Logs Endpoint 3](../images/02-vpc-networking/10.6.3-cloudwatch-logs-endpoint.png)

**Verification:**
![VPC Endpoints Overview](../images/02-vpc-networking/10.7-vpc-endpoints-overview.png)

## 4. Verification & Cost Analysis

### 4.1. VPC Endpoints Verification

```bash
# List VPC endpoints
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'VpcEndpoints[*].{Service:ServiceName,Type:VpcEndpointType,State:State}'

# Test S3 access from private subnet (sau khi deploy EKS)
# kubectl exec -it <pod> -- aws s3 ls s3://your-bucket/
```

### 4.2. Cost Comparison

| Component | Traditional NAT | VPC Endpoints | Savings |
|-----------|----------------|---------------|---------|
| **S3 Access** | NAT Gateway | S3 Gateway Endpoint (FREE) | $32/month |
| **ECR Access** | NAT Gateway | ECR API + DKR Endpoints | $17.6/month |
| **CloudWatch** | NAT Gateway | Logs Interface Endpoint | $17.2/month |
| **Total** | **$71/month** | **$21.6/month** | **70%** |

**Monthly Cost Breakdown:**
- S3 Gateway Endpoint: **FREE**
- ECR API Interface Endpoint: $7.2/month
- ECR DKR Interface Endpoint: $7.2/month  
- CloudWatch Logs Interface Endpoint: $7.2/month
- **Total: $21.6/month**

## 👉 Kết quả Task 2

✅ **VPC Infrastructure** (Console): Multi-AZ VPC với proper subnet design  
✅ **VPC Endpoints** (Console): Cost-optimized AWS services access  
✅ **Cost Optimization**: 70% savings ($21.6 vs $71/month NAT Gateway)  
✅ **Integration Ready**: Optional Terraform data sources cho subsequent tasks  

{{% notice success %}}
**🎯 Task 2 Complete!**

**Console Setup:** Complete VPC + VPC Endpoints (20 phút)  
**Optional Terraform:** Chỉ cho integration outputs (5 phút)  
**Result:** Production-ready VPC với 70% cost savings, no duplication!
{{% /notice %}}

### Architecture Achieved

```
✅ VPC Foundation (Console):
   - VPC: 10.0.0.0/16 (4 subnets across 2 AZ)
   - Internet Gateway và basic routing
   - Security Groups cho EKS, ALB, VPC Endpoints

✅ Cost Optimization (Terraform):
   - VPC Endpoints: S3 (FREE), ECR API/DKR, CloudWatch Logs
   - Total cost: $21.6/month vs $71/month NAT Gateway
   - 70% cost reduction
```

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 3**: IAM Roles & IRSA sử dụng VPC infrastructure
- **Task 4**: EKS cluster deployment với VPC Endpoints integration
- **Task 5**: EKS managed node groups trong cost-optimized private subnets
{{% /notice %}}

{{% notice info %}}
**🔧 Integration Strategy:**

**Console-created resources** được reference bởi Terraform:
- VPC ID, subnet IDs qua data sources
- Security Group IDs cho VPC Endpoints
- Route table IDs cho Gateway Endpoints

**Terraform outputs** cho subsequent tasks:
- VPC Endpoint IDs cho EKS configuration
- VPC/subnet information cho ALB, SageMaker modules
{{% /notice %}}