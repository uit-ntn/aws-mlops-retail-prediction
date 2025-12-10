---
title: "Production Networking"
weight: 5
chapter: false
pre: "<b>5. </b>"
---

## 🎯 Task 5 Objectives

Set up **Production VPC** for EKS deployment and public API demo (separate from SageMaker training VPC):

1. **Production EKS Infrastructure** - EKS Cluster and Pods in private subnets
2. **Public API Access** - ALB in public subnets for demo endpoint `/predict`
3. **High-Performance Internal Networking** - VPC Endpoints for S3/ECR access < 50ms latency
4. **Cost Optimization** - Remove NAT Gateway, only enable ALB when demo

{{% notice warning %}}
**VPC Separation Strategy**

- **Task 4**: SageMaker training uses default VPC (Quick setup) - simple, cost-effective
- **Task 5**: EKS production uses separate VPC - better security, control
- **No conflict**: 2 independent VPCs, can connect via VPC Peering if needed
  {{% /notice %}}

<div class="notices info">
**🎯 Production VPC Architecture:**

- ✅ **Private Subnets**: EKS Pods (secure, no direct Internet)
- ✅ **Public Subnets**: ALB only (public API demo access)
- ✅ **Internal Communication**: EKS ↔ S3 via VPC Endpoints
- ✅ **Demo Ready**: Public API endpoint via ALB with SSL/health checks
</div>

📥 **Input**

- AWS Account with VPC permissions
- CIDR planning: `10.0.0.0/16` (production EKS VPC)
- Demo requirements: Public API access via ALB
- Task 4 completed: SageMaker training running in default VPC
- **Input from Task 4:** Task 4 (SageMaker training) — training VPC choices and requirements
- **Input from Task 2:** Task 2 (IAM Roles & Audit) — roles and policies required for VPC, EKS and ECR access

## 1. Hybrid VPC Infrastructure Setup

### 1.1. Create VPC

1. **Access VPC Dashboard:**
   - AWS Console → VPC service → "Create VPC"

![Create VPC Console](../images/05-vpc-networking/01-create-vpc-console.png)

2. **VPC Configuration:**
   ```
   VPC Name: mlops-retail-forecast-hybrid-vpc
   IPv4 CIDR: 10.0.0.0/16
   IPv6 CIDR: No IPv6 CIDR block
   Tenancy: Default
   Enable DNS hostnames: ✅ (Required for VPC Endpoints)
   Enable DNS resolution: ✅ (Required for VPC Endpoints)
   ```

---

### 1.2. Create Subnets

VPC → **Subnets** → **Create subnet**

#### Public Subnets (ALB only)

**Public Subnet 1 (ap-southeast-1a)**

```
Name: mlops-hybrid-public-alb-ap-southeast-1a
AZ: ap-southeast-1a
CIDR: 10.0.1.0/24
```

**Public Subnet 2 (ap-southeast-1b)**

```
Name: mlops-hybrid-public-alb-ap-southeast-1b
AZ: ap-southeast-1b
CIDR: 10.0.2.0/24
```

#### Private Subnets (EKS workloads)

**Private Subnet 1 (ap-southeast-1a)**

```
Name: mlops-eks-private-workloads-ap-southeast-1a
AZ: ap-southeast-1a
CIDR: 10.0.101.0/24
```

**Private Subnet 2 (ap-southeast-1b)**

```
Name: mlops-eks-private-workloads-ap-southeast-1b
AZ: ap-southeast-1b
CIDR: 10.0.102.0/24
```

---

### 1.3 Internet Gateway (for public ALB)

1. **Create Internet Gateway:**
   - "Internet Gateways" → "Create internet gateway"
   ```
   Name: mlops-hybrid-igw
   Purpose: Public access for ALB demo endpoint
   ```

![Internet Gateway Setup](../images/05-vpc-networking/04.1-internet-gateway.png)

2. **Attach to VPC:**
   - Select Internet Gateway → "Actions" → "Attach to VPC"
   - Select: mlops-retail-forecast-hybrid-vpc

![Attach IGW to VPC](../images/05-vpc-networking/04.2-internet-gateway.png)

### 1.4. Route Tables Configuration

#### 1.4.1. Public Route Table (ALB Traffic)

Create route table:

```
Name: mlops-hybrid-public-alb-rt
VPC: mlops-retail-forecast-hybrid-vpc
```

Add route:

```
0.0.0.0/0  →  mlops-hybrid-igw
```

Associate **both public subnets** to this route table.

#### Private Route Table (workload subnets)

Create route table:

```
Name: mlops-hybrid-private-workloads-rt
VPC: mlops-retail-forecast-hybrid-vpc
```

Keep only:

```
10.0.0.0/16 → local
```

**Do NOT add** `0.0.0.0/0` (no NAT, no IGW).

Associate **both private subnets** to this route table.

---

### 1.5 Enable “Auto-assign public IPv4” for Public Subnets

VPC → Subnets → select each **public** subnet → Actions → **Modify auto-assign IP settings**

- ✅ Enable auto-assign public IPv4 address

{{% notice warning %}}
If this is OFF, your Internet-facing ALB may fail or behave incorrectly.
{{% /notice %}}

---

## 2) Security Groups (Layered Model)

### 2.1 ALB Security Group (Internet → ALB)

```
Name: mlops-hybrid-alb-sg
VPC: mlops-retail-forecast-hybrid-vpc
```

Inbound:

- TCP 80 from `0.0.0.0/0` (optional; redirect to 443)
- TCP 443 from `0.0.0.0/0` (recommended)

Outbound:

- TCP 80 → **EKS Nodes SG** (or all traffic to EKS Nodes SG for simplicity in demo)

---

### 2.2 EKS Nodes SG (ALB → Nodes/Pods)

```
Name: mlops-hybrid-eks-nodes-sg
VPC: mlops-retail-forecast-hybrid-vpc
```

Inbound:

- TCP 80 from **ALB SG** (API traffic)
- TCP 443 from **ALB SG** (if you terminate TLS at pod/service)
- All traffic from **self** (node-to-node)
- All traffic from **EKS Control Plane SG** (cluster management)

Outbound:

- TCP 443 to **VPC Endpoints SG** (AWS services over PrivateLink)
- All traffic to **self**

---

### 2.3 EKS Control Plane SG

```
Name: mlops-hybrid-eks-control-plane-sg
VPC: mlops-retail-forecast-hybrid-vpc
```

Inbound:

- TCP 443 from **EKS Nodes SG**

Outbound:

- All traffic to **EKS Nodes SG**

---

### 2.4 VPC Endpoints SG

```
Name: mlops-hybrid-vpc-endpoints-sg
VPC: mlops-retail-forecast-hybrid-vpc
```

Inbound:

![VPC Endpoints Inbound Rules](../images/05-vpc-networking/09.9-vpc-endpoints-inbound.png)

{{% notice success %}}
**🎯 Security Groups Complete!**

**4 Security Groups Created:**

- ALB SG: Public Internet access (80/443)
- EKS Nodes SG: Private workloads
- EKS Control Plane SG: Cluster management
- VPC Endpoints SG: AWS services access

**Note:** SageMaker will use default SG in default VPC (Task 4)
{{% /notice %}}

### 1.6. Enable Auto-assign Public IP for ALB Subnets

**Important for ALB functionality:**

1. **Navigate to Public Subnets:**

   - VPC Dashboard → Subnets

2. **Enable Auto-assign Public IP:**
   - Select each public subnet
   - Actions → "Modify auto-assign IP settings"
   - ✅ Enable auto-assign public IPv4 address

![Auto-assign Public IP](../images/05-vpc-networking/06-auto-assign-public-ip.png)

{{% notice warning %}}
**⚠️ Critical for ALB:** Public subnets must have auto-assign public IP enabled, otherwise ALB creation will fail.
{{% /notice %}}

### 1.7. Console Setup Complete

![Security Groups Overview](../images/05-vpc-networking/10-security-groups-overview.png)
![VPC Resource Map](../images/05-vpc-networking/11-vpc-resource-map.png)

{{% notice success %}}
**🎯 Hybrid VPC Console Setup Complete!**

**Security Architecture:**

- **Layer 1**: Internet → ALB (80/443 from 0.0.0.0/0)
- **Layer 2**: ALB → EKS Nodes (controlled access)
- **Layer 3**: EKS → VPC Endpoints (AWS services only)
- **Layer 4**: Private subnets completely isolated from Internet

**Demo Ready:** ALB can accept public traffic and route to private EKS API pods
{{% /notice %}}

## 2. VPC Endpoints for High-Performance Internal Networking

**This step is MANDATORY to ensure EKS ↔ S3 ↔ SageMaker latency < 50ms:**

