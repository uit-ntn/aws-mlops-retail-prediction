---
title: "CloudWatch Monitoring (EKS + SageMaker)"
date: 2024-01-01T00:00:00Z
weight: 11
chapter: false
pre: "<b>10. </b>"
---

{{% notice info %}}
**🎯 Task 10 Goal:** Enable basic monitoring for the EKS cluster and Retail API, including logs, Container Insights, alarms, and SageMaker training visibility in CloudWatch.
{{% /notice %}}

## 0) Inputs

- Region: `ap-southeast-1`
- Cluster: `mlops-retail-cluster`
- Namespace: `mlops`
- App label: `app=retail-api`

---

## 1) Enable Container Insights (recommended)

### Option A: EKS add-on (recommended)

Install the CloudWatch Observability add-on:

```bash
aws eks create-addon   --cluster-name mlops-retail-cluster   --addon-name amazon-cloudwatch-observability   --region ap-southeast-1
```

Verify:

```bash
aws eks describe-addon   --cluster-name mlops-retail-cluster   --addon-name amazon-cloudwatch-observability   --region ap-southeast-1
```

---

## 2) Log group retention (7–30 days)

Set retention to keep costs down:

```bash
aws logs put-retention-policy   --log-group-name "/aws/eks/mlops-retail-cluster/cluster"   --retention-in-days 14   --region ap-southeast-1 || true

aws logs put-retention-policy   --log-group-name "/aws/containerinsights/mlops-retail-cluster/application"   --retention-in-days 14   --region ap-southeast-1 || true
```

> Log group names vary depending on how you enabled logging. Use `aws logs describe-log-groups` to confirm actual names.

---

## 3) Basic alarms (CPU/Memory) for the cluster

Example alarm (CPU high):

```bash
aws cloudwatch put-metric-alarm   --region ap-southeast-1   --alarm-name "EKS-Cluster-CPU-High-mlops-retail-cluster"   --alarm-description "High CPU on EKS cluster"   --namespace "ContainerInsights"   --metric-name "node_cpu_utilization"   --dimensions Name=ClusterName,Value=mlops-retail-cluster   --statistic Average   --period 60   --evaluation-periods 5   --threshold 80   --comparison-operator GreaterThanThreshold   --treat-missing-data notBreaching
```

Example alarm (Memory high):

```bash
aws cloudwatch put-metric-alarm   --region ap-southeast-1   --alarm-name "EKS-Cluster-Memory-High-mlops-retail-cluster"   --alarm-description "High memory on EKS cluster"   --namespace "ContainerInsights"   --metric-name "node_memory_utilization"   --dimensions Name=ClusterName,Value=mlops-retail-cluster   --statistic Average   --period 60   --evaluation-periods 5   --threshold 80   --comparison-operator GreaterThanThreshold   --treat-missing-data notBreaching
```

> You can also set alarms per namespace or workload once metrics are confirmed in CloudWatch Metrics → ContainerInsights.

---

## 4) Logs Insights queries (quick triage)

### 4.1 EKS app errors (example)

```sql
fields @timestamp, @message
| filter @message like /ERROR|Exception|Traceback/
| sort @timestamp desc
| limit 50
```

### 4.2 Pod restarts / probe failures (via Kubernetes events)

If you ship events to CloudWatch, query for `Unhealthy` or `Back-off` patterns.

---

## 5) SageMaker training logs & metrics

SageMaker training jobs stream logs to CloudWatch Logs. You can:

- open CloudWatch → Log groups → search for `/aws/sagemaker/TrainingJobs`
- query by TrainingJobName keywords

Example Insights query:

```sql
fields @timestamp, @message
| filter @message like /TrainingJobName/
| sort @timestamp desc
| limit 100
```

---

## 6) Verification commands

```bash
# Check cluster add-on
kubectl -n amazon-cloudwatch get pods || true

# Check workload status
kubectl get pods -n mlops -l app=retail-api
kubectl logs -n mlops deploy/retail-api --tail=200
```

---

## 7) Cleanup

```bash
# Remove alarms
aws cloudwatch delete-alarms   --region ap-southeast-1   --alarm-names     "EKS-Cluster-CPU-High-mlops-retail-cluster"     "EKS-Cluster-Memory-High-mlops-retail-cluster" || true

# Remove add-on (optional)
aws eks delete-addon   --cluster-name mlops-retail-cluster   --addon-name amazon-cloudwatch-observability   --region ap-southeast-1 || true
```

---

## 8) Cost notes

- **Logs ingestion + storage** are the main CloudWatch costs.
- Keep retention low (7–30 days) for workshop/demo.
- Be careful with Logs Insights scans (large log groups → higher query cost).

{{% notice success %}}
**✅ Task 10 Complete (Monitoring):**

- Container Insights enabled (CloudWatch Observability add-on)
- Log retention configured
- CPU/Memory alarms created
- SageMaker training log visibility documented
  {{% /notice %}}
