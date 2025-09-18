---
title: "Elastic Load Balancing (EXTERNAL-IP)"
date: 2024-01-01T00:00:00Z
weight: 11
chapter: false
pre: "<b>11. </b>"
---

## Mục tiêu

Cung cấp public endpoint cho ứng dụng inference chạy trong EKS, cho phép client bên ngoài truy cập API thông qua Load Balancer.

## Nội dung chính

### 1. Load Balancer Options Overview

EKS cung cấp nhiều tùy chọn cho external access:

1. **Service Type LoadBalancer** - Tự động tạo Classic Load Balancer
2. **Network Load Balancer (NLB)** - Layer 4 load balancing với hiệu năng cao
3. **Application Load Balancer (ALB)** - Layer 7 với advanced routing
4. **Ingress Controller** - Quản lý routing và SSL termination

### 2. Service Type LoadBalancer

#### 2.1 Basic LoadBalancer Service

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-loadbalancer
  namespace: mlops
  labels:
    app: retail-forecast-api
  annotations:
    # Use Network Load Balancer for better performance
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "30"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-unhealthy-threshold: "3"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  - port: 443
    targetPort: 8080
    protocol: TCP
    name: https
  selector:
    app: retail-forecast-api
  loadBalancerSourceRanges:
  - 0.0.0.0/0  # Allow access from anywhere (restrict in production)
---
```

#### 2.2 Advanced NLB Configuration

```yaml
# service-nlb-advanced.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-nlb
  namespace: mlops
  labels:
    app: retail-forecast-api
  annotations:
    # Network Load Balancer configuration
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    
    # Health check configuration
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "30"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "6"
    service.beta.kubernetes.io/aws-load-balancer-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-unhealthy-threshold: "2"
    
    # Access logging
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "retail-forecast-nlb-logs"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "nlb-access-logs"
    
    # Subnet configuration
    service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-12345678,subnet-87654321"
    
    # Target type
    service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
    
    # Additional tags
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "Environment=Production,Project=RetailForecast"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: retail-forecast-api
  externalTrafficPolicy: Local  # Preserve source IP
---
```

### 3. AWS Load Balancer Controller (ALB)

#### 3.1 Install AWS Load Balancer Controller

```bash
# Add eks-charts repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Create IAM policy for ALB controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create IAM role and service account
eksctl create iamserviceaccount \
  --cluster=retail-forecast-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=retail-forecast-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### 3.2 ALB Ingress Configuration

```yaml
# ingress-alb.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-forecast-alb-ingress
  namespace: mlops
  annotations:
    # ALB configuration
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: retail-forecast-alb
    
    # Health check configuration
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    
    # SSL configuration
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/YOUR_CERT_ID
    
    # Access logging
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=retail-forecast-alb-logs,
      access_logs.s3.prefix=alb-access-logs
    
    # Additional configurations
    alb.ingress.kubernetes.io/target-group-attributes: |
      stickiness.enabled=false,
      deregistration_delay.timeout_seconds=30,
      slow_start.duration_seconds=30
    
    # WAF integration
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:YOUR_ACCOUNT_ID:regional/webacl/retail-forecast-waf/YOUR_WAF_ID
    
    # Tags
    alb.ingress.kubernetes.io/tags: Environment=Production,Project=RetailForecast
spec:
  rules:
  - host: api.retail-forecast.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retail-forecast-service
            port:
              number: 80
  - http:  # Default rule for requests without host header
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

### 4. Domain và SSL Configuration

#### 4.1 Route 53 DNS Setup

```bash
# Get the ALB DNS name
ALB_DNS=$(kubectl get ingress retail-forecast-alb-ingress -n mlops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Create Route 53 record
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.retail-forecast.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "'$ALB_DNS'"}]
      }
    }]
  }'
```

#### 4.2 ACM Certificate Request

```bash
# Request SSL certificate
aws acm request-certificate \
  --domain-name api.retail-forecast.com \
  --domain-name "*.retail-forecast.com" \
  --validation-method DNS \
  --region us-east-1

# Get certificate ARN
aws acm list-certificates --region us-east-1
```

### 5. Security Configuration

#### 5.1 WAF (Web Application Firewall)

```json
{
  "Name": "retail-forecast-waf",
  "Scope": "REGIONAL",
  "DefaultAction": {
    "Allow": {}
  },
  "Rules": [
    {
      "Name": "RateLimitRule",
      "Priority": 1,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": {
        "Block": {}
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimitRule"
      }
    },
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 2,
      "OverrideAction": {
        "None": {}
      },
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSetMetric"
      }
    }
  ]
}
```

#### 5.2 Security Group Configuration

```bash
# Create security group for ALB
aws ec2 create-security-group \
  --group-name retail-forecast-alb-sg \
  --description "Security group for Retail Forecast ALB"

# Allow HTTP traffic
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow HTTPS traffic
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

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

- [ ] **Load Balancer Service**: Service type LoadBalancer được deploy thành công
- [ ] **External IP**: Service có EXTERNAL-IP được assign từ AWS
- [ ] **Health Checks**: Load balancer health checks hoạt động
- [ ] **Public Access**: API có thể truy cập từ internet
- [ ] **SSL/TLS**: HTTPS endpoint được cấu hình (optional)
- [ ] **Domain Name**: Custom domain được mapping (optional)
- [ ] **Security**: WAF và security groups được cấu hình
- [ ] **Monitoring**: CloudWatch metrics được thiết lập
- [ ] **Load Testing**: Performance testing được thực hiện

### 📊 Verification Steps

1. **Service trong namespace mlops có EXTERNAL-IP hiển thị**
   ```bash
   kubectl get services -n mlops
   # Expected output:
   # NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE
   # retail-forecast-nlb         LoadBalancer   10.100.123.45   a1b2c3d4e5f6g7h8-1234567890.us-east-1.elb.amazonaws.com                80:31234/TCP   5m
   ```

2. **Có thể gửi request từ bên ngoài và nhận kết quả inference**
   ```bash
   # Health check
   curl http://EXTERNAL-IP/healthz
   # Expected: {"status": "healthy", "timestamp": "2024-01-01T12:00:00Z"}
   
   # Prediction request
   curl -X POST http://EXTERNAL-IP/predict \
     -H "Content-Type: application/json" \
     -d '{"features": {"store_id": 1, "product_id": 123}}'
   # Expected: {"prediction": 150.75, "model_version": "v1.0"}
   ```

3. **Load Balancer hoạt động ổn định và phân phối traffic**
   ```bash
   # Check target health
   aws elbv2 describe-target-health --target-group-arn <target-group-arn>
   # Expected: All targets should be "healthy"
   
   # Monitor load distribution
   kubectl logs -f deployment/retail-forecast-api -n mlops
   # Should see requests distributed across different pods
   ```

### 🔍 Monitoring Commands

```bash
# Check service status
kubectl get svc -n mlops -w

# Monitor ingress status
kubectl get ingress -n mlops -w

# Check load balancer controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -f

# Monitor target group health
watch "aws elbv2 describe-target-health --target-group-arn <your-target-group-arn>"

# Check ALB metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/retail-forecast-alb/1234567890123456 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Sum
```

---

**Next Step**: [Task 12: CI/CD Pipeline Automation](../12-cicd-pipeline/)