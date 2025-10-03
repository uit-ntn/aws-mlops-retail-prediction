---
title: "IAM Roles & IRSA Security"
date: 2025-08-30T12:00:00+07:00
weight: 3
chapter: false
pre: "<b>3. </b>"
---

## 🎯 Mục tiêu Task 3

Thiết lập IAM security foundation cho MLOps platform với **least privilege principle**:

1. **Basic IAM Roles** (Console) - EKS Cluster, Node Group, SageMaker execution roles
2. **Advanced IRSA Setup** (Terraform) - Service Account based authentication cho pods  
3. **Production Security** (Terraform) - Automated role management, audit trail, cross-service authentication

{{% notice info %}}
**💡 Khi nào cần Terraform cho IAM:**
- ✅ **IRSA (IAM Roles for Service Accounts)** - Complex OIDC integration
- ✅ **Cross-service authentication** với conditional policies
- ✅ **CI/CD automation** với GitHub Actions OIDC
- ✅ **Production audit trail** với CloudTrail integration
- ✅ **Policy templates** và consistent naming/tagging

**Console đủ cho:** Basic service roles (EKS, SageMaker), simple policy attachments
{{% /notice %}}

📥 **Input**
- AWS Account với admin permissions
- Project naming convention: `mlops-retail-forecast-dev`
- Target region: `ap-southeast-1`

📌 **Các bước chính**
1. **Console Setup** - Tạo basic service roles (EKS, SageMaker)
2. **Terraform Advanced** - IRSA, OIDC providers, automated policies
3. **Integration Testing** - Verify cross-service authentication

✅ **Deliverables**
- EKS Cluster & Node Group roles (Console hoặc Terraform)
- IRSA configuration với OIDC provider (Terraform)
- SageMaker execution role với S3 access (Console hoặc Terraform)
- Service Accounts với proper annotations (Kubernetes)

📊 **Acceptance Criteria**
- EKS pods có thể access S3 qua IRSA (không cần hardcoded credentials)
- SageMaker training jobs có thể read/write S3 buckets
- All IAM roles follow least privilege principle
- Audit trail enabled cho security compliance

⚠️ **Gotchas**
- IRSA requires exact namespace/service account matching
- OIDC provider thumbprint có thể thay đổi theo region
- SageMaker cần cả execution role và optional model registry permissions
- GitHub Actions OIDC cần repository-specific conditions

## Kiến trúc IAM Security

### Security Architecture Overview

```
IAM Security Architecture
├── EKS Cluster
│   ├── EKS Cluster Service Role
│   │   ├── AmazonEKSClusterPolicy
│   │   └── CloudWatch Logs permissions
│   ├── EKS Node Group Role
│   │   ├── AmazonEKSWorkerNodePolicy
│   │   ├── AmazonEKS_CNI_Policy
│   │   ├── AmazonEC2ContainerRegistryReadOnly
│   │   └── CloudWatchAgentServerPolicy
│   └── IRSA Service Accounts
│       ├── S3 Access Role (for ML workloads)
│       ├── CloudWatch Role (for monitoring)
│       └── Secrets Manager Role (for credentials)
├── SageMaker Services
│   ├── SageMaker Execution Role
│   │   ├── S3 Full Access (specific buckets)
│   │   ├── SageMaker Full Access
│   │   └── CloudWatch Logs permissions
│   └── Model Registry Role
│       ├── SageMaker Model Registry
│       └── S3 Model Artifacts access
└── CI/CD Services
    ├── CodeBuild Service Role
    │   ├── ECR Full Access
    │   ├── S3 Artifacts access
    │   └── CloudWatch Logs
    └── GitHub Actions Role
        ├── ECR push/pull permissions
        └── EKS deployment access
```

### Security Benefits

- **🔐 Least Privilege**: Mỗi service chỉ có minimum required permissions
- **🛡️ Service Isolation**: Cross-service access được control chặt chẽ
- **📊 Audit Trail**: Comprehensive logging cho security compliance
- **🚀 IRSA Integration**: Secure pod-level permissions without long-lived credentials
- **🔄 Rotation Ready**: Support credential rotation và temporary access

