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
 
{{% notice tip %}}
**Tip:** Chọn retention hợp lý (ví dụ 7–30 ngày) cho log groups dựa trên yêu cầu debug và compliance để cân bằng chi phí và khả năng truy vết.
{{% /notice %}}

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
 
  {{% notice info %}}
  **Info:** Nên cấu hình action cho alarms (ví dụ SNS topic hoặc webhook tới PagerDuty) để tự động thông báo cho team khi alarm trigger. Kiểm tra kỹ metric namespace và dimension trước khi bật alarm.
  {{% /notice %}}

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
 
{{% notice warning %}}
**Warning:** SageMaker training logs có thể sinh lượng lớn dữ liệu (đặc biệt khi in nhiều dòng hoặc debug verbose). Giới hạn level logging trong training scripts và cân nhắc filter/metric extraction để tránh chi phí ingestion/storage đột biến.
{{% /notice %}}

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

{{% notice success %}}
**🎯 Task 10 Complete - CloudWatch Monitoring**
- **Container Insights** enabled cho EKS cluster
- **CloudWatch Alarms** configured cho CPU/Memory
- **SageMaker training logs** với custom metrics
- **Log Groups** với retention policy
{{% /notice %}}

{{% notice info %}}
**Note:** Trước khi xóa log groups hoặc metric filters, export hoặc snapshot dashboard quan trọng (ví dụ Grafana/CSV) nếu cần lưu trữ cho audit hoặc so sánh sau này.
{{% /notice %}}

## 5. Clean Up Resources

### 5.1 Xóa CloudWatch Alarms

```bash
# List tất cả alarms liên quan đến dự án
aws cloudwatch describe-alarms \
  --alarm-name-prefix "EKS-" \
  --query 'MetricAlarms[].AlarmName' \
  --output table

# Xóa specific alarms
aws cloudwatch delete-alarms \
  --alarm-names "EKS-HighCPU" "EKS-HighMemory"

# Xóa alarms liên quan đến SageMaker
aws cloudwatch describe-alarms \
  --alarm-name-prefix "SageMaker-" \
  --query 'MetricAlarms[].AlarmName' \
  --output text | tr '\t' '\n' | xargs -I {} aws cloudwatch delete-alarms --alarm-names {}
```

### 5.2 Xóa Log Groups và Metric Filters

```bash
# List tất cả log groups liên quan
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/containerinsights/mlops-retail-cluster" \
  --query 'logGroups[].logGroupName' \
  --output table

# Xóa Container Insights log groups
aws logs delete-log-group \
  --log-group-name "/aws/containerinsights/mlops-retail-cluster/application"

aws logs delete-log-group \
  --log-group-name "/aws/containerinsights/mlops-retail-cluster/dataplane"

aws logs delete-log-group \
  --log-group-name "/aws/containerinsights/mlops-retail-cluster/host"

# Xóa SageMaker training log groups
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/sagemaker/TrainingJobs" \
  --query 'logGroups[].logGroupName' \
  --output text | tr '\t' '\n' | while read log_group; do
    echo "Deleting log group: $log_group"
    aws logs delete-log-group --log-group-name "$log_group"
done

# Xóa EKS cluster log groups
aws logs delete-log-group --log-group-name "/aws/eks/mlops-retail-cluster/cluster" || true
```

### 5.3 Disable Container Insights

```bash
# Disable Container Insights addon
aws eks delete-addon \
  --cluster-name mlops-retail-cluster \
  --addon-name amazon-cloudwatch-observability

# Xóa CloudWatch agent từ EKS cluster
kubectl delete namespace amazon-cloudwatch || true

# Remove CloudWatch agent DaemonSet nếu còn
kubectl delete daemonset cloudwatch-agent -n amazon-cloudwatch || true
kubectl delete configmap cwagentconfig -n amazon-cloudwatch || true
kubectl delete serviceaccount cloudwatch-agent -n amazon-cloudwatch || true
```

### 5.4 Xóa Custom Metrics và Dashboards

