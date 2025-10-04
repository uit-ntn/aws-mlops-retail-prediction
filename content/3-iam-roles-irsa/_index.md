---
title: "IAM Roles & IRSA Security"
date: 2025-08-30T12:00:00+07:00
weight: 3
chapter: false
pre: "<b>3. </b>"
---

## 🎯 Mục tiêu Task 3

Thiết lập **Basic IAM Roles** cho MLOps platform với **least privilege principle**:

1. **EKS Service Roles** (Console) - EKS Cluster, Node Group roles
2. **SageMaker Execution Role** (Console) - Training và deployment permissions
3. **Foundation Security** - Prepare IAM roles cho EKS cluster creation ở Task 4

{{% notice info %}}
**💡 Task 3 Focus - Basic IAM Roles:**
- ✅ **EKS Cluster Service Role** - Basic cluster permissions
- ✅ **EKS Node Group Role** - EC2 instance permissions  
- ✅ **SageMaker Execution Role** - Training job permissions
- ✅ **Simple policy attachments** - AWS managed policies

**IRSA sẽ được setup ở Task 4** sau khi EKS cluster đã được tạo
{{% /notice %}}

📥 **Input**
- AWS Account với admin permissions
- Project naming convention: `mlops-retail-forecast-dev`
- Target region: `ap-southeast-1`

📌 **Các bước chính**
1. **EKS Cluster Service Role** - Permissions cho EKS control plane
2. **EKS Node Group Role** - Permissions cho EC2 worker nodes
3. **SageMaker Execution Role** - Permissions cho training jobs và model deployment

✅ **Deliverables**
- EKS Cluster Service Role (Console)
- EKS Node Group Role với ECR/S3/CloudWatch permissions (Console)
- SageMaker Execution Role với S3 access (Console)

📊 **Acceptance Criteria**
- EKS cluster có thể được tạo với proper service role
- Node groups có thể pull images từ ECR
- SageMaker training jobs có thể read/write S3 buckets
- All IAM roles follow least privilege principle

⚠️ **Gotchas**
- EKS Cluster role cần AmazonEKSClusterPolicy
- Node Group role cần AmazonEKSWorkerNodePolicy + AmazonEKS_CNI_Policy
- SageMaker role cần AmazonSageMakerFullAccess + S3 permissions
- Role names phải follow naming convention cho Task 4

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

### 1.1. EKS Service Roles

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



### 1.5. Quick Verification

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
**🎯 Task 3 Complete!**

Basic IAM roles đã sẵn sàng cho EKS cluster creation ở Task 4:
- ✅ EKS Cluster Service Role
- ✅ EKS Node Group Role  
- ✅ SageMaker Execution Role

**IRSA setup sẽ được thực hiện ở Task 4** sau khi EKS cluster đã được tạo.
{{% /notice %}}

## 👉 Kết quả Task 3

✅ **EKS Cluster Service Role** (Console): AmazonEKSClusterPolicy cho control plane  
✅ **EKS Node Group Role** (Console): Worker node permissions với ECR/S3/CloudWatch  
✅ **SageMaker Execution Role** (Console): Training jobs và model deployment  
✅ **Security Foundation**: Basic least privilege principles cho core services  

{{% notice success %}}
**🎯 Task 3 Complete!**

**Console Setup:** Essential IAM roles cho EKS và SageMaker  
**Foundation Ready:** Roles đã sẵn sàng cho EKS cluster creation ở Task 4  
**Next:** IRSA setup sẽ được thực hiện ở Task 4 sau khi EKS cluster ready
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:** 
- **Task 4**: EKS cluster deployment + IRSA setup với OIDC providers
- **Task 5**: EKS managed node groups với configured IAM roles
- **Task 6**: ECR repository setup với IRSA integration
{{% /notice %}}

{{% notice warning %}}
**🔐 Security Reminders**: 
- Role names phải match exactly với Task 4 expectations
- Trust relationships configured correctly cho each service
- Basic permissions sufficient cho cluster creation
- IRSA và advanced security sẽ được setup ở Task 4
{{% /notice %}}