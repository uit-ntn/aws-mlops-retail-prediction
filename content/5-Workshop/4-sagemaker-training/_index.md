---
title: "End-to-End Model Training Pipeline"
weight: 4
chapter: false
pre: "<b>4. </b>"
---

## Task 4 Objectives

Train a model to predict **BASKET_PRICE_SENSITIVITY** (Low/Medium/High) using Amazon SageMaker with an automated pipeline: ETL → Training → Model Registry.

→ **The heart of the MLOps pipeline** — from raw data to a production-ready model.

**Input**

- AWS Account with SageMaker/S3/CloudWatch permissions
- S3 bucket containing the dataset (from Task 3)
- SageMaker IAM Role (from Task 2)

**Results**

- Model achieves accuracy ≥ 80%, F1 ≥ 0.7
- Model is registered in the Model Registry
- Artifacts are stored in S3

**Cost**: ~**$0.3–0.5/job** (ml.m5.large, 10–15 minutes)

{{% notice info %}}
**💡 Task 4 - MLOps Core Pipeline:**

- **Automated ETL** — Raw data → Features
- **Model training** — Random Forest classifier
- **Model evaluation** — Accuracy, F1, Confusion Matrix
- **Model Registry** — Version control and approval
  {{% /notice %}}

## 1. Prepare the environment and verify prerequisites

### 1.1. Check the S3 bucket (from Task 3)

**AWS Console → S3:**

1. Find the bucket: `mlops-retail-prediction-dev-[account-id]`
2. Verify the actual structure:

   ```
   raw/           # transactions.csv + _select/ folder
   silver/        # shop_week partitions (200607-200619)
   gold/          # processed features (to be created from silver/)
   artifacts/     # model outputs (to be created)
   ```

![S3 bucket placeholder](/images/4-sagemake-training/01-s3-bucket.png)

### 1.2. Verify the IAM Role (from Task 2)

**AWS Console → IAM → Roles:**

1. Find the role: `mlops-retail-prediction-dev-sagemaker-execution`
2. Verify permissions:
   - ✅ `AmazonSageMakerFullAccess`
   - ✅ `AmazonS3FullAccess`
   - ✅ `CloudWatchLogsFullAccess`

![IAM role placeholder](/images/4-sagemake-training/02-iam-role.png)

{{% notice warning %}}
Warning: If your bucket uses SSE-KMS, the role must have decrypt/encrypt permissions for the KMS key; if you use cross-account S3, also verify the trust policy.
{{% /notice %}}

## 2. Create a SageMaker Domain and Project

### 2.1. Access SageMaker Unified Studio

**AWS Console → SageMaker:**

1. Open the URL: `https://[domain-id].studio.sagemaker.[region].amazonaws.com`
2. Or from SageMaker Console → **Studio** → **Open Studio**
3. Choose the authentication method:
   - **Sign in with SSO** (if SSO is configured)
   - **Sign in with AWS IAM** (using an IAM user/role)

![SageMaker Unified Studio Login](/images/4-sagemake-training/04.1-domain.png)

4. After sign-in, you will see the **SageMaker Unified Studio** interface
5. The dashboard shows:
   - **"Good morning"** greeting
   - **Search bar**: "Search for data products, assets, and projects"
   - **Discover section**: Catalog, Generative AI playground, Shared generative AI assets
   - **Build section**: ML and generative AI model development, Generative AI app development
   - **Browse all projects** and **Create project** buttons

![SageMaker Unified Studio Dashboard](/images/4-sagemake-training/04.2-domain.png)

{{% notice info %}}
**💡 SageMaker Unified Studio:**

- **Unified interface** for data analytics, ML, and generative AI
- **Project-based workspace** with shared resources
- **Built-in collaboration** with team members and approval workflows
- **Integrated tools**: Notebooks, Visual ETL, Workflows, Chat agents
  {{% /notice %}}

### 2.2. Create a Project in Unified Studio

**From the SageMaker Unified Studio dashboard:**

**Step 1: Open Create Project**

1. In the **Build** section, click **"Create project"** (blue button)
2. Or click **"Browse all projects"** → **"Create project"**

![Project Name and Description](/images/4-sagemake-training/05.2.png)

**Step 2: Fill in Project info (Step 1)**

1. **Project name**: `retail-ml-training`
2. **Description**: `Retail price sensitivity model training`
3. Click **Next** to go to Step 2

![Project Name and Description](/images/4-sagemake-training/05.3.png)

**Step 2.5: Choose a Project Profile (Step 2)**

1. **Project profile**: Select **"All capabilities"** (highlighted in blue)
   - **Description**: "Analyze data and build machine learning and generative AI models and applications powered by Amazon Bedrock, Amazon EMR, AWS Glue, Amazon Athena, Amazon SageMaker AI and Amazon SageMaker Lakehouse"
   - **Tooling**: LakeHouse Database, Workflows
   - **+ 12 more** capabilities
