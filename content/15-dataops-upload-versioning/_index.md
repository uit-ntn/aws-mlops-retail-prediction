---
title: "DataOps - Upload & Versioning"
date: 2024-01-01T00:00:00Z
weight: 15
chapter: false
pre: "<b>15. </b>"
---

## Mục tiêu

Quản lý dữ liệu huấn luyện theo phiên bản (versioning), đảm bảo mỗi lần huấn luyện mô hình đều có thể truy vết được bộ dữ liệu đã sử dụng.

## Nội dung chính

### 1. Data Versioning Strategy

#### 1.1 S3 Data Organization Structure

```
retail-forecast-data-bucket/
├── datasets/
│   ├── v1.0/
│   │   ├── train.csv
│   │   ├── validation.csv
│   │   ├── test.csv
│   │   └── metadata.json
│   ├── v1.1/
│   │   ├── train.csv
│   │   ├── validation.csv
│   │   ├── test.csv
│   │   └── metadata.json
│   ├── 2024-01-15/
│   │   ├── train.csv
│   │   ├── validation.csv
│   │   └── metadata.json
│   └── commit-a1b2c3d/
│       ├── train.csv
│       ├── validation.csv
│       └── metadata.json
├── raw-data/
│   ├── 2024/01/15/
│   ├── 2024/01/16/
│   └── latest/
├── processed-data/
│   ├── feature-sets/
│   │   ├── v1.0/
│   │   └── v2.0/
│   └── schemas/
└── experiments/
    ├── exp-001/
    ├── exp-002/
    └── latest/
```

#### 1.2 Data Version Naming Conventions

```python
# data_versioning.py
from datetime import datetime
import hashlib
import json

class DataVersioning:
    def __init__(self, bucket_name):
        self.bucket_name = bucket_name
        self.base_prefix = "datasets"
    
    def generate_version_id(self, version_type="semantic", data_hash=None, git_commit=None):
        """Generate version ID based on different strategies"""
        
        if version_type == "semantic":
            # Manual semantic versioning (v1.0, v1.1, v2.0)
            return self.get_next_semantic_version()
        
        elif version_type == "timestamp":
            # Date-based versioning (2024-01-15)
            return datetime.now().strftime("%Y-%m-%d")
        
        elif version_type == "git_commit":
            # Git commit hash (commit-a1b2c3d)
            return f"commit-{git_commit[:7]}" if git_commit else None
        
        elif version_type == "content_hash":
            # Content-based hash (hash-abc123)
            return f"hash-{data_hash[:7]}" if data_hash else None
        
        elif version_type == "hybrid":
            # Combination (v1.0-2024-01-15-a1b2c3d)
            semantic = self.get_next_semantic_version()
            timestamp = datetime.now().strftime("%Y-%m-%d")
            commit = git_commit[:7] if git_commit else "unknown"
            return f"{semantic}-{timestamp}-{commit}"
    
    def get_next_semantic_version(self):
        """Get next semantic version based on existing versions"""
        # Implementation to check existing versions and increment
        return "v1.0"  # Simplified for example
```

### 2. Data Upload Pipeline

#### 2.1 Automated Data Upload Script

