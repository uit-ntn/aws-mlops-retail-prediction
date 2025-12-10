---
title: "Load Balancing for Retail Prediction API on EKS"
date: 2024-01-01T00:00:00Z
weight: 10
chapter: false
pre: "<b>9. </b>"
---

{{% notice info %}}
**🎯 Task 9 Goal:** Provide a secure, highly available public demo endpoint (`/predict`, `/docs`) for the Retail API, distribute traffic across pods with health checks (`/health`), and document options for ALB/NLB with cleanup and cost notes.
{{% /notice %}}

## 0) Context

You have two common patterns:

1. **Service type LoadBalancer** (simple, fast demo)
2. **AWS Load Balancer Controller + Ingress (ALB)** (best practice for HTTP/HTTPS routing)

If you only need a temporary demo endpoint, start with (1). If you need routing, TLS, WAF, access logs, use (2).

---

## 1) Option A — Service type `LoadBalancer` (simple demo)

This is what Task 8 already uses:

- Kubernetes creates a cloud Load Balancer
- Traffic spreads across pods behind the Service
- Health checks rely on pod readiness + node health

**Pros:** easy, minimal setup  
**Cons:** less control (path routing, TLS termination, WAF) compared to ALB Ingress

### Verification

```bash
kubectl get svc -n mlops retail-api-svc -o wide
kubectl get endpoints -n mlops retail-api-svc
```

### Health check

```bash
LB_HOST=$(kubectl get svc -n mlops retail-api-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -f "http://$LB_HOST/health"
```

---

## 2) Option B — AWS Load Balancer Controller + Ingress (recommended ALB)

### 2.1 Install AWS Load Balancer Controller (Helm)

Prereqs:

- EKS OIDC provider enabled
- A controller IAM policy/role for the service account
- Public subnets tagged for ALB

Install (high-level):

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

kubectl create namespace kube-system --dry-run=client -o yaml | kubectl apply -f -

# After creating the controller IAM role:
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=mlops-retail-cluster   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller
```

> In production docs, the controller IAM policy is long; keep it in a separate file and attach it to the IRSA role.

### 2.2 Create Ingress

Create `aws/k8s/retail-api-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-api-ingress
  namespace: mlops
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: retail-api-svc
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f aws/k8s/retail-api-ingress.yaml
kubectl get ingress -n mlops
```

### 2.3 TLS (HTTPS) with ACM (optional)

- Request an ACM certificate (same region)
- Update ingress:
  - `alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'`
  - `alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN>`
- Optionally redirect HTTP → HTTPS

### 2.4 DNS with Route53 (optional)

- Create an A/AAAA alias record to the ALB DNS name.

### 2.5 WAF (optional)

- Attach AWS WAF web ACL to the ALB for rate limiting and OWASP protections.

### 2.6 Access logs (optional)

Enable ALB access logs to an S3 bucket for auditing.

---

## 3) Load testing (demo)

### ApacheBench (ab)

```bash
ab -n 2000 -c 20 "http://$LB_HOST/health"
```

### Artillery

```bash
npm i -g artillery
artillery quick --count 50 --num 20 "http://$LB_HOST/health"
```

---

## 4) Troubleshooting

- Controller logs:
  ```bash
  kubectl logs -n kube-system deploy/aws-load-balancer-controller | tail -n 200
  ```
- Target group unhealthy:
  - check pod readiness
  - confirm `/health` returns HTTP 200
  - confirm security group rules
- Ingress stuck without address:
  - missing public subnet tags for ALB
  - missing IAM permissions for controller

---

## 5) Cleanup (very important to save cost)

### Option A cleanup

```bash
kubectl delete svc -n mlops retail-api-svc
```

### Option B cleanup

```bash
kubectl delete ingress -n mlops retail-api-ingress
helm uninstall aws-load-balancer-controller -n kube-system
```

Then verify in AWS console that:

- ALB is deleted
- Target groups are deleted
- Security groups created by the ALB are deleted (if any remain)

---

## 6) Cost notes

- **ALB** charges hourly + LCU (depends on traffic).
- **NLB** charges hourly + NLCU (often better for pure TCP, not needed for FastAPI HTTP).
- For this workshop: keep the ALB **enabled only during demos**.

{{% notice success %}}
**✅ Task 9 Complete (Load Balancing):**

- Documented two deployment patterns: Service LB and ALB Ingress
- Included health checks, TLS/DNS/WAF options, load testing, troubleshooting, and cleanup
  {{% /notice %}}
