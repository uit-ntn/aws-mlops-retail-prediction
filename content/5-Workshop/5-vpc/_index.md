---
title: "Production Networking"
weight: 5
chapter: false
pre: "<b>5. </b>"
---

<<<<<<< HEAD
## 🎯 Task 5 Objectives

Set up **Production VPC** for EKS deployment and public API demo (separate from SageMaker training VPC):

1. **Production EKS Infrastructure** - EKS Cluster and Pods in private subnets
2. **Public API Access** - ALB in public subnets for demo endpoint `/predict`
3. **High-Performance Internal Networking** - VPC Endpoints for S3/ECR access < 50ms latency
4. **Cost Optimization** - Remove NAT Gateway, only enable ALB when demo
=======
## 🎯 Task 5 Goal

Build a **Production VPC** for your **EKS deployment** and a **public demo API** (separate from the SageMaker training VPC):

1. **Production EKS Infrastructure** — EKS nodes/pods in **private subnets**
2. **Public Demo Access** — **ALB in public subnets** for `/predict`
3. **Fast Private Access to AWS Services** — VPC Endpoints for **S3/ECR/CloudWatch Logs**
4. **Cost Optimization** — **No NAT Gateway**, ALB **enabled only during demos**
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

{{% notice warning %}}
**VPC Separation Strategy**

<<<<<<< HEAD
- **Task 4**: SageMaker training uses default VPC (Quick setup) - simple, cost-effective
- **Task 5**: EKS production uses separate VPC - better security, control
- **No conflict**: 2 independent VPCs, can connect via VPC Peering if needed
=======
- **Task 4**: SageMaker training can stay on default / quick setup VPC (simple, fast)
- **Task 5**: EKS production runs in a **separate VPC** (security + better control)
- They do **not conflict**. If needed later, connect via VPC Peering / Transit Gateway.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
  {{% /notice %}}

---

<<<<<<< HEAD
- ✅ **Private Subnets**: EKS Pods (secure, no direct Internet)
- ✅ **Public Subnets**: ALB only (public API demo access)
- ✅ **Internal Communication**: EKS ↔ S3 via VPC Endpoints
- ✅ **Demo Ready**: Public API endpoint via ALB with SSL/health checks
{{% /notice %}}
=======
## ✅ Target Architecture (high-level)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

- **VPC**: `10.0.0.0/16`
- **Public subnets (2 AZs)**: ALB only
- **Private subnets (2 AZs)**: EKS worker nodes + pods
- **No NAT**: private workloads have **no direct Internet**
- **VPC Endpoints**:
  - **S3 Gateway Endpoint** (free hourly) for model/data access
  - **ECR API + ECR DKR** Interface Endpoints (pull images privately)
  - **CloudWatch Logs** Interface Endpoint (push logs privately)

<<<<<<< HEAD
- AWS Account with VPC permissions
- CIDR planning: `10.0.0.0/16` (production EKS VPC)
- Demo requirements: Public API access via ALB
- Task 4 completed: SageMaker training running in default VPC
- **Input from Task 4:** Task 4 (SageMaker training) — training VPC choices and requirements
- **Input from Task 2:** Task 2 (IAM Roles & Audit) — roles and policies required for VPC, EKS and ECR access
=======
---
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

## 1) Production VPC Setup

<<<<<<< HEAD
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

![VPC Configuration](../images/05-vpc-networking/02-vpc-configuration.png)

### 1.2. Create Subnets

1. **Public Subnets (ALB Only):**

   - Navigate to "Subnets" → "Create subnet"

**Public Subnet 1 (ap-southeast-1a):**
=======
### 1.1 Create the VPC

AWS Console → **VPC** → **Create VPC**
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```
VPC Name: mlops-retail-forecast-hybrid-vpc
IPv4 CIDR: 10.0.0.0/16
IPv6 CIDR: None
Tenancy: Default
Enable DNS hostnames: ✅ (Required for Interface Endpoints Private DNS)
Enable DNS resolution: ✅
```

