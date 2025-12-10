---
title: "Load Balancing"
date: 2024-01-01T00:00:00Z
weight: 10
chapter: false
pre: "<b>9. </b>"
---

{{% notice info %}}
**🎯 Task 11 Objective:**
{{% /notice %}}

Set up load balancing for the Retail Prediction API, ensuring:

- A public endpoint to demo the API (`/predict` and `/docs`)
- Automatic traffic distribution across Pods when scaling
- High availability and security of the service

📥 **Input from previous Tasks:**

- **Task 5 (Production VPC):** VPC subnets, security groups and VPC Endpoints required for ALB and EKS
- **Task 7 (EKS Cluster):** EKS cluster and Service/Ingress targets for ALB to forward traffic
- **Task 6 (ECR Container Registry):** Container images (API) deployed to EKS and exposed via ALB

## 1. Overview — Load Balancing for Retail Prediction API

Load balancing is essential in a microservices architecture on AWS EKS, especially for the Retail Prediction API which may handle high and spiky request volumes. Load balancing provides:

- **Scalability**: evenly distribute requests across Pods so the API remains responsive under high load
- **High Availability**: automatically detect and isolate unhealthy Pods to keep the service running
- **Auto-scaling integration**: work with HPA to adjust Pod counts based on real traffic
- **Consistent endpoint**: provide a single endpoint for clients (web/mobile) to access the prediction service
- **Observability**: collect metrics for traffic, latency and errors to monitor API performance

### 2. Setting up an Application Load Balancer (ALB)

AWS Application Load Balancer (ALB) is a good choice for the Retail Prediction API because:

- Operates at Layer 7 (Application Layer)
- Supports routing requests based on path or host header
- Integrates with AWS WAF for API protection
- Supports SSL/TLS termination

### 2.1 Service Type LoadBalancer with ALB

The simplest way to create an ALB is to use a Kubernetes `Service` of type `LoadBalancer`:

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
    # Specify the use of Application Load Balancer
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
When using the annotation `service.beta.kubernetes.io/aws-load-balancer-type: "application"`, EKS will automatically create an ALB instead of an NLB or Classic Load Balancer.
{{% /notice %}}

### 2.2 Health Checks and Routing

ALB continuously performs health checks against the `/health` endpoint to ensure only healthy Pods receive traffic:

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

Flow explanation:

1. ALB continuously checks the health of all Pods via the `/health` endpoint
2. Any Pod returning a non-200 status is marked unhealthy
3. Client requests are only forwarded to healthy Pods
4. When an unhealthy Pod becomes healthy again, it is automatically brought back into rotation

This mechanism keeps the API highly available even when some Pods fail.

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
  externalTrafficPolicy: Local # Preserve source IP
---
```

### 3. AWS Load Balancer Controller for Ingress

The AWS Load Balancer Controller is a Kubernetes controller that manages ALB (and NLB) for services in EKS. It enables more advanced ALB configuration via `Ingress` and `IngressClass`, rather than only using Service annotations.

#### 3.1 Cài đặt AWS Load Balancer Controller

```bash
# Add eks-charts repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Create IAM policy for the ALB controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create IAM role and service account using IRSA
eksctl create iamserviceaccount \
  --cluster=retail-prediction-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=retail-prediction-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

{{% notice info %}}
The controller watches Service and Ingress resources and automatically creates and manages the corresponding ALBs on AWS. Confirm controller status with `kubectl get pods -n kube-system | grep aws-load-balancer-controller`.
{{% /notice %}}

#### 3.2 ALB Ingress Configuration for Retail Prediction API

The Ingress configuration allows finer control over how the ALB handles traffic, including path-based routing, SSL termination and advanced features:

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
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "20"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"

    # SSL configuration
    alb.ingress.kubernetes.io/ssl-redirect: "443"
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
    - http: # Default rule for all incoming requests
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

### 4. HTTPS and SSL/TLS Configuration

Securing the API with HTTPS is a critical requirement for any production service, especially when the prediction service may contain sensitive business data.

#### 4.1 Request an SSL Certificate with AWS Certificate Manager (ACM)

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

#### 4.2 Configure HTTPS for the ALB

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

#### 4.4 Verify HTTPS is working

