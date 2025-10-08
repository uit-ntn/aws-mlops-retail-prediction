---
title: "Docker Build & Push"
date: 2024-01-01T00:00:00+07:00
weight: 7
chapter: false
pre: "<b>7. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 7:**

Đóng gói ứng dụng inference (FastAPI) thành Docker image và đẩy lên Amazon ECR để chuẩn bị triển khai trên EKS, đảm bảo reproducibility và automation trong CI/CD pipeline.
{{% /notice %}}

## Tổng quan

**Docker Build & Push** là quy trình quan trọng trong MLOps pipeline, chuyển đổi source code thành container images sẵn sàng triển khai. Task này tập trung vào việc containerize ML inference application và tích hợp với ECR registry.

### Kiến trúc Docker Build Pipeline

{{< mermaid >}}
graph TB
    subgraph "Source Code"
        SRC[server/<br/>FastAPI Application]
        REQ[requirements.txt<br/>Dependencies]
        DOCKER[Dockerfile<br/>Build Instructions]
    end
    
    subgraph "Build Process"
        BUILD[Docker Build<br/>Multi-stage]
        TAG[Image Tagging<br/>latest + git-sha]
        TEST[Image Testing<br/>Health Checks]
    end
    
    subgraph "ECR Registry"
        PUSH[Docker Push<br/>Multiple Tags]
        SCAN[Vulnerability Scan<br/>On Push]
        STORE[Image Storage<br/>Lifecycle Managed]
    end
    
    subgraph "EKS Deployment"
        PULL[Image Pull<br/>From EKS Nodes]
        DEPLOY[Pod Deployment<br/>Application Running]
    end
    
    SRC --> BUILD
    REQ --> BUILD
    DOCKER --> BUILD
    BUILD --> TAG
    TAG --> TEST
    TEST --> PUSH
    PUSH --> SCAN
    SCAN --> STORE
    STORE --> PULL
    PULL --> DEPLOY
{{< /mermaid >}}

### Thành phần chính

1. **FastAPI Application**: ML inference service
2. **Dockerfile**: Multi-stage build optimization
3. **Requirements Management**: Python dependencies
4. **Build Automation**: Scripts và CI/CD integration
5. **Image Versioning**: Git-based tagging strategy

---

## 1. Alternative: Manual Build via Console/Local

### 1.1. FastAPI Inference Application

**Tạo `server/main.py` - FastAPI Application:**

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
import pandas as pd
import boto3
import logging
from typing import List, Dict, Any
import os
from datetime import datetime
import uvicorn

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# FastAPI app initialization
app = FastAPI(
    title="Retail Forecast ML API",
    description="Machine Learning inference API for retail sales forecasting",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Pydantic models
class ForecastRequest(BaseModel):
    """Request model for forecast prediction"""
    features: Dict[str, Any]
    horizon: int = 30  # forecast horizon in days
    
class ForecastResponse(BaseModel):
    """Response model for forecast prediction"""
    predictions: List[float]
    confidence_intervals: List[Dict[str, float]]
    metadata: Dict[str, Any]

class HealthResponse(BaseModel):
    """Health check response model"""
    status: str
    timestamp: str
    version: str
    model_loaded: bool

# Global variables
model = None
model_metadata = {}

# Model loading
def load_model():
    """Load ML model from S3 or local storage"""
    global model, model_metadata
    
    try:
        # Try loading from S3 first
        if os.getenv("AWS_REGION"):
            s3_client = boto3.client('s3')
            bucket_name = os.getenv("MODEL_BUCKET", "mlops-retail-forecast-dev-models")
            model_key = os.getenv("MODEL_KEY", "models/retail_forecast_model.pkl")
            
            try:
                # Download model from S3
                s3_client.download_file(bucket_name, model_key, "/tmp/model.pkl")
                model = joblib.load("/tmp/model.pkl")
                logger.info(f"Model loaded from S3: s3://{bucket_name}/{model_key}")
            except Exception as e:
                logger.warning(f"Failed to load from S3: {e}")
                # Fall back to local model
                model = joblib.load("model/retail_forecast_model.pkl")
                logger.info("Model loaded from local storage")
        else:
            # Local development
            model = joblib.load("model/retail_forecast_model.pkl")
            logger.info("Model loaded from local storage")
            
        model_metadata = {
            "model_type": "RandomForestRegressor",
            "features": ["month", "season", "promotion", "holiday", "temperature"],
            "target": "sales",
            "training_date": "2024-01-01",
            "model_version": "v1.0.0"
        }
        
    except Exception as e:
        logger.error(f"Failed to load model: {e}")
        model = None

@app.on_event("startup")
async def startup_event():
    """Application startup event"""
    logger.info("Starting Retail Forecast API...")
    load_model()

@app.get("/", response_model=Dict[str, str])
async def root():
    """Root endpoint"""
    return {
        "message": "Retail Forecast ML API",
        "version": "1.0.0",
        "status": "running"
    }

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    return HealthResponse(
        status="healthy" if model is not None else "unhealthy",
        timestamp=datetime.utcnow().isoformat(),
        version="1.0.0",
        model_loaded=model is not None
    )

@app.get("/ready")
async def readiness_check():
    """Readiness check for Kubernetes"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    return {"status": "ready"}

@app.post("/predict", response_model=ForecastResponse)
async def predict(request: ForecastRequest):
    """Main prediction endpoint"""
    
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    try:
        # Process input features
        features_df = pd.DataFrame([request.features])
        
        # Make predictions
        predictions = model.predict(features_df).tolist()
        
        # Generate confidence intervals (mock implementation)
        confidence_intervals = []
        for pred in predictions:
            confidence_intervals.append({
                "lower": pred * 0.9,
                "upper": pred * 1.1,
                "confidence": 0.95
            })
        
        return ForecastResponse(
            predictions=predictions,
            confidence_intervals=confidence_intervals,
            metadata={
                "model_version": model_metadata.get("model_version", "unknown"),
                "prediction_timestamp": datetime.utcnow().isoformat(),
                "horizon": request.horizon
            }
        )
        
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")

@app.get("/model/info")
async def model_info():
    """Get model information"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    return model_metadata

@app.post("/model/reload")
async def reload_model(background_tasks: BackgroundTasks):
    """Reload model in background"""
    background_tasks.add_task(load_model)
    return {"message": "Model reload initiated"}

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        log_level="info"
    )
