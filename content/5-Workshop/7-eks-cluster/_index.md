---
title: "EKS Cluster Setup (Production)"
date: 2024-01-01T00:00:00Z
weight: 8
chapter: false
pre: "<b>7. </b>"
---

<<<<<<< HEAD
## 🎯 Task 7 Objectives

Deploy Amazon Elastic Kubernetes Service (EKS) as the foundation to run prediction API (FastAPI) in production environment:

1. **EKS Control Plane**: AWS-managed cluster with Kubernetes 1.30
2. **IRSA Integration**: IAM Roles for Service Accounts for secure AWS access
3. **Cost Optimization**: Free Tier t2.micro instances for development
4. **VPC Integration**: Use hybrid VPC and VPC Endpoints from Task 5
5. **ECR Integration**: Container image pulls from ECR repository

→ Ensure stable, scalable system with secure IAM integration (IRSA).

📥 **Input from Previous Tasks:**
- **Task 5 (Production VPC):** Hybrid VPC, subnets, VPC Endpoints and security groups used for EKS networking
- **Task 2 (IAM Roles & Audit):** IAM roles and policies (cluster role, node role, IRSA foundations)
- **Task 6 (ECR Registry):** ECR repository for container images that EKS will pull

## EKS Architecture in MLOps Pipeline

{{% notice tip %}}
**Tip:** Use private-only endpoint access for production clusters and limit public access CIDRs to specific IP ranges. Enable all control plane logs for security audit and compliance.
{{% /notice %}}

### Cost Optimization Strategy

| Component               | Cost           | Free Tier     | Strategy                    |
| ----------------------- | -------------- | ------------- | --------------------------- |
| **EKS Control Plane**   | $73/month      | ❌            | Use for demo, destroy after |
| **t2.micro Nodes (2x)** | **$0**         | ✅ 750h/month | 12 months FREE              |
| **EBS Storage (20GB)**  | **$0**         | ✅ 30GB FREE  | Within free tier            |
| **VPC Endpoints**       | $21.6/month    | ❌            | Shared with other services  |
| **Total**               | **~$95/month** |               | **$60/month savings**       |

**Free Tier Benefits:**

- **750 hours/month** t2.micro instances → Enough for 24/7 demo
- **30GB EBS storage** → Adequate for basic workloads
- **Total savings**: $60/month vs t3.medium instances

{{% notice warning %}}
**Warning:** EKS control plane costs $73/month regardless of usage level. For learning/testing, consider using self-managed k3s on EC2 or kind locally to avoid fixed costs. Delete cluster immediately after demo.
{{% /notice %}}

## 1. EKS Cluster Setup via Console

### 1.1. Create EKS Cluster

1. **Navigate to EKS Console:**

   - AWS Console → EKS → "Create cluster"

2. **Basic Configuration:**

   ```
   Cluster name: mlops-retail-cluster
   Kubernetes version: 1.30
   Cluster service role: Use existing from Task 2 (IAM)
   ```

3. **Networking Configuration:**
   ```
   VPC: mlops-retail-forecast-hybrid-vpc (from Task 5)
   Subnets: Select all 4 subnets (2 public + 2 private)
   Cluster endpoint access: Public and private
   Public access sources: Specify your IP range
   Security groups: mlops-hybrid-eks-control-plane-sg
   ```

![EKS Console Configuration](../images/07-eks-cluster/02-eks-console-config.png)

4. **Control Plane Logging:**
   ```
   ✅ API server
   ✅ Audit
   ✅ Authenticator
   ✅ Controller manager
   ✅ Scheduler
   ```

![Control Plane Logging Settings](../images/07-eks-cluster/03-control-plane-logging.png)

5. **Add-ons Configuration:**
   ```
   ✅ Amazon VPC CNI: v1.18.1-eksbuild.1
   ✅ CoreDNS: v1.11.1-eksbuild.4
   ✅ kube-proxy: v1.30.0-eksbuild.3
   ✅ Amazon EBS CSI Driver: v1.30.0-eksbuild.1
   ```

![Add-ons Configuration](../images/07-eks-cluster/04-addons-configuration.png)

### 1.2. Update kubeconfig

```bash
# Configure kubectl for new cluster
aws eks update-kubeconfig --region ap-southeast-1 --name mlops-retail-cluster

# Verify connection
kubectl get svc
kubectl cluster-info
```

