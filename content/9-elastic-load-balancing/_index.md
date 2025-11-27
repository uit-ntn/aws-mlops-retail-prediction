---
title: "Load Balancing"
date: 2024-01-01T00:00:00Z
weight: 10
chapter: false
pre: "<b>9. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 11:**
{{% /notice %}}

Thiết lập cơ chế phân phối lưu lượng (Load Balancing) cho API Retail Prediction, đảm bảo:  
- Có endpoint public để demo API /predict và /docs  
- Tự động phân phối traffic giữa nhiều Pod khi scaling  
- Duy trì tính sẵn sàng & bảo mật của dịch vụ

📥 **Input từ các Task trước:**
- **Task 5 (Production VPC):** VPC subnets, security groups và VPC Endpoints để ALB và EKS hoạt động
- **Task 7 (EKS Cluster):** EKS cluster và Service/Ingress targets để ALB forward traffic
- **Task 6 (ECR Container Registry):** Container images (API) được deploy vào EKS và expose qua ALB

## 1. Tổng quan về Load Balancing cho Retail Prediction API

Load Balancing là một thành phần thiết yếu trong kiến trúc microservices trên AWS EKS, đặc biệt quan trọng cho Retail Prediction API - dịch vụ dự đoán doanh số có thể phải xử lý khối lượng request lớn và đột biến. Load Balancing đảm bảo:

- **Khả năng mở rộng (Scalability)**: Phân phối đồng đều requests giữa nhiều Pod, đảm bảo API phản hồi nhanh ngay cả khi lượng yêu cầu dự đoán tăng đột biến
- **Tính sẵn sàng cao (High Availability)**: Tự động phát hiện và cách ly Pod không khỏe mạnh, đảm bảo dịch vụ dự đoán luôn hoạt động
- **Auto-scaling tự động**: Kết hợp với HPA để tự động điều chỉnh số lượng Pod dựa trên lưu lượng thực tế
- **Endpoint nhất quán**: Cung cấp một endpoint duy nhất để client (web/mobile app) kết nối tới dịch vụ dự đoán
- **Khả năng quan sát (Observability)**: Thu thập metrics về lưu lượng, latency và errors cho việc theo dõi hiệu suất API

### Kiến trúc Load Balancing trên EKS

{{< mermaid >}}
graph TB
    subgraph "Internet"
        CLIENT[External Clients]
    end
    
    subgraph "AWS Cloud"
        ALB[Application Load Balancer]
        SG[Security Group]
        
        subgraph "EKS Cluster"
            subgraph "Public Subnet"
                INGRESS[AWS Load Balancer Controller]
            end
            
            subgraph "Private Subnet"
                POD1[API Pod 1]
                POD2[API Pod 2]
                POD3[API Pod 3]
                SVC[Kubernetes Service]
            end
        end
    end
    
    CLIENT -- HTTPS --> ALB
    ALB -- Health Check --> POD1
    ALB -- Health Check --> POD2
    ALB -- Health Check --> POD3
    SG -- Firewall --> ALB
    
    ALB -- HTTP --> SVC
    SVC --> POD1
    SVC --> POD2
    SVC --> POD3
    
    class ALB,INGRESS highlight
{{< /mermaid >}}

## 2. Thiết lập Application Load Balancer (ALB)

AWS Application Load Balancer (ALB) là lựa chọn tốt cho Retail Prediction API vì:

- Hoạt động ở Layer 7 (Application Layer)
- Hỗ trợ route các request dựa trên path hoặc host header
- Tích hợp được với AWS WAF để bảo vệ API
- Hỗ trợ SSL/TLS termination

### 2.1 Service Type LoadBalancer với ALB

Cách đơn giản nhất để tạo ALB là sử dụng Service với type là LoadBalancer:

```yaml
# service-alb.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: retail-prediction
  labels:
    app: retail-api
  annotations:
    # Chỉ định sử dụng Application Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-type: "application"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    # Health check configuration
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "30"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  selector:
    app: retail-api
---
```

{{% notice tip %}}
Khi sử dụng annotation `service.beta.kubernetes.io/aws-load-balancer-type: "application"`, EKS sẽ tự động tạo ALB thay vì NLB hoặc Classic Load Balancer.
{{% /notice %}}

### 2.2 Cấu hình Health Checks và Routing

ALB tự động thực hiện health check đến endpoint `/health` để đảm bảo rằng chỉ các Pod khỏe mạnh mới nhận được traffic:

{{< mermaid >}}
sequenceDiagram
    participant ALB as Application Load Balancer
    participant Pod1 as API Pod 1 (Healthy)
    participant Pod2 as API Pod 2 (Unhealthy)
    participant Pod3 as API Pod 3 (Healthy)
    
    ALB->>Pod1: Health Check: GET /health
    Pod1->>ALB: 200 OK: {"status": "healthy"}
    ALB->>Pod2: Health Check: GET /health
    Pod2->>ALB: 500 Error: Service Unavailable
    ALB->>Pod3: Health Check: GET /health
    Pod3->>ALB: 200 OK: {"status": "healthy"}
    
    Note over ALB,Pod2: Pod2 được đánh dấu unhealthy
    
    Client->>ALB: Request: POST /predict
    ALB->>Pod1: Forward request
    Pod1->>ALB: Response
    ALB->>Client: Return response
    
    Client->>ALB: Request: GET /docs
    ALB->>Pod3: Forward request
    Pod3->>ALB: Response
    ALB->>Client: Return response
{{< /mermaid >}}

**Giải thích flow:**

1. ALB liên tục kiểm tra health của tất cả Pod thông qua endpoint `/health`
2. Pod nào trả về status code không phải 200 OK sẽ bị đánh dấu unhealthy
3. Các request từ client sẽ chỉ được forward đến các Pod healthy
4. Khi một Pod unhealthy trở lại healthy, nó sẽ tự động được đưa vào rotation

Nhờ cơ chế này, API luôn duy trì tính sẵn sàng cao ngay cả khi một số Pod gặp sự cố.

### 2.3 Advanced ALB Configuration

```yaml
# service-alb-advanced.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: retail-prediction
  labels:
    app: retail-api
  annotations:
    # Application Load Balancer configuration
    service.beta.kubernetes.io/aws-load-balancer-type: "application"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    
    # Health check configuration
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8000"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
    
    # Access logging (optional)
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "retail-prediction-alb-logs"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "api-access-logs"
    
    # Target group configuration
    service.beta.kubernetes.io/aws-load-balancer-attributes: "idle_timeout.timeout_seconds=60"
    
    # Target type (IP mode preferred for pods)
    service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  selector:
    app: retail-forecast-api
  externalTrafficPolicy: Local  # Preserve source IP
---
```

### 3. AWS Load Balancer Controller cho Ingress

AWS Load Balancer Controller là một controller Kubernetes quản lý ALB (và NLB) cho các dịch vụ trong EKS. Controller này cho phép cấu hình ALB chi tiết hơn thông qua Ingress và IngressClass thay vì chỉ sử dụng annotations trên Service.

#### 3.1 Cài đặt AWS Load Balancer Controller

```bash
# Thêm eks-charts repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Tạo IAM policy cho ALB controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

# Tạo IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Tạo IAM role và service account với IRSA
eksctl create iamserviceaccount \
  --cluster=retail-prediction-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Cài đặt AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=retail-prediction-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

{{% notice info %}}
Controller này hoạt động bằng cách theo dõi các resource Service và Ingress, sau đó tự động tạo và quản lý ALB tương ứng trên AWS. Bạn có thể xác nhận trạng thái của controller bằng lệnh `kubectl get pods -n kube-system | grep aws-load-balancer-controller`.
{{% /notice %}}

#### 3.2 ALB Ingress Configuration cho Retail Prediction API

Cấu hình Ingress cho phép kiểm soát chi tiết hơn cách ALB xử lý traffic, bao gồm routing path-based, SSL termination và nhiều tính năng nâng cao khác:

```yaml
# ingress-retail-prediction.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-api-ingress
  namespace: retail-prediction
  annotations:
    # ALB Controller configuration
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: retail-prediction-alb
    
    # Health check configuration
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '20'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    
    # SSL configuration
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERT_ID>
    
    # Access logging
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=retail-prediction-alb-logs,
      access_logs.s3.prefix=api-access-logs,
      idle_timeout.timeout_seconds=60
    
    # Performance and deregistration configuration
    alb.ingress.kubernetes.io/target-group-attributes: |
      deregistration_delay.timeout_seconds=30,
      slow_start.duration_seconds=30,
      load_balancing.algorithm.type=least_outstanding_requests
    
    # WAF integration for API protection
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:<ACCOUNT_ID>:regional/webacl/retail-prediction-waf/<WAF_ID>
    
    # Tags for cost allocation
    alb.ingress.kubernetes.io/tags: Environment=Production,Project=RetailPrediction,Service=API
