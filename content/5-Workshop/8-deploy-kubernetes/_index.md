---
title: "API Deployment on EKS (Retail Prediction API)"
date: 2024-01-01T00:00:00Z
weight: 9
chapter: false
pre: "<b>8. </b>"
---

{{% notice info %}}
**🎯 Task 8 Goal:** Deploy the FastAPI Retail Prediction API to EKS using Kubernetes manifests, connect it to SageMaker Model Registry via IRSA, expose a demo endpoint, and enable autoscaling.
{{% /notice %}}

## 0) Inputs

- Cluster: `mlops-retail-cluster` (ap-southeast-1)
- ECR image: `842676018087.dkr.ecr.ap-southeast-1.amazonaws.com/mlops/retail-api:latest`
- Namespace: `mlops`
- IRSA role ARN: `arn:aws:iam::842676018087:role/eks-sagemaker-access-role`
- Model Package Group: `retail-price-sensitivity-models`
- API endpoints:
  - `GET /health`
  - `POST /predict`
  - Optional:
    - `GET /model/info`
    - `GET /model/metrics`
    - `GET /docs` (FastAPI Swagger UI)

---

## 1) Kubernetes manifests

Create `aws/k8s/retail-api.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mlops
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: retail-api-sa
  namespace: mlops
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::842676018087:role/eks-sagemaker-access-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: mlops
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
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: AWS_DEFAULT_REGION
              value: ap-southeast-1
            - name: MODEL_PACKAGE_GROUP
              value: retail-price-sensitivity-models
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 6
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 25
            periodSeconds: 20
            timeoutSeconds: 2
            failureThreshold: 3
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: retail-api-svc
  namespace: mlops
spec:
  selector:
    app: retail-api
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 8000
```

Create `aws/k8s/retail-api-hpa.yaml`:

```yaml
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

---

## 2) Deploy

```bash
aws eks update-kubeconfig --name mlops-retail-cluster --region ap-southeast-1

kubectl apply -f aws/k8s/retail-api.yaml
kubectl apply -f aws/k8s/retail-api-hpa.yaml

kubectl get all -n mlops
kubectl get hpa -n mlops
```

---

## 3) Test the API

### 3.1 Get LoadBalancer URL

```bash
kubectl get svc -n mlops retail-api-svc -o wide
LB_HOST=$(kubectl get svc -n mlops retail-api-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://$LB_HOST"
```

### 3.2 Smoke tests

```bash
curl -s "http://$LB_HOST/health"
curl -s "http://$LB_HOST/docs" | head
curl -s -X POST "http://$LB_HOST/predict"   -H "Content-Type: application/json"   -d '{"features":{"store_id":1,"product_id":123,"basket_total":52.35}}'
```

---

## 4) Debugging cheatsheet

### Pods stuck in Pending

- NodeGroup has no capacity / wrong instance type
- Subnets / security groups not allowing scheduling
- Missing CNI permissions

```bash
kubectl describe pod -n mlops -l app=retail-api
kubectl get events -n mlops --sort-by=.lastTimestamp | tail -n 30
```

### ImagePullBackOff

- ECR endpoints missing (private-only VPC)
- Node IAM role missing ECR permissions

```bash
kubectl describe pod -n mlops -l app=retail-api | sed -n '1,200p'
```

### LoadBalancer not provisioned

- Public subnets missing proper tags for LB
- Security groups / route tables wrong
- Using “demo-only ALB” policy but service is always-on

```bash
kubectl describe svc -n mlops retail-api-svc
```

---

## 5) Cleanup

```bash
kubectl delete namespace mlops --ignore-not-found=true
```

> Ensure the AWS Load Balancer is deleted (console) to stop hourly charges.

{{% notice success %}}
**✅ Task 8 Complete (EKS API):**

- Retail API deployed (2 replicas) with liveness/readiness probes
- Service exposed via LoadBalancer (demo endpoint)
- HPA enabled (min 2, max 5, CPU 60%)
- IRSA service account attached for SageMaker access
  {{% /notice %}}