```

**Tạo `server/requirements.txt` - Python Dependencies:**

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pandas==2.1.4
numpy==1.25.2
scikit-learn==1.3.2
joblib==1.3.2
boto3==1.34.0
botocore==1.34.0
python-multipart==0.0.6
aiofiles==23.2.1
httpx==0.25.2
pytest==7.4.3
pytest-asyncio==0.21.1
requests==2.31.0
```

{{< imgborder src="/images/07-docker-build/01-fastapi-application.png" title="FastAPI inference application structure" >}}

### 1.2. Dockerfile Configuration

**Tạo `server/Dockerfile` - Multi-stage Build:**

```dockerfile
# Multi-stage build for production optimization
FROM python:3.9-slim as builder

# Set build arguments
ARG BUILD_DATE
ARG GIT_COMMIT
ARG GIT_BRANCH

# Add metadata labels
LABEL maintainer="MLOps Team"
LABEL build_date=$BUILD_DATE
LABEL git_commit=$GIT_COMMIT
LABEL git_branch=$GIT_BRANCH
LABEL description="Retail Forecast ML Inference API"

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.9-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create non-root user for security
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Copy Python packages from builder stage
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Create necessary directories
RUN mkdir -p /app/model /app/logs && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Set environment variables
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1
ENV AWS_DEFAULT_REGION=ap-southeast-1

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

{{< imgborder src="/images/07-docker-build/02-dockerfile-multistage.png" title="Multi-stage Dockerfile configuration" >}}

### 1.3. Docker Build Process (Manual)

1. **Navigate to Server Directory:**
   ```bash
   cd server/
   ```

2. **Build Docker Image:**
   ```bash
   # Get current git information
   GIT_COMMIT=$(git rev-parse --short HEAD)
   GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
   BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
   
   # Build with build arguments
   docker build \
     --build-arg BUILD_DATE=$BUILD_DATE \
     --build-arg GIT_COMMIT=$GIT_COMMIT \
     --build-arg GIT_BRANCH=$GIT_BRANCH \
     -t retail-forecast:$GIT_COMMIT \
     .
   ```

{{< imgborder src="/images/07-docker-build/03-docker-build-manual.png" title="Manual Docker build process" >}}

3. **Test Image Locally:**
   ```bash
   # Run container locally
   docker run -d \
     --name retail-forecast-test \
     -p 8000:8000 \
     retail-forecast:$GIT_COMMIT
   
   # Test health endpoint
   curl http://localhost:8000/health
   
   # Test API endpoint
   curl -X POST http://localhost:8000/predict \
     -H "Content-Type: application/json" \
     -d '{"features": {"month": 1, "season": "winter", "promotion": 1, "holiday": 0, "temperature": 10}}'
   
   # Stop test container
   docker stop retail-forecast-test
   docker rm retail-forecast-test
   ```

{{< imgborder src="/images/07-docker-build/04-local-testing.png" title="Local container testing" >}}

### 1.4. Manual Push to ECR

1. **ECR Authentication:**
   ```bash
   # Get AWS account ID
   AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
   AWS_REGION="ap-southeast-1"
   ECR_REPOSITORY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/retail-forecast"
   
   # Login to ECR
   aws ecr get-login-password --region $AWS_REGION | \
     docker login --username AWS --password-stdin $ECR_REPOSITORY
   ```

2. **Tag Images:**
   ```bash
   # Tag for ECR
   docker tag retail-forecast:$GIT_COMMIT $ECR_REPOSITORY:latest
   docker tag retail-forecast:$GIT_COMMIT $ECR_REPOSITORY:$GIT_COMMIT
   docker tag retail-forecast:$GIT_COMMIT $ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT
   ```

3. **Push to ECR:**
   ```bash
   # Push all tags
   docker push $ECR_REPOSITORY:latest
   docker push $ECR_REPOSITORY:$GIT_COMMIT
   docker push $ECR_REPOSITORY:$GIT_BRANCH-$GIT_COMMIT
   ```

{{< imgborder src="/images/07-docker-build/05-manual-ecr-push.png" title="Manual push to ECR" >}}

### 1.5. Verify in ECR Console

1. **Check ECR Repository:**
   - Navigate to ECR Console
   - Select `retail-forecast` repository
   - Verify images với multiple tags

{{< imgborder src="/images/07-docker-build/06-verify-ecr-images.png" title="Verify images in ECR Console" >}}

2. **Review Image Details:**
   - Check image sizes
   - Verify vulnerability scan results
   - Review push timestamps

{{< imgborder src="/images/07-docker-build/07-image-details.png" title="Review ECR image details" >}}

{{% notice success %}}
**🎯 Manual Build Complete!**

Docker image đã được build và push thành công với:
- ✅ Multi-stage optimized Dockerfile
- ✅ FastAPI inference application
- ✅ Multiple image tags (latest, git-sha, branch-sha)
- ✅ Security best practices (non-root user)
- ✅ Health checks configured
- ✅ Images verified in ECR
{{% /notice %}}

---

## 2. Automated Build Scripts

### 2.1. Linux/macOS Build Script

**Tạo `scripts/build-and-push.sh`:**

```bash
#!/bin/bash

