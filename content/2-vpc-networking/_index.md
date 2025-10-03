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

## 2. Terraform cho VPC Endpoints Cost Optimization

{{% notice info %}}
**💡 Tại sao cần Terraform cho VPC Endpoints:**
- ✅ **Complex configuration** - Multiple endpoints với specific settings
- ✅ **Cost optimization** - Automated endpoint management
- ✅ **Integration outputs** - Structured data cho EKS, ALB modules
- ✅ **Environment consistency** - Reproducible across dev/staging/prod

**Console setup VPC Endpoints rất phức tạp và dễ sai sót**
{{% /notice %}}

### 2.1. VPC Endpoints Configuration

**File: `aws/infra/vpc-endpoints.tf`**

```hcl
# Data sources - reference existing VPC từ Console
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-vpc"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Name"
    values = ["*private*"]
  }
}

data "aws_route_tables" "private" {
  vpc_id = data.aws_vpc.main.id
  filter {
    name   = "tag:Name"
    values = ["*private*"]
  }
}

data "aws_security_group" "vpc_endpoints" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-dev-vpc-endpoints-sg"]
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}

# Variables
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
    Project     = "MLOpsRetailForecast"
    Environment = "dev"
    ManagedBy   = "Terraform"
    Component   = "vpc-endpoints"
  }
}

# S3 Gateway Endpoint (FREE - no hourly charges)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = data.aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
  
  # Associate với private route tables
  route_table_ids = data.aws_route_tables.private.ids
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-s3-endpoint"
    Type = "gateway-endpoint"
    Cost = "free"
  })
}

# ECR API Endpoint - cho Docker registry API calls
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = data.aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = data.aws_subnets.private.ids
  security_group_ids  = [data.aws_security_group.vpc_endpoints.id]
  
  # Enable DNS resolution
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-api-endpoint"
    Type = "interface-endpoint"
    Cost = "7.2-usd-monthly"
  })
}

# ECR DKR Endpoint - cho Docker image pulls
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = data.aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = data.aws_subnets.private.ids
  security_group_ids  = [data.aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-ecr-dkr-endpoint"
    Type = "interface-endpoint"
    Cost = "7.2-usd-monthly"
  })
}

# CloudWatch Logs Endpoint - cho logging
resource "aws_vpc_endpoint" "logs" {
  vpc_id              = data.aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = data.aws_subnets.private.ids
  security_group_ids  = [data.aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-logs-endpoint"
    Type = "interface-endpoint"
    Cost = "7.2-usd-monthly"
  })
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

## 3. VPC Endpoints Deployment

### 3.1. Terraform Deployment

```bash
# Navigate to infrastructure directory
cd aws/infra

# Initialize Terraform (if first time)
terraform init

# Plan VPC Endpoints deployment
terraform plan -target=aws_vpc_endpoint.s3 \
               -target=aws_vpc_endpoint.ecr_api \
               -target=aws_vpc_endpoint.ecr_dkr \
               -target=aws_vpc_endpoint.logs \
               -var-file="terraform.tfvars"

# Apply VPC Endpoints
terraform apply -target=aws_vpc_endpoint.s3 \
                -target=aws_vpc_endpoint.ecr_api \
                -target=aws_vpc_endpoint.ecr_dkr \
                -target=aws_vpc_endpoint.logs \
                -var-file="terraform.tfvars"
```

**Expected Output:**
```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

VPC Endpoints Created:
✅ S3 Gateway Endpoint (FREE)
✅ ECR API Interface Endpoint ($7.2/month)
✅ ECR DKR Interface Endpoint ($7.2/month)
✅ CloudWatch Logs Interface Endpoint ($7.2/month)

Total Monthly Cost: ~$21.6 (vs $71 with NAT Gateway)
Cost Savings: 70% reduction
```

### 3.2. Manual VPC Endpoints via Console (Alternative)

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

✅ **Basic VPC Infrastructure** (Console): Multi-AZ VPC với proper subnet design  
✅ **VPC Endpoints** (Terraform): Cost-optimized AWS services access  
✅ **Cost Optimization**: 70% savings ($21.6 vs $71/month)  
✅ **Integration Ready**: Terraform outputs cho EKS, ALB, SageMaker modules  

{{% notice success %}}
**🎯 Task 2 Optimized!**

**Console Setup:** Quick VPC foundation (15 phút)  
**Terraform VPC Endpoints:** Cost optimization và automation (5 phút)  
**Result:** Production-ready VPC với 70% cost savings
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