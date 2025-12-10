---
title: "Proposal"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Nền tảng MLOps Dự đoán Price Sensitivity trên AWS

## End-to-End MLOps: Data Lake → Training → Model Registry → Container → EKS API → Monitoring → CI/CD → Tối ưu chi phí

## 1. Tóm tắt (Executive Summary)

Workshop này xây dựng một quy trình MLOps hoàn chỉnh trên AWS cho **Retail Prediction API**, dự đoán nhãn **BASKET_PRICE_SENSITIVITY (Low/Medium/High)** từ dữ liệu bán lẻ đã được chuẩn hoá. Hệ thống sử dụng S3 làm data lake (raw/silver/gold), pipeline training trên **Amazon SageMaker** (RandomForest) có đánh giá chất lượng (**Accuracy ≥ 0.8, F1 ≥ 0.7**), quản lý phiên bản qua **SageMaker Model Registry**, đóng gói inference bằng **FastAPI** đẩy lên **Amazon ECR**, triển khai production-like trên **Amazon EKS** có autoscaling và public endpoint phục vụ demo. Quan sát hệ thống bằng **Amazon CloudWatch**, tự động hoá build–deploy bằng CI/CD (GitHub Actions hoặc Jenkins), và kết thúc bằng chiến lược **tối ưu chi phí + teardown** để không phát sinh phí ngoài ý muốn.

## 2. Bài toán (Problem Statement)

### 2.1 Vấn đề là gì?

Khi đưa ML vào production, team thường gặp các khó khăn:

- Pipeline dữ liệu không lặp lại được (ETL thủ công, chia tập không nhất quán)
- Thiếu governance cho model (không registry, không approve version, khó trace)
- Inference khó triển khai ổn định (không chuẩn hoá container)
- EKS phức tạp (networking, IAM/IRSA, image pull ECR)
- Thiếu monitoring/logging → khó debug lỗi và khó kiểm soát chi phí
- Chi phí tăng nhanh nếu tài nguyên chạy 24/7 (EKS nodes, ALB, logs, storage)

### 2.2 Giải pháp

Workshop triển khai một hệ thống thống nhất:

- Chuẩn hoá data lake trên S3: **silver/** → **gold/** (tạo train/val/test deterministically)
- Train **RandomForest classifier** trên SageMaker, đạt KPI (**Accuracy ≥ 0.8, F1 ≥ 0.7**)
- Lưu artifacts vào **S3 artifacts/** và quản lý version qua **SageMaker Model Registry** (register + approve)
- Build container inference **FastAPI** theo best-practice, push lên **ECR** (scan-on-push + lifecycle)
- Deploy API lên **EKS** (multi-replica + health check + HPA), public endpoint cho demo (**/health, /docs, /predict**)
- Thêm monitoring bằng **CloudWatch** (Container Insights, log retention, alarms)
- Thêm CI/CD tự động build–test–scan–push–deploy và (tuỳ chọn) retrain+register
- Tối ưu chi phí bằng Spot/schedule/lifecycle/budgets và teardown đầy đủ

### 2.3 Lợi ích / Giá trị

- Triển khai ML nhanh, lặp lại được, có kiểm soát phiên bản và audit trail
- Độ tin cậy cao hơn nhờ health check + autoscaling + monitoring
- Chi phí thấp hơn nhờ Spot/schedule/lifecycle và tắt/bỏ tài nguyên khi không dùng
- Kiến trúc mẫu tái sử dụng cho các ML APIs khác

## 3. Kiến trúc giải pháp (Solution Architecture)

### 3.1 Kiến trúc tổng quan (mô tả)

**Tầng dữ liệu**

- Amazon S3: `raw/`, `silver/`, `gold/`, `artifacts/`, `logs/`, `tmp/`
- ETL tự động từ silver partitions → gold splits (train/val/test)

**Tầng training & governance**

- Amazon SageMaker Training (RandomForest)
- Log metric/training qua CloudWatch
- Model artifacts lưu S3 `artifacts/`
- SageMaker Model Registry: Model Package Group + approve phiên bản

**Tầng container & triển khai**

- Amazon ECR: `mlops/retail-api` (immutability/scan/lifecycle)
- Amazon EKS: `mlops-retail-cluster` (VPC production, private subnets cho nodes/pods)
- Public demo endpoint qua LoadBalancer/ALB (bật khi demo, tắt khi không dùng)

**Tầng quan sát & tự động hoá**

- CloudWatch: Container Insights, log groups retention, alarms
- CI/CD: GitHub Actions (OIDC) hoặc Jenkins (EC2) cho build/test/scan/push/deploy + optional retrain
- Cost: S3/ECR lifecycle, log retention, schedule start/stop, teardown scripts

> **Yêu cầu bắt buộc về region:** đồng bộ toàn bộ tài nguyên ở **ap-southeast-1** để tránh lỗi/redirect và phát sinh phức tạp (ví dụ S3 301).

### 3.2 AWS services sử dụng

- **Amazon S3**: data lake + artifacts + logs
- **Amazon SageMaker**: training jobs + Model Registry
- **Amazon ECR**: private image registry cho inference API
- **Amazon EKS**: nền tảng chạy API production-like
- **Elastic Load Balancing (ALB/NLB)**: public endpoint cho demo `/predict` và `/docs`
- **Amazon CloudWatch**: logs, metrics, Container Insights, alarms
- **AWS IAM + IRSA**: cấp quyền an toàn cho pod truy cập S3/SageMaker/CloudWatch
- **Amazon VPC + VPC Endpoints**: private access đến S3/ECR/CloudWatch Logs (không cần NAT)
- **CI/CD**: GitHub Actions hoặc Jenkins