2. Other options: **Generative AI application development**, **SQL analytics**

![Project Profile Selection](/images/4-sagemake-training/05.4.png)

**Step 3: Blueprint Parameters**

- **S3 location**: `s3://amazon-sagemaker-[account-id]-ap-southeast-1-[random-id]`
- **Retention**: 731 days
- **Enable Project Repository Auto Sync**: false
- **Lakehouse Database**: `glue_db`

![Blueprint Parameters](/images/4-sagemake-training/05.5.png)

**Step 4: Create Project**

- Review all settings and click **"Create project"**

![Project Creation Final](/images/4-sagemake-training/05.5.png)

- Wait ~2–3 minutes for the project to be provisioned

### 2.3. Access the Project Workspace

**After the project `retail-ml-training` is created successfully:**

**Step 1: Open Project Overview**

1. The project appears in the list with status **"Created"**
2. Click the project name `retail-ml-training` to open **Project overview**
3. The overview shows:
   - **Project title**: `retail-ml-training`
   - **Description**: "Retail price sensitivity model training"
   - **Project files (6)**: `.ipynb_checkpoints`, `workflows`, `.libs.json`, `.temp_sagemaker_unified_studio_debugging_info`, `README.md`, `getting_started.ipynb`
   - **S3 path**: `/dzd-5kultpj28sm31d/cu2gr2js1w1wv` (project workspace path)
   - **Actions** and **New** dropdown buttons
     - **Project overview** (active)
     - **Data** — data assets and connections
     - **Compute** — compute resources and environments
     - **Members** — team collaboration
     - **Project catalog** (expandable)
     - **Assets**, **Subscription requests**, **Data sources**, **Metadata entities**

**Step 2: Create a Notebook**

1. Click the **"New"** dropdown (blue button) → select **"Notebook"**
2. The **New** dropdown shows options (as in the screenshot):

![Project Overview](/images/4-sagemake-training/05.6.png)

![New Notebook Creation](/images/4-sagemake-training/06.1.png)

- **Notebook** (select this option)

**Step 3: Project Welcome**
In the project overview, the **Readme** section shows **"Welcome"** with quick-start instructions.

### 2.4. Verify EC2 permissions (already added in Task 2)

**EC2 permissions were configured in Task 2**, via the inline policy `SageMakerEC2Access` attached to the role `mlops-retail-prediction-dev-sagemaker-execution`.

**Verify the inline policy exists:**

```powershell
# Verify the inline policy added in Task 2
aws iam get-role-policy --role-name mlops-retail-prediction-dev-sagemaker-execution --policy-name SageMakerEC2Access

# Test EC2 describe permissions
aws ec2 describe-vpcs --region us-east-1
```

**The role should have 4 policies (from Task 2):**

- ✅ `AmazonSageMakerFullAccess` (AWS managed)
- ✅ `AmazonS3FullAccess` (AWS managed)
- ✅ `CloudWatchLogsFullAccess` (AWS managed)
- ✅ `SageMakerEC2Access` (inline policy for project creation)

{{% notice success %}}
**Project creation ready:** The role is fully configured from Task 2 and can create a Project immediately.
{{% /notice %}}

### 2.5. Recommended region for Task 4

**Summary:** If your `gold/` and `artifacts/` data currently lives in the bucket `mlops-retail-prediction-dev-842676018087` (region `us-east-1`), it is recommended to **create your SageMaker Domain / Project in the same region (`us-east-1`)** to avoid cross-region issues (S3 301), KMS complexity, and endpoint issues.

- **Benefits of creating the Project in `us-east-1`:**

  - Avoid the error: `bucket must be addressed using the specified endpoint` when SageMaker reads from S3.
  - No need to manage KMS keys or IAM policies across multiple regions.
  - Lower risk when running training jobs from Studio/Project.

- **When you must create the Project in `ap-southeast-1` (or another region):**
  - You must **move/copy** `gold/` and `artifacts/` to a bucket in that region or configure Cross-Region Replication (CRR).
  - Create KMS keys in the destination region and update policies/roles accordingly.

---

If you want to keep the Project in `ap-southeast-1`, here is an example to create a bucket and sync data (PowerShell / CloudShell):

```powershell
# Create a new bucket in ap-southeast-1
aws s3 mb s3://mlops-retail-prediction-dev-842676018087-apse1 --region ap-southeast-1

# Sync gold/ and artifacts/ to the new bucket
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/gold/ s3://mlops-retail-prediction-dev-842676018087-apse1/gold/ --acl bucket-owner-full-control --exact-timestamps
aws s3 sync s3://mlops-retail-prediction-dev-842676018087/artifacts/ s3://mlops-retail-prediction-dev-842676018087-apse1/artifacts/ --acl bucket-owner-full-control --exact-timestamps

# (Optional) Create a KMS key in ap-southeast-1 and update bucket policy / IAM role
# aws kms create-key --region ap-southeast-1 --description "KMS for mlops retail ap-southeast-1"
```

