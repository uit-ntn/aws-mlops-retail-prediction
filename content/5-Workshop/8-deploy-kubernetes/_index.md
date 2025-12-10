---
title: "API Deployment on EKS"
date: 2024-01-01T00:00:00Z
weight: 9
chapter: false
pre: "<b>8. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 9:**
{{% /notice %}}
**🎯 Objective of Task 9:**
Deploy the Retail Prediction API (FastAPI) to an EKS cluster, connect the model from S3, and expose a public endpoint via a Load Balancer (ALB).
→ Ensure the service runs reliably, auto-scales, is secure, and can be demoed live.

📥 **Inputs from previous tasks:**

- **Task 5 (Production VPC):** VPC design, subnets, VPC Endpoints and ALB networking required for the cluster and load balancer
- **Task 6 (ECR Container Registry):** Container images and repository URIs to deploy
- **Task 2 (IAM Roles & Audit):** IRSA roles and policies for Pods to access S3 and other AWS services
- **Task 7 (EKS Cluster):** EKS cluster and node groups where manifests will be applied

## 1. Overview

**API Deployment** is the step to deploy the prediction service containerized to Kubernetes (EKS). This ensures the application is deployed as a microservice, can auto-scale, and achieves high availability.

### Deployment architecture

**EKS Deployment Architecture:**

```
Client → ALB → EKS Service → API Pods → S3 Models
                    ↓
            Auto-scaling (HPA)
```

**Components:**

- **Namespace**: `mlops`
- **ServiceAccount**: IRSA service account for SageMaker access
- **Deployment**: API pods using the ECR Singapore image
- **Service**: LoadBalancer service
- **HPA**: Auto-scaling based on CPU

## 2. Kubernetes manifests

**Create the following 5 main files:**

- `namespace.yaml` - Create the `mlops` namespace
- `serviceaccount.yaml` - IRSA service account
- `deployment.yaml` - API deployment configured with the SageMaker registry
- `service.yaml` - LoadBalancer service
- `hpa.yaml` - Auto-scaling

### 2.1 Namespace configuration

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mlops
  labels:
    app.kubernetes.io/name: retail-api
---
```

### 2.2 ServiceAccount with IRSA

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: retail-api-sa
  namespace: mlops
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::842676018087:role/eks-sagemaker-access-role
---
```

### 2.3 Deployment configuration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: mlops
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
      serviceAccountName: retail-api-sa
      containers:
        - name: retail-api
          image: 842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest
          ports:
            - containerPort: 8000
          env:
            - name: PORT
              value: "8000"
            - name: AWS_DEFAULT_REGION
              value: "ap-southeast-1"
            - name: MODEL_PACKAGE_GROUP
              value: "retail-price-sensitivity-models"
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
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
---
```

## 3. Service (Load Balancer)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: mlops
  labels:
    app: retail-api
spec:
  selector:
    app: retail-api
  ports:
    - name: http
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

## 4. Auto-scaling (HPA)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-api-hpa
  namespace: mlops
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

## 5. Deploy to EKS

### 5.1 Apply manifests

```bash
# Deploy all manifests in order
kubectl apply -f namespace.yaml
kubectl apply -f serviceaccount.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```

### 5.2 Check deployment status

```bash
# Check pod status
kubectl get pods -n mlops

# Check service and load balancer
kubectl get svc -n mlops

# Check horizontal pod autoscaler
kubectl get hpa -n mlops

# Check pod logs
kubectl logs -l app=retail-api -n mlops --tail=50
```

### 5.3 Get the LoadBalancer URL and test the API

```bash
# Get the LoadBalancer URL
kubectl get svc retail-api-service -n mlops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Test the health check endpoint
curl http://[LOAD_BALANCER_URL]/health

# Test the API documentation
curl http://[LOAD_BALANCER_URL]/docs

# Test the prediction endpoint with a real data payload
curl -X POST http://[LOAD_BALANCER_URL]/predict \
  -H "Content-Type: application/json" \
  -d '{
    "BASKET_SIZE": "M",
    "BASKET_TYPE": "MIXED",
    "STORE_REGION": "LONDON",
    "STORE_FORMAT": "LS",
    "SPEND": 125.50,
    "QUANTITY": 3,
    "PROD_CODE_20": "FOOD",
    "PROD_CODE_30": "FRESH"
  }'
```

## 6. Check via the AWS Console

### 6.1 EKS Console - Check cluster status

1. **Open the EKS Console:**

```
  AWS Console → EKS → Clusters → mlops-retail-cluster
```

![](/images/08-deploy-kubernetes/01.png)

2. **Check the Resources tab:**

```
  mlops-retail-cluster → Resources → All namespaces → Filter: mlops
```

![](/images/08-deploy-kubernetes/02.png)

### 6.2 EKS workloads - Deployment details

1. **Check the Deployment:**

```
  Resources → Deployments → retail-api