```python
# scripts/upload_data.py
import boto3
import pandas as pd
import json
import hashlib
import argparse
from datetime import datetime
import os
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataUploader:
    def __init__(self, bucket_name, aws_region='us-east-1'):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3', region_name=aws_region)
        self.versioning = DataVersioning(bucket_name)
    
    def calculate_data_hash(self, file_path):
        """Calculate hash of data file for content versioning"""
        hasher = hashlib.sha256()
        with open(file_path, 'rb') as f:
            for chunk in iter(lambda: f.read(4096), b""):
                hasher.update(chunk)
        return hasher.hexdigest()
    
    def validate_data_quality(self, file_path):
        """Validate data quality before upload"""
        try:
            df = pd.read_csv(file_path)
            
            validation_results = {
                'total_rows': len(df),
                'total_columns': len(df.columns),
                'missing_values': df.isnull().sum().to_dict(),
                'data_types': df.dtypes.to_dict(),
                'duplicates': df.duplicated().sum(),
                'validation_passed': True,
                'issues': []
            }
            
            # Check for critical issues
            if len(df) == 0:
                validation_results['validation_passed'] = False
                validation_results['issues'].append("Empty dataset")
            
            if df.isnull().all().any():
                validation_results['validation_passed'] = False
                validation_results['issues'].append("Columns with all null values")
            
            # Check for required columns (example for retail forecast)
            required_columns = ['store_id', 'product_id', 'date', 'sales']
            missing_columns = [col for col in required_columns if col not in df.columns]
            if missing_columns:
                validation_results['validation_passed'] = False
                validation_results['issues'].append(f"Missing required columns: {missing_columns}")
            
            return validation_results
            
        except Exception as e:
            return {
                'validation_passed': False,
                'issues': [f"Failed to validate data: {str(e)}"]
            }
    
    def create_metadata(self, file_path, version_id, validation_results, git_commit=None):
        """Create metadata for the dataset version"""
        file_stats = os.stat(file_path)
        
        metadata = {
            'version_id': version_id,
            'upload_timestamp': datetime.now().isoformat(),
            'file_size_bytes': file_stats.st_size,
            'file_hash': self.calculate_data_hash(file_path),
            'git_commit': git_commit,
            'validation_results': validation_results,
            'uploader': os.getenv('USER', 'unknown'),
            'source_file': os.path.basename(file_path),
            'schema_version': '1.0'
        }
        
        return metadata
    
    def upload_dataset(self, local_files, version_id, dataset_type='train', git_commit=None, 
                      dry_run=False, force_upload=False):
        """Upload dataset with versioning"""
        
        if not force_upload and self.version_exists(version_id):
            logger.warning(f"Version {version_id} already exists. Use --force to overwrite")
            return False
        
        s3_prefix = f"datasets/{version_id}"
        uploaded_files = []
        
        try:
            for local_file in local_files:
                if not os.path.exists(local_file):
                    logger.error(f"File not found: {local_file}")
                    continue
                
                # Validate data quality
                logger.info(f"Validating {local_file}...")
                validation_results = self.validate_data_quality(local_file)
                
                if not validation_results['validation_passed']:
                    logger.error(f"Data validation failed for {local_file}: {validation_results['issues']}")
                    if not force_upload:
                        continue
                
                # Create metadata
                metadata = self.create_metadata(local_file, version_id, validation_results, git_commit)
                
                # Determine S3 key
                file_name = os.path.basename(local_file)
                s3_key = f"{s3_prefix}/{file_name}"
                metadata_key = f"{s3_prefix}/metadata.json"
                
                if dry_run:
                    logger.info(f"DRY RUN: Would upload {local_file} to s3://{self.bucket_name}/{s3_key}")
                    continue
                
                # Upload data file
                logger.info(f"Uploading {local_file} to s3://{self.bucket_name}/{s3_key}")
                self.s3_client.upload_file(
                    local_file, 
                    self.bucket_name, 
                    s3_key,
                    ExtraArgs={
                        'ServerSideEncryption': 'aws:kms',
                        'SSEKMSKeyId': 'alias/retail-forecast-s3-key',
                        'Metadata': {
                            'version-id': version_id,
                            'upload-timestamp': metadata['upload_timestamp'],
                            'file-hash': metadata['file_hash']
                        }
                    }
                )
                
                uploaded_files.append({
                    'local_file': local_file,
                    's3_key': s3_key,
                    'metadata': metadata
                })
            
            # Upload consolidated metadata
            if uploaded_files and not dry_run:
                consolidated_metadata = {
                    'version_id': version_id,
                    'upload_timestamp': datetime.now().isoformat(),
                    'files': uploaded_files,
                    'total_files': len(uploaded_files)
                }
                
                metadata_json = json.dumps(consolidated_metadata, indent=2)
                self.s3_client.put_object(
                    Bucket=self.bucket_name,
                    Key=metadata_key,
                    Body=metadata_json,
                    ContentType='application/json',
                    ServerSideEncryption='aws:kms',
                    SSEKMSKeyId='alias/retail-forecast-s3-key'
                )
                
                logger.info(f"Upload completed. Version: {version_id}")
                logger.info(f"Metadata stored at: s3://{self.bucket_name}/{metadata_key}")
            
            return True
            
        except Exception as e:
            logger.error(f"Upload failed: {str(e)}")
            return False
    
    def version_exists(self, version_id):
        """Check if version already exists"""
        try:
            self.s3_client.head_object(
                Bucket=self.bucket_name,
                Key=f"datasets/{version_id}/metadata.json"
            )
            return True
        except:
            return False
    
    def list_versions(self):
        """List all available data versions"""
        try:
            response = self.s3_client.list_objects_v2(
                Bucket=self.bucket_name,
                Prefix="datasets/",
                Delimiter="/"
            )
            
            versions = []
            for prefix in response.get('CommonPrefixes', []):
                version_id = prefix['Prefix'].split('/')[-2]
                versions.append(version_id)
            
            return sorted(versions)
            
        except Exception as e:
            logger.error(f"Failed to list versions: {str(e)}")
            return []

def main():
    parser = argparse.ArgumentParser(description='Upload versioned datasets to S3')
    parser.add_argument('--files', nargs='+', required=True, help='Data files to upload')
    parser.add_argument('--bucket', required=True, help='S3 bucket name')
    parser.add_argument('--version-id', help='Version ID (auto-generated if not provided)')
    parser.add_argument('--version-type', choices=['semantic', 'timestamp', 'git_commit', 'hybrid'], 
                       default='timestamp', help='Version ID generation strategy')
    parser.add_argument('--git-commit', help='Git commit hash')
    parser.add_argument('--dry-run', action='store_true', help='Show what would be uploaded without actually doing it')
    parser.add_argument('--force', action='store_true', help='Force upload even if version exists')
    parser.add_argument('--list-versions', action='store_true', help='List existing versions')
    
    args = parser.parse_args()
    
    uploader = DataUploader(args.bucket)
    
    if args.list_versions:
        versions = uploader.list_versions()
        print("Available versions:")
        for version in versions:
            print(f"  - {version}")
        return
    
    # Generate version ID if not provided
    version_id = args.version_id
    if not version_id:
        versioning = DataVersioning(args.bucket)
        version_id = versioning.generate_version_id(
            version_type=args.version_type,
            git_commit=args.git_commit
        )
    
    logger.info(f"Using version ID: {version_id}")
    
    success = uploader.upload_dataset(
        local_files=args.files,
        version_id=version_id,
        git_commit=args.git_commit,
        dry_run=args.dry_run,
        force_upload=args.force
    )
    
    if success:
        print(f"Dataset uploaded successfully with version: {version_id}")
    else:
        print("Dataset upload failed")
        exit(1)

if __name__ == "__main__":
    main()
```

