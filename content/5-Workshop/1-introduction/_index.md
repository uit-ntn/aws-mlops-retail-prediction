---
title: "MLOps Architecture Design"
date: 2025-08-30T11:00:00+07:00
weight: 1
chapter: false
pre: "<b>1. </b>"
---

# AWS MLOps Retail Prediction Platform

**AWS MLOps Retail Prediction** is a complete end-to-end MLOps system built on AWS Cloud, automating the entire lifecycle from infrastructure provisioning, model training, inference API deployment, to monitoring and cost optimization. The project is designed to ensure scalability, reliability, and strong security for real-world Machine Learning applications.

## 1. MLOps Architecture on AWS Cloud

![End-to-end MLOps architecture for Retail Prediction on AWS](/images/01-introduction/MLOps-AWS-Architecture.png)

### 1.1 Project Objectives

**Fully automate the MLOps workflow:**

- 🏗️ **Infrastructure as Code**: Automate infrastructure provisioning using Terraform (VPC, EKS, IAM, EC2, ECR, S3)
- 🤖 **ML Training**: Distributed model training on SageMaker with Model Registry
- 🚀 **Container Deployment**: Package & deploy the inference API on EKS with autoscaling
- 📊 **Monitoring & Security**: Monitoring with CloudWatch, security with KMS & CloudTrail
- 🔄 **CI/CD Pipeline**: Automated pipeline from code/data changes → build → train → deploy
- 💰 **Cost Optimization**: Integrate DataOps and teardown strategies to optimize costs

### 1.2 Overall Flow

**Infrastructure → Training → Deployment → Monitoring → CI/CD → Cost Optimization**

## 2. Customer Price Sensitivity Prediction Problem

### 2.1 Dataset Description - dunnhumby Source Files

