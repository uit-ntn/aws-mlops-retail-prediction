---
title: "Cost Management & Teardown"
date: 2024-01-01T00:00:00Z
weight: 13
chapter: false
pre: "<b>12. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 12:** Quản lý và tối ưu chi phí vận hành toàn bộ hạ tầng MLOps trên AWS:
{{% /notice %}}
- Giảm thiểu chi phí compute (EC2, SageMaker, ALB)
- Tự động scale-down hoặc xóa tài nguyên không sử dụng
- Áp dụng lifecycle policies cho dữ liệu và container images
- Đảm bảo pipeline vừa hoạt động hiệu quả, vừa tiết kiệm chi phí tối đa

## 1. Chi phí vận hành hệ thống MLOps

Khi triển khai hệ thống MLOps đầy đủ như Retail Prediction API, chi phí vận hành có thể nhanh chóng tăng cao nếu không được quản lý hợp lý. Các thành phần chính góp phần vào chi phí bao gồm:

### 1.1. Phân tích chi phí theo thành phần

| Dịch vụ | Chi phí không tối ưu | Nguyên nhân | Giải pháp |
|---------|---------------------|------------|-----------|
| **EKS NodeGroup** | 0.04 USD/giờ/node | On-demand instances chạy 24/7 | Spot instances + Auto scaling + Schedule |
| **SageMaker Training** | 0.3 USD/job | Instances size lớn, chạy lâu | Spot training + Hyperparameter tuning tối ưu |
| **S3 Storage** | 0.023 USD/GB/tháng | Lưu trữ tất cả dữ liệu ở Standard tier | Lifecycle policy + Intelligent-Tiering |
| **CloudWatch Logs** | 0.50 USD/GB | Lưu trữ logs không giới hạn | Log retention policy + Insights query tối ưu |
| **ALB** | 0.027 USD/giờ | Chạy liên tục kể cả khi không có traffic | Schedule shutdown khi không sử dụng |
| **ECR Storage** | 0.10 USD/GB/tháng | Lưu trữ tất cả image versions | Lifecycle policy xóa image cũ |

## 2. Sử dụng EC2 Spot Instance

EC2 Spot Instances là một trong những cách hiệu quả nhất để tiết kiệm chi phí compute trên AWS, với mức giảm lên đến 70-90% so với On-demand instances.

### 2.1. Sử dụng Spot cho EKS NodeGroup

Cập nhật file cấu hình Terraform cho EKS NodeGroup:

```hcl
module "eks_managed_node_group" {
  source = "terraform-aws-modules/eks/aws//modules/eks-managed-node-group"
  
  name            = "retail-forecast-nodes"
  cluster_name    = module.eks.cluster_name
  cluster_version = module.eks.cluster_version
  
  # Cấu hình Spot Instance
  capacity_type   = "SPOT"  # Thay vì ON_DEMAND
  
  # Diversify instance types để tăng khả năng có Spot
  instance_types  = ["t3.medium", "t3a.medium", "t2.medium"]
  
  min_size        = 2
  max_size        = 5
  desired_size    = 2
  
  # Thêm labels và taints cho Kubernetes scheduler
  labels = {
    app = "retail-api"
  }
}
```

### 2.2. Sử dụng Spot cho SageMaker Training

Update script tạo SageMaker training job để sử dụng spot instances:

