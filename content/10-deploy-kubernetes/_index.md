---
title: "Kubernetes Deployment"
date: 2024-01-01T00:00:00Z
weight: 10
chapter: false
pre: "<b>10. </b>"
---

## Mục tiêu

Triển khai mô hình đã huấn luyện thành API inference service chạy trên EKS, kèm theo khả năng autoscaling dựa trên tài nguyên (CPU).

## Nội dung chính

### 1. Chuẩn bị Kubernetes Manifests

#### 1.1 Namespace Configuration

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: retail-forecast
  labels:
    name: retail-forecast
    environment: production
---
```

#### 1.2 ConfigMap cho Environment Variables

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: retail-forecast-config
  namespace: retail-forecast
data:
  # S3 Configuration
  S3_MODEL_BUCKET: "retail-forecast-artifacts-bucket"
  S3_MODEL_KEY: "models/retail-forecast-model/model.tar.gz"
  S3_REGION: "us-east-1"
  
  # Model Configuration
  MODEL_NAME: "retail-forecast-model"
  MODEL_VERSION: "1.0"
  
  # API Configuration
  API_PORT: "8080"
  API_HOST: "0.0.0.0"
  WORKERS: "4"
  
  # Logging Configuration
  LOG_LEVEL: "INFO"
  LOG_FORMAT: "json"
---
```

#### 1.3 Secret cho AWS Credentials

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: retail-forecast
type: Opaque
data:
  # Base64 encoded AWS credentials (use AWS IAM roles for EKS in production)
  AWS_ACCESS_KEY_ID: ""  # Empty - using IRSA
  AWS_SECRET_ACCESS_KEY: ""  # Empty - using IRSA
---
```

#### 1.4 ServiceAccount với IRSA

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: retail-forecast-sa
  namespace: retail-forecast
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_ACCOUNT_ID:role/RetailForecastEKSRole
---
```

### 2. Deployment Configuration

#### 2.1 Main Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-forecast-api
  namespace: retail-forecast
  labels:
    app: retail-forecast-api
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: retail-forecast-api
  template:
    metadata:
      labels:
        app: retail-forecast-api
        version: v1
    spec:
      serviceAccountName: retail-forecast-sa
      containers:
      - name: retail-forecast-api
        image: YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-forecast:latest
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        
        # Environment Variables
        env:
        - name: S3_MODEL_BUCKET
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: S3_MODEL_BUCKET
        - name: S3_MODEL_KEY
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: S3_MODEL_KEY
        - name: S3_REGION
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: S3_REGION
        - name: MODEL_NAME
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: MODEL_NAME
        - name: API_PORT
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: API_PORT
        - name: WORKERS
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: WORKERS
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: retail-forecast-config
              key: LOG_LEVEL
        
        # Health Checks
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Resource Limits
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        # Security Context
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        
        # Volume Mounts
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: model-cache
          mountPath: /app/models
      
      # Volumes
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: model-cache
        emptyDir:
          sizeLimit: 2Gi
      
      # Pod Security
      securityContext:
        fsGroup: 1000
      
      # Node Selection
      nodeSelector:
        kubernetes.io/arch: amd64
      
      # Tolerations for spot instances
      tolerations:
      - key: "node.kubernetes.io/spot"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
---
```

### 3. Service Configuration

#### 3.1 ClusterIP Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-service
  namespace: retail-forecast
  labels:
    app: retail-forecast-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: retail-forecast-api
---
```

#### 3.2 LoadBalancer Service (for external access)

```yaml
# service-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-service-lb
  namespace: retail-forecast
  labels:
    app: retail-forecast-api
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: retail-forecast-api
---
```

### 4. Horizontal Pod Autoscaler (HPA)

#### 4.1 HPA Configuration

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-forecast-hpa
  namespace: retail-forecast
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-forecast-api
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  
  # Custom metrics (requests per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
---
```

### 5. Ingress Configuration

#### 5.1 Application Load Balancer Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-forecast-ingress
  namespace: retail-forecast
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: retail-forecast-alb
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: retail-forecast-api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
---
```

### 6. Monitoring và Observability

#### 6.1 ServiceMonitor for Prometheus

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: retail-forecast-monitor
  namespace: retail-forecast
  labels:
    app: retail-forecast-api
spec:
  selector:
    matchLabels:
      app: retail-forecast-api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
---
```

#### 6.2 PodDisruptionBudget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: retail-forecast-pdb
  namespace: retail-forecast
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: retail-forecast-api
---
```

### 7. Deployment Commands

#### 7.1 Deploy All Resources

```bash
# Create namespace
kubectl apply -f namespace.yaml

# Apply ConfigMaps and Secrets
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f serviceaccount.yaml

# Deploy the application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml

# Optional: Deploy ingress
kubectl apply -f ingress.yaml

# Optional: Deploy monitoring
kubectl apply -f servicemonitor.yaml
kubectl apply -f pdb.yaml
```

#### 7.2 Verify Deployment

```bash
# Check namespace
kubectl get namespace retail-forecast

# Check all resources
kubectl get all -n retail-forecast

# Check pod status
kubectl get pods -n retail-forecast -w

# Check pod logs
kubectl logs -f deployment/retail-forecast-api -n retail-forecast

# Check service endpoints
kubectl get endpoints -n retail-forecast

# Check HPA status
kubectl get hpa -n retail-forecast
```

### 8. Health Check và Testing

#### 8.1 Port Forward for Testing