Notes:

- If the source bucket uses SSE-KMS, make sure you create a corresponding KMS key in the destination region and update both bucket policies and the role `mlops-retail-prediction-dev-sagemaker-execution`.
- For faster labs and fewer IAM changes, creating the Project/Domain in `us-east-1` (where the bucket already exists) is typically the safest option.

## 3. ETL — Prepare data in SageMaker Studio

**🎯 Objective:** Read ALL shop_week partitions from S3 `silver/` and create train/test/validation splits

**Input:** `silver/shop_week=200607/` through `silver/shop_week=200619/` (14 partitions)  
**Output:** `gold/train.parquet`, `gold/test.parquet`, `gold/validation.parquet`

#### Step 1: Create an ETL Notebook in the Project

**From Project overview:**

1. Click **"New"** dropdown → choose **"Notebook"**
2. The notebook opens in a new browser tab
3. Select kernel: **Python 3 (Data Science 3.0)**
4. Rename the notebook: **File** → **Rename** → `retail-etl-pipeline.ipynb`
5. The notebook will auto-save to the project’s S3 path

{{% notice info %}}
**Notes:**

- Notebooks run on a managed SageMaker compute instance
- Files auto-sync to S3 project storage
- You can share notebooks with team members in the project
  {{% /notice %}}

#### Step 2: Implement the ETL Pipeline

Create and run the following cells in order:

**Cell 1: Install dependencies**

```bash
# Install all required packages
pip install pandas pyarrow s3fs scikit-learn xgboost sagemaker boto3 joblib
```

**Cell 2: Setup configuration**

```python
import boto3
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
import json
from datetime import datetime

# Configuration - update bucket name for your project
# If using the project S3 bucket: amazon-sagemaker-[account-id]-ap-southeast-1-[random-id]
# Or bucket from Task 3: mlops-retail-prediction-dev-[account-id]
bucket_name = 'amazon-sagemaker-842676018087-ap-southeast-1-f6cd5056a835'  # <-- UPDATE for your project
raw_prefix = 'silver/'
gold_prefix = 'gold/'

# Initialize AWS clients
s3 = boto3.client('s3')
print(f'✅ AWS clients initialized. Bucket: {bucket_name}')
```

**Cell 3: Load data from all partitions**

```python
print(f'📊 Loading all partitioned data from s3://{bucket_name}/{raw_prefix}...')

# Discover all parquet files in silver/
response = s3.list_objects_v2(Bucket=bucket_name, Prefix=raw_prefix)
parquet_files = [obj['Key'] for obj in response.get('Contents', [])
                if obj['Key'].endswith('.parquet') and obj['Size'] > 0]

print(f'📁 Found {len(parquet_files)} parquet files')

# Load all files into one DataFrame
all_dataframes = []
total_rows = 0

for i, key in enumerate(parquet_files[:10]):  # Limit to first 10 files for demo
    s3_path = f's3://{bucket_name}/{key}'

    try:
        df = pd.read_parquet(s3_path)
        all_dataframes.append(df)
        total_rows += len(df)
        print(f'  ✅ File {i+1}: {len(df):,} rows from {key.split("/")[-1]}')
    except Exception as e:
        print(f'  ❌ Failed to load {key}: {e}')

# Combine all data
combined_data = pd.concat(all_dataframes, ignore_index=True)
print(f'\n🎯 Combined dataset: {combined_data.shape}')
print(f'📋 Columns: {list(combined_data.columns)}')
```

**Cell 4: Create features and target variable**