```python
# aws/script/create_training_job.py

import boto3
import argparse
import time

def create_training_job(job_name, data_bucket, output_bucket, instance_type, use_spot=True):
    sagemaker = boto3.client('sagemaker')
    
    # Tính toán thời gian stop cho Spot (giới hạn 1 giờ)
    current_time = int(time.time())
    stop_time = current_time + 3600  # 1 hour
    
    # Cấu hình training job
    training_params = {
        'TrainingJobName': job_name,
        'AlgorithmSpecification': {
            'TrainingImage': '123456789012.dkr.ecr.us-east-1.amazonaws.com/retail-forecast-training:latest',
            'TrainingInputMode': 'File'
        },
        'RoleArn': 'arn:aws:iam::123456789012:role/SageMakerExecutionRole',
        'InputDataConfig': [
            {
                'ChannelName': 'train',
                'DataSource': {
                    'S3DataSource': {
                        'S3DataType': 'S3Prefix',
                        'S3Uri': f's3://{data_bucket}/train',
                        'S3DataDistributionType': 'FullyReplicated'
                    }
                }
            }
        ],
        'OutputDataConfig': {
            'S3OutputPath': f's3://{output_bucket}/output'
        },
        'ResourceConfig': {
            'InstanceType': instance_type,
            'InstanceCount': 1,
            'VolumeSizeInGB': 10
        },
        'StoppingCondition': {
            'MaxRuntimeInSeconds': 3600
        },
        'Tags': [
            {
                'Key': 'Project',
                'Value': 'RetailMLOps'
            }
        ]
    }
    
    # Cấu hình Spot training nếu được yêu cầu
    if use_spot:
        training_params['EnableManagedSpotTraining'] = True
        training_params['StoppingCondition']['MaxWaitTimeInSeconds'] = 3900  # Thêm thời gian chờ tối đa
    
    response = sagemaker.create_training_job(**training_params)
    return response

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument('--job-name', type=str, required=True)
    parser.add_argument('--data-bucket', type=str, required=True)
    parser.add_argument('--output-bucket', type=str, required=True)
    parser.add_argument('--instance-type', type=str, default='ml.m5.large')
    parser.add_argument('--use-spot', type=bool, default=True)
    
    args = parser.parse_args()
    
    response = create_training_job(
        args.job_name,
        args.data_bucket,
        args.output_bucket,
        args.instance_type,
        args.use_spot
    )
    
    print(f"Training job created: {response['TrainingJobArn']}")
```

### 2.3. Xử lý gián đoạn Spot Instance

Để đảm bảo hệ thống vẫn hoạt động khi các Spot Instances bị thu hồi, cần cấu hình:

1. **Pod Disruption Budget (PDB)** để đảm bảo số lượng pod tối thiểu:

```yaml
# aws/k8s/pdb/retail-forecast-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: retail-forecast-pdb
  namespace: retail-forecast
spec:
  minAvailable: 1  # Luôn đảm bảo ít nhất 1 pod đang chạy
  selector:
    matchLabels:
      app: retail-forecast-api
```

2. **Cluster Autoscaler** với Spot Instance Handling:

```yaml
# aws/k8s/addons/cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.26.2
          name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/retail-forecast-cluster
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```

## 3. S3 Lifecycle & Intelligent-Tiering

### 3.1. Cấu hình Lifecycle Policy cho S3 Bucket

Triển khai lifecycle policy thông qua Terraform:

```hcl
# infra/modules/s3/main.tf
resource "aws_s3_bucket" "retail_forecast_data" {
  bucket = "retail-forecast-data-${var.environment}"
  
  tags = {
    Project = "RetailMLOps"
  }
}

# Lifecycle configuration
resource "aws_s3_bucket_lifecycle_configuration" "data_lifecycle" {
  bucket = aws_s3_bucket.retail_forecast_data.id

  rule {
    id     = "raw-data-tier"
    status = "Enabled"
    
    filter {
      prefix = "raw/"
    }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }
  }
  
  rule {
    id     = "silver-data-tier"
    status = "Enabled"
    
    filter {
      prefix = "silver/"
    }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }
  }
  
  rule {
    id     = "artifacts-archive"
    status = "Enabled"
    
    filter {
      prefix = "artifacts/"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
    
    transition {
      days          = 180
      storage_class = "DEEP_ARCHIVE"
    }
  }
  
  rule {
    id     = "logs-archive"
    status = "Enabled"
    
    filter {
      prefix = "logs/"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
    
    transition {
      days          = 180
      storage_class = "DEEP_ARCHIVE"
    }
  }
  
  rule {
    id     = "temp-cleanup"
    status = "Enabled"
    
    filter {
      and {
        prefix = "tmp/"
        tag {
          key   = "temporary"
          value = "true"
        }
      }
    }
    
    expiration {
      days = 7
    }
  }
}
```

### 3.2. Cấu hình Intelligent-Tiering

```hcl
# infra/modules/s3/intelligent_tiering.tf
resource "aws_s3_bucket_intelligent_tiering_configuration" "retail_data_tiering" {
  bucket = aws_s3_bucket.retail_forecast_data.id
  name   = "RetailDataTiering"
  
  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }
  
  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }
  
  filter {
    prefix = "data/"
  }
}
```

