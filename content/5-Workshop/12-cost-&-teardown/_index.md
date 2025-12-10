---
title: "Cost Management & Teardown"
date: 2024-01-01T00:00:00Z
weight: 13
chapter: false
pre: "<b>12. </b>"
---

{{% notice info %}}
**🎯 Task 12 Goal:** Manage and optimize the cost of the full AWS MLOps infrastructure:
{{% /notice %}}

- Reduce compute costs (EKS/EC2, SageMaker, Load Balancer)
- Automatically scale down or remove idle resources
- Apply lifecycle policies for S3 data and ECR images
- Ensure the pipeline remains effective while keeping costs low

> Workshop standard region: **ap-southeast-1**. Keep S3/ECR/EKS/SageMaker in the same region to avoid cross-region cost and S3 301 redirect issues.

---

## 1) Operational cost drivers

A full MLOps stack (EKS + SageMaker + ALB + CloudWatch + S3 + ECR) can grow expensive if left running 24/7.
The biggest drivers are compute and always-on networking components.

### 1.1 Cost-by-component (typical)

| Service                     |             Unoptimized cost | Root cause              | Fix                                             |
| --------------------------- | ---------------------------: | ----------------------- | ----------------------------------------------- |
| **EKS NodeGroup (EC2)**     | On-demand nodes running 24/7 | Always-on compute       | Spot + autoscaling + scheduled scale-to-zero    |
| **SageMaker training**      | Bigger instances + long runs | No cost controls        | Managed Spot Training + right-size + limit runs |
| **S3 storage**              |    All data in Standard tier | No lifecycle            | Lifecycle + Intelligent-Tiering                 |
| **CloudWatch logs**         |          Unlimited retention | log bloat               | Retention 7–30 days + reduce verbosity          |
| **Load Balancer (ALB/NLB)** |         Running continuously | Demo endpoint always on | Enable only during demos                        |
| **ECR storage**             |          Many image versions | no lifecycle            | Lifecycle policy (keep only latest)             |

---

## 2) Use Spot Instances (EKS + SageMaker)

### 2.1 Spot for EKS NodeGroup (Terraform example)

```hcl
module "eks_managed_node_group" {
  source = "terraform-aws-modules/eks/aws//modules/eks-managed-node-group"

  name            = "retail-forecast-nodes"
  cluster_name    = module.eks.cluster_name
  cluster_version = module.eks.cluster_version

  capacity_type   = "SPOT"
  instance_types  = ["t3.medium", "t3a.medium", "t2.medium"]

  min_size     = 1
  max_size     = 5
  desired_size = 2

  labels = { app = "retail-api" }
}
```

### 2.2 Managed Spot Training for SageMaker

In your training job definition, enable managed spot:

```python
training_params["EnableManagedSpotTraining"] = True
training_params["StoppingCondition"]["MaxWaitTimeInSeconds"] = 3900
```

### 2.3 Handle Spot interruptions

1. PodDisruptionBudget (PDB):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: retail-api-pdb
  namespace: mlops
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: retail-api
```

2. Cluster Autoscaler (if you use it) to react to lost capacity.

---

## 3) S3 Lifecycle + Intelligent-Tiering

### 3.1 Lifecycle policy (Terraform example)

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_lifecycle" {
  bucket = aws_s3_bucket.retail_forecast_data.id

  rule {
    id     = "raw-tiering"
    status = "Enabled"
    filter { prefix = "raw/" }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }
  }

  rule {
    id     = "silver-tiering"
    status = "Enabled"
    filter { prefix = "silver/" }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }
  }

  rule {
    id     = "artifacts-archive"
    status = "Enabled"
    filter { prefix = "artifacts/" }

    transition { days = 90  storage_class = "GLACIER_IR" }
    transition { days = 180 storage_class = "DEEP_ARCHIVE" }
  }

  rule {
    id     = "logs-archive"
    status = "Enabled"
    filter { prefix = "logs/" }

    transition { days = 90  storage_class = "GLACIER_IR" }
    transition { days = 180 storage_class = "DEEP_ARCHIVE" }
  }

  rule {
    id     = "tmp-cleanup"
    status = "Enabled"
    filter { prefix = "tmp/" }
    expiration { days = 7 }
  }
}
```

---

## 4) Auto stop/start off-hours (Lambda + EventBridge)

### 4.1 Scaling EKS NodeGroup down to zero

