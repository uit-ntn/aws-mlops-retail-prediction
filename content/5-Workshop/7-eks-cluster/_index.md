---
title: "EKS Cluster Setup (Production)"
date: 2024-01-01T00:00:00Z
weight: 8
chapter: false
pre: "<b>7. </b>"
---

{{% notice info %}}
**🎯 Task 7 Goal:** Provision an Amazon EKS cluster in **ap-southeast-1** connected to the Production VPC, enable essential add-ons/logging, configure IRSA (OIDC) for AWS access from pods, and deploy a sample app that pulls from ECR.
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

---

## 7) Cleanup (recommended for cost control)

```bash
kubectl delete namespace mlops-retail-forecast --ignore-not-found=true
eksctl delete cluster --name mlops-retail-cluster --region ap-southeast-1
```

---

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