```bash
# Port forward to test locally
kubectl port-forward service/retail-forecast-service 8080:80 -n retail-forecast
```

#### 8.2 Health Check Tests

```bash
# Test health endpoint
curl http://localhost:8080/healthz

# Test readiness endpoint
curl http://localhost:8080/ready

# Test prediction endpoint
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{
    "features": {
      "store_id": 1,
      "product_id": 123,
      "date": "2024-01-01",
      "price": 29.99,
      "promotion": 0
    }
  }'
```

#### 8.3 Load Testing

```bash
# Install siege for load testing
# Run load test to trigger autoscaling
siege -c 10 -t 2m http://localhost:8080/healthz

# Watch HPA during load test
kubectl get hpa retail-forecast-hpa -n retail-forecast -w
```

### 9. Scaling và Performance

#### 9.1 Manual Scaling

```bash
# Scale manually
kubectl scale deployment retail-forecast-api --replicas=5 -n retail-forecast

# Check scaling status
kubectl get deployment retail-forecast-api -n retail-forecast
```

#### 9.2 Monitor Resource Usage

```bash
# Check resource usage
kubectl top pods -n retail-forecast

# Check node resource usage
kubectl top nodes

# Describe HPA for detailed metrics
kubectl describe hpa retail-forecast-hpa -n retail-forecast
```

### 10. Troubleshooting

#### 10.1 Common Issues

```bash
# Check pod events
kubectl describe pod <pod-name> -n retail-forecast

# Check deployment events
kubectl describe deployment retail-forecast-api -n retail-forecast

# Check service endpoints
kubectl describe service retail-forecast-service -n retail-forecast

# Check HPA events
kubectl describe hpa retail-forecast-hpa -n retail-forecast
```

#### 10.2 Debug Pod Issues

```bash
# Get shell access to pod
kubectl exec -it <pod-name> -n retail-forecast -- /bin/bash

# Check logs from previous crashed container
kubectl logs <pod-name> -n retail-forecast --previous

# Check all containers in pod
kubectl logs <pod-name> -n retail-forecast --all-containers=true
```

### 11. Updates và Rollbacks

#### 11.1 Rolling Update

```bash
# Update image
kubectl set image deployment/retail-forecast-api \
  retail-forecast-api=YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-forecast:v2 \
  -n retail-forecast

# Check rollout status
kubectl rollout status deployment/retail-forecast-api -n retail-forecast
```

#### 11.2 Rollback Deployment

```bash
# Check rollout history
kubectl rollout history deployment/retail-forecast-api -n retail-forecast

# Rollback to previous version
kubectl rollout undo deployment/retail-forecast-api -n retail-forecast

# Rollback to specific revision
kubectl rollout undo deployment/retail-forecast-api --to-revision=2 -n retail-forecast
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Namespace**: Namespace `retail-forecast` được tạo thành công
- [ ] **ConfigMap/Secret**: Environment variables được cấu hình
- [ ] **ServiceAccount**: IRSA được thiết lập cho AWS access
- [ ] **Deployment**: Pod ở trạng thái Running
- [ ] **Service**: Service hoạt động và có endpoints
- [ ] **HPA**: Horizontal Pod Autoscaler được cấu hình
- [ ] **Health Checks**: `/healthz` endpoint trả về 200 OK
- [ ] **Readiness**: `/ready` endpoint trả về 200 OK
- [ ] **Load Balancing**: Traffic được phân phối đều giữa pods
- [ ] **Auto Scaling**: HPA tự động scale dựa trên CPU usage

### 📊 Verification Steps

1. **Pod ở trạng thái Running trong namespace Kubernetes**
   ```bash
   kubectl get pods -n retail-forecast
   # Expected: All pods in Running state
   ```

2. **Service hoạt động, có thể gọi endpoint /healthz trả về 200 OK**
   ```bash
   kubectl port-forward service/retail-forecast-service 8080:80 -n retail-forecast &
   curl http://localhost:8080/healthz
   # Expected: {"status": "healthy"}
   ```

3. **HPA hiển thị target CPU và có thể scale số pod**
   ```bash
   kubectl get hpa -n retail-forecast
   # Expected: Shows current CPU percentage and target
   ```

4. **Load balancing hoạt động**
   ```bash
   kubectl get endpoints retail-forecast-service -n retail-forecast
   # Expected: Multiple IP addresses listed
   ```

### 🔍 Monitoring Commands

```bash
# Watch pod status
kubectl get pods -n retail-forecast -w

# Monitor HPA
kubectl get hpa retail-forecast-hpa -n retail-forecast -w

# Check resource usage
kubectl top pods -n retail-forecast

# View deployment status
kubectl get deployment retail-forecast-api -n retail-forecast

# Check service status
kubectl get service -n retail-forecast

# View events
kubectl get events -n retail-forecast --sort-by='.lastTimestamp'
```

## Performance Optimization

### Resource Tuning

1. **CPU/Memory Requests and Limits**
   - Monitor actual usage with `kubectl top pods`
   - Adjust requests/limits based on observed patterns
   - Use VPA (Vertical Pod Autoscaler) for recommendations

2. **HPA Tuning**
   - Adjust target CPU percentage based on performance
   - Configure scale-up/scale-down policies
   - Add custom metrics for better scaling decisions

3. **Node Selection**
   - Use node selectors for optimal placement
   - Configure tolerations for spot instances
   - Implement pod anti-affinity for high availability

---

**Next Step**: [Task 11: CI/CD Pipeline Integration](../11-cicd-pipeline/)