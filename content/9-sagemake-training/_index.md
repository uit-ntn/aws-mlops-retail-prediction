---
title: "Task 9: SageMaker Training & Model Registry"
date: 2024-01-01T00:00:00Z
weight: 9
chapter: false
pre: "<b>9. </b>"
---

## Mục tiêu

Huấn luyện mô hình machine learning trên Amazon SageMaker và đăng ký mô hình vào Model Registry để quản lý phiên bản.

## Nội dung chính

### 1. Chuẩn bị Training Script

#### 1.1 Tạo Training Script (train.py)

```python
import argparse
import os
import pandas as pd
import numpy as np
import joblib
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error, mean_absolute_error
from sklearn.model_selection import train_test_split
import json
import logging

# Thiết lập logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def parse_args():
    """Parse command line arguments"""
    parser = argparse.ArgumentParser()
    
    # SageMaker environment variables
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAIN'))
    parser.add_argument('--validation', type=str, default=os.environ.get('SM_CHANNEL_VALIDATION'))
    
    # Hyperparameters
    parser.add_argument('--n-estimators', type=int, default=100)
    parser.add_argument('--max-depth', type=int, default=10)
    parser.add_argument('--random-state', type=int, default=42)
    parser.add_argument('--test-size', type=float, default=0.2)
    
    return parser.parse_args()

def load_data(train_path, validation_path=None):
    """Load training and validation data"""
    logger.info(f"Loading training data from {train_path}")
    
    # Load training data
    train_files = [f for f in os.listdir(train_path) if f.endswith('.csv')]
    train_data = pd.concat([pd.read_csv(os.path.join(train_path, f)) for f in train_files])
    
    # Load validation data if provided
    validation_data = None
    if validation_path and os.path.exists(validation_path):
        logger.info(f"Loading validation data from {validation_path}")
        val_files = [f for f in os.listdir(validation_path) if f.endswith('.csv')]
        validation_data = pd.concat([pd.read_csv(os.path.join(validation_path, f)) for f in val_files])
    
    return train_data, validation_data

def preprocess_data(data):
    """Preprocess the data for training"""
    logger.info("Preprocessing data...")
    
    # Assume the target column is 'sales' and features are other columns
    if 'sales' not in data.columns:
        raise ValueError("Target column 'sales' not found in data")
    
    # Handle missing values
    data = data.fillna(data.mean(numeric_only=True))
    
    # Separate features and target
    X = data.drop(['sales'], axis=1)
    y = data['sales']
    
    # Select only numeric columns for simplicity
    X = X.select_dtypes(include=[np.number])
    
    logger.info(f"Data shape: {X.shape}, Target shape: {y.shape}")
    
    return X, y

def train_model(X_train, y_train, args):
    """Train the Random Forest model"""
    logger.info("Training Random Forest model...")
    
    model = RandomForestRegressor(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth,
        random_state=args.random_state,
        n_jobs=-1
    )
    
    model.fit(X_train, y_train)
    
    logger.info("Model training completed")
    return model

def evaluate_model(model, X_test, y_test):
    """Evaluate the trained model"""
    logger.info("Evaluating model...")
    
    y_pred = model.predict(X_test)
    
    mse = mean_squared_error(y_test, y_pred)
    mae = mean_absolute_error(y_test, y_pred)
    rmse = np.sqrt(mse)
    
    metrics = {
        'mse': float(mse),
        'mae': float(mae),
        'rmse': float(rmse)
    }
    
    logger.info(f"Model metrics: {metrics}")
    return metrics

def save_model(model, model_dir, metrics):
    """Save the trained model and metadata"""
    logger.info(f"Saving model to {model_dir}")
    
    # Save the model
    joblib.dump(model, os.path.join(model_dir, 'model.joblib'))
    
    # Save metrics
    with open(os.path.join(model_dir, 'metrics.json'), 'w') as f:
        json.dump(metrics, f)
    
    # Save feature names for inference
    feature_names = model.feature_names_in_.tolist() if hasattr(model, 'feature_names_in_') else []
    with open(os.path.join(model_dir, 'feature_names.json'), 'w') as f:
        json.dump(feature_names, f)
    
    logger.info("Model saved successfully")

def main():
    """Main training function"""
    args = parse_args()
    
    try:
        # Load data
        train_data, validation_data = load_data(args.train, args.validation)
        
        # Preprocess data
        X, y = preprocess_data(train_data)
        
        # Split data if validation data is not provided
        if validation_data is None:
            X_train, X_test, y_train, y_test = train_test_split(
                X, y, test_size=args.test_size, random_state=args.random_state
            )
        else:
            X_train, y_train = X, y
            X_test, y_test = preprocess_data(validation_data)
        
        # Train model
        model = train_model(X_train, y_train, args)
        
        # Evaluate model
        metrics = evaluate_model(model, X_test, y_test)
        
        # Save model
        save_model(model, args.model_dir, metrics)
        
        logger.info("Training completed successfully")
        
    except Exception as e:
        logger.error(f"Training failed: {str(e)}")
        raise

if __name__ == '__main__':
    main()
```

