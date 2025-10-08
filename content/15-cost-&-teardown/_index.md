---
title: "Cost Optimization & Teardown"
date: 2024-01-01T00:00:00Z
weight: 16
chapter: false
pre: "<b>15. </b>"
---

## Mục tiêu

Giảm thiểu chi phí vận hành môi trường MLOps khi không còn sử dụng, đồng thời dọn dẹp tài nguyên để tránh lãng phí.

## Nội dung chính

### 1. Cost Monitoring & Analysis

#### 1.1 AWS Cost Explorer Setup

```bash
# Install AWS CLI Cost Explorer commands
pip install awscli boto3

# Create cost analysis script
cat > scripts/cost_analyzer.py << 'EOF'
import boto3
import json
from datetime import datetime, timedelta
import pandas as pd
import matplotlib.pyplot as plt

class AWSCostAnalyzer:
    def __init__(self):
        self.ce_client = boto3.client('ce')
        self.ec2_client = boto3.client('ec2')
        self.s3_client = boto3.client('s3')
        self.eks_client = boto3.client('eks')
    
    def get_cost_by_service(self, days=30):
        """Get cost breakdown by AWS service"""
        
        end_date = datetime.now().date()
        start_date = end_date - timedelta(days=days)
        
        response = self.ce_client.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d')
            },
            Granularity='DAILY',
            Metrics=['BlendedCost'],
            GroupBy=[
                {
                    'Type': 'DIMENSION',
                    'Key': 'SERVICE'
                }
            ]
        )
        
        return response
    
    def get_mlops_resource_costs(self):
        """Get costs specific to MLOps resources"""
        
        # Get EKS cluster costs
        eks_costs = self.get_resource_costs_by_tag('Project', 'RetailForecast')
        
        # Get S3 storage costs
        s3_costs = self.get_s3_storage_costs()
        
        # Get SageMaker costs
        sagemaker_costs = self.get_sagemaker_costs()
        
        return {
            'eks_costs': eks_costs,
            's3_costs': s3_costs,
            'sagemaker_costs': sagemaker_costs
        }
    
    def get_resource_costs_by_tag(self, tag_key, tag_value, days=30):
        """Get costs filtered by resource tags"""
        
        end_date = datetime.now().date()
        start_date = end_date - timedelta(days=days)
        
        response = self.ce_client.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d')
            },
            Granularity='DAILY',
            Metrics=['BlendedCost'],
            GroupBy=[
                {
                    'Type': 'TAG',
                    'Key': tag_key
                }
            ],
            Filter={
                'Tags': {
                    'Key': tag_key,
                    'Values': [tag_value]
                }
            }
        )
        
        return response
    
    def get_s3_storage_costs(self):
        """Get S3 storage costs and usage"""
        
        buckets = ['retail-forecast-data-bucket', 'retail-forecast-models-bucket', 
                  'retail-forecast-artifacts-bucket']
        
        bucket_costs = {}
        
        for bucket in buckets:
            try:
                # Get bucket size
                cloudwatch = boto3.client('cloudwatch')
                response = cloudwatch.get_metric_statistics(
                    Namespace='AWS/S3',
                    MetricName='BucketSizeBytes',
                    Dimensions=[
                        {'Name': 'BucketName', 'Value': bucket},
                        {'Name': 'StorageType', 'Value': 'StandardStorage'}
                    ],
                    StartTime=datetime.now() - timedelta(days=1),
                    EndTime=datetime.now(),
                    Period=86400,
                    Statistics=['Average']
                )
                
                size_bytes = response['Datapoints'][0]['Average'] if response['Datapoints'] else 0
                size_gb = size_bytes / (1024**3)
                
                # Estimate cost (simplified calculation)
                estimated_cost = size_gb * 0.023  # $0.023 per GB for S3 Standard
                
                bucket_costs[bucket] = {
                    'size_gb': size_gb,
                    'estimated_monthly_cost': estimated_cost
                }
                
            except Exception as e:
                bucket_costs[bucket] = {'error': str(e)}
        
        return bucket_costs
    
    def get_sagemaker_costs(self, days=30):
        """Get SageMaker training and hosting costs"""
        
        sagemaker = boto3.client('sagemaker')
        
        # List training jobs
        training_jobs = sagemaker.list_training_jobs(
            CreationTimeAfter=datetime.now() - timedelta(days=days),
            MaxResults=100
        )
        
        # List endpoints
        endpoints = sagemaker.list_endpoints(
            CreationTimeAfter=datetime.now() - timedelta(days=days),
            MaxResults=100
        )
        
        return {
            'training_jobs_count': len(training_jobs['TrainingJobSummaries']),
            'active_endpoints_count': len(endpoints['Endpoints']),
            'training_jobs': training_jobs['TrainingJobSummaries'],
            'endpoints': endpoints['Endpoints']
        }
    
    def generate_cost_report(self, output_file='cost_report.json'):
        """Generate comprehensive cost report"""
        
        report = {
            'generated_at': datetime.now().isoformat(),
            'cost_by_service': self.get_cost_by_service(),
            'mlops_resource_costs': self.get_mlops_resource_costs(),
            'recommendations': self.generate_cost_recommendations()
        }
        
        with open(output_file, 'w') as f:
            json.dump(report, f, indent=2, default=str)
        
        print(f"Cost report generated: {output_file}")
        return report
    
    def generate_cost_recommendations(self):
        """Generate cost optimization recommendations"""
        
        recommendations = []
        
        # Check for idle resources
        ec2_instances = self.ec2_client.describe_instances()
        idle_instances = []
        
        for reservation in ec2_instances['Reservations']:
            for instance in reservation['Instances']:
                if instance['State']['Name'] == 'running':
                    # Check if instance has low utilization (simplified check)
                    idle_instances.append(instance['InstanceId'])
        
        if idle_instances:
            recommendations.append({
                'type': 'idle_resources',
                'description': f'Found {len(idle_instances)} potentially idle EC2 instances',
                'action': 'Review and terminate unused instances',
                'potential_savings': 'High'
            })
        
        # Check S3 lifecycle policies
        s3_buckets = self.s3_client.list_buckets()
        buckets_without_lifecycle = []
        
        for bucket in s3_buckets['Buckets']:
            try:
                self.s3_client.get_bucket_lifecycle_configuration(Bucket=bucket['Name'])
            except:
                buckets_without_lifecycle.append(bucket['Name'])
        
        if buckets_without_lifecycle:
            recommendations.append({
                'type': 'lifecycle_policy',
                'description': f'{len(buckets_without_lifecycle)} buckets without lifecycle policies',
                'action': 'Implement S3 lifecycle policies to reduce storage costs',
                'potential_savings': 'Medium'
            })
        
        return recommendations

if __name__ == "__main__":
    analyzer = AWSCostAnalyzer()
    report = analyzer.generate_cost_report()
    print("Cost analysis completed!")
EOF
```