**Expected output:**

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   5m

Kubernetes control plane is running at https://ABC123.gr7.ap-southeast-1.eks.amazonaws.com
CoreDNS is running at https://ABC123.gr7.ap-southeast-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

![kubeconfig Verification](../images/07-eks-cluster/05-kubeconfig-verification.png)

### 1.3. Verify Add-ons

```bash
# Check core add-ons
kubectl get pods -n kube-system

# Verify VPC CNI
kubectl get daemonset -n kube-system aws-node

# Check CoreDNS
kubectl get deployment -n kube-system coredns

# Verify kube-proxy
kubectl get daemonset -n kube-system kube-proxy

# Check EBS CSI driver
kubectl get deployment -n kube-system ebs-csi-controller
```

![Add-ons Status Verification](../images/07-eks-cluster/06-addons-verification.png)

{{% notice success %}}
**🎯 EKS Control Plane Ready!**

**Control Plane Status:**

- ✅ Kubernetes 1.30 cluster ACTIVE
- ✅ Multi-AZ managed control plane
- ✅ All essential add-ons running
- ✅ CloudWatch logging enabled
- ✅ kubectl access configured
  {{% /notice %}}

{{% notice info %}}
**Info:** EKS control plane automatically runs across multiple AZs. To optimize costs, ensure worker nodes are balanced across AZs to avoid cross-AZ data transfer charges ($0.01/GB).
=======
{{% notice info %}}
**🎯 Task 7 Goal:** Provision an Amazon EKS cluster in **ap-southeast-1** connected to the Production VPC, enable essential add-ons/logging, configure IRSA (OIDC) for AWS access from pods, and deploy a sample app that pulls from ECR.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
{{% /notice %}}

## 0) Inputs from previous tasks

- Production VPC: `10.0.0.0/16` (private subnets for nodes/pods; public subnets for ALB demos)
- No NAT Gateway (cost optimization). Use **VPC Endpoints**:
  - S3 **Gateway** endpoint
  - ECR API + ECR DKR **Interface** endpoints
  - CloudWatch Logs **Interface** endpoint
- Cluster name: `mlops-retail-cluster`
- Region: `ap-southeast-1`
- ECR: `842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest`

---

## 1) Create the EKS cluster

### Option A (Recommended): eksctl (fast)

> If you already created the VPC in Task 5, plug in your subnet IDs.

Create `eksctl-cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: mlops-retail-cluster
  region: ap-southeast-1
  version: "1.29"

vpc:
  id: "<PRODUCTION_VPC_ID>"
  subnets:
    private:
      ap-southeast-1a: { id: "<PRIVATE_SUBNET_ID_A>" }
      ap-southeast-1b: { id: "<PRIVATE_SUBNET_ID_B>" }
    public:
      ap-southeast-1a: { id: "<PUBLIC_SUBNET_ID_A>" }
      ap-southeast-1b: { id: "<PUBLIC_SUBNET_ID_B>" }

cloudWatch:
  clusterLogging:
    enableTypes:
      ["api", "audit", "authenticator", "controllerManager", "scheduler"]

managedNodeGroups:
  - name: retail-ng-dev
    instanceType: t2.micro
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    privateNetworking: true
    labels:
      role: dev
    iam:
      withAddonPolicies:
        cloudWatch: true
        ecr: true
        autoScaler: true
```

Create cluster:

```bash
eksctl create cluster -f eksctl-cluster.yaml
```

### Option B: AWS Console (if you must)

1. EKS → Clusters → Create
2. Name: `mlops-retail-cluster`, Region: `ap-southeast-1`
3. Networking: select Production VPC + **private** subnets for nodes
4. Enable control plane logs
5. After cluster is ready → create Managed Node Group (`t2.micro` for dev)

---

## 2) Add-ons & observability baseline

Recommended EKS add-ons:

- `vpc-cni`
- `coredns`
- `kube-proxy`
- `aws-ebs-csi-driver` (optional)

```bash
aws eks list-addons --cluster-name mlops-retail-cluster --region ap-southeast-1
aws eks create-addon --cluster-name mlops-retail-cluster --addon-name vpc-cni --region ap-southeast-1
aws eks create-addon --cluster-name mlops-retail-cluster --addon-name coredns --region ap-southeast-1
aws eks create-addon --cluster-name mlops-retail-cluster --addon-name kube-proxy --region ap-southeast-1
```

