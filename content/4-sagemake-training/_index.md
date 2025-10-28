---
title: "SageMaker Training & Model Registry"
weight: 4
chapter: false
pre: "<b>4. </b>"
---

## 🎯 Mục tiêu Task 4

Huấn luyện mô hình dự báo **BASKET_PRICE_SENSITIVITY** (Low/Medium/High) bằng Amazon SageMaker, tự động hóa toàn bộ luồng **ETL → Training → Evaluation → Model Registry**.

→ **Đây là trái tim của pipeline MLOps** — nơi thực hiện quy trình xử lý dữ liệu tự động (ETL) và huấn luyện mô hình học máy.

📥 **Input**
- AWS Account với quyền SageMaker/S3/CloudWatch
- Retail transaction data trong S3 `raw/`
- Project naming: `mlops-retail-prediction-dev`

✅ **Kết quả mong đợi**
- Luồng ETL → Train → Evaluate → Save → (Register) chạy tự động end-to-end
- Model đạt accuracy ≥ 80%, F1 ≥ 0.7
- Artifact và kết quả huấn luyện được lưu đầy đủ trong S3

💰 **Chi phí ước tính**: ~**$0.3-0.5/job** (instance ml.m5.large, thời gian ~10-15 phút). Nếu bật Spot → giảm 70-80%.


📌 **Các bước**
1. **ETL Pipeline** - Automated data processing  
2. **Multi-Model Training** - LR, RF, XGBoost comparison
3. **Model Evaluation** - Comprehensive metrics + confusion matrix
4. **Model Registry** - Version control và approval workflow
5. **CI/CD Integration** - Export model info for deployment

{{% notice info %}}
**💡 Task 4 Focus - MLOps Core Pipeline:**
- ✅ **Automated ETL** - Raw data → Gold features pipeline
- ✅ **Multi-Model Training** - LR/RF/XGBoost comparison  
- ✅ **Comprehensive Evaluation** - Accuracy, F1, Confusion Matrix
- ✅ **Model Registry** - Version control và metadata tracking
- ✅ **Cost Optimization** - Spot instances + auto cleanup

**Trái tim của MLOps** - tự động hóa hoàn toàn từ data đến model
{{% /notice %}}

📊 **Success Criteria**
- ✅ **ETL Success** - Raw data → clean features pipeline hoạt động
- ✅ **Model Performance** - Accuracy ≥ 80%, F1 ≥ 0.7
- ✅ **Automation** - End-to-end pipeline chạy tự động
- ✅ **Cost Efficiency** - Spot instances, auto cleanup

⚠️ **Lưu ý**
- **ETL Pipeline** cần memory đủ lớn để xử lý transaction data
- **Model Training** sử dụng Spot để giảm chi phí 
- **Evaluation Metrics** phải consistent với business requirements

## Thực hiện bằng AWS Console - Hướng dẫn chi tiết từng bước

### Bước 1: Kiểm tra dữ liệu trong S3 (từ Task 3)

#### 1.1. Xác minh bucket và dữ liệu
1. **AWS Console** → **S3** 
2. **Tìm bucket**: `mlops-retail-prediction-dev-[account-id]` (đã tạo ở Task 3)
3. **Kiểm tra cấu trúc**:
   ```
   ✅ raw/        # có file dunnhumby_The-Complete-Journey.csv
   ✅ silver/     # sẽ chứa dữ liệu đã xử lý
   ✅ gold/       # sẽ chứa features training
   ✅ artifacts/  # sẽ chứa model outputs
   ```

#### 1.2. Xác minh IAM Role (từ Task 2)
1. **AWS Console** → **IAM** → **Roles**
2. **Tìm role**: `mlops-retail-prediction-dev-sagemaker-execution` (đã tạo ở Task 2)
3. **Kiểm tra permissions**:
   - ✅ `AmazonSageMakerFullAccess`
   - ✅ `AmazonS3FullAccess`
   - ✅ `CloudWatchLogsFullAccess`

### Bước 2: Training Model với SageMaker Studio

#### 2.1. Mở SageMaker Studio
1. **AWS Console** → Tìm "SageMaker" → Click **Amazon SageMaker**
2. **Sidebar trái** → Click **"Studio"**
3. **Click "Create a SageMaker domain"** (nếu chưa có)
4. **Domain name**: `mlops-retail-domain`
5. **Default execution role**: Chọn `mlops-retail-prediction-dev-sagemaker-execution` (từ Task 2)
6. Click **"Submit"** → Đợi 5-10 phút
7. **Domain created** → Click **"Launch"** → **"Studio"**

#### 2.2. Tạo Training Job
1. **SageMaker Studio mở** → **Sidebar trái** → Click **"Training"** → **"Training jobs"**
2. Click **"Create training job"**

#### 2.3. Cấu hình Job Details
1. **Job name**: `retail-prediction-training-20241011`
2. **IAM role**: Chọn `mlops-retail-prediction-dev-sagemaker-execution` (từ Task 2)
3. Click **"Next"**

#### 2.4. Algorithm Options
1. **Algorithm source**: Chọn **"Your own algorithm container in ECR"**
2. **Container path**: 
   ```
   382416733822.dkr.ecr.ap-southeast-1.amazonaws.com/scikit-learn:1.0-1-cpu-py3
   ```
3. **Input mode**: `File`
4. Click **"Next"**

#### 2.5. Configure Resource
1. **Instance type**: `ml.m5.large`
2. **Instance count**: `1`
3. **Additional storage per instance (GB)**: `30`
4. **Use Spot training**: ✅ **Check** (tiết kiệm 70% chi phí)
5. **Max wait time**: `7200` seconds (2 hours)
6. **Max training time**: `3600` seconds (1 hour)
7. Click **"Next"**

#### 2.6. Configure Input Data
1. **Channel name**: `training`
2. **Input mode**: `File`
3. **Data source**: `S3`
4. **S3 location**: `s3://mlops-retail-prediction-dev-[account-id]/raw/`
5. **Content type**: `text/csv`
6. **Compression**: `None`
7. **Record wrapper**: `None`
8. Click **"Add channel"**
9. Click **"Next"**

#### 2.7. Configure Output Data
1. **S3 output path**: `s3://mlops-retail-prediction-dev-[account-id]/artifacts/` (bucket từ Task 3)
2. **Encryption**: `None`
3. Click **"Next"**

#### 2.8. Configure Hyperparameters
1. Click **"Add hyperparameter"** cho từng tham số:

| Key | Value |
|-----|-------|
| `model_type` | `all` |
| `random_state` | `42` |
| `cv_folds` | `5` |
| `test_size` | `0.2` |

2. Click **"Next"**

#### 2.9. Review và Create
1. **Review** tất cả cài đặt
2. Click **"Create training job"**

### Bước 3: Upload Training Script

#### 3.1. Tạo Training Script
1. **SageMaker Studio** → Click **"+"** → **"Python 3"** notebook
2. **Tạo cell mới** và paste code sau:

```python
%%writefile train.py

import pandas as pd
import numpy as np
import joblib
import os
import json
import argparse
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.metrics import accuracy_score, f1_score, classification_report
import xgboost as xgb

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAINING'))
    parser.add_argument('--model_type', type=str, default='all')
    args = parser.parse_args()
    
    # Load data
    input_files = [os.path.join(args.train, file) for file in os.listdir(args.train)]
    raw_data = pd.concat([pd.read_csv(file) for file in input_files])
    
    # Basic preprocessing
    clean_data = raw_data.dropna(subset=['BASKET_ID', 'SPEND', 'BASKET_PRICE_SENSITIVITY'])
    
    # Create features
    features = clean_data.groupby('BASKET_ID').agg({
        'SPEND': ['sum', 'mean', 'std', 'count'],
        'QUANTITY': ['sum', 'mean'],
        'PROD_CODE': 'nunique',
        'BASKET_PRICE_SENSITIVITY': lambda x: x.iloc[0]
    }).reset_index()
    
    # Flatten columns
    features.columns = ['basket_id', 'total_spend', 'avg_spend', 'spend_std', 
                       'basket_size', 'total_qty', 'avg_qty', 'unique_products', 'target']
    features['spend_std'] = features['spend_std'].fillna(0)
    
    # Prepare for ML
    X = features[['total_spend', 'avg_spend', 'spend_std', 'basket_size', 
                  'total_qty', 'avg_qty', 'unique_products']]
    y = features['target']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    
    # Train models
    models = {
        'random_forest': RandomForestClassifier(n_estimators=100, random_state=42),
        'logistic': LogisticRegression(random_state=42, max_iter=1000),
        'decision_tree': DecisionTreeClassifier(random_state=42, max_depth=10)
    }
    
    best_model = None
    best_score = 0
    results = {}
    
    for name, model in models.items():
        model.fit(X_train, y_train)
        pred = model.predict(X_test)
        f1 = f1_score(y_test, pred, average='macro')
        acc = accuracy_score(y_test, pred)
        
        results[name] = {'accuracy': acc, 'f1_score': f1}
        
        if f1 > best_score:
            best_score = f1
            best_model = model
            best_name = name
    
    # Save best model
    joblib.dump(best_model, os.path.join(args.model_dir, 'model.joblib'))
    
    # Save results
    with open(os.path.join(args.model_dir, 'results.json'), 'w') as f:
        json.dump({
            'best_model': best_name,
            'best_f1_score': best_score,
            'all_results': results
        }, f)
    
    print(f"Best model: {best_name} with F1-score: {best_score:.4f}")

if __name__ == '__main__':
    main()
```

3. **Run cell** → File `train.py` được tạo

#### 3.2. Upload script lên S3 (sử dụng bucket từ Task 3)
1. **Tạo cell mới**:

```python
import boto3

s3 = boto3.client('s3')
bucket_name = 'mlops-retail-prediction-dev-[account-id]'  # Thay [account-id]

# Upload training script
s3.upload_file('train.py', bucket_name, 'code/train.py')
print("✅ Training script uploaded to S3")
```

2. **Run cell**

### Bước 4: Chạy Training Job

#### 4.1. Theo dõi Training Job
1. **SageMaker Console** → **Training jobs**
2. **Tìm job**: `retail-prediction-training-20241011`
3. **Status**: `InProgress` → `Completed` (5-10 phút)
4. **Click vào job name** để xem details

#### 4.2. Xem kết quả
1. **Scroll xuống** → **Monitor** section
2. **CloudWatch logs** → Click **"View logs"**
3. **Tìm dòng**: `Best model: random_forest with F1-score: 0.8234`

### Bước 5: Model Registry

#### 5.1. Tạo Model Package Group
1. **SageMaker Console** → **Sidebar** → **"Inference"** → **"Model registry"**
2. Click **"Create model package group"**
3. **Name**: `retail-price-sensitivity-models`
4. **Description**: `Models for retail customer price sensitivity prediction`
5. Click **"Create model package group"**

#### 5.2. Register Model
1. **Training jobs** → Click job `retail-prediction-training-20241011`
2. **Scroll xuống** → **Model artifacts** section
3. Click **"Create model package"**
4. **Model package group**: `retail-price-sensitivity-models`
5. **Model package version description**: `First retail prediction model v1.0`
6. **Approval status**: `PendingManualApproval`
7. Click **"Create model package"**

#### 5.3. Approve Model (nếu F1-score ≥ 0.7)
1. **Model registry** → **Model package groups** → Click `retail-price-sensitivity-models`
2. **Versions** tab → Click **version 1**
3. **Update status** → **"Approved"**
4. **Description**: `Auto-approved: F1-score ≥ 0.7`
5. Click **"Update status"**

### Bước 6: Kiểm tra kết quả

#### 6.1. Download Model Artifacts
1. **Training job details** → **Output** section
2. **S3 model artifacts**: Click **S3 URI**
3. **S3 Console mở** → Click **"Download"** file `model.tar.gz`
4. **Extract** → Kiểm tra file `model.joblib` và `results.json`

#### 6.2. Verification
```json
{
  "best_model": "random_forest",
  "best_f1_score": 0.8234,
  "all_results": {
    "random_forest": {"accuracy": 0.8456, "f1_score": 0.8234},
    "logistic": {"accuracy": 0.8123, "f1_score": 0.7891},
    "decision_tree": {"accuracy": 0.7765, "f1_score": 0.7456}
  }
}
```

### ✅ Hoàn thành!

**Bạn đã thành công:**
- ✅ **Tạo S3 bucket** và upload dữ liệu
- ✅ **Cấu hình IAM role** cho SageMaker  
- ✅ **Train model** với Random Forest, Logistic Regression, Decision Tree
- ✅ **Chọn best model** dựa trên F1-score
- ✅ **Register model** trong Model Registry
- ✅ **Approve model** cho production