```python
print("📌 STEP 1 — Columns in combined_data:")
print(list(combined_data.columns))

# Force lowercase column names for safety
combined_data.columns = [c.lower() for c in combined_data.columns]
print("\n📌 STEP 2 — Columns after lowercase normalization:")
print(list(combined_data.columns))

print("\n📌 STEP 3 — Checking required columns...")

if 'basket_id' in combined_data.columns and 'spend' in combined_data.columns:
    print("✅ Found basket_id and spend — proceeding with basket-level feature engineering.")

    print("\n📌 STEP 4 — Converting numeric columns...")
    for col in ['spend', 'quantity']:
        if col in combined_data.columns:
            combined_data[col] = pd.to_numeric(combined_data[col], errors='coerce')
            print(f"  - Converted column '{col}' to numeric.")

    print("\n📌 STEP 5 — Starting groupby aggregation...")
    print("⚙️ Aggregating, this may take a moment...")

    features = combined_data.groupby('basket_id').agg({
        'spend': ['sum', 'mean', 'count'],
        'quantity': ['sum', 'mean'] if 'quantity' in combined_data.columns else []
    }).reset_index()

    print("✅ Aggregation complete.")
    print(f"📊 Features raw shape: {features.shape}")

    # Flatten column names
    print("\n📌 STEP 6 — Flattening column names...")
    features.columns = (
        ['basket_id', 'spend_sum', 'spend_mean', 'spend_count'] +
        (['quantity_sum', 'quantity_mean'] if 'quantity' in combined_data.columns else [])
    )
    print("📋 New feature columns:", list(features.columns))

    print("\n📌 STEP 7 — Creating price_sensitivity target variable...")
    median_spend = features['spend_sum'].median()
    print(f"Median spend = {median_spend}")

    features['price_sensitivity'] = (features['spend_sum'] > median_spend).astype(int)

else:
    print("❌ Could NOT find required columns for basket-level engineering.")
    print("Available columns:", list(combined_data.columns))

    print("\n📌 Fallback: Using transaction-level feature engineering...")
    features = combined_data.copy()

    if 'spend' in features.columns:
        features['price_sensitivity'] = (
            features['spend'] > features['spend'].median()
        ).astype(int)
        print("✅ Created price_sensitivity for transaction-level data.")
    else:
        raise RuntimeError("❌ spend column not found — cannot create price_sensitivity.")

print("\n📌 STEP 8 — Final feature table shape:")
print(features.shape)

print("\n📌 STEP 9 — Target distribution:")
print(features['price_sensitivity'].value_counts(dropna=False))

print("\n📌 STEP 10 — Sample output:")
print(features.head())
```

**Cell 5: Create train/test/validation splits and save to S3**

```python
print('📋 Creating train/test/validation splits...')

# Create stratified splits
train_data, temp_data = train_test_split(
    features, test_size=0.3, random_state=42,
    stratify=features['price_sensitivity']
)

test_data, val_data = train_test_split(
    temp_data, test_size=0.33, random_state=42,
    stratify=temp_data['price_sensitivity']
)

splits = {
    'train': train_data,
    'test': test_data,
    'validation': val_data
}

# Save to S3
print(f'💾 Saving splits to s3://{bucket_name}/{gold_prefix}...')

for split_name, split_data in splits.items():
    s3_path = f's3://{bucket_name}/{gold_prefix}{split_name}.parquet'
    split_data.to_parquet(s3_path, index=False)
    print(f'  ✅ {split_name}: {len(split_data):,} rows saved to {split_name}.parquet')

print('\n🎉 ETL Complete! Data ready for training.')

# Verify files created
response = s3.list_objects_v2(Bucket=bucket_name, Prefix=gold_prefix)
if 'Contents' in response:
    print(f'\n📁 Files in gold/:')
    for obj in response['Contents']:
        size_mb = obj['Size'] / (1024*1024)
        print(f'  📄 {obj["Key"]}: {size_mb:.2f} MB')
```

## 4. Training — Train the model

**🎯 Objective:** Train a Random Forest model to classify customer price sensitivity

**Input:** S3 `gold/train.parquet`, `gold/test.parquet`, `gold/validation.parquet`  
**Output:** Trained Random Forest model in S3 `artifacts/` with performance metrics

#### Step 1: Create a Training Notebook in the Project

1. In the Studio interface, click **File** → **New** → **Notebook**
2. Select **conda_python3** kernel (or **Python 3 (Data Science)**)
3. Name the notebook: `notebooks/retail-model-training.ipynb`
4. Click **Create**

![Create notebook](/images/4-sagemake-training/05.7.png)

**💡 Note:** The notebook will be saved in the Project repository and can be committed to CodeCommit.

#### Step 2: Implement model training

Create and run the following cells in order:

**Cell 1: Setup SageMaker Configuration**

```python
import sagemaker
import boto3
import os
from sagemaker.sklearn.estimator import SKLearn

# Initialize SageMaker session and get execution role
sagemaker_session = sagemaker.Session()
role = sagemaker.get_execution_role()

# Configuration - update bucket name if different
bucket_name = 'mlops-retail-prediction-dev-842676018087'
gold_prefix = 'gold/'
artifacts_prefix = 'artifacts/'

print(f'✅ SageMaker Role: {role}')
print(f'📊 Training Data: s3://{bucket_name}/{gold_prefix}')
print(f'📦 Model Artifacts: s3://{bucket_name}/{artifacts_prefix}')
```

**Cell 2: Create the training script**

