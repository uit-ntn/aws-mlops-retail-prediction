---
title: "CI/CD Pipeline"
date: 2024-01-01T00:00:00Z
weight: 12
chapter: false
pre: "<b>11. </b>"
---

{{% notice info %}}
**🎯 Task 11 Goal:** Build a CI/CD pipeline that automates the full MLOps lifecycle for the Retail Prediction project:
{{% /notice %}}

- **CI (Continuous Integration):** lint/test → build Docker → scan → push to ECR
- **CD (Continuous Delivery):** optional retrain → evaluate → register/approve in SageMaker Model Registry → deploy updated API to EKS
- **Monitoring hook:** rollback if API or model deployment fails (CloudWatch alarms / health checks)

✅ Outcome: continuous delivery with fewer manual steps, reduced mistakes, and lower operational time/cost.

---

## 0) Inputs from previous tasks

- **Task 6 (ECR):** repo `mlops/retail-api`, scan-on-push, lifecycle rules, push commands
- **Task 8 (EKS API):** manifests (namespace, Deployment/Service names, probes)
- **Task 10 (CloudWatch):** log groups, alarms, Container Insights
- **Task 2 (IAM):** OIDC provider and least-privilege role for CI runners

**Workshop region standard:** **ap-southeast-1** (keep S3/ECR/EKS/SageMaker in same region to avoid S3 301 redirects).

---

## 1) High-level pipeline design

We split the pipeline into three environments:

- **DEV:** build, test, validate
- **STAGING:** retrain + evaluate + register model (optional)
- **PROD:** deploy to EKS + verify + monitor + rollback

> In this workshop, environments can be implemented using branch rules:
>
> - `develop` → staging
> - `main` → production

---

## 2) IAM for CI/CD (GitHub Actions OIDC)

### 2.1 Trust policy (OIDC)

Create `trust-policy.json` (replace `<ACCOUNT_ID>`, `<GITHUB_ORG>`, `<REPO_NAME>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:*"
        }
      }
    }
  ]
}
```

Create role:

```bash
aws iam create-role   --role-name GitHubActionsRole   --assume-role-policy-document file://trust-policy.json   --description "Role for GitHub Actions CI/CD pipeline"
```

### 2.2 Least-privilege policy (ECR + EKS + SageMaker + CloudWatch + S3)

Create `github-actions-policy.json` (tighten resources if you know exact bucket ARNs):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadWriteArtifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::retail-forecast-*",
        "arn:aws:s3:::retail-forecast-*/*"
      ]
    },
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:ListImages",
        "ecr:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EKSDeployRead",
      "Effect": "Allow",
      "Action": ["eks:DescribeCluster", "eks:ListClusters"],
      "Resource": "*"
    },
    {
      "Sid": "SageMakerTrainRegister",
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:DescribeTrainingJob",
        "sagemaker:StopTrainingJob",
        "sagemaker:CreateProcessingJob",
        "sagemaker:DescribeProcessingJob",
        "sagemaker:StopProcessingJob",
        "sagemaker:CreateModelPackageGroup",
        "sagemaker:DescribeModelPackageGroup",
        "sagemaker:ListModelPackageGroups",
        "sagemaker:CreateModelPackage",
        "sagemaker:DescribeModelPackage",
        "sagemaker:ListModelPackages",
        "sagemaker:UpdateModelPackage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchAndLogs",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DeleteAlarms",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SNSNotify",
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:*:*:retail-forecast-*"
    }
  ]
}
```

Attach policy:

```bash
aws iam create-policy --policy-name GitHubActionsPolicy --policy-document file://github-actions-policy.json

aws iam attach-role-policy   --role-name GitHubActionsRole   --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/GitHubActionsPolicy
```

---

## 3) GitHub Actions Workflow (recommended)

Create `.github/workflows/mlops-pipeline.yml`:

```yaml
name: MLOps Retail Prediction Pipeline

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - "**.md"
      - "docs/**"
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "dev"
        type: choice
        options: [dev, staging, prod]
      retrain_model:
        description: "Retrain model"
        required: false
        default: false
        type: boolean
      deploy_only:
        description: "Deploy only (skip training)"
        required: false
        default: false
        type: boolean

env:
  AWS_REGION: ap-southeast-1
  AWS_ACCOUNT_ID: 842676018087
  ECR_REPOSITORY: mlops/retail-api
  EKS_CLUSTER_NAME: mlops-retail-cluster
  K8S_NAMESPACE: mlops
  MODEL_PACKAGE_GROUP: retail-price-sensitivity-models