---

### 1.2 Create Subnets (Multi-AZ)

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

<<<<<<< HEAD
1. **Create Internet Gateway:**
   - "Internet Gateways" → "Create internet gateway"
   ```
   Name: mlops-hybrid-igw
   Purpose: Public access for ALB demo endpoint
   ```
=======
---
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 1.3 Internet Gateway (for public ALB)

VPC → **Internet Gateways** → **Create internet gateway**

```
Name: mlops-hybrid-igw
```

Attach it to:

```
VPC: mlops-retail-forecast-hybrid-vpc
```

---

### 1.4 Route Tables

#### Public Route Table (ALB subnets)

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

<<<<<<< HEAD
2. **Inbound Rules (Public Demo Access):**

   - **Rule 1**: HTTP (80) from 0.0.0.0/0 (redirect to HTTPS)
   - **Rule 2**: HTTPS (443) from 0.0.0.0/0 (secure API demo)

3. **Outbound Rules:**
   - **Rule**: All traffic to EKS Security Group (ALB → EKS communication)

![ALB Security Group Basic](../images/05-vpc-networking/09.1-alb-security-group.png)

#### 1.5.2. EKS Worker Nodes Security Group (Private Workloads)

1. **Basic Details:**

   ```
   Security group name: mlops-hybrid-eks-nodes-sg
   Description: Security group for EKS worker nodes in private subnets
   VPC: mlops-retail-forecast-hybrid-vpc
   ```

2. **Inbound Rules (Controlled Access):**

   - **Rule 1**: HTTP (80) from ALB Security Group (API traffic)
   - **Rule 2**: HTTPS (443) from ALB Security Group (secure API traffic)
   - **Rule 3**: All Traffic from self (inter-node communication)
   - **Rule 4**: All Traffic from EKS Control Plane SG (cluster management)

3. **Outbound Rules:**
   - **Rule 1**: HTTPS (443) to VPC Endpoints SG (AWS services access)
   - **Rule 2**: All Traffic to self (inter-node communication)

![EKS Nodes Security Group](../images/05-vpc-networking/09.4-eks-nodes-basic.png)

#### 1.5.3. EKS Control Plane Security Group

1. **Basic Details:**

   ```
   Security group name: mlops-hybrid-eks-control-plane-sg
   Description: Security group for EKS control plane
   VPC: mlops-retail-forecast-hybrid-vpc
   ```

2. **Inbound Rules:**

   - **Rule**: HTTPS (443) from EKS Nodes SG (cluster API access)

3. **Outbound Rules:**
   - **Rule**: All Traffic to EKS Nodes SG (cluster management)

![EKS Control Plane Security Group](../images/05-vpc-networking/09.7-eks-control-plane.png)

#### 1.5.4. VPC Endpoints Security Group (AWS Services Access)

1. **Basic Details:**

   ```
   Security group name: mlops-hybrid-vpc-endpoints-sg
   Description: Security group for VPC endpoints (S3, ECR, SageMaker)
   VPC: mlops-retail-forecast-hybrid-vpc
   ```

2. **Inbound Rules (Internal Access Only):**
   - **Rule 1**: HTTPS (443) from EKS Nodes SG (EKS → AWS services)
   - **Rule 2**: HTTPS (443) from SageMaker SG (SageMaker → S3/ECR)
   - **Rule 3**: HTTPS (443) from VPC CIDR (10.0.0.0/16) - fallback

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
=======
- ✅ Enable auto-assign public IPv4 address
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

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

- TCP 443 from **EKS Nodes SG**
- (Optional) TCP 443 from `10.0.0.0/16` (fallback for debugging)

Outbound:

- All traffic (default ok)

---

## 3) REQUIRED: Subnet Tags (so EKS + Load Balancers work later)

Even if you create subnets via Console, **tagging is critical** for later tasks (EKS + AWS Load Balancer Controller).