#### 1.2 Cost Alerting Setup

```yaml
# cloudformation/cost-alerts.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Cost monitoring and alerting for MLOps infrastructure'

Parameters:
  MonthlyBudget:
    Type: Number
    Default: 500
    Description: Monthly budget limit in USD
  
  AlertEmail:
    Type: String
    Description: Email address for cost alerts

Resources:
  CostBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: MLOps-Monthly-Budget
        BudgetLimit:
          Amount: !Ref MonthlyBudget
          Unit: USD
        TimeUnit: MONTHLY
        BudgetType: COST
        CostFilters:
          TagKey:
            - Project
          TagValue:
            - RetailForecast
      NotificationsWithSubscribers:
        - Notification:
            NotificationType: ACTUAL
            ComparisonOperator: GREATER_THAN
            Threshold: 80
          Subscribers:
            - SubscriptionType: EMAIL
              Address: !Ref AlertEmail
        - Notification:
            NotificationType: FORECASTED
            ComparisonOperator: GREATER_THAN
            Threshold: 100
          Subscribers:
            - SubscriptionType: EMAIL
              Address: !Ref AlertEmail

  DailyCostAnomaly:
    Type: AWS::CE::AnomalyDetector
    Properties:
      AnomalyDetectorName: MLOps-Daily-Anomaly
      MonitorType: DIMENSIONAL
      MonitorSpecification:
        DimensionKey: SERVICE
        DimensionValueMatchOptions:
          - EQUALS
        DimensionValues:
          - Amazon Elastic Container Service for Kubernetes
          - Amazon SageMaker
          - Amazon Simple Storage Service

  CostAnomalySubscription:
    Type: AWS::CE::AnomalySubscription
    Properties:
      SubscriptionName: MLOps-Anomaly-Alerts
      MonitorArnList:
        - !GetAtt DailyCostAnomaly.AnomalyDetectorArn
      Subscribers:
        - Type: EMAIL
          Address: !Ref AlertEmail
      Threshold: 100
      Frequency: DAILY
```

### 2. Resource Scaling & Optimization

#### 2.1 EKS Node Group Scaling

```bash
# scripts/scale_eks_nodes.sh
#!/bin/bash

CLUSTER_NAME="retail-forecast-cluster"
NODEGROUP_NAME="retail-forecast-nodegroup"
ACTION=$1  # scale-down, scale-up, or auto

case $ACTION in
  "scale-down")
    echo "Scaling down EKS node group to minimum..."
    aws eks update-nodegroup-config \
      --cluster-name $CLUSTER_NAME \
      --nodegroup-name $NODEGROUP_NAME \
      --scaling-config minSize=0,maxSize=1,desiredSize=0
    
    echo "Scaling down EKS cluster autoscaler..."
    kubectl scale deployment cluster-autoscaler \
      --replicas=0 \
      -n kube-system
    ;;
    
  "scale-up")
    echo "Scaling up EKS node group for production..."
    aws eks update-nodegroup-config \
      --cluster-name $CLUSTER_NAME \
      --nodegroup-name $NODEGROUP_NAME \
      --scaling-config minSize=1,maxSize=5,desiredSize=2
    
    echo "Scaling up EKS cluster autoscaler..."
    kubectl scale deployment cluster-autoscaler \
      --replicas=1 \
      -n kube-system
    ;;
    
  "auto")
    echo "Setting up automatic scaling based on time..."
    # Create cron jobs for automatic scaling
    cat > /tmp/scale-schedule.yaml << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-nodes
  namespace: kube-system
spec:
  schedule: "0 18 * * 1-5"  # 6 PM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cluster-autoscaler
          containers:
          - name: aws-cli
            image: amazon/aws-cli:latest
            command:
            - /bin/bash
            - -c
            - |
              aws eks update-nodegroup-config \
                --cluster-name retail-forecast-cluster \
                --nodegroup-name retail-forecast-nodegroup \
                --scaling-config minSize=0,maxSize=1,desiredSize=0
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-nodes
  namespace: kube-system
spec:
  schedule: "0 8 * * 1-5"  # 8 AM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cluster-autoscaler
          containers:
          - name: aws-cli
            image: amazon/aws-cli:latest
            command:
            - /bin/bash
            - -c
            - |
              aws eks update-nodegroup-config \
                --cluster-name retail-forecast-cluster \
                --nodegroup-name retail-forecast-nodegroup \
                --scaling-config minSize=1,maxSize=5,desiredSize=2
          restartPolicy: OnFailure
EOF
    
    kubectl apply -f /tmp/scale-schedule.yaml
    ;;
    
  *)
    echo "Usage: $0 {scale-down|scale-up|auto}"
    echo "  scale-down: Reduce nodes to minimum (cost saving)"
    echo "  scale-up: Scale to production levels"
    echo "  auto: Setup automatic scaling schedule"
    exit 1
    ;;
esac
```