```bash
# Test HTTPS endpoint
curl -k https://api.retail-prediction.example.com/health

# Verify with proper SSL validation
curl https://api.retail-prediction.example.com/health

# Test HTTP to HTTPS redirect
curl -v http://api.retail-prediction.example.com/health
# Expected: HTTP/1.1 301 Moved Permanently
```

### 5. Security for the Retail Prediction API

Securing the prediction API is important because it may contain sensitive sales and business strategy information. We'll implement multiple security layers:

#### 5.1 AWS WAF (Web Application Firewall)

WAF protects the API from common attacks such as SQL Injection, XSS and DDoS:

```bash
# Create WAF Web ACL
aws wafv2 create-web-acl \
  --name retail-prediction-waf \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=RetailAPI \
  --rules file://waf-rules.json
```

# Contents of `waf-rules.json`:

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
# Create a security group for the ALB
aws ec2 create-security-group \
  --group-name retail-prediction-alb-sg \
  --description "Security group for Retail Prediction API ALB" \
  --vpc-id vpc-0123456789abcdef0

# Allow HTTPS traffic (port 443)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --description "HTTPS from anywhere"

# Allow HTTP traffic (port 80) - only for redirect to HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --description "HTTP from anywhere (for redirect only)"
```

#### 5.3 API Authorization with JWT Authentication

To protect the API from unauthorized access implement JWT authentication:

```python
# Add the following code to your FastAPI `main.py`
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

### 6. Deployment and Verification

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

### 7. Load Testing and Performance

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

### 8. Monitoring and Observability

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

## Expected Results

### ✅ Completion Checklist

- [ ] **Application Load Balancer**: Service type `LoadBalancer` with annotation `aws-load-balancer-type: "application"` deployed successfully
- [ ] **External Endpoint**: ALB DNS assigned and accessible
- [ ] **Health Checks**: ALB health checks against `/health` are functioning correctly
- [ ] **API Access**: Endpoints `/predict` and `/docs` are reachable from the internet
- [ ] **Logging**: Access logs are written to an S3 bucket
- [ ] **SSL/TLS**: HTTPS endpoint configured (optional)
- [ ] **Domain Name**: Custom domain mapped via Route 53 (optional)
- [ ] **Security**: WAF rules and security groups configured
- [ ] **Monitoring**: CloudWatch ALB metrics enabled
- [ ] **Load Testing**: Performance testing meets throughput and latency targets

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

## 11. Clean Up Resources (AWS CLI)

### 11.1. Xóa Load Balancers

```bash
# Liệt kê ALBs
aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `retail`)].{Name:LoadBalancerName,ARN:LoadBalancerArn}' --output table

# Xóa ALB (tự động xóa listeners)
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>

# Liệt kê Target Groups
aws elbv2 describe-target-groups --query 'TargetGroups[?contains(TargetGroupName, `retail`)].{Name:TargetGroupName,ARN:TargetGroupArn}' --output table

# Xóa Target Groups
aws elbv2 delete-target-group --target-group-arn <target-group-arn>
```

### 11.2. Xóa AWS Load Balancer Controller

```bash
# Uninstall AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system

# Xóa IRSA role cho load balancer controller
aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy

aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole

# Xóa IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy
```

### 11.3. Xóa Ingress Resources

```bash
# Xóa Ingress resources
kubectl delete ingress --all -n mlops

# Xóa IngressClass
kubectl delete ingressclass alb

# Verify cleanup
kubectl get ingress -A
kubectl get ingressclass
```

### 11.4. Xóa WAF và Security Configurations

```bash
# Liệt kê WAF Web ACLs
aws wafv2 list-web-acls --scope REGIONAL --region ap-southeast-1

# Xóa WAF rules trước
aws wafv2 update-web-acl \
  --scope REGIONAL \
  --id <web-acl-id> \
  --lock-token <lock-token> \
  --rules '[]' \
  --default-action Allow={}

# Xóa WAF Web ACL
aws wafv2 delete-web-acl \
  --scope REGIONAL \
  --id <web-acl-id> \
  --lock-token <lock-token>
```

### 11.5. Clean Up SSL Certificates