### 3. Data Retrieval and Usage

#### 3.1 Data Retrieval Script

```python
# scripts/get_data.py
import boto3
import json
import argparse
import pandas as pd
from datetime import datetime

class DataRetriever:
    def __init__(self, bucket_name, aws_region='us-east-1'):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3', region_name=aws_region)
    
    def get_latest_version(self):
        """Get the most recent data version"""
        versions = self.list_versions()
        if not versions:
            return None
        
        # Sort versions and return latest
        # This is simplified - in practice, you'd want more sophisticated version comparison
        return sorted(versions)[-1]
    
    def get_version_metadata(self, version_id):
        """Get metadata for a specific version"""
        try:
            response = self.s3_client.get_object(
                Bucket=self.bucket_name,
                Key=f"datasets/{version_id}/metadata.json"
            )
            
            metadata = json.loads(response['Body'].read().decode('utf-8'))
            return metadata
            
        except Exception as e:
            print(f"Failed to get metadata for version {version_id}: {str(e)}")
            return None
    
    def download_dataset(self, version_id, local_dir="./data", files=None):
        """Download dataset for a specific version"""
        metadata = self.get_version_metadata(version_id)
        if not metadata:
            return False
        
        import os
        os.makedirs(local_dir, exist_ok=True)
        
        downloaded_files = []
        
        for file_info in metadata.get('files', []):
            s3_key = file_info['s3_key']
            file_name = os.path.basename(s3_key)
            
            # Skip if specific files requested and this isn't one of them
            if files and file_name not in files:
                continue
            
            local_path = os.path.join(local_dir, file_name)
            
            try:
                print(f"Downloading {s3_key} to {local_path}")
                self.s3_client.download_file(
                    self.bucket_name,
                    s3_key,
                    local_path
                )
                downloaded_files.append(local_path)
                
            except Exception as e:
                print(f"Failed to download {s3_key}: {str(e)}")
                return False
        
        print(f"Downloaded {len(downloaded_files)} files to {local_dir}")
        return downloaded_files
    
    def get_data_uri(self, version_id, file_name=None):
        """Get S3 URI for training jobs"""
        if file_name:
            return f"s3://{self.bucket_name}/datasets/{version_id}/{file_name}"
        else:
            return f"s3://{self.bucket_name}/datasets/{version_id}/"
    
    def list_versions(self):
        """List all available versions"""
        try:
            response = self.s3_client.list_objects_v2(
                Bucket=self.bucket_name,
                Prefix="datasets/",
                Delimiter="/"
            )
            
            versions = []
            for prefix in response.get('CommonPrefixes', []):
                version_id = prefix['Prefix'].split('/')[-2]
                versions.append(version_id)
            
            return sorted(versions)
            
        except Exception as e:
            print(f"Failed to list versions: {str(e)}")
            return []
    
    def compare_versions(self, version1, version2):
        """Compare two data versions"""
        meta1 = self.get_version_metadata(version1)
        meta2 = self.get_version_metadata(version2)
        
        if not meta1 or not meta2:
            return None
        
        comparison = {
            'version1': version1,
            'version2': version2,
            'upload_time_diff': meta2['upload_timestamp'] - meta1['upload_timestamp'],
            'file_changes': [],
            'size_changes': {}
        }
        
        # Compare files
        files1 = {f['s3_key']: f for f in meta1.get('files', [])}
        files2 = {f['s3_key']: f for f in meta2.get('files', [])}
        
        for key in set(files1.keys()) | set(files2.keys()):
            if key in files1 and key in files2:
                if files1[key]['metadata']['file_hash'] != files2[key]['metadata']['file_hash']:
                    comparison['file_changes'].append(f"Modified: {key}")
            elif key in files1:
                comparison['file_changes'].append(f"Removed: {key}")
            else:
                comparison['file_changes'].append(f"Added: {key}")
        
        return comparison

def main():
    parser = argparse.ArgumentParser(description='Retrieve versioned datasets from S3')
    parser.add_argument('--bucket', required=True, help='S3 bucket name')
    parser.add_argument('--version-id', help='Version ID to retrieve')
    parser.add_argument('--latest', action='store_true', help='Get latest version')
    parser.add_argument('--download', action='store_true', help='Download files locally')
    parser.add_argument('--local-dir', default='./data', help='Local directory for downloads')
    parser.add_argument('--files', nargs='+', help='Specific files to download')
    parser.add_argument('--list-versions', action='store_true', help='List available versions')
    parser.add_argument('--metadata', action='store_true', help='Show version metadata')
    parser.add_argument('--uri', action='store_true', help='Get S3 URI for training')
    
    args = parser.parse_args()
    
    retriever = DataRetriever(args.bucket)
    
    if args.list_versions:
        versions = retriever.list_versions()
        print("Available versions:")
        for version in versions:
            print(f"  - {version}")
        return
    
    # Determine version to use
    version_id = args.version_id
    if args.latest or not version_id:
        version_id = retriever.get_latest_version()
        if not version_id:
            print("No versions available")
            return
        print(f"Using latest version: {version_id}")
    
    if args.metadata:
        metadata = retriever.get_version_metadata(version_id)
        if metadata:
            print(json.dumps(metadata, indent=2))
    
    if args.uri:
        uri = retriever.get_data_uri(version_id)
        print(f"S3 URI: {uri}")
    
    if args.download:
        files = retriever.download_dataset(
            version_id, 
            args.local_dir, 
            args.files
        )
        if files:
            print("Downloaded files:")
            for file in files:
                print(f"  - {file}")

if __name__ == "__main__":
    main()
```