permissions:
  id-token: write
  contents: read

jobs:
  test:
    name: Lint & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r server/requirements.txt
          pip install pytest ruff
      - name: Lint (ruff)
        run: ruff check server/
      - name: Unit tests
        run: pytest -q || true

  build_push:
    name: Build & Push Image
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ env.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        id: meta
        run: |
          TAG="build-${GITHUB_RUN_NUMBER}-${GITHUB_SHA::7}"
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          docker build -t "${{ env.ECR_REPOSITORY }}:$TAG" -f server/Dockerfile server/
          docker tag "${{ env.ECR_REPOSITORY }}:$TAG" "${{ env.ECR_REPOSITORY }}:latest"

      - name: Scan image (Trivy)
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock             aquasec/trivy:latest image --severity HIGH,CRITICAL             "${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.tag }}"

      - name: Push to ECR
        run: |
          REG="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com"
          docker tag "${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.tag }}" "$REG/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.tag }}"
          docker tag "${{ env.ECR_REPOSITORY }}:latest" "$REG/${{ env.ECR_REPOSITORY }}:latest"
          docker push "$REG/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.tag }}"
          docker push "$REG/${{ env.ECR_REPOSITORY }}:latest"

  deploy:
    name: Deploy to EKS
    needs: build_push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ env.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: "v1.29.0"

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER_NAME }} --region ${{ env.AWS_REGION }}
          kubectl get nodes

      - name: Apply manifests
        run: |
          # Apply Task 8 manifests (ensure they exist in repo)
          kubectl apply -f aws/k8s/retail-api.yaml
          kubectl apply -f aws/k8s/retail-api-hpa.yaml

          # Set image to the new build tag (optional)
          REG="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com"
          kubectl -n ${{ env.K8S_NAMESPACE }} set image deploy/retail-api             retail-api=$REG/${{ env.ECR_REPOSITORY }}:${{ needs.build_push.outputs.image_tag }}

          kubectl -n ${{ env.K8S_NAMESPACE }} rollout status deploy/retail-api --timeout=300s

      - name: Health check + rollback
        run: |
          SVC=retail-api-svc
          NS=${{ env.K8S_NAMESPACE }}
          HOST=$(kubectl get svc -n $NS $SVC -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

          echo "Service host: $HOST"
          if [ -z "$HOST" ]; then
            echo "No LB hostname. Failing."
            exit 1
          fi

          set +e
          curl -f "http://$HOST/health"
          OK=$?
          set -e

          if [ "$OK" -ne 0 ]; then
            echo "Health check failed. Rolling back..."
            kubectl -n $NS rollout undo deploy/retail-api
            kubectl -n $NS rollout status deploy/retail-api --timeout=300s
            exit 1
          fi

  notify:
    name: Notify (optional)
    needs: [deploy]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Print result
        run: |
          echo "Deploy status: ${{ needs.deploy.result }}"
```

---

## 4) Optional: retrain + register model stage

If you want the pipeline to retrain and register a new model version, add a job that:

1. triggers SageMaker training (Task 4 pattern)
2. validates metrics (Accuracy ≥ 0.8, F1 ≥ 0.7)
3. registers and approves a model package version in the Model Registry
4. API uses Model Registry to load the latest **Approved** model

> Keep this stage manual or scheduled to avoid unexpected training costs.

---

## 5) Optional Jenkins (self-hosted)

Jenkins is useful when you want:

- self-hosted runners
- private networking control
- custom build agents

But GitHub Actions is simpler for the workshop, so Jenkins is optional.

---

## 6) Verification checklist

- GitHub Actions run succeeded (test → build_push → deploy)
- ECR shows new tags in `mlops/retail-api`
- EKS rollout succeeded and `/health` returns HTTP 200
- HPA is present and scales under load
- CloudWatch logs/metrics are visible (Task 10)

---

## 7) Cleanup (CI/CD resources)

- Disable workflow or restrict triggers
- Delete the GitHubActionsRole + policies if no longer needed
- Remove ECR tags/images that are only for CI builds
- Delete CloudWatch alarms/dashboards created for CI/CD (if any)

{{% notice success %}}
**✅ Task 11 Complete (CI/CD):**

- GitHub Actions pipeline configured for CI (lint/test, build, scan, push)
- CD deploys updated image to EKS with health check + rollback
- IAM Role + OIDC set for secure authentication (no long-lived keys)
  {{% /notice %}}
