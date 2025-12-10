---
title: "Proposal"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

In this section, you need to summarize the contents of the workshop that you **plan** to conduct.

# Retail Price Sensitivity MLOps Platform on AWS

## End-to-End MLOps Pipeline: Data Lake → Training → Model Registry → Container → EKS API → Monitoring → CI/CD → Cost Control

## 1. Executive Summary

This workshop delivers a complete, production-oriented MLOps workflow on AWS for a **Retail Prediction API** that predicts **BASKET_PRICE_SENSITIVITY (Low/Medium/High)** from curated retail features. The system uses an S3-based data lake (raw/silver/gold), an automated training pipeline on **Amazon SageMaker** (RandomForest), **Model Registry** for versioning/approval, containerization with **Amazon ECR**, and scalable API deployment on **Amazon EKS** behind a public load balancer for demo usage. Observability is implemented with **Amazon CloudWatch**, and delivery is automated with a CI/CD pipeline (GitHub Actions or Jenkins). The workshop also emphasizes **cost optimization** and a clean **teardown** strategy to avoid unexpected charges.

## 2. Problem Statement

### 2.1 What’s the Problem?

Deploying ML to production is more than training a model. Teams often struggle with:

- Data pipelines that are not repeatable (manual ETL, inconsistent splits)
- No model governance (no registry, no approval workflow, no version traceability)
- Inference code that cannot be deployed reliably (no container standardization)
- Kubernetes deployment complexity (networking, IAM permissions, image pulls)
- Missing monitoring/logging (hard to debug outages and cost spikes)
- Costs that grow quickly when infrastructure runs 24/7

### 2.2 The Solution

This workshop builds a unified MLOps system that:

- Standardizes data flow in S3: **silver/** → **gold/** with deterministic train/val/test splits
- Trains a **RandomForest classifier** on SageMaker with evaluation targets (**Accuracy ≥ 0.8**, **F1 ≥ 0.7**)
- Stores artifacts in **S3 artifacts/** and manages model versions in **SageMaker Model Registry** (register + approve)
- Packages inference API as a hardened **FastAPI** container and pushes to **ECR** with lifecycle + scan-on-push
- Deploys to **EKS** with HPA autoscaling and public endpoint for demo (**/health**, **/docs**, **/predict**)
- Adds CloudWatch monitoring (Container Insights, alarms, log retention)
- Adds CI/CD automation for build–scan–push–deploy and optional retrain+register
- Applies cost controls (Spot, schedules, lifecycle rules, teardown scripts)

### 2.3 Benefits and Return on Investment

- Faster, repeatable deployment of ML models with clear version governance
- Easier debugging and reliability through monitoring and health checks
- Lower operational costs through Spot/scheduling and lifecycle policies
- A reusable reference architecture for future ML services (not limited to retail)

## 3. Solution Architecture

### 3.1 High-Level Architecture (Textual)

**Data Layer**

- Amazon S3: `raw/`, `silver/`, `gold/`, `artifacts/`, `logs/`, `tmp/`
- Automated ETL from silver partitions → gold splits (train/val/test)

**Training & Governance**

- Amazon SageMaker Training (RandomForest)
- Evaluation + metrics logging (CloudWatch)
- Model artifacts stored in S3 `artifacts/`
- SageMaker Model Registry: Model Package Group + approve versions

**Container & Deployment**

- Amazon ECR: `mlops/retail-api` (tag immutability, scan-on-push, lifecycle rules)
- Amazon EKS: `mlops-retail-cluster` (Production VPC, private subnets for nodes/pods)
- Public entry point for demo via LoadBalancer/ALB (enabled only during demos)

**Observability & Automation**

- Amazon CloudWatch: Container Insights, log groups retention, alarms
- CI/CD: GitHub Actions (OIDC) or Jenkins pipeline for build/test/scan/push/deploy + optional retraining
- Cost Management: S3/ECR lifecycle, log retention, schedule stop/start, teardown scripts

> **Region alignment requirement:** All components are aligned to **ap-southeast-1** to avoid cross-region issues (e.g., S3 301 redirects).

### 3.2 AWS Services Used

- **Amazon S3**: data lake + model artifacts + logs
- **Amazon SageMaker**: training jobs + Model Registry
- **Amazon ECR**: private image registry for inference API
- **Amazon EKS**: production deployment platform for FastAPI
- **Elastic Load Balancing (ALB/NLB)**: public demo endpoint for `/predict` and `/docs`
- **Amazon CloudWatch**: logs, metrics, Container Insights, alarms
- **AWS IAM + IRSA**: secure access from pods to AWS (S3/SageMaker/CloudWatch)
- **Amazon VPC + VPC Endpoints**: private connectivity for ECR/S3/CloudWatch Logs (no NAT gateway)
- **CI/CD tooling**: GitHub Actions (OIDC role) or Jenkins on EC2

