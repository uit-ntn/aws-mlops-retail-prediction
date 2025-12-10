---
title: "Production Networking"
weight: 5
chapter: false
pre: "<b>5. </b>"
---

## 🎯 Task 5 Goal

Build a **Production VPC** for your **EKS deployment** and a **public demo API** (separate from the SageMaker training VPC):

1. **Production EKS Infrastructure** — EKS nodes/pods in **private subnets**
2. **Public Demo Access** — **ALB in public subnets** for `/predict`
3. **Fast Private Access to AWS Services** — VPC Endpoints for **S3/ECR/CloudWatch Logs**
4. **Cost Optimization** — **No NAT Gateway**, ALB **enabled only during demos**

{{% notice warning %}}
**VPC Separation Strategy**

- **Task 4**: SageMaker training can stay on default / quick setup VPC (simple, fast)
- **Task 5**: EKS production runs in a **separate VPC** (security + better control)
- They do **not conflict**. If needed later, connect via VPC Peering / Transit Gateway.
  {{% /notice %}}

---

## ✅ Target Architecture (high-level)

- **VPC**: `10.0.0.0/16`
- **Public subnets (2 AZs)**: ALB only
- **Private subnets (2 AZs)**: EKS worker nodes + pods
- **No NAT**: private workloads have **no direct Internet**
- **VPC Endpoints**:
  - **S3 Gateway Endpoint** (free hourly) for model/data access
  - **ECR API + ECR DKR** Interface Endpoints (pull images privately)
  - **CloudWatch Logs** Interface Endpoint (push logs privately)

---

## 1) Production VPC Setup

### 1.1 Create the VPC

AWS Console → **VPC** → **Create VPC**

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

---

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

VPC → **Endpoints** → **Create endpoint**

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
{{% /notice %}}

---

## 6) CLI Verification (CloudShell-friendly)

### 6.1 Verify VPC and Subnets

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

---

## 7) Common Issues (No NAT design)

### 7.1 Pods cannot pull images from ECR

Symptoms:

- `ImagePullBackOff`, `ErrImagePull`

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

aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>

aws ec2 delete-vpc --vpc-id <vpc-id>
```

---

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