---

## 3) Configure kubectl access

```bash
aws eks update-kubeconfig --name mlops-retail-cluster --region ap-southeast-1
kubectl get nodes
```

---

## 4) IRSA (IAM Roles for Service Accounts)

### 4.1 Associate OIDC provider

```bash
eksctl utils associate-iam-oidc-provider   --cluster mlops-retail-cluster   --region ap-southeast-1   --approve
```

### 4.2 Create an IAM role for pods (example: S3 read + CloudWatch logs)

Create policy `irsa-s3-cw.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadArtifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::<YOUR_BUCKET>", "arn:aws:s3:::<YOUR_BUCKET>/*"]
    },
    {
      "Sid": "CWLogsWrite",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

Create policy + role (example via CLI):

```bash
aws iam create-policy --policy-name EKSIRSA-S3-CW --policy-document file://irsa-s3-cw.json
# Create role trust policy is easier using eksctl serviceaccount (next section)
```

Create K8s namespace and service account with eksctl:

```bash
kubectl create namespace mlops-retail-forecast || true

eksctl create iamserviceaccount   --cluster mlops-retail-cluster   --region ap-southeast-1   --namespace mlops-retail-forecast   --name retail-sa   --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/EKSIRSA-S3-CW   --approve   --override-existing-serviceaccounts
```

---

## 5) Verify ECR access (image pull)

If your node group has the standard EKS worker role, it can pull from ECR.
If you run strictly private subnets with no NAT, confirm your VPC endpoints exist:

- `com.amazonaws.ap-southeast-1.ecr.api`
- `com.amazonaws.ap-southeast-1.ecr.dkr`
- `com.amazonaws.ap-southeast-1.logs`
- S3 Gateway endpoint

---

## 6) Deploy a sample app (smoke test)

Create `k8s/sample-app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mlops-retail-forecast
<<<<<<< HEAD
  labels:
    name: mlops-retail-forecast
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-irsa-s3-access-role
  labels:
    app.kubernetes.io/name: s3-access-service-account
    app.kubernetes.io/component: rbac
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudwatch-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-irsa-cloudwatch-role
  labels:
    app.kubernetes.io/name: cloudwatch-service-account
    app.kubernetes.io/component: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: retail-api-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-irsa-s3-access-role
  labels:
    app.kubernetes.io/name: retail-api-service-account
    app.kubernetes.io/component: api
```

**Apply service accounts:**

```bash
# Replace ACCOUNT_ID with your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed "s/ACCOUNT_ID/$ACCOUNT_ID/g" k8s/service-accounts.yaml | kubectl apply -f -

# Verify service accounts
kubectl get serviceaccounts -n mlops-retail-forecast
kubectl describe serviceaccount s3-access-sa -n mlops-retail-forecast
```

{{% notice success %}}
**🎯 IRSA Setup Complete!**

**IRSA Components:**

- ✅ OIDC identity provider associated with EKS
- ✅ S3 access role for ML workloads
- ✅ CloudWatch role for monitoring
- ✅ Kubernetes service accounts with proper annotations
- ✅ Secure AWS access without hardcoded credentials
  {{% /notice %}}

{{% notice tip %}}
**Tip:** Use separate service accounts for each workload with minimal IAM policies. Avoid sharing service accounts between applications and regularly audit IRSA role permissions.
{{% /notice %}}

## 3. Managed Node Group Setup

### 3.1. Create Node Group via Console

1. **Navigate to Node Groups:**

   - EKS Console → mlops-retail-cluster → Compute → Node groups → "Add node group"

2. **Node Group Configuration:**

   ```
   Name: mlops-retail-nodegroup-t2micro
   Node IAM role: mlops-hybrid-eks-nodes-role (from Task 5)
   ```

3. **Instance Configuration:**

   ```
   AMI type: Amazon Linux 2 (AL2_x86_64)
   Capacity type: On-Demand
   Instance types: t2.micro
   Disk size: 20 GB
   ```

4. **Scaling Configuration:**

   ```
   Desired size: 2
   Minimum size: 1
   Maximum size: 4
   ```

5. **Network Configuration:**
   ```
   Subnets: Both private workload subnets
   Configure remote access: Enable (optional)
   SSH key pair: Your EC2 key pair (optional)
   Source security groups: mlops-hybrid-eks-nodes-sg
   ```

![Node Group Console Configuration](../images/07-eks-cluster/08-nodegroup-console-config.png)

### 3.2. Alternative: eksctl Node Group (Command Line)

**Create file `scripts/create-nodegroup.sh`:**

```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="mlops-retail-cluster"
REGION="ap-southeast-1"
NODEGROUP_NAME="mlops-retail-nodegroup-t2micro"