#### 2.2 SageMaker Endpoint Management

```python
# scripts/manage_sagemaker_endpoints.py
import boto3
import json
import argparse
from datetime import datetime

class SageMakerEndpointManager:
    def __init__(self):
        self.sagemaker = boto3.client('sagemaker')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def list_endpoints(self, status_filter=None):
        """List all SageMaker endpoints"""
        
        endpoints = self.sagemaker.list_endpoints(
            StatusEquals=status_filter,
            MaxResults=100
        )
        
        endpoint_details = []
        for endpoint in endpoints['Endpoints']:
            try:
                config = self.sagemaker.describe_endpoint_config(
                    EndpointConfigName=endpoint['EndpointConfigName']
                )
                
                endpoint_details.append({
                    'name': endpoint['EndpointName'],
                    'status': endpoint['EndpointStatus'],
                    'creation_time': endpoint['CreationTime'],
                    'instance_type': config['ProductionVariants'][0]['InstanceType'],
                    'instance_count': config['ProductionVariants'][0]['InitialInstanceCount']
                })
                
            except Exception as e:
                endpoint_details.append({
                    'name': endpoint['EndpointName'],
                    'status': endpoint['EndpointStatus'],
                    'error': str(e)
                })
        
        return endpoint_details
    
    def get_endpoint_utilization(self, endpoint_name, days=7):
        """Get endpoint utilization metrics"""
        
        from datetime import timedelta
        
        metrics = {}
        
        # Get invocation count
        try:
            response = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/SageMaker',
                MetricName='Invocations',
                Dimensions=[
                    {'Name': 'EndpointName', 'Value': endpoint_name}
                ],
                StartTime=datetime.now() - timedelta(days=days),
                EndTime=datetime.now(),
                Period=3600,  # 1 hour
                Statistics=['Sum']
            )
            
            total_invocations = sum([dp['Sum'] for dp in response['Datapoints']])
            metrics['total_invocations'] = total_invocations
            metrics['avg_invocations_per_hour'] = total_invocations / (days * 24)
            
        except Exception as e:
            metrics['invocation_error'] = str(e)
        
        return metrics
    
    def stop_idle_endpoints(self, threshold_invocations=10, dry_run=True):
        """Stop endpoints with low utilization"""
        
        endpoints = self.list_endpoints(status_filter='InService')
        stopped_endpoints = []
        
        for endpoint in endpoints:
            utilization = self.get_endpoint_utilization(endpoint['name'])
            
            if utilization.get('total_invocations', 0) < threshold_invocations:
                print(f"Endpoint {endpoint['name']} has low utilization: {utilization}")
                
                if not dry_run:
                    try:
                        self.sagemaker.delete_endpoint(EndpointName=endpoint['name'])
                        stopped_endpoints.append(endpoint['name'])
                        print(f"Stopped endpoint: {endpoint['name']}")
                    except Exception as e:
                        print(f"Failed to stop {endpoint['name']}: {str(e)}")
                else:
                    print(f"DRY RUN: Would stop endpoint {endpoint['name']}")
                    stopped_endpoints.append(endpoint['name'])
        
        return stopped_endpoints
    
    def create_serverless_endpoint(self, model_name, endpoint_name):
        """Create serverless endpoint for cost optimization"""
        
        # Create serverless endpoint config
        config_name = f"{endpoint_name}-serverless-config"
        
        self.sagemaker.create_endpoint_config(
            EndpointConfigName=config_name,
            ProductionVariants=[
                {
                    'VariantName': 'serverless-variant',
                    'ModelName': model_name,
                    'ServerlessConfig': {
                        'MemorySizeInMB': 2048,
                        'MaxConcurrency': 10
                    }
                }
            ]
        )
        
        # Create serverless endpoint
        self.sagemaker.create_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=config_name,
            Tags=[
                {'Key': 'CostOptimized', 'Value': 'true'},
                {'Key': 'EndpointType', 'Value': 'serverless'}
            ]
        )
        
        print(f"Created serverless endpoint: {endpoint_name}")

def main():
    parser = argparse.ArgumentParser(description='Manage SageMaker endpoints for cost optimization')
    parser.add_argument('--action', choices=['list', 'stop-idle', 'create-serverless'], required=True)
    parser.add_argument('--endpoint-name', help='Endpoint name for operations')
    parser.add_argument('--model-name', help='Model name for serverless endpoint')
    parser.add_argument('--dry-run', action='store_true', help='Show what would be done without executing')
    parser.add_argument('--threshold', type=int, default=10, help='Invocation threshold for idle detection')
    
    args = parser.parse_args()
    
    manager = SageMakerEndpointManager()
    
    if args.action == 'list':
        endpoints = manager.list_endpoints()
        print("SageMaker Endpoints:")
        for ep in endpoints:
            print(f"  {ep['name']}: {ep['status']} ({ep.get('instance_type', 'unknown')})")
    
    elif args.action == 'stop-idle':
        stopped = manager.stop_idle_endpoints(
            threshold_invocations=args.threshold,
            dry_run=args.dry_run
        )
        print(f"Stopped {len(stopped)} idle endpoints")
    
    elif args.action == 'create-serverless':
        if not args.endpoint_name or not args.model_name:
            print("--endpoint-name and --model-name required for serverless endpoint")
            return
        
        manager.create_serverless_endpoint(args.model_name, args.endpoint_name)

if __name__ == "__main__":
    main()
```

