---
title: "IAM Roles & IRSA Security"
date: 2025-08-30T12:00:00+07:00
weight: 3
chapter: false
pre: "<b>3. </b>"
---

## Mục tiêu Task 3

Thiết lập cơ chế phân quyền an toàn cho các thành phần trong hệ thống MLOps, đảm bảo mỗi dịch vụ chỉ có đúng quyền cần thiết để hoạt động:

1. **EKS Node Group IAM Roles** - Quyền cho worker nodes truy cập ECR và CloudWatch
2. **IRSA Configuration** - IAM Roles for Service Accounts để pods truy cập AWS services
3. **SageMaker Execution Roles** - Quyền cho training jobs truy cập S3 và Model Registry
4. **Least Privilege Principle** - Mỗi service chỉ có minimum required permissions
5. **Security Audit Trail** - Proper tagging và naming cho IAM resources

{{% notice info %}}
**📋 Scope Task 3: IAM Security Foundation**

Task này thiết lập secure access control cho toàn bộ MLOps platform:
- ✅ EKS Node Group roles với ECR và CloudWatch access
- ✅ IRSA cho pods truy cập S3 security
- ✅ SageMaker execution roles với S3 và Model Registry permissions
- ✅ CI/CD service roles cho ECR push/pull
- ✅ Cross-service authentication với least privilege
{{% /notice %}}

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

## 1. Alternative: AWS Console Implementation

### 1.1. Tạo IAM Roles qua Console

1. **Navigate to IAM Console:**
   - Đăng nhập AWS Console
   - Navigate to IAM service
   - Chọn "Roles" → "Create role"

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

2. **Attach Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```

![Node Group Policies](../images/03-iam-roles-irsa/04-node-group-policies.png)

3. **Role Details:**
   ```
   Role name: mlops-retail-forecast-dev-eks-nodegroup-role
   Description: EKS node group role with ECR and CloudWatch access
   ```

### 1.4. SageMaker Execution Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: SageMaker
   ```

2. **Attach Policies:**
   ```
   ✅ AmazonSageMakerFullAccess
   ```

![SageMaker Role Policies](../images/03-iam-roles-irsa/05-sagemaker-role-policies.png)

3. **Custom S3 Policy:**
   - Click "Create policy" → JSON tab
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

### 1.6. Verification qua Console

1. **IAM Roles Summary:**
   Navigate to IAM → Roles và verify:
   ```
   ✅ mlops-retail-forecast-dev-eks-cluster-role
   ✅ mlops-retail-forecast-dev-eks-nodegroup-role
   ✅ mlops-retail-forecast-dev-sagemaker-execution
   ✅ mlops-retail-forecast-dev-irsa-s3-access
   ✅ mlops-retail-forecast-dev-irsa-cloudwatch
   ```

![Roles Overview](../images/03-iam-roles-irsa/10-roles-overview.png)

2. **Trust Relationships Verification:**
   - Click vào từng role
   - Verify Trust relationships tab
   - Đảm bảo correct trusted entities

![Trust Relationships](../images/03-iam-roles-irsa/11-trust-relationships.png)

{{% notice success %}}
**🎯 Console Implementation Complete!**

Bạn đã tạo thành công tất cả IAM roles cần thiết qua AWS Console. Các roles này sẽ được sử dụng trong EKS cluster và SageMaker deployment.
{{% /notice %}}

{{% notice info %}}
**💡 Console vs Terraform:**

**Console Advantages:**
- ✅ Visual policy builder dễ hiểu
- ✅ Real-time validation
- ✅ Step-by-step guidance

**Terraform Advantages:**
- ✅ Infrastructure as Code
- ✅ Version control
- ✅ Reproducible deployments

Khuyến nghị: Học Console để hiểu concepts, dùng Terraform cho production.
{{% /notice %}}

## 2. EKS IAM Roles Setup (Terraform)

### 2.1. EKS Cluster Service Role

**File: `aws/infra/iam-eks.tf`**