### 4. Integration with SageMaker Training

#### 4.1 Updated Training Script with Data Versioning

```python
# training/train_versioned.py
import argparse
import os
import json
import boto3
import sagemaker
from sagemaker.sklearn import SKLearn

def create_training_job_with_versioned_data(
    job_name, 
    data_version, 
    model_version,
    bucket_name,
    role_arn,
    git_commit=None
):
    """Create SageMaker training job with versioned data"""
    
    sagemaker_client = boto3.client('sagemaker')
    
    # Construct data paths
    train_data_uri = f"s3://{bucket_name}/datasets/{data_version}/train.csv"
    validation_data_uri = f"s3://{bucket_name}/datasets/{data_version}/validation.csv"
    
    # Model output path with version
    output_path = f"s3://{bucket_name}/models/{model_version}/"
    
    training_job_config = {
        'TrainingJobName': job_name,
        'RoleArn': role_arn,
        'AlgorithmSpecification': {
            'TrainingImage': '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-scikit-learn:0.23-1-cpu-py3',
            'TrainingInputMode': 'File'
        },
        'InputDataConfig': [
            {
                'ChannelName': 'train',
                'DataSource': {
                    'S3DataSource': {
                        'S3DataType': 'S3Prefix',
                        'S3Uri': train_data_uri,
                        'S3DataDistributionType': 'FullyReplicated'
                    }
                },
                'ContentType': 'text/csv'
            },
            {
                'ChannelName': 'validation',
                'DataSource': {
                    'S3DataSource': {
                        'S3DataType': 'S3Prefix',
                        'S3Uri': validation_data_uri,
                        'S3DataDistributionType': 'FullyReplicated'
                    }
                },
                'ContentType': 'text/csv'
            }
        ],
        'OutputDataConfig': {
            'S3OutputPath': output_path
        },
        'ResourceConfig': {
            'InstanceType': 'ml.m5.large',
            'InstanceCount': 1,
            'VolumeSizeInGB': 30
        },
        'StoppingCondition': {
            'MaxRuntimeInSeconds': 3600
        },
        'HyperParameters': {
            'n-estimators': '100',
            'max-depth': '10',
            'data-version': data_version,
            'model-version': model_version
        },
        'Tags': [
            {'Key': 'DataVersion', 'Value': data_version},
            {'Key': 'ModelVersion', 'Value': model_version},
            {'Key': 'GitCommit', 'Value': git_commit or 'unknown'},
            {'Key': 'Project', 'Value': 'RetailForecast'}
        ]
    }
    
    # Add metadata about data version to training job
    metadata = {
        'data_version': data_version,
        'model_version': model_version,
        'git_commit': git_commit,
        'train_data_uri': train_data_uri,
        'validation_data_uri': validation_data_uri,
        'output_path': output_path
    }
    
    # Store metadata in S3 for lineage tracking
    s3_client = boto3.client('s3')
    metadata_key = f"training-metadata/{job_name}/lineage.json"
    s3_client.put_object(
        Bucket=bucket_name,
        Key=metadata_key,
        Body=json.dumps(metadata, indent=2),
        ContentType='application/json'
    )
    
    response = sagemaker_client.create_training_job(**training_job_config)
    
    print(f"Training job created: {job_name}")
    print(f"Data version: {data_version}")
    print(f"Model version: {model_version}")
    print(f"Lineage metadata: s3://{bucket_name}/{metadata_key}")
    
    return response

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--job-name', required=True)
    parser.add_argument('--data-version', required=True)
    parser.add_argument('--model-version', required=True)
    parser.add_argument('--bucket', required=True)
    parser.add_argument('--role-arn', required=True)
    parser.add_argument('--git-commit')
    
    args = parser.parse_args()
    
    create_training_job_with_versioned_data(
        args.job_name,
        args.data_version,
        args.model_version,
        args.bucket,
        args.role_arn,
        args.git_commit
    )

if __name__ == "__main__":
    main()
```