```bash
# List custom metrics trong RetailMLOps namespace
aws cloudwatch list-metrics \
  --namespace "RetailMLOps" \
  --query 'Metrics[].MetricName' \
  --output table

# Xóa custom metric filters
aws logs describe-metric-filters \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --query 'metricFilters[].filterName' \
  --output text | tr '\t' '\n' | while read filter; do
    echo "Deleting metric filter: $filter"
    aws logs delete-metric-filter \
      --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
      --filter-name "$filter"
done

# List và xóa CloudWatch Dashboards
aws cloudwatch list-dashboards \
  --dashboard-name-prefix "RetailMLOps" \
  --query 'DashboardEntries[].DashboardName' \
  --output text | tr '\t' '\n' | while read dashboard; do
    echo "Deleting dashboard: $dashboard"
    aws cloudwatch delete-dashboards --dashboard-names "$dashboard"
done
```

### 5.5 Verification Clean Up

```bash
# Verify alarms đã bị xóa
aws cloudwatch describe-alarms \
  --alarm-name-prefix "EKS-" \
  --query 'MetricAlarms[].AlarmName'

# Verify log groups đã bị xóa
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/containerinsights/mlops-retail-cluster" \
  --query 'logGroups[].logGroupName'

# Verify Container Insights addon đã bị remove
aws eks describe-addon \
  --cluster-name mlops-retail-cluster \
  --addon-name amazon-cloudwatch-observability || echo "Addon removed successfully"

# Check namespaces trong EKS
kubectl get namespaces | grep cloudwatch
```

## 6. Bảng giá CloudWatch Monitoring (ap-southeast-1)

### 6.1. CloudWatch Logs Pricing

| Service | Ingestion | Storage | Analysis |
|---------|-----------|---------|----------|
| **Log Ingestion** | $0.67/GB | - | - |
| **Log Storage** | - | $0.033/GB/month | - |
| **Log Insights** | - | - | $0.0067/GB scanned |

### 6.2. Container Insights Pricing

| Component | Volume | Monthly Cost | Description |
|-----------|---------|--------------|-------------|
| **Performance Logs** | ~2GB/month | $1.34 | Node và Pod metrics |
| **Application Logs** | ~1GB/month | $0.67 | Container stdout/stderr |
| **Storage (30 days)** | 3GB total | $0.099 | Log retention |
| **Insights Queries** | ~0.5GB scanned | $0.0034 | Monthly analysis |
| **Total Container Insights** | | **$2.11/month** | Per cluster |

### 6.3. CloudWatch Alarms và Metrics

| Feature | Quantity | Unit Cost | Monthly Cost |
|---------|----------|-----------|--------------|
| **Standard Metrics** | Free | $0 | $0 |
| **Custom Metrics** | 10 metrics | $0.30/metric | $3.00 |
| **Alarms** | 5 alarms | $0.10/alarm | $0.50 |
| **API Calls** | 1M calls | $0.01/1000 calls | $10.00 |
| **Total Monitoring** | | | **$13.50/month** |

### 6.4. SageMaker Training Logs

```bash
# Estimate SageMaker training log volume
# Typical training job generates ~100MB logs
# With 4 training jobs per month
```

| Training Scenario | Log Volume | Ingestion Cost | Storage Cost | Total Cost |
|-------------------|------------|----------------|--------------|------------|
| **Single Training** | 100MB | $0.067 | $0.0033 | $0.070 |
| **4 Jobs/month** | 400MB | $0.268 | $0.013 | $0.281 |
| **Daily Training** | 3GB/month | $2.01 | $0.099 | $2.11 |

### 6.5. CloudWatch Dashboards

| Dashboard Type | Widgets | Monthly Cost | Use Case |
|----------------|---------|--------------|----------|
| **Basic Dashboard** | 3 widgets | $3.00 | EKS cluster overview |
| **Detailed Dashboard** | 10 widgets | $3.00 | Full MLOps monitoring |
| **Custom Dashboard** | 20 widgets | $3.00 | Multi-service view |

*Note: CloudWatch Dashboard pricing is $3.00/month per dashboard, regardless of widget count*

### 6.6. Log Insights Query Costs

```bash
# Example query costs for different scenarios
```

| Query Type | Data Scanned | Cost per Query | Monthly Queries | Monthly Cost |
|------------|---------------|----------------|-----------------|--------------|
| **Error Analysis** | 100MB | $0.00067 | 50 | $0.034 |
| **Performance Review** | 500MB | $0.0034 | 20 | $0.068 |
| **Full Log Search** | 2GB | $0.013 | 10 | $0.13 |
| **Total Query Cost** | | | | **$0.23/month** |

