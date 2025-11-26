---
title: "API Deployment on EKS"
date: 2024-01-01T00:00:00Z
weight: 9
chapter: false
pre: "<b>9. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 9:**

Triển khai Retail Prediction API (FastAPI) lên EKS Cluster, kết nối model từ S3 và expose endpoint public qua Load Balancer (ALB).  
→ Đảm bảo dịch vụ chạy ổn định, tự động scale, bảo mật, và có thể demo API thật.
{{% /notice %}}

📥 **Input từ các Task trước:**
- **Task 5 (Production VPC):** VPC design, subnets, VPC Endpoints and ALB networking required for cluster and load balancer
- **Task 6 (ECR Container Registry):** Container images and repository URIs to deploy
- **Task 8 (API Containerization):** Docker image layout, Dockerfile and runtime environment variables
- **Task 2 (IAM Roles & Audit):** IRSA roles and policies for Pods to access S3 and other AWS services
- **Task 7 (EKS Cluster):** EKS cluster and node groups where manifests will be applied

## 1. Tổng quan

**API Deployment** là bước triển khai service dự đoán đã được container hóa lên Kubernetes (EKS). Bước này đảm bảo ứng dụng được triển khai theo kiến trúc microservice, tự động scale và có tính sẵn sàng cao.

### Kiến trúc triển khai

**EKS Deployment Architecture:**

```
Client → ALB → EKS Service → API Pods → S3 Models
                    ↓
            Auto-scaling (HPA)
```

**Components:**
- **Namespace**: `retail-prediction` 
- **Deployment**: API pods với IRSA
- **Service**: Internal load balancing
- **HPA**: Auto-scaling dựa trên CPU
- **ConfigMap**: Environment variables

## 2. Kubernetes Manifests

**Cần tạo 4 file chính:**
- `namespace.yaml` - Tạo namespace
- `deployment.yaml` - API application 
- `service.yaml` - Internal load balancer
- `hpa.yaml` - Auto-scaling

### 2.1 Namespace Configuration

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: retail-prediction
  labels:
    name: retail-prediction
    environment: production
---
```

### 2.2 ConfigMap cho Environment Variables

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: retail-api-config
  namespace: retail-prediction
data:
  # S3 Configuration
  MODEL_BUCKET: "mlops-retail-forecast-models"
  MODEL_KEY: "models/retail-price-sensitivity/model.joblib"
  AWS_REGION: "ap-southeast-1"
  
  # API Configuration
  PORT: "8000"
  HOST: "0.0.0.0"
  WORKERS: "1"
  
  # Logging Configuration
  LOG_LEVEL: "INFO"
  LOG_FORMAT: "json"
---
```

### 2.3 ServiceAccount với IRSA (IAM Role for Service Account)

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access
  namespace: retail-prediction
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/RetailAPIModelAccess
---
```

{{% notice info %}}
**IRSA đã được cấu hình trong Task 7** - Sử dụng service account `s3-access-sa` đã tạo.
{{% /notice %}}

## 3. Basic Deployment

### 3.1 Simple API Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: retail-prediction
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-api
  template:
    metadata:
      labels:
        app: retail-api
    spec:
      serviceAccountName: s3-access-sa  # From Task 7
      containers:
      - name: retail-api
        image: <ACCOUNT-ID>.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
        ports:
        - containerPort: 8000
        env:
        - name: MODEL_BUCKET
          value: "mlops-retail-forecast-models"
        - name: AWS_REGION
          value: "ap-southeast-1"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
        
        # Environment Variables
        env:
        - name: MODEL_BUCKET
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: MODEL_BUCKET
        - name: MODEL_KEY
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: MODEL_KEY
        - name: AWS_DEFAULT_REGION
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: AWS_REGION
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: PORT
        - name: WORKERS
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: WORKERS
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: retail-api-config
              key: LOG_LEVEL
        
        # Health Checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
```

## 4. Service (Load Balancer)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: retail-prediction
spec:
  type: LoadBalancer  # Creates AWS NLB automatically
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: retail-api
```

## 5. Auto-scaling (HPA)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-api-hpa
  namespace: retail-prediction
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-api
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
---
```

{{% notice tip %}}
HPA (Horizontal Pod Autoscaler) giúp tự động scale số lượng Pod dựa trên mức sử dụng CPU. Khi CPU usage vượt quá 70%, HPA sẽ tự động tăng số lượng Pod (tối đa 10) để đảm bảo hiệu năng ổn định.
{{% /notice %}}



## 6. Deploy to EKS

### 6.1 Apply Manifests

```bash
# Create namespace
kubectl create namespace retail-prediction

# Apply all manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl apply -f aws/k8s/hpa.yaml
```

### 7.2 Kiểm tra Trạng thái Deployment

```bash
# Kiểm tra trạng thái pods
kubectl get pods -n retail-prediction

# Kiểm tra service và load balancer
kubectl get svc -n retail-prediction

# Kiểm tra horizontal pod autoscaler
kubectl get hpa -n retail-prediction

# Kiểm tra logs của pod
kubectl logs -f deployment/retail-api -n retail-prediction
```

### 7.3 Kiểm tra API Endpoint

```bash
# Lấy URL của LoadBalancer
export API_URL=$(kubectl get svc retail-api-service -n retail-prediction -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test health check endpoint
curl http://$API_URL/health

# Test API documentation
curl http://$API_URL/docs

# Test prediction endpoint
curl -X POST http://$API_URL/predict \
  -H "Content-Type: application/json" \
  -d '{
    "basket_items": {
      "P1001": {"product_id": "P1001", "quantity": 2, "price": 10.99, "category": "grocery"},
      "P2002": {"product_id": "P2002", "quantity": 1, "price": 25.50, "category": "electronics"}
    },
    "customer_id": "CUST123"
  }'
```

