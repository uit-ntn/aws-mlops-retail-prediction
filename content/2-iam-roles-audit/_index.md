---
title: "IAM Roles & Audit"
date: 2025-08-30T12:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

## 🎯 Mục tiêu MLOps Retail Prediction - Task 2

Thiết lập **phân quyền truy cập (IAM)** cho toàn bộ dịch vụ AWS trong pipeline và **bật CloudTrail** để giám sát, ghi lại mọi hoạt động trên tài khoản AWS.

→ **Đảm bảo bảo mật, kiểm soát truy cập, và minh chứng hoạt động nhóm.**

📥 **Input**
- AWS Account với quyền admin
- Convention đặt tên dự án: `mlops-retail-prediction-dev`
- Vùng mục tiêu: `ap-southeast-1`
- CloudTrail đa vùng được bật


✅ **Output**
- Các dịch vụ AWS có quyền **Least Privilege** phù hợp vai trò
- Toàn bộ thao tác đều được **CloudTrail ghi lại**
- Đáp ứng tiêu chí rubric: **bảo mật, phân quyền, quản lý dự án trên cloud**

💰 **Chi phí ước tính**
≈ **0.05 USD/tháng** (CloudTrail + lưu trữ S3 cho logs)

📌 **Các bước chính**
1. **CloudTrail Setup** - Thiết lập audit logging đa vùng
2. **S3 CloudTrail Bucket** - Lưu trữ log tập trung
3. **EKS Cluster Service Role** - Role cho control plane
4. **EKS Node Group Role** - Role cho worker node
5. **SageMaker Execution Role** - Role cho training & deploy
6. **IRSA Foundation** - Chuẩn bị quyền ở mức Pod

✅ **Deliverables**
- CloudTrail multi-region trail với logging vào S3
- EKS Cluster Service Role (Console)
- EKS Node Group Role với quyền ECR/S3/CloudWatch
- SageMaker Execution Role với quyền S3 cần thiết
- Nền tảng an toàn cho thiết lập IRSA

📊 **Acceptance Criteria**
- CloudTrail ghi lại tất cả API calls và hoạt động người dùng
- EKS cluster có thể tạo với các service role phù hợp
- SageMaker training jobs có quyền đọc/ghi S3
- Node groups có thể pull image từ ECR
- Tất cả IAM role tuân thủ principle of least privilege
- Audit trail sẵn sàng cho compliance và monitoring

## 1. CloudTrail Setup - Nền tảng Audit

### 1.1. Tạo S3 Bucket cho CloudTrail

**Đi tới S3 Console:**
AWS Console → S3 → "Create bucket"

**Cấu hình Bucket:**
```
Bucket name: mlops-cloudtrail-logs-ap-southeast-1
Region: ap-southeast-1
Block all public access: ✅ Enabled
Versioning: ✅ Enabled  
Server-side encryption: ✅ SSE-S3
```

**Cấu hình Lifecycle Policy:**

**Bước 1**. S3 Console → chọn bucket `mlops-cloudtrail-logs-ap-southeast-1` → Management → Create lifecycle rule.  

**Bước 2**. Đặt tên (ví dụ `CloudTrailLogLifecycle`), Apply to all objects hoặc dùng Prefix `mlops-logs/`.  

**Bước 3**. Chọn actions (Current versions):
   - After 30 days → STANDARD_IA
   - After 90 days → GLACIER / GLACIER_IR (tùy chọn)
   - After 365 days → DEEP_ARCHIVE

**Bước 4**. (Tùy chọn) Thiết lập Transition cho noncurrent versions tương tự; hoặc Expire current versions theo retention (ví dụ 7 năm) nếu cần compliance.  

**Bước 5**. Review → Create rule → kiểm tra rule đã Active trong tab Management.  
![CloudTrail and S3 lifecycle diagram](/images/2-iam-roles-audit/01-cloudtrail-s3-lifecycle-01.png "CloudTrail multi-region trail -> S3 bucket -> Lifecycle transitions")
Lưu ý ngắn: bật Versioning nếu chuyển noncurrent versions; giữ encryption và block public access cho bucket.

**Đi tới CloudTrail Console:**
AWS Console → CloudTrail → "Create trail"

### 1.2 Cấu hình Trail

```
Trail name: mlops-retail-prediction-audit-trail
Apply trail to all regions: ✅ Yes
Management events: ✅ Read/Write
Data events: ✅ S3 bucket data events
Insights events: ✅ Enabled (phát hiện pattern bất thường)

S3 bucket: mlops-cloudtrail-logs-ap-southeast-1
Log file prefix: mlops-logs/
```

**Tích hợp CloudWatch Logs:**
```
CloudWatch Logs: ✅ Enabled
Log group: mlops-cloudtrail-log-group
IAM Role: CloudTrail_CloudWatchLogs_Role (auto-created)
```