### 5. Data Lineage Tracking

#### 5.1 Lineage Tracking System

```python
# scripts/lineage_tracker.py
import boto3
import json
from datetime import datetime
import networkx as nx
import matplotlib.pyplot as plt

class DataLineageTracker:
    def __init__(self, bucket_name):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3')
        self.sagemaker_client = boto3.client('sagemaker')
    
    def create_lineage_record(self, data_version, model_version, training_job_name, git_commit=None):
        """Create lineage record linking data version to model version"""
        
        lineage_record = {
            'timestamp': datetime.now().isoformat(),
            'data_version': data_version,
            'model_version': model_version,
            'training_job_name': training_job_name,
            'git_commit': git_commit,
            'lineage_id': f"{data_version}-{model_version}-{training_job_name}"
        }
        
        # Store lineage record
        key = f"lineage/{lineage_record['lineage_id']}.json"
        self.s3_client.put_object(
            Bucket=self.bucket_name,
            Key=key,
            Body=json.dumps(lineage_record, indent=2),
            ContentType='application/json'
        )
        
        return lineage_record
    
    def get_data_lineage(self, model_version=None, data_version=None):
        """Get lineage information for models or data"""
        
        try:
            response = self.s3_client.list_objects_v2(
                Bucket=self.bucket_name,
                Prefix="lineage/"
            )
            
            lineage_records = []
            for obj in response.get('Contents', []):
                obj_response = self.s3_client.get_object(
                    Bucket=self.bucket_name,
                    Key=obj['Key']
                )
                record = json.loads(obj_response['Body'].read().decode('utf-8'))
                
                # Filter by model or data version if specified
                if model_version and record['model_version'] != model_version:
                    continue
                if data_version and record['data_version'] != data_version:
                    continue
                    
                lineage_records.append(record)
            
            return lineage_records
            
        except Exception as e:
            print(f"Failed to get lineage: {str(e)}")
            return []
    
    def trace_model_to_data(self, model_version):
        """Trace model back to its source data"""
        lineage = self.get_data_lineage(model_version=model_version)
        
        if not lineage:
            return None
        
        # For simplicity, assume one lineage record per model version
        record = lineage[0]
        
        # Get data metadata
        data_version = record['data_version']
        try:
            metadata_response = self.s3_client.get_object(
                Bucket=self.bucket_name,
                Key=f"datasets/{data_version}/metadata.json"
            )
            data_metadata = json.loads(metadata_response['Body'].read().decode('utf-8'))
            
            return {
                'model_version': model_version,
                'data_version': data_version,
                'training_job': record['training_job_name'],
                'git_commit': record.get('git_commit'),
                'data_metadata': data_metadata,
                'lineage_record': record
            }
            
        except Exception as e:
            print(f"Failed to get data metadata: {str(e)}")
            return record
    
    def generate_lineage_graph(self, output_file='lineage_graph.png'):
        """Generate visual lineage graph"""
        
        lineage_records = self.get_data_lineage()
        
        # Create directed graph
        G = nx.DiGraph()
        
        for record in lineage_records:
            data_node = f"Data\n{record['data_version']}"
            model_node = f"Model\n{record['model_version']}"
            training_node = f"Training\n{record['training_job_name']}"
            
            G.add_node(data_node, node_type='data')
            G.add_node(training_node, node_type='training')
            G.add_node(model_node, node_type='model')
            
            G.add_edge(data_node, training_node)
            G.add_edge(training_node, model_node)
        
        # Create layout
        pos = nx.spring_layout(G)
        
        # Draw graph
        plt.figure(figsize=(12, 8))
        
        # Draw nodes by type
        data_nodes = [n for n, d in G.nodes(data=True) if d.get('node_type') == 'data']
        training_nodes = [n for n, d in G.nodes(data=True) if d.get('node_type') == 'training']
        model_nodes = [n for n, d in G.nodes(data=True) if d.get('node_type') == 'model']
        
        nx.draw_networkx_nodes(G, pos, nodelist=data_nodes, node_color='lightblue', 
                              node_size=1500, alpha=0.7)
        nx.draw_networkx_nodes(G, pos, nodelist=training_nodes, node_color='lightgreen', 
                              node_size=1500, alpha=0.7)
        nx.draw_networkx_nodes(G, pos, nodelist=model_nodes, node_color='lightcoral', 
                              node_size=1500, alpha=0.7)
        
        nx.draw_networkx_edges(G, pos, alpha=0.5, arrows=True)
        nx.draw_networkx_labels(G, pos, font_size=8)
        
        plt.title("Data and Model Lineage")
        plt.axis('off')
        plt.tight_layout()
        plt.savefig(output_file)
        plt.show()
        
        print(f"Lineage graph saved to {output_file}")

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Track data and model lineage')
    parser.add_argument('--bucket', required=True, help='S3 bucket name')
    parser.add_argument('--action', choices=['create', 'trace', 'list', 'graph'], required=True)
    parser.add_argument('--data-version', help='Data version')
    parser.add_argument('--model-version', help='Model version')
    parser.add_argument('--training-job', help='Training job name')
    parser.add_argument('--git-commit', help='Git commit hash')
    
    args = parser.parse_args()
    
    tracker = DataLineageTracker(args.bucket)
    
    if args.action == 'create':
        if not all([args.data_version, args.model_version, args.training_job]):
            print("Create action requires --data-version, --model-version, and --training-job")
            return
        
        record = tracker.create_lineage_record(
            args.data_version,
            args.model_version,
            args.training_job,
            args.git_commit
        )
        print("Lineage record created:")
        print(json.dumps(record, indent=2))
    
    elif args.action == 'trace':
        if not args.model_version:
            print("Trace action requires --model-version")
            return
        
        lineage = tracker.trace_model_to_data(args.model_version)
        if lineage:
            print("Model lineage:")
            print(json.dumps(lineage, indent=2, default=str))
        else:
            print(f"No lineage found for model version: {args.model_version}")
    
    elif args.action == 'list':
        records = tracker.get_data_lineage(args.model_version, args.data_version)
        print(f"Found {len(records)} lineage records:")
        for record in records:
            print(f"  {record['data_version']} -> {record['model_version']} ({record['training_job_name']})")
    
    elif args.action == 'graph':
        tracker.generate_lineage_graph()

if __name__ == "__main__":
    main()
```