{{% notice success %}}
**🎯 Security Best Practices:** Zero Trust Architecture với IAM

**Production Security:**
- ✅ **No hardcoded credentials** trong application code
- ✅ **Temporary credentials** thông qua IRSA và instance profiles
- ✅ **Fine-grained permissions** cho từng AWS service
- ✅ **Audit logging** cho all IAM actions
{{% /notice %}}

## 1. Console Setup - Basic Service Roles

{{% notice success %}}
**🎯 Console Approach:** Quick setup cho learning và development

**Ưu điểm:**
- ✅ Visual policy builder dễ hiểu
- ✅ Real-time validation và error checking  
- ✅ Step-by-step guidance từ AWS
- ✅ Immediate testing và verification

**Nhược điểm:**
- ❌ Không reproducible across environments
- ❌ Manual process, dễ sai sót
- ❌ Khó version control và audit
{{% /notice %}}

### 1.1. EKS Service Roles (Console)

**Navigate to IAM Console:**
- AWS Console → IAM → Roles → "Create role"

![Create IAM Role](../images/03-iam-roles-irsa/01-create-iam-role.png)

### 1.2. EKS Cluster Service Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EKS - Cluster
   ```

![EKS Cluster Trusted Entity](../images/03-iam-roles-irsa/02-eks-cluster-trusted-entity.png)

2. **Permissions:**
   ```
   Policy: AmazonEKSClusterPolicy
   ```
3. **Role Details:**
   ```
   Role name: mlops-retail-forecast-dev-eks-cluster-role
   Description: EKS cluster service role for MLOps platform
   ```

![EKS Cluster Role Config](../images/03-iam-roles-irsa/03-eks-cluster-role-config.png)


### 1.3. EKS Node Group Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EC2
   ```
![Node Group Policies](../images/03-iam-roles-irsa/04.1-node-group-policies.png)


2. **Attach Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```

3. **Role Details:**
   ```
   Role name: mlops-retail-forecast-dev-eks-nodegroup-role
   Description: EKS node group role with ECR and CloudWatch access
   ```
![Node Group Policies](../images/03-iam-roles-irsa/04.3-node-group-policies.png)

![Node Group Policies](../images/03-iam-roles-irsa/04.4-node-group-policies.png)


### 1.4. SageMaker Execution Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: SageMaker
   ```

![SageMaker Trusted Entity](../images/03-iam-roles-irsa/05.1-sagemaker-role-policies.png)

2. **Attach Policies:**
   ```
   ✅ AmazonSageMakerFullAccess
   ```

![SageMaker Attach Policies](../images/03-iam-roles-irsa/05.2-sagemaker-role-policies.png)

3. **Role Details:**
   ```
   Role name: mlops-retail-forecast-dev-sagemaker-execution
   Description: SageMaker execution role for MLOps training jobs and model deployment
   ```

![SageMaker Role Details](../images/03-iam-roles-irsa/05.3-sagemaker-role-details.png)

![SageMaker Role Details](../images/03-iam-roles-irsa/05.4-sagemaker-role-details.png)


4. **Custom S3 Policy (Optional):**
   - Nếu cần restrict S3 access, click "Create policy" → JSON tab
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:PutObject",
           "s3:DeleteObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::mlops-retail-forecast-dev-*",
           "arn:aws:s3:::mlops-retail-forecast-dev-*/*",
           "arn:aws:s3:::sagemaker-*"
         ]
       }
     ]
   }
   ```

![Custom S3 Policy](../images/03-iam-roles-irsa/06-custom-s3-policy.png)

### 1.5. IRSA Setup qua Console

1. **Get EKS OIDC Issuer URL:**
   - Navigate to EKS Console
   - Chọn cluster đã tạo
   - Copy OIDC issuer URL

![EKS OIDC Issuer](../images/03-iam-roles-irsa/07-eks-oidc-issuer.png)

2. **Create OIDC Identity Provider:**
   - Navigate to IAM → Identity providers
   - Choose "OpenID Connect"
   ```
   Provider URL: [EKS OIDC issuer URL]
   Audience: sts.amazonaws.com
   ```

![Create OIDC Provider](../images/03-iam-roles-irsa/08-create-oidc-provider.png)

3. **Create IRSA Role for S3 Access:**
   
   **Trusted Entity - Custom trust policy:**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLE"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:mlops-retail-forecast:s3-access-sa",
             "oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com"
           }
         }
       }
     ]
   }
   ```