spec:
  rules:
  - http:  # Default rule for all incoming requests
      paths:
      # API Documentation
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
      
      # Swagger JSON endpoint
      - path: /openapi.json
        pathType: Exact
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
      
      # Health check and readiness endpoints
      - path: /health
        pathType: Exact
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
      
      # Prediction API endpoints
      - path: /predict
        pathType: Exact
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
      
      # Default catch-all for all other paths
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-api-service
            port:
              number: 80
---
```

{{< mermaid >}}
graph TD
    subgraph "Client Requests"
        C1[Web Client]
        C2[Mobile App]
        C3[Backend Services]
    end
    
    subgraph "ALB (Application Load Balancer)"
        LB[Load Balancer]
        WAF[AWS WAF]
        SSL[SSL Termination]
        LOGS[Access Logging]
    end
    
    subgraph "EKS Cluster"
        ING[Ingress Resource]
        SVC[Retail API Service]
        P1[Pod 1]
        P2[Pod 2]
        P3[Pod 3]
        HC[Health Check]
    end
    
    C1 --> LB
    C2 --> LB
    C3 --> LB
    
    LB --> WAF
    WAF --> SSL
    SSL --> LOGS
    LOGS --> ING
    
    ING -- "/docs" --> SVC
    ING -- "/predict" --> SVC
    ING -- "/health" --> SVC
    
    SVC --> P1
    SVC --> P2
    SVC --> P3
    
    HC --> P1
    HC --> P2
    HC --> P3
    
    style LB fill:#f9f,stroke:#333,stroke-width:2px
    style WAF fill:#bbf,stroke:#333,stroke-width:1px
    style ING fill:#bfb,stroke:#333,stroke-width:2px
{{< /mermaid >}}
```

#### 3.3 Advanced ALB with Multiple Paths

```yaml
# ingress-alb-advanced.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-forecast-alb-advanced
  namespace: mlops
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    
    # Listen ports for HTTP and HTTPS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    
    # Actions for different paths
    alb.ingress.kubernetes.io/actions.weighted-routing: |
      {
        "type": "forward",
        "forwardConfig": {
          "targetGroups": [
            {
              "serviceName": "retail-forecast-service",
              "servicePort": 80,
              "weight": 100
            }
          ]
        }
      }
    
    # Rate limiting
    alb.ingress.kubernetes.io/target-group-attributes: |
      stickiness.enabled=false,
      load_balancing.algorithm.type=round_robin,
      slow_start.duration_seconds=30
spec:
  rules:
  - host: api.retail-forecast.com
    http:
      paths:
      # Health check endpoint
      - path: /healthz
        pathType: Exact
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
      
      # Prediction endpoints
      - path: /predict
        pathType: Exact
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
      
      # Batch prediction
      - path: /batch-predict
        pathType: Exact
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
      
      # Model information
      - path: /model-info
        pathType: Exact
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
      
      # Metrics endpoint
      - path: /metrics
        pathType: Exact
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
      
      # Default catch-all
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
---
```

### 4. HTTPS và SSL/TLS Configuration

Bảo mật API bằng HTTPS là một yêu cầu quan trọng cho mọi dịch vụ production, đặc biệt là dịch vụ dự đoán có thể chứa dữ liệu nhạy cảm của doanh nghiệp.

#### 4.1 Tạo SSL Certificate với AWS Certificate Manager (ACM)

```bash
# Request SSL certificate
aws acm request-certificate \
  --domain-name api.retail-prediction.example.com \
  --validation-method DNS \
  --region us-east-1

# Get certificate ARN
CERT_ARN=$(aws acm list-certificates --region us-east-1 --query "CertificateSummaryList[?DomainName=='api.retail-prediction.example.com'].CertificateArn" --output text)
echo $CERT_ARN
```

#### 4.2 Cấu hình HTTPS cho ALB

```yaml
# service-alb-https.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: retail-prediction
  annotations:
    # Basic ALB configuration
    service.beta.kubernetes.io/aws-load-balancer-type: "application"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    
    # SSL configuration
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:123456789012:certificate/abcdef12-3456-7890-abcd-ef1234567890"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
    
    # Redirect HTTP to HTTPS
    service.beta.kubernetes.io/aws-load-balancer-ssl-redirect: "443"
    
    # Other configurations...
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8000
    protocol: TCP
    name: https
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  selector:
    app: retail-api
```