set -e

# Script configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
SERVER_DIR="$PROJECT_ROOT/server"

# Default values
PROJECT_NAME="mlops-retail-forecast"
ENVIRONMENT="dev"
AWS_REGION="ap-southeast-1"
ECR_REPO_NAME="retail-forecast"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Functions
log_info() {
    echo -e "${BLUE}ℹ️  $1${NC}"
}

log_success() {
    echo -e "${GREEN}✅ $1${NC}"
}

log_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

log_error() {
    echo -e "${RED}❌ $1${NC}"
}

# Parse command line arguments
while [[ $# -gt 0 ]]; do
    case $1 in
        -e|--environment)
            ENVIRONMENT="$2"
            shift 2
            ;;
        -r|--region)
            AWS_REGION="$2"
            shift 2
            ;;
        --no-push)
            NO_PUSH=true
            shift
            ;;
        --no-cache)
            NO_CACHE="--no-cache"
            shift
            ;;
        -h|--help)
            echo "Usage: $0 [OPTIONS]"
            echo "Options:"
            echo "  -e, --environment ENV    Environment (default: dev)"
            echo "  -r, --region REGION      AWS Region (default: ap-southeast-1)"
            echo "  --no-push                Build only, don't push to ECR"
            echo "  --no-cache               Build without Docker cache"
            echo "  -h, --help               Show this help"
            exit 0
            ;;
        *)
            log_error "Unknown option: $1"
            exit 1
            ;;
    esac
done

# Validate AWS CLI
if ! command -v aws &> /dev/null; then
    log_error "AWS CLI not found. Please install AWS CLI."
    exit 1
fi

# Validate Docker
if ! command -v docker &> /dev/null; then
    log_error "Docker not found. Please install Docker."
    exit 1
fi

# Check if we're in a git repository
if ! git rev-parse --git-dir > /dev/null 2>&1; then
    log_error "Not in a git repository"
    exit 1
fi

# Get AWS Account ID
log_info "Getting AWS account information..."
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
if [ -z "$AWS_ACCOUNT_ID" ]; then
    log_error "Failed to get AWS account ID. Check your AWS credentials."
    exit 1
fi

# ECR repository URL
ECR_REPOSITORY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME"

# Get Git information
GIT_COMMIT=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
GIT_DIRTY=""
if [[ -n $(git status --porcelain) ]]; then
    GIT_DIRTY="-dirty"
    log_warning "Working directory has uncommitted changes"
fi

# Build timestamp
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')

# Image tags
IMAGE_TAG_LATEST="latest"
IMAGE_TAG_COMMIT="${GIT_COMMIT}${GIT_DIRTY}"
IMAGE_TAG_BRANCH="${GIT_BRANCH}-${GIT_COMMIT}${GIT_DIRTY}"
IMAGE_TAG_ENV="${ENVIRONMENT}-${GIT_COMMIT}${GIT_DIRTY}"