```hcl
# EKS Cluster Service Role
resource "aws_iam_role" "eks_cluster_role" {
  name = "${var.project_name}-${var.environment}-eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-cluster-role"
    Type = "iam-role"
    Service = "eks-cluster"
  })
}

# Attach required policies to EKS Cluster Role
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

# CloudWatch Logs permissions for EKS Control Plane
resource "aws_iam_role_policy" "eks_cluster_cloudwatch" {
  name = "${var.project_name}-${var.environment}-eks-cluster-cloudwatch"
  role = aws_iam_role.eks_cluster_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams",
          "logs:DescribeLogGroups"
        ]
        Resource = [
          "arn:aws:logs:ap-southeast-1:*:log-group:/aws/eks/${var.project_name}-${var.environment}-cluster/*"
        ]
      }
    ]
  })
}
```

### 2.2. EKS Node Group IAM Role

```hcl
# EKS Node Group Role
resource "aws_iam_role" "eks_node_group_role" {
  name = "${var.project_name}-${var.environment}-eks-nodegroup-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-nodegroup-role"
    Type = "iam-role"
    Service = "eks-nodegroup"
  })
}

# Required policies for EKS Node Group
resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  role       = aws_iam_role.eks_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = aws_iam_role.eks_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "eks_container_registry_policy" {
  role       = aws_iam_role.eks_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

# CloudWatch Agent permissions for Node Group
resource "aws_iam_role_policy_attachment" "eks_cloudwatch_agent_policy" {
  role       = aws_iam_role.eks_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# Additional S3 permissions for node group (for pulling ML artifacts)
resource "aws_iam_role_policy" "eks_node_s3_access" {
  name = "${var.project_name}-${var.environment}-eks-node-s3"
  role = aws_iam_role.eks_node_group_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts",
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-artifacts/*"
        ]
      }
    ]
  })
}
```

## 3. IRSA (IAM Roles for Service Accounts) - Terraform

### 3.1. OIDC Identity Provider

```hcl
# Get EKS cluster OIDC issuer URL
data "aws_eks_cluster" "main" {
  name = aws_eks_cluster.main.name
}

data "tls_certificate" "eks_oidc" {
  url = data.aws_eks_cluster.main.identity[0].oidc[0].issuer
}

# Create OIDC Identity Provider
resource "aws_iam_openid_connect_provider" "eks_oidc" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks_oidc.certificates[0].sha1_fingerprint]
  url             = data.aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-eks-oidc"
    Type = "oidc-provider"
  })
}
```

### 3.2. IRSA Role for S3 Access

```hcl
# IRSA Role for ML workloads to access S3
resource "aws_iam_role" "irsa_s3_access" {
  name = "${var.project_name}-${var.environment}-irsa-s3-access"

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

# S3 access policy for IRSA
resource "aws_iam_role_policy" "irsa_s3_policy" {
  name = "${var.project_name}-${var.environment}-irsa-s3-policy"
  role = aws_iam_role.irsa_s3_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
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

### 3.3. IRSA Role for CloudWatch Monitoring

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

## 4. SageMaker IAM Roles - Terraform

### 4.1. SageMaker Execution Role

**File: `aws/infra/iam-sagemaker.tf`**

```hcl
# SageMaker Execution Role
resource "aws_iam_role" "sagemaker_execution_role" {
  name = "${var.project_name}-${var.environment}-sagemaker-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "sagemaker.amazonaws.com"
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-sagemaker-execution"
    Type = "iam-role"
    Service = "sagemaker"
  })
}

# SageMaker full access policy
resource "aws_iam_role_policy_attachment" "sagemaker_execution_policy" {
  role       = aws_iam_role.sagemaker_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSageMakerFullAccess"
}

# S3 access for SageMaker
resource "aws_iam_role_policy" "sagemaker_s3_access" {
  name = "${var.project_name}-${var.environment}-sagemaker-s3"
  role = aws_iam_role.sagemaker_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket",
          "s3:CreateBucket",
          "s3:GetBucketLocation",
          "s3:ListAllMyBuckets"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-*",
          "arn:aws:s3:::${var.project_name}-${var.environment}-*/*",
          "arn:aws:s3:::sagemaker-*"
        ]
      }
    ]
  })
}