```bash
# Liệt kê ACM certificates
aws acm list-certificates --region ap-southeast-1 --query 'CertificateSummaryList[*].{Domain:DomainName,ARN:CertificateArn}'

# Xóa certificate (chỉ khi không còn sử dụng)
aws acm delete-certificate --certificate-arn <certificate-arn> --region ap-southeast-1

# Xóa Route53 records
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "api.retail-prediction.example.com",
        "Type": "A",
        "AliasTarget": {
          "DNSName": "<alb-dns-name>",
          "EvaluateTargetHealth": false,
          "HostedZoneId": "<alb-zone-id>"
        }
      }
    }]
  }'
```

### 11.6. Load Balancing Cleanup Script

```bash
#!/bin/bash
# loadbalancer-cleanup.sh

NAMESPACE="mlops"
CLUSTER_NAME="retail-prediction-cluster"
REGION="ap-southeast-1"

echo "🧹 Cleaning up Load Balancer resources..."

# 1. Delete Kubernetes resources
echo "Deleting Ingress resources..."
kubectl delete ingress --all -n $NAMESPACE
kubectl delete service --selector=app=retail-api -n $NAMESPACE

# 2. Wait for ALB deletion
echo "Waiting for ALB deletion..."
sleep 60

# 3. Clean up AWS resources
echo "Cleaning up AWS Load Balancer Controller..."
helm uninstall aws-load-balancer-controller -n kube-system 2>/dev/null || true

# 4. Delete remaining target groups
echo "Cleaning up target groups..."
TARGET_GROUPS=$(aws elbv2 describe-target-groups --query 'TargetGroups[?contains(TargetGroupName, `k8s-`)].TargetGroupArn' --output text)
for tg in $TARGET_GROUPS; do
    aws elbv2 delete-target-group --target-group-arn $tg 2>/dev/null || true
done

echo "✅ Load Balancer cleanup completed"
```

---

## 12. Bảng giá Load Balancing (ap-southeast-1)

### 12.1. Chi phí Application Load Balancer (ALB)

| Component                             | Giá (USD/hour) | Giá (USD/month) | Ghi chú                 |
| ------------------------------------- | -------------- | --------------- | ----------------------- |
| **ALB Base Cost**                     | $0.0225        | $16.43          | Per ALB instance        |
| **LCU (Load Balancer Capacity Unit)** | $0.008         | $5.84           | Per LCU-hour            |
| **New connections**                   | Included       | Included        | Up to 25/second per LCU |
| **Active connections**                | Included       | Included        | Up to 3,000 per LCU     |
| **Data processed**                    | Included       | Included        | Up to 1GB per LCU       |
| **Rule evaluations**                  | Included       | Included        | Up to 1,000 per LCU     |

### 12.2. Chi phí Network Load Balancer (NLB)

| Component                    | Giá (USD/hour) | Giá (USD/month) | Ghi chú                   |
| ---------------------------- | -------------- | --------------- | ------------------------- |
| **NLB Base Cost**            | $0.0225        | $16.43          | Per NLB instance          |
| **NLCU (Network LCU)**       | $0.006         | $4.38           | Per NLCU-hour             |
| **New connections/flows**    | Included       | Included        | Up to 800/second per NLCU |
| **Active connections/flows** | Included       | Included        | Up to 100,000 per NLCU    |
| **Data processed**           | Included       | Included        | Up to 1GB per NLCU        |

### 12.3. Chi phí Classic Load Balancer (CLB)

| Component         | Giá (USD/hour) | Giá (USD/month) | Ghi chú          |
| ----------------- | -------------- | --------------- | ---------------- |
| **CLB Base Cost** | $0.025         | $18.25          | Per CLB instance |
| **Data Transfer** | $0.008/GB      | Variable        | Data processed   |

### 12.4. So sánh các loại Load Balancer

| Feature                | ALB                  | NLB               | CLB          | Best For                             |
| ---------------------- | -------------------- | ----------------- | ------------ | ------------------------------------ |
| **Layer**              | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) | Layer 4/7    | ALB: Web apps, NLB: High performance |
| **Base Cost**          | $16.43/month         | $16.43/month      | $18.25/month | ALB/NLB cheaper                      |
| **Capacity Units**     | LCU ($5.84)          | NLCU ($4.38)      | Fixed        | NLB most cost-effective              |
| **SSL Termination**    | ✅                   | ✅                | ✅           | All support                          |
| **Path-based Routing** | ✅                   | ❌                | ❌           | ALB only                             |
| **WebSocket**          | ✅                   | ✅                | ❌           | ALB/NLB                              |
| **Static IP**          | ❌                   | ✅                | ❌           | NLB only                             |