log_info "Build Configuration:"
echo "  Project: $PROJECT_NAME"
echo "  Environment: $ENVIRONMENT"
echo "  AWS Region: $AWS_REGION"
echo "  AWS Account: $AWS_ACCOUNT_ID"
echo "  ECR Repository: $ECR_REPOSITORY"
echo "  Git Commit: $GIT_COMMIT$GIT_DIRTY"
echo "  Git Branch: $GIT_BRANCH"
echo "  Build Date: $BUILD_DATE"
echo ""

# Check if server directory exists
if [ ! -d "$SERVER_DIR" ]; then
    log_error "Server directory not found: $SERVER_DIR"
    exit 1
fi

# Check required files
REQUIRED_FILES=("$SERVER_DIR/Dockerfile" "$SERVER_DIR/main.py" "$SERVER_DIR/requirements.txt")
for file in "${REQUIRED_FILES[@]}"; do
    if [ ! -f "$file" ]; then
        log_error "Required file not found: $file"
        exit 1
    fi
done

# Change to server directory
cd "$SERVER_DIR"

# Build Docker image
log_info "Building Docker image..."
docker build \
    $NO_CACHE \
    --build-arg BUILD_DATE="$BUILD_DATE" \
    --build-arg GIT_COMMIT="$GIT_COMMIT" \
    --build-arg GIT_BRANCH="$GIT_BRANCH" \
    --build-arg ENVIRONMENT="$ENVIRONMENT" \
    -t "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" \
    .

if [ $? -eq 0 ]; then
    log_success "Docker image built successfully"
else
    log_error "Docker build failed"
    exit 1
fi

# Tag images
log_info "Tagging images..."
docker tag "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" "$ECR_REPOSITORY:$IMAGE_TAG_LATEST"
docker tag "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" "$ECR_REPOSITORY:$IMAGE_TAG_COMMIT"
docker tag "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" "$ECR_REPOSITORY:$IMAGE_TAG_BRANCH"
docker tag "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" "$ECR_REPOSITORY:$IMAGE_TAG_ENV"

# Test image locally (optional)
log_info "Testing image locally..."
CONTAINER_ID=$(docker run -d -p 8001:8000 "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT")
sleep 5

if curl -f http://localhost:8001/health &> /dev/null; then
    log_success "Local health check passed"
else
    log_warning "Local health check failed, but continuing..."
fi

docker stop "$CONTAINER_ID" > /dev/null
docker rm "$CONTAINER_ID" > /dev/null

# Push to ECR (unless --no-push specified)
if [ "$NO_PUSH" != "true" ]; then
    log_info "Logging into ECR..."
    aws ecr get-login-password --region "$AWS_REGION" | \
        docker login --username AWS --password-stdin "$ECR_REPOSITORY"
    
    if [ $? -eq 0 ]; then
        log_success "ECR login successful"
    else
        log_error "ECR login failed"
        exit 1
    fi
    
    log_info "Pushing images to ECR..."
    
    # Push all tags
    TAGS=("$IMAGE_TAG_LATEST" "$IMAGE_TAG_COMMIT" "$IMAGE_TAG_BRANCH" "$IMAGE_TAG_ENV")
    for tag in "${TAGS[@]}"; do
        log_info "Pushing $ECR_REPOSITORY:$tag"
        docker push "$ECR_REPOSITORY:$tag"
        if [ $? -eq 0 ]; then
            log_success "Pushed $tag"
        else
            log_error "Failed to push $tag"
            exit 1
        fi
    done
else
    log_warning "Skipping ECR push (--no-push specified)"
fi

# Clean up local images
log_info "Cleaning up local images..."
docker rmi "$ECR_REPO_NAME:$IMAGE_TAG_COMMIT" > /dev/null 2>&1 || true
for tag in "${TAGS[@]}"; do
    docker rmi "$ECR_REPOSITORY:$tag" > /dev/null 2>&1 || true
done

log_success "Build and push completed successfully!"
echo ""
log_info "Image URLs:"
for tag in "${TAGS[@]}"; do
    echo "  📍 $ECR_REPOSITORY:$tag"
done

echo ""
log_info "To deploy this image:"
echo "  kubectl set image deployment/retail-forecast-api api=$ECR_REPOSITORY:$IMAGE_TAG_COMMIT"
echo ""
```

### 2.2. Windows PowerShell Script

**Tạo `scripts/build-and-push.ps1`:**

```powershell
[CmdletBinding()]
param(
    [string]$Environment = "dev",
    [string]$Region = "ap-southeast-1",
    [switch]$NoPush,
    [switch]$NoCache,
    [switch]$Help
)

# Show help
if ($Help) {
    Write-Host @"
Usage: .\build-and-push.ps1 [OPTIONS]

Options:
  -Environment ENV     Environment (default: dev)
  -Region REGION       AWS Region (default: ap-southeast-1)
  -NoPush              Build only, don't push to ECR
  -NoCache             Build without Docker cache
  -Help                Show this help

Examples:
  .\build-and-push.ps1
  .\build-and-push.ps1 -Environment prod -Region us-west-2
  .\build-and-push.ps1 -NoPush
"@
    exit 0
}