### 2.1. S3 Gateway Endpoint (FREE - Model/Data Access)

1. **Create S3 Gateway Endpoint:**
   - VPC Dashboard → "Endpoints" → "Create endpoint"
   ```
   Endpoint name: mlops-hybrid-s3-gateway-endpoint
   Service: com.amazonaws.ap-southeast-1.s3
   Type: Gateway
   VPC: mlops-retail-forecast-hybrid-vpc
   Route Tables: mlops-hybrid-private-workloads-rt
   Policy: Full Access (demo purposes)
   ```

---

**Purpose:** EKS Pods load model artifacts from S3 < 50ms latency

Create these as **Interface** endpoints:

#### ECR API

```
Name: mlops-hybrid-ecr-api-endpoint
Service: com.amazonaws.ap-southeast-1.ecr.api
Type: Interface
Subnets: both private subnets
Security group: mlops-hybrid-vpc-endpoints-sg
Private DNS: ✅ Enabled
```

#### ECR DKR

**Purpose:** EKS pull container images from ECR repository

### 2.3. ECR DKR Interface Endpoint (Docker Registry)

1. **Create ECR DKR Endpoint:**
   ```
   Endpoint name: mlops-hybrid-ecr-dkr-endpoint
   Service: com.amazonaws.ap-southeast-1.ecr.dkr
   Type: Interface
   VPC: mlops-retail-forecast-hybrid-vpc
   Subnets: Both private workload subnets
   Security Groups: mlops-hybrid-vpc-endpoints-sg
   Private DNS: ✅ Enabled
   ```

#### CloudWatch Logs

**Purpose:** Docker layer downloads for EKS container runtime

### 2.4. CloudWatch Logs Interface Endpoint

1. **Create CloudWatch Logs Endpoint:**
   ```
   Endpoint name: mlops-hybrid-logs-endpoint
   Service: com.amazonaws.ap-southeast-1.logs
   Type: Interface
   VPC: mlops-retail-forecast-hybrid-vpc
   Subnets: Both private workload subnets
   Security Groups: mlops-hybrid-vpc-endpoints-sg
   Private DNS: ✅ Enabled
   ```

![CloudWatch Logs Endpoint](../images/05-vpc-networking/10.7-logs-endpoint.png)
![CloudWatch Logs Endpoint](../images/05-vpc-networking/10.7-logs-endpoint2.png)

**Purpose:** Monitoring and logging for EKS/SageMaker workloads

### 2.5. VPC Endpoints Verification

![VPC Endpoints Overview](../images/05-vpc-networking/10.8-vpc-endpoints-overview.png)

**Expected Results:**

- **S3 Gateway**: Route added to private route table automatically
- **3x Interface Endpoints**: ENI created in each private subnet
- **Private DNS**: All endpoints resolvable via internal DNS

**🎯 VPC Endpoints Complete!**

**High-Performance Internal Network:**

- EKS ↔ S3: < 50ms (Gateway Endpoint)
- EKS ↔ ECR: < 50ms (Interface Endpoints)
- EKS ↔ CloudWatch: < 50ms (Logs Endpoint)
- No Internet dependency for AWS services

**Cost Optimized:** ~$7.2/month savings (no need for SageMaker VPC Endpoint)

## 3. Application Load Balancer Setup

### 3.1. Create Application Load Balancer

1. **Navigate to Load Balancers:**

   - EC2 Dashboard → Load Balancers → "Create load balancer"

2. **ALB Configuration:**
   ```
   Name: mlops-hybrid-api-demo-alb
   Scheme: Internet-facing
   IP address type: IPv4
   VPC: mlops-retail-forecast-hybrid-vpc
   Mappings: Both public ALB subnets
   Security groups: mlops-hybrid-alb-sg
   ```

#### 5.2 Create Target Group (for EKS later)

EC2 → Target Groups → **Create**

```
Target type: IP addresses
Name: mlops-hybrid-eks-api-tg
Protocol: HTTP
Port: 80
VPC: mlops-retail-forecast-hybrid-vpc
Health check path: /health
Success codes: 200
```

#### 5.3 Listener

- HTTP 80 → (optional) redirect to HTTPS 443
- HTTPS 443 → forward to target group (needs ACM cert if real TLS)

{{% notice info %}}
**💡 SSL Certificate Options:**