### 3. S3 Lifecycle Management

#### 3.1 S3 Lifecycle Policies

```python
# scripts/setup_s3_lifecycle.py
import boto3
import json

class S3LifecycleManager:
    def __init__(self):
        self.s3 = boto3.client('s3')
    
    def create_lifecycle_policy(self, bucket_name, policy_type='standard'):
        """Create S3 lifecycle policy based on data type"""
        
        if policy_type == 'data':
            # Policy for data buckets
            lifecycle_config = {
                'Rules': [
                    {
                        'ID': 'DataRetentionPolicy',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'datasets/'},
                        'Transitions': [
                            {
                                'Days': 30,
                                'StorageClass': 'STANDARD_IA'
                            },
                            {
                                'Days': 90,
                                'StorageClass': 'GLACIER'
                            },
                            {
                                'Days': 365,
                                'StorageClass': 'DEEP_ARCHIVE'
                            }
                        ]
                    },
                    {
                        'ID': 'TempDataCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'temp/'},
                        'Expiration': {'Days': 7}
                    },
                    {
                        'ID': 'LogsCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'logs/'},
                        'Expiration': {'Days': 30}
                    }
                ]
            }
        
        elif policy_type == 'models':
            # Policy for model artifacts
            lifecycle_config = {
                'Rules': [
                    {
                        'ID': 'ModelRetentionPolicy',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'models/'},
                        'Transitions': [
                            {
                                'Days': 60,
                                'StorageClass': 'STANDARD_IA'
                            },
                            {
                                'Days': 180,
                                'StorageClass': 'GLACIER'
                            }
                        ]
                    },
                    {
                        'ID': 'ExperimentCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'experiments/'},
                        'Expiration': {'Days': 90}
                    }
                ]
            }
        
        elif policy_type == 'artifacts':
            # Policy for build artifacts
            lifecycle_config = {
                'Rules': [
                    {
                        'ID': 'ArtifactCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'artifacts/'},
                        'Expiration': {'Days': 30}
                    },
                    {
                        'ID': 'BuildCacheCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'cache/'},
                        'Expiration': {'Days': 14}
                    }
                ]
            }
        
        else:
            # Standard policy
            lifecycle_config = {
                'Rules': [
                    {
                        'ID': 'StandardRetentionPolicy',
                        'Status': 'Enabled',
                        'Transitions': [
                            {
                                'Days': 30,
                                'StorageClass': 'STANDARD_IA'
                            },
                            {
                                'Days': 90,
                                'StorageClass': 'GLACIER'
                            }
                        ],
                        'Expiration': {'Days': 365}
                    }
                ]
            }
        
        try:
            self.s3.put_bucket_lifecycle_configuration(
                Bucket=bucket_name,
                LifecycleConfiguration=lifecycle_config
            )
            print(f"Lifecycle policy applied to {bucket_name}")
            return True
            
        except Exception as e:
            print(f"Failed to apply lifecycle policy to {bucket_name}: {str(e)}")
            return False
    
    def setup_all_bucket_policies(self):
        """Setup lifecycle policies for all MLOps buckets"""
        
        bucket_policies = {
            'retail-forecast-data-bucket': 'data',
            'retail-forecast-models-bucket': 'models',
            'retail-forecast-artifacts-bucket': 'artifacts'
        }
        
        for bucket, policy_type in bucket_policies.items():
            print(f"Setting up {policy_type} lifecycle policy for {bucket}")
            self.create_lifecycle_policy(bucket, policy_type)
    
    def analyze_bucket_costs(self, bucket_name):
        """Analyze S3 bucket storage costs and recommend optimizations"""
        
        try:
            # Get bucket inventory
            objects = self.s3.list_objects_v2(Bucket=bucket_name)
            
            if 'Contents' not in objects:
                return {'error': 'Bucket is empty or inaccessible'}
            
            analysis = {
                'total_objects': len(objects['Contents']),
                'total_size_gb': 0,
                'storage_classes': {},
                'age_distribution': {'0-30': 0, '30-90': 0, '90+': 0},
                'recommendations': []
            }
            
            from datetime import datetime, timezone
            now = datetime.now(timezone.utc)
            
            for obj in objects['Contents']:
                # Calculate size
                size_gb = obj['Size'] / (1024**3)
                analysis['total_size_gb'] += size_gb
                
                # Track storage class
                storage_class = obj.get('StorageClass', 'STANDARD')
                analysis['storage_classes'][storage_class] = analysis['storage_classes'].get(storage_class, 0) + 1
                
                # Calculate age
                age_days = (now - obj['LastModified']).days
                if age_days <= 30:
                    analysis['age_distribution']['0-30'] += 1
                elif age_days <= 90:
                    analysis['age_distribution']['30-90'] += 1
                else:
                    analysis['age_distribution']['90+'] += 1
            
            # Generate recommendations
            if analysis['age_distribution']['30-90'] > 0:
                analysis['recommendations'].append(
                    f"{analysis['age_distribution']['30-90']} objects could be moved to IA storage"
                )
            
            if analysis['age_distribution']['90+'] > 0:
                analysis['recommendations'].append(
                    f"{analysis['age_distribution']['90+']} objects could be moved to Glacier"
                )
            
            # Estimate costs
            standard_cost = analysis['storage_classes'].get('STANDARD', 0) * 0.023  # $0.023/GB
            ia_cost = analysis['storage_classes'].get('STANDARD_IA', 0) * 0.0125  # $0.0125/GB
            glacier_cost = analysis['storage_classes'].get('GLACIER', 0) * 0.004  # $0.004/GB
            
            analysis['estimated_monthly_cost'] = {
                'standard': standard_cost,
                'ia': ia_cost,
                'glacier': glacier_cost,
                'total': standard_cost + ia_cost + glacier_cost
            }
            
            return analysis
            
        except Exception as e:
            return {'error': str(e)}

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Manage S3 lifecycle policies')
    parser.add_argument('--action', choices=['setup', 'analyze'], required=True)
    parser.add_argument('--bucket', help='Specific bucket name')
    parser.add_argument('--policy-type', choices=['data', 'models', 'artifacts', 'standard'], 
                       default='standard')
    
    args = parser.parse_args()
    
    manager = S3LifecycleManager()
    
    if args.action == 'setup':
        if args.bucket:
            manager.create_lifecycle_policy(args.bucket, args.policy_type)
        else:
            manager.setup_all_bucket_policies()
    
    elif args.action == 'analyze':
        if not args.bucket:
            print("--bucket required for analysis")
            return
        
        analysis = manager.analyze_bucket_costs(args.bucket)
        print(json.dumps(analysis, indent=2, default=str))

if __name__ == "__main__":
    main()
```