```

![](/images/08-deploy-kubernetes/03.png)

2. **Check the Pods:**

- Click the Deployment → Pods tab
- **Pod status:** Running (if Pending then there is a resource issue)
- **Restart count:** 0 (if > 0 there may have been crashes)

![](/images/08-deploy-kubernetes/04.png)

### 6.3 Debugging when Pods are Pending

1. **If Pods are Pending:**

- Check the Events section for errors:
  - **Insufficient CPU/Memory:** Need to scale nodes
  - **Image pull error:** ECR permissions issue
  - **PodSecurityPolicy:** IAM role issue

2. **If LoadBalancer times out / connection refused:**

- **Target Groups unhealthy:** Pods are not passing the health check (/health endpoint)
- **Security Groups:** EKS worker nodes must allow inbound from the Load Balancer
- **Subnets:** Load Balancer requires at least 2 public subnets

3. **Check Events in the EKS Console:**

```
  Resources → Events → Filter namespace: mlops
```

- Look for Warning/Error events related to the deployment

## 7. Testing and load testing

### 7.1 Local testing with port-forward

```bash
# Port-forward the service to localhost (if the LoadBalancer is not ready)
kubectl port-forward service/retail-api-service 8080:80 -n mlops

# Test via port forward
curl http://localhost:8080/health
```

### 7.2 Test SageMaker Model Registry integration

```bash
# Check the model info endpoint
curl http://[LOAD_BALANCER_URL]/model/info

# Check model metrics from the SageMaker Registry
curl http://[LOAD_BALANCER_URL]/model/metrics

# Expected response: Accuracy 84.7%, F1-Score 83.2% from the Registry
```

### 7.3 Load testing to validate auto-scaling

```bash
# Load test with the correct data format
for i in {1..100}; do
  curl -X POST http://[LOAD_BALANCER_URL]/predict \
    -H "Content-Type: application/json" \
    -d '{"BASKET_SIZE":"M","BASKET_TYPE":"MIXED","STORE_REGION":"LONDON","STORE_FORMAT":"LS","SPEND":125.50,"QUANTITY":3,"PROD_CODE_20":"FOOD","PROD_CODE_30":"FRESH"}' &
done

# Monitor HPA scaling
kubectl get hpa retail-api-hpa -n mlops -w

# Monitor pods scaling up (from 2 → up to 5)
kubectl get pods -n mlops -w
```

## 8. Estimated costs

| Component                   | Estimate            | Notes                             |
| --------------------------- | ------------------- | --------------------------------- |
| EKS Pod (2 replica on Spot) | ~0.012 USD/h        | Compute cost                      |
| ALB/NLB (public)            | ~0.02 USD/h         | Only enabled for the demo         |
| **Total (1h demo)**         | **≈ 0.03–0.04 USD** | Very low if terminated after demo |

{{% notice info %}}
Cost estimates are based on Spot instances (t3.medium) and NLB in the ap-southeast-1 region. Actual costs may vary depending on the configuration and usage time.
{{% /notice %}}

{{% notice success %}}
**🎯 Task 9 Complete - API Deployment on EKS**

- **Kubernetes manifests** ready
- **EKS deployment** configured with IRSA
- **Load Balancer service** for external access
- **Auto-scaling** with HPA
  {{% /notice %}}

## 9. Clean up resources

### 9.1 Delete the deployment and resources

```bash
# Delete all resources in the mlops namespace
kubectl delete namespace mlops

# Or delete individual resources
kubectl delete deployment retail-api -n mlops
kubectl delete service retail-api-service -n mlops
kubectl delete hpa retail-api-hpa -n mlops
kubectl delete serviceaccount retail-api-sa -n mlops

# Check that the LoadBalancer has been removed
aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-mlops`)].LoadBalancerArn'
```

### 9.2 Delete ECR images (optional)

```bash
# List images in the repository
aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1

# Delete a specific image tag
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids imageTag=v3 \
  --region ap-southeast-1

# Delete all images
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --image-ids "$(aws ecr describe-images --repository-name mlops/retail-api --region ap-southeast-1 --query 'imageDetails[].imageDigest' --output text | tr '\t' '\n' | sed 's/.*/imageDigest=&/')" \
  --region ap-southeast-1
```

### 9.3 Verify clean up