- **Production**: Use AWS Certificate Manager (ACM) with domain
- **Demo**: Create self-signed certificate or use HTTP only
- **Development**: Skip SSL, use HTTP listener only
  {{% /notice %}}

---

## 6) CLI Verification (CloudShell-friendly)

### 6.1 Verify VPC and Subnets

```bash
# Enable VPC Flow Logs for security monitoring
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name VPCFlowLogs \
  --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/flowlogsRole
```

### 4.2. Network Performance Testing

**Test EKS ↔ S3 Latency:**

```bash
# From EKS Pod (after cluster setup)
kubectl run test-pod --image=amazonlinux --restart=Never -- sleep 3600
kubectl exec -it test-pod -- bash

# Inside pod
yum install -y awscli
time aws s3 ls s3://your-model-bucket/
# Expected: < 50ms for VPC Endpoint
```

**Test ALB ↔ EKS Connectivity:**

```bash
# Test from outside VPC
curl -I http://your-alb-dns-name/health
# Expected: HTTP 200 OK when EKS API is running
```

### 4.3. Cost Monitoring Setup

**CloudWatch Cost Alarms:**

```bash
# Create cost alarm for VPC Endpoints
aws cloudwatch put-metric-alarm \
  --alarm-name "VPC-Endpoints-Cost-Alert" \
  --alarm-description "Alert when VPC Endpoints cost > $30/month" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 30 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD Name=ServiceName,Value=AmazonVPC
```

## 5. Terraform Outputs

{{% notice info %}}
**💡 When to use Terraform outputs:**

- ✅ Task 7-10 will use Terraform (EKS cluster, ALB controller)
- ✅ Need automated deployment across environments
- ✅ Want to reference infrastructure programmatically

**If Task 7-10 use Console:** Skip this section completely!
{{% /notice %}}

### 5.1. Data Sources (Reference Console-created Resources)

**File: `aws/infra/vpc-data-sources.tf`**

```hcl
# Reference VPC infrastructure from Console
data "aws_vpc" "hybrid" {
  filter {
    name   = "tag:Name"
    values = ["mlops-retail-forecast-hybrid-vpc"]
  }
}

# Public subnets (ALB)
data "aws_subnets" "public_alb" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.hybrid.id]
  }
  filter {
    name   = "tag:Name"
    values = ["*public-alb*"]
  }
}

# Private subnets (EKS + SageMaker)
data "aws_subnets" "private_workloads" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.hybrid.id]
  }
  filter {
    name   = "tag:Name"
    values = ["*private-workloads*"]
  }
}

# Security Groups
data "aws_security_group" "alb" {
  filter {
    name   = "tag:Name"
    values = ["mlops-hybrid-alb-sg"]
  }
}

data "aws_security_group" "eks_nodes" {
  filter {
    name   = "tag:Name"
    values = ["mlops-hybrid-eks-nodes-sg"]
  }
}

data "aws_security_group" "eks_control_plane" {
  filter {
    name   = "tag:Name"
    values = ["mlops-hybrid-eks-control-plane-sg"]
  }
}

# ALB
data "aws_lb" "api_demo" {
  name = "mlops-hybrid-api-demo-alb"
}

data "aws_lb_target_group" "eks_api" {
  name = "mlops-hybrid-eks-api-tg"
}

# VPC Endpoints
data "aws_vpc_endpoint" "s3" {
  vpc_id       = data.aws_vpc.hybrid.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
}

data "aws_vpc_endpoint" "ecr_api" {
  vpc_id       = data.aws_vpc.hybrid.id
  service_name = "com.amazonaws.ap-southeast-1.ecr.api"
}

data "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id       = data.aws_vpc.hybrid.id
  service_name = "com.amazonaws.ap-southeast-1.ecr.dkr"
}

data "aws_vpc_endpoint" "sagemaker_runtime" {
  vpc_id       = data.aws_vpc.hybrid.id
  service_name = "com.amazonaws.ap-southeast-1.sagemaker.runtime"
}
```

### 5.2. Outputs for EKS/ALB Integration

**File: `aws/infra/networking-outputs.tf`**

