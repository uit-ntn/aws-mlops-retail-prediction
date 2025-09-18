---
title: "AWS MLOps Retail Forecast Platform"
date: 2025-08-30T00:00:00+00:00
weight: 0
draft: false
type: "page"
pre: "<b>0. </b>"
---

<h2 style="margin: 20px 0 30px 0; font-size: 2.5rem; color: #131314ff; text-align: center;">AWS MLOps Retail Forecast Platform</h2>


<div style="background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%); color: white; padding: 30px; border-radius: 16px; margin: 30px 0; box-shadow: 0 8px 25px rgba(30, 60, 114, 0.4);">
  <h2 style="margin-top: 0; color: white; font-size: 1.8rem; display: flex; align-items: center; gap: 10px;">
    MLOps Technology Stack & Architecture
  </h2>
  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 20px; margin-top: 20px;">
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #60a5fa; font-weight: bold; margin-bottom: 8px;">�️ Infrastructure</div>
      <div>Terraform + EKS + VPC + IAM</div>
    </div>
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #34d399; font-weight: bold; margin-bottom: 8px;">🤖 ML Training</div>
      <div>SageMaker + Model Registry + S3</div>
    </div>
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #f472b6; font-weight: bold; margin-bottom: 8px;">🐳 Containers</div>
      <div>EKS + ECR + Docker + HPA</div>
    </div>
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #fbbf24; font-weight: bold; margin-bottom: 8px;">💾 Data & Storage</div>
      <div>S3 Data Lake + Versioning + Lifecycle</div>
    </div>
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #a78bfa; font-weight: bold; margin-bottom: 8px;">🚀 CI/CD & DataOps</div>
      <div>Jenkins + Travis + Pipeline Automation</div>
    </div>
    <div style="background: rgba(255,255,255,0.1); padding: 20px; border-radius: 12px; backdrop-filter: blur(10px);">
      <div style="color: #fb7185; font-weight: bold; margin-bottom: 8px;">📊 Monitoring & Security</div>
      <div>CloudWatch + KMS + CloudTrail</div>
    </div>
  </div>
</div>

<h2 style="margin: 40px 0 30px 0; font-size: 2.2rem; color: #1e293b; text-align: center;">📚 MLOps Workshop Steps</h2>

<div style="background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%); border: 2px dashed #cbd5e1; padding: 30px; border-radius: 16px; margin: 30px 0; text-align: center;">
  <div style="font-size: 3rem; margin-bottom: 15px;">🎯</div>
  <h3 style="margin: 0 0 15px 0; color: #1e293b; font-size: 1.5rem;">16 Task MLOps Hoàn Chỉnh</h3>
  <p style="color: #64748b; font-size: 16px; margin: 0 0 20px 0; max-width: 600px; margin-left: auto; margin-right: auto;">
    End-to-end MLOps pipeline từ Infrastructure as Code đến Model Deployment với Monitoring và Cost Optimization
  </p>
  
  <div style="background: rgba(255,255,255,0.8); padding: 20px; border-radius: 12px; margin: 20px 0; text-align: left; border: 1px solid #cbd5e1;">
    <h4 style="margin: 0 0 15px 0; color: #1e293b; text-align: center;">📁 Project Structure Thực Tế</h4>
    <pre style="background: #1e293b; color: #f8fafc; padding: 15px; border-radius: 8px; overflow-x: auto; font-size: 12px; line-height: 1.4;">
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
    </pre>
  </div>
  <div style="display: flex; justify-content: center; gap: 20px; flex-wrap: wrap; margin-top: 20px;">
    <div style="background: #3b82f6; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🏗️ Infrastructure</div>
    <div style="background: #10b981; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🤖 ML Training</div>
    <div style="background: #8b5cf6; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">� Deployment</div>
    <div style="background: #ef4444; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">� Monitoring</div>
    <div style="background: #06b6d4; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🚀 CI/CD</div>
    <div style="background: #f59e0b; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">� Cost Optimization</div>
  </div>
</div>
<p style="text-align: center; color: #64748b; font-size: 18px; margin-bottom: 40px;">16 bước thực hành MLOps được thiết kế theo quy trình production-ready</p>