#### 1.2 Tạo Inference Script (inference.py)

```python
import joblib
import json
import numpy as np
import pandas as pd
import os

def model_fn(model_dir):
    """Load the model for inference"""
    model = joblib.load(os.path.join(model_dir, 'model.joblib'))
    return model

def input_fn(request_body, content_type):
    """Parse input data for inference"""
    if content_type == 'application/json':
        data = json.loads(request_body)
        return pd.DataFrame(data)
    elif content_type == 'text/csv':
        return pd.read_csv(request_body)
    else:
        raise ValueError(f"Unsupported content type: {content_type}")

def predict_fn(input_data, model):
    """Make predictions"""
    predictions = model.predict(input_data)
    return predictions.tolist()

def output_fn(prediction, accept):
    """Format the prediction output"""
    if accept == 'application/json':
        return json.dumps({'predictions': prediction})
    else:
        return str(prediction)
```

### 2. Tạo SageMaker Training Job

#### 2.1 Setup Training Environment

```python
import boto3
import sagemaker
from sagemaker.sklearn import SKLearn
from sagemaker.inputs import TrainingInput
from datetime import datetime
import json

# Initialize SageMaker session
sagemaker_session = sagemaker.Session()
role = sagemaker.get_execution_role()
region = sagemaker_session.boto_region_name

# Define S3 paths
bucket_name = 'retail-forecast-data-bucket'
data_prefix = 'training-data'
code_prefix = 'code'
model_prefix = 'models'

# S3 URIs
train_data_uri = f's3://{bucket_name}/{data_prefix}/train.csv'
validation_data_uri = f's3://{bucket_name}/{data_prefix}/validation.csv'
code_location = f's3://{bucket_name}/{code_prefix}'
output_path = f's3://{bucket_name}/{model_prefix}'

print(f"Training data: {train_data_uri}")
print(f"Model artifacts will be stored at: {output_path}")
```

#### 2.2 Upload Training Code

```python
# Upload training script to S3
import tarfile
import os

# Create a tar file with training code
with tarfile.open('sourcedir.tar.gz', 'w:gz') as tar:
    tar.add('train.py')
    tar.add('inference.py')

# Upload to S3
s3_client = boto3.client('s3')
s3_client.upload_file('sourcedir.tar.gz', bucket_name, f'{code_prefix}/sourcedir.tar.gz')
```

#### 2.3 Create and Start Training Job

```python
# Define hyperparameters
hyperparameters = {
    'n-estimators': 100,
    'max-depth': 10,
    'random-state': 42,
    'test-size': 0.2
}

# Create SKLearn estimator
sklearn_estimator = SKLearn(
    entry_point='train.py',
    source_dir='.',
    role=role,
    instance_type='ml.m5.large',
    instance_count=1,
    framework_version='1.0-1',
    py_version='py3',
    output_path=output_path,
    code_location=code_location,
    hyperparameters=hyperparameters,
    max_run=3600,  # 1 hour timeout
    use_spot_instances=True,
    max_wait=7200,  # 2 hours max wait
    checkpoint_s3_uri=f's3://{bucket_name}/checkpoints'
)

# Define input data
train_input = TrainingInput(train_data_uri, content_type='text/csv')
validation_input = TrainingInput(validation_data_uri, content_type='text/csv')

# Start training job
job_name = f"retail-forecast-training-{datetime.now().strftime('%Y-%m-%d-%H-%M-%S')}"

sklearn_estimator.fit(
    {
        'train': train_input,
        'validation': validation_input
    },
    job_name=job_name,
    wait=True
)

print(f"Training job completed: {job_name}")
print(f"Model artifacts: {sklearn_estimator.model_data}")
```

### 3. Monitor Training Job

#### 3.1 Check Training Status