**Kết quả:**
- 🎯 **Best Model**: Random Forest
- 📊 **F1-Score**: 0.8234 (> 0.7 ✅)
- 📊 **Accuracy**: 0.8456 (> 0.8 ✅)
- 💰 **Chi phí**: ~$0.3 (với Spot instances)

**Next Steps:** Model đã sẵn sàng cho deployment trong Task 10!

## Thực hiện bằng Code

### 1. ETL Pipeline - Automated Data Processing

### 1.1. ETL Processing Script

**Tạo file `etl_processing.py`:**
```python
import pandas as pd
import numpy as np
import boto3
import os
import argparse
from sklearn.preprocessing import LabelEncoder, StandardScaler
from sklearn.model_selection import train_test_split
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def parse_args():
    """Parse command line arguments for ETL"""
    parser = argparse.ArgumentParser()
    parser.add_argument('--input-path', type=str, required=True)
    parser.add_argument('--output-path', type=str, required=True)
    parser.add_argument('--bucket-name', type=str, required=True)
    return parser.parse_args()

def load_raw_data(bucket_name, raw_prefix='raw/'):
    """Load transaction data from S3 raw folder"""
    logger.info(f"Loading raw data from s3://{bucket_name}/{raw_prefix}")
    
    s3 = boto3.client('s3')
    
    # List all CSV files in raw folder
    response = s3.list_objects_v2(Bucket=bucket_name, Prefix=raw_prefix)
    csv_files = [obj['Key'] for obj in response.get('Contents', []) if obj['Key'].endswith('.csv')]
    
    # Load and concatenate all transaction files
    dataframes = []
    for file_key in csv_files:
        logger.info(f"Loading {file_key}")
        df = pd.read_csv(f's3://{bucket_name}/{file_key}')
        dataframes.append(df)
    
    combined_df = pd.concat(dataframes, ignore_index=True)
    logger.info(f"Combined dataset shape: {combined_df.shape}")
    
    return combined_df

def clean_data(df):
    """Clean and preprocess transaction data"""
    logger.info("Starting data cleaning...")
    
    # Remove missing values in critical columns
    critical_cols = ['BASKET_ID', 'SPEND', 'QUANTITY', 'BASKET_PRICE_SENSITIVITY']
    df = df.dropna(subset=critical_cols)
    
    # Convert date columns
    if 'SHOP_DATE' in df.columns:
        df['SHOP_DATE'] = pd.to_datetime(df['SHOP_DATE'], errors='coerce')
    
    # Remove outliers (spend > 99th percentile)
    spend_99 = df['SPEND'].quantile(0.99)
    df = df[df['SPEND'] <= spend_99]
    
    # Filter valid price sensitivity values
    valid_sensitivity = ['Low', 'Medium', 'High']
    df = df[df['BASKET_PRICE_SENSITIVITY'].isin(valid_sensitivity)]
    
    logger.info(f"After cleaning: {df.shape}")
    return df

def create_features(df):
    """Create ML features from transaction data"""
    logger.info("Creating features...")
    
    # Aggregate by BASKET_ID to create basket-level features
    basket_features = df.groupby('BASKET_ID').agg({
        'SPEND': ['sum', 'mean', 'std', 'count'],
        'QUANTITY': ['sum', 'mean'],
        'PROD_CODE': 'nunique',
        'STORE_FORMAT': lambda x: x.iloc[0],
        'STORE_REGION': lambda x: x.iloc[0],
        'SHOP_WEEK': lambda x: x.iloc[0],
        'SHOP_WEEKDAY': lambda x: x.iloc[0] if 'SHOP_WEEKDAY' in df.columns else 1,
        'SHOP_HOUR': lambda x: x.iloc[0] if 'SHOP_HOUR' in df.columns else 12,
        'BASKET_PRICE_SENSITIVITY': lambda x: x.iloc[0]  # Target
    }).reset_index()
    
    # Flatten column names
    basket_features.columns = [
        'basket_id', 'total_spend', 'avg_spend', 'spend_std', 'basket_size',
        'total_quantity', 'avg_quantity', 'unique_products',
        'store_format', 'store_region', 'shop_week', 'shop_weekday', 'shop_hour',
        'price_sensitivity'
    ]
    
    # Fill NaN values
    basket_features['spend_std'] = basket_features['spend_std'].fillna(0)
    
    # Create additional features
    basket_features['spend_per_item'] = basket_features['total_spend'] / basket_features['basket_size']
    basket_features['quantity_per_product'] = basket_features['total_quantity'] / basket_features['unique_products']
    
    # Encode categorical variables
    le_store_format = LabelEncoder()
    le_store_region = LabelEncoder()
    
    basket_features['store_format_encoded'] = le_store_format.fit_transform(basket_features['store_format'])
    basket_features['store_region_encoded'] = le_store_region.fit_transform(basket_features['store_region'])
    
    # Save encoders for later use
    encoders = {
        'store_format': le_store_format.classes_.tolist(),
        'store_region': le_store_region.classes_.tolist()
    }
    
    logger.info(f"Features created: {basket_features.shape}")
    return basket_features, encoders

def split_and_save_data(features_df, output_path, bucket_name, test_size=0.2):
    """Split data and save to S3 gold folder"""
    logger.info("Splitting and saving data...")
    
    # Prepare features and target
    feature_cols = [col for col in features_df.columns if col not in ['basket_id', 'price_sensitivity', 'store_format', 'store_region']]
    X = features_df[feature_cols]
    y = features_df['price_sensitivity']
    
    # Train-test split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=42, stratify=y
    )
    
    # Create train and test datasets
    train_df = X_train.copy()
    train_df['price_sensitivity'] = y_train
    
    test_df = X_test.copy()
    test_df['price_sensitivity'] = y_test
    
    # Save to S3 as Parquet
    train_path = f's3://{bucket_name}/gold/train_features.parquet'
    test_path = f's3://{bucket_name}/gold/test_features.parquet'
    
    train_df.to_parquet(train_path, compression='snappy', index=False)
    test_df.to_parquet(test_path, compression='snappy', index=False)
    
    logger.info(f"Train data saved: {train_path} ({train_df.shape})")
    logger.info(f"Test data saved: {test_path} ({test_df.shape})")
    
    return train_path, test_path, feature_cols

def main():
    """Main ETL function"""
    args = parse_args()
    
    try:
        # Load raw data
        raw_df = load_raw_data(args.bucket_name)
        
        # Clean data
        clean_df = clean_data(raw_df)
        
        # Create features
        features_df, encoders = create_features(clean_df)
        
        # Split and save
        train_path, test_path, feature_cols = split_and_save_data(
            features_df, args.output_path, args.bucket_name
        )
        
        # Save metadata
        metadata = {
            'feature_columns': feature_cols,
            'encoders': encoders,
            'train_samples': len(features_df) * 0.8,
            'test_samples': len(features_df) * 0.2,
            'target_classes': ['Low', 'Medium', 'High'],
            'etl_timestamp': pd.Timestamp.now().isoformat()
        }
        
        metadata_path = f's3://{args.bucket_name}/gold/metadata.json'
        with open('/tmp/metadata.json', 'w') as f:
            json.dump(metadata, f, indent=2)
        
        s3 = boto3.client('s3')
        s3.upload_file('/tmp/metadata.json', args.bucket_name, 'gold/metadata.json')
        
        logger.info("✅ ETL Pipeline completed successfully")
        
    except Exception as e:
        logger.error(f"❌ ETL Pipeline failed: {str(e)}")
        raise

if __name__ == '__main__':
    main()
```

