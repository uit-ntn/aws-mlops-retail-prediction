---
title: "Task 8: S3 Data Storage"
date: 2024-01-01T00:00:00Z
weight: 8
chapter: false
pre: "<b>8. </b>"
---

## Mục tiêu

Thiết lập Amazon S3 để lưu trữ dữ liệu huấn luyện và model artifacts (đầu ra sau training). Đây là kho dữ liệu trung tâm cho pipeline ML.

## Nội dung chính

### 1. Tạo S3 Buckets

Chúng ta sẽ tạo 2 bucket chính:

- **Data bucket**: Lưu dữ liệu huấn luyện (ví dụ train.csv)
- **Artifact bucket**: Lưu trữ model artifact sinh ra từ SageMaker training job

### 2. Cấu hình S3 Buckets

#### 2.1 Tạo Data Bucket

```bash
# Tạo bucket cho dữ liệu huấn luyện
aws s3 mb s3://retail-forecast-data-bucket-$(date +%s) --region us-east-1
```

#### 2.2 Tạo Artifact Bucket

```bash
# Tạo bucket cho model artifacts
aws s3 mb s3://retail-forecast-artifacts-bucket-$(date +%s) --region us-east-1
```

### 3. Cấu hình Bucket Properties

#### 3.1 Bật Versioning

```bash
# Bật versioning cho data bucket
aws s3api put-bucket-versioning \
    --bucket retail-forecast-data-bucket \
    --versioning-configuration Status=Enabled

# Bật versioning cho artifact bucket
aws s3api put-bucket-versioning \
    --bucket retail-forecast-artifacts-bucket \
    --versioning-configuration Status=Enabled
```

#### 3.2 Thiết lập Lifecycle Rules

Tạo file `lifecycle-policy.json`:

```json
{
    "Rules": [
        {
            "ID": "DataLifecycleRule",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "training-data/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                }
            ]
        },
        {
            "ID": "ModelArtifactLifecycleRule",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "models/"
            },
            "Transitions": [
                {
                    "Days": 60,
                    "StorageClass": "STANDARD_IA"
                }
            ]
        }
    ]
}
```

Áp dụng lifecycle policy:

```bash
# Áp dụng cho data bucket
aws s3api put-bucket-lifecycle-configuration \
    --bucket retail-forecast-data-bucket \
    --lifecycle-configuration file://lifecycle-policy.json

# Áp dụng cho artifact bucket
aws s3api put-bucket-lifecycle-configuration \
    --bucket retail-forecast-artifacts-bucket \
    --lifecycle-configuration file://lifecycle-policy.json
```

#### 3.3 Bật Block Public Access

```bash
# Block public access cho data bucket
aws s3api put-public-access-block \
    --bucket retail-forecast-data-bucket \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Block public access cho artifact bucket
aws s3api put-public-access-block \
    --bucket retail-forecast-artifacts-bucket \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### 4. Cấu hình IAM Permissions

#### 4.1 Tạo IAM Policy cho S3 Access

Tạo file `s3-access-policy.json`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::retail-forecast-data-bucket/*",
                "arn:aws:s3:::retail-forecast-artifacts-bucket/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::retail-forecast-data-bucket",
                "arn:aws:s3:::retail-forecast-artifacts-bucket"
            ]
        }
    ]
}
```

#### 4.2 Gắn Policy vào SageMaker Execution Role

```bash
# Tạo policy
aws iam create-policy \
    --policy-name RetailForecastS3AccessPolicy \
    --policy-document file://s3-access-policy.json

# Gắn policy vào SageMaker role
aws iam attach-role-policy \
    --role-name SageMaker-ExecutionRole \
    --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/RetailForecastS3AccessPolicy
```

### 5. Tích hợp với SageMaker

#### 5.1 Upload Training Data

```bash
# Upload dữ liệu huấn luyện
aws s3 cp train.csv s3://retail-forecast-data-bucket/training-data/train.csv

# Upload validation data
aws s3 cp validation.csv s3://retail-forecast-data-bucket/training-data/validation.csv
```

#### 5.2 Cấu hình SageMaker Training Job

```python
import boto3
from sagemaker import get_execution_role
from sagemaker.sklearn import SKLearn

# Khởi tạo SageMaker session
sagemaker_session = boto3.Session().region_name
role = get_execution_role()

# Định nghĩa S3 paths
data_bucket = 'retail-forecast-data-bucket'
artifact_bucket = 'retail-forecast-artifacts-bucket'

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
```

### 6. Monitoring và Logging

#### 6.1 Thiết lập CloudTrail cho S3 Events

```bash
# Tạo CloudTrail để theo dõi S3 events
aws cloudtrail create-trail \
    --name retail-forecast-s3-trail \
    --s3-bucket-name retail-forecast-cloudtrail-logs
```

#### 6.2 Thiết lập S3 Access Logging

```bash
# Bật access logging
aws s3api put-bucket-logging \
    --bucket retail-forecast-data-bucket \
    --bucket-logging-status file://logging-config.json
```

### 7. Validation và Testing

#### 7.1 Kiểm tra Bucket Configuration

```bash
# Kiểm tra versioning
aws s3api get-bucket-versioning --bucket retail-forecast-data-bucket

# Kiểm tra lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket retail-forecast-data-bucket

# Kiểm tra public access block
aws s3api get-public-access-block --bucket retail-forecast-data-bucket
```

#### 7.2 Test Data Upload/Download

```bash
# Test upload
echo "test data" > test-file.txt
aws s3 cp test-file.txt s3://retail-forecast-data-bucket/test/

# Test download
aws s3 cp s3://retail-forecast-data-bucket/test/test-file.txt downloaded-test-file.txt

# Verify content
cat downloaded-test-file.txt
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Bucket Creation**: 2 bucket được tạo thành công (data & artifacts)
- [ ] **Versioning**: Versioning được bật cho cả 2 bucket
- [ ] **Lifecycle Rules**: Lifecycle policy được cấu hình để tối ưu chi phí
- [ ] **Security**: Block public access được thiết lập
- [ ] **IAM Permissions**: SageMaker có quyền truy cập S3 buckets
- [ ] **Data Upload**: Có thể upload và kiểm tra file dữ liệu huấn luyện
- [ ] **SageMaker Integration**: Training job có thể đọc từ S3 và ghi model artifacts
- [ ] **Monitoring**: CloudTrail và access logging được thiết lập

### 📊 Verification Steps

1. **Bucket được tạo thành công và hiển thị trong AWS Console**
   ```bash
   aws s3 ls | grep retail-forecast
   ```

2. **Có thể upload và kiểm tra file dữ liệu huấn luyện**
   ```bash
   aws s3 ls s3://retail-forecast-data-bucket/training-data/
   ```

3. **Model artifact từ SageMaker xuất hiện trong artifact bucket**
   ```bash
   aws s3 ls s3://retail-forecast-artifacts-bucket/models/
   ```

4. **Dữ liệu và model artifact được quản lý an toàn với versioning**
   ```bash
   aws s3api list-object-versions --bucket retail-forecast-data-bucket
   ```

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

### Useful Commands

```bash
# Kiểm tra bucket size
aws s3 ls s3://retail-forecast-data-bucket --recursive --human-readable --summarize

# Sync local folder với S3
aws s3 sync ./local-data/ s3://retail-forecast-data-bucket/training-data/

# Copy giữa các bucket
aws s3 cp s3://source-bucket/file s3://destination-bucket/file
```

---

**Next Step**: [Task 9: EKS Node Group Setup](../9-eks-nodegroup/)