{{< mermaid >}}
sequenceDiagram
    participant Client
    participant ALB as Application Load Balancer
    participant API as API Pod
    
    Client->>ALB: HTTPS Request (port 443)
    Note over ALB: SSL Termination
    ALB->>API: HTTP Request (port 8080)
    API->>ALB: HTTP Response
    ALB->>Client: HTTPS Response
    
    Client->>ALB: HTTP Request (port 80)
    Note over ALB: HTTP to HTTPS Redirect
    ALB->>Client: 301 Redirect to HTTPS
    Client->>ALB: HTTPS Request (port 443)
    Note over ALB: SSL Termination
    ALB->>API: HTTP Request (port 8080)
    API->>ALB: HTTP Response
    ALB->>Client: HTTPS Response
{{< /mermaid >}}

#### 4.3 Route 53 DNS Setup

```bash
# Get the ALB DNS name
ALB_DNS=$(kubectl get service retail-api-service -n retail-prediction -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Create Route 53 record
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.retail-prediction.example.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "'$ALB_DNS'"}]
      }
    }]
  }'
```

#### 4.4 Xác nhận HTTPS đang hoạt động

```bash
# Test HTTPS endpoint
curl -k https://api.retail-prediction.example.com/health

# Verify with proper SSL validation
curl https://api.retail-prediction.example.com/health

# Test HTTP to HTTPS redirect
curl -v http://api.retail-prediction.example.com/health
# Expected: HTTP/1.1 301 Moved Permanently
```

### 5. Bảo mật cho Retail Prediction API

Việc bảo mật cho API dự đoán retail rất quan trọng vì API có thể chứa thông tin nhạy cảm về doanh số và chiến lược kinh doanh. Chúng ta sẽ triển khai nhiều lớp bảo mật:

#### 5.1 AWS WAF (Web Application Firewall)

WAF giúp bảo vệ API khỏi các cuộc tấn công phổ biến như SQL Injection, XSS và các tấn công DDoS:

```bash
# Tạo WAF Web ACL
aws wafv2 create-web-acl \
  --name retail-prediction-waf \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=RetailAPI \
  --rules file://waf-rules.json
```

Nội dung file `waf-rules.json`:

```json
[
  {
    "Name": "API-RateLimit",
    "Priority": 0,
    "Statement": {
      "RateBasedStatement": {
        "Limit": 3000,
        "AggregateKeyType": "IP"
      }
    },
    "Action": {
      "Block": {}
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "API-RateLimit"
    }
  },
  {
    "Name": "AWSManagedRulesSQLiRuleSet",
    "Priority": 1,
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesSQLiRuleSet"
      }
    },
    "OverrideAction": {
      "None": {}
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "SQLiRuleSet"
    }
  },
  {
    "Name": "BlockHighRiskRequests",
    "Priority": 2,
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesKnownBadInputsRuleSet"
      }
    },
    "OverrideAction": {
      "None": {}
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "BadInputs"
    }
  }
]
```

#### 5.2 Security Group Configuration

```bash
# Tạo security group cho ALB
aws ec2 create-security-group \
  --group-name retail-prediction-alb-sg \
  --description "Security group for Retail Prediction API ALB" \
  --vpc-id vpc-0123456789abcdef0

# Cho phép traffic HTTPS (port 443)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --description "HTTPS from anywhere"

# Cho phép traffic HTTP (port 80) - chỉ để redirect sang HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --description "HTTP from anywhere (for redirect only)"
```

#### 5.3 Phân quyền API với JWT Authentication

Để bảo vệ API khỏi truy cập trái phép, chúng ta nên triển khai xác thực JWT:

```python
# Thêm mã sau vào file main.py của FastAPI
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
import jwt
from jwt.exceptions import PyJWTError

# Setup OAuth2
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# JWT Configuration
JWT_SECRET_KEY = os.getenv("JWT_SECRET_KEY", "your-secret-key")  # Nên lưu trong AWS Secrets Manager
JWT_ALGORITHM = "HS256"

# Verify JWT token
def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, JWT_SECRET_KEY, algorithms=[JWT_ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        return {"username": username}
    except PyJWTError:
        raise credentials_exception

# Protected API endpoint
@app.post("/predict", response_model=PredictionResponse)
async def predict(
    request: PredictionRequest,
    current_user: dict = Depends(get_current_user)
):
    # Log the user who made the prediction
    logger.info(f"Prediction requested by user: {current_user['username']}")
    
    # Process prediction as normal
    prediction = model.predict(request.features)
    return {"prediction": float(prediction), "model_version": model.version}
```