# Script configuration
$ScriptDir = Split-Path -Parent $MyInvocation.MyCommand.Definition
$ProjectRoot = Split-Path -Parent $ScriptDir
$ServerDir = Join-Path $ProjectRoot "server"

$ProjectName = "mlops-retail-forecast"
$EcrRepoName = "retail-forecast"

# Helper functions
function Write-ColoredOutput {
    param([string]$Message, [string]$Color = "White")
    
    $colors = @{
        "Red" = [ConsoleColor]::Red
        "Green" = [ConsoleColor]::Green
        "Yellow" = [ConsoleColor]::Yellow
        "Blue" = [ConsoleColor]::Blue
        "White" = [ConsoleColor]::White
    }
    
    Write-Host $Message -ForegroundColor $colors[$Color]
}

function Log-Info { param([string]$Message) Write-ColoredOutput "ℹ️  $Message" "Blue" }
function Log-Success { param([string]$Message) Write-ColoredOutput "✅ $Message" "Green" }
function Log-Warning { param([string]$Message) Write-ColoredOutput "⚠️  $Message" "Yellow" }
function Log-Error { param([string]$Message) Write-ColoredOutput "❌ $Message" "Red" }

# Validate dependencies
Log-Info "Validating dependencies..."

if (-not (Get-Command aws -ErrorAction SilentlyContinue)) {
    Log-Error "AWS CLI not found. Please install AWS CLI."
    exit 1
}

if (-not (Get-Command docker -ErrorAction SilentlyContinue)) {
    Log-Error "Docker not found. Please install Docker."
    exit 1
}

if (-not (Get-Command git -ErrorAction SilentlyContinue)) {
    Log-Error "Git not found. Please install Git."
    exit 1
}

# Check if we're in a git repository
try {
    git rev-parse --git-dir | Out-Null
} catch {
    Log-Error "Not in a git repository"
    exit 1
}

# Get AWS Account ID
Log-Info "Getting AWS account information..."
try {
    $AwsAccountId = aws sts get-caller-identity --query Account --output text
    if (-not $AwsAccountId) {
        throw "Failed to get account ID"
    }
} catch {
    Log-Error "Failed to get AWS account ID. Check your AWS credentials."
    exit 1
}

# ECR repository URL
$EcrRepository = "$AwsAccountId.dkr.ecr.$Region.amazonaws.com/$EcrRepoName"

# Get Git information
$GitCommit = git rev-parse --short HEAD
$GitBranch = git rev-parse --abbrev-ref HEAD
$GitDirty = ""

# Check for uncommitted changes
$GitStatus = git status --porcelain
if ($GitStatus) {
    $GitDirty = "-dirty"
    Log-Warning "Working directory has uncommitted changes"
}

# Build timestamp
$BuildDate = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

# Image tags
$ImageTagLatest = "latest"
$ImageTagCommit = "$GitCommit$GitDirty"
$ImageTagBranch = "$GitBranch-$GitCommit$GitDirty"
$ImageTagEnv = "$Environment-$GitCommit$GitDirty"

Log-Info "Build Configuration:"
Write-Host "  Project: $ProjectName"
Write-Host "  Environment: $Environment"
Write-Host "  AWS Region: $Region"
Write-Host "  AWS Account: $AwsAccountId"
Write-Host "  ECR Repository: $EcrRepository"
Write-Host "  Git Commit: $GitCommit$GitDirty"
Write-Host "  Git Branch: $GitBranch"
Write-Host "  Build Date: $BuildDate"
Write-Host ""

# Check server directory
if (-not (Test-Path $ServerDir)) {
    Log-Error "Server directory not found: $ServerDir"
    exit 1
}

# Check required files
$RequiredFiles = @(
    (Join-Path $ServerDir "Dockerfile"),
    (Join-Path $ServerDir "main.py"),
    (Join-Path $ServerDir "requirements.txt")
)

foreach ($file in $RequiredFiles) {
    if (-not (Test-Path $file)) {
        Log-Error "Required file not found: $file"
        exit 1
    }
}

# Change to server directory
Set-Location $ServerDir

# Build Docker image
Log-Info "Building Docker image..."

$BuildArgs = @(
    "build"
    "--build-arg", "BUILD_DATE=$BuildDate"
    "--build-arg", "GIT_COMMIT=$GitCommit"
    "--build-arg", "GIT_BRANCH=$GitBranch"
    "--build-arg", "ENVIRONMENT=$Environment"
    "-t", "$EcrRepoName`:$ImageTagCommit"
)

if ($NoCache) {
    $BuildArgs += "--no-cache"
}

$BuildArgs += "."

try {
    & docker @BuildArgs
    if ($LASTEXITCODE -ne 0) {
        throw "Docker build failed"
    }
    Log-Success "Docker image built successfully"
} catch {
    Log-Error "Docker build failed: $_"
    exit 1
}