```hcl
# VPC Information
output "vpc_id" {
  description = "Hybrid VPC ID"
  value       = data.aws_vpc.hybrid.id
}

output "vpc_cidr_block" {
  description = "VPC CIDR block"
  value       = data.aws_vpc.hybrid.cidr_block
}

# Subnet Information
output "public_alb_subnet_ids" {
  description = "Public subnet IDs for ALB"
  value       = data.aws_subnets.public_alb.ids
}

output "private_workload_subnet_ids" {
  description = "Private subnet IDs for EKS and SageMaker"
  value       = data.aws_subnets.private_workloads.ids
}

# Security Group Information
output "alb_security_group_id" {
  description = "ALB Security Group ID"
  value       = data.aws_security_group.alb.id
}

output "eks_nodes_security_group_id" {
  description = "EKS Nodes Security Group ID"
  value       = data.aws_security_group.eks_nodes.id
}

output "eks_control_plane_security_group_id" {
  description = "EKS Control Plane Security Group ID"
  value       = data.aws_security_group.eks_control_plane.id
}

# ALB Information
output "alb_arn" {
  description = "ALB ARN for API demo"
  value       = data.aws_lb.api_demo.arn
}

output "alb_dns_name" {
  description = "ALB DNS name for public API access"
  value       = data.aws_lb.api_demo.dns_name
}

output "alb_zone_id" {
  description = "ALB Zone ID for Route53 integration"
  value       = data.aws_lb.api_demo.zone_id
}

output "eks_api_target_group_arn" {
  description = "Target group ARN for EKS API pods"
  value       = data.aws_lb_target_group.eks_api.arn
}

# VPC Endpoints Information
output "s3_vpc_endpoint_id" {
  description = "S3 VPC Endpoint ID"
  value       = data.aws_vpc_endpoint.s3.id
}

output "ecr_api_vpc_endpoint_id" {
  description = "ECR API VPC Endpoint ID"
  value       = data.aws_vpc_endpoint.ecr_api.id
}

output "sagemaker_runtime_vpc_endpoint_id" {
  description = "SageMaker Runtime VPC Endpoint ID"
  value       = data.aws_vpc_endpoint.sagemaker_runtime.id
}

# Demo Configuration
output "api_demo_config" {
  description = "Configuration for API demo"
  value = {
    public_endpoint = "https://${data.aws_lb.api_demo.dns_name}"
    health_check    = "https://${data.aws_lb.api_demo.dns_name}/health"
    predict_endpoint = "https://${data.aws_lb.api_demo.dns_name}/predict"
  }
}
```

### 5.3. Deploy Terraform Outputs
```bash
# Navigate to infrastructure directory
cd aws/infra

# Initialize Terraform
terraform init

# Plan (should show 0 resources to create, only data sources)
terraform plan

# Apply outputs
terraform apply -auto-approve

# Verify outputs
terraform output api_demo_config
```

Expected output:

```json
{
  "health_check" = "https://mlops-hybrid-api-demo-alb-123456789.ap-southeast-1.elb.amazonaws.com/health"
  "predict_endpoint" = "https://mlops-hybrid-api-demo-alb-123456789.ap-southeast-1.elb.amazonaws.com/predict"
  "public_endpoint" = "https://mlops-hybrid-api-demo-alb-123456789.ap-southeast-1.elb.amazonaws.com"
}
```

## 6. Verification & Performance Testing

### 6.1. Network Architecture Verification

**Verify Hybrid VPC Setup:**

```bash
# Check VPC configuration
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=mlops-retail-forecast-hybrid-vpc" \
  --query 'Vpcs[0].{VpcId:VpcId,Cidr:CidrBlock,State:State}' --output table

aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,MapPublicIp:MapPublicIpOnLaunch}' \
  --output table
```

### 6.2 Verify Route Tables

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'RouteTables[*].{RT:RouteTableId,Routes:Routes}' \
  --output json
```

### 6.3 Verify Endpoints

```bash
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'VpcEndpoints[*].{Service:ServiceName,PrivateDnsEnabled:PrivateDnsEnabled}'
```

### 7.2. ALB Issues

**Problem: ALB not accessible from Internet**

```bash
# Check Internet Gateway attached
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$(terraform output -raw vpc_id)"

# Check public subnet route table
aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=mlops-hybrid-public-alb-rt" \
  --query 'RouteTables[0].Routes'
# Should have 0.0.0.0/0 → Internet Gateway
```

**Problem: ALB can't reach EKS targets**

```bash
# Check security group rules
aws ec2 describe-security-groups \
  --group-names mlops-hybrid-eks-nodes-sg \
  --query 'SecurityGroups[0].IpPermissions[?IpProtocol==`tcp` && FromPort==`80`]'
