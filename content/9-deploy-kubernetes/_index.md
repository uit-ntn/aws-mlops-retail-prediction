---
title: "API Deployment on EKS"
date: 2024-01-01T00:00:00Z
weight: 9
chapter: false
pre: "<b>9. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 10:**

Triển khai Retail Prediction API (FastAPI) lên EKS Cluster, kết nối model từ S3 và expose endpoint public qua Load Balancer (ALB).  
→ Đảm bảo dịch vụ chạy ổn định, tự động scale, bảo mật, và có thể demo API thật.
{{% /notice %}}

## 1. Tổng quan

**API Deployment** là bước triển khai service dự đoán đã được container hóa lên Kubernetes (EKS). Bước này đảm bảo ứng dụng được triển khai theo kiến trúc microservice, tự động scale và có tính sẵn sàng cao.

### Kiến trúc triển khai

{{< mermaid >}}
graph TD
    subgraph "AWS Cloud"
        subgraph "Amazon EKS Cluster"
            subgraph "Retail Prediction Namespace"
                PODS[API Pods]
                SVC[Service]
                HPA[HorizontalPodAutoscaler]
                CM[ConfigMap]
                SA[ServiceAccount]
            end
            
            subgraph "Infrastructure"
                CA[Cluster Autoscaler]
                NODE1[Worker Node]
                NODE2[Worker Node Spot]
            end
        end
        
        S3[S3 Bucket<br/>Models]
        ALB[Application Load Balancer]
        IAM[IAM Role<br/>for ServiceAccount]
    end
    
    subgraph "Client Applications"
        CLIENT1[Web Frontend]
        CLIENT2[Mobile App]
    end
    
    IAM --> SA
    SA --> PODS
    PODS --> S3
    CM --> PODS
    HPA --> PODS
    SVC --> PODS
    ALB --> SVC
    CLIENT1 --> ALB
    CLIENT2 --> ALB
    CA --> NODE1
    CA --> NODE2
    PODS --> NODE1
    PODS --> NODE2
{{< /mermaid >}}

## 2. Chuẩn bị file Kubernetes Manifests

Cấu trúc file Kubernetes để triển khai Retail Prediction API trên EKS:

```
aws/k8s/
├── namespace.yaml
├── configmap.yaml
├── serviceaccount.yaml
├── deployment.yaml
├── service.yaml
└── hpa.yaml
```

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
  MODEL_BUCKET: "mlops-retail-models"
  MODEL_KEY: "artifacts/model-v1/model.tar.gz"
  AWS_REGION: "ap-southeast-1"
  
  # API Configuration
  PORT: "8080"
  HOST: "0.0.0.0"
  WORKERS: "2"
  
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

{{% notice note %}}
IRSA (IAM Role for Service Account) cho phép Pod Kubernetes sử dụng IAM role để access AWS services mà không cần hardcode credentials. Đây là best practice khi kết nối đến S3, DynamoDB, và các dịch vụ AWS khác.
{{% /notice %}}

#### Tạo IAM Role cho IRSA

1. **Tạo IAM Trust Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-1.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:retail-prediction:s3-access"
        }
      }
    }
  ]
}
```

2. **Tạo IAM Policy (S3 Access):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mlops-retail-models",
        "arn:aws:s3:::mlops-retail-models/*"
      ]
    }
  ]
}
```

3. **Tạo IAM Role sử dụng eksctl:**

```bash
# Tạo IAM Role và liên kết với ServiceAccount
eksctl create iamserviceaccount \
  --name=s3-access \
  --namespace=retail-prediction \
  --cluster=mlops-eks-cluster \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/RetailAPIModelAccess \
  --approve \
  --region=ap-southeast-1
```

## 3. Deployment Configuration

### 3.1 Main Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: retail-prediction
  labels:
    app: retail-api
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
      serviceAccountName: s3-access
      containers:
      - name: retail-api
        image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/retail-prediction-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        
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
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
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
        
        # Volume Mounts
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: model-cache
          mountPath: /app/model
      
      # Volumes
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: model-cache
        emptyDir:
          sizeLimit: 1Gi
      
      # Tolerations for spot instances (cost saving)
      tolerations:
      - key: "node.kubernetes.io/instance-type"
        operator: "Equal" 
        value: "spot"
        effect: "NoSchedule"