## 4. Tự động dừng tài nguyên ngoài giờ

### 4.1. Schedule Lambda với EventBridge

Tạo Lambda function để stop/start tài nguyên:

```python
# aws/scripts/resource_scheduler.py
import boto3
import os

def lambda_handler(event, context):
    action = event.get('action', 'stop')  # 'stop' or 'start'
    
    # EKS NodeGroup scaling
    eks = boto3.client('eks')
    autoscaling = boto3.client('autoscaling')
    
    cluster_name = os.environ.get('EKS_CLUSTER_NAME', 'retail-forecast-cluster')
    nodegroup_name = os.environ.get('NODEGROUP_NAME', 'retail-forecast-nodes')
    
    if action == 'stop':
        # Scale down to 0
        print(f"Scaling down nodegroup {nodegroup_name} in cluster {cluster_name}")
        
        # Lấy auto scaling group name từ nodegroup
        response = eks.describe_nodegroup(
            clusterName=cluster_name,
            nodegroupName=nodegroup_name
        )
        
        asg_name = response['nodegroup']['resources']['autoScalingGroups'][0]['name']
        
        # Scale down ASG về 0
        autoscaling.update_auto_scaling_group(
            AutoScalingGroupName=asg_name,
            MinSize=0,
            DesiredCapacity=0
        )
        
        print(f"NodeGroup {nodegroup_name} scaled down to 0")
    
    elif action == 'start':
        # Scale up to desired capacity
        print(f"Scaling up nodegroup {nodegroup_name} in cluster {cluster_name}")
        
        response = eks.describe_nodegroup(
            clusterName=cluster_name,
            nodegroupName=nodegroup_name
        )
        
        asg_name = response['nodegroup']['resources']['autoScalingGroups'][0]['name']
        
        # Scale up ASG to desired capacity
        autoscaling.update_auto_scaling_group(
            AutoScalingGroupName=asg_name,
            MinSize=2,
            DesiredCapacity=2
        )
        
        print(f"NodeGroup {nodegroup_name} scaled up to 2")
    
    # SageMaker Endpoint
    if os.environ.get('SAGEMAKER_ENDPOINT'):
        sm_client = boto3.client('sagemaker')
        endpoint_name = os.environ.get('SAGEMAKER_ENDPOINT')
        
        if action == 'stop':
            # Endpoint không thể dừng nhưng có thể xóa và tạo lại sau
            print(f"Deleting SageMaker endpoint {endpoint_name}")
            try:
                sm_client.delete_endpoint(EndpointName=endpoint_name)
                print(f"Endpoint {endpoint_name} deleted")
            except Exception as e:
                print(f"Error deleting endpoint: {e}")
        
    return {
        'statusCode': 200,
        'body': f"Successfully executed {action} action"
    }
```

Terraform để tạo EventBridge schedule và Lambda:

```hcl
# infra/modules/scheduler/main.tf

resource "aws_lambda_function" "resource_scheduler" {
  function_name = "retail-forecast-resource-scheduler"
  handler       = "resource_scheduler.lambda_handler"
  runtime       = "python3.9"
  role          = aws_iam_role.lambda_role.arn
  filename      = "resource_scheduler.zip"
  timeout       = 300
  
  environment {
    variables = {
      EKS_CLUSTER_NAME   = var.eks_cluster_name
      NODEGROUP_NAME     = var.nodegroup_name
      SAGEMAKER_ENDPOINT = var.sagemaker_endpoint
    }
  }
}

# Tạo EventBridge rule để dừng tài nguyên lúc 19:00 UTC
resource "aws_cloudwatch_event_rule" "stop_resources" {
  name                = "retail-forecast-stop-resources"
  description         = "Stop resources at 19:00 UTC"
  schedule_expression = "cron(0 19 * * ? *)"
}

# Gắn Lambda với rule stop
resource "aws_cloudwatch_event_target" "stop_resources_target" {
  rule      = aws_cloudwatch_event_rule.stop_resources.name
  target_id = "RetailForecastStopResources"
  arn       = aws_lambda_function.resource_scheduler.arn
  input     = jsonencode({
    action = "stop"
  })
}

# Tạo EventBridge rule để khởi động tài nguyên lúc 7:00 UTC
resource "aws_cloudwatch_event_rule" "start_resources" {
  name                = "retail-forecast-start-resources"
  description         = "Start resources at 7:00 UTC"
  schedule_expression = "cron(0 7 * * ? *)"
}

# Gắn Lambda với rule start
resource "aws_cloudwatch_event_target" "start_resources_target" {
  rule      = aws_cloudwatch_event_rule.start_resources.name
  target_id = "RetailForecastStartResources"
  arn       = aws_lambda_function.resource_scheduler.arn
  input     = jsonencode({
    action = "start"
  })
}

# Cấp quyền cho EventBridge để gọi Lambda
resource "aws_lambda_permission" "allow_eventbridge_stop" {
  statement_id  = "AllowExecutionFromEventBridgeStop"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.resource_scheduler.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.stop_resources.arn
}

resource "aws_lambda_permission" "allow_eventbridge_start" {
  statement_id  = "AllowExecutionFromEventBridgeStart"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.resource_scheduler.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.start_resources.arn
}
```

