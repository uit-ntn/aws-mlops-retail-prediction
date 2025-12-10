---
title: "EKS Cluster Setup"
weight: 7
chapter: false
pre: "<b>7. </b>"
---

## 🎯 Mục tiêu Task 7

Triển khai Amazon Elastic Kubernetes Service (EKS) để làm nền tảng chạy API dự đoán (FastAPI) trong môi trường production:

1. **EKS Control Plane**: Cluster quản lý bởi AWS với Kubernetes 1.30
2. **IRSA Integration**: IAM Roles for Service Accounts cho secure AWS access
3. **Cost Optimization**: Free Tier t2.micro instances cho development
4. **VPC Integration**: Sử dụng hybrid VPC và VPC Endpoints từ Task 5
5. **ECR Integration**: Container image pulls từ ECR repository

→ Đảm bảo hệ thống ổn định, mở rộng linh hoạt (scalable), và tích hợp bảo mật với IAM (IRSA).

📥 **Input từ các Task trước:**
- **Task 5 (Production VPC):** Hybrid VPC, subnets, VPC Endpoints and security groups used for EKS networking
- **Task 2 (IAM Roles & Audit):** IAM roles and policies (cluster role, node role, IRSA foundations)
- **Task 6 (ECR Registry):** ECR repository for container images that EKS will pull

## Kiến trúc EKS trong MLOps Pipeline

{{% notice tip %}}
**Tip:** Sử dụng private-only endpoint access cho production clusters và giới hạn public access CIDRs theo IP ranges cụ thể. Bật tất cả control plane logs để audit bảo mật và tuân thủ quy định.
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
**Warning:** EKS control plane tốn $73/tháng bất kể mức sử dụng. Để học tập/testing, cân nhắc dùng self-managed k3s trên EC2 hoặc kind locally để tránh chi phí cố định. Xóa cluster ngay sau khi demo.
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
**Info:** EKS control plane tự động chạy trên nhiều AZs. Để tối ưu chi phí, đảm bảo worker nodes cân bằng giữa các AZs để tránh phí cross-AZ data transfer ($0.01/GB).
{{% /notice %}}

## 2. IRSA (IAM Roles for Service Accounts) Setup

### 2.1. Associate OIDC Provider

```bash
# Check if OIDC provider exists
aws eks describe-cluster --name mlops-retail-cluster \
    --region ap-southeast-1 \
    --query "cluster.identity.oidc.issuer" --output text

# Associate OIDC identity provider with cluster
eksctl utils associate-iam-oidc-provider \
    --cluster mlops-retail-cluster \
    --region ap-southeast-1 \
    --approve
```

**Expected output:**

```
✅ Created OIDC identity provider for cluster "mlops-retail-cluster" in region "ap-southeast-1"
```

![OIDC Provider Setup](../images/07-eks-cluster/07-oidc-provider-setup.png)

### 2.2. Create IRSA Role for S3 Access

**Create file `scripts/create-irsa-s3-role.sh`:**

```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="mlops-retail-cluster"
REGION="ap-southeast-1"
NAMESPACE="mlops-retail-forecast"
SERVICE_ACCOUNT="s3-access-sa"
ROLE_NAME="mlops-irsa-s3-access-role"
POLICY_NAME="mlops-irsa-s3-policy"

# Get OIDC issuer URL
OIDC_ISSUER=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --region $REGION \
    --query "cluster.identity.oidc.issuer" \
    --output text | sed 's|https://||')

# Get AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "🔧 Creating IRSA role for S3 access..."
echo "Cluster: $CLUSTER_NAME"
echo "OIDC Issuer: $OIDC_ISSUER"
echo "Namespace: $NAMESPACE"
echo "Service Account: $SERVICE_ACCOUNT"

# Create trust policy
cat > trust-policy.json << EOF
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
          "${OIDC_ISSUER}:sub": "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT}",
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
    --assume-role-policy-document file://trust-policy.json \
    --description "IRSA role for EKS pods to access S3"

# Create S3 access policy
cat > s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::mlops-retail-forecast-models",
        "arn:aws:s3:::mlops-retail-forecast-models/*",
        "arn:aws:s3:::mlops-retail-forecast-data",
        "arn:aws:s3:::mlops-retail-forecast-data/*"
      ]
    }
  ]
}
EOF

# Create and attach policy
aws iam create-policy \
    --policy-name $POLICY_NAME \
    --policy-document file://s3-policy.json \
    --description "S3 access policy for ML workloads"

aws iam attach-role-policy \
    --role-name $ROLE_NAME \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/${POLICY_NAME}

# Clean up temp files
rm trust-policy.json s3-policy.json

echo "✅ IRSA S3 role created successfully!"
echo "Role ARN: arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"
```