![IRSA Trust Policy](../images/03-iam-roles-irsa/09-irsa-trust-policy.png)

4. **Custom S3 Access Policy:**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:PutObject",
           "s3:DeleteObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::mlops-retail-forecast-dev-ml-data",
           "arn:aws:s3:::mlops-retail-forecast-dev-ml-data/*",
           "arn:aws:s3:::mlops-retail-forecast-dev-ml-artifacts",
           "arn:aws:s3:::mlops-retail-forecast-dev-ml-artifacts/*"
         ]
       }
     ]
   }
   ```

5. **Role Details:**
   ```
   Role name: mlops-retail-forecast-dev-irsa-s3-access
   Description: IRSA role for pods to access S3 buckets
   ```

### 1.6. Quick Verification

**IAM Roles Summary:**
   Navigate to IAM → Roles và verify:
   ```
   ✅ mlops-retail-forecast-dev-eks-cluster-role
   ✅ mlops-retail-forecast-dev-eks-nodegroup-role
   ✅ mlops-retail-forecast-dev-sagemaker-execution
```

![Roles Overview](../images/03-iam-roles-irsa/10-roles-overview.png)

**Trust Relationships Check:**
- Click vào từng role → Trust relationships tab
- Verify correct trusted entities (eks.amazonaws.com, ec2.amazonaws.com, sagemaker.amazonaws.com)

![Trust Relationships](../images/03-iam-roles-irsa/11-trust-relationships.png)

{{% notice success %}}
**🎯 Console Setup Complete!**

Basic service roles đã sẵn sàng cho EKS và SageMaker. Tiếp theo sẽ setup IRSA với Terraform cho advanced authentication.
{{% /notice %}}

## 2. Terraform cho Advanced IRSA & Automation

{{% notice info %}}
**💡 Khi nào cần Terraform cho IAM:**
- ✅ **IRSA (IAM Roles for Service Accounts)** - Complex OIDC integration không thể làm dễ dàng qua Console
- ✅ **CI/CD automation** với GitHub Actions OIDC provider
- ✅ **Production environments** cần consistent, reproducible IAM setup
- ✅ **Cross-service policies** với complex conditions và resource restrictions

**Console đủ cho:** EKS/SageMaker service roles, basic policy attachments
{{% /notice %}}

### 2.0. Terraform Code Purpose & Expected Results

{{% notice success %}}
**🎯 Mục đích của Terraform code trong Task 3:**

**Input:** 
- EKS cluster từ Task 4 (OIDC issuer URL)
- Basic service roles từ Console (EKS, SageMaker roles)
- GitHub repository information cho CI/CD

**Terraform sẽ làm gì:**
1. **Create OIDC providers** cho EKS và GitHub Actions
2. **Setup IRSA roles** với fine-grained S3 và CloudWatch permissions
3. **Configure CI/CD automation** với GitHub Actions OIDC
4. **Implement security policies** với least privilege principles
5. **Enable audit trail** cho compliance và monitoring

**Kết quả sau khi chạy:**
- ✅ IRSA hoạt động: Pods access S3 không cần hardcoded credentials
- ✅ GitHub Actions có thể deploy lên EKS securely
- ✅ Service Accounts với proper annotations
- ✅ Least privilege: Mỗi service chỉ có minimum required permissions
- ✅ Audit ready: CloudTrail integration cho security compliance
- ✅ Production security: Zero long-lived credentials
{{% /notice %}}

### 2.1. IRSA Foundation - OIDC Provider

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Tìm EKS cluster** đã tạo ở Task 4 để lấy OIDC issuer URL
2. **Get SSL certificate** từ EKS OIDC endpoint cho security validation
3. **Create OIDC Identity Provider** trong AWS IAM để trust EKS cluster
4. **Enable IRSA authentication** cho Kubernetes Service Accounts

**Kết quả:** AWS IAM có thể trust và authenticate Kubernetes Service Accounts
{{% /notice %}}

**File: `aws/infra/iam-irsa.tf`**

```hcl
# BƯỚC 1: Tìm EKS cluster từ Task 4 (không tạo mới)
data "aws_eks_cluster" "main" {
  name = "${var.project_name}-${var.environment}-cluster"  # EKS từ Task 4
}