### 4.2. Terraform Destroy cho CI/CD Pipeline

Thêm job vào GitHub Actions workflow để xóa toàn bộ tài nguyên sau khi hoàn thành:

```yaml
# .github/workflows/mlops-pipeline.yml
jobs:
  # Các job hiện tại...
  
  terraform_destroy:
    name: Destroy Infrastructure
    runs-on: ubuntu-latest
    needs: [deploy_eks, monitoring]
    if: github.event.inputs.destroy_after_demo == 'true'
    environment: 
      name: ${{ needs.setup.outputs.environment }}
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
    
    - name: Terraform Init
      run: |
        cd aws/infra
        terraform init
    
    - name: Terraform Destroy
      run: |
        cd aws/infra
        terraform destroy -auto-approve
```

## 5. Cost Visibility & Alerts

### 5.1. Tag Strategy cho AWS Resources

```hcl
# Thêm vào tất cả resource modules

locals {
  common_tags = {
    Project = "RetailMLOps"
  }
}

# Áp dụng tag vào tất cả resource
```

### 5.2. AWS Budgets Alert

```hcl
# infra/modules/budget/main.tf
resource "aws_budgets_budget" "cost" {
  name              = "retail-forecast-${var.environment}-monthly-budget"
  budget_type       = "COST"
  limit_amount      = var.monthly_limit
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  time_period_start = "2023-01-01_00:00"
  
  cost_filter {
    name = "TagKeyValue"
    values = [
      "user:Project$RetailForecastMLOps"
    ]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = var.notification_emails
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = var.notification_emails
    subscriber_sns_topic_arns  = [var.sns_topic_arn]
  }
}
```

Script để tạo AWS Budget qua CLI:

```bash
# aws/scripts/create_budget.sh
#!/bin/bash

ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
EMAIL="your-email@example.com"
BUDGET_NAME="MLOpsBudget"
BUDGET_LIMIT=5 # USD

# Tạo AWS Budget
aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget '{ 
    "BudgetName": "'$BUDGET_NAME'", 
    "BudgetLimit": { 
      "Amount": "'$BUDGET_LIMIT'", 
      "Unit": "USD" 
    },
    "CostFilters": {
      "TagKeyValue": [
        "user:Project$RetailForecastMLOps"
      ]
    },
    "BudgetType": "COST", 
    "TimeUnit": "MONTHLY" 
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": { 
        "NotificationType": "ACTUAL", 
        "ComparisonOperator": "GREATER_THAN", 
        "Threshold": 80, 
        "ThresholdType": "PERCENTAGE" 
      },
      "Subscribers": [
        { 
          "SubscriptionType": "EMAIL", 
          "Address": "'$EMAIL'" 
        }
      ]
    },
    {
      "Notification": { 
        "NotificationType": "ACTUAL", 
        "ComparisonOperator": "GREATER_THAN", 
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE" 
      },
      "Subscribers": [
        { 
          "SubscriptionType": "EMAIL", 
          "Address": "'$EMAIL'" 
        }
      ]
    }
  ]'
```

## 6. ECR & CloudWatch Optimization

### 6.1. ECR Lifecycle Policy