**Run the script:**

```bash
chmod +x scripts/create-irsa-s3-role.sh
./scripts/create-irsa-s3-role.sh
```

### 2.3. Create IRSA Role for CloudWatch

**Create file `scripts/create-irsa-cloudwatch-role.sh`:**

```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="mlops-retail-cluster"
REGION="ap-southeast-1"
NAMESPACE="mlops-retail-forecast"
SERVICE_ACCOUNT="cloudwatch-sa"
ROLE_NAME="mlops-irsa-cloudwatch-role"

# Get OIDC issuer and account ID
OIDC_ISSUER=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --region $REGION \
    --query "cluster.identity.oidc.issuer" \
    --output text | sed 's|https://||')

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "🔧 Creating IRSA role for CloudWatch access..."

# Create trust policy for CloudWatch service account
cat > cloudwatch-trust-policy.json << EOF
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
          "${OIDC_ISSUER}:sub": "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT}",
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
    --assume-role-policy-document file://cloudwatch-trust-policy.json \
    --description "IRSA role for EKS pods to access CloudWatch"

# Attach CloudWatch agent policy
aws iam attach-role-policy \
    --role-name $ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Clean up
rm cloudwatch-trust-policy.json

echo "✅ IRSA CloudWatch role created successfully!"
echo "Role ARN: arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"
```

**Run the script:**

```bash
chmod +x scripts/create-irsa-cloudwatch-role.sh
./scripts/create-irsa-cloudwatch-role.sh
```

### 2.4. Create Kubernetes Service Accounts

**Create file `k8s/service-accounts.yaml`:**

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: mlops-retail-forecast
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
**Tip:** Sử dụng service accounts riêng biệt cho từng workload với minimal IAM policies. Tránh chia sẻ service accounts giữa các ứng dụng và thường xuyên audit IRSA role permissions.
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
**Warning (t2.micro):** Giới hạn 1 vCPU và 1GB RAM. Theo dõi CPU credits và cân nhắc giới hạn burstable performance. Cho production ML workloads, sử dụng t3.medium+ instances.
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
**💰 Monthly Cost Breakdown với t2.micro + VPC Endpoints:**

**EKS Control Plane:** $73.00/month (fixed)  
**t2.micro Nodes:** $0.00/month (FREE tier - 750 hours)  
**VPC Endpoints:** ~$7.20/month (4 endpoints × 2 AZ × $0.01/hour)  
**Data Transfer:** $0.00/month (all private via VPC endpoints)

**Total Monthly Cost:** ~$80.20/month  
**Savings vs NAT Gateway:** ~$45/month (NAT Gateway cost)
{{% /notice %}}

{{% notice info %}}
**Info:** VPC Endpoints loại bỏ internet routing cho AWS services nhưng tốn $0.01/giờ per endpoint per AZ. Theo dõi usage patterns và consolidate endpoints khi có thể.
{{% /notice %}}

## 5. Deploy Sample Application with IRSA

### 5.1. Deploy FastAPI Prediction Service

**Create file `k8s/retail-api-deployment.yaml`:**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: mlops-retail-forecast
  labels:
    app: retail-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-api
  template:
    metadata:
      labels:
        app: retail-api
    spec:
      serviceAccountName: retail-api-sa # IRSA Service Account
      containers:
        - name: retail-api
          image: ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: AWS_DEFAULT_REGION
              value: "ap-southeast-1"
            - name: S3_BUCKET_MODELS
              value: "mlops-retail-forecast-models"
            - name: S3_BUCKET_DATA
              value: "mlops-retail-forecast-data"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: mlops-retail-forecast
  labels:
    app: retail-api
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8000
      name: http
  selector:
    app: retail-api
```

### 5.2. Deploy Application with IRSA

```bash
# Replace ACCOUNT_ID in deployment file
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed "s/ACCOUNT_ID/$ACCOUNT_ID/g" k8s/retail-api-deployment.yaml | kubectl apply -f -

# Verify deployment
kubectl get deployments -n mlops-retail-forecast
kubectl get pods -n mlops-retail-forecast
kubectl get services -n mlops-retail-forecast