# Should allow port 80 from ALB security group
```

### 7.3. Performance Issues

**Problem: High latency EKS ↔ S3**

```bash
# Verify using VPC endpoint
kubectl exec -it network-test -- traceroute s3.ap-southeast-1.amazonaws.com
# Should not go through Internet (no public IPs in trace)

# Check VPC endpoint policy
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.ap-southeast-1.s3" \
  --query 'VpcEndpoints[0].PolicyDocument'
```

## 👉 Task 5 Results

✅ **Hybrid VPC Architecture** - Public ALB + Private workloads security model  
✅ **Public API Demo Ready** - ALB configured for demo endpoint `/predict`  
✅ **High-Performance Internal** - VPC Endpoints < 50ms latency EKS ↔ S3  
✅ **Cost Optimized** - $21.6/month base + $0.02/hour ALB demo usage  
✅ **Production Security** - Layered security groups, no Internet access for workloads

### Architecture Delivered

```
✅ Hybrid VPC Foundation:
   - Public Subnets: ALB demo access (Internet facing)
   - Private Subnets: EKS + SageMaker (secure, no Internet)
   - Multi-AZ: High availability for API demo

✅ Public Demo Capability:
   - ALB: Public endpoint for API demo
   - Target Groups: Ready for EKS API pods
   - SSL/Health checks: Production-ready demo

✅ High-Performance Internal Network:
   - S3 Gateway Endpoint: FREE, < 50ms model access
   - ECR Interface Endpoints: Fast container pulls
   - SageMaker Runtime Endpoint: Low-latency inference

✅ Security Architecture:
   Internet → ALB SG → EKS SG → VPC Endpoints SG
   (Layered access control)
```

{{% notice success %}}
**🎯 Task 5 Complete - Demo-Ready Hybrid VPC!**

**Public Access**: ALB provides secure public API demo endpoint  
**Private Security**: EKS/SageMaker workloads completely isolated  
**High Performance**: < 50ms internal AWS services latency  
**Cost Efficient**: $21.6 base + demo usage only when needed  
**Production Ready**: SSL, health checks, multi-AZ availability  
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 6**: ECR container registry for API container images
- **Task 7**: EKS cluster deployment in private subnets
- **Task 8**: EKS node groups with auto-scaling
- **Task 10**: Deploy API service with ALB integration
- **Task 11**: ALB ingress controller configuration

**Demo Commands Ready:**

```bash
# Public API demo endpoint (after deployment)
curl https://your-alb-dns/predict -d '{"data": "your-input"}'

# Health check endpoint
curl https://your-alb-dns/health
```

{{% /notice %}}

{{% notice info %}}
**📊 Performance Benchmarks Achieved:**

- **EKS ↔ S3 Latency**: < 50ms (VPC Gateway Endpoint)
- **EKS ↔ ECR Latency**: < 100ms (Interface Endpoints)
- **ALB ↔ EKS Latency**: < 10ms (same VPC)
- **Internet ↔ ALB**: Standard Internet latency
- **Cost**: $21.6/month base + $0.02/hour demo usage
- **Availability**: Multi-AZ (99.99% SLA)
  {{% /notice %}}

---

**Next Step**: [Task 6: ECR Container Registry](../6-ecr-registry/)

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 3**: IAM Roles & IRSA using VPC infrastructure
- **Task 4**: EKS cluster deployment with VPC Endpoints integration
- **Task 5**: EKS managed node groups in cost-optimized private subnets
  {{% /notice %}}

## 8. Clean Up Resources (AWS CLI)

### 8.1. Delete Application Load Balancer and Target Groups

```bash
# List ALB
aws elbv2 describe-load-balancers --names mlops-hybrid-api-demo-alb --query 'LoadBalancers[*].[LoadBalancerArn,DNSName]' --output table

# Delete ALB (automatically deletes listeners)
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>

# Delete Target Groups
aws elbv2 describe-target-groups --names mlops-hybrid-eks-api-tg --query 'TargetGroups[*].TargetGroupArn' --output text | xargs -I {} aws elbv2 delete-target-group --target-group-arn {}
```

### 8.2. Delete VPC Endpoints

```bash
# List VPC Endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>" --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName]' --output table

# Delete Interface Endpoints (ECR, CloudWatch Logs)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <ecr-api-endpoint-id> <ecr-dkr-endpoint-id> <logs-endpoint-id>