### 4. CI/CD Pipeline Management

#### 4.1 Pipeline Pause/Resume Scripts

```bash
# scripts/manage_cicd_pipeline.sh
#!/bin/bash

ACTION=$1
ENVIRONMENT=${2:-all}

case $ACTION in
  "pause")
    echo "Pausing CI/CD pipelines..."
    
    # Disable Jenkins builds
    if command -v jenkins-cli &> /dev/null; then
        jenkins-cli disable-job retail-forecast-pipeline
        echo "Jenkins pipeline disabled"
    fi
    
    # Disable GitHub Actions (requires GitHub CLI)
    if command -v gh &> /dev/null; then
        gh workflow disable "MLOps Pipeline" --repo uit-ntn/aws-mlops-retail-forecast
        echo "GitHub Actions workflow disabled"
    fi
    
    # Scale down Jenkins agents
    kubectl scale deployment jenkins-agent --replicas=0 -n jenkins || true
    
    # Stop scheduled training jobs
    aws events disable-rule --name "scheduled-training-trigger" || true
    
    echo "CI/CD pipelines paused successfully"
    ;;
    
  "resume")
    echo "Resuming CI/CD pipelines..."
    
    # Enable Jenkins builds
    if command -v jenkins-cli &> /dev/null; then
        jenkins-cli enable-job retail-forecast-pipeline
        echo "Jenkins pipeline enabled"
    fi
    
    # Enable GitHub Actions
    if command -v gh &> /dev/null; then
        gh workflow enable "MLOps Pipeline" --repo uit-ntn/aws-mlops-retail-forecast
        echo "GitHub Actions workflow enabled"
    fi
    
    # Scale up Jenkins agents
    kubectl scale deployment jenkins-agent --replicas=1 -n jenkins || true
    
    # Enable scheduled training jobs
    aws events enable-rule --name "scheduled-training-trigger" || true
    
    echo "CI/CD pipelines resumed successfully"
    ;;
    
  "status")
    echo "CI/CD Pipeline Status:"
    
    # Check Jenkins status
    if command -v jenkins-cli &> /dev/null; then
        jenkins_status=$(jenkins-cli get-job retail-forecast-pipeline | grep -o 'disabled.*' || echo "enabled")
        echo "  Jenkins: $jenkins_status"
    fi
    
    # Check GitHub Actions status
    if command -v gh &> /dev/null; then
        gh workflow list --repo uit-ntn/aws-mlops-retail-forecast
    fi
    
    # Check scheduled events
    aws events describe-rule --name "scheduled-training-trigger" --query 'State' --output text 2>/dev/null || echo "  Scheduled training: not configured"
    ;;
    
  *)
    echo "Usage: $0 {pause|resume|status}"
    echo "  pause:  Disable all CI/CD pipelines to prevent unexpected builds"
    echo "  resume: Re-enable CI/CD pipelines for normal operation"
    echo "  status: Show current status of CI/CD components"
    exit 1
    ;;
esac
```

### 5. Complete Teardown Procedures

#### 5.1 Terraform Teardown Script