# Check logs to verify IRSA working
kubectl logs -l app=retail-api -n mlops-retail-forecast
```

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
# Deploy service accounts với IRSA annotations
kubectl apply -f aws/k8s/service-accounts.yaml

# Deploy test pod với IRSA authentication
kubectl apply -f aws/k8s/test-pod-irsa.yaml

# Verify pod có thể access S3 qua IRSA (no AWS credentials needed!)
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

## 7. Monitoring và Logging

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
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-cloudwatch
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloudwatch-agent
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      name: cloudwatch-agent
  template:
    metadata:
      labels:
        name: cloudwatch-agent
    spec:
      serviceAccountName: cloudwatch-agent
      containers:
        - name: cloudwatch-agent
          image: amazon/cloudwatch-agent:1.300026.2b251814
          env:
            - name: AWS_REGION
              value: ap-southeast-1
            - name: CLUSTER_NAME
              value: mlops-retail-forecast-dev-cluster
          volumeMounts:
            - name: cwagentconfig
              mountPath: /etc/cwagentconfig
            - name: rootfs
              mountPath: /rootfs
              readOnly: true
            - name: dockersock
              mountPath: /var/run/docker.sock
              readOnly: true
            - name: varlibdocker
              mountPath: /var/lib/docker
              readOnly: true
      volumes:
        - name: cwagentconfig
          configMap:
            name: cwagentconfig
        - name: rootfs
          hostPath:
            path: /
        - name: dockersock
          hostPath:
            path: /var/run/docker.sock
        - name: varlibdocker
          hostPath:
            path: /var/lib/docker
```

## 8. Security Hardening

### 8.1. Network Security

```bash
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

## 9. Dọn dẹp Resources (AWS CLI)

### 9.1. Xóa Node Groups

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

### 9.2. Xóa IRSA Roles và OIDC Provider

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

### 9.3. Xóa EKS Cluster

```bash
# Delete the cluster (ensure node groups are deleted first)
aws eks delete-cluster --name mlops-retail-cluster --region ap-southeast-1

# Monitor deletion progress
aws eks describe-cluster --name mlops-retail-cluster --region ap-southeast-1 --query 'cluster.status'

# Verify cluster is deleted (should return error)
aws eks list-clusters --region ap-southeast-1 --query 'clusters[?contains(@, `mlops-retail-cluster`)]'
```

### 9.4. Dọn dẹp Kubernetes Resources

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
kubectl delete namespace mlops-retail-forecast --ignore-not-found=true

# 2. Delete node group
echo "Deleting node group..."
aws eks delete-nodegroup \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $NODEGROUP_NAME \
    --region $REGION

# Wait for node group deletion
echo "Waiting for node group deletion..."
aws eks wait nodegroup-deleted \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $NODEGROUP_NAME \
    --region $REGION

# 3. Delete IRSA roles
echo "Cleaning up IRSA roles..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Delete S3 role
aws iam detach-role-policy --role-name mlops-irsa-s3-access-role --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/mlops-irsa-s3-policy 2>/dev/null
aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/mlops-irsa-s3-policy 2>/dev/null
aws iam delete-role --role-name mlops-irsa-s3-access-role 2>/dev/null

# Delete CloudWatch role
aws iam detach-role-policy --role-name mlops-irsa-cloudwatch-role --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy 2>/dev/null
aws iam delete-role --role-name mlops-irsa-cloudwatch-role 2>/dev/null

# 4. Delete OIDC provider
echo "Deleting OIDC provider..."
OIDC_URL=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION --query 'cluster.identity.oidc.issuer' --output text 2>/dev/null | sed 's|https://||')
if [ ! -z "$OIDC_URL" ]; then
    aws iam delete-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_URL} 2>/dev/null
fi

# 5. Delete cluster
echo "Deleting EKS cluster..."
aws eks delete-cluster --name $CLUSTER_NAME --region $REGION

# Wait for cluster deletion
echo "Waiting for cluster deletion (this may take 10-15 minutes)..."
aws eks wait cluster-deleted --name $CLUSTER_NAME --region $REGION

# 6. Clean up kubeconfig
echo "Cleaning up kubeconfig..."
kubectl config delete-cluster arn:aws:eks:$REGION:${ACCOUNT_ID}:cluster/$CLUSTER_NAME 2>/dev/null
kubectl config delete-context arn:aws:eks:$REGION:${ACCOUNT_ID}:cluster/$CLUSTER_NAME 2>/dev/null
kubectl config unset users.arn:aws:eks:$REGION:${ACCOUNT_ID}:cluster/$CLUSTER_NAME 2>/dev/null

echo "✅ EKS cleanup completed successfully!"
```