### Public subnets (ALB)

Add tag:

```
kubernetes.io/role/elb = 1
```

### Private subnets (internal LB / node placement)

Add tag:

```
kubernetes.io/role/internal-elb = 1
```

### Cluster ownership tag (add later after cluster name is final)

When your cluster name is known (Task 7), add on **all subnets used by the cluster**:

```
kubernetes.io/cluster/<your-cluster-name> = shared
```

Example:

```
kubernetes.io/cluster/mlops-retail-cluster = shared
```

---

## 4) VPC Endpoints (No NAT, Private AWS Access)

{{% notice warning %}}
If you run **private subnets with NO NAT**, then **endpoints are not optional**—without them, nodes/pods will fail to pull images, write logs, call STS, etc.
{{% /notice %}}

### 4.1 S3 Gateway Endpoint (recommended + free hourly)

<<<<<<< HEAD
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

![S3 Endpoint Configuration](../images/05-vpc-networking/10.2-s3-endpoint-config.png)
![S3 Endpoint Configuration](../images/05-vpc-networking/10.2-s3-endpoint-config2.png)

**Purpose:** EKS Pods load model artifacts from S3 < 50ms latency

### 2.2. ECR API Interface Endpoint (Container Images)

1. **Create ECR API Endpoint:**
   ```
   Endpoint name: mlops-hybrid-ecr-api-endpoint
   Service: com.amazonaws.ap-southeast-1.ecr.api
   Type: Interface
   VPC: mlops-retail-forecast-hybrid-vpc
   Subnets: Both private workload subnets
   Security Groups: mlops-hybrid-vpc-endpoints-sg
   Private DNS: ✅ Enabled
   ```

![ECR API Endpoint](../images/05-vpc-networking/10.3-ecr-api-endpoint.png)
![ECR API Configuration](../images/05-vpc-networking/10.4-ecr-api-config.png)

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

![ECR DKR Endpoint](../images/05-vpc-networking/10.5-ecr-dkr-endpoint.png)
![ECR DKR Endpoint](../images/05-vpc-networking/10.5-ecr-dkr-endpoint2.png)

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

![ALB Basic Configuration](../images/05-vpc-networking/11.1-alb-basic-config.png)
![ALB Network Configuration](../images/05-vpc-networking/11.2-alb-network-config.png)

### 3.2. Create Target Group for EKS API

1. **Target Group Configuration:**
   ```
   Target type: IP addresses
   Target group name: mlops-hybrid-eks-api-tg
   Protocol: HTTP
   Port: 80
   VPC: mlops-retail-forecast-hybrid-vpc
   Health check path: /health
   Health check port: 80
   ```

![Target Group Configuration](../images/05-vpc-networking/11.3-target-group-config.png)

2. **Health Check Settings:**
   ```
   Health check protocol: HTTP
   Health check path: /health
   Health check port: 80
   Healthy threshold: 2
   Unhealthy threshold: 2
   Timeout: 5 seconds
   Interval: 30 seconds
   Success codes: 200
   ```

![Health Check Settings](../images/05-vpc-networking/11.4-health-check-config.png)

### 3.3. Configure ALB Listeners

1. **HTTP Listener (Redirect to HTTPS):**

   ```
   Protocol: HTTP
   Port: 80
   Default action: Redirect to HTTPS
   ```

2. **HTTPS Listener (API Traffic):**
   ```
   Protocol: HTTPS
   Port: 443
   Default action: Forward to target group
   SSL certificate: ACM certificate (hoặc self-signed for demo)
   ```

![ALB Listeners Configuration](../images/05-vpc-networking/11.5-alb-listeners.png)
![ALB Listeners Configuration](../images/05-vpc-networking/11.5-alb-listeners2.png)
![ALB Listeners Configuration](../images/05-vpc-networking/11.5-alb-listeners3.png)

{{% notice info %}}
**💡 SSL Certificate Options:**

