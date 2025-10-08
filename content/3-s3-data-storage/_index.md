---
title: "S3 Data Storage"
date: 2024-01-01T00:00:00Z
weight: 8
chapter: false
pre: "<b>8. </b>"
---

## 🎯 Mục tiêu

Thiết lập Amazon S3 để lưu trữ dữ liệu huấn luyện và model artifacts (đầu ra sau training). Đây là kho dữ liệu trung tâm cho pipeline ML.

{{% notice info %}}
Console đủ để triển khai S3 cho môi trường dev/prod cơ bản. IaC (Terraform) chỉ cần khi bạn muốn automation và reproducibility.
{{% /notice %}}

## 📥 Input

- AWS Account với quyền S3/IAM/CloudTrail
- Tên bucket duy nhất toàn cầu (data, artifacts)
- SageMaker Execution Role (sẽ gắn policy S3)

## 📌 Các bước chính

1) Tạo 2 S3 buckets (data, artifacts) qua Console
2) Bật Versioning, thiết lập Lifecycle, xác nhận Block Public Access
3) Tạo IAM policy giới hạn theo bucket và gắn vào SageMaker role
4) Upload training/validation data qua Console
5) Chạy SageMaker training job, xuất artifact về bucket `artifacts`
6) Bật CloudTrail data events và (tuỳ chọn) Server Access Logging
7) Xác thực cấu hình và quy trình upload/download

## 1. S3 Buckets via Console

Chúng ta sẽ tạo 2 bucket chính:

- **Data bucket**: Lưu dữ liệu huấn luyện (ví dụ train.csv)
- **Artifact bucket**: Lưu trữ model artifact sinh ra từ SageMaker training job

Thao tác trên AWS Console:

1) Vào AWS Console → S3 → Create bucket

   ![Create bucket](images/s3/ui-create-bucket.png)

2) Đặt tên:
   - Gợi ý `retail-forecast-data-<suffix>` và `retail-forecast-artifacts-<suffix>` (đảm bảo duy nhất toàn cầu)
   - Suffix có thể là accountId, timestamp, hay tên đội (vd: `retail-forecast-data-123456789012`)

   ![Bucket names](images/s3/ui-bucket-names.png)

3) Region: chọn đúng Region bạn sẽ chạy SageMaker (vd: us-east-1) để giảm chi phí cross-region.

   ![Select region](images/s3/ui-select-region.png)

4) Object Ownership: để mặc định (Bucket owner enforced). Block Public Access: bật cả 4 mục.

   ![Block public access](images/s3/ui-block-public-access.png)

5) Encryption: có thể để SSE-S3 mặc định; nếu có KMS key nội bộ, chọn SSE-KMS và chỉ định CMK.

   ![Default encryption](images/s3/ui-default-encryption.png)

6) Nhấn Create bucket. Lặp lại tương tự cho bucket artifacts.

   ![Bucket list](images/s3/ui-bucket-list.png)

Lưu ý
- Nên thống nhất convention: `retail-forecast-data-<env>-<suffix>` và `retail-forecast-artifacts-<env>-<suffix>` (vd: `-dev-`/`-prod-`).
- Tránh dùng ký tự hoa hoặc khoảng trắng; tên bucket là lowercase và không có underscore.

## 2. Cấu hình Bucket Properties

### 3. Cấu hình Bucket Properties (UI)

#### 3.1 Bật Versioning

Thao tác:

1) Mở bucket `retail-forecast-data-<suffix>` → tab Properties → Object Versioning → Edit → Enable → Save

   ![Enable versioning](images/s3/ui-enable-versioning.png)

2) Lặp lại cho bucket `retail-forecast-artifacts-<suffix>`

Gợi ý
- Bật Versioning giúp rollback file dữ liệu và artifact khi có lỗi cập nhật.

#### 3.2 Thiết lập Lifecycle Rules

Thao tác:

1) Mở bucket → tab Management → Lifecycle rules → Create lifecycle rule
2) Tên rule: "DataLifecycleRule" → Scope: Prefix = `training-data/`
3) Transition: After 30 days → STANDARD_IA; After 90 days → GLACIER Flexible Retrieval (tùy nhu cầu)
4) Save
5) Tạo rule thứ hai cho bucket artifacts: tên "ModelArtifactLifecycleRule" → Prefix = `models/` → Transition after 60 days → STANDARD_IA → Save

   ![Lifecycle rules](images/s3/ui-lifecycle-rules.png)

Mẹo tối ưu chi phí
- Với dữ liệu ít truy cập lại, cân nhắc GLACIER Deep Archive sau 180–365 ngày.
- Không áp dụng transition cho các tiền tố cần truy cập thường xuyên.

#### 3.3 Xác nhận Block Public Access

Thao tác:

1) Vào bucket → tab Permissions → Block public access (bucket settings) → Edit
2) Đảm bảo cả 4 tùy chọn đều bật → Save

   ![Confirm BPA](images/s3/ui-confirm-bpa.png)

## 3. Cấu hình IAM Permissions

#### 4.1 Tạo IAM Policy giới hạn theo bucket

Thao tác:

1) AWS Console → IAM → Policies → Create policy
2) Visual editor → Service: S3
3) Actions: `ListBucket`, `GetObject`, `PutObject`, `DeleteObject`
4) Resources:
   - Bucket: chọn 2 bucket `retail-forecast-data-<suffix>`, `retail-forecast-artifacts-<suffix>`
   - Object: chọn All objects cho cả 2 bucket
5) Next → Đặt tên: `RetailForecastS3AccessPolicy` → Create policy

   ![IAM create policy](images/iam/ui-create-s3-policy.png)

#### 4.2 Gắn Policy vào SageMaker Execution Role

Thao tác:

1) IAM → Roles → tìm role SageMaker execution (ví dụ `AmazonSageMaker-ExecutionRole-...`)
2) Attach policies → chọn `RetailForecastS3AccessPolicy` → Add permissions

   ![Attach policy to role](images/iam/ui-attach-policy-role.png)

## 4. Tích hợp với SageMaker

#### 5.1 Upload Training/Validation Data (UI)

Thao tác:

1) Mở bucket `retail-forecast-data-<suffix>` → Create folder `training-data/`
2) Mở folder `training-data/` → Upload → kéo thả `train.csv`, `validation.csv` → Upload

   ![Upload data](images/s3/ui-upload-training-data.png)

Khuyến nghị cấu trúc thư mục
- `training-data/`, `validation-data/`, `test-data/`
- `models/` (trên bucket artifacts) — SageMaker sẽ ghi artifact theo job name.

#### 5.2 Cấu hình SageMaker Training Job

```python
import boto3
from sagemaker import get_execution_role
from sagemaker.sklearn import SKLearn

# Khởi tạo SageMaker session
sagemaker_session = boto3.Session().region_name
role = get_execution_role()

# Định nghĩa S3 paths
data_bucket = '<your-data-bucket>'  # vd: retail-forecast-data-123456
artifact_bucket = '<your-artifacts-bucket>'  # vd: retail-forecast-artifacts-123456

training_data_uri = f's3://{data_bucket}/training-data/'
model_artifacts_uri = f's3://{artifact_bucket}/models/'

# Tạo SKLearn estimator
sklearn_estimator = SKLearn(
    entry_point='train.py',
    role=role,
    instance_type='ml.m5.large',
    framework_version='0.23-1',
    py_version='py3',
    output_path=model_artifacts_uri,
    code_location=model_artifacts_uri
)

# Bắt đầu training job
sklearn_estimator.fit({'train': training_data_uri})

# Gợi ý: có thể thêm channel 'validation' nếu cần
# sklearn_estimator.fit({'train': training_data_uri, 'validation': validation_data_uri})
```

## 5. Monitoring và Logging

#### 6.1 Thiết lập CloudTrail cho S3 Events

Thao tác:

1) AWS Console → CloudTrail → Trails → Create trail
2) Tên: `retail-forecast-s3-trail` → Create new log bucket (hoặc chọn bucket logging có sẵn)
3) Event type: Management events ON; Data events: Add data event → S3 → chọn 2 bucket → Read/Write theo nhu cầu → Create trail

   ![CloudTrail data events](images/cloudtrail/ui-data-events-s3.png)

#### 6.2 Bật S3 Server Access Logging (tùy chọn)

Thao tác:

1) Tạo (hoặc chọn) một bucket log riêng (khác 2 bucket trên)
2) Mở bucket nguồn → tab Properties → Server access logging → Edit → Enable → chọn bucket log đích → Save

   ![S3 access logging](images/s3/ui-server-access-logging.png)

## 6. Validation và Testing

#### 7.1 Kiểm tra Bucket Configuration

- Versioning: bucket → Properties → Object Versioning = Enabled
- Lifecycle: bucket → Management → Lifecycle rules hiển thị 2 rule tương ứng
- Public access: bucket → Permissions → Block public access = ON (4 mục)

   ![Check properties](images/s3/ui-check-properties.png)

#### 7.2 Test Upload/Download (UI)

1) Upload: bucket data → Create folder `test/` → Upload file `test-file.txt`
2) Download: chọn file → Download → mở file để xác nhận nội dung

   ![Download object](images/s3/ui-download-object.png)

## Kết quả kỳ vọng

## ✅ Deliverables

- [ ] **Bucket Creation**: 2 bucket được tạo thành công (data & artifacts)
- [ ] **Versioning**: Versioning được bật cho cả 2 bucket
- [ ] **Lifecycle Rules**: Lifecycle policy được cấu hình để tối ưu chi phí
- [ ] **Security**: Block public access được thiết lập
- [ ] **IAM Permissions**: SageMaker có quyền truy cập S3 buckets
- [ ] **Data Upload**: Có thể upload và kiểm tra file dữ liệu huấn luyện
- [ ] **SageMaker Integration**: Training job có thể đọc từ S3 và ghi model artifacts
- [ ] **Monitoring**: CloudTrail và access logging được thiết lập

## 📊 Acceptance Criteria

1) Bucket hiển thị trong S3 Console và không public
2) `training-data/` chứa `train.csv`, `validation.csv`
3) Artifact xuất hiện trong `models/` sau khi training
4) Object versions hiển thị tại tab Versions (Show versions ON)

## Troubleshooting

### Common Issues

1. **Permission Denied khi upload/download**
   - Kiểm tra IAM permissions
   - Verify bucket policy

2. **Lifecycle rules không hoạt động**
   - Kiểm tra syntax của lifecycle policy
   - Verify prefix matching

3. **SageMaker không thể truy cập S3**
   - Kiểm tra execution role permissions
   - Verify S3 bucket names trong code

{{% notice warning %}}
⚠️ Gotchas

- Thiếu Versioning → khó rollback khi ghi đè dữ liệu/artifacts
- Prefix Lifecycle sai → object không chuyển lớp lưu trữ theo kỳ vọng
- Policy quá rộng ("*") → rủi ro bảo mật, hãy giới hạn theo bucket/object
- Tên bucket trùng → tạo thất bại, cần suffix duy nhất
{{% /notice %}}

## 💰 Cost Optimization (Gợi ý)

- Dữ liệu hiếm truy cập: chuyển STANDARD_IA sau 30 ngày, GLACIER sau 90–180 ngày
- Bật CloudTrail data events chỉ cho bucket critical để giảm chi phí log
- Sử dụng cùng Region với SageMaker để tránh chi phí cross-region

## 🔐 Security Hardening (Gợi ý)

- Luôn bật Block Public Access (4 tuỳ chọn)
- Dùng SSE-KMS với customer-managed CMK nếu có yêu cầu compliance
- Bucket policy deny public và enforce TLS (aws:SecureTransport = true)

{{% notice success %}}
🎯 Hoàn tất: Task 8 (S3 Data Storage) đã sẵn sàng cho tích hợp ở các task kế tiếp (training, inference, monitoring).
{{% /notice %}}