---
```

{{% notice tip %}}
Chạy API trên Spot Instances giúp tiết kiệm chi phí tới 70% so với On-Demand Instances. Chúng ta đã thiết lập tolerations để pods có thể chạy trên spot nodes.
{{% /notice %}}

#### Cấu hình Worker Nodes (Spot Instances)

Để sử dụng Spot instances với EKS, bạn có thể tạo một Node Group với Spot instances:

```bash
eksctl create nodegroup \
  --cluster=mlops-eks-cluster \
  --region=ap-southeast-1 \
  --name=spot-workers \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=4 \
  --spot \
  --labels="lifecycle=Ec2Spot" \
  --tags="k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/mlops-eks-cluster=true"
```

## 4. Service Configuration

### 4.1 LoadBalancer Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: retail-prediction
  labels:
    app: retail-api
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
    app: retail-api
---
```

{{% notice info %}}
Khi sử dụng `type: LoadBalancer` trong AWS EKS, AWS sẽ tự động provision một Network Load Balancer (NLB) hoặc Application Load Balancer (ALB) để expose service ra internet. ALB có thể được cấu hình qua Ingress để có thêm nhiều tính năng như SSL, path routing...
{{% /notice %}}

## 5. Horizontal Pod Autoscaler (HPA)

### 5.1 HPA Configuration

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

## 6. Ingress Configuration (Optional)

Nếu bạn muốn sử dụng AWS Application Load Balancer (ALB) với nhiều tính năng routing nâng cao hơn NLB, bạn có thể sử dụng Ingress resource:

### 6.1 Cài đặt AWS Load Balancer Controller

Trước khi sử dụng Ingress với ALB, cần cài đặt AWS Load Balancer Controller:

```bash
# Tạo IAM Policy cho AWS Load Balancer Controller
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# Tạo IAM Role cho ServiceAccount
eksctl create iamserviceaccount \
  --cluster=mlops-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# Cài đặt AWS Load Balancer Controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=mlops-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 6.2 Application Load Balancer Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-api-ingress
  namespace: retail-prediction
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: retail-prediction-alb
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
---
```

## 7. Triển khai và Xác minh

### 7.1 Triển khai Resources

```bash
# Tạo namespace
kubectl apply -f aws/k8s/namespace.yaml

# Tạo ConfigMap và ServiceAccount
kubectl apply -f aws/k8s/configmap.yaml
kubectl apply -f aws/k8s/serviceaccount.yaml

# Triển khai ứng dụng
kubectl apply -f aws/k8s/deployment.yaml
kubectl apply -f aws/k8s/service.yaml
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

## 9. Monitoring và Troubleshooting

### 9.1 Monitoring Resources

```bash
# Kiểm tra resource usage của pods
kubectl top pods -n retail-prediction

# Kiểm tra resource usage của nodes
kubectl top nodes

# Xem chi tiết metrics của HPA
kubectl describe hpa retail-api-hpa -n retail-prediction

# Theo dõi logs của pods
kubectl logs -f deployment/retail-api -n retail-prediction
```

### 9.2 Manual Scaling

Trong trường hợp cần scale thủ công:

```bash
# Scale deployment lên 5 replicas
kubectl scale deployment retail-api --replicas=5 -n retail-prediction

# Kiểm tra trạng thái scaling
kubectl get deployment retail-api -n retail-prediction
```

### 9.3 Troubleshooting

Các lệnh phổ biến để debug issues:

```bash
# Kiểm tra events của pod
kubectl describe pod <pod-name> -n retail-prediction

# Kiểm tra events của deployment
kubectl describe deployment retail-api -n retail-prediction

# Kiểm tra endpoints của service
kubectl describe service retail-api-service -n retail-prediction

# Kiểm tra events của HPA
kubectl describe hpa retail-api-hpa -n retail-prediction

# Truy cập shell trong pod
kubectl exec -it <pod-name> -n retail-prediction -- /bin/sh

# Kiểm tra logs của container trước đó (nếu restart)
kubectl logs <pod-name> -n retail-prediction --previous
```

## 10. Updates và Rollbacks

### 10.1 Cập nhật version mới

```bash
# Update image version
kubectl set image deployment/retail-api \
  retail-api=<AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/retail-prediction-api:v2 \
  -n retail-prediction

# Kiểm tra trạng thái rollout
kubectl rollout status deployment/retail-api -n retail-prediction
```

### 10.2 Rollback khi cần thiết

```bash
# Kiểm tra lịch sử rollout
kubectl rollout history deployment/retail-api -n retail-prediction

# Rollback về version trước đó
kubectl rollout undo deployment/retail-api -n retail-prediction

# Rollback về một revision cụ thể
kubectl rollout undo deployment/retail-api --to-revision=2 -n retail-prediction
```

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