echo "🔧 Creating EKS node group with t2.micro instances..."

# Create node group using eksctl
eksctl create nodegroup \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --name=$NODEGROUP_NAME \
    --node-type=t2.micro \
    --nodes=2 \
    --nodes-min=1 \
    --nodes-max=4 \
    --node-volume-size=20 \
    --node-volume-type=gp3 \
    --ssh-access \
    --ssh-public-key=YOUR_KEY_NAME \
    --managed \
    --node-private-networking \
    --node-zones=ap-southeast-1a,ap-southeast-1b

echo "✅ Node group creation initiated!"
echo "Check status: eksctl get nodegroup --cluster $CLUSTER_NAME --region $REGION"
```

### 3.3. Verify Node Group

```bash
# Check node group status
aws eks describe-nodegroup \
    --cluster-name mlops-retail-cluster \
    --nodegroup-name mlops-retail-nodegroup-t2micro \
    --region ap-southeast-1

# Verify nodes in Kubernetes
kubectl get nodes
kubectl get nodes -o wide

# Check node labels and annotations
kubectl describe nodes
```

**Expected output:**

```
NAME                                              STATUS   ROLES    AGE   VERSION
ip-10-0-101-123.ap-southeast-1.compute.internal   Ready    <none>   5m    v1.30.0-eks-xyz
ip-10-0-102-456.ap-southeast-1.compute.internal   Ready    <none>   5m    v1.30.0-eks-xyz
```

![Nodes Ready Status](../images/07-eks-cluster/11-nodes-ready-status.png)

{{% notice success %}}
**🎯 Node Group Ready!**

**Node Group Status:**

- ✅ 2x t2.micro instances (Free Tier)
- ✅ Private subnet deployment
- ✅ Auto Scaling Group configured (1-4 instances)
- ✅ All nodes in Ready state
- ✅ EKS optimized AMI with latest patches
  {{% /notice %}}

{{% notice warning %}}
**Warning (t2.micro):** Limited to 1 vCPU and 1GB RAM. Monitor CPU credits and consider burstable performance limits. For production ML workloads, use t3.medium+ instances.
{{% /notice %}}

## 4. Cost Optimization with VPC Endpoints

### 4.1. Review VPC Endpoints for EKS

**VPC Endpoints Created in Task 5:**

```bash
# Verify VPC endpoints exist for EKS cost optimization
aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=vpc-xxxxx" \
    --query 'VpcEndpoints[?ServiceName==`com.amazonaws.ap-southeast-1.s3`]'

aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=vpc-xxxxx" \
    --query 'VpcEndpoints[?ServiceName==`com.amazonaws.ap-southeast-1.ecr.api`]'
```

**Expected VPC Endpoints:**

- ✅ S3 Gateway Endpoint (FREE - no data charges)
- ✅ ECR API Endpoint ($0.01/hour per AZ)
- ✅ ECR Docker Endpoint ($0.01/hour per AZ)
- ✅ CloudWatch Logs Endpoint ($0.01/hour per AZ)

### 4.2. Cost Savings with VPC Endpoints

| Data Transfer  | Without VPC Endpoints  | With VPC Endpoints     | Savings  |
| -------------- | ---------------------- | ---------------------- | -------- |
| **S3 Access**  | $0.09/GB (NAT Gateway) | **$0.00/GB** (Gateway) | **100%** |
| **ECR Pull**   | $0.09/GB (Internet)    | **$0.00/GB** (Private) | **100%** |
| **CloudWatch** | $0.09/GB (NAT Gateway) | **$0.00/GB** (Private) | **100%** |

{{% notice success %}}
**💰 Monthly Cost Breakdown with t2.micro + VPC Endpoints:**

**EKS Control Plane:** $73.00/month (fixed)  
**t2.micro Nodes:** $0.00/month (FREE tier - 750 hours)  
**VPC Endpoints:** ~$7.20/month (4 endpoints × 2 AZ × $0.01/hour)  
**Data Transfer:** $0.00/month (all private via VPC endpoints)

**Total Monthly Cost:** ~$80.20/month  
**Savings vs NAT Gateway:** ~$45/month (NAT Gateway cost)
{{% /notice %}}

{{% notice info %}}
**Info:** VPC Endpoints eliminate internet routing for AWS services but cost $0.01/hour per endpoint per AZ. Monitor usage patterns and consolidate endpoints when possible.
{{% /notice %}}

## 5. Deploy Sample Application with IRSA

### 5.1. Deploy FastAPI Prediction Service

**Create file `k8s/retail-api-deployment.yaml`:**

```yaml
=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-nginx
  namespace: mlops-retail-forecast
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-nginx
  template:
    metadata:
      labels:
        app: sample-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: sample-nginx-svc
  namespace: mlops-retail-forecast