### 6. CI/CD Integration for DataOps

#### 6.1 Updated CI/CD Pipeline with Data Versioning

```groovy
// Updated Jenkinsfile with data versioning
stage('Data Management') {
    steps {
        script {
            // Check if data has changed
            def dataChanged = sh(
                script: "git diff --name-only HEAD~1 HEAD | grep '^data/' || true",
                returnStdout: true
            ).trim()
            
            if (dataChanged) {
                echo "Data changes detected: ${dataChanged}"
                
                // Generate data version
                env.DATA_VERSION = sh(
                    script: "python scripts/upload_data.py --bucket ${S3_DATA_BUCKET} --version-type hybrid --git-commit ${GIT_COMMIT} --dry-run | grep 'Using version ID' | cut -d: -f2 | xargs",
                    returnStdout: true
                ).trim()
                
                // Upload new data version
                sh '''
                    python scripts/upload_data.py \
                        --files data/train.csv data/validation.csv data/test.csv \
                        --bucket $S3_DATA_BUCKET \
                        --version-id $DATA_VERSION \
                        --git-commit $GIT_COMMIT
                '''
                
                // Force model retraining for new data
                env.RETRAIN_MODEL = "true"
                
            } else {
                // Use latest data version
                env.DATA_VERSION = sh(
                    script: "python scripts/get_data.py --bucket ${S3_DATA_BUCKET} --latest --uri | grep 'S3 URI' | cut -d: -f3- | xargs",
                    returnStdout: true
                ).trim()
                
                echo "Using existing data version: ${env.DATA_VERSION}"
            }
        }
    }
}

stage('Model Training with Lineage') {
    when {
        anyOf {
            environment name: 'RETRAIN_MODEL', value: 'true'
            params.RETRAIN_MODEL
        }
    }
    steps {
        script {
            // Create training job with versioned data
            sh '''
                python training/train_versioned.py \
                    --job-name "retail-forecast-training-${BUILD_VERSION}" \
                    --data-version $DATA_VERSION \
                    --model-version $BUILD_VERSION \
                    --bucket $S3_DATA_BUCKET \
                    --role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/SageMakerExecutionRole \
                    --git-commit $GIT_COMMIT
            '''
            
            // Create lineage record
            sh '''
                python scripts/lineage_tracker.py \
                    --bucket $S3_DATA_BUCKET \
                    --action create \
                    --data-version $DATA_VERSION \
                    --model-version $BUILD_VERSION \
                    --training-job "retail-forecast-training-${BUILD_VERSION}" \
                    --git-commit $GIT_COMMIT
            '''
        }
    }
}
```

### 7. Data Quality Monitoring

#### 7.1 Data Drift Detection