# Delete Gateway Endpoint (S3)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <s3-gateway-endpoint-id>
```

### 8.3. Delete Security Groups

```bash
# List Security Groups (except default)
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>" --query 'SecurityGroups[?GroupName!=`default`].[GroupId,GroupName]' --output table

# Delete Security Groups (in reverse dependency order)
aws ec2 delete-security-group --group-id <vpc-endpoints-sg-id>
aws ec2 delete-security-group --group-id <eks-control-plane-sg-id>
aws ec2 delete-security-group --group-id <eks-nodes-sg-id>
aws ec2 delete-security-group --group-id <alb-sg-id>
```

### 8.4. Delete Subnets and Route Tables

```bash
# List Route Tables (except main)
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>" --query 'RouteTables[?Associations[0].Main!=`true`].[RouteTableId,Tags[0].Value]' --output table

# Delete Route Tables
aws ec2 delete-route-table --route-table-id <public-rt-id>
aws ec2 delete-route-table --route-table-id <private-rt-id>

# List Subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --query 'Subnets[*].[SubnetId,Tags[0].Value,CidrBlock]' --output table

# Delete Subnets
aws ec2 delete-subnet --subnet-id <public-subnet-1a-id>
aws ec2 delete-subnet --subnet-id <public-subnet-1b-id>
aws ec2 delete-subnet --subnet-id <private-subnet-1a-id>
aws ec2 delete-subnet --subnet-id <private-subnet-1b-id>
```

### 8.5. Delete Internet Gateway and VPC

```bash
# Detach and delete Internet Gateway
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=<vpc-id>" --query 'InternetGateways[*].InternetGatewayId' --output text | xargs -I {} aws ec2 detach-internet-gateway --internet-gateway-id {} --vpc-id <vpc-id>

aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>

# Delete VPC (last)
aws ec2 delete-vpc --vpc-id <vpc-id>

# Verify clean up
aws ec2 describe-vpcs --vpc-ids <vpc-id>
```

### 8.6. Clean Up Helper Script

```bash
#!/bin/bash
# vpc-cleanup.sh

VPC_ID="vpc-xxxxxxxxx"  # Replace with actual VPC ID

echo "🧹 Cleaning up VPC resources for $VPC_ID..."

# 1. Delete ALB and Target Groups
echo "Deleting ALB..."
ALB_ARN=$(aws elbv2 describe-load-balancers --names mlops-hybrid-api-demo-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text 2>/dev/null)
if [ "$ALB_ARN" != "None" ] && [ ! -z "$ALB_ARN" ]; then
    aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
    echo "ALB deleted: $ALB_ARN"
fi

# 2. Delete VPC Endpoints
echo "Deleting VPC Endpoints..."
ENDPOINTS=$(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query 'VpcEndpoints[*].VpcEndpointId' --output text)
for endpoint in $ENDPOINTS; do
    aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $endpoint
    echo "VPC Endpoint deleted: $endpoint"
done

# 3. Wait for resources to be deleted
echo "Waiting for resources to be deleted..."
sleep 60

# 4. Delete Security Groups
echo "Deleting Security Groups..."
SECURITY_GROUPS=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[?GroupName!=`default`].GroupId' --output text)
for sg in $SECURITY_GROUPS; do
    aws ec2 delete-security-group --group-id $sg 2>/dev/null
    echo "Security Group deleted: $sg"
done