spec:
  type: ClusterIP
  selector:
    app: sample-nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply + verify:

```bash
kubectl apply -f k8s/sample-app.yaml
kubectl get pods -n mlops-retail-forecast
kubectl get svc  -n mlops-retail-forecast
```

<<<<<<< HEAD
### 5.3. Test IRSA S3 Access

**Create test pod:**

```bash
# Create test pod to verify IRSA S3 access
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-irsa-s3
  namespace: mlops-retail-forecast
spec:
  serviceAccountName: s3-access-sa
  containers:
  - name: test-s3
    image: amazon/aws-cli:latest
    command: ["/bin/sh"]
    args: ["-c", "aws s3 ls s3://mlops-retail-forecast-models && sleep 3600"]
    env:
    - name: AWS_DEFAULT_REGION
      value: "ap-southeast-1"
  restartPolicy: Never
EOF

# Check if S3 access works via IRSA
kubectl logs test-irsa-s3 -n mlops-retail-forecast
```

**Expected output:**

```
2024-01-15 10:30:45    model-v1.0.pkl
2024-01-15 10:31:20    model-v1.1.pkl
                           PRE training-data/
```

## 6. Cluster Verification

### 6.1. Control Plane Health Check

```bash
# Check cluster status
kubectl get --raw='/readyz?verbose'

# Check API server health
kubectl get --raw='/healthz'

# View cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 6.2. Add-ons Verification

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check kube-proxy
kubectl get daemonset -n kube-system kube-proxy

# Check VPC CNI
kubectl get daemonset -n kube-system aws-node

# Check EBS CSI driver
kubectl get deployment -n kube-system ebs-csi-controller
```

### 6.3. RBAC and Permissions

```bash
# Check current user permissions
kubectl auth can-i "*" "*"

# List available API resources
kubectl api-resources

# Check cluster roles
kubectl get clusterroles | grep eks
```

### 6.4. IRSA Verification

```bash
# Deploy service accounts with IRSA annotations
kubectl apply -f aws/k8s/service-accounts.yaml

# Deploy test pod with IRSA authentication
kubectl apply -f aws/k8s/test-pod-irsa.yaml

# Verify pod can access S3 via IRSA (no AWS credentials needed!)
kubectl exec -it test-irsa-s3-access -- aws s3 ls

# Check IRSA role annotations
kubectl get serviceaccount s3-access-sa -o yaml

# Verify OIDC provider
aws iam list-open-id-connect-providers

# List IRSA roles
aws iam list-roles --query 'Roles[?contains(RoleName, `irsa`)].{RoleName:RoleName,CreateDate:CreateDate}'

# Test CloudWatch access
kubectl exec -it test-irsa-s3-access -- aws cloudwatch list-metrics --namespace "MLOps/RetailForecast"

# Cleanup test pod
kubectl delete pod test-irsa-s3-access
```

## 7. Monitoring and Logging

### 7.1. CloudWatch Integration

```bash
# Check if cluster logging is enabled
aws eks describe-cluster \
  --name mlops-retail-forecast-dev-cluster \
  --query 'cluster.logging'

# View control plane logs in CloudWatch
aws logs describe-log-groups \
  --log-group-name-prefix /aws/eks/mlops-retail-forecast-dev-cluster
```


### 7.2. Setup CloudWatch Container Insights

**File: `aws/k8s/cloudwatch-insights.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: amazon-cloudwatch
  labels:
    name: amazon-cloudwatch
=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
---

## 7) Cleanup (recommended for cost control)

```bash
<<<<<<< HEAD
# Verify security groups
aws ec2 describe-security-groups \
  --group-ids $(terraform output -raw eks_control_plane_security_group_id) \
  --query 'SecurityGroups[0].{GroupId:GroupId,IpPermissions:IpPermissions}'