```python
# scripts/data_drift_detector.py
import pandas as pd
import numpy as np
from scipy import stats
import boto3
import json
from datetime import datetime

class DataDriftDetector:
    def __init__(self, bucket_name, reference_version):
        self.bucket_name = bucket_name
        self.reference_version = reference_version
        self.s3_client = boto3.client('s3')
    
    def download_data_version(self, version_id, file_name='train.csv'):
        """Download specific data version for comparison"""
        
        import tempfile
        import os
        
        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.csv')
        
        try:
            self.s3_client.download_file(
                self.bucket_name,
                f"datasets/{version_id}/{file_name}",
                temp_file.name
            )
            
            df = pd.read_csv(temp_file.name)
            return df
            
        finally:
            os.unlink(temp_file.name)
    
    def detect_statistical_drift(self, reference_df, current_df, threshold=0.05):
        """Detect statistical drift using Kolmogorov-Smirnov test"""
        
        drift_results = {
            'timestamp': datetime.now().isoformat(),
            'reference_version': self.reference_version,
            'drift_detected': False,
            'column_drifts': {},
            'overall_drift_score': 0.0
        }
        
        numeric_columns = reference_df.select_dtypes(include=[np.number]).columns
        
        p_values = []
        
        for column in numeric_columns:
            if column in current_df.columns:
                # Perform KS test
                ks_stat, p_value = stats.ks_2samp(
                    reference_df[column].dropna(),
                    current_df[column].dropna()
                )
                
                drift_detected = p_value < threshold
                
                drift_results['column_drifts'][column] = {
                    'ks_statistic': ks_stat,
                    'p_value': p_value,
                    'drift_detected': drift_detected,
                    'drift_magnitude': 'high' if p_value < 0.01 else 'medium' if p_value < 0.05 else 'low'
                }
                
                p_values.append(p_value)
                
                if drift_detected:
                    drift_results['drift_detected'] = True
        
        # Calculate overall drift score
        if p_values:
            drift_results['overall_drift_score'] = 1 - np.mean(p_values)
        
        return drift_results
    
    def detect_distribution_drift(self, reference_df, current_df):
        """Detect distribution changes"""
        
        distribution_changes = {}
        
        numeric_columns = reference_df.select_dtypes(include=[np.number]).columns
        
        for column in numeric_columns:
            if column in current_df.columns:
                ref_stats = {
                    'mean': reference_df[column].mean(),
                    'std': reference_df[column].std(),
                    'median': reference_df[column].median(),
                    'min': reference_df[column].min(),
                    'max': reference_df[column].max()
                }
                
                curr_stats = {
                    'mean': current_df[column].mean(),
                    'std': current_df[column].std(),
                    'median': current_df[column].median(),
                    'min': current_df[column].min(),
                    'max': current_df[column].max()
                }
                
                # Calculate percentage changes
                changes = {}
                for stat in ref_stats:
                    if ref_stats[stat] != 0:
                        changes[f'{stat}_change_pct'] = (
                            (curr_stats[stat] - ref_stats[stat]) / ref_stats[stat] * 100
                        )
                    else:
                        changes[f'{stat}_change_pct'] = 0
                
                distribution_changes[column] = {
                    'reference_stats': ref_stats,
                    'current_stats': curr_stats,
                    'changes': changes
                }
        
        return distribution_changes
    
    def check_drift(self, current_version, output_file=None):
        """Check for data drift between reference and current version"""
        
        # Download data
        reference_df = self.download_data_version(self.reference_version)
        current_df = self.download_data_version(current_version)
        
        # Detect drifts
        statistical_drift = self.detect_statistical_drift(reference_df, current_df)
        distribution_drift = self.detect_distribution_drift(reference_df, current_df)
        
        # Combine results
        drift_report = {
            'reference_version': self.reference_version,
            'current_version': current_version,
            'comparison_timestamp': datetime.now().isoformat(),
            'statistical_drift': statistical_drift,
            'distribution_changes': distribution_drift,
            'recommendations': self.generate_recommendations(statistical_drift, distribution_drift)
        }
        
        # Save report
        if output_file:
            with open(output_file, 'w') as f:
                json.dump(drift_report, f, indent=2)
        
        # Upload to S3
        report_key = f"drift-reports/{current_version}-vs-{self.reference_version}.json"
        self.s3_client.put_object(
            Bucket=self.bucket_name,
            Key=report_key,
            Body=json.dumps(drift_report, indent=2),
            ContentType='application/json'
        )
        
        return drift_report
    
    def generate_recommendations(self, statistical_drift, distribution_drift):
        """Generate recommendations based on drift detection"""
        
        recommendations = []
        
        if statistical_drift['drift_detected']:
            recommendations.append("Statistical drift detected. Consider retraining the model.")
            
            high_drift_columns = [
                col for col, info in statistical_drift['column_drifts'].items() 
                if info['drift_magnitude'] == 'high'
            ]
            
            if high_drift_columns:
                recommendations.append(f"High drift in columns: {', '.join(high_drift_columns)}")
        
        # Check for large distribution changes
        large_changes = []
        for column, info in distribution_drift.items():
            mean_change = abs(info['changes'].get('mean_change_pct', 0))
            if mean_change > 10:  # 10% change threshold
                large_changes.append(f"{column} (mean changed by {mean_change:.1f}%)")
        
        if large_changes:
            recommendations.append(f"Large distribution changes in: {', '.join(large_changes)}")
        
        if not recommendations:
            recommendations.append("No significant drift detected. Current model should perform well.")
        
        return recommendations

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Detect data drift between versions')
    parser.add_argument('--bucket', required=True, help='S3 bucket name')
    parser.add_argument('--reference-version', required=True, help='Reference data version')
    parser.add_argument('--current-version', required=True, help='Current data version to compare')
    parser.add_argument('--output', help='Output file for drift report')
    parser.add_argument('--threshold', type=float, default=0.05, help='P-value threshold for drift detection')
    
    args = parser.parse_args()
    
    detector = DataDriftDetector(args.bucket, args.reference_version)
    
    drift_report = detector.check_drift(args.current_version, args.output)
    
    print("Data Drift Analysis Complete")
    print(f"Drift detected: {drift_report['statistical_drift']['drift_detected']}")
    print(f"Overall drift score: {drift_report['statistical_drift']['overall_drift_score']:.3f}")
    print("\nRecommendations:")
    for rec in drift_report['recommendations']:
        print(f"  - {rec}")

if __name__ == "__main__":
    main()
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Data Versioning Structure**: S3 có cấu trúc versioning rõ ràng
- [ ] **Automated Upload**: Script upload data với version tự động
- [ ] **Data Validation**: Quality checks trước khi upload
- [ ] **Metadata Management**: Comprehensive metadata cho mỗi version
- [ ] **Training Integration**: SageMaker training job sử dụng versioned data
- [ ] **Lineage Tracking**: Mapping giữa data version và model version
- [ ] **Data Retrieval**: Easy access to specific data versions
- [ ] **Drift Detection**: Monitoring changes between data versions
- [ ] **CI/CD Integration**: Automated data versioning trong pipeline

### 📊 Verification Steps

1. **Dữ liệu huấn luyện trong S3 được lưu theo nhiều phiên bản, không bị ghi đè**
   ```bash
   # List all data versions
   python scripts/get_data.py --bucket retail-forecast-data-bucket --list-versions
   
   # Expected output: Multiple versions listed
   # v1.0, v1.1, 2024-01-15, commit-a1b2c3d, etc.
   
   # Check specific version structure
   aws s3 ls s3://retail-forecast-data-bucket/datasets/ --recursive
   ```

2. **Mỗi model trong Model Registry đều gắn được với dataset version cụ thể**
   ```bash
   # Check lineage for a model
   python scripts/lineage_tracker.py \
     --bucket retail-forecast-data-bucket \
     --action trace \
     --model-version v1.2
   
   # Expected: Shows data version, training job, git commit
   
   # List all lineage records
   python scripts/lineage_tracker.py \
     --bucket retail-forecast-data-bucket \
     --action list
   ```

3. **Quá trình training có thể dễ dàng tái hiện lại bằng cách chỉ định prefix dữ liệu**
   ```bash
   # Reproduce training with specific data version
   python training/train_versioned.py \
     --job-name "reproduce-training-$(date +%s)" \
     --data-version "2024-01-15" \
     --model-version "reproduce-v1.0" \
     --bucket retail-forecast-data-bucket \
     --role-arn arn:aws:iam::123456789012:role/SageMakerExecutionRole
   
   # Check if training uses correct data version
   aws sagemaker describe-training-job \
     --training-job-name "reproduce-training-XXXXX" \
     --query 'InputDataConfig[0].DataSource.S3DataSource.S3Uri'
   ```

### 🔍 Monitoring Commands

```bash
# Check data upload history
aws s3 ls s3://retail-forecast-data-bucket/datasets/ --recursive | grep metadata.json