```hcl
# infra/modules/ecr/main.tf
resource "aws_ecr_repository" "retail_forecast" {
  name                 = "retail-forecast"
  image_tag_mutability = "MUTABLE"
  
  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_ecr_lifecycle_policy" "retail_forecast_policy" {
  repository = aws_ecr_repository.retail_forecast.name
  
  policy = jsonencode({
    rules = [
      {
        rulePriority = 1,
        description  = "Keep only 3 latest untagged images",
        selection = {
          tagStatus     = "untagged",
          countType     = "imageCountMoreThan",
          countNumber   = 3
        },
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2,
        description  = "Keep only 3 latest images per tag prefix",
        selection = {
          tagStatus     = "tagged",
          tagPrefixList = ["prod", "stage", "dev"],
          countType     = "imageCountMoreThan",
          countNumber   = 3
        },
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 3,
        description  = "Keep only the 10 most recent images",
        selection = {
          tagStatus   = "any",
          countType   = "imageCountMoreThan",
          countNumber = 10
        },
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

Hoặc sử dụng AWS CLI:

```bash
aws ecr put-lifecycle-policy \
  --repository-name retail-forecast \
  --lifecycle-policy-text '{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Keep only 3 latest untagged images",
            "selection": {
                "tagStatus": "untagged",
                "countType": "imageCountMoreThan",
                "countNumber": 3
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 2,
            "description": "Keep only 3 latest images per tag prefix",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["prod", "stage", "dev"],
                "countType": "imageCountMoreThan",
                "countNumber": 3
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 3,
            "description": "Keep only the 10 most recent images",
            "selection": {
                "tagStatus": "any",
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {
                "type": "expire"
            }
        }
    ]
}'
```

### 6.2. CloudWatch Log Retention Policy

```hcl
# infra/modules/logs/main.tf
resource "aws_cloudwatch_log_group" "eks_logs" {
  name              = "/aws/eks/mlops-retail-cluster/cluster"
  retention_in_days = 7
}

resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/aws/retail/api"
  retention_in_days = 7
}
```

Hoặc sử dụng AWS CLI:

```bash
# Đặt retention policy cho CloudWatch Logs
aws logs put-retention-policy \
  --log-group-name "/aws/eks/retail-forecast-cluster/cluster" \
  --retention-in-days 30

aws logs put-retention-policy \
  --log-group-name "/aws/retail-forecast/api" \
  --retention-in-days 30

aws logs put-retention-policy \
  --log-group-name "/aws/sagemaker/TrainingJobs" \
  --retention-in-days 30
```

## 7. Tự động Teardown toàn bộ hạ tầng

### 7.1. Script Terraform Destroy

```bash
# aws/scripts/teardown.sh
#!/bin/bash

echo "Starting teardown..."

# Xóa Kubernetes resources
kubectl delete namespace mlops --ignore-not-found=true

# Xóa ECR images
aws ecr batch-delete-image --repository-name mlops/retail-api --image-ids imageTag=latest imageTag=v2 imageTag=v3

# Terraform destroy
cd ../infra
terraform destroy -auto-approve

echo "Teardown completed!"
```

### 7.2. Thêm vào GitHub Actions Workflow

```yaml
# .github/workflows/teardown.yml
name: MLOps Infrastructure Teardown

on:
  workflow_dispatch:
    inputs:
      confirmation:
        description: 'Type "destroy" to confirm deletion of all resources'
        required: true
        
env:
  AWS_REGION: us-east-1
  TF_VAR_environment: dev

jobs:
  teardown:
    name: Teardown Infrastructure
    runs-on: ubuntu-latest
    if: github.event.inputs.confirmation == 'destroy'
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --name retail-forecast-cluster --region ${{ env.AWS_REGION }}
    
    - name: Delete Kubernetes resources
      run: |
        kubectl delete namespace retail-forecast --ignore-not-found=true
    
    - name: Delete SageMaker endpoints
      run: |
        ENDPOINTS=$(aws sagemaker list-endpoints --name-contains retail-forecast --query "Endpoints[].EndpointName" --output text)
        if [ ! -z "$ENDPOINTS" ]; then
          for ENDPOINT in $ENDPOINTS; do
            echo "Deleting endpoint: $ENDPOINT"
            aws sagemaker delete-endpoint --endpoint-name $ENDPOINT
          done
        fi
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
    
    - name: Terraform Init
      run: |
        cd aws/infra
        terraform init
    
    - name: Terraform Destroy
      run: |
        cd aws/infra
        terraform destroy -auto-approve
    
    - name: Send notification
      if: always()
      run: |
        if [ "${{ job.status }}" == "success" ]; then
          MESSAGE="✅ Infrastructure teardown completed successfully"
        else
          MESSAGE="❌ Infrastructure teardown failed"
        fi
        
        aws sns publish \
          --topic-arn ${{ secrets.SNS_TOPIC_ARN }} \
          --subject "MLOps Infrastructure Teardown" \
          --message "$MESSAGE"