# Tag images
Log-Info "Tagging images..."
$Tags = @($ImageTagLatest, $ImageTagCommit, $ImageTagBranch, $ImageTagEnv)

foreach ($tag in $Tags) {
    docker tag "$EcrRepoName`:$ImageTagCommit" "$EcrRepository`:$tag"
    if ($LASTEXITCODE -ne 0) {
        Log-Error "Failed to tag image with $tag"
        exit 1
    }
}

# Test image locally
Log-Info "Testing image locally..."
try {
    $ContainerId = docker run -d -p 8001:8000 "$EcrRepoName`:$ImageTagCommit"
    Start-Sleep -Seconds 5
    
    try {
        $response = Invoke-WebRequest -Uri "http://localhost:8001/health" -TimeoutSec 10
        if ($response.StatusCode -eq 200) {
            Log-Success "Local health check passed"
        } else {
            Log-Warning "Local health check failed, but continuing..."
        }
    } catch {
        Log-Warning "Local health check failed, but continuing..."
    } finally {
        docker stop $ContainerId | Out-Null
        docker rm $ContainerId | Out-Null
    }
} catch {
    Log-Warning "Failed to test image locally, but continuing..."
}

# Push to ECR (unless -NoPush specified)
if (-not $NoPush) {
    Log-Info "Logging into ECR..."
    
    try {
        $loginPassword = aws ecr get-login-password --region $Region
        $loginPassword | docker login --username AWS --password-stdin $EcrRepository
        
        if ($LASTEXITCODE -ne 0) {
            throw "ECR login failed"
        }
        Log-Success "ECR login successful"
    } catch {
        Log-Error "ECR login failed: $_"
        exit 1
    }
    
    Log-Info "Pushing images to ECR..."
    
    foreach ($tag in $Tags) {
        Log-Info "Pushing $EcrRepository`:$tag"
        docker push "$EcrRepository`:$tag"
        
        if ($LASTEXITCODE -eq 0) {
            Log-Success "Pushed $tag"
        } else {
            Log-Error "Failed to push $tag"
            exit 1
        }
    }
} else {
    Log-Warning "Skipping ECR push (-NoPush specified)"
}

# Clean up local images
Log-Info "Cleaning up local images..."
docker rmi "$EcrRepoName`:$ImageTagCommit" 2>$null | Out-Null
foreach ($tag in $Tags) {
    docker rmi "$EcrRepository`:$tag" 2>$null | Out-Null
}

Log-Success "Build and push completed successfully!"
Write-Host ""
Log-Info "Image URLs:"
foreach ($tag in $Tags) {
    Write-Host "  📍 $EcrRepository`:$tag"
}

Write-Host ""
Log-Info "To deploy this image:"
Write-Host "  kubectl set image deployment/retail-forecast-api api=$EcrRepository`:$ImageTagCommit"
Write-Host ""
```

### 2.3. Docker Compose for Local Development

**Tạo `docker-compose.yml`:**

```yaml
version: '3.8'

services:
  retail-forecast-api:
    build:
      context: ./server
      dockerfile: Dockerfile
      args:
        BUILD_DATE: ${BUILD_DATE:-2024-01-01T00:00:00Z}
        GIT_COMMIT: ${GIT_COMMIT:-local}
        GIT_BRANCH: ${GIT_BRANCH:-local}
    ports:
      - "8000:8000"
    environment:
      - AWS_DEFAULT_REGION=ap-southeast-1
      - AWS_REGION=ap-southeast-1
      - MODEL_BUCKET=mlops-retail-forecast-dev-models
      - MODEL_KEY=models/retail_forecast_model.pkl
    volumes:
      - ./server/model:/app/model:ro
      - ./server/logs:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    
  # Optional: Redis for caching
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

---

## 3. CI/CD Integration

### 3.1. GitHub Actions Workflow

**Tạo `.github/workflows/docker-build.yml`:**

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main, develop]
    paths:
      - 'server/**'
      - '.github/workflows/docker-build.yml'
  pull_request:
    branches: [main]
    paths:
      - 'server/**'

env:
  AWS_REGION: ap-southeast-1
  ECR_REPOSITORY: retail-forecast

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Extract metadata
      id: meta
      run: |
        echo "git-commit=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
        echo "git-branch=${GITHUB_REF#refs/heads/}" >> $GITHUB_OUTPUT
        echo "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT
        echo "image-tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: ./server
        file: ./server/Dockerfile
        push: true
        tags: |
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.git-branch }}-${{ steps.meta.outputs.image-tag }}
        build-args: |
          BUILD_DATE=${{ steps.meta.outputs.build-date }}
          GIT_COMMIT=${{ steps.meta.outputs.git-commit }}
          GIT_BRANCH=${{ steps.meta.outputs.git-branch }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Test container
      run: |
        # Run container for testing
        docker run -d --name test-container \
          -p 8000:8000 \
          ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}
        
        # Wait for container to start
        sleep 30
        
        # Test health endpoint
        curl -f http://localhost:8000/health || exit 1
        
        # Stop test container
        docker stop test-container
        docker rm test-container

    - name: Output image details
      run: |
        echo "🚀 Successfully built and pushed:"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.image-tag }}"
        echo "📍 ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.git-branch }}-${{ steps.meta.outputs.image-tag }}"