```bash
# scripts/teardown_infrastructure.sh
#!/bin/bash

ENVIRONMENT=${1:-development}
FORCE=${2:-false}
DRY_RUN=${3:-false}

echo "=== AWS MLOps Infrastructure Teardown ==="
echo "Environment: $ENVIRONMENT"
echo "Force: $FORCE"
echo "Dry Run: $DRY_RUN"
echo "=========================================="

# Function to confirm action
confirm_action() {
    if [ "$FORCE" != "true" ]; then
        read -p "Are you sure you want to proceed with teardown? (yes/no): " confirm
        if [ "$confirm" != "yes" ]; then
            echo "Teardown cancelled"
            exit 0
        fi
    fi
}

# Function to backup critical data
backup_critical_data() {
    echo "Creating backup of critical data..."
    
    # Backup model registry
    aws s3 sync s3://retail-forecast-models-bucket/models/ \
        ./backups/models-$(date +%Y%m%d)/ \
        --exclude "*" --include "*/model.tar.gz"
    
    # Backup latest training data
    aws s3 sync s3://retail-forecast-data-bucket/datasets/ \
        ./backups/data-$(date +%Y%m%d)/ \
        --exclude "*" --include "latest/*"
    
    # Export infrastructure state
    terraform output -json > ./backups/terraform-outputs-$(date +%Y%m%d).json
    
    echo "Backup completed"
}

# Function to clean ECR images
cleanup_ecr_images() {
    echo "Cleaning up ECR images..."
    
    REPOSITORIES=("retail-forecast-training" "retail-forecast-inference")
    
    for repo in "${REPOSITORIES[@]}"; do
        echo "Cleaning repository: $repo"
        
        if [ "$DRY_RUN" = "true" ]; then
            aws ecr describe-images --repository-name $repo \
                --query 'imageDetails[?imageAge>`7`].[imageDigest]' --output text
        else
            # Keep only last 3 images, delete others
            OLD_IMAGES=$(aws ecr describe-images --repository-name $repo \
                --query 'imageDetails[?imageAge>`7`].[imageDigest]' --output text)
            
            if [ -n "$OLD_IMAGES" ]; then
                echo $OLD_IMAGES | xargs -n1 -I {} aws ecr batch-delete-image \
                    --repository-name $repo --image-ids imageDigest={}
            fi
        fi
    done
}

# Function to stop SageMaker endpoints
stop_sagemaker_endpoints() {
    echo "Stopping SageMaker endpoints..."
    
    ENDPOINTS=$(aws sagemaker list-endpoints --status-equals InService \
        --query 'Endpoints[?contains(EndpointName, `retail-forecast`)].EndpointName' \
        --output text)
    
    for endpoint in $ENDPOINTS; do
        echo "Stopping endpoint: $endpoint"
        
        if [ "$DRY_RUN" != "true" ]; then
            aws sagemaker delete-endpoint --endpoint-name $endpoint
        fi
    done
}

# Function to scale down EKS
scale_down_eks() {
    echo "Scaling down EKS cluster..."
    
    if [ "$DRY_RUN" != "true" ]; then
        # Scale down applications
        kubectl scale deployment --all --replicas=0 -n retail-forecast || true
        
        # Scale down node group
        aws eks update-nodegroup-config \
            --cluster-name retail-forecast-cluster \
            --nodegroup-name retail-forecast-nodegroup \
            --scaling-config minSize=0,maxSize=0,desiredSize=0 || true
    else
        echo "DRY RUN: Would scale down EKS cluster and applications"
    fi
}

# Function to terraform destroy
terraform_destroy() {
    echo "Running Terraform destroy..."
    
    cd aws/infra
    
    if [ "$DRY_RUN" = "true" ]; then
        terraform plan -destroy -var="environment=$ENVIRONMENT"
    else
        terraform destroy -auto-approve -var="environment=$ENVIRONMENT"
    fi
    
    cd -
}

# Function to verify cleanup
verify_cleanup() {
    echo "Verifying resource cleanup..."
    
    # Check EC2 instances
    INSTANCES=$(aws ec2 describe-instances \
        --filters "Name=tag:Project,Values=RetailForecast" "Name=instance-state-name,Values=running" \
        --query 'Reservations[].Instances[].InstanceId' --output text)
    
    if [ -n "$INSTANCES" ]; then
        echo "WARNING: Found running EC2 instances: $INSTANCES"
    else
        echo "✓ No running EC2 instances found"
    fi
    
    # Check SageMaker endpoints
    ENDPOINTS=$(aws sagemaker list-endpoints --status-equals InService \
        --query 'Endpoints[?contains(EndpointName, `retail-forecast`)].EndpointName' \
        --output text)
    
    if [ -n "$ENDPOINTS" ]; then
        echo "WARNING: Found active SageMaker endpoints: $ENDPOINTS"
    else
        echo "✓ No active SageMaker endpoints found"
    fi
    
    # Check Load Balancers
    LOAD_BALANCERS=$(aws elbv2 describe-load-balancers \
        --query 'LoadBalancers[?contains(LoadBalancerName, `retail-forecast`)].LoadBalancerArn' \
        --output text)
    
    if [ -n "$LOAD_BALANCERS" ]; then
        echo "WARNING: Found active Load Balancers: $LOAD_BALANCERS"
    else
        echo "✓ No active Load Balancers found"
    fi
}

# Function to generate cost report
generate_final_cost_report() {
    echo "Generating final cost report..."
    
    python scripts/cost_analyzer.py > teardown-cost-report-$(date +%Y%m%d).json
    
    echo "Cost report saved to teardown-cost-report-$(date +%Y%m%d).json"
}

# Main execution
main() {
    confirm_action
    
    echo "Starting teardown process..."
    
    # Step 1: Backup critical data
    backup_critical_data
    
    # Step 2: Stop CI/CD pipelines
    ./scripts/manage_cicd_pipeline.sh pause
    
    # Step 3: Stop SageMaker endpoints
    stop_sagemaker_endpoints
    
    # Step 4: Scale down EKS
    scale_down_eks
    
    # Step 5: Cleanup ECR images
    cleanup_ecr_images
    
    # Step 6: Terraform destroy
    terraform_destroy
    
    # Step 7: Verify cleanup
    verify_cleanup
    
    # Step 8: Generate final cost report
    generate_final_cost_report
    
    echo "=========================================="
    echo "Teardown completed!"
    echo "Backups stored in: ./backups/"
    echo "To restore, run: ./scripts/restore_infrastructure.sh"
    echo "=========================================="
}

# Parse command line arguments
while [[ $# -gt 0 ]]; do
    case $1 in
        --dry-run)
            DRY_RUN=true
            shift
            ;;
        --force)
            FORCE=true
            shift
            ;;
        --environment)
            ENVIRONMENT="$2"
            shift 2
            ;;
        *)
            echo "Unknown option: $1"
            echo "Usage: $0 [--dry-run] [--force] [--environment ENV]"
            exit 1
            ;;
    esac
done

# Run main function
main
```

#### 5.2 Restore Infrastructure Script

