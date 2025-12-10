---
title: "Amazon ECR Container Registry (MLOps)"
date: 2024-01-01T00:00:00Z
weight: 7
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Task 6 Goal:** Create a private Amazon ECR repository for the Retail API image, enforce image hygiene (immutability, scan-on-push, lifecycle), and publish a production-ready FastAPI container image to ECR.
{{% /notice %}}

## 0) Inputs from previous tasks

- Production AWS Region: **ap-southeast-1**
- AWS Account ID: **842676018087**
- Source folder: `server/` (FastAPI inference API)
- Target ECR repository: `mlops/retail-api`

---

## 1) Create the ECR repository (recommended settings)

### 1.1 Create repo (CLI)

```bash
export AWS_REGION="ap-southeast-1"
export REPO_NAME="mlops/retail-api"

aws ecr create-repository   --region "$AWS_REGION"   --repository-name "$REPO_NAME"   --image-tag-mutability IMMUTABLE   --image-scanning-configuration scanOnPush=true
```

> If the repo already exists, the command will fail. In that case, just update settings in the console.

### 1.2 Enable scan-on-push and immutability (Console)

ECR → Repositories → `mlops/retail-api` → **Edit**:

- **Image tag mutability:** Immutable
- **Scan on push:** Enabled

---

## 2) Add lifecycle policy (keep prod/dev tags, expire untagged)

Create `ecr-lifecycle.json`:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 images for prod/dev tag prefixes",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod", "dev", "staging", "latest"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}
```

Apply it:

```bash
aws ecr put-lifecycle-policy   --region "$AWS_REGION"   --repository-name "$REPO_NAME"   --lifecycle-policy-text file://ecr-lifecycle.json
```

---

## 3) Build a production-ready FastAPI container image

### 3.1 Multi-stage Dockerfile (non-root + healthcheck)

Create `server/Dockerfile`:

```dockerfile
# ---- builder ----
FROM python:3.11-slim AS builder
WORKDIR /app

ENV PIP_DISABLE_PIP_VERSION_CHECK=1     PIP_NO_CACHE_DIR=1     PYTHONDONTWRITEBYTECODE=1     PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends     build-essential  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN python -m venv /opt/venv  && /opt/venv/bin/pip install --upgrade pip  && /opt/venv/bin/pip install -r requirements.txt

# ---- runtime ----
FROM python:3.11-slim AS runtime
WORKDIR /app

ENV PATH="/opt/venv/bin:$PATH"     PYTHONDONTWRITEBYTECODE=1     PYTHONUNBUFFERED=1

# Create non-root user
RUN addgroup --system app && adduser --system --ingroup app app

COPY --from=builder /opt/venv /opt/venv
COPY . .

# Healthcheck endpoint should exist in your FastAPI app
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3   CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()" || exit 1

EXPOSE 8000
USER app

# Uvicorn entrypoint
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create `server/.dockerignore`:

```gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
*.log
.env
.venv/
venv/
tests/
dist/
build/
.git/
.github/
*.ipynb
```

---

## 4) Authenticate and push to ECR

### 4.1 Bash (Linux/macOS) script

Create `scripts/ecr_push.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

AWS_REGION="ap-southeast-1"
AWS_ACCOUNT_ID="842676018087"
REPO_NAME="mlops/retail-api"
IMAGE_TAG="${1:-latest}"

ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"

echo "[1/4] ECR login..."
aws ecr get-login-password --region "$AWS_REGION"   | docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

echo "[2/4] Build image..."
docker build -t "${REPO_NAME}:${IMAGE_TAG}" -f server/Dockerfile server/

echo "[3/4] Tag image..."
docker tag "${REPO_NAME}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"

echo "[4/4] Push image..."
docker push "${ECR_URI}:${IMAGE_TAG}"

echo "✅ Pushed: ${ECR_URI}:${IMAGE_TAG}"
```

### 4.2 PowerShell (Windows) script

Create `scripts/ecr_push.ps1`:

```powershell
param(
  [string]$ImageTag = "latest"
)

$AWS_REGION = "ap-southeast-1"
$AWS_ACCOUNT_ID = "842676018087"
$REPO_NAME = "mlops/retail-api"
$ECR_URI = "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME"

Write-Host "[1/4] ECR login..."
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

Write-Host "[2/4] Build image..."
docker build -t "$REPO_NAME:$ImageTag" -f server/Dockerfile server/

Write-Host "[3/4] Tag image..."
docker tag "$REPO_NAME:$ImageTag" "$ECR_URI:$ImageTag"

Write-Host "[4/4] Push image..."
docker push "$ECR_URI:$ImageTag"

Write-Host "✅ Pushed: $ECR_URI:$ImageTag"
```

---

## 5) Verify in console / CLI

```bash
aws ecr describe-repositories --region ap-southeast-1 --query 'repositories[?repositoryName==`mlops/retail-api`].repositoryUri' --output text

aws ecr describe-images   --region ap-southeast-1   --repository-name mlops/retail-api   --max-items 10
```

---

## 6) Cleanup scripts (optional)

### 6.1 Delete local build caches (safe)

```bash
docker image prune -f
docker builder prune -f
```

### 6.2 Delete untagged images in ECR (manual)

```bash
aws ecr list-images   --region ap-southeast-1   --repository-name mlops/retail-api   --filter tagStatus=UNTAGGED   --query 'imageIds[*]'   --output json > untagged.json

aws ecr batch-delete-image   --region ap-southeast-1   --repository-name mlops/retail-api   --image-ids file://untagged.json
```

---

## 7) Rough cost notes (ap-southeast-1)

- **ECR storage** is billed per GB-month; lifecycle policies keep it low.
- **Image scanning** is enabled; costs depend on usage (keep scans on push, but avoid pushing too frequently).
- **Data transfer to EKS** is typically intra-region; cross-region pulls cost more (avoid by keeping everything in ap-southeast-1).

{{% notice success %}}
**✅ Task 6 Complete (ECR):**

- Private repo `mlops/retail-api` created in `ap-southeast-1`
- Scan-on-push + tag immutability enabled
- Lifecycle policy applied
- FastAPI image built (multi-stage, non-root, healthcheck) and pushed to ECR
  {{% /notice %}}