```

### 3.2. Jenkins Pipeline

**Tạo `aws/Jenkinsfile`:**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-southeast-1'
        ECR_REPOSITORY = 'retail-forecast'
        PROJECT_NAME = 'mlops-retail-forecast'
        ENVIRONMENT = 'dev'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
                    env.BUILD_DATE = sh(returnStdout: true, script: 'date -u +%Y-%m-%dT%H:%M:%SZ').trim()
                }
            }
        }
        
        stage('Validate') {
            steps {
                script {
                    // Check required files
                    def requiredFiles = [
                        'server/Dockerfile',
                        'server/main.py', 
                        'server/requirements.txt'
                    ]
                    
                    requiredFiles.each { file ->
                        if (!fileExists(file)) {
                            error("Required file not found: ${file}")
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('server') {
                        def imageTag = "${env.GIT_COMMIT}"
                        def imageName = "${env.ECR_REPOSITORY}:${imageTag}"
                        
                        sh """
                            docker build \
                                --build-arg BUILD_DATE='${env.BUILD_DATE}' \
                                --build-arg GIT_COMMIT='${env.GIT_COMMIT}' \
                                --build-arg GIT_BRANCH='${env.GIT_BRANCH}' \
                                -t ${imageName} \
                                .
                        """
                        
                        env.IMAGE_NAME = imageName
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }
        
        stage('Test Image') {
            steps {
                script {
                    // Run container for testing
                    def containerId = sh(
                        returnStdout: true,
                        script: "docker run -d -p 8001:8000 ${env.IMAGE_NAME}"
                    ).trim()
                    
                    try {
                        // Wait for container to start
                        sleep(30)
                        
                        // Test health endpoint
                        sh "curl -f http://localhost:8001/health"
                        
                        echo "✅ Container health check passed"
                        
                    } finally {
                        // Always clean up
                        sh "docker stop ${containerId} || true"
                        sh "docker rm ${containerId} || true"
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Run Trivy security scan
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v \$PWD:/workspace \
                            aquasec/trivy:latest \
                            image --format json --output /workspace/trivy-report.json \
                            ${env.IMAGE_NAME}
                    """
                    
                    // Archive scan results
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }
        
        stage('Push to ECR') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    // Get AWS account ID
                    def awsAccountId = sh(
                        returnStdout: true,
                        script: 'aws sts get-caller-identity --query Account --output text'
                    ).trim()
                    
                    def ecrRegistry = "${awsAccountId}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    def ecrRepository = "${ecrRegistry}/${env.ECR_REPOSITORY}"
                    
                    // Login to ECR
                    sh """
                        aws ecr get-login-password --region ${env.AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ecrRegistry}
                    """
                    
                    // Tag images
                    def tags = ['latest', env.IMAGE_TAG, "${env.GIT_BRANCH}-${env.IMAGE_TAG}"]
                    
                    tags.each { tag ->
                        sh "docker tag ${env.IMAGE_NAME} ${ecrRepository}:${tag}"
                        sh "docker push ${ecrRepository}:${tag}"
                        echo "✅ Pushed ${ecrRepository}:${tag}"
                    }
                    
                    // Store ECR URLs for later stages
                    env.ECR_REPOSITORY_URL = ecrRepository
                }
            }
        }
        
        stage('Update Deployment') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Update Kubernetes deployment
                    sh """
                        kubectl set image deployment/retail-forecast-api \
                            api=${env.ECR_REPOSITORY_URL}:${env.IMAGE_TAG} \
                            --record
                    """
                    
                    echo "✅ Deployment updated with new image"
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    // Clean up local images
                    sh "docker rmi ${env.IMAGE_NAME} || true"
                    sh "docker system prune -f || true"
                }
            }
        }
    }
    
    post {
        always {
            // Clean up workspace
            cleanWs()
        }
        
        success {
            echo """
            🎉 Pipeline completed successfully!
            
            📍 Image URLs:
            - ${env.ECR_REPOSITORY_URL}:latest
            - ${env.ECR_REPOSITORY_URL}:${env.IMAGE_TAG}
            - ${env.ECR_REPOSITORY_URL}:${env.GIT_BRANCH}-${env.IMAGE_TAG}
            
            🚀 Next steps:
            - Verify deployment in EKS
            - Run integration tests
            - Monitor application metrics
            """
        }
        
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
```

---

## 4. Image Optimization & Best Practices

### 4.1. Multi-stage Build Optimization

**Advanced Dockerfile với optimization:**