---

## 10. Bảng giá EKS (ap-southeast-1)

### 10.1. Chi phí EKS Control Plane

| Component | Giá (USD/cluster/giờ) | Giá (USD/cluster/tháng) | Ghi chú |
|-----------|------------------------|-------------------------|----------|
| **EKS Control Plane** | $0.10 | $73.00 | Chi phí cố định bất kể mức sử dụng |
| **Fargate** | $0.04048/vCPU/giờ | Biến đổi | Trả theo tài nguyên pod |
| **EC2 Worker Nodes** | Giá EC2 | Biến đổi | t2.micro có FREE tier |

### 10.2. Chi phí EC2 Worker Nodes

| Loại Instance | vCPU | Memory | Giá (USD/giờ) | Giá (USD/tháng) | Free Tier |
|---------------|------|--------|----------------|-----------------|------------|
| **t2.micro** | 1 | 1GB | $0.0116 | $8.50 | ✅ 750h/tháng |
| **t3.micro** | 2 | 1GB | $0.0104 | $7.61 | ❌ |
| **t3.small** | 2 | 2GB | $0.0208 | $15.22 | ❌ |
| **t3.medium** | 2 | 4GB | $0.0416 | $30.45 | ❌ |
| **m5.large** | 2 | 8GB | $0.096 | $70.27 | ❌ |

### 10.3. Chi phí VPC Endpoints cho EKS

| Loại Endpoint | Giá (USD/giờ/endpoint) | Chi phí hàng tháng (2 AZ) | Ghi chú |
|---------------|-------------------------|---------------------|----------|
| **S3 Gateway** | Miễn phí | $0.00 | Không phí theo giờ |
| **ECR API** | $0.01 | $14.40 | Bắt buộc cho image pulls |
| **ECR Docker** | $0.01 | $14.40 | Bắt buộc cho image pulls |
| **CloudWatch Logs** | $0.01 | $14.40 | Tùy chọn cho logging |
| **EC2** | $0.01 | $14.40 | Tùy chọn cho instance metadata |

### 10.4. Ước tính chi phí cho Task 7

**Cấu hình EKS Cluster:**
- Control Plane: 1 cluster
- Worker Nodes: 2x t2.micro instances
- VPC Endpoints: ECR API + ECR Docker
- Storage: 20GB EBS mỗi node

**Chi phí hàng tháng:**

| Component | Cấu hình | Chi phí hàng tháng | Giảm giá Free Tier |
|-----------|---------------|--------------|--------------------|
| **EKS Control Plane** | 1 cluster | $73.00 | ❌ |
| **t2.micro Instances** | 2x instances (1,500h) | $17.00 | **-$17.00** (MIỄN PHÍ) |
| **EBS Storage** | 40GB gp3 | $3.20 | **-$3.20** (30GB MIỄN PHÍ) |
| **ECR VPC Endpoints** | 2 endpoints × 2 AZ | $28.80 | ❌ |
| **Data Transfer** | VPC nội bộ | $0.00 | ✅ MIỄN PHÍ |
| **Tổng cộng** | | **$122.00** | **-$20.20** |
| **Chi phí thực tế** | | | **$101.80** |

### 10.5. So sánh chi phí với các lựa chọn khác

**EKS vs Self-Managed Kubernetes:**

| Tính năng | EKS | Self-Managed K8s | Tiết kiệm |
|---------|-----|------------------|----------|
| **Control Plane** | $73/tháng | $0 (tự làm) | -$73 |
| **Quản lý hệ thống** | ✅ Được quản lý | ❌ Cập nhật thủ công | Tiết kiệm thời gian |
| **Bản vá bảo mật** | ✅ Tự động | ❌ Thủ công | Bảo mật |
| **Multi-AZ HA** | ✅ Sẵn có | ❌ Thiết lập phức tạp | Độ tin cậy |
| **Tích hợp AWS** | ✅ Native | ❌ Thủ công | Dễ sử dụng |

**EKS vs ECS Fargate:**

| Loại workload | Chi phí EKS | Chi phí ECS Fargate | Thắng |
|----------|----------|------------------|--------|
| **APIs nhỏ (0.25 vCPU)** | $73 + instance | $7.30/tháng | **Fargate** |
| **Batch Jobs** | $73 + compute | Trả theo lần chạy | **Fargate** |
| **Dịch vụ luôn chạy** | $73 + compute | $29.20/tháng | **EKS** |
| **Ứng dụng nhiều service** | $73 + compute | $N × chi phí service | **EKS** |