# Check VPC configuration
aws eks describe-cluster \
  --name mlops-retail-forecast-dev-cluster \
  --query 'cluster.resourcesVpcConfig'
```

### 8.2. RBAC Configuration

**File: `aws/k8s/rbac.yaml`**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mlops-admin
  namespace: mlops-retail-forecast
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mlops-retail-forecast
  name: mlops-admin-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mlops-admin-binding
  namespace: mlops-retail-forecast
subjects:
  - kind: ServiceAccount
    name: mlops-admin
    namespace: mlops-retail-forecast
roleRef:
  kind: Role
  name: mlops-admin-role
  apiGroup: rbac.authorization.k8s.io
```

## 9. Clean Up Resources (AWS CLI)

### 9.1. Delete Node Groups

```bash
# List all node groups
aws eks list-nodegroups --cluster-name mlops-retail-cluster --region ap-southeast-1

# Delete node group (this will take 5-10 minutes)
aws eks delete-nodegroup \
    --cluster-name mlops-retail-cluster \
    --nodegroup-name mlops-retail-nodegroup-t2micro \
    --region ap-southeast-1

# Monitor deletion status
aws eks describe-nodegroup \
    --cluster-name mlops-retail-cluster \
    --nodegroup-name mlops-retail-nodegroup-t2micro \
    --region ap-southeast-1 \
    --query 'nodegroup.status'
```

### 9.2. Delete IRSA Roles and OIDC Provider

```bash
# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Detach and delete S3 access role
aws iam detach-role-policy \
    --role-name mlops-irsa-s3-access-role \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/mlops-irsa-s3-policy

aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/mlops-irsa-s3-policy
aws iam delete-role --role-name mlops-irsa-s3-access-role

# Delete CloudWatch access role
aws iam detach-role-policy \
    --role-name mlops-irsa-cloudwatch-role \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam delete-role --role-name mlops-irsa-cloudwatch-role

# Delete OIDC provider
OIDC_URL=$(aws eks describe-cluster --name mlops-retail-cluster --region ap-southeast-1 --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_URL}
```

### 9.3. Delete EKS Cluster

```bash
# Delete the cluster (ensure node groups are deleted first)
aws eks delete-cluster --name mlops-retail-cluster --region ap-southeast-1

# Monitor deletion progress
aws eks describe-cluster --name mlops-retail-cluster --region ap-southeast-1 --query 'cluster.status'

# Verify cluster is deleted (should return error)
aws eks list-clusters --region ap-southeast-1 --query 'clusters[?contains(@, `mlops-retail-cluster`)]'
```

### 9.4. Clean Up Kubernetes Resources

```bash
# Delete namespace (this removes all resources in the namespace)
kubectl delete namespace mlops-retail-forecast

# Remove cluster from kubeconfig
kubectl config delete-cluster arn:aws:eks:ap-southeast-1:${ACCOUNT_ID}:cluster/mlops-retail-cluster
kubectl config delete-context arn:aws:eks:ap-southeast-1:${ACCOUNT_ID}:cluster/mlops-retail-cluster

# Remove user from kubeconfig
kubectl config unset users.arn:aws:eks:ap-southeast-1:${ACCOUNT_ID}:cluster/mlops-retail-cluster
```

### 9.5. Script Dọn dẹp EKS Tự động

```bash
#!/bin/bash
# eks-cleanup.sh

CLUSTER_NAME="mlops-retail-cluster"
REGION="ap-southeast-1"
NODEGROUP_NAME="mlops-retail-nodegroup-t2micro"

echo "🧹 Starting EKS cluster cleanup..."

# 1. Delete applications and namespace
echo "Deleting Kubernetes resources..."
=======
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
kubectl delete namespace mlops-retail-forecast --ignore-not-found=true
eksctl delete cluster --name mlops-retail-cluster --region ap-southeast-1
```

---

<<<<<<< HEAD
## 10. EKS Pricing Table (ap-southeast-1)

### 10.1. EKS Control Plane Cost