# Monitor data drift
python scripts/data_drift_detector.py \
  --bucket retail-forecast-data-bucket \
  --reference-version v1.0 \
  --current-version v1.1 \
  --output drift-report.json

# Check lineage graph
python scripts/lineage_tracker.py \
  --bucket retail-forecast-data-bucket \
  --action graph

# Validate data version integrity
python scripts/get_data.py \
  --bucket retail-forecast-data-bucket \
  --version-id v1.0 \
  --metadata | jq '.validation_results'
```

## Best Practices Summary

### 📁 **Data Organization**
- Consistent versioning scheme across all datasets
- Immutable data versions (no overwrites)
- Comprehensive metadata for each version
- Clear folder structure and naming conventions

### 🔍 **Quality Assurance**
- Automated data validation before upload
- Data drift detection between versions
- Schema validation and consistency checks
- Content-based versioning for change detection

### 🔗 **Lineage & Traceability**
- Complete mapping between data and model versions
- Git commit integration for code-data synchronization
- Training job metadata with lineage information
- Visual lineage graphs for understanding relationships

### 🚀 **Automation & Integration**
- CI/CD pipeline integration for automated versioning
- Conditional model retraining based on data changes
- Automated lineage record creation
- Integration with existing MLOps tools

---

**Next Step**: [Task 16: Cost Optimization & Teardown](../16-cost-teardown/)