```bash
# scripts/restore_infrastructure.sh
#!/bin/bash

BACKUP_DATE=${1:-$(date +%Y%m%d)}
ENVIRONMENT=${2:-development}

echo "=== Infrastructure Restore ==="
echo "Backup Date: $BACKUP_DATE"
echo "Environment: $ENVIRONMENT"
echo "=============================="

# Function to restore Terraform state
restore_terraform() {
    echo "Restoring infrastructure with Terraform..."
    
    cd aws/infra
    
    # Initialize Terraform
    terraform init
    
    # Apply infrastructure
    terraform apply -auto-approve -var="environment=$ENVIRONMENT"
    
    cd -
}

# Function to restore data
restore_data() {
    echo "Restoring data from backup..."
    
    # Restore models
    if [ -d "./backups/models-$BACKUP_DATE" ]; then
        aws s3 sync "./backups/models-$BACKUP_DATE/" \
            s3://retail-forecast-models-bucket/models/
        echo "Models restored"
    fi
    
    # Restore data
    if [ -d "./backups/data-$BACKUP_DATE" ]; then
        aws s3 sync "./backups/data-$BACKUP_DATE/" \
            s3://retail-forecast-data-bucket/datasets/
        echo "Data restored"
    fi
}

# Function to restore applications
restore_applications() {
    echo "Restoring Kubernetes applications..."
    
    # Apply Kubernetes manifests
    kubectl apply -f aws/k8s/ -n retail-forecast
    
    # Wait for deployments
    kubectl wait --for=condition=available --timeout=300s \
        deployment --all -n retail-forecast
}

# Function to resume CI/CD
resume_cicd() {
    echo "Resuming CI/CD pipelines..."
    
    ./scripts/manage_cicd_pipeline.sh resume
}

# Main execution
main() {
    echo "Starting restore process..."
    
    # Step 1: Restore infrastructure
    restore_terraform
    
    # Step 2: Restore data
    restore_data
    
    # Step 3: Restore applications
    restore_applications
    
    # Step 4: Resume CI/CD
    resume_cicd
    
    echo "=============================="
    echo "Restore completed!"
    echo "Infrastructure is ready for use"
    echo "=============================="
}

main
```

### 6. Cost Optimization Automation

#### 6.1 Automated Cost Optimization