| Component | Price (USD/cluster/hour) | Price (USD/cluster/month) | Notes |
|-----------|------------------------|-------------------------|----------|
| **EKS Control Plane** | $0.10 | $73.00 | Fixed cost regardless of usage |
| **Fargate** | $0.04048/vCPU/hour | Variable | Pay per pod resources |
| **EC2 Worker Nodes** | EC2 pricing | Variable | t2.micro has FREE tier |

### 10.2. EC2 Worker Nodes Cost

| Instance Type | vCPU | Memory | Price (USD/hour) | Price (USD/month) | Free Tier |
|---------------|------|--------|----------------|-----------------|------------|
| **t2.micro** | 1 | 1GB | $0.0116 | $8.50 | ✅ 750h/month |
| **t3.micro** | 2 | 1GB | $0.0104 | $7.61 | ❌ |
| **t3.small** | 2 | 2GB | $0.0208 | $15.22 | ❌ |
| **t3.medium** | 2 | 4GB | $0.0416 | $30.45 | ❌ |
| **m5.large** | 2 | 8GB | $0.096 | $70.27 | ❌ |

### 10.3. VPC Endpoints Cost for EKS

| Endpoint Type | Price (USD/hour/endpoint) | Monthly Cost (2 AZ) | Notes |
|---------------|-------------------------|---------------------|----------|
| **S3 Gateway** | Free | $0.00 | No hourly charge |
| **ECR API** | $0.01 | $14.40 | Required for image pulls |
| **ECR Docker** | $0.01 | $14.40 | Required for image pulls |
| **CloudWatch Logs** | $0.01 | $14.40 | Optional for logging |
| **EC2** | $0.01 | $14.40 | Optional for instance metadata |

### 10.4. Estimated Cost for Task 7

**EKS Cluster Configuration:**
- Control Plane: 1 cluster
- Worker Nodes: 2x t2.micro instances
- VPC Endpoints: ECR API + ECR Docker
- Storage: 20GB EBS per node

**Chi phí hàng tháng:**

| Component | Configuration | Monthly Cost | Free Tier Discount |
|-----------|---------------|--------------|--------------------|
| **EKS Control Plane** | 1 cluster | $73.00 | ❌ |
| **t2.micro Instances** | 2x instances (1,500h) | $17.00 | **-$17.00** (FREE) |
| **EBS Storage** | 40GB gp3 | $3.20 | **-$3.20** (30GB FREE) |
| **ECR VPC Endpoints** | 2 endpoints × 2 AZ | $28.80 | ❌ |
| **Data Transfer** | Internal VPC | $0.00 | ✅ FREE |
| **Total** | | **$122.00** | **-$20.20** |
| **Actual Cost** | | | **$101.80** |

### 10.5. Cost Comparison with Other Options

**EKS vs Self-Managed Kubernetes:**

| Feature | EKS | Self-Managed K8s | Savings |
|---------|-----|------------------|----------|
| **Control Plane** | $73/month | $0 (self-managed) | -$73 |
| **System Management** | ✅ Managed | ❌ Manual updates | Time savings |
| **Security Patches** | ✅ Automatic | ❌ Manual | Security |
| **Multi-AZ HA** | ✅ Available | ❌ Complex setup | Reliability |
| **AWS Integration** | ✅ Native | ❌ Manual | Ease of use |

**EKS vs ECS Fargate:**

| Workload Type | EKS Cost | ECS Fargate Cost | Winner |
|----------|----------|------------------|--------|
| **Small APIs (0.25 vCPU)** | $73 + instance | $7.30/month | **Fargate** |
| **Batch Jobs** | $73 + compute | Pay per run | **Fargate** |
| **Always-on Services** | $73 + compute | $29.20/month | **EKS** |
| **Multi-service Apps** | $73 + compute | $N × service cost | **EKS** |

### 10.6. Data Transfer Costs

**EKS Data Transfer Scenarios:**

| Transfer Type | Cost | Use Case |
|---------------|------|----------|
| **Pod to Pod (same AZ)** | Free | Microservices communication |
| **Pod to Pod (different AZ)** | $0.01/GB | Multi-AZ deployment |
| **Pod to Internet** | $0.12/GB | API responses to users |
| **Pod to S3 (VPC Endpoint)** | Free | Model/data access |
| **Pod to S3 (Internet)** | $0.12/GB | No VPC endpoint |

### 10.7. Cost Optimization Strategies

