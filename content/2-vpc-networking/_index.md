---
title: "VPC - Subnet - NAT - Security Group"
date: 2025-08-30T11:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## Mục tiêu Task 2

Tạo networking foundation cho MLOps infrastructure:

1. **VPC Setup** - Tạo Virtual Private Cloud với multi-AZ cho high availability
2. **Subnet Design** - Public và Private subnets trong 2 Availability Zones
3. **NAT Gateway** - Cho phép private subnets truy cập Internet (pull images, updates)
4. **Security Groups** - Network access control với least privilege principles
5. **Terraform Infrastructure** - Infrastructure as Code với proper state management

{{% notice info %}}
**📋 Scope Task 2: VPC Networking Foundation**

Task này tạo network infrastructure base cho toàn bộ MLOps platform:
- ✅ VPC với multi-AZ design cho high availability
- ✅ Public subnets cho Load Balancers và NAT Gateways
- ✅ Private subnets cho EKS nodes và SageMaker
- ✅ Security Groups với proper inbound/outbound rules
- ✅ Terraform modules với reusable components
{{% /notice %}}

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
│   │   ├── outputs.tf           # Output values
│   │   ├── providers.tf         # AWS provider config
│   │   └── versions.tf          # Terraform version constraints
│   ├── modules/
│   │   └── vpc/                 # Reusable VPC module
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   └── terraform.tfvars         # Environment-specific values
```

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

## 3. Terraform Deployment

### 3.1. Initialize và Plan

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

### 3.2. Apply Infrastructure

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

## 4. Verification và Testing

### 4.1. AWS Console Verification

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

### 4.2. CLI Verification

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

### 4.3. Connectivity Testing

```bash
# Test Internet connectivity from private subnet (will be used in later tasks)
# This command will be used when EKS nodes are deployed

# For now, verify route tables are properly configured
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'RouteTables[*].{RouteTableId:RouteTableId,Routes:Routes[*].{Destination:DestinationCidrBlock,Target:GatewayId//NatGatewayId}}'
```

## 5. Cost Optimization Notes

### 5.1. Current Cost Impact

**Monthly Costs (ap-southeast-1):**
- **NAT Gateways**: 2 × $32 = $64/month
- **Elastic IPs**: 2 × $3.6 = $7.2/month  
- **Data Transfer**: Variable based on usage
- **Total Baseline**: ~$71/month

### 5.2. Cost Optimization Strategies

1. **Single NAT Gateway** (Dev Environment):
   ```hcl
   # For development, use single NAT Gateway
   resource "aws_nat_gateway" "main" {
     count = var.environment == "prod" ? length(var.public_subnets) : 1
     # ... rest of configuration
   }
   ```

2. **VPC Endpoints** (Future Tasks):
   - S3 Gateway Endpoint (Free)
   - ECR Interface Endpoint (Reduce NAT Gateway usage)

3. **Spot Instances** (EKS Nodes):
   - 60-90% cost savings for worker nodes
   - Will be configured in Task 5

## Kết quả Task 2

✅ **VPC Infrastructure**: Multi-AZ VPC với proper CIDR design  
✅ **Subnets**: Public/Private subnets trong 2 availability zones  
✅ **NAT Gateways**: High availability Internet access cho private subnets  
✅ **Security Groups**: Least privilege access rules cho EKS, ALB, SageMaker  
✅ **Terraform State**: Infrastructure as Code với proper outputs  

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 3**: IAM Roles & IRSA configuration
- **Task 4**: EKS cluster deployment sử dụng VPC infrastructure
- **Task 5**: EKS managed node groups trong private subnets
{{% /notice %}}

{{% notice warning %}}
**💰 Cost Reminder**: NAT Gateways là major cost component (~$71/month). Sẽ optimize bằng VPC Endpoints trong tasks sau.
{{% /notice %}}