echo "✅ VPC cleanup completed for $VPC_ID"
```

---

## 9. VPC and Networking Pricing Table (ap-southeast-1)

### 9.1. VPC Core Components Cost

| Component | Price (USD) | Notes |
|-----------|-----------|---------|
| **VPC** | Free | Unlimited VPCs |
| **Subnets** | Free | Unlimited subnets |
| **Route Tables** | Free | Routing configuration |
| **Internet Gateway** | Free | One per VPC |
| **Security Groups** | Free | Firewall rules |

### 9.2. VPC Endpoints Cost

| Endpoint Type | Price (USD/hour) | Price (USD/month) | Data Transfer |
|---------------|----------------|-----------------|---------------|
| **Gateway Endpoint (S3)** | Free | Free | Free |
| **Interface Endpoint** | $0.01 | $7.2 | $0.01/GB |
| **PrivateLink Endpoint** | $0.01 | $7.2 | $0.01/GB |

**VPC Endpoints cost for Task 5:**
- S3 Gateway: Free
- ECR API Interface: $7.2/month
- ECR DKR Interface: $7.2/month  
- CloudWatch Logs Interface: $7.2/month
- **Total:** $21.6/month

### 9.3. Application Load Balancer Cost

| Component | Price (USD/hour) | Price (USD/month) | Notes |
|-----------|----------------|-----------------|---------|
| **ALB Fixed Cost** | $0.0225 | $16.2 | Always running |
| **LCU (Load Balancer Capacity Unit)** | $0.008 | $5.76 | Per LCU-hour |
| **Rule Evaluations** | $0.008 | $5.76 | Per million requests |

**Estimated ALB cost:**
- Base ALB: $16.2/month
- 1 LCU (basic usage): $5.76/month
- **Total ALB:** ~$22/month continuous

### 9.4. NAT Gateway Cost (Not used in Task 5)

| Component | Price (USD/hour) | Price (USD/month) | Data Transfer |
|-----------|----------------|-----------------|---------------|
| **NAT Gateway** | $0.045 | $32.4 | $0.045/GB |
| **Data Processing** | | | $0.045/GB |

**Savings:** $32.4/month by using VPC Endpoints instead of NAT Gateway

### 9.5. Data Transfer Pricing

| Transfer Type | Price (USD/GB) | Notes |
|---------------|--------------|---------|
| **VPC Internal** | Free | Same AZ |
| **Cross-AZ** | $0.01 | Different AZ in region |
| **VPC Endpoints** | $0.01 | Interface endpoints |
| **Internet OUT** | $0.12 | First 1GB free/month |
| **S3 Transfer** | Free | Via Gateway endpoint |

### 9.6. Estimated Total Cost for Task 5

**Monthly Baseline Cost:**

| Component | Monthly Cost | Purpose |
|-----------|--------------|---------|
| VPC + Subnets + IGW | $0 | Core networking |
| VPC Endpoints (3x Interface) | $21.6 | ECR + CloudWatch |
| S3 Gateway Endpoint | $0 | Model access |
| **Subtotal** | **$21.6** | Always running |

**Demo Usage Cost:**

| Usage Pattern | ALB Cost | Total Cost | Use Case |
|---------------|----------|------------|----------|
| **Development (8h/day)** | $5.4/month | $27/month | Daily development |
| **Demo only (3h/day)** | $2.0/month | $23.6/month | Presentation demos |
| **Production (24/7)** | $22/month | $43.6/month | Live production |
| **Testing (1h/day)** | $0.7/month | $22.3/month | Occasional testing |

### 9.7. Cost Comparison with Traditional Setup

**Task 5 (VPC Endpoints)** vs **Traditional (NAT Gateway)**:

| Architecture | Monthly Cost | Performance | Security |
|--------------|--------------|-------------|----------|
| **VPC Endpoints** | $21.6 | < 50ms latency | Private network |
| **NAT Gateway** | $32.4 + data | Variable | Internet routing |
| **Savings** | **-$10.8** | **Better** | **Higher** |

### 9.8. Cost Optimization Tips

**Immediate Savings:**
- ✅ Use S3 Gateway Endpoint (Free instead of $7.2/month Interface)
- ✅ Skip NAT Gateway (-$32.4/month)
- ✅ Turn off ALB when not demoing (-$22/month)

**Long-term Optimization:**
- Use Spot instances for EKS nodes (60-70% savings)
- S3 Intelligent Tiering for model storage
- CloudWatch Logs retention policy (7-30 days)

**Demo Cost Management:**
```bash
# Enable ALB only when demo
aws elbv2 create-load-balancer --name demo-alb --type application

# Disable ALB after demo  
aws elbv2 delete-load-balancer --load-balancer-arn <arn>
```

{{% notice info %}}
**💰 Cost Summary for Task 5:**
- **Baseline:** $21.6/month (VPC Endpoints, always on)
- **Demo usage:** $0.02/hour ALB (only when needed)
- **Savings:** $10.8/month compared to NAT Gateway approach
- **Performance:** < 50ms internal latency guaranteed
{{% /notice %}}

---

{{% notice info %}}

**_Console-created resources_** ready for subsequent tasks:

- VPC ID, subnet IDs for EKS cluster creation
- Security Group IDs for EKS and ALB configuration
- VPC Endpoint IDs for cost-optimized AWS services access

{{% /notice %}}

---

## 📹 Task 5 Implementation Video

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=rVhZpuD1dgE&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=4" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 06: ECR Registry](../6-ecr-registry)