⚠️ **Lưu ý**
- CloudTrail phải được bật trước khi tạo các resource khác để ghi nhận mọi hoạt động
- EKS Cluster role cần AmazonEKSClusterPolicy
- Node Group role cần AmazonEKSWorkerNodePolicy + AmazonEKS_CNI_Policy  
- SageMaker role cần quyền S3 phù hợp cho dữ liệu & model
- CloudTrail S3 bucket cần lifecycle policies để quản lý chi phí
- Tên role phải theo convention: `mlops-retail-prediction-dev-*`

## 2. Thiết lập IAM Roles - Quyền cho dịch vụ

### 2.1. Đi tới IAM Console
AWS Console → IAM → Roles → "Create role"

### 2.2. EKS Cluster Service Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EKS - Cluster
   ```
2. **Gán Policy:**
   ```
   Policy: AmazonEKSClusterPolicy
   ```
3. **Chi tiết Role:**
   ```
   Role name: mlops-retail-prediction-dev-eks-cluster-role
   Description: EKS cluster service role for retail prediction MLOps platform
   ```
   ![EKS Cluster Service Role - Trust relationship and policies](/images/2-iam-roles-audit/02-eks-cluster-role-trust-policies.png "EKS Cluster Service Role")

### 2.3. EKS Node Group Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: EC2
   ```
2. **Gán Policies:**
   ```
   ✅ AmazonEKSWorkerNodePolicy
   ✅ AmazonEKS_CNI_Policy
   ✅ AmazonEC2ContainerRegistryReadOnly
   ✅ CloudWatchAgentServerPolicy
   ```
3. **Chi tiết Role:**
   ```
   Role name: mlops-retail-prediction-dev-eks-nodegroup-role
   Description: EKS node group role with ECR, S3, and CloudWatch access for retail prediction
   ```
   ![EKS Node Group Role - Trust relationship and attached policies](/images/2-iam-roles-audit/03-eks-nodegroup-role-policies.png "EKS Node Group Role - Trust relationship and attached policies")

### 2.4. SageMaker Execution Role

1. **Trusted Entity Type:**
   ```
   AWS service
   Service: SageMaker
   ```
2. **Gán Policies:**
   ```
   ✅ AmazonSageMakerFullAccess
   ✅ AmazonS3FullAccess (cho lưu trữ dữ liệu và model)
   ```
3. **Chi tiết Role:**
   ```
   Role name: mlops-retail-prediction-dev-sagemaker-execution
   Description: SageMaker execution role for retail prediction training jobs and model deployment
   ```
   ![SageMaker Execution Role - Trust relationship and attached policies](/images/2-iam-roles-audit/04-sagemaker-execution-role.png "SageMaker execution role trust relationship and attached policies")

## 3. Xác thực & Kiểm tra an ninh

### 3.1. Xác minh CloudTrail

**Kiểm tra trạng thái CloudTrail:**
AWS Console → CloudTrail → Trails
```
✅ mlops-retail-prediction-audit-trail: Active
✅ Multi-region trail: Enabled
✅ Management events: Read/Write
✅ Data events: S3 configured
✅ CloudWatch Logs: Integrated
```

**Xác minh S3 Logging:**
AWS Console → S3 → mlops-cloudtrail-logs-ap-southeast-1
```
✅ Log files được tạo: /mlops-logs/AWSLogs/[account-id]/CloudTrail/
✅ Encryption: SSE-S3 enabled
✅ Lifecycle policy: Applied
✅ Access logging: Configured
```

### 3.2. Tổng hợp IAM Roles

**Đi tới IAM → Roles và kiểm tra:**
```
✅ mlops-retail-prediction-dev-eks-cluster-role
✅ mlops-retail-prediction-dev-eks-nodegroup-role  
✅ mlops-retail-prediction-dev-sagemaker-execution
✅ CloudTrail_CloudWatchLogs_Role (auto-created)
```

**Kiểm tra Trust Relationships:**
- Vào từng role → tab Trust relationships
- Xác minh các trusted entities đúng:
  - `eks.amazonaws.com` (EKS cluster role)
  - `ec2.amazonaws.com` (EKS node group role)
  - `sagemaker.amazonaws.com` (SageMaker execution role)
  - `cloudtrail.amazonaws.com` (CloudTrail logging role)

### 3.3. Kiểm tra bảo mật

**Kiểm tra CloudTrail Logging:**
1. Thực hiện một API call thử nghiệm (ví dụ list S3 buckets)
2. Kiểm tra CloudTrail logs trong 5-10 phút
3. Xác nhận event xuất hiện trong CloudWatch Logs

**Kiểm tra quyền IAM:**
```bash
# Test SageMaker role có thể assume và truy cập S3
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/mlops-retail-prediction-dev-sagemaker-execution --role-session-name test
```
![Kiểm tra assume-role SageMaker và truy cập S3](/images/2-iam-roles-audit/05-sagemaker-assume-role-test.png "Assume-role test và kiểm tra truy cập S3")