### 3.3 Component Design

- **ETL & Dataset Prep**: Auto-generate gold splits from silver partitions; store under S3 `gold/`
- **Model Training**: SageMaker training job writes artifacts to S3 `artifacts/`; outputs metrics
- **Model Governance**: Register model packages; only approved versions are used by inference
- **Inference Service**: FastAPI app loads approved model from registry, supports `/predict`
- **K8s Reliability**: health probes, multi-replica deployment, autoscaling via HPA
- **Security**: private subnets, endpoints, least-privilege IAM, image scanning

## 4. Technical Implementation

### 4.1 Implementation Phases (Mapped to Workshop Tasks)

- **Task 4 – Model Training Pipeline (SageMaker):** ETL silver→gold, train RF, evaluate, artifacts in S3, register/approve in Model Registry
- **Task 5 – Production Networking:** Separate Production VPC (`10.0.0.0/16`), private subnets for EKS, public subnets for ALB (demo-only), VPC endpoints, no NAT
- **Task 6 – ECR:** Hardened multi-stage FastAPI container, scan-on-push, lifecycle, push scripts
- **Task 7 – EKS Setup:** Cluster + add-ons + IRSA + nodegroup, deploy sample app and verify pulls from ECR
- **Task 8 – API Deployment:** Deploy retail API to namespace `mlops`, ServiceAccount with IRSA role, LB endpoint + HPA + tests
- **Task 9 – Load Balancing:** Improve demo endpoint via ALB/Ingress (optional AWS Load Balancer Controller), TLS/DNS (optional)
- **Task 10 – Monitoring:** Container Insights, log retention, alarms, Logs Insights queries
- **Task 11 – CI/CD:** Automate build/test/scan/push/deploy; optional retrain+register; rollback hooks
- **Task 12 – Cost & Teardown:** Spot/schedules/lifecycle/budgets and full teardown scripts

### 4.2 Technical Requirements

- AWS account access with permissions for EKS, SageMaker, ECR, S3, VPC, CloudWatch, IAM
- Docker + AWS CLI configured locally (or CloudShell)
- Kubernetes tooling (kubectl, eksctl optional), Helm (if using Load Balancer Controller)
- Repository structure aligned with workshop (e.g., `aws/scripts/`, `aws/k8s/`, `aws/infra/`)

## 5. Timeline & Milestones

- **Milestone 1:** Data pipeline + SageMaker training + Model Registry operational (model versioned & approved)
- **Milestone 2:** Containerized FastAPI image in ECR (scan/lifecycle enabled)
- **Milestone 3:** EKS cluster + IRSA working; API deployed with health checks & HPA
- **Milestone 4:** Public demo endpoint available; monitoring dashboards/alarms configured
- **Milestone 5:** CI/CD working end-to-end; cost controls validated; teardown verified

## 6. Budget Estimation

The workshop emphasizes **cost-minimized production-like setup**, including:

- EKS nodes (prefer Spot / small dev instances for demos)
- Load balancer enabled only when presenting demos
- SageMaker training optimized (managed spot training if possible)
- CloudWatch logs retention (7–30 days)
- S3/ECR lifecycle policies to reduce storage costs
- Budget alerts to prevent overruns

_(Optional: attach AWS Pricing Calculator screenshot or a simple cost table from Task 12.)_

## 7. Risk Assessment

### 7.1 Risk Matrix (Typical Workshop Risks)

- **Cost overruns**: Medium impact, medium probability
- **Networking issues (private subnets/endpoints)**: High impact, medium probability
- **IAM/IRSA misconfiguration**: High impact, medium probability
- **EKS image pull failures (ECR auth/endpoint)**: Medium impact, medium probability
- **Model mismatch / wrong approved version**: Medium impact, low probability

### 7.2 Mitigation Strategies

- Enforce region alignment and naming conventions
- Use VPC endpoints (S3 gateway, ECR API/DKR, CloudWatch Logs) to avoid NAT dependency
- Use least-privilege IAM policies and validate IRSA early with a smoke test pod
- Health checks + rollback steps in CI/CD
- Budget alerts + teardown scripts run after demo

## 8. Expected Outcomes

### 8.1 Technical Outcomes

- Fully working MLOps workflow: **S3 → SageMaker Training → Model Registry → ECR → EKS API**
- Public demo endpoint with autoscaling and health checks
- Monitoring and logs for debugging and operational visibility
- CI/CD pipeline enabling repeatable deployments

### 8.2 Long-term Value

- A reusable blueprint for deploying additional ML APIs
- Clear governance and reproducibility for model versions and artifacts
- Cost-aware operational practices suitable for student projects and production prototypes