### 10.6. Chi phí Data Transfer

**Các tình huống EKS Data Transfer:**

| Loại Transfer | Chi phí | Trường hợp sử dụng |
|---------------|------|----------|
| **Pod to Pod (cùng AZ)** | Miễn phí | Giao tiếp microservices |
| **Pod to Pod (khác AZ)** | $0.01/GB | Triển khai multi-AZ |
| **Pod to Internet** | $0.12/GB | API responses cho users |
| **Pod to S3 (VPC Endpoint)** | Miễn phí | Truy cập model/data |
| **Pod to S3 (Internet)** | $0.12/GB | Không có VPC endpoint |

### 10.7. Chiến lược tối ưu chi phí

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
# Mix spot và on-demand cho cost savings
capacity_type = "SPOT"     # 60-70% savings
# hoặc
capacity_type = "ON_DEMAND" # Stable pricing
```

### 10.8. Tối ưu hóa Free Tier

**Lợi ích Free Tier 12 tháng:**
- **750 giờ/tháng** t2.micro EC2 instances
- **30 GB** EBS General Purpose (SSD) storage
- **2 triệu** requests tới Lambda (nếu dùng cho automation)
- **1 GB** CloudWatch Logs (5GB đầu miễn phí)

**Miễn phí vĩnh viễn:**
- **1 triệu** AWS Lambda requests mỗi tháng
- **5 GB** CloudWatch monitoring data
- **S3 transfers** trong cùng region qua VPC endpoints

{{% notice info %}}
**💰 Tóm tắt chi phí cho Task 7:**
- **Chi phí cố định:** $73/tháng (EKS control plane)
- **Chi phí biến đổi:** $28.80/tháng (VPC endpoints)
- **Tiết kiệm Free Tier:** $20.20/tháng (instances + storage)
- **Tổng cộng:** **$81.60/tháng** với free tier optimizations
- **Mở rộng production:** Thêm instance types dựa vào workload needs
{{% /notice %}}

{{% notice success %}}
**Success tip:** Theo dõi cluster costs với AWS Cost Explorer và thiết lập billing alerts. Sử dụng eksctl hoặc Terraform cho infrastructure as code để đảm bảo triển khai consistent và reproducible.
{{% /notice %}}

---

## 👉 Kết quả Task 7

Sau Task 7, bạn sẽ có EKS Cluster production-ready, chạy hoàn toàn trong private subnet và tích hợp với VPC Endpoints từ Task 5, tiết kiệm chi phí NAT Gateway và tăng mức độ bảo mật.

### Deliverables Completed

- **EKS Control Plane ACTIVE**: Managed Kubernetes cluster với multi-AZ high availability
- **IRSA Configured**: OIDC provider và Service Account authentication setup
- **Managed Node Groups**: 2x t2.micro instances (FREE tier) trải đều trên ≥2 AZ
- **VPC Endpoints Integration**: Sử dụng ECR, S3 endpoints từ Task 2 (70% cost savings)
- **Core Add-ons**: VPC CNI, CoreDNS, kube-proxy, metrics-server, EBS CSI driver
- **Secure Pod Access**: Pods có thể access S3/CloudWatch qua IRSA (no hardcoded credentials)
- **kubectl Access**: Local development environment configured và tested
- **Cost Optimization**: $117.40/month saved với FREE tier + VPC Endpoints

{{% notice success %}}
**🎯 Ready for Next Tasks:**

EKS cluster foundation đã sẵn sàng để deploy:

- ✅ **Task 5**: EKS node groups scaling và optimization
- ✅ **Task 6**: ECR repository setup cho container images
- ✅ **Task 7**: Build và push inference API container
- ✅ **Task 8**: S3 data storage integration
- ✅ **Task 13**: Deploy inference API lên EKS cluster
  {{% /notice %}}

{{% notice warning %}}
**🔐 Lưu ý Bảo mật & Bảo trì:**

- **Public Access**: Giới hạn `cluster_endpoint_public_access_cidrs` theo IP ranges thực tế trong production
- **Logging**: Bật tất cả control plane logging để audit bảo mật  
- **Updates**: Thường xuyên cập nhật Kubernetes version và add-ons (hàng quý)
- **RBAC**: Triển khai role-based access control phù hợp cho team members
- **Monitoring**: Thiết lập alerts cho node health, pod failures, và resource usage  
- **Backup**: Cân nhắc EKS cluster backup strategy cho disaster recovery
  {{% /notice %}}

## 🎬 Video thực hiện Task 7

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