``` bash
# Test EKS roles sẵn sàng cho việc tạo cluster
aws eks describe-cluster --name test-cluster --region ap-southeast-1
```

![Ví dụ CloudTrail event được ghi lại trong CloudWatch / S3](/images/2-iam-roles-audit/06-cloudtrail-verify-event.png "CloudTrail event sample")


## 4. Tối ưu chi phí & Tuân thủ
### 4.1. Quản lý chi phí CloudTrail — Bảng so sánh

| Hạng mục | Đơn giá | Ghi chú / Assumptions | Ví dụ ước tính |
|---|---:|---|---:|
| S3 — Standard | $0.023 / GB‑month | Hot logs (ngày 0–30) | 1 GB → $0.023 |
| S3 — Standard‑IA | $0.0125 / GB‑month | Sau 30 ngày (truy cập ít) | 1 GB → $0.0125 |
| S3 — Glacier | $0.004 / GB‑month | Lưu trữ dài hạn (90–365 ngày) | 1 GB → $0.004 |
| S3 — Deep Archive | $0.00099 / GB‑month | Retention >365 ngày | 1 GB → $0.00099 |
| CloudTrail — Management events | Miễn phí (bản sao đầu tiên) | Management API calls | — |
| CloudTrail — Data events | $0.10 / 100,000 events | S3 object-level, Lambda, v.v. | 100k events → $0.10 |
| CloudTrail — Insights | $0.35 / 100,000 events | Tùy chọn phát hiện bất thường | 100k events → $0.35 |

Tình huống mẫu (ước tính hàng tháng)
- Minimal (ví dụ project nhỏ): 0.5 GB lưu trữ (chủ yếu Standard‑IA) + 10k data events  
   → S3 ≈ 0.5 * $0.0125 = $0.0063 ; Data events ≈ (10k/100k)*$0.10 = $0.01  
   → Tổng ≈ $0.016 → khớp khoảng "≈ $0.01–$0.02"
- Typical (logs tăng, vài chục GB, vài chục nghìn events): 5 GB pha trộn các lớp + 50k data events  
   → S3 (mix) ≈ $0.02–$0.04 ; Data events ≈ $0.05 ; Insights (tuỳ dùng) có thể thêm $0.00–$0.35  
   → Tổng ~ $0.02–$0.05 (thường thấy cho dự án nhỏ)

Ghi chú ngắn:
- Lifecycle chuyển objects sang IA/Glacier/Deep Archive là chìa khoá giảm chi phí dài hạn.
- Data events và Insights tăng theo số events — tối ưu sampling / chỉ log cần thiết để tiết kiệm.
- Kiểm tra thực tế bằng billing/Cost Explorer để hiệu chỉnh các giả định trên.  

## 👉 Kết quả Task 2

✅ **CloudTrail Multi-Region** - Bản ghi kiểm toán toàn diện cho tất cả hoạt động AWS  
✅ **Lưu trữ Audit S3** - Lưu giữ log tối ưu chi phí với lifecycle policies  
✅ **Role Bảo mật EKS** - Quyền cho cluster và node group đã sẵn sàng  
✅ **Role Thực thi SageMaker** - Training jobs có quyền truy cập S3 phù hợp  
✅ **Nền tảng Bảo mật** - Kiến trúc least-privilege chuẩn doanh nghiệp  
✅ **Sẵn sàng Tuân thủ** - Audit trail phù hợp với yêu cầu pháp lý  

**💰 Chi phí hàng tháng**: ~0.05 USD (CloudTrail + lưu trữ S3)  
**🔍 Phủ sóng kiểm toán**: 100% các API call và hoạt động người dùng  
**🛡️ Tư thế bảo mật**: Quyền truy cập theo nguyên tắc least-privilege, sẵn sàng cho production

{{% notice tip %}}
**🚀 Bước tiếp theo:** 
- **Task 3**: Thiết lập S3 data lake với tích hợp bảo mật
- **Task 4**: VPC networking với security groups
- **Task 5**: Triển khai EKS cluster với IAM roles đã cấu hình
- **Task 6**: Thiết lập IRSA cho quyền ở mức Pod
{{% /notice %}}

{{% notice warning %}}
**🔐 Lưu ý bảo mật**: 
- CloudTrail logs chứa thông tin nhạy cảm - đảm bảo bảo mật bucket S3
- Tên role sẽ được sử dụng chính xác trong các task tiếp theo
- IRSA yêu cầu OIDC provider cho EKS (Task 5)
- Giám sát chi phí CloudTrail bằng AWS Cost Explorer
- Rà soát logs định kỳ để phát hiện hoạt động bất thường
{{% /notice %}}