### 12.5. AWS Load Balancer Controller Costs

| Component                | Cost | Ghi chú                      |
| ------------------------ | ---- | ---------------------------- |
| **Controller Pods**      | Free | Runs on existing EKS nodes   |
| **IRSA Role**            | Free | IAM integration              |
| **Webhook Certificate**  | Free | TLS for admission controller |
| **Target Group Binding** | Free | CRD for pod registration     |
| **Ingress Management**   | Free | Kubernetes native            |

### 12.6. SSL/TLS Certificate Costs

| Service                           | Cost              | Features                    |
| --------------------------------- | ----------------- | --------------------------- |
| **AWS Certificate Manager (ACM)** | Free              | Public SSL certificates     |
| **Route 53 DNS**                  | $0.50/hosted zone | Domain validation           |
| **Third-party Certificates**      | $10-100/year      | Extended validation options |

### 12.7. WAF (Web Application Firewall) Costs

| Component         | Giá (USD/month) | Ghi chú                 |
| ----------------- | --------------- | ----------------------- |
| **WAF Web ACL**   | $1              | Per Web ACL             |
| **WAF Rules**     | $0.60           | Per rule per month      |
| **WAF Requests**  | $0.60           | Per million requests    |
| **Bot Control**   | $10             | Advanced bot protection |
| **Rate Limiting** | $2              | Per rate-based rule     |

### 12.8. Ước tính chi phí Task 9

**Basic ALB Setup:**

| Component           | Usage         | Monthly Cost |
| ------------------- | ------------- | ------------ |
| **ALB Base**        | 1 ALB         | $16.43       |
| **LCU Usage**       | 1 LCU average | $5.84        |
| **ACM Certificate** | 1 domain      | $0           |
| **Route 53**        | 1 hosted zone | $0.50        |
| **WAF (optional)**  | Basic rules   | $3.20        |
| **Total**           |               | **$26.00**   |

**Production Setup với High Availability:**

| Component           | Usage            | Monthly Cost |
| ------------------- | ---------------- | ------------ |
| **ALB Base**        | 1 ALB (Multi-AZ) | $16.43       |
| **LCU Usage**       | 3 LCU average    | $17.52       |
| **ACM Certificate** | 2 domains        | $0           |
| **Route 53**        | 2 hosted zones   | $1.00        |
| **WAF**             | Advanced rules   | $15.20       |
| **CloudFront**      | CDN integration  | $8.50        |
| **Total**           |                  | **$58.65**   |

### 12.9. Cost Optimization Strategies

**Single ALB for Multiple Services:**

```yaml
# Cost-effective: 1 ALB serves multiple applications
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shared-alb-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: shared-alb
spec:
  rules:
    - host: api.retail.com
      http:
        paths:
          - path: /predict
            backend:
              service:
                name: retail-api
                port:
                  number: 80
    - host: admin.retail.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 80
```

**Right-size Load Balancer:**

- Monitor LCU/NLCU usage với CloudWatch
- Optimize connection pooling
- Use appropriate health check intervals
- Configure proper idle timeouts

### 12.10. Monitoring và Cost Control

```bash
# Monitor ALB metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name ConsumedLCUs \
  --dimensions Name=LoadBalancer,Value=<alb-full-name> \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum

# Check ALB costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Load Balancing"]}}'

# Monitor target health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Health:TargetHealth.State}'
```

**Cost alerts:**

```bash
# Create cost alarm for Load Balancing
aws cloudwatch put-metric-alarm \
  --alarm-name "LoadBalancer-Cost-Alert" \
  --alarm-description "Alert when Load Balancer cost > $50/month" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD Name=ServiceName,Value=AmazonELB
```

{{% notice info %}}
**💰 Cost Summary cho Task 9:**

- **Basic ALB:** $26/month (single service)
- **Shared ALB:** $22/month (multiple services sharing 1 ALB)
- **Production:** $58.65/month (với WAF, CloudFront)
- **Optimization:** 15-30% savings với proper resource sharing
  {{% /notice %}}

---

## 🎬 Video thực hiện Task 9

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/gYYxZNCSR_Q" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 10: CloudWatch Monitoring & Alerting](../10-cloudwatch/)