### 1.2. SageMaker Processing Job Setup

**Chạy ETL Pipeline trên SageMaker:**
```python
import boto3
import sagemaker
from sagemaker.sklearn.processing import SKLearnProcessor
from sagemaker.processing import ProcessingInput, ProcessingOutput

# Initialize SageMaker session
sagemaker_session = sagemaker.Session()
role = sagemaker.get_execution_role()
bucket_name = 'mlops-retail-prediction-dev-123456789012'

# Create SKLearn processor
sklearn_processor = SKLearnProcessor(
    framework_version='1.0-1',
    role=role,
    instance_type='ml.m5.large',
    instance_count=1,
    base_job_name='retail-etl-processing',
    sagemaker_session=sagemaker_session
)

# Run ETL processing job
sklearn_processor.run(
    code='etl_processing.py',
    inputs=[
        ProcessingInput(
            source=f's3://{bucket_name}/raw/',
            destination='/opt/ml/processing/input'
        )
    ],
    outputs=[
        ProcessingOutput(
            output_name='features',
            source='/opt/ml/processing/output',
            destination=f's3://{bucket_name}/gold/'
        )
    ],
    arguments=[
        '--input-path', '/opt/ml/processing/input',
        '--output-path', '/opt/ml/processing/output', 
        '--bucket-name', bucket_name
    ]
)

print("✅ ETL Processing job completed")
```

## 2. Multi-Model Training Pipeline

### 2.1. Multi-Model Training Script