**Data source**: [dunnhumby Source Files](https://www.dunnhumby.com/source-files/)  
**Dataset used**: `transaction.csv` (≈ 2.67 million rows, 22 columns)  
**Description**: Each row represents a product in a customer's single shopping transaction.

#### 2.1.1 Detailed Data Schema

| **Column Name**            | **Data Type** | **Description / Meaning**                                                                              | **Example Value** |
| -------------------------- | ------------- | ------------------------------------------------------------------------------------------------------ | ----------------- |
| `SHOP_WEEK`                | `int64`       | Shopping week (format YYYYWW)                                                                          | 200807            |
| `SHOP_DATE`                | `int64`       | Shopping date (format YYYYMMDD)                                                                        | 20080407          |
| `SHOP_WEEKDAY`             | `int64`       | Day of week (1=Sunday, 2=Monday, …, 7=Saturday)                                                        | 2                 |
| `SHOP_HOUR`                | `int64`       | Transaction hour (0–23)                                                                                | 14                |
| `QUANTITY`                 | `int64`       | Quantity purchased in the transaction line                                                             | 1                 |
| `SPEND`                    | `float64`     | **Amount spent (currency: British Pound £)** for the line (product × quantity)                         | 1.01              |
| `PROD_CODE`                | `object`      | Detailed product code (lowest level)                                                                   | PRD0900005        |
| `PROD_CODE_10`             | `object`      | Product group code level 1 (main category)                                                             | CL00155           |
| `PROD_CODE_20`             | `object`      | Product group code level 2                                                                             | DEP00053          |
| `PROD_CODE_30`             | `object`      | Product group code level 3                                                                             | G00016            |
| `PROD_CODE_40`             | `object`      | Product group code level 4 (may represent a sub-category)                                              | NaN / G00420      |
| `CUST_CODE`                | `object`      | Customer identifier (anonymized)                                                                       | CUST0000123       |
| `seg_1`                    | `object`      | Customer segment level 1 (high-level behavior classification)                                          | BG / AZ / NaN     |
| `seg_2`                    | `object`      | Customer segment level 2 (more detailed than `seg_1`)                                                  | DI / CZ / BU      |
| `BASKET_ID`                | `int64`       | Basket ID (each purchase event of a customer)                                                          | 994110500233340   |
| `BASKET_SIZE`              | `object`      | Basket size (Small/Medium/Large)                                                                       | S / M / L         |
| `BASKET_PRICE_SENSITIVITY` | `object`      | **Customer price sensitivity** within the transaction (Low/Medium/High or short codes like LA, MM, UM) | MM / LA / UM      |
| `BASKET_TYPE`              | `object`      | Basket type (Full Shop / Small Shop / Top Up / Fresh / Nonfood...)                                     | Full Shop         |
| `BASKET_DOMINANT_MISSION`  | `object`      | **Primary basket mission** (Mixed, Grocery, Fresh, Nonfood, ...), indicating dominant product intent   | Mixed / Fresh     |
| `STORE_CODE`               | `object`      | Store code where the transaction occurred                                                              | STORE00001        |
| `STORE_FORMAT`             | `object`      | **Store format** (LS = Large Store, SS = Small Store, Express, etc.)                                   | LS                |
| `STORE_REGION`             | `object`      | **Geographic region** of the store (E01–E05, corresponding to regions in the UK)                       | E02               |

#### 2.1.2 Feature Groups by Business Context

| **Group**   | **Columns**                                             | **Meaning**                             |
| ----------- | ------------------------------------------------------- | --------------------------------------- |
| 🛒 Basket   | `BASKET_SIZE`, `BASKET_TYPE`, `BASKET_DOMINANT_MISSION` | Basket size, type, and shopping mission |
| 💸 Spending | `SPEND`, `QUANTITY`                                     | Amount spent and quantity purchased     |
| 🏬 Store    | `STORE_REGION`, `STORE_FORMAT`                          | Store region and store type             |
| 📦 Product  | `PROD_CODE_20`, `PROD_CODE_30`                          | Main product group                      |
| 🎯 Label    | `BASKET_PRICE_SENSITIVITY`                              | Price sensitivity – Low / Medium / High |

### 2.2 Problem Objective

**Build a multi-class supervised machine learning classifier to predict customer price sensitivity (Low / Medium / High) for each transaction, based on basket features, store context, and shopping behavior.**

💡 **Use case**: Segment customers by price sensitivity → enable dynamic pricing, personalized promotions, and revenue optimization.

### 2.3 Problem Type and Models

**Type**: Multi-class classification (Supervised Learning)

**Planned models**:

| **Model**                         | **Reason**                         |
| --------------------------------- | ---------------------------------- |
| Decision Tree                     | Easy to interpret feature impact   |
| Random Forest                     | High accuracy, reduces overfitting |
| Logistic Regression (multi-class) | Baseline for comparison            |
| XGBoost                           | Strong performance on tabular data |

**Evaluation**: Accuracy, Precision, Recall, F1-score, Confusion Matrix

### 2.4 Overall Architecture (AWS MLOps)

**Core components**:

- **Amazon S3**: Store raw/silver/gold datasets, partitioned by `STORE_REGION`, `BASKET_TYPE`
- **Glue/Athena**: ETL and data exploration
- **SageMaker Feature Store**: Manage feature parity between training ↔ inference
- **SageMaker Training & Model Registry**: Model training and versioning
- **EKS (FastAPI)**: Deploy a real-time price sensitivity prediction API
- **CloudWatch**: Track latency, real-world accuracy, and costs

### 2.5 KPIs and Expected Results

| **Group** | **Metric**                | **Target** |
| --------- | ------------------------- | ---------- |
| ML        | Accuracy                  | ≥ 0.75     |
| ML        | Macro F1                  | ≥ 0.70     |
| ML        | Precision (per class)     | ≥ 0.65     |
| Ops       | P95 latency (API)         | < 200 ms   |
| Ops       | Throughput (requests/s)   | ≥ 100      |
| Business  | Reduce pricing mistakes   | ≥ 10%      |
| Cost      | Infrastructure cost/month | < $500     |

## 3. Project Structure

The project is organized in a modular structure with clear separation of concerns:

```text
retail-forecast/
├── README.md                    # Project overview & setup guide
├── .gitignore                   # Git ignore patterns
├── aws/                         # AWS-specific configurations
│   ├── .travis.yml              # Travis CI configuration
│   ├── Jenkinsfile              # Jenkins pipeline configuration
│   ├── infra/                   # Terraform infrastructure
│   │   ├── main.tf              # Main infrastructure config
│   │   ├── variables.tf         # Input variables
│   │   └── output.tf            # Output values
│   ├── k8s/                     # Kubernetes manifests
│   │   ├── deployment.yaml      # Application deployment
│   │   ├── service.yaml         # Service configuration
│   │   ├── hpa.yaml             # Horizontal Pod Autoscaler
│   │   └── namespace.yaml       # Namespace definition
│   └── script/                  # Automation scripts
│       ├── create_training_job.py    # SageMaker training job
│       ├── register_model.py         # Model registry script
│       ├── deploy_endpoint.py        # Model deployment
│       └── autoscaling_endpoint.py   # Auto-scaling setup
├── azure/                       # Azure-specific configurations
│   ├── azure-pipelines.yml      # Azure DevOps pipeline
│   ├── aml/                     # Azure ML configurations
│   │   ├── train-job.yml        # Training job definition
│   │   ├── train.Dockerfile     # Training container
│   │   └── infer.Dockerfile     # Inference container
│   ├── infra/                   # Bicep infrastructure
│   │   └── main.bicep           # Azure infrastructure
│   └── k8s/                     # AKS manifests
│       ├── deployment.yaml      # Application deployment
│       ├── service.yaml         # Service configuration
│       └── hpa.yaml             # Horizontal Pod Autoscaler
├── core/                        # Shared ML core modules
│   └── requirements.txt         # Core Python dependencies
├── server/                      # Inference API server
│   ├── DockerFile               # Container definition
│   ├── requirements.txt         # Server dependencies
│   └── Readme.md                # Server documentation
└── tests/                       # Test suites
    └── (test files)             # Unit & integration tests
```

### 3.1 Detailed Folder Structure

**📂 `aws/` - AWS Implementation**

- `infra/`: Terraform Infrastructure as Code
- `k8s/`: Kubernetes manifests for EKS deployment
- `script/`: Python scripts for SageMaker automation
- CI/CD configurations (Jenkins, Travis)

**📂 `azure/` - Azure Implementation**

- `infra/`: Bicep templates for Azure resources
- `aml/`: Azure ML configurations
- `k8s/`: AKS manifests
- Azure DevOps pipeline

**📂 `core/` - Shared Components**

- Common ML utilities and libraries
- Shared dependencies and configurations

**📂 `server/` - Inference API**

- FastAPI application
- Docker containerization
- API documentation

**📂 `tests/` - Testing Framework**

- Unit tests for the ML pipeline
- Integration tests for infrastructure
- End-to-end testing scenarios

## 4. Technologies Used

### 4.1 Infrastructure & Platform Stack

- **Infrastructure as Code**: Terraform for automated provisioning
- **Container Platform**: Amazon EKS (Kubernetes) with managed node groups
- **Container Registry**: Amazon ECR with vulnerability scanning
- **Networking**: Multi-AZ VPC, NAT gateways, security groups
- **Load Balancing**: Application Load Balancer with health checks

### 4.2 ML & Data Platform Stack

- **ML Training**: Amazon SageMaker with distributed training
- **Data Storage**: Amazon S3 data lake with versioning
- **Model Registry**: SageMaker Model Registry for version control
- **Data Processing**: Automated preprocessing and feature engineering
- **ML Framework**: TensorFlow/PyTorch on SageMaker training jobs

### 4.3 DevOps & Security Stack

- **CI/CD Platform**: Jenkins or Travis CI for automated pipelines
- **Monitoring**: CloudWatch (logs, metrics, dashboards, alarms)
- **Security**: KMS encryption, CloudTrail auditing, IAM with IRSA
- **DataOps**: S3-based data versioning and lifecycle management

## 5. Detailed MLOps Architecture

### 5.1 Phase 1: Infrastructure Foundation

**Terraform Infrastructure as Code**

- VPC with multi-AZ public/private subnets
- EKS cluster with managed node groups (auto-scaling)
- IAM roles with IRSA (IAM Roles for Service Accounts)
- Security groups using least privilege access
- ECR repositories for container images

**Network Architecture**

- Public subnets: NAT Gateway, Load Balancer
- Private subnets: EKS worker nodes, SageMaker
- VPC endpoints: S3, ECR, CloudWatch (reduce data transfer costs)

### 5.2 Phase 2: ML Training & Model Management

**SageMaker Training Pipeline**

- **Data Ingestion**: S3 data lake with automated validation
- **Distributed Training**: SageMaker training jobs with Spot Instances
- **Model Registry**: Versioned model artifacts with metadata tracking
- **Experiment Tracking**: Performance metrics and hyperparameter tuning

**Data Management Strategy**

- Raw data → Processed data → Feature store → Model artifacts
- S3 Intelligent-Tiering for cost optimization
- Data lineage tracking and version control

### 5.3 Phase 3: Containerized Inference Platform

**EKS Deployment Architecture**

- **Docker Containers**: FastAPI inference service
- **Kubernetes Deployment**: Rolling updates with zero downtime
- **Horizontal Pod Autoscaler**: Dynamic scaling based on CPU/memory
- **Service Discovery**: Internal service communication
- **Application Load Balancer**: External access with SSL termination

**Monitoring & Observability**

- **CloudWatch Logs**: Centralized logging from all components
- **Custom Metrics**: Model performance, latency, throughput
- **Alarms & Notifications**: Automated alerting when issues occur
- **Dashboards**: Real-time visualization of system health

### 5.4 Phase 4: CI/CD & Automation

**Automated Pipeline Flow**

```bash
1. Code/Data Change → Git Webhook
2. Jenkins/Travis Build → Run Tests
3. SageMaker Training → Model Validation
4. Docker Build → Push to ECR
5. Kubernetes Deploy → Rolling Update
6. Health Check → Monitor Performance
```

**DataOps Workflow**

- **Data Versioning**: S3 with metadata tracking
- **Data Quality**: Automated validation and testing
- **Feature Engineering**: Reproducible pipelines
- **Model Deployment**: A/B testing capabilities

## 6. Scope & Expected Outcomes

### 6.1 In Scope

✅ **Complete Infrastructure**: Terraform IaC for all AWS resources  
✅ **ML Training**: SageMaker distributed training with hyperparameter tuning  
✅ **Container Deployment**: EKS with autoscaling and load balancing  
✅ **Security Best Practices**: KMS encryption, CloudTrail auditing, IAM least privilege  
✅ **Monitoring & Alerting**: Comprehensive CloudWatch monitoring  
✅ **CI/CD Automation**: End-to-end pipeline from code to production  
✅ **Cost Optimization**: Auto-scaling, Spot Instances, lifecycle policies

### 6.2 Out of Scope

❌ Multi-region deployment (focus on ap-southeast-1)  
❌ Advanced ML features (A/B testing, canary deployments)  
❌ Real-time streaming inference (batch-focused)  
❌ Custom monitoring solutions (CloudWatch-only)

### 6.3 Expected Outcomes

🎯 **Production-Ready MLOps Platform**: Scalable, reliable, cost-effective  
🎯 **Automated ML Lifecycle**: From data ingestion to model deployment  
🎯 **Infrastructure Reproducibility**: Terraform state management  
🎯 **Operational Excellence**: Comprehensive monitoring and alerting  
🎯 **Cost Efficiency**: Optimized resource usage with auto-scaling

{{% notice info %}}
This architecture is designed to support enterprise-grade ML workloads, scaling from proof-of-concept to production with millions of requests/day.
{{% /notice %}}

{{% notice warning %}}
**Prerequisites**: AWS account with permissions for EKS, SageMaker, ECR, S3, CloudWatch, IAM. Terraform >= 1.0, kubectl, AWS CLI, Docker, Python 3.8+.
{{% /notice %}}