<div style="background: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%); border: 1px solid #0284c7; border-left: 6px solid #0284c7; padding: 25px; margin: 30px 0; border-radius: 12px;">
  <h3 style="margin: 0 0 20px 0; color: #0c4a6e; display: flex; align-items: center; gap: 12px; font-size: 1.4rem;">
    <span style="background: #0284c7; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center;">📋</span>
    Quy trình Implementation theo cấu trúc thực tế
  </h3>
  
  <div style="display: grid; grid-template-columns: 1fr; gap: 15px;">
    <div style="background: rgba(255,255,255,0.8); padding: 15px; border-radius: 8px; border-left: 4px solid #3b82f6;">
      <h4 style="margin: 0 0 10px 0; color: #1e40af;">Phase 1: Infrastructure Setup (Tasks 1-5)</h4>
      <ul style="margin: 0; padding-left: 20px; color: #1e40af;">
        <li>Tạo và configure <code>aws/infra/</code> với Terraform files</li>
        <li>Setup VPC, subnets, security groups trong <code>main.tf</code></li>
        <li>Configure IAM roles và policies trong <code>variables.tf</code></li>
        <li>Deploy EKS cluster và managed node groups</li>
      </ul>
    </div>
    
    <div style="background: rgba(255,255,255,0.8); padding: 15px; border-radius: 8px; border-left: 4px solid #10b981;">
      <h4 style="margin: 0 0 10px 0; color: #047857;">Phase 2: ML Pipeline (Tasks 6-9)</h4>
      <ul style="margin: 0; padding-left: 20px; color: #047857;">
        <li>Setup ECR registry cho container images</li>
        <li>Implement <code>aws/script/create_training_job.py</code> cho SageMaker</li>
        <li>Configure <code>aws/script/register_model.py</code> cho model registry</li>
        <li>Setup S3 buckets cho data storage trong core modules</li>
      </ul>
    </div>
    
    <div style="background: rgba(255,255,255,0.8); padding: 15px; border-radius: 8px; border-left: 4px solid #8b5cf6;">
      <h4 style="margin: 0 0 10px 0; color: #6d28d9;">Phase 3: Container Deployment (Tasks 10-12)</h4>
      <ul style="margin: 0; padding-left: 20px; color: #6d28d9;">
        <li>Build inference API trong <code>server/</code> với FastAPI</li>
        <li>Create <code>server/DockerFile</code> cho containerization</li>
        <li>Deploy với <code>aws/k8s/deployment.yaml</code> và <code>service.yaml</code></li>
        <li>Configure HPA trong <code>aws/k8s/hpa.yaml</code></li>
        <li>Implement <code>aws/script/deploy_endpoint.py</code></li>
      </ul>
    </div>
    
    <div style="background: rgba(255,255,255,0.8); padding: 15px; border-radius: 8px; border-left: 4px solid #ef4444;">
      <h4 style="margin: 0 0 10px 0; color: #b91c1c;">Phase 4: CI/CD & Monitoring (Tasks 13-16)</h4>
      <ul style="margin: 0; padding-left: 20px; color: #b91c1c;">
        <li>Setup Jenkins pipeline với <code>aws/Jenkinsfile</code></li>
        <li>Configure Travis CI với <code>aws/.travis.yml</code></li>
        <li>Implement CloudWatch monitoring và alerting</li>
        <li>Setup testing framework trong <code>tests/</code></li>
        <li>Configure <code>aws/script/autoscaling_endpoint.py</code></li>
      </ul>
    </div>
  </div>