## 8. Testing và Load Testing

### 8.1 Local Testing với Port Forward

Để test API locally mà không cần LoadBalancer:

```bash
# Port forward service đến localhost
kubectl port-forward service/retail-api-service 8080:80 -n retail-prediction
```

### 8.2 Test API Endpoints

```bash
# Test health endpoint
curl http://localhost:8080/health

# Test API documentation
curl http://localhost:8080/docs

# Test prediction endpoint
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{
    "basket_items": {
      "P1001": {"product_id": "P1001", "quantity": 2, "price": 10.99, "category": "grocery"},
      "P2002": {"product_id": "P2002", "quantity": 1, "price": 25.50, "category": "electronics"}
    },
    "customer_id": "CUST123"
  }'
```

### 8.3 Load Testing

Để kích hoạt autoscaling và kiểm tra khả năng scale của hệ thống:

```bash
# Cài đặt hey (tool để load testing)
# Linux/MacOS: wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
# Windows: Sử dụng WSL hoặc download từ https://hey-release.s3.us-east-2.amazonaws.com/hey_windows_amd64.exe

# Thực hiện load test trên API endpoint
hey -n 1000 -c 50 -m POST -H "Content-Type: application/json" \
  -d '{"basket_items":{"P1001":{"product_id":"P1001","quantity":2,"price":10.99,"category":"grocery"}}}' \
  http://$API_URL/predict

# Theo dõi HPA trong quá trình load test
kubectl get hpa retail-api-hpa -n retail-prediction -w

# Theo dõi pods được tạo mới
kubectl get pods -n retail-prediction -w
```

## 7. Summary

{{% notice success %}}
**🎯 Task 9 Complete - API Deployment on EKS**

✅ **Kubernetes manifests** ready 
✅ **EKS deployment** configured với IRSA
✅ **Load Balancer service** cho external access
✅ **Auto-scaling** với HPA

**API đã sẵn sàng để access qua Load Balancer!**
{{% /notice %}}

---

**Next Step**: [Task 10: Load Balancing](../10-elastic-load-balancing/)

## 11. Chi phí ước tính

| Thành phần | Ước tính | Ghi chú |
|------------|----------|---------|
| EKS Pod (2 replica Spot node) | ~0.012 USD/h | Chi phí compute |
| ALB/NLB (public) | ~0.02 USD/h | Chỉ bật khi demo |
| **Tổng (1h demo)** | **≈ 0.03–0.04 USD** | Cực thấp nếu tắt ngay sau demo |

{{% notice info %}}
Chi phí tính toán dựa trên Spot instances t3.medium và NLB tại region ap-southeast-1. Chi phí thực tế có thể thay đổi tùy theo cấu hình và thời gian sử dụng.
{{% /notice %}}

## 12. Kết quả kỳ vọng

### ✅ Checklist hoàn thành

- [ ] **Namespace**: Namespace `retail-prediction` được tạo thành công
- [ ] **ConfigMap**: Environment variables được cấu hình
- [ ] **ServiceAccount**: IRSA được thiết lập cho S3 access
- [ ] **Deployment**: Pod ở trạng thái Running
- [ ] **Service**: LoadBalancer hoạt động với external IP/hostname
- [ ] **HPA**: Horizontal Pod Autoscaler được cấu hình
- [ ] **Health Checks**: `/health` endpoint trả về 200 OK
- [ ] **Load Testing**: API có khả năng scale khi tải tăng
- [ ] **Model Access**: Container có thể tải model từ S3
- [ ] **Prediction API**: Endpoint `/predict` có thể xử lý requests

### 📊 Kiểm tra xác nhận

1. **Pod ở trạng thái Running trong namespace Kubernetes**
   ```bash
   kubectl get pods -n retail-prediction
   # Expected: All pods in Running state with STATUS = Running
   ```

2. **Service hoạt động, có thể gọi endpoint /health trả về 200 OK**
   ```bash
   export API_URL=$(kubectl get svc retail-api-service -n retail-prediction -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
   curl http://$API_URL/health
   # Expected: {"status": "healthy"}
   ```

3. **HPA hiển thị target CPU và có thể scale số pod**
   ```bash
   kubectl get hpa retail-api-hpa -n retail-prediction
   # Expected: Shows current CPU percentage and target of 70%
   ```

4. **Load balancing hoạt động**
   ```bash
   kubectl get endpoints retail-api-service -n retail-prediction
   # Expected: Multiple IP addresses listed
   ```

### 🔍 Monitoring & Maintenance

```bash
# Theo dõi trạng thái pods
kubectl get pods -n retail-prediction -w

# Monitoring HPA
kubectl get hpa retail-api-hpa -n retail-prediction -w

# Kiểm tra resource usage
kubectl top pods -n retail-prediction

# Kiểm tra logs
kubectl logs -l app=retail-api -n retail-prediction --tail=100

# Kiểm tra events
kubectl get events -n retail-prediction --sort-by='.lastTimestamp'
```

## 13. Tổng kết

Trong task này, chúng ta đã triển khai thành công API dự đoán đã được containerize lên EKS cluster. Với cấu hình này, API có thể:

✅ **Truy cập an toàn đến model trong S3** sử dụng IRSA

✅ **Tự động scale** dựa trên CPU utilization

✅ **Public endpoint** qua AWS Load Balancer

✅ **Tối ưu chi phí** với Spot instances

Kiến trúc này đảm bảo high availability, scalability và cost-effectiveness cho ML serving layer trong MLOps pipeline.

---

**Next Step**: [Task 11: Elastic Load Balancing](../11-elastic-load-balancing/)