```

## 8. Chi phí ước tính sau khi tối ưu

Dưới đây là chi phí dự kiến sau khi áp dụng các biện pháp tối ưu:

| Thành phần | Trước tối ưu | Sau tối ưu | Tiết kiệm (%) |
|------------|--------------|------------|---------------|
| EKS NodeGroup | 28.80 USD | 2.40 USD | 92% |
| S3 Storage | 1.15 USD | 0.63 USD | 45% |
| CloudWatch Logs | 2.50 USD | 0.75 USD | 70% |
| LoadBalancer | 19.44 USD | 5.40 USD | 72% |
| ECR Storage | 0.50 USD | 0.20 USD | 60% |
| **Tổng chi phí** | **52.39 USD** | **9.38 USD** | **82%** |

## 9. Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- **EC2 Spot Instance**: Cấu hình EKS NodeGroup và SageMaker sử dụng Spot
- **S3 Lifecycle**: Triển khai lifecycle policies cho data và artifacts
- **Auto Schedule**: Lambda + EventBridge để tự động dừng/khởi động tài nguyên
- **Budget Alert**: Thiết lập giám sát chi phí và cảnh báo
- **ECR Lifecycle**: Chỉ giữ lại 3 phiên bản image mới nhất
- **Log Retention**: CloudWatch logs retention policy 30 ngày
- **Complete Teardown**: Script để xóa hoàn toàn tài nguyên sau demo

### 📊 Verification Steps

```bash
# 1. Kiểm tra EKS NodeGroup đang sử dụng Spot
aws eks describe-nodegroup \
  --cluster-name retail-forecast-cluster \
  --nodegroup-name retail-forecast-nodes \
  --query 'nodegroup.capacityType'

# 2. Kiểm tra S3 lifecycle policy
aws s3api get-bucket-lifecycle-configuration \
  --bucket retail-forecast-data-dev

# 3. Kiểm tra CloudWatch logs retention
aws logs describe-log-groups \
  --log-group-name-prefix /aws/retail-forecast \
  --query 'logGroups[*].[logGroupName,retentionInDays]'

# 4. Kiểm tra ECR lifecycle policy
aws ecr get-lifecycle-policy \
  --repository-name retail-forecast

# 5. Kiểm tra AWS Budget đã được tạo
aws budgets describe-budgets \
  --account-id $(aws sts get-caller-identity --query "Account" --output text)

# 6. Thực hiện teardown và kiểm tra xóa sạch
cd aws/scripts
./teardown.sh

# Kiểm tra không còn resources
aws eks list-clusters --query 'clusters[*]' | grep retail
aws s3 ls | grep retail-forecast
aws ecr describe-repositories --query 'repositories[*].repositoryName' | grep retail
aws sagemaker list-endpoints --query 'Endpoints[*].EndpointName' | grep retail
```

## Tổng kết

Quản lý chi phí hiệu quả là một trong những khía cạnh quan trọng nhất của MLOps trên AWS. Bằng việc áp dụng các chiến lược tối ưu như Spot Instances, S3 lifecycle policies, auto scheduling, và resource cleanup, chúng ta có thể giảm chi phí vận hành đến 80% mà vẫn duy trì được khả năng mở rộng và hiệu suất của hệ thống.

Các biện pháp tối ưu chi phí này không chỉ giúp tiết kiệm ngân sách mà còn giúp hệ thống MLOps hoạt động hiệu quả hơn thông qua việc tự động hóa quản lý tài nguyên, giám sát chi phí, và thực hiện các best practices trong quản lý vòng đời của dữ liệu và container images.

**Kết quả chính:**
- Tiết kiệm 82% chi phí vận hành (~9.38 USD/tháng so với ~52.39 USD/tháng)
- Tự động schedule start/stop resources
- S3 lifecycle policies tiết kiệm storage
- CloudWatch logs retention 7 ngày
- Complete teardown script

---