### 6.7. Data Transfer Costs

| Transfer Type | Volume | Cost | Monthly Estimate |
|---------------|---------|------|------------------|
| **CloudWatch API** | 1GB/month | $0.12/GB | $0.12 |
| **Log Streaming** | 500MB/month | $0.12/GB | $0.06 |
| **Cross-AZ Logs** | 200MB/month | $0.01/GB | $0.002 |
| **Total Transfer** | | | **$0.18/month** |

### 6.8. Retention Cost Analysis

| Retention Period | Storage Multiplier | Cost Impact | Use Case |
|------------------|-------------------|-------------|----------|
| **1 day** | 1x | Baseline | Development |
| **7 days** | 7x | 7x storage cost | Testing |
| **30 days** | 30x | 30x storage cost | Production |
| **1 year** | 365x | 365x storage cost | Compliance |

```bash
# Example: 1GB/day logs with different retention
# 1 day: $0.033 storage cost
# 30 days: $0.99 storage cost
# 1 year: $12.05 storage cost
```

### 6.9. Cost Optimization Strategies

**Log Filtering:**
```bash
# Chỉ log ERROR và WARNING levels
aws logs put-metric-filter \
  --log-group-name "/aws/containerinsights/mlops-retail-cluster/application" \
  --filter-name "ErrorsOnly" \
  --filter-pattern "ERROR WARN" \
  --metric-transformations metricName=ErrorCount,metricNamespace=RetailMLOps,metricValue=1
```

**Intelligent Log Routing:**
```yaml
# Kubernetes fluent-bit configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.var.log.containers.*error*
        log_group_name /aws/containerinsights/cluster/errors
        
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.var.log.containers.*info*
        log_group_name /aws/containerinsights/cluster/info
        retention_in_days 7
```

**Metric Sampling:**
```bash
# Sample metrics every 5 minutes instead of 1 minute
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-HighCPU-Sampled" \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80
```

### 6.10. Tổng kết chi phí Task 10

**Scenario 1: Basic Monitoring (Development)**

| Component | Monthly Cost |
|-----------|--------------|
| Container Insights (basic) | $2.11 |
| 3 CloudWatch Alarms | $0.30 |
| 1 Dashboard | $3.00 |
| Log storage (7 days) | $0.23 |
| **Total Development** | **$5.64/month** |

**Scenario 2: Production Monitoring**

| Component | Monthly Cost |
|-----------|--------------|
| Container Insights (full) | $6.00 |
| 10 CloudWatch Alarms | $1.00 |
| 2 Dashboards | $6.00 |
| SageMaker training logs | $0.28 |
| Log storage (30 days) | $0.99 |
| Custom metrics | $3.00 |
| Log Insights queries | $0.23 |
| **Total Production** | **$17.50/month** |

**Scenario 3: Enterprise Monitoring**

| Component | Monthly Cost |
|-----------|--------------|
| Container Insights (multi-cluster) | $12.00 |
| 25 CloudWatch Alarms | $2.50 |
| 5 Dashboards | $15.00 |
| Daily SageMaker training | $2.11 |
| Extended log retention | $5.00 |
| Heavy custom metrics | $10.00 |
| Frequent queries | $2.00 |
| **Total Enterprise** | **$48.61/month** |

### 6.11. Monitoring Cost Commands

```bash
# Track CloudWatch costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon CloudWatch"]}}'

# Monitor log ingestion volume
aws logs describe-log-groups \
  --query 'logGroups[?storedBytes > `1000000`].[logGroupName,storedBytes]' \
  --output table

# Check metric usage
aws cloudwatch list-metrics \
  --namespace "RetailMLOps" \
  --query 'length(Metrics)'
```

{{% notice info %}}
**💰 Cost Summary cho Task 10:**
- **Development:** $5.64/month (basic monitoring)
- **Production:** $17.50/month (full monitoring) 
- **Enterprise:** $48.61/month (extensive monitoring)
- **Optimization potential:** 30-50% cost reduction với log filtering và retention tuning
{{% /notice %}}

## 🎬 Video thực hiện Task 10

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=PGu3QTZm_r8&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=9" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 11: CI/CD Pipeline](../11-cicd-jenkins-travis/)