</div>

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(380px, 1fr)); gap: 25px; margin: 25px 0;">
  
  <div style="border: 2px solid #3b82f6; padding: 25px; border-radius: 16px; background: linear-gradient(135deg, #eff6ff 0%, #dbeafe 100%); box-shadow: 0 4px 15px rgba(59, 130, 246, 0.1); transition: transform 0.3s ease;">
    <h3 style="color: #1e40af; margin: 0 0 15px 0; display: flex; align-items: center; gap: 10px; font-size: 1.3rem;">
      <span style="background: #3b82f6; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center; font-size: 16px;">🏗️</span>
      Infrastructure Foundation
    </h3>
    <ul style="margin: 0; padding-left: 0; list-style: none;">
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #3b82f6;">▸</span><a href="/1-introduction/" style="color: #1e40af; text-decoration: none;">MLOps architecture overview & objectives</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #3b82f6;">▸</span><a href="/2-vpc-networking/" style="color: #1e40af; text-decoration: none;">VPC, subnets, NAT, security groups</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #3b82f6;">▸</span><a href="/3-iam-roles-irsa/" style="color: #1e40af; text-decoration: none;">IAM roles & IRSA configuration</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #3b82f6;">▸</span><a href="/4-eks-cluster/" style="color: #1e40af; text-decoration: none;">EKS control plane setup</a></li>
      <li style="margin-bottom: 0; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #3b82f6;">▸</span><a href="/5-managed-nodegroup/" style="color: #1e40af; text-decoration: none;">EC2 managed node groups</a></li>
    </ul>
  </div>
  
  <div style="border: 2px solid #10b981; padding: 25px; border-radius: 16px; background: linear-gradient(135deg, #ecfdf5 0%, #d1fae5 100%); box-shadow: 0 4px 15px rgba(16, 185, 129, 0.1); transition: transform 0.3s ease;">
    <h3 style="color: #047857; margin: 0 0 15px 0; display: flex; align-items: center; gap: 10px; font-size: 1.3rem;">
      <span style="background: #10b981; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center; font-size: 16px;">🤖</span>
      ML Training & Registry
    </h3>
    <ul style="margin: 0; padding-left: 0; list-style: none;">
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #10b981;">▸</span><a href="/6-ecr-registry/" style="color: #047857; text-decoration: none;">ECR container registry setup</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #10b981;">▸</span><a href="/7-docker-build/" style="color: #047857; text-decoration: none;">Build & push inference API</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #10b981;">▸</span><a href="/8-s3-data-storage/" style="color: #047857; text-decoration: none;">S3 data lake & model artifacts</a></li>
      <li style="margin-bottom: 0; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #10b981;">▸</span><a href="/9-sagemaker-training/" style="color: #047857; text-decoration: none;">SageMaker training & model registry</a></li>
    </ul>
  </div>
  
  <div style="border: 2px solid #8b5cf6; padding: 25px; border-radius: 16px; background: linear-gradient(135deg, #f5f3ff 0%, #ede9fe 100%); box-shadow: 0 4px 15px rgba(139, 92, 246, 0.1); transition: transform 0.3s ease;">
    <h3 style="color: #6d28d9; margin: 0 0 15px 0; display: flex; align-items: center; gap: 10px; font-size: 1.3rem;">
      <span style="background: #8b5cf6; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center; font-size: 16px;">�</span>
      Container Deployment
    </h3>
    <ul style="margin: 0; padding-left: 0; list-style: none;">
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #8b5cf6;">▸</span><a href="/10-kubernetes-deploy/" style="color: #6d28d9; text-decoration: none;">Kubernetes Deployment/Service/HPA</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #8b5cf6;">▸</span><a href="/11-load-balancer/" style="color: #6d28d9; text-decoration: none;">Application Load Balancer setup</a></li>
      <li style="margin-bottom: 0; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #8b5cf6;">▸</span><a href="/12-cloudwatch/" style="color: #6d28d9; text-decoration: none;">Monitoring, logs, and metrics</a></li>
    </ul>
  </div>
  
  <div style="border: 2px solid #ef4444; padding: 25px; border-radius: 16px; background: linear-gradient(135deg, #fef2f2 0%, #fecaca 100%); box-shadow: 0 4px 15px rgba(239, 68, 68, 0.1); transition: transform 0.3s ease;">
    <h3 style="color: #b91c1c; margin: 0 0 15px 0; display: flex; align-items: center; gap: 10px; font-size: 1.3rem;">
      <span style="background: #ef4444; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center; font-size: 16px;">�</span>
      Security & Audit
    </h3>
    <ul style="margin: 0; padding-left: 0; list-style: none;">
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #ef4444;">▸</span><a href="/13-security-audit/" style="color: #b91c1c; text-decoration: none;">KMS encryption & CloudTrail</a></li>
      <li style="margin-bottom: 8px; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #ef4444;">▸</span><a href="/14-cicd-pipeline/" style="color: #b91c1c; text-decoration: none;">Jenkins/Travis CI/CD setup</a></li>
      <li style="margin-bottom: 0; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #ef4444;">▸</span><a href="/15-dataops/" style="color: #b91c1c; text-decoration: none;">Data versioning & lifecycle</a></li>
    </ul>
  </div>
  
  <div style="border: 2px solid #06b6d4; padding: 25px; border-radius: 16px; background: linear-gradient(135deg, #ecfeff 0%, #cffafe 100%); box-shadow: 0 4px 15px rgba(6, 182, 212, 0.1); transition: transform 0.3s ease;">
    <h3 style="color: #0891b2; margin: 0 0 15px 0; display: flex; align-items: center; gap: 10px; font-size: 1.3rem;">
      <span style="background: #06b6d4; color: white; padding: 8px; border-radius: 50%; width: 36px; height: 36px; display: flex; align-items: center; justify-content: center; font-size: 16px;">�</span>
      Cost & Optimization
    </h3>
    <ul style="margin: 0; padding-left: 0; list-style: none;">
      <li style="margin-bottom: 0; padding-left: 20px; position: relative;"><span style="position: absolute; left: 0; color: #06b6d4;">▸</span><a href="/16-cost-teardown/" style="color: #0891b2; text-decoration: none;">Cost optimization & teardown</a></li>
    </ul>
  </div>
  