```python
import time

def monitor_training_job(job_name):
    """Monitor training job progress"""
    sm_client = boto3.client('sagemaker')
    
    while True:
        response = sm_client.describe_training_job(TrainingJobName=job_name)
        status = response['TrainingJobStatus']
        
        print(f"Training job status: {status}")
        
        if status in ['Completed', 'Failed', 'Stopped']:
            break
            
        time.sleep(30)
    
    # Get final details
    if status == 'Completed':
        print("Training completed successfully!")
        print(f"Model artifacts: {response['ModelArtifacts']['S3ModelArtifacts']}")
        return response['ModelArtifacts']['S3ModelArtifacts']
    else:
        print(f"Training failed with status: {status}")
        if 'FailureReason' in response:
            print(f"Failure reason: {response['FailureReason']}")
        return None

# Monitor the training job
model_artifacts_uri = monitor_training_job(job_name)
```

#### 3.2 Get Training Metrics

```python
def get_training_metrics(job_name):
    """Extract training metrics from CloudWatch logs"""
    logs_client = boto3.client('logs')
    
    log_group = '/aws/sagemaker/TrainingJobs'
    log_stream = job_name
    
    try:
        response = logs_client.get_log_events(
            logGroupName=log_group,
            logStreamName=log_stream
        )
        
        metrics = {}
        for event in response['events']:
            message = event['message']
            if 'Model metrics:' in message:
                # Extract metrics from log message
                metrics_str = message.split('Model metrics: ')[1]
                metrics = json.loads(metrics_str)
                break
        
        return metrics
    except Exception as e:
        print(f"Could not retrieve metrics: {e}")
        return {}

training_metrics = get_training_metrics(job_name)
print(f"Training metrics: {training_metrics}")
```

### 4. Model Registry Registration

#### 4.1 Create Model Package Group

```python
model_package_group_name = "retail-forecast-model-group"

# Create model package group
sm_client = boto3.client('sagemaker')

try:
    response = sm_client.create_model_package_group(
        ModelPackageGroupName=model_package_group_name,
        ModelPackageGroupDescription="Retail forecast models for demand prediction"
    )
    print(f"Created model package group: {model_package_group_name}")
except sm_client.exceptions.ResourceInUse:
    print(f"Model package group {model_package_group_name} already exists")
```

#### 4.2 Register Model in Model Registry

```python
from sagemaker.model import Model
from sagemaker.model_metrics import MetricsSource, ModelMetrics
import uuid

# Create a unique model name
model_name = f"retail-forecast-model-{uuid.uuid4().hex[:8]}"

# Create model metrics
model_metrics = ModelMetrics(
    model_statistics=MetricsSource(
        s3_uri=f"{sklearn_estimator.model_data.replace('model.tar.gz', 'metrics.json')}",
        content_type="application/json"
    )
)

# Register model package
model_package_arn = sklearn_estimator.register(
    content_types=["text/csv", "application/json"],
    response_types=["application/json"],
    inference_instances=["ml.t2.medium", "ml.m5.large"],
    transform_instances=["ml.m5.large"],
    model_package_group_name=model_package_group_name,
    approval_status="PendingManualApproval",
    model_metrics=model_metrics,
    description="Retail demand forecasting model trained with Random Forest",
    metadata_properties={
        "training_job_name": job_name,
        "framework": "scikit-learn",
        "algorithm": "RandomForest"
    }
)

print(f"Model registered with ARN: {model_package_arn}")
```

#### 4.3 Approve Model for Production

```python
def approve_model(model_package_arn):
    """Approve model for production use"""
    sm_client = boto3.client('sagemaker')
    
    response = sm_client.update_model_package(
        ModelPackageArn=model_package_arn,
        ModelApprovalStatus="Approved",
        ApprovalDescription="Model approved for production deployment"
    )
    
    print(f"Model approved: {model_package_arn}")
    return response

# Approve the model (in practice, this would be done after validation)
approve_model(model_package_arn)
```

### 5. Model Retrieval for Deployment

#### 5.1 List Available Models

```python
def list_model_packages(model_package_group_name, status="Approved"):
    """List available model packages"""
    sm_client = boto3.client('sagemaker')
    
    response = sm_client.list_model_packages(
        ModelPackageGroupName=model_package_group_name,
        ModelApprovalStatus=status,
        SortBy='CreationTime',
        SortOrder='Descending'
    )
    
    return response['ModelPackageSummaryList']

# Get latest approved model
approved_models = list_model_packages(model_package_group_name)
if approved_models:
    latest_model = approved_models[0]
    print(f"Latest approved model: {latest_model['ModelPackageArn']}")
    print(f"Creation time: {latest_model['CreationTime']}")
```

#### 5.2 Get Model Details for Deployment