### 3.3 Thiết kế thành phần

- **ETL & Dataset Prep**: tạo gold splits từ silver partitions; lưu dưới `gold/`
- **Model Training**: training job ghi artifact vào `artifacts/` và xuất metrics
- **Model Governance**: register model package và chỉ dùng version đã approve
- **Inference Service**: FastAPI gọi model approved từ registry; endpoint `/predict`
- **Độ tin cậy**: probes, replicas, autoscaling HPA
- **Bảo mật**: private subnets, endpoints, least-privilege IAM, scan image

## 4. Triển khai kỹ thuật (Technical Implementation)

### 4.1 Các giai đoạn triển khai (mapping theo workshop tasks)

- **Task 4 – Training Pipeline (SageMaker):** ETL silver→gold, train RF, evaluate, artifacts S3, register/approve Model Registry
- **Task 5 – Production Networking:** tạo Production VPC (`10.0.0.0/16`), private subnets cho EKS, public subnets cho ALB demo-only, VPC endpoints, không NAT
- **Task 6 – ECR:** build FastAPI container multi-stage, non-root, healthcheck, scan-on-push, lifecycle, push scripts
- **Task 7 – EKS Setup:** tạo cluster + add-ons + IRSA + nodegroup, deploy app mẫu và verify ECR pull
- **Task 8 – API Deployment:** deploy retail API (namespace `mlops`), ServiceAccount gắn IRSA role, LB endpoint + HPA + test
- **Task 9 – Load Balancing:** nâng cấp demo endpoint bằng ALB/Ingress + controller (tuỳ chọn TLS/DNS)
- **Task 10 – Monitoring:** Container Insights, log retention, alarms, Logs Insights
- **Task 11 – CI/CD:** build/test/scan/push/deploy + optional retrain+register + rollback hooks
- **Task 12 – Cost & Teardown:** Spot/schedule/lifecycle/budget + teardown script và verify xóa sạch

### 4.2 Yêu cầu kỹ thuật

- AWS account có quyền EKS, SageMaker, ECR, S3, VPC, CloudWatch, IAM
- Docker + AWS CLI (hoặc CloudShell)
- kubectl (và Helm nếu dùng AWS Load Balancer Controller)
- Repo structure theo workshop: `aws/scripts/`, `aws/k8s/`, `aws/infra/`, ...

## 5. Timeline & Milestones

- **Milestone 1:** Data pipeline + SageMaker training + Model Registry hoạt động (model version có approve)
- **Milestone 2:** Container FastAPI lên ECR (scan/lifecycle bật đầy đủ)
- **Milestone 3:** EKS + IRSA ổn; API chạy có health check + HPA
- **Milestone 4:** Public demo endpoint hoạt động; monitoring/alarms xong
- **Milestone 5:** CI/CD chạy end-to-end; cost controls + teardown verify ok

## 6. Ước tính chi phí (Budget Estimation)

Workshop hướng tới mô hình production-like nhưng **tiết kiệm**:

- EKS node ưu tiên nhỏ/Spot cho demo
- Load balancer chỉ bật lúc demo
- SageMaker training tối ưu (Managed Spot Training nếu phù hợp)
- CloudWatch log retention 7–30 ngày
- S3/ECR lifecycle để giảm chi phí storage
- Budget alerts để tránh vượt ngân sách

_(Tuỳ chọn: đính kèm bảng chi phí/ảnh AWS Pricing Calculator hoặc trích bảng từ Task 12.)_

## 7. Đánh giá rủi ro (Risk Assessment)

### 7.1 Risk Matrix (rủi ro phổ biến)

- **Vượt chi phí**: ảnh hưởng vừa, xác suất vừa
- **Lỗi networking (private subnets/endpoints)**: ảnh hưởng cao, xác suất vừa
- **Sai IAM/IRSA**: ảnh hưởng cao, xác suất vừa
- **Image pull lỗi (ECR auth/endpoint)**: ảnh hưởng vừa, xác suất vừa
- **Dùng nhầm version model / chưa approve**: ảnh hưởng vừa, xác suất thấp

### 7.2 Giảm thiểu rủi ro

- Đồng bộ region và đặt naming convention rõ ràng
- Dùng VPC endpoints (S3 gateway, ECR API/DKR, CloudWatch Logs) để giảm phụ thuộc NAT
- Test IRSA sớm bằng pod smoke-test
- Health checks + rollback trong CI/CD
- Budget alerts + teardown sau demo

## 8. Kết quả kỳ vọng (Expected Outcomes)

### 8.1 Kết quả kỹ thuật

- Pipeline MLOps hoàn chỉnh: **S3 → SageMaker Training → Model Registry → ECR → EKS API**
- Public demo endpoint hoạt động với autoscaling và health checks
- Monitoring/logs đầy đủ để debug và theo dõi vận hành
- CI/CD giúp triển khai lặp lại nhanh và chuẩn

### 8.2 Giá trị dài hạn

- Blueprint tái sử dụng cho các ML services khác
- Governance rõ ràng cho model version + artifacts
- Thực hành vận hành tiết kiệm chi phí, phù hợp đồ án/thử nghiệm production