</div>

<div style="background: linear-gradient(135deg, #f1f5f9 0%, #e2e8f0 100%); border: 1px solid #cbd5e1; border-left: 6px solid #3b82f6; padding: 30px; margin: 40px 0; border-radius: 12px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);">
  <h3 style="margin: 0 0 20px 0; color: #1e293b; display: flex; align-items: center; gap: 12px; font-size: 1.5rem;">
    <span style="background: #3b82f6; color: white; padding: 10px; border-radius: 50%; width: 40px; height: 40px; display: flex; align-items: center; justify-content: center;">🚀</span>
    Bắt đầu MLOps Workshop
  </h3>
  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 25px;">
    <div>
      <div style="color: #64748b; font-weight: 600; margin-bottom: 8px;">📋 Prerequisites</div>
      <div style="color: #475569;">AWS Account, Terraform, kubectl, Docker</div>
    </div>
    <div>
      <div style="color: #64748b; font-weight: 600; margin-bottom: 8px;">⏱️ Thời gian</div>
      <div style="color: #475569;">20-25 giờ (MLOps end-to-end)</div>
    </div>
    <div>
      <div style="color: #64748b; font-weight: 600; margin-bottom: 8px;">📈 Level</div>
      <div style="color: #475569;">Intermediate to Advanced</div>
    </div>
  </div>
  <a href="/1-introduction/" style="background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%); color: white; padding: 16px 32px; text-decoration: none; border-radius: 8px; font-weight: 600; display: inline-flex; align-items: center; gap: 8px; box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3); transition: all 0.3s ease;">
    <span>🎯</span> Bắt đầu với Task 1: MLOps Architecture Overview
    <span style="font-size: 16px;">→</span>
  </a>
</div>

<div style="margin-top: 50px; padding: 30px; background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%); border-radius: 16px; border: 1px solid #e2e8f0;">
  <h3 style="margin: 0 0 25px 0; color: #1e293b; text-align: center; font-size: 1.6rem;">✨ Điểm nổi bật của MLOps Workshop</h3>
  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 20px;">
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">🏗️</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">Infrastructure as Code</div>
      <div style="color: #64748b; font-size: 14px;">Terraform automation cho toàn bộ AWS resources</div>
    </div>
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">🤖</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">SageMaker Training</div>
      <div style="color: #64748b; font-size: 14px;">Distributed ML training với model registry</div>
    </div>
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">�</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">EKS Deployment</div>
      <div style="color: #64748b; font-size: 14px;">Kubernetes với autoscaling</div>
    </div>
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">🔒</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">Security-first</div>
      <div style="color: #64748b; font-size: 14px;">KMS encryption + CloudTrail audit</div>
    </div>
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">🔄</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">CI/CD Pipeline</div>
      <div style="color: #64748b; font-size: 14px;">Automated build → train → deploy</div>
    </div>
    <div style="text-align: center; padding: 20px; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);">
      <div style="font-size: 2rem; margin-bottom: 10px;">💰</div>
      <div style="font-weight: 600; color: #1e293b; margin-bottom: 5px;">Cost Optimized</div>
      <div style="color: #64748b; font-size: 14px;">Auto-scaling + spot instances</div>
    </div>
  </div>
</div>

<!-- Floating scroll indicator -->
<div style="position: fixed; bottom: 30px; right: 30px; z-index: 1000;">
  <div onclick="window.scrollTo({top: 0, behavior: 'smooth'})" style="background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%); color: white; width: 60px; height: 60px; border-radius: 50%; display: flex; align-items: center; justify-content: center; cursor: pointer; box-shadow: 0 4px 15px rgba(59, 130, 246, 0.4); transition: all 0.3s ease; font-size: 24px;" onmouseover="this.style.transform='scale(1.1)'" onmouseout="this.style.transform='scale(1)'">
    ⬆️
  </div>
</div>

<!-- Smooth scroll behavior -->
<style>
  html {
    scroll-behavior: smooth;
  }
  
  /* Show scroll indicator only when scrolled */
  .scroll-top {
    opacity: 0;
    transition: opacity 0.3s ease;
  }
  
  .scroll-top.visible {
    opacity: 1;
  }
</style>

<script>
  // Show/hide scroll to top button
  window.addEventListener('scroll', function() {
    const scrollTop = document.querySelector('[onclick*="scrollTo"]');
    if (scrollTop) {
      if (window.pageYOffset > 300) {
        scrollTop.style.opacity = '1';
      } else {
        scrollTop.style.opacity = '0';
      }
    }
  });
</script>