# ECR access for SageMaker (for custom training images)
resource "aws_iam_role_policy" "sagemaker_ecr_access" {
  name = "${var.project_name}-${var.environment}-sagemaker-ecr"
  role = aws_iam_role.sagemaker_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
        Resource = "*"
      }
    ]
  })
}

# CloudWatch Logs for SageMaker
resource "aws_iam_role_policy" "sagemaker_cloudwatch_logs" {
  name = "${var.project_name}-${var.environment}-sagemaker-logs"
  role = aws_iam_role.sagemaker_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams",
          "logs:DescribeLogGroups"
        ]
        Resource = [
          "arn:aws:logs:ap-southeast-1:*:log-group:/aws/sagemaker/*"
        ]
      }
    ]
  })
}
```

### 4.2. SageMaker Model Registry Role

```hcl
# SageMaker Model Registry Role
resource "aws_iam_role" "sagemaker_model_registry_role" {
  name = "${var.project_name}-${var.environment}-sagemaker-model-registry"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = [
            "sagemaker.amazonaws.com",
            "lambda.amazonaws.com"
          ]
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-sagemaker-model-registry"
    Type = "iam-role"
    Service = "sagemaker-registry"
  })
}

# Model Registry specific permissions
resource "aws_iam_role_policy" "sagemaker_model_registry_policy" {
  name = "${var.project_name}-${var.environment}-model-registry"
  role = aws_iam_role.sagemaker_model_registry_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sagemaker:CreateModel",
          "sagemaker:CreateModelPackage",
          "sagemaker:CreateModelPackageGroup",
          "sagemaker:DescribeModel",
          "sagemaker:DescribeModelPackage",
          "sagemaker:DescribeModelPackageGroup",
          "sagemaker:ListModelPackages",
          "sagemaker:ListModelPackageGroups",
          "sagemaker:UpdateModelPackage"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-model-registry/*"
        ]
      }
    ]
  })
}
```

## 5. CI/CD IAM Roles - Terraform

### 5.1. CodeBuild Service Role

```hcl
# CodeBuild Service Role
resource "aws_iam_role" "codebuild_role" {
  name = "${var.project_name}-${var.environment}-codebuild-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codebuild.amazonaws.com"
        }
      }
    ]
  })

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-codebuild-role"
    Type = "iam-role"
    Service = "codebuild"
  })
}

# CodeBuild permissions
resource "aws_iam_role_policy" "codebuild_policy" {
  name = "${var.project_name}-${var.environment}-codebuild-policy"
  role = aws_iam_role.codebuild_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:ap-southeast-1:*:log-group:/aws/codebuild/*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:GetAuthorizationToken",
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
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-artifacts",
          "arn:aws:s3:::${var.project_name}-${var.environment}-artifacts/*"
        ]
      }
    ]
  })
}
```

### 5.2. GitHub Actions Role (OIDC)

```hcl
# GitHub OIDC Provider
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

# GitHub Actions Role
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

# GitHub Actions permissions
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

## 6. Kubernetes Service Accounts

### 6.1. S3 Access Service Account

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

### 6.2. Pod Deployment with IRSA

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

## 7. Verification và Testing - CLI

### 7.1. Terraform Deployment

```bash
# Navigate to infrastructure directory
cd aws/infra

# Plan IAM resources
terraform plan -target=module.iam -var-file="../terraform.tfvars"

# Apply IAM roles
terraform apply -target=module.iam -var-file="../terraform.tfvars"
```

### 7.2. Verify IAM Roles

```bash
# List created IAM roles
aws iam list-roles --query 'Roles[?contains(RoleName, `mlops-retail-forecast`)].{RoleName:RoleName,CreateDate:CreateDate}'

# Get EKS cluster OIDC issuer
aws eks describe-cluster --name mlops-retail-forecast-dev-cluster --query 'cluster.identity.oidc.issuer'

# Verify OIDC provider
aws iam list-open-id-connect-providers
```

### 7.3. Test IRSA Configuration

```bash
# Deploy service accounts
kubectl apply -f aws/k8s/service-accounts.yaml

# Deploy test pod with IRSA
kubectl apply -f aws/k8s/deployment-with-irsa.yaml

# Test S3 access from pod
kubectl exec -it deployment/retail-forecast-api -- aws s3 ls s3://mlops-retail-forecast-dev-ml-data/
```

### 7.4. SageMaker Role Testing

```bash
# Test SageMaker execution role
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-sagemaker-execution \
  --role-session-name test-session

# Verify S3 access from assumed role
aws s3 ls s3://mlops-retail-forecast-dev-ml-artifacts/ --region ap-southeast-1
```

## 8. Security Best Practices

### 8.1. Least Privilege Implementation

```hcl
# Example: Restricted S3 access with conditions
resource "aws_iam_role_policy" "restricted_s3_access" {
  name = "${var.project_name}-${var.environment}-restricted-s3"
  role = aws_iam_role.example_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-ml-data/*"
        ]
        Condition = {
          StringEquals = {
            "s3:x-amz-server-side-encryption" = "AES256"
          }
          IpAddress = {
            "aws:SourceIp" = var.vpc_cidr
          }
        }
      }
    ]
  })
}
```

### 8.2. IAM Policy Validation

```bash
# Validate IAM policies using AWS CLI
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT_ID:role/mlops-retail-forecast-dev-irsa-s3-access \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::mlops-retail-forecast-dev-ml-data/train.csv
```

### 8.3. Monitoring IAM Usage

```hcl
# CloudTrail for IAM audit
resource "aws_cloudtrail" "iam_audit" {
  name           = "${var.project_name}-${var.environment}-iam-audit"
  s3_bucket_name = aws_s3_bucket.cloudtrail_logs.bucket
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::IAM::Role"
      values = ["arn:aws:iam::*:role/${var.project_name}-*"]
    }
  }

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-${var.environment}-iam-audit"
    Type = "cloudtrail"
  })
}
```

## 9. Outputs

**File: `aws/infra/outputs.tf` (thêm vào)**

```hcl
# IAM Role ARNs
output "eks_cluster_role_arn" {
  description = "ARN of the EKS cluster IAM role"
  value       = aws_iam_role.eks_cluster_role.arn
}

output "eks_node_group_role_arn" {
  description = "ARN of the EKS node group IAM role"
  value       = aws_iam_role.eks_node_group_role.arn
}

output "sagemaker_execution_role_arn" {
  description = "ARN of the SageMaker execution role"
  value       = aws_iam_role.sagemaker_execution_role.arn
}

output "irsa_s3_access_role_arn" {
  description = "ARN of the IRSA S3 access role"
  value       = aws_iam_role.irsa_s3_access.arn
}

output "irsa_cloudwatch_role_arn" {
  description = "ARN of the IRSA CloudWatch role"
  value       = aws_iam_role.irsa_cloudwatch_access.arn
}

# OIDC Provider
output "eks_oidc_provider_arn" {
  description = "ARN of the EKS OIDC provider"
  value       = aws_iam_openid_connect_provider.eks_oidc.arn
}

output "github_oidc_provider_arn" {
  description = "ARN of the GitHub OIDC provider"
  value       = aws_iam_openid_connect_provider.github_oidc.arn
}
```

## Kết quả Task 3

✅ **EKS IAM Roles**: Cluster và Node Group roles với proper permissions  
✅ **IRSA Configuration**: Service Account based authentication cho pods  
✅ **SageMaker Roles**: Execution role với S3 và Model Registry access  
✅ **CI/CD Integration**: CodeBuild và GitHub Actions roles  
✅ **Security Controls**: Least privilege với comprehensive audit trail  

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 4**: EKS cluster deployment sử dụng IAM roles
- **Task 5**: EKS managed node groups với configured roles
- **Task 6**: ECR repository setup với IAM integration
{{% /notice %}}

{{% notice warning %}}
**🔐 Security Reminder**: 
- Replace `ACCOUNT_ID` với actual AWS Account ID
- Review và customize IAM policies theo business requirements
- Enable CloudTrail logging cho IAM audit trail
- Regularly rotate credentials và review permissions
{{% /notice %}}