**Tạo file `train_multi_model.py`:**
```python
import argparse
import os
import pandas as pd
import numpy as np
import joblib
import json
import logging
from datetime import datetime

# ML models
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
from sklearn.tree import DecisionTreeClassifier
import xgboost as xgb

# Metrics
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    classification_report, confusion_matrix
)
from sklearn.model_selection import cross_val_score
import matplotlib.pyplot as plt
import seaborn as sns

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def parse_args():
    """Parse command line arguments"""
    parser = argparse.ArgumentParser()
    
    # SageMaker paths
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAIN'))
    parser.add_argument('--test', type=str, default=os.environ.get('SM_CHANNEL_TEST'))
    
    # Model selection
    parser.add_argument('--model-type', type=str, default='all', 
                       choices=['logistic', 'decision_tree', 'random_forest', 'xgboost', 'all'])
    
    # Hyperparameters
    parser.add_argument('--random-state', type=int, default=42)
    parser.add_argument('--cv-folds', type=int, default=5)
    
    return parser.parse_args()

def load_data(train_path, test_path):
    """Load training and test data"""
    logger.info(f"Loading data from {train_path} and {test_path}")
    
    train_df = pd.read_parquet(f'{train_path}/train_features.parquet')
    test_df = pd.read_parquet(f'{test_path}/test_features.parquet')
    
    # Separate features and target
    feature_cols = [col for col in train_df.columns if col != 'price_sensitivity']
    
    X_train = train_df[feature_cols]
    y_train = train_df['price_sensitivity']
    X_test = test_df[feature_cols]
    y_test = test_df['price_sensitivity']
    
    logger.info(f"Train shape: {X_train.shape}, Test shape: {X_test.shape}")
    logger.info(f"Target distribution: {y_train.value_counts().to_dict()}")
    
    return X_train, X_test, y_train, y_test, feature_cols

def train_logistic_regression(X_train, y_train, random_state=42):
    """Train Logistic Regression model"""
    logger.info("Training Logistic Regression...")
    
    model = LogisticRegression(
        random_state=random_state,
        max_iter=1000,
        multi_class='multinomial',
        solver='lbfgs'
    )
    model.fit(X_train, y_train)
    
    return model

def train_decision_tree(X_train, y_train, random_state=42):
    """Train Decision Tree model"""
    logger.info("Training Decision Tree...")
    
    model = DecisionTreeClassifier(
        random_state=random_state,
        max_depth=10,
        min_samples_split=20,
        min_samples_leaf=10
    )
    model.fit(X_train, y_train)
    
    return model

def train_random_forest(X_train, y_train, random_state=42):
    """Train Random Forest model"""
    logger.info("Training Random Forest...")
    
    model = RandomForestClassifier(
        n_estimators=100,
        max_depth=15,
        min_samples_split=20,
        min_samples_leaf=5,
        random_state=random_state,
        n_jobs=-1
    )
    model.fit(X_train, y_train)
    
    return model

def train_xgboost(X_train, y_train, random_state=42):
    """Train XGBoost model"""
    logger.info("Training XGBoost...")
    
    # Encode target labels for XGBoost
    from sklearn.preprocessing import LabelEncoder
    le = LabelEncoder()
    y_encoded = le.fit_transform(y_train)
    
    model = xgb.XGBClassifier(
        n_estimators=100,
        max_depth=6,
        learning_rate=0.1,
        subsample=0.8,
        colsample_bytree=0.8,
        random_state=random_state,
        eval_metric='mlogloss'
    )
    model.fit(X_train, y_encoded)
    
    # Store label encoder
    model.label_encoder = le
    
    return model

def evaluate_model(model, X_train, X_test, y_train, y_test, model_name, cv_folds=5):
    """Comprehensive model evaluation"""
    logger.info(f"Evaluating {model_name}...")
    
    # Handle XGBoost predictions
    if model_name == 'xgboost':
        y_train_pred = model.label_encoder.inverse_transform(model.predict(X_train))
        y_test_pred = model.label_encoder.inverse_transform(model.predict(X_test))
        
        # Cross-validation with encoded labels
        y_train_encoded = model.label_encoder.transform(y_train)
        cv_scores = cross_val_score(model, X_train, y_train_encoded, cv=cv_folds, scoring='accuracy')
    else:
        y_train_pred = model.predict(X_train)
        y_test_pred = model.predict(X_test)
        cv_scores = cross_val_score(model, X_train, y_train, cv=cv_folds, scoring='accuracy')
    
    # Calculate metrics
    metrics = {
        'model_name': model_name,
        'train_accuracy': accuracy_score(y_train, y_train_pred),
        'test_accuracy': accuracy_score(y_test, y_test_pred),
        'test_precision': precision_score(y_test, y_test_pred, average='macro'),
        'test_recall': recall_score(y_test, y_test_pred, average='macro'),
        'test_f1': f1_score(y_test, y_test_pred, average='macro'),
        'cv_accuracy_mean': cv_scores.mean(),
        'cv_accuracy_std': cv_scores.std()
    }
    
    # Classification report
    class_report = classification_report(y_test, y_test_pred, output_dict=True)
    
    # Confusion matrix
    cm = confusion_matrix(y_test, y_test_pred)
    
    logger.info(f"{model_name} - Test Accuracy: {metrics['test_accuracy']:.4f}, F1: {metrics['test_f1']:.4f}")
    
    return metrics, class_report, cm, y_test_pred

def save_confusion_matrix(cm, model_name, output_dir, class_names=['High', 'Low', 'Medium']):
    """Save confusion matrix plot"""
    plt.figure(figsize=(8, 6))
    sns.heatmap(cm, annot=True, fmt='d', cmap='Blues', 
                xticklabels=class_names, yticklabels=class_names)
    plt.title(f'Confusion Matrix - {model_name}')
    plt.ylabel('True Label')
    plt.xlabel('Predicted Label')
    
    cm_path = os.path.join(output_dir, f'confusion_matrix_{model_name.lower()}.png')
    plt.savefig(cm_path, dpi=300, bbox_inches='tight')
    plt.close()
    
    return cm_path

def compare_models(all_metrics):
    """Compare all models and select best one"""
    logger.info("Comparing models...")
    
    comparison_df = pd.DataFrame(all_metrics)
    comparison_df = comparison_df.sort_values('test_f1', ascending=False)
    
    best_model_name = comparison_df.iloc[0]['model_name']
    logger.info(f"Best model: {best_model_name} (F1: {comparison_df.iloc[0]['test_f1']:.4f})")
    
    return comparison_df, best_model_name

def main():
    """Main training function"""
    args = parse_args()
    
    try:
        # Load data
        X_train, X_test, y_train, y_test, feature_cols = load_data(args.train, args.test)
        
        # Initialize results storage
        all_models = {}
        all_metrics = []
        all_reports = {}
        all_predictions = {}
        
        # Define models to train
        models_to_train = ['logistic', 'decision_tree', 'random_forest', 'xgboost'] if args.model_type == 'all' else [args.model_type]
        
        # Train models
        for model_name in models_to_train:
            logger.info(f"\n{'='*50}")
            logger.info(f"Training {model_name.upper()}")
            logger.info(f"{'='*50}")
            
            # Train model
            if model_name == 'logistic':
                model = train_logistic_regression(X_train, y_train, args.random_state)
            elif model_name == 'decision_tree':
                model = train_decision_tree(X_train, y_train, args.random_state)
            elif model_name == 'random_forest':
                model = train_random_forest(X_train, y_train, args.random_state)
            elif model_name == 'xgboost':
                model = train_xgboost(X_train, y_train, args.random_state)
            
            # Evaluate model
            metrics, class_report, cm, y_pred = evaluate_model(
                model, X_train, X_test, y_train, y_test, model_name, args.cv_folds
            )
            
            # Store results
            all_models[model_name] = model
            all_metrics.append(metrics)
            all_reports[model_name] = class_report
            all_predictions[model_name] = y_pred
            
            # Save confusion matrix
            save_confusion_matrix(cm, model_name, args.model_dir)
        
        # Compare models
        comparison_df, best_model_name = compare_models(all_metrics)
        
        # Save best model
        best_model = all_models[best_model_name]
        joblib.dump(best_model, os.path.join(args.model_dir, 'model.joblib'))
        
        # Save all evaluation results
        evaluation_results = {
            'model_comparison': comparison_df.to_dict('records'),
            'best_model': best_model_name,
            'classification_reports': all_reports,
            'feature_columns': feature_cols,
            'target_classes': ['High', 'Low', 'Medium'],
            'training_timestamp': datetime.now().isoformat()
        }
        
        with open(os.path.join(args.model_dir, 'evaluation_results.json'), 'w') as f:
            json.dump(evaluation_results, f, indent=2)
        
        # Save model metadata
        metadata = {
            'model_type': best_model_name,
            'accuracy': float(comparison_df.iloc[0]['test_accuracy']),
            'f1_score': float(comparison_df.iloc[0]['test_f1']),
            'precision': float(comparison_df.iloc[0]['test_precision']),
            'recall': float(comparison_df.iloc[0]['test_recall']),
            'cv_accuracy': float(comparison_df.iloc[0]['cv_accuracy_mean']),
            'feature_count': len(feature_cols),
            'train_samples': len(X_train),
            'test_samples': len(X_test)
        }
        
        with open(os.path.join(args.model_dir, 'model_metadata.json'), 'w') as f:
            json.dump(metadata, f, indent=2)
        
        logger.info(f"\n✅ Training completed successfully!")
        logger.info(f"Best model: {best_model_name}")
        logger.info(f"Test accuracy: {metadata['accuracy']:.4f}")
        logger.info(f"Test F1-score: {metadata['f1_score']:.4f}")
        
    except Exception as e:
        logger.error(f"❌ Training failed: {str(e)}")
        raise

if __name__ == '__main__':
    main()
```

### 2.2. SageMaker Training Job Setup