# BƯỚC 2: Get OIDC issuer certificate cho security validation
data "tls_certificate" "eks_oidc" {
  url = data.aws_eks_cluster.main.identity[0].oidc[0].issuer  # EKS OIDC endpoint
}

# BƯỚC 3: Create OIDC Identity Provider trong AWS IAM
resource "aws_iam_openid_connect_provider" "eks_oidc" {
  client_id_list  = ["sts.amazonaws.com"]  # AWS STS service
  thumbprint_list = [data.tls_certificate.eks_oidc.certificates[0].sha1_fingerprint]  # SSL cert validation
  url             = data.aws_eks_cluster.main.identity[0].oidc[0].issuer  # EKS OIDC URL

  # Purpose: Cho phép AWS IAM trust Kubernetes Service Accounts
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-oidc"
    Type = "oidc-provider"
    Purpose = "irsa-authentication"
  })
}
```

### 2.2. IRSA Role for ML Workloads (S3 Access)

{{% notice tip %}}
**🔍 Code này làm gì:**
1. **Create IAM role** chỉ có thể được assume bởi specific Kubernetes Service Account
2. **Setup trust policy** với exact namespace và service account matching
3. **Grant S3 permissions** chỉ cho ML data buckets (least privilege)
4. **Enable secure access** từ pods mà không cần hardcoded AWS credentials

**Kết quả:** Pods với Service Account `s3-access-sa` có thể access S3 securely
{{% /notice %}}

```hcl
# BƯỚC 1: Create IRSA Role cho ML workloads access S3
resource "aws_iam_role" "irsa_s3_access" {
  name = "${var.project_name}-${var.environment}-irsa-s3-access"

  # Trust Policy: CHỈ specific Service Account có thể assume role này
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks_oidc.arn  # OIDC provider từ bước trước
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            # QUAN TRỌNG: Exact match namespace và service account name
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:sub" = "system:serviceaccount:mlops-retail-forecast:s3-access-sa"
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-irsa-s3-access"
    Type = "iam-role"
    Service = "irsa-s3"
  })
}