```bash
# Ensure no pods remain
kubectl get pods -n mlops

# Ensure no services remain
kubectl get svc -n mlops

# Verify LoadBalancer termination
aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-mlops`)]'
```

## 10. Pricing for Kubernetes deployment (ap-southeast-1)

### 10.1 Pod resource costs

| Resource Type     | Request | Limit | Cost impact          |
| ----------------- | ------- | ----- | -------------------- |
| **CPU**           | 250m    | 500m  | ~25% of node CPU     |
| **Memory**        | 512Mi   | 1Gi   | ~25% of node memory  |
| **Storage (EBS)** | -       | -     | Based on EBS pricing |

**With t2.micro node (1 vCPU, 1GB RAM):**

- 1 API pod uses ~50% of the node resources
- Can run 2 pods with the specified resource requests
- Scaling is limited by node capacity

### 10.2 Load Balancer pricing

| Load Balancer Type | Price (USD/hour) | Price (USD/month) | Data processing  |
| ------------------ | ---------------- | ----------------- | ---------------- |
| **Classic LB**     | $0.025           | $18.25            | $0.008/GB        |
| **Application LB** | $0.0225          | $16.43            | $0.008/LCU-hour  |
| **Network LB**     | $0.0225          | $16.43            | $0.006/NLCU-hour |

### 10.3 Service type costs

| Service Type     | AWS Resource        | Monthly Cost | Use case                   |
| ---------------- | ------------------- | ------------ | -------------------------- |
| **ClusterIP**    | None                | $0           | Internal communication     |
| **NodePort**     | EC2 Security Groups | $0           | Development testing        |
| **LoadBalancer** | ELB/ALB/NLB         | $16.43+      | Production external access |
| **ExternalName** | None                | $0           | External service mapping   |

### 10.4 Auto-scaling costs

**Horizontal Pod Autoscaler (HPA):**

- HPA controller: free (part of EKS)
- Additional pods: EC2 instance costs
- Scaling triggers: CPU/Memory metrics (free)

**Cluster Autoscaler:**

- Controller: free
- New nodes: full EC2 instance pricing
- Scale-down: automatic cost reduction

### 10.5 Task 8 cost estimate

**Basic deployment (2 replicas):**

| Component                | Quantity     | Resource usage    | Monthly cost          |
| ------------------------ | ------------ | ----------------- | --------------------- |
| **API Pods**             | 2 replicas   | 500m CPU, 1Gi RAM | Included in node cost |
| **LoadBalancer Service** | 1 ALB        | Base + LCU usage  | $16.43 + usage        |
| **HPA**                  | 1 autoscaler | Controller only   | $0                    |
| **Ingress**              | Optional     | Same ALB          | $0 additional         |
| **Total**                |              |                   | **~$16.43 + LCU**     |

**With auto-scaling (2-5 replicas):**

| Scenario        | Pods     | Node requirements  | Additional cost |
| --------------- | -------- | ------------------ | --------------- |
| **Low load**    | 2 pods   | 2x t2.micro (free) | $0              |
| **Medium load** | 3-4 pods | 1x t3.small        | $15.18          |
| **High load**   | 5 pods   | 1x t3.medium       | $30.37          |

### 10.6 Data transfer costs

| Transfer type                | Cost     | Use case                 |
| ---------------------------- | -------- | ------------------------ |
| **Pod-to-Pod (same AZ)**     | Free     | Internal communication   |
| **Pod-to-Pod (cross-AZ)**    | $0.01/GB | Multi-AZ deployment      |
| **LoadBalancer to Internet** | $0.12/GB | API responses to clients |
| **VPC Endpoints**            | Free     | S3/ECR access            |

### 10.7 Storage costs for persistent volumes

```yaml
# Example PVC for model storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: mlops
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3
```

**Storage pricing:**

- 10GB gp3: $0.80/month
- Snapshots: $0.50/month (10GB)
- IOPS (if > 3000): $0.065/IOPS/month

### 10.8 Cost optimization for deployments

**Resource right-sizing:**

```yaml
resources:
  requests:
    memory: "256Mi" # Start smaller
    cpu: "100m" # Minimal CPU request
  limits:
    memory: "512Mi" # Reasonable limit
    cpu: "250m" # Allow bursting
```

**Efficient pod scheduling:**

```yaml
# Node affinity for cost optimization
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: kubernetes.io/instance-type
              operator: In
              values: ["t3.micro", "t3.small"] # Prefer cheaper instances
```

**LoadBalancer optimization:**

```yaml
# Use a single ALB for multiple services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shared-alb
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
    - http:
        paths:
          - path: /api/v1/*
            backend:
              service:
                name: retail-api-service
                port:
                  number: 80
          - path: /admin/*
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

### 10.9 Monitoring costs

```bash
# Monitor pod resource usage
kubectl top pods -n mlops

# Check actual vs requested resources
kubectl describe pod <pod-name> -n mlops

# Monitor HPA behavior
kubectl get hpa -w -n mlops

# Check LoadBalancer usage
aws elbv2 describe-load-balancers --names <alb-name>
```

**Cost tracking commands:**

```bash
# ELB costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Load Balancing"]}}'

# EC2 costs for nodes
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=INSTANCE_TYPE
```

{{% notice info %}}
**💰 Cost summary for Task 8:**

- **Pods:** Included in node cost (no additional charge)
- **LoadBalancer:** $16.43/month base + usage
- **Auto-scaling:** $0-30.37/month depending on load
- **Storage:** $0.80/month per 10GB PVC
- **Total:** $17-47/month depending on scaling
  {{% /notice %}}

## 🎬 Video for Task 8

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/0BgK5B4XBU4" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 09: Elastic Load Balancing](../9-elastic-load-balancing/)