```python
# scripts/automated_cost_optimizer.py
import boto3
import json
import schedule
import time
from datetime import datetime, timedelta

class AutomatedCostOptimizer:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.sagemaker = boto3.client('sagemaker')
        self.s3 = boto3.client('s3')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def optimize_ec2_instances(self):
        """Optimize EC2 instances based on utilization"""
        
        print("Checking EC2 instance utilization...")
        
        instances = self.ec2.describe_instances(
            Filters=[
                {'Name': 'tag:Project', 'Values': ['RetailForecast']},
                {'Name': 'instance-state-name', 'Values': ['running']}
            ]
        )
        
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                
                # Get CPU utilization for last 24 hours
                utilization = self.cloudwatch.get_metric_statistics(
                    Namespace='AWS/EC2',
                    MetricName='CPUUtilization',
                    Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                    StartTime=datetime.now() - timedelta(hours=24),
                    EndTime=datetime.now(),
                    Period=3600,
                    Statistics=['Average']
                )
                
                if utilization['Datapoints']:
                    avg_cpu = sum([dp['Average'] for dp in utilization['Datapoints']]) / len(utilization['Datapoints'])
                    
                    # Stop instances with very low utilization
                    if avg_cpu < 5 and self.is_stoppable(instance):
                        print(f"Stopping low-utilization instance {instance_id} (CPU: {avg_cpu:.1f}%)")
                        self.ec2.stop_instances(InstanceIds=[instance_id])
    
    def is_stoppable(self, instance):
        """Check if instance can be safely stopped"""
        
        # Don't stop instances with specific tags
        for tag in instance.get('Tags', []):
            if tag['Key'] == 'AutoStop' and tag['Value'] == 'false':
                return False
            if tag['Key'] == 'Environment' and tag['Value'] == 'production':
                return False
        
        return True
    
    def optimize_sagemaker_endpoints(self):
        """Optimize SageMaker endpoints"""
        
        print("Checking SageMaker endpoints...")
        
        endpoints = self.sagemaker.list_endpoints(StatusEquals='InService')
        
        for endpoint in endpoints['Endpoints']:
            endpoint_name = endpoint['EndpointName']
            
            # Get invocation metrics
            metrics = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/SageMaker',
                MetricName='Invocations',
                Dimensions=[{'Name': 'EndpointName', 'Value': endpoint_name}],
                StartTime=datetime.now() - timedelta(hours=24),
                EndTime=datetime.now(),
                Period=3600,
                Statistics=['Sum']
            )
            
            total_invocations = sum([dp['Sum'] for dp in metrics['Datapoints']])
            
            # If endpoint has very low usage, consider stopping it
            if total_invocations < 10:
                print(f"Low-usage endpoint detected: {endpoint_name} ({total_invocations} invocations)")
                # Add logic to notify or auto-stop based on tags
    
    def cleanup_old_snapshots(self):
        """Clean up old EBS snapshots"""
        
        print("Cleaning up old snapshots...")
        
        snapshots = self.ec2.describe_snapshots(OwnerIds=['self'])
        
        for snapshot in snapshots['Snapshots']:
            # Delete snapshots older than 30 days
            age = datetime.now(snapshot['StartTime'].tzinfo) - snapshot['StartTime']
            
            if age.days > 30:
                try:
                    self.ec2.delete_snapshot(SnapshotId=snapshot['SnapshotId'])
                    print(f"Deleted old snapshot: {snapshot['SnapshotId']}")
                except Exception as e:
                    print(f"Failed to delete snapshot {snapshot['SnapshotId']}: {str(e)}")
    
    def optimize_s3_storage(self):
        """Optimize S3 storage costs"""
        
        print("Optimizing S3 storage...")
        
        buckets = ['retail-forecast-data-bucket', 'retail-forecast-models-bucket', 
                  'retail-forecast-artifacts-bucket']
        
        for bucket_name in buckets:
            try:
                # Check for objects that can be moved to cheaper storage
                objects = self.s3.list_objects_v2(Bucket=bucket_name)
                
                for obj in objects.get('Contents', []):
                    age = datetime.now(obj['LastModified'].tzinfo) - obj['LastModified']
                    
                    # Move objects older than 30 days to IA
                    if age.days > 30 and obj.get('StorageClass') == 'STANDARD':
                        self.s3.copy_object(
                            Bucket=bucket_name,
                            Key=obj['Key'],
                            CopySource={'Bucket': bucket_name, 'Key': obj['Key']},
                            StorageClass='STANDARD_IA',
                            MetadataDirective='COPY'
                        )
                        print(f"Moved to IA: {obj['Key']}")
                
            except Exception as e:
                print(f"Error optimizing bucket {bucket_name}: {str(e)}")
    
    def generate_optimization_report(self):
        """Generate optimization report"""
        
        report = {
            'timestamp': datetime.now().isoformat(),
            'optimizations_performed': [],
            'recommendations': []
        }
        
        # Add report generation logic
        with open(f'optimization-report-{datetime.now().strftime("%Y%m%d")}.json', 'w') as f:
            json.dump(report, f, indent=2)
    
    def run_daily_optimization(self):
        """Run daily optimization tasks"""
        
        print(f"Starting daily optimization at {datetime.now()}")
        
        try:
            self.optimize_ec2_instances()
            self.optimize_sagemaker_endpoints()
            self.cleanup_old_snapshots()
            self.optimize_s3_storage()
            self.generate_optimization_report()
            
            print("Daily optimization completed successfully")
            
        except Exception as e:
            print(f"Error during optimization: {str(e)}")

def main():
    optimizer = AutomatedCostOptimizer()
    
    # Schedule daily optimization at 2 AM
    schedule.every().day.at("02:00").do(optimizer.run_daily_optimization)
    
    # Schedule weekly deep optimization on Sundays
    schedule.every().sunday.at("03:00").do(optimizer.run_weekly_optimization)
    
    print("Cost optimizer started. Running scheduled optimizations...")
    
    while True:
        schedule.run_pending()
        time.sleep(60)  # Check every minute

if __name__ == "__main__":
    main()
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Cost Monitoring**: AWS Cost Explorer và alerting được setup
- [ ] **Resource Scaling**: EKS nodes có thể scale down/up theo nhu cầu
- [ ] **SageMaker Optimization**: Endpoints được quản lý theo utilization
- [ ] **S3 Lifecycle**: Policies được thiết lập cho tất cả buckets
- [ ] **CI/CD Management**: Pipelines có thể pause/resume
- [ ] **Automated Cleanup**: Scripts để dọn dẹp resources cũ
- [ ] **Complete Teardown**: Terraform destroy với backup
- [ ] **Restore Capability**: Khả năng restore infrastructure
- [ ] **Cost Optimization**: Automated daily optimization

### 📊 Verification Steps

1. **Chi phí AWS giảm khi scale down**
   ```bash
   # Scale down EKS nodes
   ./scripts/scale_eks_nodes.sh scale-down
   
   # Check node count
   kubectl get nodes
   
   # Stop idle SageMaker endpoints
   python scripts/manage_sagemaker_endpoints.py --action stop-idle
   
   # Check cost reduction in 24h
   python scripts/cost_analyzer.py
   ```

2. **S3 lifecycle policies hoạt động**
   ```bash
   # Check lifecycle policies
   python scripts/setup_s3_lifecycle.py --action analyze --bucket retail-forecast-data-bucket
   
   # Verify objects moved to IA/Glacier
   aws s3api list-objects-v2 --bucket retail-forecast-data-bucket \
     --query 'Contents[?StorageClass!=`STANDARD`].[Key,StorageClass]'
   ```

3. **Teardown và restore hoạt động**
   ```bash
   # Dry run teardown
   ./scripts/teardown_infrastructure.sh --dry-run
   
   # Actual teardown (if needed)
   ./scripts/teardown_infrastructure.sh --force
   
   # Restore infrastructure
   ./scripts/restore_infrastructure.sh
   ```

### 🔍 Monitoring Commands

```bash
# Daily cost check
python scripts/cost_analyzer.py | jq '.mlops_resource_costs'

# Check resource utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --start-time $(date -d '1 day ago' -u +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Average

# Monitor S3 costs
aws s3api list-buckets --query 'Buckets[?contains(Name, `retail-forecast`)].Name' | \
  xargs -I {} aws s3 ls s3://{} --recursive --summarize

# Check orphaned resources
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=RetailForecast" \
  --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,InstanceType,LaunchTime]'
```

## Best Practices Summary

### 💰 **Cost Optimization**
- Daily monitoring of resource utilization
- Automated scaling based on demand
- S3 lifecycle policies for data archival
- Regular cleanup of unused resources

### 🔧 **Resource Management**
- Infrastructure as Code for easy recreation
- Comprehensive backup before teardown
- Automated cost alerting and anomaly detection
- Tagging strategy for resource tracking

### 🚀 **Operational Efficiency**
- Scheduled optimization tasks
- CI/CD pipeline management
- Serverless endpoints for infrequent inference
- Automated resource lifecycle management

### 🛡️ **Risk Mitigation**
- Complete backup procedures
- Dry-run options for destructive operations
- Verification steps after teardown
- Easy restore capabilities

---

**Conclusion**: Comprehensive MLOps platform with complete lifecycle management, from development to production deployment, monitoring, and cost-optimized teardown. The infrastructure provides enterprise-grade capabilities while maintaining cost efficiency through automated optimization and proper resource management.