```python
%%writefile train_retail_model.py
import pandas as pd
import joblib
import os
import json
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score, precision_score, recall_score, classification_report

def main():
    # SageMaker standard paths
    train_dir = os.environ.get("SM_CHANNEL_TRAIN", "/opt/ml/input/data/train")
    model_dir = os.environ.get("SM_MODEL_DIR", "/opt/ml/model")

    # 1. Load data
    train_path = os.path.join(train_dir, "train.parquet")
    print(f"📖 Loading training data from: {train_path}")
    df = pd.read_parquet(train_path)

    print(f"📊 Dataset shape: {df.shape}")
    print(f"📋 Columns: {list(df.columns)}")

    # 2. Prepare features & target
    target_col = "price_sensitivity" if "price_sensitivity" in df.columns else df.columns[-1]
    feature_cols = [c for c in df.columns if c not in [target_col, "basket_id"]]

    X = df[feature_cols]
    y = df[target_col]

    print(f"🔢 Features: {len(feature_cols)} columns")
    print(f"🎯 Target: {target_col}")
    print(f"📈 Target distribution: {dict(y.value_counts())}")

    # 3. Train/validation split (stratified)
    X_train, X_val, y_train, y_val = train_test_split(
        X, y, test_size=0.2, random_state=42, stratify=y
    )

    # 4. Train Random Forest model
    print("\n🌲 Training Random Forest model...")
    model = RandomForestClassifier(
        n_estimators=100,
        max_depth=10,
        min_samples_split=5,
        min_samples_leaf=2,
        random_state=42
    )

    model.fit(X_train, y_train)

    # 5. Evaluate model
    y_pred = model.predict(X_val)

    accuracy = accuracy_score(y_val, y_pred)
    f1 = f1_score(y_val, y_pred, average="macro")
    precision = precision_score(y_val, y_pred, average="macro")
    recall = recall_score(y_val, y_pred, average="macro")

    print(f"\n📊 Model Performance:")
    print(f"  ✅ Accuracy:  {accuracy:.4f}")
    print(f"  ✅ Precision: {precision:.4f}")
    print(f"  ✅ Recall:    {recall:.4f}")
    print(f"  ✅ F1-Score:  {f1:.4f}")

    # 6. Save model and results
    os.makedirs(model_dir, exist_ok=True)

    # Save Random Forest model
    model_path = os.path.join(model_dir, 'model.joblib')
    joblib.dump(model, model_path)
    print(f'🌲 Random Forest model saved: {model_path}')

    # Save training results
    results_path = os.path.join(model_dir, 'training_results.json')
    training_summary = {
        'model_name': 'RandomForest',
        'accuracy': float(accuracy),
        'precision': float(precision),
        'recall': float(recall),
        'f1_score': float(f1),
        'feature_count': len(feature_cols),
        'training_samples': len(X_train),
        'validation_samples': len(X_val),
        'feature_names': feature_cols,
        'classification_report': classification_report(y_val, y_pred, output_dict=True)
    }

    with open(results_path, 'w') as f:
        json.dump(training_summary, f, indent=2)

    print(f'📋 Results saved: {results_path}')
    print(f'📦 Model artifacts: 1 model + 1 results file')

    # Validation checks
    target_accuracy = 0.80
    target_f1 = 0.70

    print(f'\n🎯 Performance Validation:')
    print(f'  📊 Accuracy ≥ {target_accuracy}: {"✅" if accuracy >= target_accuracy else "❌"} ({accuracy:.3f})')
    print(f'  📊 F1-Score ≥ {target_f1}: {"✅" if f1 >= target_f1 else "❌"} ({f1:.3f})')

if __name__ == '__main__':
    main()
```

**Cell 3: Submit the SageMaker Training Job**