**Khởi chạy multi-model training:**
```python
import boto3
import sagemaker
from sagemaker.sklearn import SKLearn
from sagemaker.inputs import TrainingInput
from datetime import datetime

# Initialize SageMaker
sagemaker_session = sagemaker.Session()
role = sagemaker.get_execution_role()
bucket_name = 'mlops-retail-prediction-dev-123456789012'

# Define hyperparameters
hyperparameters = {
    'model-type': 'all',  # Train all models
    'random-state': 42,
    'cv-folds': 5
}

# Create SKLearn estimator with Spot instances
sklearn_estimator = SKLearn(
    entry_point='train_multi_model.py',
    source_dir='.',
    role=role,
    instance_type='ml.m5.large',
    instance_count=1,
    framework_version='1.0-1',
    py_version='py3',
    output_path=f's3://{bucket_name}/artifacts/',
    hyperparameters=hyperparameters,
    max_run=3600,  # 1 hour timeout
    use_spot_instances=True,  # Cost optimization
    max_wait=7200,  # 2 hours max wait
    checkpoint_s3_uri=f's3://{bucket_name}/checkpoints',
    base_job_name='retail-prediction-training'
)

# Define input data channels
train_input = TrainingInput(
    s3_data=f's3://{bucket_name}/gold/',
    content_type='application/x-parquet',
    s3_data_type='S3Prefix',
    input_mode='File'
)

# Start training job
job_name = f"retail-prediction-{datetime.now().strftime('%Y%m%d-%H%M%S')}"

sklearn_estimator.fit(
    {
        'train': train_input,
        'test': train_input
    },
    job_name=job_name,
    wait=True
)

print(f"✅ Multi-model training completed: {job_name}")
print(f"Model artifacts: {sklearn_estimator.model_data}")
```

## 3. Model Evaluation & Analysis
### 3.1. Model Performance Analysis

**Analyze training results:**
```python
import boto3
import json
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

def download_and_analyze_results(bucket_name, job_name):
    """Download and analyze training results"""
    s3 = boto3.client('s3')
    
    # Download evaluation results
    eval_key = f'artifacts/{job_name}/output/model.tar.gz'
    
    # Extract and load results
    import tarfile
    import tempfile
    
    with tempfile.TemporaryDirectory() as temp_dir:
        # Download model artifacts
        s3.download_file(bucket_name, eval_key, f'{temp_dir}/model.tar.gz')
        
        # Extract
        with tarfile.open(f'{temp_dir}/model.tar.gz', 'r:gz') as tar:
            tar.extractall(temp_dir)
        
        # Load evaluation results
        with open(f'{temp_dir}/evaluation_results.json', 'r') as f:
            eval_results = json.load(f)
        
        return eval_results

def create_model_comparison_chart(eval_results):
    """Create model comparison visualization"""
    comparison_df = pd.DataFrame(eval_results['model_comparison'])
    
    # Metrics to compare
    metrics = ['test_accuracy', 'test_f1', 'test_precision', 'test_recall']
    
    fig, axes = plt.subplots(2, 2, figsize=(15, 10))
    axes = axes.ravel()
    
    for i, metric in enumerate(metrics):
        ax = axes[i]
        bars = ax.bar(comparison_df['model_name'], comparison_df[metric])
        ax.set_title(f'{metric.replace("_", " ").title()}')
        ax.set_ylabel('Score')
        ax.set_ylim(0, 1)
        
        # Add value labels on bars
        for bar in bars:
            height = bar.get_height()
            ax.text(bar.get_x() + bar.get_width()/2., height + 0.01,
                   f'{height:.3f}', ha='center', va='bottom')
    
    plt.tight_layout()
    plt.savefig('model_comparison.png', dpi=300, bbox_inches='tight')
    plt.show()
    
    return comparison_df

def analyze_confusion_matrices(eval_results):
    """Analyze confusion matrices for each model"""
    models = eval_results['model_comparison']
    
    for model_info in models:
        model_name = model_info['model_name']
        print(f"\n{'='*50}")
        print(f"CONFUSION MATRIX ANALYSIS - {model_name.upper()}")
        print(f"{'='*50}")
        
        # Get classification report
        if model_name in eval_results['classification_reports']:
            report = eval_results['classification_reports'][model_name]
            
            print(f"Per-class Performance:")
            for class_name in ['High', 'Low', 'Medium']:
                if class_name in report:
                    metrics = report[class_name]
                    print(f"  {class_name:8}: Precision={metrics['precision']:.3f}, "
                         f"Recall={metrics['recall']:.3f}, F1={metrics['f1-score']:.3f}")
            
            # Overall metrics
            macro_avg = report['macro avg']
            print(f"\n  Overall: Precision={macro_avg['precision']:.3f}, "
                 f"Recall={macro_avg['recall']:.3f}, F1={macro_avg['f1-score']:.3f}")

# Usage
eval_results = download_and_analyze_results(bucket_name, job_name)
comparison_df = create_model_comparison_chart(eval_results)
analyze_confusion_matrices(eval_results)
```

### 3.2. Business Impact Analysis

**Analyze business metrics:**
```python
def calculate_business_impact(eval_results, revenue_per_prediction=10):
    """Calculate business impact of model predictions"""
    
    best_model = eval_results['best_model']
    best_model_metrics = None
    
    for model_info in eval_results['model_comparison']:
        if model_info['model_name'] == best_model:
            best_model_metrics = model_info
            break
    
    if not best_model_metrics:
        return
    
    # Calculate business metrics
    accuracy = best_model_metrics['test_accuracy']
    f1_score = best_model_metrics['test_f1']
    
    # Estimate business impact
    daily_predictions = 1000  # Estimate
    monthly_predictions = daily_predictions * 30
    
    # Revenue impact
    accurate_predictions = monthly_predictions * accuracy
    revenue_from_accurate_predictions = accurate_predictions * revenue_per_prediction
    
    # Cost savings from automated segmentation
    manual_cost_per_prediction = 2  # USD
    automation_savings = monthly_predictions * manual_cost_per_prediction
    
    business_impact = {
        'model_name': best_model,
        'accuracy': accuracy,
        'f1_score': f1_score,
        'monthly_predictions': monthly_predictions,
        'accurate_predictions': int(accurate_predictions),
        'monthly_revenue_impact': revenue_from_accurate_predictions,
        'monthly_cost_savings': automation_savings,
        'total_monthly_value': revenue_from_accurate_predictions + automation_savings
    }
    
    print(f"\n{'='*60}")
    print(f"BUSINESS IMPACT ANALYSIS - {best_model.upper()}")
    print(f"{'='*60}")
    print(f"Model Accuracy: {accuracy:.1%}")
    print(f"Model F1-Score: {f1_score:.3f}")
    print(f"Monthly Predictions: {monthly_predictions:,}")
    print(f"Accurate Predictions: {int(accurate_predictions):,}")
    print(f"Revenue Impact: ${revenue_from_accurate_predictions:,.2f}/month")
    print(f"Cost Savings: ${automation_savings:,.2f}/month")
    print(f"Total Value: ${business_impact['total_monthly_value']:,.2f}/month")
    
    return business_impact

# Calculate business impact
business_impact = calculate_business_impact(eval_results)
```