```dockerfile
# Build stage với dependency caching
FROM python:3.9-slim as dependencies

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy requirements first for Docker layer caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --user -r requirements.txt

# Production base image
FROM python:3.9-slim as base

# Install only runtime dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Production stage
FROM base as production

ARG BUILD_DATE
ARG GIT_COMMIT  
ARG GIT_BRANCH

# Metadata labels
LABEL maintainer="MLOps Team"
LABEL build_date=$BUILD_DATE
LABEL git_commit=$GIT_COMMIT
LABEL git_branch=$GIT_BRANCH
LABEL description="Retail Forecast ML Inference API"

WORKDIR /app

# Copy dependencies from builder
COPY --from=dependencies /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Create required directories
RUN mkdir -p /app/model /app/logs && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1
ENV AWS_DEFAULT_REGION=ap-southeast-1

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Start application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

### 4.2. .dockerignore Configuration

**Tạo `server/.dockerignore`:**

```
# Git
.git
.gitignore

# Python
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.pytest_cache

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Documentation
README.md
*.md
docs/

# Test files
test_*
*_test.py
tests/

# Development files
docker-compose.yml
Dockerfile.dev

# Logs
logs/
*.log

# Temporary files
tmp/
temp/
.tmp

# Build artifacts
build/
dist/
*.egg-info/

# Model files (large files should be downloaded at runtime)
model/*.pkl
model/*.joblib
*.model
```

### 4.3. Image Size Optimization

**Strategies để giảm image size:**

1. **Use Alpine base images:**
```dockerfile
FROM python:3.9-alpine as builder
# Install build dependencies for Alpine
RUN apk add --no-cache gcc musl-dev libffi-dev
```

2. **Multi-stage builds:**
```dockerfile
# Only copy production dependencies
COPY --from=builder /root/.local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
```

3. **Remove unnecessary packages:**
```dockerfile
RUN apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 4.4. Security Best Practices

**Security hardening:**

```dockerfile
# Use specific version tags
FROM python:3.9.18-slim

# Don't run as root
USER appuser

# Use read-only filesystem
COPY --chown=appuser:appuser --chmod=755 ./scripts/ /app/scripts/

# Set security headers
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Health check with timeout
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

---

## 5. Monitoring & Troubleshooting

### 5.1. Build Monitoring

**Script để monitor build performance:**

```bash
#!/bin/bash
# monitor-build.sh

# Function to measure build time
measure_build_time() {
    local start_time=$(date +%s)
    
    # Run build
    ./scripts/build-and-push.sh "$@"
    local exit_code=$?
    
    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    
    echo "Build completed in ${duration} seconds (exit code: $exit_code)"
    
    # Log to file
    echo "$(date -u +%Y-%m-%dT%H:%M:%SZ),${duration},${exit_code}" >> build-metrics.csv
    
    return $exit_code
}

# Run build with timing
measure_build_time "$@"
```

### 5.2. Common Issues & Solutions

**Issue 1: Build fails due to dependency conflicts**
```bash
# Solution: Use pip-tools for dependency management
pip install pip-tools
pip-compile requirements.in  # Generate requirements.txt with locked versions
```

**Issue 2: Large image sizes**
```bash
# Analyze image layers
docker history retail-forecast:latest

# Use dive tool for detailed analysis
dive retail-forecast:latest
```

**Issue 3: ECR push fails with authentication errors**
```bash
# Refresh ECR login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin $ECR_REPOSITORY

# Check AWS credentials
aws sts get-caller-identity
```

**Issue 4: Container startup fails**
```bash
# Debug container startup
docker run -it --entrypoint /bin/bash retail-forecast:latest

# Check application logs
docker logs <container-id>
```

---

{{% notice success %}}
**🎯 Task 7 Complete!**

Docker Build & Push pipeline đã được triển khai thành công với:

✅ **FastAPI inference application** với health checks và monitoring  
✅ **Multi-stage Dockerfile** optimized cho production  
✅ **Automated build scripts** (Linux/macOS + Windows PowerShell)  
✅ **CI/CD integration** (GitHub Actions + Jenkins)  
✅ **Image optimization** và security best practices  
✅ **Multiple tagging strategy** (latest, git-sha, branch-sha)  
✅ **ECR integration** với automated push và scanning  

**Next Steps:**
- Task 8: S3 Data Lake setup cho model artifacts
- Task 9: SageMaker integration cho model training
- Task 10: Kubernetes deployment configurations
{{% /notice %}}

{{% notice tip %}}
**💡 Production Considerations:**

- **Security scanning**: Integrate Trivy/Snyk trong CI/CD pipeline
- **Image signing**: Use AWS Signer cho supply chain security
- **Multi-architecture builds**: Support ARM64 cho cost optimization
- **Layer caching**: Optimize Docker layer caching trong CI/CD
- **Vulnerability management**: Automated image updates cho security patches
- **Performance monitoring**: Track build times và optimize bottlenecks
{{% /notice %}}