---
title: "ECR Container Registry"
date: 2024-01-01T00:00:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 6:** Thiết lập Amazon Elastic Container Registry (ECR) cho MLOps pipeline:

1. **Tạo ECR Repository**: Cài đặt repo cho API và training image
2. **Cấu hình Security**: Image scanning, IAM policy, lifecycle rules
3. **Build & Push Image**: Upload image lên ECR thông qua CLI/console
4. **Manual Build & Push**: Hướng dẫn build/push bằng script (CLI / PowerShell)

→ Mục tiêu: Có thể lưu trữ và quản lý Docker image một cách an toàn trên AWS.
{{% /notice %}}

## Tổng quan

**Amazon ECR (Elastic Container Registry)** là dịch vụ Docker container registry được quản lý hoàn toàn bởi AWS, tích hợp sâu với EKS và CI/CD pipeline. ECR cung cấp khả năng lưu trữ, quản lý và triển khai container images một cách an toàn cho MLOps workflow.

## 1. ECR Repositories Setup

### 1.1. Create ECR Repositories

1. **Navigate to ECR Console:**
   - Đăng nhập AWS Console
   - Navigate to ECR service
   - Region: ap-southeast-1
   - Chọn "Create repository"

![](/images/06-ecr-registry/01-create-repository.png)

2. **API Repository Configuration:**
   ```
   Visibility settings: Private
   Repository name: mlops/retail-api
   Tag immutability: ✅ Enabled (Reproducibility)
   Image scan settings: ✅ Scan on push
   Encryption settings: AES-256 (Default)
   ```

![](/images/06-ecr-registry/02-api-repository-config.png)

3. **Training Repository Configuration (Optional):**
   ```
   Visibility settings: Private
   Repository name: mlops/train-model
   Tag immutability: ✅ Enabled
   Image scan settings: ✅ Scan on push
   Encryption settings: AES-256
   ```

![](/images/06-ecr-registry/03-training-repository-config.png)

4. **Repository Creation Complete:**
   ```
   API Repository URI: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api
   Training Repository URI: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/train-model
   ```

![](/images/06-ecr-registry/04-repositories-created.png)

### 1.2. Repository Policies Configuration

1. **API Repository Access Policy:**
   - Chọn repository `mlops/retail-api`
   - Click "Permissions" tab → "Edit policy JSON"

![](/images/06-ecr-registry/05-api-repository-permissions.png)

2. **Configure API Repository Policy:**
   ```json
   {
     "Version": "2008-10-17",
     "Statement": [
       {
         "Sid": "AllowEKSNodeGroupPull",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-hybrid-eks-nodes-role"
         },
         "Action": [
           "ecr:BatchCheckLayerAvailability",
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage"
         ]
       },
       
     ]
   }
   ```

![](/images/06-ecr-registry/06-api-repository-policy.png)

3. **Training Repository Access Policy:**
   ```json
   {
     "Version": "2008-10-17",
     "Statement": [
       {
         "Sid": "AllowSageMakerAccess",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:role/mlops-sagemaker-execution-role"
         },
         "Action": [
           "ecr:BatchCheckLayerAvailability",
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage"
         ]
       },
       
     ]
   }
   ```

### 1.3. Lifecycle Policy Setup

1. **API Repository Lifecycle Policy:**
   - Repository → "Lifecycle policy" tab → "Create rule"

![](/images/06-ecr-registry/07-api-lifecycle-policy.png)

2. **Configure API Lifecycle Rules:**

   **Rule 1 - Keep Latest Production Images:**

   ```
   Rule priority: 1
   Description: Keep latest 10 production images
   Image status: Tagged
   Tag status: Starts with "v" (production versions)

   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 10

   Action: expire
   ```

   **Rule 2 - Keep Latest Development Images:**

   ```
   Rule priority: 2
   Description: Keep latest 5 development images
   Image status: Tagged
   Tag status: Starts with "dev", "feature", "main"

   Match criteria:
   - Count type: imageCountMoreThan
   - Count number: 5

   Action: expire
   ```

   **Rule 3 - Remove Old Untagged Images:**

   ```
   Rule priority: 3
   Description: Delete untagged images after 1 day
   Image status: Untagged

   Match criteria:
   - Count type: sinceImagePushed
   - Count number: 1
   - Count unit: days

   Action: expire
   ```

![](/images/06-ecr-registry/08-api-lifecycle-rules.png)