- **Production**: Use AWS Certificate Manager (ACM) with domain
- **Demo**: Create self-signed certificate or use HTTP only
- **Development**: Skip SSL, use HTTP listener only
  {{% /notice %}}

### 3.4. ALB Creation Complete

![ALB Creation Complete](../images/05-vpc-networking/11.6-alb-complete.png)

**ALB DNS Name:** Will be used for public API demo access
=======
VPC → **Endpoints** → **Create endpoint**
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```
Name: mlops-hybrid-s3-gateway-endpoint
Service: com.amazonaws.ap-southeast-1.s3
Type: Gateway
VPC: mlops-retail-forecast-hybrid-vpc
Route tables: mlops-hybrid-private-workloads-rt
Policy: Full access (demo) / least-privilege (prod)
```

---

### 4.2 Interface Endpoints (ECR + CloudWatch Logs)

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

```
Name: mlops-hybrid-ecr-dkr-endpoint
Service: com.amazonaws.ap-southeast-1.ecr.dkr
Type: Interface
Subnets: both private subnets
Security group: mlops-hybrid-vpc-endpoints-sg
Private DNS: ✅ Enabled
```

#### CloudWatch Logs

```
Name: mlops-hybrid-logs-endpoint
Service: com.amazonaws.ap-southeast-1.logs
Type: Interface
Subnets: both private subnets
Security group: mlops-hybrid-vpc-endpoints-sg
Private DNS: ✅ Enabled
```

---

### 4.3 (Strongly Recommended) Add STS Endpoint for IRSA / AWS SDK calls

If your pods use IAM roles (IRSA) and call AWS APIs without NAT, add:

```
com.amazonaws.ap-southeast-1.sts
```

This avoids common failures when pods need to call STS but have no Internet path.

<<<<<<< HEAD
{{% notice info %}}
**💡 When to use Terraform outputs:**

- ✅ Task 7-10 will use Terraform (EKS cluster, ALB controller)
- ✅ Need automated deployment across environments
- ✅ Want to reference infrastructure programmatically

**If Task 7-10 use Console:** Skip this section completely!
=======
---

### 4.4 Verification checklist (Endpoints)

Expected:

- S3 Gateway endpoint adds routes automatically to `mlops-hybrid-private-workloads-rt`
- Interface endpoints create ENIs in **each private subnet**
- Private DNS is enabled for interface endpoints

---

## 5) ALB for Demo (Two Valid Approaches)

### Option A (Recommended): Don’t create ALB manually — let EKS create it later

In real EKS workflows, you typically deploy:

- **AWS Load Balancer Controller** (Task 9)
- An **Ingress** that provisions the ALB automatically

**In Task 5**, you only prepare:

- public subnets + tags
- security groups
- IGW + routes

This is cleaner and avoids “ALB exists but no targets yet”.

### Option B (Manual ALB Now): Pre-create ALB + Target Group

(Works, but you still must integrate targets later)

#### 5.1 Create ALB

EC2 → Load Balancers → **Create**

```
Name: mlops-hybrid-api-demo-alb
Scheme: Internet-facing
IP type: IPv4
VPC: mlops-retail-forecast-hybrid-vpc
Mappings: both public ALB subnets
Security Group: mlops-hybrid-alb-sg
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

{{% notice tip %}}
For a classroom demo, you can run **HTTP only** first, then add ACM/HTTPS later.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
{{% /notice %}}

---

## 6) CLI Verification (CloudShell-friendly)

<<<<<<< HEAD
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
=======
### 6.1 Verify VPC and Subnets
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```bash
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
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'VpcEndpoints[*].{Id:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,PrivateDns:PrivateDnsEnabled,State:State}' \
  --output table