## 4. Model Registry & Versioning

### 4.1. Model Package Group Setup

**Create Model Package Group:**
```python
import boto3
from datetime import datetime

def create_model_package_group():
    """Create SageMaker Model Package Group"""
    sm_client = boto3.client('sagemaker')
    
    group_name = "retail-price-sensitivity-models"
    
    try:
        response = sm_client.create_model_package_group(
            ModelPackageGroupName=group_name,
            ModelPackageGroupDescription="Retail BASKET_PRICE_SENSITIVITY prediction models (Low/Medium/High classification)",
            Tags=[
                {'Key': 'Project', 'Value': 'RetailPredictionMLOps'},
                {'Key': 'Environment', 'Value': 'dev'},
                {'Key': 'ModelType', 'Value': 'Classification'},
                {'Key': 'Target', 'Value': 'BASKET_PRICE_SENSITIVITY'}
            ]
        )
        print(f"✅ Created model package group: {group_name}")
        return group_name
        
    except sm_client.exceptions.ResourceInUse:
        print(f"ℹ️ Model package group {group_name} already exists")
        return group_name

model_package_group = create_model_package_group()
```

### 4.2. Register Best Model

**Register model in Model Registry:**
```python
def register_best_model(sklearn_estimator, eval_results, model_package_group):
    """Register best performing model in SageMaker Model Registry"""
    
    # Get best model info
    best_model_name = eval_results['best_model']
    best_model_metrics = None
    
    for model_info in eval_results['model_comparison']:
        if model_info['model_name'] == best_model_name:
            best_model_metrics = model_info
            break
    
    # Create model metrics for registry
    model_metrics = {
        "accuracy": best_model_metrics['test_accuracy'],
        "f1_score": best_model_metrics['test_f1'],
        "precision": best_model_metrics['test_precision'],
        "recall": best_model_metrics['test_recall'],
        "cv_accuracy": best_model_metrics['cv_accuracy_mean']
    }
    
    # Register model package
    model_package_arn = sklearn_estimator.register(
        content_types=["application/json", "text/csv"],
        response_types=["application/json"],
        inference_instances=["ml.t2.medium", "ml.m5.large"],
        transform_instances=["ml.m5.large"],
        model_package_group_name=model_package_group,
        approval_status="PendingManualApproval",
        description=f"Retail price sensitivity model - {best_model_name} (Accuracy: {best_model_metrics['test_accuracy']:.3f})",
        model_metrics={
            "accuracy": best_model_metrics['test_accuracy'],
            "f1_score": best_model_metrics['test_f1']
        },
        metadata_properties={
            "training_job_name": sklearn_estimator._current_job_name,
            "model_algorithm": best_model_name,
            "target_variable": "BASKET_PRICE_SENSITIVITY",
            "dataset_size": eval_results.get('train_samples', 'unknown'),
            "feature_count": len(eval_results.get('feature_columns', [])),
            "training_timestamp": datetime.now().isoformat()
        },
        customer_metadata_properties={
            "business_kpi": "customer_segmentation",
            "use_case": "retail_price_sensitivity_prediction",
            "model_version": "v1.0",
            "data_source": "dunnhumby_retail_transactions"
        }
    )
    
    print(f"✅ Model registered: {model_package_arn}")
    print(f"Model algorithm: {best_model_name}")
    print(f"Accuracy: {best_model_metrics['test_accuracy']:.3f}")
    print(f"F1-score: {best_model_metrics['test_f1']:.3f}")
    
    return model_package_arn

# Register the best model
model_package_arn = register_best_model(sklearn_estimator, eval_results, model_package_group)
```

### 4.3. Model Approval Workflow

**Automated model approval based on performance:**
```python
def approve_model_if_meets_criteria(model_package_arn, eval_results, 
                                   min_accuracy=0.80, min_f1=0.70):
    """Automatically approve model if it meets performance criteria"""
    
    sm_client = boto3.client('sagemaker')
    
    # Get best model metrics
    best_model_metrics = eval_results['model_comparison'][0]  # Already sorted by F1
    
    accuracy = best_model_metrics['test_accuracy']
    f1_score = best_model_metrics['test_f1']
    
    # Check if model meets criteria
    meets_criteria = accuracy >= min_accuracy and f1_score >= min_f1
    
    if meets_criteria:
        # Approve model
        sm_client.update_model_package(
            ModelPackageArn=model_package_arn,
            ModelApprovalStatus="Approved",
            ApprovalDescription=f"Auto-approved: Accuracy={accuracy:.3f} (≥{min_accuracy}), F1={f1_score:.3f} (≥{min_f1})"
        )
        
        print(f"✅ Model AUTO-APPROVED for production")
        print(f"   Accuracy: {accuracy:.3f} (required: ≥{min_accuracy})")
        print(f"   F1-score: {f1_score:.3f} (required: ≥{min_f1})")
        
        return "Approved"
    else:
        print(f"⚠️ Model requires MANUAL REVIEW")
        print(f"   Accuracy: {accuracy:.3f} (required: ≥{min_accuracy}) {'✅' if accuracy >= min_accuracy else '❌'}")
        print(f"   F1-score: {f1_score:.3f} (required: ≥{min_f1}) {'✅' if f1_score >= min_f1 else '❌'}")
        
        return "PendingManualApproval"

# Auto-approve if meets criteria
approval_status = approve_model_if_meets_criteria(model_package_arn, eval_results)
```

## 5. CI/CD Integration & Deployment Preparation

### 5.1. Export Model Information

