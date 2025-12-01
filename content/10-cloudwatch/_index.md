---
title: "CloudWatch Monitoring"
date: 2024-01-01T00:00:00Z
weight: 11
chapter: false
pre: "<b>10. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 10:**
{{% /notice %}}

Thiết lập CloudWatch monitoring cơ bản cho EKS cluster và retail API.

## 1. Container Insights

```bash
# Enable Container Insights cho EKS cluster
aws eks update-addon \
  --cluster-name mlops-retail-cluster \
  --addon-name amazon-cloudwatch-observability \
  --addon-version v1.0.0-eksbuild.1

# Tạo Log Groups
aws logs create-log-group \
  --log-group-name /aws/containerinsights/mlops-retail-cluster/application \
  --retention-in-days 7
```

## 2. Basic Alarms

```bash
# CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-HighCPU" \
  --alarm-description "CPU > 80%" \
  --metric-name node_cpu_utilization \
  --namespace ContainerInsights \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ClusterName,Value=mlops-retail-cluster

# Memory alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-HighMemory" \
  --alarm-description "Memory > 85%" \
  --metric-name node_memory_utilization \
  --namespace ContainerInsights \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ClusterName,Value=mlops-retail-cluster
```

## 3. SageMaker Training Logs

```bash
# Tạo SageMaker Training Job với logging configuration
aws sagemaker create-training-job \
  --training-job-name retail-prediction-training \
  --algorithm-specification TrainingImage=842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:v3 \
  --role-arn arn:aws:iam::842676018087:role/SageMakerExecutionRole \
  --input-data-config '[{
    "ChannelName": "training",
    "DataSource": {
      "S3DataSource": {
        "S3DataType": "S3Prefix",
        "S3Uri": "s3://retail-prediction-bucket/data/",
        "S3DataDistributionType": "FullyReplicated"
      }
    },
    "ContentType": "text/csv",
    "CompressionType": "None"
  }]' \
  --output-data-config S3OutputPath=s3://retail-prediction-bucket/model-artifacts/ \
  --resource-config '{
    "InstanceType": "ml.m5.xlarge",
    "InstanceCount": 1,
    "VolumeSizeInGB": 50
  }' \
  --stopping-condition MaxRuntimeInSeconds=3600

# Log Analysis với CloudWatch Insights
aws logs start-query \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --start-time $(date -d '24 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @message like "loss" 
    | parse @message "loss: *," as loss_value 
    | parse @message "epoch * " as epoch_num 
    | stats avg(loss_value) as avg_loss by bin(1h)'

# Tạo metric filter cho training loss
aws logs put-metric-filter \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --filter-name "TrainingLoss" \
  --filter-pattern '"loss: "' \
  --metric-transformations \
      metricName=TrainingLoss,metricNamespace=RetailMLOps,metricValue=1,unit=None
```

## 4. Verification

```bash
# Check Container Insights
kubectl get pods -n amazon-cloudwatch

# Check alarms
aws cloudwatch describe-alarms --alarm-names "EKS-HighCPU" "EKS-HighMemory"

# Check logs
aws logs describe-log-groups --log-group-name-prefix "/aws/containerinsights/mlops-retail-cluster"

# Check SageMaker training logs
aws logs describe-log-streams \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training"
```

## Chi phí ước tính

| Thành phần | Ước tính |
|------------|----------|
| CloudWatch Logs | ~3.38 USD/tháng |
| Container Insights | ~6.00 USD/tháng |
| **Tổng** | **~9.38 USD/tháng** |

---

**Next Step**: [Task 11: Cost Optimization & Teardown](../11-cost-optimization/)