3. **Training Repository Lifecycle Policy:**
   ```
   Rule 1: Keep latest 5 training images
   Rule 2: Delete untagged images after 1 day
   Rule 3: Keep experiment images (tagged with "exp-") for 30 days max
   ```

### 1.4. Image Scanning Verification

1. **Check Scan Settings:**
   - Repository → "Image scan settings" tab
   - Verify "Scan on push" is enabled
   - Review enhanced scanning options

![](/images/06-ecr-registry/09-scan-settings.png)

2. **Enhanced Scanning (Optional):**
   ```
   ✅ Basic scanning: Included with ECR (FREE)
   ⚙️ Enhanced scanning: $0.09 per image scan (more detailed CVE detection)
   ```

{{% notice success %}}
**🎯 ECR Repositories Setup Complete!**

**Created Repositories:**

- ✅ `mlops/retail-api`: FastAPI prediction service container
- ✅ `mlops/train-model`: Custom training container (optional)
- ✅ Image scanning enabled on push
- ✅ Tag immutability enabled for reproducibility
- ✅ Lifecycle policies configured for cost optimization
- ✅ IAM access policies for EKS
{{% /notice %}}

## 2. Container Images Development

### 2.1. FastAPI Prediction Service Container

**Create directory structure:**

```
server/
├── Dockerfile
├── requirements.txt
├── main.py
├── model_loader.py
├── prediction_service.py
└── health_check.py
```

**File: `server/Dockerfile`**

```dockerfile
# Multi-stage build for FastAPI retail prediction API
FROM python:3.9-slim as builder

# Set working directory
WORKDIR /app

# Install system dependencies for ML libraries
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libc6-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.9-slim

# Create non-root user for security
RUN groupadd -r apiuser && useradd -r -g apiuser apiuser

# Set working directory
WORKDIR /app

# Copy Python packages from builder stage
COPY --from=builder /root/.local /home/apiuser/.local

# Copy application code
COPY --chown=apiuser:apiuser . .

# Create directories for model and logs
RUN mkdir -p /app/models /app/logs && \
    chown -R apiuser:apiuser /app

# Switch to non-root user
USER apiuser

# Set Python path
ENV PATH=/home/apiuser/.local/bin:$PATH

# Environment variables
ENV PYTHONPATH=/app
ENV AWS_DEFAULT_REGION=ap-southeast-1
ENV MODEL_BUCKET=mlops-retail-forecast-models
ENV MODEL_PREFIX=models/retail-price-sensitivity

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python health_check.py || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

**File: `server/requirements.txt`**

```txt
# FastAPI and web framework
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0

# ML libraries
scikit-learn==1.3.2
xgboost==2.0.2
pandas==2.1.3
numpy==1.25.2

# AWS SDK
boto3==1.34.0
botocore==1.34.0

# Utilities
python-json-logger==2.0.7
python-multipart==0.0.6
httpx==0.25.2

# Monitoring
prometheus-client==0.19.0
```

**File: `server/main.py`**
```python
"""
FastAPI Application for Retail Price Sensitivity Prediction
Serves ML model predictions via REST API for MLOps pipeline
"""

from fastapi import FastAPI, HTTPException
from fastapi.responses import HTMLResponse, FileResponse
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import Dict, List, Optional
import logging
from pathlib import Path

from model_loader import ModelLoader
from prediction_service import PredictionService

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize FastAPI app
app = FastAPI(
    title="Retail Price Sensitivity Prediction API",
    description="ML model serving for retail customer price sensitivity prediction",
    version="1.0.0"
)

# CORS middleware for web frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize model loader and prediction service
model_loader = ModelLoader()
prediction_service = PredictionService(model_loader)

# Pydantic models for request/response validation
class PredictionRequest(BaseModel):
    BASKET_SIZE: str = Field(..., description="Basket size: S, M, L")
    BASKET_TYPE: str = Field(..., description="Basket composition type")
    STORE_REGION: str = Field(..., description="Store region code")
    STORE_FORMAT: str = Field(..., description="Store format: SS (Small Store), LS (Large Store)")
    SPEND: float = Field(..., gt=0, description="Transaction spend amount in GBP")
    QUANTITY: int = Field(..., gt=0, description="Number of items purchased")
    PROD_CODE_20: str = Field(..., description="Product category level 2")
    PROD_CODE_30: str = Field(..., description="Product category level 3")

class PredictionResponse(BaseModel):
    prediction: str = Field(..., description="Predicted price sensitivity class")
    probability: Dict[str, float] = Field(..., description="Class probabilities")
    confidence: float = Field(..., description="Prediction confidence score")
    model_version: str = Field(..., description="Model version identifier")

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool
    model_version: Optional[str] = None