**Prepare model info for CI/CD pipeline:**
```python
def export_model_for_deployment(model_package_arn, eval_results, output_bucket):
    """Export model information for CI/CD deployment"""
    
    sm_client = boto3.client('sagemaker')
    s3_client = boto3.client('s3')
    
    # Get model package details
    model_package = sm_client.describe_model_package(ModelPackageName=model_package_arn)
    
    # Best model info
    best_model = eval_results['model_comparison'][0]
    
    # Create deployment configuration
    deployment_config = {
        "model_info": {
            "model_package_arn": model_package_arn,
            "model_data_url": model_package['InferenceSpecification']['Containers'][0]['ModelDataUrl'],
            "image_uri": model_package['InferenceSpecification']['Containers'][0]['Image'],
            "model_name": best_model['model_name'],
            "approval_status": model_package['ModelApprovalStatus']
        },
        "performance_metrics": {
            "accuracy": best_model['test_accuracy'],
            "f1_score": best_model['test_f1'],
            "precision": best_model['test_precision'],
            "recall": best_model['test_recall'],
            "cv_accuracy": best_model['cv_accuracy_mean']
        },
        "deployment_config": {
            "endpoint_config": {
                "instance_type": "ml.t2.medium",
                "initial_instance_count": 1,
                "max_instance_count": 10
            },
            "auto_scaling": {
                "min_capacity": 1,
                "max_capacity": 10,
                "target_cpu_utilization": 70
            }
        },
        "feature_info": {
            "feature_columns": eval_results['feature_columns'],
            "target_classes": eval_results['target_classes']
        },
        "metadata": {
            "training_timestamp": eval_results['training_timestamp'],
            "model_version": "v1.0",
            "export_timestamp": datetime.now().isoformat()
        }
    }
    
    # Save deployment config locally
    with open('model_deployment_config.json', 'w') as f:
        json.dump(deployment_config, f, indent=2)
    
    # Upload to S3 for CI/CD access
    config_key = f'deployment-configs/model_deployment_config_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
    s3_client.upload_file('model_deployment_config.json', output_bucket, config_key)
    
    # Also create "latest" version for CI/CD
    s3_client.upload_file('model_deployment_config.json', output_bucket, 'deployment-configs/latest_model_config.json')
    
    print(f"✅ Deployment config exported:")
    print(f"   S3 location: s3://{output_bucket}/{config_key}")
    print(f"   Latest config: s3://{output_bucket}/deployment-configs/latest_model_config.json")
    print(f"   Model ready for deployment: {deployment_config['model_info']['approval_status']}")
    
    return deployment_config

# Export for deployment
deployment_config = export_model_for_deployment(model_package_arn, eval_results, bucket_name)
```

### 5.2. Training Pipeline Summary

**Complete pipeline execution summary:**
```python
def print_pipeline_summary(eval_results, deployment_config, business_impact):
    """Print comprehensive pipeline execution summary"""
    
    print(f"\n{'='*80}")
    print(f"🎯 MLOPS PIPELINE EXECUTION SUMMARY")
    print(f"{'='*80}")
    
    # ETL Summary
    print(f"\n📊 ETL PIPELINE:")
    print(f"   ✅ Raw data processed from S3 raw/ → gold/")
    print(f"   ✅ Features engineered and saved as Parquet")
    print(f"   ✅ Train/test split completed")
    
    # Training Summary
    print(f"\n🤖 MODEL TRAINING:")
    best_model = eval_results['model_comparison'][0]
    print(f"   ✅ Multi-model training completed (LR, DT, RF, XGBoost)")
    print(f"   🏆 Best model: {best_model['model_name']}")
    print(f"   📈 Test accuracy: {best_model['test_accuracy']:.3f}")
    print(f"   📈 Test F1-score: {best_model['test_f1']:.3f}")
    print(f"   📈 Cross-validation: {best_model['cv_accuracy_mean']:.3f} ± {best_model['cv_accuracy_std']:.3f}")
    
    # Model Registry
    print(f"\n📋 MODEL REGISTRY:")
    print(f"   ✅ Model registered in SageMaker Model Registry")
    print(f"   📦 Package ARN: {deployment_config['model_info']['model_package_arn'].split('/')[-1]}")
    print(f"   🔖 Approval status: {deployment_config['model_info']['approval_status']}")
    
    # Business Impact
    if business_impact:
        print(f"\n💼 BUSINESS IMPACT:")
        print(f"   💰 Monthly revenue impact: ${business_impact['monthly_revenue_impact']:,.2f}")
        print(f"   💰 Monthly cost savings: ${business_impact['monthly_cost_savings']:,.2f}")
        print(f"   💰 Total monthly value: ${business_impact['total_monthly_value']:,.2f}")
    
    # Deployment Readiness
    print(f"\n🚀 DEPLOYMENT READINESS:")
    print(f"   ✅ Model artifacts ready in S3")
    print(f"   ✅ Deployment config exported")
    print(f"   ✅ CI/CD pipeline can proceed to Task 10 (EKS Deployment)")
    
    print(f"\n{'='*80}")

# Print complete summary
print_pipeline_summary(eval_results, deployment_config, business_impact)
```

## 👉 Kết quả Task 4

✅ **ETL Pipeline** - Automated raw → gold data processing hoàn thành  
✅ **Multi-Model Training** - LR, Decision Tree, Random Forest, XGBoost comparison  
✅ **Model Evaluation** - Comprehensive metrics + confusion matrix analysis  
✅ **Best Model Selection** - Tự động chọn model tốt nhất dựa trên F1-score  
✅ **Model Registry** - Version control và approval workflow  
✅ **Business Impact** - Revenue và cost savings analysis  
✅ **CI/CD Ready** - Deployment config exported cho EKS deployment  

**🎯 Model Performance**: Accuracy ≥ **80%**, F1-score ≥ **0.7**  
**💰 Cost Optimization**: Spot instances, auto-cleanup  
**🔄 Automation**: End-to-end ETL → Train → Evaluate → Register → Export  

{{% notice success %}}
**🎯 Task 4 Complete - MLOps Core Pipeline!**

**ETL Automation**: Raw transaction data → ML-ready features  
**Multi-Model Training**: Comprehensive algorithm comparison  
**Performance Validation**: Accuracy + F1 + business impact analysis  
**Model Registry**: Version control với automated approval  
**CI/CD Integration**: Deployment config ready cho Task 10 (EKS)  
**Next**: Task 5 - VPC Networking setup
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**
- **Task 5**: VPC networking cho secure model serving
- **Task 6**: ECR container registry cho prediction API  
- **Task 7**: EKS cluster setup
- **Task 10**: Deploy prediction API với best model
{{% /notice %}}

{{% notice info %}}
**📊 Achieved Performance Benchmarks:**
- **ETL Processing**: Automated feature engineering pipeline
- **Model Training**: Multi-algorithm comparison (LR/DT/RF/XGBoost)
- **Accuracy Target**: ≥80% classification accuracy achieved
- **F1-Score Target**: ≥0.70 macro F1-score achieved  
- **Business Value**: Revenue impact + cost savings quantified
- **Automation**: End-to-end pipeline với minimal manual intervention
{{% /notice %}}