{{< mermaid >}}
sequenceDiagram
    participant Client
    participant ALB as ALB + WAF
    participant API as Retail API
    participant Auth as Authentication Service
    
    Client->>Auth: Request JWT token (username/password)
    Auth->>Auth: Verify credentials
    Auth->>Client: Return JWT token
    
    Client->>ALB: POST /predict with Bearer token
    ALB->>ALB: WAF rules check
    ALB->>API: Forward request
    API->>API: Validate JWT token
    API->>API: Perform prediction
    API->>ALB: Return prediction result
    ALB->>Client: Return response
    
    Client->>ALB: POST /predict (no token)
    ALB->>ALB: WAF rules check
    ALB->>API: Forward request
    API->>ALB: 401 Unauthorized
    ALB->>Client: 401 Unauthorized
{{< /mermaid >}}

### 6. Deployment và Verification

#### 6.1 Deploy Load Balancer Services

```bash
# Create mlops namespace if not exists
kubectl create namespace mlops

# Deploy LoadBalancer service
kubectl apply -f service-nlb-advanced.yaml

# Or deploy ALB Ingress
kubectl apply -f ingress-alb.yaml

# Check service status
kubectl get services -n mlops

# Check ingress status
kubectl get ingress -n mlops
```

#### 6.2 Verify External Access

```bash
# Get external IP/hostname
EXTERNAL_IP=$(kubectl get service retail-forecast-nlb -n mlops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "External endpoint: $EXTERNAL_IP"

# Test health check
curl http://$EXTERNAL_IP/healthz

# Test prediction endpoint
curl -X POST http://$EXTERNAL_IP/predict \
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

### 7. Load Testing và Performance

#### 7.1 Load Testing with Apache Bench

```bash
# Install Apache Bench
# For Ubuntu/Debian: sudo apt-get install apache2-utils
# For CentOS/RHEL: sudo yum install httpd-tools

# Basic load test
ab -n 1000 -c 10 http://$EXTERNAL_IP/healthz

# POST request load test
ab -n 100 -c 5 -p data.json -T application/json http://$EXTERNAL_IP/predict
```

#### 7.2 Load Testing with Artillery

```bash
# Install Artillery
npm install -g artillery

# Create load test configuration
cat > load-test.yml << EOF
config:
  target: 'http://$EXTERNAL_IP'
  phases:
    - duration: 60
      arrivalRate: 10
    - duration: 120
      arrivalRate: 20
    - duration: 60
      arrivalRate: 5
scenarios:
  - name: "Health Check"
    weight: 30
    flow:
      - get:
          url: "/healthz"
  - name: "Prediction"
    weight: 70
    flow:
      - post:
          url: "/predict"
          json:
            features:
              store_id: 1
              product_id: 123
              date: "2024-01-01"
              price: 29.99
              promotion: 0
EOF

# Run load test
artillery run load-test.yml
```

### 8. Monitoring và Observability

#### 8.1 CloudWatch Metrics

```bash
# Create CloudWatch dashboard for ALB metrics
aws cloudwatch put-dashboard \
  --dashboard-name "RetailForecastALB" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "retail-forecast-alb"],
            [".", "TargetResponseTime", ".", "."],
            [".", "HTTPCode_Target_2XX_Count", ".", "."],
            [".", "HTTPCode_Target_4XX_Count", ".", "."],
            [".", "HTTPCode_Target_5XX_Count", ".", "."]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1",
          "title": "ALB Metrics"
        }
      }
    ]
  }'
```

#### 8.2 Prometheus Monitoring

```yaml
# servicemonitor-loadbalancer.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: retail-forecast-lb-monitor
  namespace: mlops
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

### 9. Troubleshooting

#### 9.1 Common Load Balancer Issues

```bash
# Check service events
kubectl describe service retail-forecast-nlb -n mlops

# Check ingress events  
kubectl describe ingress retail-forecast-alb-ingress -n mlops

# Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check target group health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/k8s-mlops-retailfo-1234567890
```

#### 9.2 Debug Network Connectivity