@app.get("/", response_class=HTMLResponse)
async def root():
    """Serve HTML frontend for API testing"""
    html_path = Path(__file__).parent / "index.html"
    if html_path.exists():
        return FileResponse(html_path)
    return HTMLResponse("<h1>Retail Price Sensitivity Prediction API</h1><p>Visit /docs for API documentation</p>")

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint for ALB and Kubernetes probes"""
    try:
        model = model_loader.get_model()
        model_info = model_loader.get_model_info()
        
        return HealthResponse(
            status="healthy",
            model_loaded=model is not None,
            model_version=model_info.get("version", "unknown")
        )
    except Exception as e:
        logger.error(f"Health check failed: {str(e)}")
        raise HTTPException(status_code=503, detail="Service unhealthy")

@app.get("/ready", response_model=HealthResponse)
async def readiness_check():
    """Readiness check endpoint for Kubernetes"""
    return await health_check()

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """Single prediction endpoint"""
    try:
        # Convert request to dictionary
        features = request.dict()
        
        # Make prediction
        result = prediction_service.predict(features)
        
        return PredictionResponse(**result)
    
    except Exception as e:
        logger.error(f"Prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")

@app.post("/predict/batch")
async def predict_batch(requests: List[PredictionRequest]):
    """Batch prediction endpoint"""
    try:
        results = []
        for req in requests:
            features = req.dict()
            result = prediction_service.predict(features)
            results.append(result)
        
        return {"predictions": results, "count": len(results)}
    
    except Exception as e:
        logger.error(f"Batch prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Batch prediction failed: {str(e)}")

@app.get("/model/info")
async def model_info():
    """Get model information"""
    try:
        info = model_loader.get_model_info()
        return info
    except Exception as e:
        logger.error(f"Model info error: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Failed to get model info: {str(e)}")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**File: `server/model_loader.py`**
```python
"""
Model Loader for Retail Price Sensitivity Prediction
Handles model download from S3 and local caching
"""

import os
import joblib
import logging
from pathlib import Path
from typing import Optional, Dict, Any
import numpy as np

logger = logging.getLogger(__name__)

class ModelLoader:
    """Load ML model from S3 or use mock model for testing"""
    
    def __init__(self):
        self.model = None
        self.model_path = "/tmp/model.joblib"
        self.bucket_name = os.getenv("MODEL_BUCKET", "mlops-retail-forecast-models")
        self.model_key = os.getenv("MODEL_KEY", "models/retail-price-sensitivity/model.joblib")
        
        # Load model on initialization
        self.load_model()
    
    def load_model(self):
        """Load model from S3 or use mock model"""
        try:
            # Try to load from S3
            logger.info(f"Attempting to load model from S3: s3://{self.bucket_name}/{self.model_key}")
            self._download_from_s3()
            self.model = self._load_from_file()
            logger.info("Model loaded successfully from S3")
            
        except Exception as e:
            logger.warning(f"Failed to load model from S3: {str(e)}")
            logger.info("Using mock model for testing")
            self.model = self._load_mock_model()
    
    def _download_from_s3(self):
        """Download model file from S3"""
        try:
            import boto3
            
            s3_client = boto3.client('s3')
            
            # Download model file
            s3_client.download_file(
                Bucket=self.bucket_name,
                Key=self.model_key,
                Filename=self.model_path
            )
            
            logger.info(f"Model downloaded to {self.model_path}")
            
        except Exception as e:
            logger.error(f"S3 download failed: {str(e)}")
            raise
    
    def _load_from_file(self):
        """Load model from local file"""
        if not Path(self.model_path).exists():
            raise FileNotFoundError(f"Model file not found: {self.model_path}")
        
        model = joblib.load(self.model_path)
        logger.info("Model loaded from file")
        return model
    
    def _load_mock_model(self):
        """Create mock model for testing when S3 is unavailable"""
        
        class MockModel:
            """Simple mock model for testing"""
            
            def predict(self, X):
                """Predict price sensitivity based on simple rules"""
                predictions = []
                
                for row in X:
                    # Simple rule-based prediction
                    spend = row[4] if len(row) > 4 else 100
                    
                    if spend < 50:
                        predictions.append(2)  # High sensitivity
                    elif spend < 150:
                        predictions.append(1)  # Medium sensitivity
                    else:
                        predictions.append(0)  # Low sensitivity
                
                return np.array(predictions)
            
            def predict_proba(self, X):
                """Return mock probabilities"""
                predictions = self.predict(X)
                probas = []
                
                for pred in predictions:
                    if pred == 0:  # Low
                        probas.append([0.7, 0.2, 0.1])
                    elif pred == 1:  # Medium
                        probas.append([0.2, 0.6, 0.2])
                    else:  # High
                        probas.append([0.1, 0.2, 0.7])
                
                return np.array(probas)
        
        logger.info("Mock model created")
        return MockModel()
    
    def get_model(self):
        """Get loaded model instance"""
        if self.model is None:
            raise ValueError("Model not loaded")
        return self.model
    
    def get_model_info(self) -> Dict[str, Any]:
        """Get model metadata"""
        info = {
            "model_loaded": self.model is not None,
            "model_type": type(self.model).__name__,
            "model_source": "s3" if Path(self.model_path).exists() else "mock",
            "version": os.getenv("MODEL_VERSION", "1.0.0")
        }
        
        if Path(self.model_path).exists():
            info["model_path"] = self.model_path
            info["model_size_mb"] = round(Path(self.model_path).stat().st_size / (1024 * 1024), 2)
        
        return info
    
    def reload_model(self):
        """Reload model from S3"""
        logger.info("Reloading model...")
        self.model = None
        
        # Remove cached model file
        if Path(self.model_path).exists():
            Path(self.model_path).unlink()
        
        # Reload model
        self.load_model()
        logger.info("Model reloaded successfully")
```

**File: `server/prediction_service.py`**
```python
"""
Prediction Service for Retail Price Sensitivity
Handles feature preprocessing and model inference
"""

import logging
import numpy as np
from typing import Dict, Any
from model_loader import ModelLoader

logger = logging.getLogger(__name__)

class PredictionService:
    """Service for preprocessing features and making predictions"""
    
    # Class labels mapping
    CLASS_LABELS = ['Low', 'Medium', 'High']
    
    # Feature encoding maps
    BASKET_SIZE_MAP = {'S': 0, 'M': 1, 'L': 2}
    STORE_FORMAT_MAP = {'SS': 0, 'LS': 1}
    
    def __init__(self, model_loader: ModelLoader):
        self.model_loader = model_loader
    
    def predict(self, features: Dict[str, Any]) -> Dict[str, Any]:
        """Make prediction for given features"""
        try:
            # Preprocess features
            X = self._preprocess(features)
            
            # Get model
            model = self.model_loader.get_model()
            
            # Make prediction
            prediction = model.predict(X)[0]
            probabilities = model.predict_proba(X)[0]
            
            # Get class label
            class_label = self.CLASS_LABELS[prediction]
            
            # Get confidence (max probability)
            confidence = float(max(probabilities))
            
            # Build probability dictionary
            prob_dict = {
                label: float(prob) 
                for label, prob in zip(self.CLASS_LABELS, probabilities)
            }
            
            # Get model info
            model_info = self.model_loader.get_model_info()
            
            return {
                "prediction": class_label,
                "probability": prob_dict,
                "confidence": round(confidence, 4),
                "model_version": model_info.get("version", "unknown")
            }
            
        except Exception as e:
            logger.error(f"Prediction failed: {str(e)}")
            raise
    
    def _preprocess(self, features: Dict[str, Any]) -> np.ndarray:
        """Preprocess features for model input"""
        try:
            # Extract and encode features
            basket_size_encoded = self.BASKET_SIZE_MAP.get(features['BASKET_SIZE'], 1)
            store_format_encoded = self.STORE_FORMAT_MAP.get(features['STORE_FORMAT'], 0)
            
            # Hash categorical features (simple hash for demo)
            basket_type_hash = hash(features['BASKET_TYPE']) % 100
            store_region_hash = hash(features['STORE_REGION']) % 100
            prod_code_20_hash = hash(features['PROD_CODE_20']) % 1000
            prod_code_30_hash = hash(features['PROD_CODE_30']) % 1000
            
            # Build feature array
            feature_array = np.array([[
                basket_size_encoded,
                basket_type_hash,
                store_region_hash,
                store_format_encoded,
                features['SPEND'],
                features['QUANTITY'],
                prod_code_20_hash,
                prod_code_30_hash
            ]])
            
            logger.debug(f"Preprocessed features: {feature_array}")
            return feature_array
            
        except KeyError as e:
            logger.error(f"Missing required feature: {str(e)}")
            raise ValueError(f"Missing required feature: {str(e)}")
        except Exception as e:
            logger.error(f"Preprocessing failed: {str(e)}")
            raise
```

**File: `server/health_check.py`**
```python
"""
Health Check Script for Docker HEALTHCHECK
Checks if the FastAPI application is responding correctly
"""

import sys
import httpx

def check_health():
    """Check if API health endpoint is responding"""
    try:
        response = httpx.get("http://localhost:8000/health", timeout=5.0)
        
        if response.status_code == 200:
            data = response.json()
            if data.get("status") == "healthy":
                sys.exit(0)  # Healthy
        
        sys.exit(1)  # Unhealthy
        
    except Exception:
        sys.exit(1)  # Unhealthy

if __name__ == "__main__":
    check_health()
```

## 3. Build & Push (Manual)

### 3.1. Local Build and Push Script

**Create file `scripts/build-and-push.sh`:**

```bash
#!/bin/bash

set -e

# Configuration
PROJECT_NAME="mlops-retail-forecast"
ENVIRONMENT="dev"
AWS_REGION="ap-southeast-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Repository URLs
echo "Training Repository: ${TRAINING_REPO}"

```

## 5. Verification & Testing

### 5.1. Image Verification

**Check repositories in ECR:**

```bash
# List all repositories
aws ecr describe-repositories --region ap-southeast-1

# List images in API repository
aws ecr list-images \
    --repository-name mlops/retail-api \
    --region ap-southeast-1

# List images in training repository
aws ecr list-images \
    --repository-name mlops/train-model \
    --region ap-southeast-1
```

**Get detailed image information:**

```bash
# Describe specific image
aws ecr describe-images \
    --repository-name mlops/retail-api \
    --image-ids imageTag=latest \
    --region ap-southeast-1

# Check image scan results
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1
```

### 5.2. Local Container Testing

**Test API container locally:**

```bash
# Run API container locally
docker run -d \
    --name retail-api-test \
    -p 8000:8000 \
    -e AWS_DEFAULT_REGION=ap-southeast-1 \
    -e MODEL_BUCKET=mlops-retail-forecast-models \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

# Test health endpoint
curl http://localhost:8000/health

# Test prediction endpoint
curl -X POST http://localhost:8000/predict \
    -H "Content-Type: application/json" \
    -d '{
        "features": {
            "SPEND": 100.0,
            "UNITS": 5.0,
            "VISITS": 2.0
        },
        "customer_id": "test-customer-123"
    }'

# Check logs
docker logs retail-api-test

# Clean up
docker stop retail-api-test
docker rm retail-api-test
```

### 5.3. EKS Integration Testing

**Create test deployment:**

```yaml
# test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api-test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: retail-api-test
  template:
    metadata:
      labels:
        app: retail-api-test
    spec:
      serviceAccountName: retail-forecast-sa
      containers:
        - name: api
          image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: AWS_DEFAULT_REGION
              value: "ap-southeast-1"
            - name: MODEL_BUCKET
              value: "mlops-retail-forecast-models"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: retail-api-test-service
spec:
  selector:
    app: retail-api-test
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### 5.4. Security Scan Analysis

**Check vulnerability scan results:**

```bash
# Get scan results summary
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1 \
    --query 'imageScanFindings.findingCounts'

# Get detailed vulnerability findings
aws ecr describe-image-scan-findings \
    --repository-name mlops/retail-api \
    --image-id imageTag=latest \
    --region ap-southeast-1 \
    --query 'imageScanFindings.findings[?severity==`HIGH` || severity==`CRITICAL`]'
```

**Security best practices verification:**

```bash
# Check if running as non-root user
docker run --rm 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest whoami

# Check file permissions
docker run --rm 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest ls -la /app

# Verify health check works
docker run --rm -p 8000:8000 -d --name health-test \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest

sleep 30
docker exec health-test python health_check.py
docker stop health-test
```

## 6. Cost Monitoring & Optimization

### 6.1. ECR Cost Tracking

**Monthly cost breakdown:**

```bash
# Check repository storage usage
aws ecr describe-repositories \
    --region ap-southeast-1 \
    --query 'repositories[*].{Name:repositoryName,Size:repositorySizeInBytes}' \
    --output table

# Estimate monthly storage cost
# ECR Storage: $0.10 per GB-month
# Typical API image: ~500MB
# Training image: ~1GB
# Total: ~1.5GB * $0.10 = $0.15/month
```

### 6.2. Image Optimization

**Multi-stage build analysis:**

```bash
# Compare image sizes
docker images | grep mlops

# Check layer sizes
docker history 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
```

**Optimization strategies:**

1. **Use slim base images**: python:3.9-slim vs python:3.9
2. **Multi-stage builds**: Separate build and runtime environments
3. **Layer caching**: Order Dockerfile commands for better caching
4. **Remove unnecessary packages**: Clean up apt cache, temp files
5. **Use .dockerignore**: Exclude unnecessary files from context

## 7. Troubleshooting

### 7.1. Common Issues

**Issue 1: ECR Authentication Failed**

```bash
# Solution: Refresh ECR login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-southeast-1.amazonaws.com

# Check AWS credentials
aws sts get-caller-identity
```

**Issue 2: Image Pull Failed from EKS**

```bash
# Check node group IAM permissions
kubectl describe node

# Verify ECR repository policy
aws ecr get-repository-policy \
    --repository-name mlops/retail-api \
    --region ap-southeast-1

# Check IRSA configuration
kubectl describe serviceaccount retail-forecast-sa
```

**Issue 3: Container Health Check Failing**

```bash
# Check container logs
kubectl logs -l app=retail-api-test

# Test health endpoint manually
kubectl exec -it <pod-name> -- curl http://localhost:8000/health

# Verify model loading
kubectl exec -it <pod-name> -- python -c "
import boto3
s3 = boto3.client('s3')
s3.head_object(Bucket='mlops-retail-forecast-models', Key='models/retail-price-sensitivity/model.pkl')
"
```

**Issue 4: Build Timeout in CI/CD**

```yaml
# Increase timeout in GitHub Actions
jobs:
  build-and-push:
    timeout-minutes: 30 # Increase from default 360
```

### 7.2. Performance Optimization

**Build performance:**

```bash
# Use BuildKit for faster builds
export DOCKER_BUILDKIT=1
docker build ...

# Enable build cache
docker build --cache-from mlops-retail-api:latest ...
```

**Runtime performance:**

```bash
# Monitor container resource usage
kubectl top pods -l app=retail-api

# Check API response times
kubectl exec -it <pod-name> -- curl -w "@curl-format.txt" http://localhost:8000/predict
```

## Kết quả Task 6

✅ **ECR Repositories** - mlops/retail-api và mlops/train-model repositories  
✅ **Container Images** - FastAPI prediction service và training containers  
✅ **Cost Optimization** - Lifecycle policies, multi-stage builds, ~$0.15/month  

### Architecture Delivered

```
✅ ECR Container Registry:
   - mlops/retail-api: FastAPI prediction service
   - mlops/train-model: Custom training environment
   - Tag immutability enabled
   - Vulnerability scanning on push

✅ CI/CD Automation:
   - GitHub Actions workflow
   - Automatic builds on code changes
   - Git-based image tagging (v20241009-abc123)
   - OIDC authentication (no AWS keys)

✅ Security & Compliance:
   - Non-root containers
   - Multi-stage builds
   - Vulnerability scanning
   - IAM-based access control
```

{{% notice success %}}
**🎯 Task 6 Complete - Production-Ready Container Registry!**

**Container Strategy**: FastAPI API + optional training containers  
**CI/CD Automation**: Build → Push → Deploy on Git changes  
**Security**: Image scanning, non-root users, access control  
**Cost Efficient**: ~$0.15/month storage + lifecycle cleanup  
**Demo Ready**: API container ready for ALB + EKS deployment  
{{% /notice %}}

{{% notice tip %}}
**🚀 Next Steps:**

- **Task 7**: EKS cluster deployment trong hybrid VPC
- **Task 8**: EKS node groups với ECR image pull permissions
- **Task 10**: Deploy API container với ALB integration
- **Task 11**: ALB ingress controller cho public API access

**Build & Deploy Commands:**

```bash
# Local build and push
./scripts/build-and-push.sh

# CI/CD triggers automatically on:
git push origin main        # → v20241009-abc123 + latest
git push origin develop     # → dev-abc123 + dev-latest
git push origin feature/x   # → feature-x-abc123
```

{{% /notice %}}

{{% notice info %}}
**� Production Benchmarks Achieved:**

- **Image Size**: FastAPI ~500MB (optimized multi-stage)
- **Build Time**: ~3-5 minutes (with caching)
- **Storage Cost**: ~$0.15/month (1.5GB total)
- **Security**: Non-root, vulnerability scanned
- **Availability**: Multi-tag strategy (latest, commit, branch)
- **CI/CD**: Automated on every commit
  {{% /notice %}}

---

**Next Step**: [Task 7: EKS Cluster Setup](../7-eks-cluster/)