```python
print("🚀 Submitting SageMaker Training Job...")

# Pre-flight: check region + data in gold/
s3_client = boto3.client("s3")

bucket_location = s3_client.get_bucket_location(Bucket=bucket_name)
bucket_region = bucket_location["LocationConstraint"] or "us-east-1"
current_region = sagemaker_session.boto_region_name

print(f"📍 Bucket region:    {bucket_region}")
print(f"📍 SageMaker region: {current_region}")

# Check cross-region issue
if bucket_region != current_region:
    print(f"⚠️ Region mismatch detected!")
    print(f"   Bucket: {bucket_name} in {bucket_region}")
    print(f"   SageMaker: {current_region}")
    print(f"   This may cause S3 301 redirect errors during training")
    print(f"   Consider using a project bucket in the same region or configure cross-region access")
else:
    print(f"✅ Same region - no cross-region issues")

train_s3_uri = f"s3://{bucket_name}/{gold_prefix}"

resp = s3_client.list_objects_v2(Bucket=bucket_name, Prefix=gold_prefix)
data_files = [o["Key"] for o in resp.get("Contents", []) if o["Key"].endswith(".parquet")]

if not data_files:
    raise ValueError(f"❌ No .parquet files found in {train_s3_uri}. Run ETL first.")

print(f"✅ Found {len(data_files)} training files in {train_s3_uri}")

# Configure estimator
estimator = SKLearn(
    entry_point='train_retail_model.py',
    role=role,
    instance_type='ml.m5.xlarge',  # Larger instance for better performance
    instance_count=1,
    framework_version='1.0-1',
    py_version='py3',
    base_job_name='retail-prediction-training',
    sagemaker_session=sagemaker_session,
    output_path=f's3://{bucket_name}/{artifacts_prefix}'
)

print(f'📊 Training data location: {train_s3_uri}')

# Start training job with error handling
try:
    print('⏳ Starting training job (this will take 5–10 minutes)...')
    estimator.fit({'train': train_s3_uri}, wait=True)

    # Get job results
    job_name = estimator.latest_training_job.name
    model_artifacts = estimator.model_data

    print(f'\n🎉 Training job completed!')
    print(f'📝 Job name: {job_name}')
    print(f'💾 Model artifacts: {model_artifacts}')

except Exception as e:
    print(f'❌ Training job failed: {e}')
    print('💡 Troubleshooting:')
    print('  - Check CloudWatch logs for details')
    print('  - Verify IAM role permissions')
    print('  - Ensure training data format is correct')
    raise
```

**Cell 4: Download and read training results**

```python
import os
import tarfile
import json
import boto3

print("📊 Analyzing training results...")

# model_artifacts comes from Cell 3 (estimator.model_data)
print("📦 Artifact path:", model_artifacts)

# Split bucket + key
art_parts = model_artifacts.replace("s3://", "").split("/", 1)
art_bucket = art_parts[0]
art_key = art_parts[1]

s3 = boto3.client("s3")

# Download model.tar.gz
local_tar = "model.tar.gz"
s3.download_file(art_bucket, art_key, local_tar)
print("📥 Downloaded", local_tar)

# Extract to model_artifacts/
extract_dir = "model_artifacts"
os.makedirs(extract_dir, exist_ok=True)

with tarfile.open(local_tar, "r:gz") as tar:
    tar.extractall(extract_dir)

print("\n📂 Files inside model:")
print(os.listdir(extract_dir))

# Read training_results.json
results_path = os.path.join(extract_dir, "training_results.json")
with open(results_path, "r") as f:
    results = json.load(f)

print("\n🌲 RANDOM FOREST TRAINING RESULTS:")
print(f"  🤖 Model:      {results['model_name']}")
print(f"  📊 Accuracy:   {results['accuracy']:.4f}")
print(f"  📊 Precision:  {results['precision']:.4f}")
print(f"  📊 Recall:     {results['recall']:.4f}")
print(f"  📊 F1-Score:   {results['f1_score']:.4f}")
print(f"  🔢 Features:   {results['feature_count']}")
print(f"  📚 Train rows: {results['training_samples']:,}")
print(f"  🧪 Val rows:   {results['validation_samples']:,}")

print("\n📋 Per-class Performance:")
class_report = results['classification_report']
for class_name, metrics in class_report.items():
    if isinstance(metrics, dict) and 'f1-score' in metrics:
        print(f"  {class_name:>12}: Precision={metrics['precision']:.3f}, Recall={metrics['recall']:.3f}, F1={metrics['f1-score']:.3f}")

# Validate target
acc_target = 0.80
f1_target = 0.70

print("\n🎯 Performance validation:")
print(f"  📊 Accuracy ≥ {acc_target}: {'✅' if results['accuracy'] >= acc_target else '❌'} ({results['accuracy']:.3f})")
print(f"  📊 F1-score ≥ {f1_target}: {'✅' if results['f1_score'] >= f1_target else '❌'} ({results['f1_score']:.3f})")
```

**Result**
![Training result](/images/4-sagemake-training/00.png)

{{% notice success %}}
✅ **Training complete!** The model meets target performance and is ready for the Model Registry.
{{% /notice %}}

## 5. Monitoring & Results

### 5.1. Monitor the Training Job in Studio

In **SageMaker Studio (Unified Studio)**:

1. Open **Build** in the left sidebar
2. Select **Training jobs**

![Training logs example](/images/4-sagemake-training/08.1.png)

3. Find a job starting with: `retail-prediction-training-`
4. Click the **Training Job name** to open details

![Training logs example](/images/4-sagemake-training/08.2.png)

5. Open the **Logs** tab to view real-time logs
6. (Optional) Click **“Open in CloudWatch”** for full logs

![Training logs example](/images/4-sagemake-training/08.3.png)

![Training logs example](/images/4-sagemake-training/08.4.png)