**Right-sizing Instance:**
```bash
# Use Kubernetes resource requests/limits
resources:
  requests:
    cpu: 100m      # 0.1 vCPU
    memory: 128Mi  # 128MB
  limits:
    cpu: 500m      # 0.5 vCPU
    memory: 512Mi  # 512MB
```

**Node Scaling:**
```bash
# Cluster Autoscaler configuration
desired_size = 2
min_size     = 1
max_size     = 10

# Scale down quickly when not needed
scale_down_delay_after_add = "2m"
scale_down_unneeded_time   = "2m"
```

**Spot Instances:**
```bash
# Mix spot and on-demand for cost savings
capacity_type = "SPOT"     # 60-70% savings
# or
capacity_type = "ON_DEMAND" # Stable pricing
```

### 10.8. Free Tier Optimization

**12-Month Free Tier Benefits:**
- **750 hours/month** t2.micro EC2 instances
- **30 GB** EBS General Purpose (SSD) storage
- **2 million** Lambda requests (if used for automation)
- **1 GB** CloudWatch Logs (first 5GB free)

**Always Free:**
- **1 million** AWS Lambda requests per month
- **5 GB** CloudWatch monitoring data
- **S3 transfers** within same region via VPC endpoints

{{% notice info %}}
**💰 Cost Summary for Task 7:**
- **Fixed cost:** $73/month (EKS control plane)
- **Variable cost:** $28.80/month (VPC endpoints)
- **Free Tier savings:** $20.20/month (instances + storage)
- **Total:** **$81.60/month** with free tier optimizations
- **Production scaling:** Add instance types based on workload needs
{{% /notice %}}

{{% notice success %}}
**Success tip:** Monitor cluster costs with AWS Cost Explorer and set up billing alerts. Use eksctl or Terraform for infrastructure as code to ensure consistent and reproducible deployments.
{{% /notice %}}

---

## 👉 Task 7 Results

After Task 7, you will have a production-ready EKS Cluster, running completely in private subnets and integrated with VPC Endpoints from Task 5, saving NAT Gateway costs and increasing security.

### Deliverables Completed

- **EKS Control Plane ACTIVE**: Managed Kubernetes cluster with multi-AZ high availability
- **IRSA Configured**: OIDC provider and Service Account authentication setup
- **Managed Node Groups**: 2x t2.micro instances (FREE tier) distributed across ≥2 AZ
- **VPC Endpoints Integration**: Using ECR, S3 endpoints from Task 2 (70% cost savings)
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **Secure Pod Access**: Pods can access S3/CloudWatch via IRSA (no hardcoded credentials)
- **kubectl Access**: Local development environment configured and tested
- **Cost Optimization**: $117.40/month saved with FREE tier + VPC Endpoints

{{% notice success %}}
**🎯 Ready for Next Tasks:**

EKS cluster foundation is ready for deployment:

- ✅ **Task 5**: EKS node groups scaling and optimization
- ✅ **Task 6**: ECR repository setup for container images
- ✅ **Task 7**: Build and push inference API container
- ✅ **Task 8**: S3 data storage integration
- ✅ **Task 13**: Deploy inference API to EKS cluster
  {{% /notice %}}

{{% notice warning %}}
**🔐 Security & Maintenance Notes:**

- **Public Access**: Limit `cluster_endpoint_public_access_cidrs` to actual IP ranges in production
- **Logging**: Enable all control plane logging for security audit  
- **Updates**: Regularly update Kubernetes version and add-ons (quarterly)
- **RBAC**: Deploy appropriate role-based access control for team members
- **Monitoring**: Set up alerts for node health, pod failures, and resource usage  
- **Backup**: Consider EKS cluster backup strategy for disaster recovery
  {{% /notice %}}

## 🎬 Task 7 Implementation Video

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/Ueafl18K38Y" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 08: Deploy Kubernetes](../8-deploy-kubernetes)
=======
## 8) Rough cost notes

- EKS control plane has a fixed hourly cost.
- NodeGroup (EC2) is your main variable cost. Use tiny instances for dev and shut down when idle.
- Avoid NAT Gateway costs by using VPC Endpoints (as designed in Task 5).

{{% notice success %}}
**✅ Task 7 Complete (EKS):**

- EKS cluster created in ap-southeast-1
- Control plane logs enabled + add-ons configured
- IRSA enabled (OIDC provider + service account role)
- Sample workload deployed successfully
  {{% /notice %}}
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