# BƯỚC 2: S3 access policy - LEAST PRIVILEGE cho ML buckets only
resource "aws_iam_role_policy" "irsa_s3_policy" {
  name = "${var.project_name}-${var.environment}-irsa-s3-policy"
  role = aws_iam_role.irsa_s3_access.id

  # Permissions: Chỉ access ML data buckets, không phải tất cả S3
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",     # Read files
          "s3:PutObject",     # Upload files  
          "s3:DeleteObject",  # Delete files
          "s3:ListBucket"     # List bucket contents
        ]
        Resource = [
          # CHỈ access specific ML buckets
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-data",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-data/*",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts/*"
        ]
      }
    ]
  })
}
```

### 2.3. IRSA Role for CloudWatch Monitoring

```hcl
# IRSA Role for CloudWatch monitoring
resource "aws_iam_role" "irsa_cloudwatch_access" {
  name = "${var.project_name}-${var.environment}-irsa-cloudwatch"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks_oidc.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:sub" = "system:serviceaccount:mlops-retail-forecast:cloudwatch-sa"
            "${replace(aws_iam_openid_connect_provider.eks_oidc.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-irsa-cloudwatch"
    Type = "iam-role"
    Service = "irsa-cloudwatch"
  })
}

# CloudWatch permissions for IRSA
resource "aws_iam_role_policy_attachment" "irsa_cloudwatch_policy" {
  role       = aws_iam_role.irsa_cloudwatch_access.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# Custom CloudWatch metrics policy
resource "aws_iam_role_policy" "irsa_cloudwatch_custom" {
  name = "${var.project_name}-${var.environment}-irsa-cloudwatch-custom"
  role = aws_iam_role.irsa_cloudwatch_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData",
          "cloudwatch:GetMetricStatistics",
          "cloudwatch:ListMetrics"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudwatch:namespace" = [
              "MLOps/RetailForecast",
              "AWS/EKS",
              "ContainerInsights"
            ]
          }
        }
      }
    ]
  })
}
```

## 3. CI/CD Automation với GitHub Actions OIDC

{{% notice info %}}
**💡 GitHub Actions OIDC:** 
Thay thế long-lived access keys bằng temporary credentials cho CI/CD pipeline. Chỉ cần Terraform - không thể setup qua Console.
{{% /notice %}}

### 3.1. GitHub OIDC Provider

**File: `aws/infra/iam-cicd.tf`**

```hcl
# GitHub OIDC Provider cho CI/CD
resource "aws_iam_openid_connect_provider" "github_oidc" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com"
  ]

  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-github-oidc"
    Type = "oidc-provider"
    Service = "github-actions"
  })
}

# GitHub Actions Role với repository restrictions
resource "aws_iam_role" "github_actions_role" {
  name = "${var.project_name}-${var.environment}-github-actions"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github_oidc.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:your-org/${var.project_name}:*"
          }
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-github-actions"
    Type = "iam-role"
    Service = "github-actions"
  })
}

# GitHub Actions permissions cho ECR và EKS
resource "aws_iam_role_policy" "github_actions_policy" {
  name = "${var.project_name}-${var.environment}-github-actions-policy"
  role = aws_iam_role.github_actions_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:DescribeNodegroup"
        ]
        Resource = [
          "arn:aws:eks:ap-southeast-1:*:cluster/${var.project_name}-${var.environment}-cluster"
        ]
      }
    ]
  })
}
```

## 4. Kubernetes Service Accounts Integration

### 4.1. Service Accounts với IRSA Annotations

**File: `aws/k8s/service-accounts.yaml`**

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: mlops-retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-s3-access
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
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-cloudwatch
  labels:
    app.kubernetes.io/name: cloudwatch-service-account
    app.kubernetes.io/component: monitoring
```

### 4.2. Pod Deployment với IRSA Authentication

**File: `aws/k8s/deployment-with-irsa.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-forecast-api
  namespace: mlops-retail-forecast
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-forecast
  template:
    metadata:
      labels:
        app: retail-forecast
    spec:
      serviceAccountName: s3-access-sa  # IRSA Service Account
      containers:
      - name: retail-forecast
        image: ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/retail-forecast:latest
        ports:
        - containerPort: 8000
        env:
        - name: AWS_DEFAULT_REGION
          value: "ap-southeast-1"
        - name: S3_BUCKET
          value: "mlops-retail-forecast-dev-ml-data"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        # No AWS credentials needed - IRSA handles authentication
```

## 5. Deployment & Verification

### 5.1. Terraform Deployment (IRSA Only)

{{% notice success %}}
**🚀 IRSA Deployment Process:**

**Bước 1:** Terraform tìm EKS cluster từ Task 4  
**Bước 2:** Create OIDC Identity Provider cho IRSA  
**Bước 3:** Setup IRSA roles với least privilege policies  
**Bước 4:** Configure Service Accounts với proper annotations  
**Bước 5:** Test secure access từ pods (no hardcoded credentials!)  

**Time required:** ~5-10 phút cho IRSA setup
{{% /notice %}}

```bash
# BƯỚC 1: Navigate to infrastructure directory
cd aws/infra

# BƯỚC 2: Plan IRSA resources (xem Terraform sẽ tạo gì)
terraform plan -target=aws_iam_openid_connect_provider.eks_oidc \
               -target=aws_iam_role.irsa_s3_access \
               -target=aws_iam_role.irsa_cloudwatch_access \
               -var-file="terraform.tfvars"

# BƯỚC 3: Apply IRSA configuration
terraform apply -target=aws_iam_openid_connect_provider.eks_oidc \
                -target=aws_iam_role.irsa_s3_access \
                -target=aws_iam_role.irsa_cloudwatch_access \
                -var-file="terraform.tfvars"
```

**Expected Apply Output:**
```
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Resources Created:
✅ aws_iam_openid_connect_provider.eks_oidc (OIDC provider cho EKS)
✅ aws_iam_role.irsa_s3_access (S3 access role cho ML workloads)
✅ aws_iam_role_policy.irsa_s3_policy (Least privilege S3 permissions)
✅ aws_iam_role.irsa_cloudwatch_access (CloudWatch monitoring role)
✅ aws_iam_role_policy_attachment.irsa_cloudwatch_policy (CloudWatch permissions)

Security Benefits:
🔐 Zero hardcoded credentials trong pods
🔐 Least privilege: Chỉ access specific S3 buckets
🔐 Service Account based authentication
🔐 Audit trail ready cho compliance
```

### 5.2. Verify IRSA Setup

```bash
# Verify OIDC provider
aws iam list-open-id-connect-providers

# Get EKS cluster OIDC issuer
aws eks describe-cluster --name mlops-retail-forecast-dev-cluster --query 'cluster.identity.oidc.issuer'

# List IRSA roles
aws iam list-roles --query 'Roles[?contains(RoleName, `irsa`)].{RoleName:RoleName,CreateDate:CreateDate}'
```

### 5.3. Test IRSA Authentication

```bash
# Deploy service accounts
kubectl apply -f aws/k8s/service-accounts.yaml

# Deploy test pod with IRSA
kubectl apply -f aws/k8s/deployment-with-irsa.yaml

# Test S3 access from pod (no AWS credentials needed!)
kubectl exec -it deployment/retail-forecast-api -- aws s3 ls s3://mlops-retail-forecast-dev-ml-data/

# Verify CloudWatch access
kubectl logs deployment/retail-forecast-api
```

## 6. Security Best Practices & Monitoring

### 6.1. Policy Validation

```bash
# Validate IRSA policies
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-s3-access \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::mlops-retail-forecast-dev-ml-data/train.csv

# Test GitHub Actions OIDC
aws sts get-caller-identity --region ap-southeast-1
```

### 6.2. Terraform Outputs

**File: `aws/infra/outputs.tf` (thêm vào)**

```hcl
# IRSA Role ARNs (Terraform managed)
output "irsa_s3_access_role_arn" {
  description = "ARN of the IRSA S3 access role"
  value       = aws_iam_role.irsa_s3_access.arn
}

output "irsa_cloudwatch_role_arn" {
  description = "ARN of the IRSA CloudWatch role"
  value       = aws_iam_role.irsa_cloudwatch_access.arn
}

# OIDC Providers
output "eks_oidc_provider_arn" {
  description = "ARN of the EKS OIDC provider"
  value       = aws_iam_openid_connect_provider.eks_oidc.arn
}

output "github_oidc_provider_arn" {
  description = "ARN of the GitHub OIDC provider"
  value       = aws_iam_openid_connect_provider.github_oidc.arn
}
```

## 👉 Kết quả Task 3

✅ **Basic Service Roles** (Console): EKS Cluster, Node Group, SageMaker execution roles  
✅ **Advanced IRSA** (Terraform): OIDC provider với Service Account authentication  
✅ **CI/CD Integration** (Terraform): GitHub Actions OIDC cho secure deployments  
✅ **Security Foundation**: Least privilege principles với audit capabilities  

{{% notice success %}}
**🎯 Task 3 Optimized!**

**Console Setup:** Quick basic roles cho learning và development  
**Terraform Advanced:** IRSA, OIDC, automation cho production environments  
**Best Practice:** Combine both approaches based on complexity needs
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 4**: EKS cluster deployment sử dụng Console/Terraform roles
- **Task 5**: EKS managed node groups với configured IAM roles
- **Task 6**: ECR repository setup với IRSA integration
{{% /notice %}}

{{% notice warning %}}
**🔐 Security Reminders**: 
- Replace `ACCOUNT_ID` và `your-org` với actual values
- IRSA namespace/service account names phải match exactly
- GitHub repository restrictions trong OIDC conditions
- Regular audit IAM permissions và access patterns
{{% /notice %}}