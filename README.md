# AWS MLOps Retail Price Sensitivity Workshop
> **End-to-End MLOps Pipeline on AWS (Terraform + EKS + SageMaker + CI/CD + Monitoring)**  
> **Use case:** *Multi-class classification to predict `BASKET_PRICE_SENSITIVITY` from retail transactions*

![AWS MLOps Architecture](static/images/01-introduction/MLOps-AWS-Architecture.png)

---

## Table of Contents
- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Dataset Overview](#dataset-overview)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [MLOps Flow](#mlops-flow)
- [In Scope / Out of Scope](#in-scope--out-of-scope)
- [Core Tasks (1–16)](#core-tasks-116)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Naming & Tagging Standards](#naming--tagging-standards)
- [Acceptance Criteria](#acceptance-criteria)
- [Monitoring & Security](#monitoring--security)
- [CI/CD & DataOps](#cicd--dataops)
- [Cost Optimization](#cost-optimization)
- [FAQ](#faq)

---

## Overview
This workshop demonstrates a production-grade MLOps workflow on AWS for **price sensitivity classification** in retail.  
Key outcomes:
- **Infrastructure as Code**: **Terraform** for automated provisioning  
- **Training & Registry**: **SageMaker** for model training and model registry  
- **Container Platform**: **EKS** for scalable real-time inference API  
- **Artifact & Data Management**: **ECR** (images), **S3** (data lake & artifacts)  
- **Monitoring & Security**: **CloudWatch**, **KMS**, **CloudTrail**, **IAM/IRSA**  
- **CI/CD & DataOps**: automated pipelines for data validation, training, deploy, and monitoring

---

## Problem Statement
**Goal:** Build a **multi-class classifier** to predict **`BASKET_PRICE_SENSITIVITY`** (e.g., Low/Medium/High or LA/MM/UM) for each transaction, using basket-level, store-level, and behavioral features.

**Business benefits:**
- Identify customer groups with high price sensitivity → flexible pricing
- Personalize promotions by store format/region and basket mission
- Improve campaign ROI and protect margins

**Modeling targets & metrics:**
- **Target:** `BASKET_PRICE_SENSITIVITY`  
- **Models:** Logistic Regression (baseline), Decision Tree, Random Forest  
- **Metrics:** Accuracy, Precision/Recall/F1 (macro), Confusion Matrix  

---

## Dataset Overview
**Source:** dunnhumby *Source Files* (sample used: `transactions_200807.csv`, ~2.67M rows, 22 columns).  
**Granularity:** one row per product-line within a customer basket.  

**Key columns**
- **Time:** `SHOP_WEEK`, `SHOP_DATE`, `SHOP_WEEKDAY`, `SHOP_HOUR`  
- **Spend/Qty:** `SPEND`, `QUANTITY`  
- **Product hierarchy:** `PROD_CODE`, `PROD_CODE_10/20/30/40`  
- **Customer segments:** `CUST_CODE`, `seg_1`, `seg_2`  
- **Basket:** `BASKET_ID`, `BASKET_SIZE`, `BASKET_TYPE`, `BASKET_DOMINANT_MISSION`, **`BASKET_PRICE_SENSITIVITY` (label)**  
- **Store:** `STORE_CODE`, `STORE_FORMAT`, `STORE_REGION`  

> See `content/1-introduction/` for **Data Dictionary** and examples.

---

## Architecture Overview

### Phases
1) **Infrastructure & Foundation**
   - Terraform IaC → VPC, subnets, security groups, IAM roles with IRSA  
   - EKS Cluster + Managed Node Groups (separate groups for API vs. batch)  
   - ECR for container images; S3 data lake for raw/silver/gold and artifacts

2) **ML Training & Model Registry**
   - SageMaker training jobs (sklearn/xgboost) with S3 input/output  
   - Model Registry with metrics & lineage; approval & promotion workflow  
   - Feature management via **SageMaker Feature Store** (offline/online)

3) **Deployment & Operations**
   - Inference API (FastAPI) on EKS + ALB; HPA for autoscaling  
   - Monitoring with CloudWatch (latency, throughput, errors, model metrics)  
   - Security with KMS encryption, CloudTrail auditing, least-privilege IAM

### Data Lake Layout (S3)
```
s3://mlops-{env}-retail/
└── retail/
    ├── raw/        # original CSV files (partitioned by SHOP_WEEK)
    ├── silver/     # cleaned, typed, de-duplicated (Parquet, partitioned)
    ├── gold/       # model features: STORE×PROD×WEEK/HOUR or basket-level
    └── artifacts/  # trained models, reports, validation outputs
```

---

## Project Structure

```
aws-mlops-retail-price-sensitivity/
├── README.md
├── config.toml
├── content/
│   ├── _index.md
│   ├── 1-introduction/           # Problem, dataset, data dictionary
│   ├── 2-vpc-networking/
│   ├── 3-iam-roles/
│   ├── 4-eks-cluster/
│   ├── 5-managed-nodegroup/
│   ├── 6-ecr-registry/
│   ├── 7-docker-build/
│   ├── 8-s3-data-storage/
│   ├── 9-sagemaker-training/     # Classification & Model Registry
│   ├── 10-kubernetes-deploy/     # FastAPI /predict endpoint
│   ├── 11-load-balancer/
│   ├── 12-cloudwatch/
│   ├── 13-security-audit/
│   ├── 14-cicd-pipeline/
│   └── 15-cost-teardown/
├── aws/
│   ├── infra/                    # Terraform modules
│   ├── k8s/                      # Manifests for Deployment/Service/HPA
│   └── script/                   # Automation scripts
├── server/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/                      # FastAPI app (POST /predict)
├── core/
│   └── requirements.txt          # ML dependencies (sklearn, xgboost, pandas)
└── static/
    └── images/
```

---

## MLOps Flow

### 1) Infrastructure Provisioning (Terraform)
```bash
terraform init && terraform plan && terraform apply
# Provisions: VPC → EKS → IAM → ECR → S3
```

### 2) Data Validation & Feature Engineering
```text
Ingestion (S3 raw) → Great Expectations checks → silver (typed/clean)
→ feature building → gold (Parquet) → register to Feature Store
```

### 3) Model Training & Registry (SageMaker)
```text
S3 gold → SageMaker Training (sklearn/xgboost) → Evaluate (acc, F1, confusion)
→ Register to Model Registry → approve/push tag (staging/prod)
```

### 4) Containerized Deployment (EKS)
```bash
# Build and push
docker build -t $ECR_URI/mlops/inference-api:$GIT_SHA ./server
docker push $ECR_URI/mlops/inference-api:$GIT_SHA

# Deploy
kubectl apply -f aws/k8s/deployment.yaml
kubectl apply -f aws/k8s/service.yaml
kubectl apply -f aws/k8s/hpa.yaml
```

### 5) Monitoring & Operations
```text
CloudWatch Metrics & Logs → Dashboards (latency P50/P95, error rate, QPS)
S3 & API throughput benchmarks (baseline vs optimized)
Alarms → SNS notifications
```

### 6) CI/CD Automation
```text
Code push → CI (build/test) → Data validation (GE) → Train (SageMaker)
→ Register → Deploy to EKS → Post-deploy checks → Monitor
```

---

## In Scope / Out of Scope
**In scope**
- Multi-class classification for `BASKET_PRICE_SENSITIVITY`
- 15-task MLOps workflow (infra, training, deployment, monitoring, CI/CD)
- SageMaker Feature Store; Model Registry; EKS real-time API
- S3 data versioning, lifecycle, and performance benchmarking

**Out of scope**
- Multi-region active-active (single region focus: `ap-southeast-1`)
- Advanced rollouts (A/B, canary) beyond standard rolling updates
- Real-time streaming ingestion (batch-oriented ingestion)

---

## Core Tasks (1–15)

**1. Introduction:** Introduce the problem (predicting price sensitivity), present the dunnhumby dataset and data dictionary, describe AWS MLOps architecture, set KPIs (accuracy ≥75%, F1 ≥0.7, latency <200ms).

**2. VPC / Networking:** Create VPC, subnets, security groups, VPC Endpoint for S3, enable multi-AZ for high availability.

**3. IAM Roles & CloudTrail Audit:** Create IAM roles for ETL, SageMaker training, and EKS inference. Enable CloudTrail to audit and monitor user actions and API activity for all IAM roles, ensuring traceability and compliance.

**4. EKS Cluster:** Initialize control plane, create and connect IRSA, deploy cluster for inference API and DataOps jobs. 

**5. Managed Node Group:** Create two node groups (on-demand for API, spot for ETL/training), enable auto scaling and multi-AZ.

**6. ECR Registry:** Create repositories for training and API images, enable image scanning for security.

**7. Docker Build & Push:** Build images for FastAPI and sklearn/xgboost, use multi-stage Dockerfile, healthcheck, tag by git commit.

**8. S3 Data Storage:** Design a medallion layout (raw/silver/gold) for the data lake. Clean and standardize data, store in Parquet format (snappy compression), and partition by `SHOP_WEEK` and `STORE_REGION`. Present the theory behind storage formats and optimization for read/write performance (S3 Transfer Acceleration, multipart upload, data compression).

**9. SageMaker Training:** Train multi-class models (sklearn/xgboost), perform EDA, compare metrics, visualize results, save to Model Registry.

**10. Kubernetes Deployment:** Deploy FastAPI on EKS, load model from S3, build web client, test API, describe architecture flow.

**11. Load Balancer:** Configure ALB/Ingress, route endpoints, monitor health checks, integrate HTTPS/TLS.

**12. CloudWatch Monitoring:** Create dashboard for API latency, S3 throughput, training duration, compare performance, set alerts.

**13. Security:** Enable KMS encryption.

**14. CI/CD Pipeline:** Automate build, validation, training, registration, deployment; auto-approve/rollback based on metrics.

**15. Cost & Teardown:** Use Spot Instances, set S3 lifecycle policies, shut down resources off-hours, compare costs, automate teardown.

---

## Technology Stack

**Infrastructure & Platform**
- Terraform, Amazon EKS (Kubernetes), Amazon ECR, Amazon S3, Application Load Balancer

**ML & Data Platform**
- Amazon SageMaker (Training, Model Registry, Feature Store)
- Great Expectations (data validation), AWS Glue/Athena (optional ETL & discovery)

**Monitoring & Security**
- Amazon CloudWatch (logs, metrics, dashboards, alarms)
- AWS KMS, AWS CloudTrail, IAM with IRSA, AWS Secrets Manager

**CI/CD & Automation**
- GitHub Actions / Jenkins / GitLab CI (any CI supported)
- Docker multi-stage builds
- Kubernetes rolling updates; HPA

---

## Prerequisites
- AWS permissions for EKS, SageMaker, ECR, S3, CloudWatch, IAM
- Terraform >= 1.0, kubectl, AWS CLI, Docker
- Python 3.8+ for ML code
- **Region:** `ap-southeast-1`

---

## Naming & Tagging Standards
**Naming**
- `mlops-{env}-{component}` (e.g., `mlops-prod-eks-cluster`)  
- S3: `mlops-{env}-{purpose}-{suffix}`  
- ECR: `mlops/{service}` (e.g., `mlops/inference-api`)

**Tags**
`Project=PriceSensitivityMLOps`, `Environment=dev|stg|prod`, `Component=infra|ml|app`, `Owner=DataTeam`, `CostCenter=ML-Platform`

---

## Acceptance Criteria
- **Infra** provisioned by Terraform; state managed
- **ML Training** completes with metrics (accuracy, F1) logged and **model registered**
- **Deployment**: `/predict` reachable via ALB; **P95 latency < 200 ms**
- **Scaling**: HPA scales from 2 to 10 replicas based on CPU/QPS
- **Security**: S3/EBS/KMS encryption; IRSA; CloudTrail enabled
- **Monitoring**: Dashboards + alarms for model & system KPIs
- **CI/CD**: Code push triggers **validate → train → register → deploy**
- **Cost**: Spot for batch; lifecycle policies; off-hours scale-down

---

## Monitoring & Security
- **Logs**: EKS pods, SageMaker jobs, infra logs centralized in CloudWatch
- **Metrics**: P50/P95 latency, error rate, QPS, model accuracy trend
- **Alarms**: Threshold breaches trigger SNS notifications
- **Security**: KMS encryption, IAM least-privilege, VPC-only endpoints
- **Audit**: CloudTrail for all API activity; ECR image scan results gated in CI

---

## CI/CD & DataOps
- **Stages**: Code → Build → Test → Data Validate (GE) → Train → Register → Deploy → Monitor
- **Model Registry gates**: automatic promotion to `staging/prod` on metric thresholds
- **Rollback**: On GE failure or prod accuracy degradation
- **Data versioning**: S3 object versioning + Glue Catalog schemas

---

## Cost Optimization
- **EC2 Spot** for batch training; right-size nodes for inference
- **S3 Lifecycle**: Intelligent-Tiering for `raw/`, Glacier for logs & old artifacts
- **Auto Scaling**: Node groups & HPA scale on demand
- **Scheduling**: Shut down dev during off-hours
- **Cost visibility**: Tags + Cost Explorer + CloudWatch billing alarms

**Estimated monthly costs** *(approximate, can be reduced for student use)*  
- Dev: ~$50–150 (minimal resources, auto-shutdown enabled)  
- Prod: ~$300–600 (scale up only if needed)

---

## FAQ
**Q. Why EKS for inference?**  
Kubernetes ecosystem, portability, and strong support for ML serving patterns (FastAPI/KServe/Triton).

**Q. Can I use a different CI/CD tool?**  
Yes. GitHub Actions/GitLab/Jenkins are supported as long as they can build/push to ECR and deploy to EKS.

**Q. How do we detect drift or degradation?**  
CloudWatch custom metrics track accuracy and latency; alarms trigger rollback/promotion logic through CI/CD gates.

**Q. Batch vs real-time?**  
EKS handles real-time; SageMaker Batch Transform can be added for large offline scoring.

**Q. Data privacy & compliance?**  
KMS encryption, private subnets/VPC endpoints, CloudTrail audit, IAM least privilege.