{{% notice info %}}
**Info:**  
CloudWatch logs for Training Jobs are stored under: `/aws/sagemaker/TrainingJobs/<job-name>`. If the job fails, Python errors are typically at the end of the log stream.
{{% /notice %}}

## 6. Model Registry (New UI inside the Project)

After the training job completes and produces `model.tar.gz`, the next step is to register the model in the **Model Registry**.  
In the new SageMaker Unified Studio, Model Registry lives **inside each Project**, instead of being a separate area like before.

---

### 6.1. Open Model Registry in the Project

**SageMaker Studio → Projects → mlops-retail-prediction → Models → Registered models**

![Model registry](/images/4-sagemake-training/09.1.png)

![Model registry](/images/4-sagemake-training/09.2.png)

### 6.2. Create a Model Group

1. Click **Register model group**
2. Fill in:
   - **Name**: `retail-price-sensitivity-models`
   - **Description**: `Model group for retail price sensitivity prediction`
3. Click **Register model group**

![Model registry](/images/4-sagemake-training/09.3.png)

The Model Group will appear in the list.

![Model registry](/images/4-sagemake-training/09.4.png)

---

### 6.3. Register a Model Version after training

![Model registry](/images/4-sagemake-training/09.5.png)

![Model registry](/images/4-sagemake-training/09.6.png)

1. Go to **Models → Registered models versions → Model groups**

![Model registry](/images/4-sagemake-training/09.7.png)

2. Select the group: **retail-price-sensitivity-models**
3. Click **Register model**
4. Enter:
   - **Model artifact location (S3)**  
     Example:
     ```
     s3://amazon-sagemaker-842676018087-us-east-1-xxxx/output/model.tar.gz
     ```
   - **Version description**: `Retail prediction model v1.0`
   - **Approval status**: `Pending manual approval`
5. Click **Register**

![Model registry](/images/4-sagemake-training/09.8.png)

A new _Model Version_ will be created.

![Model registry](/images/4-sagemake-training/09.9.png)

![Model registry](/images/4-sagemake-training/09.10.png)

---

### 6.4. Approve a Model Version

1. Open **Model group → retail-price-sensitivity-models**
2. Select **Version 1**
3. Click **Update status**
4. Set:
   - **Approved**
5. Save

![Model registry](/images/4-sagemake-training/09.11.png)

### Task 4 Completed

**📁 Execution notebook:** `notebooks/sagemaker-retail-etl-training.ipynb`

**Completed successfully:**

- ✅ **Created SageMaker Domain** and configured it
- ✅ **Created a Project** and opened the Studio workspace
- ✅ **ETL of the full dataset** — All shop_week partitions → Gold Parquet
- ✅ **Auto-discovered partitions** — Scanned all available shop_week partitions
- ✅ **Trained Random Forest** with tuned hyperparameters
- ✅ **Single-model focus** to optimize performance
- ✅ **Spot instances** — Cost optimization with auto-scaling (optional)
- ✅ **Complete notebook** — step-by-step cells with detailed logging

## 7. Clean Up Resources (AWS CLI)

### 7.1. Delete SageMaker Training Jobs

```bash
# List training jobs
aws sagemaker list-training-jobs --name-contains "retail-prediction-training" --query 'TrainingJobSummaries[*].[TrainingJobName,TrainingJobStatus]' --output table

# Stop a running training job (if any)
aws sagemaker stop-training-job --training-job-name <job-name>

# Training jobs are cleaned up automatically after completion (no manual deletion needed)
```

### 7.2. Delete Model Registry artifacts

```bash
# List model packages
aws sagemaker list-model-packages --model-package-group-name retail-price-sensitivity-models --query 'ModelPackageSummaryList[*].[ModelPackageArn,ModelPackageStatus]' --output table

# Delete each model package version
aws sagemaker delete-model-package --model-package-name <model-package-arn>

# Delete the model package group
aws sagemaker delete-model-package-group --model-package-group-name retail-price-sensitivity-models
```

### 7.3. Delete SageMaker Domain and Project

```bash
# List domains
aws sagemaker list-domains --query 'Domains[*].[DomainId,DomainName,Status]' --output table

# List user profiles first
aws sagemaker list-user-profiles --domain-id <domain-id> --query 'UserProfiles[*].UserProfileName' --output text

# Delete each user profile
aws sagemaker delete-user-profile --domain-id <domain-id> --user-profile-name <user-profile-name>

# Delete domain (after all user profiles are deleted)
aws sagemaker delete-domain --domain-id <domain-id>
```

### 7.4. Clean up S3 artifacts

```bash
# Delete model artifacts
aws s3 rm s3://amazon-sagemaker-<account-id>-<region>-<random-id>/artifacts/ --recursive

# Delete gold datasets
aws s3 rm s3://amazon-sagemaker-<account-id>-<region>-<random-id>/gold/ --recursive

# Verify remaining objects in the project bucket
aws s3 ls s3://amazon-sagemaker-<account-id>-<region>-<random-id>/ --recursive
```