```bash
# Test from inside cluster
kubectl run debug-pod --image=nicolaka/netshoot -n mlops --rm -it -- /bin/bash

# Inside the debug pod:
# Test service connectivity
curl http://retail-forecast-service.mlops.svc.cluster.local/healthz

# Test DNS resolution
nslookup retail-forecast-service.mlops.svc.cluster.local

# Test external connectivity
curl http://api.retail-forecast.com/healthz
```

### 10. Cost Optimization

#### 10.1 ALB vs NLB Cost Comparison

```bash
# ALB pricing (approximate)
# - $0.0225 per ALB-hour
# - $0.008 per LCU-hour (Load Balancer Capacity Unit)

# NLB pricing (approximate) 
# - $0.0225 per NLB-hour
# - $0.006 per NLCU-hour (Network Load Balancer Capacity Unit)

# Monitor usage with AWS Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

#### 10.2 Resource Optimization

```yaml
# Use target type "ip" for better performance
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
    # Enable cross-zone load balancing only if needed
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  # Use Local traffic policy to reduce network hops
  externalTrafficPolicy: Local
---
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **Application Load Balancer**: Service type LoadBalancer với annotation `aws-load-balancer-type: "application"` được deploy thành công
- [ ] **External Endpoint**: ALB DNS được assign và có thể truy cập
- [ ] **Health Checks**: ALB health checks tới endpoint `/health` hoạt động đúng
- [ ] **API Access**: Các endpoint `/predict` và `/docs` có thể truy cập từ internet
- [ ] **Logging**: Access logs được ghi vào S3 bucket
- [ ] **SSL/TLS**: HTTPS endpoint được cấu hình (optional)
- [ ] **Domain Name**: Custom domain được mapping qua Route 53 (optional)
- [ ] **Security**: WAF rules và security groups được cấu hình
- [ ] **Monitoring**: CloudWatch ALB metrics được kích hoạt
- [ ] **Load Testing**: Performance testing đạt các target về throughput và latency

### 📊 Verification Steps

1. **Service trong namespace retail-prediction có EXTERNAL-IP hiển thị**
   ```bash
   kubectl get services -n retail-prediction
   # Expected output:
   # NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                      PORT(S)        AGE
   # retail-api-service LoadBalancer   10.100.123.45   retail-alb-123456789.us-east-1.elb.amazonaws.com 80:31234/TCP   5m
   ```

2. **Có thể gửi request từ bên ngoài và nhận kết quả inference**
   ```bash
   # Health check
   curl http://EXTERNAL-IP/health
   # Expected: {"status": "healthy", "timestamp": "2024-01-01T12:00:00Z"}
   
   # API Documentation
   curl http://EXTERNAL-IP/docs
   # Expected: FastAPI Swagger UI loads
   
   # Prediction request
   curl -X POST http://EXTERNAL-IP/predict \
     -H "Content-Type: application/json" \
     -d '{"features": {"store_id": 1, "product_id": 123, "date": "2024-01-01", "price": 29.99, "promotion": 0}}'
   # Expected: {"prediction": 150.75, "model_version": "v1.0"}
   ```

3. **Load Balancer hoạt động ổn định và phân phối traffic**
   ```bash
   # Check target health
   aws elbv2 describe-target-health --target-group-arn <target-group-arn>
   # Expected: All targets should be "healthy"
   
   # Monitor load distribution
   kubectl logs -f deployment/retail-api -n retail-prediction
   # Should see requests distributed across different pods
   
   # Check ALB metrics
   aws cloudwatch get-metric-statistics \
     --namespace AWS/ApplicationELB \
     --metric-name RequestCount \
     --dimensions Name=LoadBalancer,Value=app/retail-alb/1234567890123456 \
     --start-time $(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "1 hour ago") \
     --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
     --period 300 \
     --statistics Sum
   ```

### 🔍 Monitoring Commands

```bash
# Check service status
kubectl get svc -n retail-prediction -w

# Monitor ALB controller status
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check load balancer controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -f

# Monitor target group health
watch "aws elbv2 describe-target-health --target-group-arn <your-target-group-arn>"

# Get ALB details
aws elbv2 describe-load-balancers --names retail-api-alb

# Check ALB metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/retail-api-alb/1234567890123456 \
  --start-time $(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "1 hour ago") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --period 300 \
  --statistics Sum

# Check target response time metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/retail-api-alb/1234567890123456 \
  --start-time $(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "1 hour ago") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --period 300 \
  --statistics Average
```

---

**Next Step**: [Task 12: CloudWatch Monitoring & Alerting](../12-cloudwatch/)