EKS managed node groups are backed by an Auto Scaling Group.
A cost-effective approach is to schedule **DesiredCapacity=0** outside working hours.

Example Lambda pseudocode (high-level):

- describe nodegroup → get ASG name
- update ASG → MinSize=0, DesiredCapacity=0 (stop)
- update ASG → MinSize=1–2, DesiredCapacity=1–2 (start)

> IMPORTANT: If your API uses a public LoadBalancer, it may still incur cost even if pods scale to zero. Delete the Service/Ingress when not demoing.

### 4.2 Scheduler times (Vietnam time)

Your timezone is **Asia/Ho_Chi_Minh (UTC+7)**.
If you use EventBridge cron (UTC), convert carefully.

Example (stop at 22:00 Vietnam time = 15:00 UTC):

```text
cron(0 15 * * ? *)
```

---

## 5) Cost visibility & alerts

### 5.1 Tagging strategy

Tag everything:

- `Project=RetailMLOps`
- `Environment=dev|staging|prod`
- `Owner=<your-name>`
- `CostCenter=<optional>`

### 5.2 AWS Budgets

Create a monthly budget (small limit for workshop) and alert at 80% / 100%.

---

## 6) ECR + CloudWatch optimization

### 6.1 ECR lifecycle policy (keep small)

Keep only the last few images and expire untagged quickly (see Task 6).

### 6.2 CloudWatch log retention

Set 7–30 days retention for:

- `/aws/eks/...`
- `/aws/containerinsights/...`
- `/aws/sagemaker/TrainingJobs`

---

## 7) Full teardown (delete everything after demo)

### 7.1 Manual teardown order (safe)

1. Delete Kubernetes namespace(s) (`mlops`, etc.)
2. Confirm Load Balancers are deleted
3. Delete SageMaker endpoints (if any)
4. Delete ECR images (optional)
5. Destroy infra (Terraform / CloudFormation)
6. Delete CloudWatch alarms/log groups (optional)

### 7.2 One-shot teardown script (example)

Create `aws/scripts/teardown.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

AWS_REGION="ap-southeast-1"

echo "[1/6] Delete Kubernetes namespace (mlops)..."
kubectl delete namespace mlops --ignore-not-found=true

echo "[2/6] Delete SageMaker endpoints (filter by name)..."
ENDPOINTS=$(aws sagemaker list-endpoints --region "$AWS_REGION" --query "Endpoints[].EndpointName" --output text | tr '\t' '\n' | grep -E "retail|forecast" || true)
for ep in $ENDPOINTS; do
  echo "Deleting endpoint: $ep"
  aws sagemaker delete-endpoint --region "$AWS_REGION" --endpoint-name "$ep" || true
done

echo "[3/6] (Optional) Delete ECR untagged images..."
# See Task 6 for batch-delete example

echo "[4/6] Terraform destroy (if used)..."
# cd aws/infra && terraform destroy -auto-approve

echo "[5/6] Done."
```

---

## 8) Expected monthly cost after optimization (example)

| Component       |    Before |    After | Savings |
| --------------- | --------: | -------: | ------: |
| EKS NodeGroup   |     28.80 |     2.40 |     92% |
| S3 Storage      |      1.15 |     0.63 |     45% |
| CloudWatch Logs |      2.50 |     0.75 |     70% |
| Load Balancer   |     19.44 |     5.40 |     72% |
| ECR Storage     |      0.50 |     0.20 |     60% |
| **Total**       | **52.39** | **9.38** | **82%** |

---

## 9) Verification commands

```bash
# Node group capacity type (Spot)
aws eks describe-nodegroup   --region ap-southeast-1   --cluster-name mlops-retail-cluster   --nodegroup-name retail-ng-dev   --query 'nodegroup.capacityType' --output text

# S3 lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket <YOUR_BUCKET>

# CloudWatch retention
aws logs describe-log-groups   --region ap-southeast-1   --log-group-name-prefix "/aws/eks/"   --query 'logGroups[*].[logGroupName,retentionInDays]'   --output table
```

{{% notice success %}}
**✅ Task 12 Complete (Cost & Teardown):**

- Spot for EKS + Managed Spot Training for SageMaker (where appropriate)
- S3 lifecycle + Intelligent-Tiering for data/artifacts/logs
- Scheduled scale-to-zero / demo-only LoadBalancer policy
- Budgets + tagging strategy for cost visibility
- Complete teardown checklist + scripts
  {{% /notice %}}