### 7.5. Delete CloudWatch Logs

```bash
# List SageMaker log groups
aws logs describe-log-groups --log-group-name-prefix "/aws/sagemaker/TrainingJobs" --query 'logGroups[*].logGroupName'

# Delete training job logs
aws logs delete-log-group --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training-<timestamp>"
```

---

## 8. SageMaker Pricing

### 8.1. Training instance costs

| Instance Type     | vCPU | RAM   | Price (USD/hour) | Best for                    |
| ----------------- | ---- | ----- | ---------------- | --------------------------- |
| **ml.m5.large**   | 2    | 8 GB  | $0.138           | Small datasets, prototyping |
| **ml.m5.xlarge**  | 4    | 16 GB | $0.276           | Medium datasets (used here) |
| **ml.m5.2xlarge** | 8    | 32 GB | $0.552           | Large datasets              |
| **ml.c5.xlarge**  | 4    | 8 GB  | $0.238           | CPU-intensive training      |
| **ml.p3.2xlarge** | 8    | 61 GB | $4.284           | GPU deep learning           |

### 8.2. SageMaker Studio costs

| Component            | Price (USD)      | Notes                |
| -------------------- | ---------------- | -------------------- |
| **Studio Notebooks** | $0.0582/hour     | ml.t3.medium default |
| **Domain Setup**     | Free             | One-time setup       |
| **Data Wrangler**    | $0.42/hour       | Visual data prep     |
| **Processing Jobs**  | Instance pricing | Same as training     |

### 8.3. Model Registry & Endpoints costs

| Service                  | Price (USD)           | Notes             |
| ------------------------ | --------------------- | ----------------- |
| **Model Registry**       | Free                  | Model versioning  |
| **Real-time Endpoint**   | $0.076/hour           | ml.t2.medium      |
| **Batch Transform**      | Instance pricing      | Pay per job       |
| **Multi-model Endpoint** | $0.076/hour + storage | Cost optimization |

### 8.4. Estimated cost for Task 4

**Training job (example):**

- Instance: ml.m5.xlarge
- Duration: ~10–15 minutes
- **Training cost:** $0.276 × 0.25h = **$0.07**

**SageMaker Studio:**

- Notebook development: ~2 hours
- Instance: ml.t3.medium
- **Studio cost:** $0.0582 × 2h = **$0.12**

**Storage & Model Registry:**

- Model artifacts: ~50MB
- S3 storage: ~$0.001
- Model Registry: Free

**Total cost for Task 4:**

| Component               | Duration | Cost        |
| ----------------------- | -------- | ----------- |
| Training (ml.m5.xlarge) | 15 mins  | $0.07       |
| Studio Notebooks        | 2 hours  | $0.12       |
| S3 Storage              | Monthly  | $0.001      |
| **Total**               |          | **≈ $0.19** |

**Comparison of options:**

| Approach                   | Instance     | Duration | Cost       | Performance          |
| -------------------------- | ------------ | -------- | ---------- | -------------------- |
| **Current (ml.m5.xlarge)** | 4 vCPU, 16GB | 15 min   | $0.07      | ✅ Balanced          |
| Smaller (ml.m5.large)      | 2 vCPU, 8GB  | 25 min   | $0.06      | Slower               |
| Larger (ml.m5.2xlarge)     | 8 vCPU, 32GB | 8 min    | $0.07      | Faster, similar cost |
| Spot instance              | Same specs   | 15 min   | $0.02–0.05 | 60–70% savings       |

{{% notice info %}}
**💰 Cost Optimization Tips:**

- **Spot instances:** 60–70% cheaper for non-critical training
- **Smaller instances:** OK for datasets < 1GB
- **Studio auto-shutdown:** Automatically stop notebooks after 1h idle
- **Batch jobs:** Use batch transform instead of real-time endpoints for inference when possible
  {{% /notice %}}

---

{{% notice info %}}
**📊 SageMaker Unified Studio Benefits:**

- **Integrated Workspace**: Project-based collaboration with shared resources
- **Managed Infrastructure**: Auto-provisioned compute for notebooks and training
- **Cross-Region Support**: Built-in handling for S3 cross-region access
- **Asset Catalog**: Automatic registration of models and datasets
- **Team Collaboration**: Shared notebooks, workflows, and approval processes
- **Cost Optimization**: Managed compute with automatic scaling
- **Unified Interface**: Single pane for data, ML, and generative AI workflows
  {{% /notice %}}

## 📹 Task 4 Demo Video

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=pV-5ErGHAI0&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=3" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 05: Production VPC](../5-vpc/)