```

<<<<<<< HEAD
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

=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
---

## 7) Common Issues (No NAT design)

### 7.1 Pods cannot pull images from ECR

<<<<<<< HEAD
- **Task 3**: IAM Roles & IRSA using VPC infrastructure
- **Task 4**: EKS cluster deployment with VPC Endpoints integration
- **Task 5**: EKS managed node groups in cost-optimized private subnets
  {{% /notice %}}
=======
Symptoms:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

- `ImagePullBackOff`, `ErrImagePull`

<<<<<<< HEAD
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
=======
Fix checklist:

- Have **ECR API + ECR DKR** interface endpoints
- Have **S3 Gateway** endpoint (ECR layers rely on S3 paths)
- Ensure endpoint SG allows **443 from nodes SG**

### 7.2 Pods cannot call AWS APIs (IRSA)

Symptoms:

- STS / credential errors in app logs

Fix:

- Add **STS interface endpoint** (`com.amazonaws.<region>.sts`)
- Make sure Private DNS is enabled

### 7.3 Private nodes cannot join EKS control plane

Fix (in Task 7):

- Enable **EKS private endpoint access**
- Avoid relying on public endpoint when you have no NAT/Internet

---

## 8) Cost Notes (Be precise about “per AZ”)

### 8.1 Interface Endpoints (PrivateLink)

Interface endpoints are billed **per AZ-hour** and **per GB processed** (rates vary by region).

Example math (illustrative):

- $0.01 / endpoint-hour / AZ
- 1 endpoint across **2 AZs** ⇒ billed as **2 endpoint-hours**
- Monthly (30 days): `0.01 * 24 * 30 * 2 = $14.40` per endpoint

So **3 interface endpoints** (ECR API + ECR DKR + Logs) across **2 AZs**:

- `3 * $14.40 = $43.20 / month` (plus data processing)

> Your earlier “$21.6/month” is what you get if you accidentally count only **1 AZ**. In your design you’re using **2 AZs**, so budget for roughly **2×**.

### 8.2 ALB

ALB cost is typically:

- hourly load balancer charge + LCU usage charge (region-dependent)

### 8.3 NAT Gateway (what you’re avoiding)

NAT Gateway pricing includes hourly + per-GB processing (region-dependent)

---

## 9) Cleanup (AWS CLI)

### 9.1 Delete ALB + Target Group (if you created Option B)

```bash
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>
aws elbv2 delete-target-group  --target-group-arn <tg-arn>
```

### 9.2 Delete VPC Endpoints

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-ids...>
```

### 9.3 Delete Security Groups (reverse dependency order)

```bash
aws ec2 delete-security-group --group-id <endpoints-sg>
aws ec2 delete-security-group --group-id <eks-control-plane-sg>
aws ec2 delete-security-group --group-id <eks-nodes-sg>
aws ec2 delete-security-group --group-id <alb-sg>
```

### 9.4 Delete Route Tables, Subnets, IGW, VPC

```bash
aws ec2 delete-route-table --route-table-id <public-rt>
aws ec2 delete-route-table --route-table-id <private-rt>

aws ec2 delete-subnet --subnet-id <subnet-id-1>
aws ec2 delete-subnet --subnet-id <subnet-id-2>
aws ec2 delete-subnet --subnet-id <subnet-id-3>
aws ec2 delete-subnet --subnet-id <subnet-id-4>
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>

<<<<<<< HEAD
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
=======
aws ec2 delete-vpc --vpc-id <vpc-id>
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
```

---

<<<<<<< HEAD
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
=======
## ✅ Task 5 Output (what must exist before Task 6/7)

- VPC `10.0.0.0/16`
- 2 public subnets (ALB) + 2 private subnets (EKS)
- IGW + correct public route table
- Private route table (no NAT route)
- Security groups (ALB / Nodes / Control plane / Endpoints)
- VPC Endpoints: S3 gateway + ECR API + ECR DKR + CloudWatch Logs (and ideally STS)
- Subnet tags prepared for EKS + ALB Controller

---

**Next Step:** `Task 6: ECR Container Registry`
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