```python
def get_model_package_details(model_package_arn):
    """Get detailed information about a model package"""
    sm_client = boto3.client('sagemaker')
    
    response = sm_client.describe_model_package(
        ModelPackageName=model_package_arn
    )
    
    return {
        'model_package_arn': model_package_arn,
        'model_artifacts_uri': response['InferenceSpecification']['Containers'][0]['ModelDataUrl'],
        'image_uri': response['InferenceSpecification']['Containers'][0]['Image'],
        'instance_types': response['InferenceSpecification']['SupportedRealtimeInferenceInstanceTypes'],
        'content_types': response['InferenceSpecification']['SupportedContentTypes'],
        'response_types': response['InferenceSpecification']['SupportedResponseMIMETypes'],
        'creation_time': response['CreationTime'],
        'status': response['ModelPackageStatus']
    }

# Get details for deployment
if approved_models:
    model_details = get_model_package_details(latest_model['ModelPackageArn'])
    print(json.dumps(model_details, indent=2, default=str))
```

### 6. CI/CD Integration

#### 6.1 Export Model Information for Pipeline

```python
import json

def export_model_info_for_cicd(model_package_arn, output_file='model_info.json'):
    """Export model information for CI/CD pipeline"""
    
    model_details = get_model_package_details(model_package_arn)
    
    cicd_info = {
        'model_package_arn': model_package_arn,
        'model_artifacts_uri': model_details['model_artifacts_uri'],
        'image_uri': model_details['image_uri'],
        'timestamp': datetime.now().isoformat(),
        'deployment_config': {
            'instance_type': 'ml.t2.medium',
            'initial_instance_count': 1,
            'content_type': 'application/json'
        }
    }
    
    # Save to file
    with open(output_file, 'w') as f:
        json.dump(cicd_info, f, indent=2)
    
    # Upload to S3 for CI/CD access
    s3_client = boto3.client('s3')
    s3_client.upload_file(
        output_file, 
        bucket_name, 
        f'deployment-configs/{output_file}'
    )
    
    print(f"Model info exported to: s3://{bucket_name}/deployment-configs/{output_file}")
    
    return cicd_info

# Export model info for CI/CD
if approved_models:
    cicd_info = export_model_info_for_cicd(latest_model['ModelPackageArn'])
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Training Script**: Script huấn luyện (train.py) và inference (inference.py) được tạo
- [ ] **Training Job**: SageMaker training job chạy thành công với trạng thái "Completed"
- [ ] **Model Artifacts**: Model artifacts được lưu trong S3 bucket
- [ ] **Model Registry**: Mô hình được đăng ký vào SageMaker Model Registry
- [ ] **Model Approval**: Mô hình được approved cho production
- [ ] **Model Retrieval**: Pipeline có thể truy xuất model từ registry
- [ ] **Monitoring**: Training job được monitor và log metrics
- [ ] **CI/CD Integration**: Model info được export cho pipeline deployment

### 📊 Verification Steps

1. **Training job chạy thành công, trạng thái Completed**
   ```bash
   aws sagemaker describe-training-job --training-job-name <job-name>
   ```

2. **Model artifact được lưu trong bucket S3**
   ```bash
   aws s3 ls s3://retail-forecast-data-bucket/models/ --recursive
   ```

3. **Mô hình xuất hiện trong Model Registry với Model Package ARN**
   ```bash
   aws sagemaker list-model-packages --model-package-group-name retail-forecast-model-group
   ```

4. **Pipeline CI/CD có thể tham chiếu tới model version**
   ```bash
   aws s3 cp s3://retail-forecast-data-bucket/deployment-configs/model_info.json ./
   cat model_info.json
   ```

## Troubleshooting

### Common Issues

1. **Training Job Failed**
   ```bash
   # Check training job logs
   aws logs get-log-events \
     --log-group-name /aws/sagemaker/TrainingJobs \
     --log-stream-name <training-job-name>
   ```

2. **Model Registration Failed**
   - Verify model artifacts exist in S3
   - Check IAM permissions for SageMaker
   - Validate model package group exists

3. **Cannot Access Model in Registry**
   - Check model approval status
   - Verify model package ARN
   - Validate IAM permissions

### Useful Commands

```bash
# List all training jobs
aws sagemaker list-training-jobs --sort-by CreationTime --sort-order Descending

# Get model package details
aws sagemaker describe-model-package --model-package-name <model-package-arn>

# List model packages in group
aws sagemaker list-model-packages --model-package-group-name retail-forecast-model-group

# Check S3 model artifacts
aws s3 ls s3://retail-forecast-data-bucket/models/ --recursive --human-readable
```

---

**Next Step**: [Task 10: EKS Deployment](../10-eks-deployment/)