---
title: "CloudWatch Monitoring & Log Insights"
date: 2024-01-01T00:00:00Z
weight: 11
chapter: false
pre: "<b>10. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 12:**

Thiết lập hệ thống giám sát toàn diện cho pipeline MLOps Retail Prediction gồm:  
- Theo dõi hiệu năng API (latency, error rate, throughput)  
- Giám sát tài nguyên (CPU, RAM, network) của EKS  
- Lưu log model inference và training vào CloudWatch  
→ Giúp phát hiện sớm lỗi, đảm bảo SLA, và đánh giá hiệu quả mô hình sau triển khai.
{{% /notice %}}

📥 **Input từ các Task trước:**
- **Task 7 (EKS Cluster):** Cluster, namespaces và pods cần metrics/logs collection
- **Task 9 (API Deployment on EKS):** Deployed services and endpoints (`/health`, `/predict`) to instrument for logs and metrics
- **Task 2 (IAM Roles & Audit):** IAM roles/policies required for CloudWatch Agents and log delivery

## Kiến trúc tổng quan

Dưới đây là kiến trúc tổng thể của hệ thống monitoring cho MLOps Retail Prediction:

{{< mermaid >}}
flowchart TD
    subgraph "AWS Cloud"
        subgraph "EKS Cluster"
            CWA[CloudWatch Agent]
            POD1[Retail API Pod]
            POD2[Retail API Pod]
            POD3[Retail API Pod]
            CWA --- POD1
            CWA --- POD2
            CWA --- POD3
        end
        
        subgraph "SageMaker"
            TRAIN[Training Job]
        end
        
        subgraph "CloudWatch"
            LOGS[(CloudWatch Logs)]
            METRICS[(CloudWatch Metrics)]
            ALARMS[CloudWatch Alarms]
            DASH[CloudWatch Dashboard]
        end
        
        subgraph "Auto Remediation"
            SNS[SNS Topic]
            LAMBDA[Lambda Function]
            ASG[Auto Scaling Group]
        end
    end
    
    subgraph "Alerts"
        EMAIL[Email]
        SLACK[Slack]
    end
    
    subgraph "User Access"
        CONSOLE[AWS Console]
        CLI[AWS CLI]
    end
    
    POD1 --> |Logs| LOGS
    POD2 --> |Logs| LOGS
    POD3 --> |Logs| LOGS
    TRAIN --> |Logs| LOGS
    
    CWA --> |System Metrics| METRICS
    LOGS --> |Log Metrics| METRICS
    
    METRICS --> DASH
    METRICS --> ALARMS
    
    ALARMS --> SNS
    SNS --> LAMBDA
    LAMBDA --> ASG
    SNS --> EMAIL
    SNS --> SLACK
    
    DASH --> CONSOLE
    LOGS --> CLI
    
    style CWA fill:#f9f,stroke:#333,stroke-width:2px
    style METRICS fill:#bbf,stroke:#333,stroke-width:2px
    style DASH fill:#bfb,stroke:#333,stroke-width:2px
    style ALARMS fill:#fbb,stroke:#333,stroke-width:2px
    style LAMBDA fill:#fbf,stroke:#333,stroke-width:2px
{{< /mermaid >}}

Kiến trúc này bao gồm 3 thành phần chính:
1. **Data Collection**: CloudWatch Agent thu thập metrics từ EKS cluster, trong khi application logs được gửi đến CloudWatch Logs
2. **Processing & Storage**: CloudWatch xử lý và lưu trữ logs và metrics
3. **Monitoring & Alerting**: Dashboard cho visualization và Alarms cho alerting

## Nội dung chính

## 1. Tích hợp CloudWatch với EKS

### 1.1 Cài đặt CloudWatch Agent bằng Helm

CloudWatch Agent là thành phần quan trọng giúp thu thập metrics và logs từ EKS cluster và gửi về CloudWatch:

```bash
# Tạo namespace cho CloudWatch Agent
kubectl create namespace amazon-cloudwatch

# Cài đặt CloudWatch Agent bằng Helm
helm install cloudwatch-agent \
  --namespace amazon-cloudwatch \
  --set clusterName=retail-prediction-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=cloudwatch-agent \
  --set cloudWatchLogs.enabled=true \
  oci://public.ecr.aws/cloudwatch-agent/chart
```

### 1.2 Tạo CloudWatch Log Groups

```bash
# Tạo Log Group cho EKS cluster logs
aws logs create-log-group \
  --log-group-name /aws/eks/retail-prediction-cluster/cluster \
  --retention-in-days 30

# Tạo Log Group cho API application logs
aws logs create-log-group \
  --log-group-name /aws/container/retail-api \
  --retention-in-days 30

# Tạo Log Group cho load balancer logs
aws logs create-log-group \
  --log-group-name /aws/applicationloadbalancer/retail-prediction \
  --retention-in-days 30
```

#### 1.2 Configure Fluent Bit for Log Collection

```yaml
# fluent-bit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: amazon-cloudwatch
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    @INCLUDE application-log.conf
    @INCLUDE dataplane-log.conf
    @INCLUDE host-log.conf

  application-log.conf: |
    [INPUT]
        Name              tail
        Tag               application.*
        Path              /var/log/containers/retail-forecast-api*.log
        Parser            docker
        DB                /var/fluent-bit/state/flb_container.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               application.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     application.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name                cloudwatch_logs
        Match               application.*
        region              us-east-1
        log_group_name      /aws/eks/retail-forecast/application
        log_stream_prefix   ${HOSTNAME}-
        auto_create_group   true

  parsers.conf: |
    [PARSER]
        Name   apache
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   apache2
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   apache_error
        Format regex
        Regex  ^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\](?: \[pid (?<pid>[^\]]*)\])?( \[client (?<client>[^\]]*)\])? (?<message>.*)$

    [PARSER]
        Name   nginx
        Format regex
        Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
---
```

#### 1.3 Deploy Fluent Bit DaemonSet

```yaml
# fluent-bit-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
  labels:
    k8s-app: fluent-bit
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit
  template:
    metadata:
      labels:
        k8s-app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: public.ecr.aws/aws-observability/aws-for-fluent-bit:stable
        imagePullPolicy: Always
        env:
        - name: AWS_REGION
          valueFrom:
            configMapKeyRef:
              name: fluent-bit-cluster-info
              key: logs.region
        - name: CLUSTER_NAME
          valueFrom:
            configMapKeyRef:
              name: fluent-bit-cluster-info
              key: cluster.name
        - name: HTTP_SERVER
          value: "On"
        - name: HTTP_PORT
          value: "2020"
        - name: READ_FROM_HEAD
          value: "Off"
        - name: READ_FROM_TAIL
          value: "On"
        - name: HOST_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: CI_VERSION
          value: "k8s/1.3.17"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 500m
            memory: 100Mi
        volumeMounts:
        - name: fluentbitstate
          mountPath: /var/fluent-bit/state
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        - name: runlogjournal
          mountPath: /run/log/journal
          readOnly: true
        - name: dmesg
          mountPath: /var/log/dmesg
          readOnly: true
      terminationGracePeriodSeconds: 10
      hostNetwork: false
      dnsPolicy: ClusterFirst
      volumes:
      - name: fluentbitstate
        hostPath:
          path: /var/fluent-bit/state
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: runlogjournal
        hostPath:
          path: /run/log/journal
      - name: dmesg
        hostPath:
          path: /var/log/dmesg
---
```

## 2. Container Insights Setup

Container Insights cung cấp visibility vào hiệu năng của các ứng dụng containerized và toàn bộ EKS cluster.

### 2.1 Kích hoạt Container Insights

```bash
# Kích hoạt Container Insights cho EKS cluster
aws eks update-cluster-config \
  --region us-east-1 \
  --name retail-prediction-cluster \
  --logging '{"enable":["api","audit","authenticator","controllerManager","scheduler"]}'

# Kiểm tra trạng thái của Container Insights
aws eks describe-cluster \
  --name retail-prediction-cluster \
  --query "cluster.logging" \
  --output json
```

### 2.2 Metrics thu thập từ Container Insights

CloudWatch Agent tự động thu thập các metrics sau từ EKS cluster:

{{< mermaid >}}
graph TD
    subgraph "System Metrics"
        CPU[CPUUtilization]
        MEM[MemoryUtilization]
        DISK[DiskUsage]
        NET[NetworkIn/Out]
    end
    
    subgraph "Pod Metrics"
        PCPU[pod_cpu_utilization]
        PMEM[pod_memory_utilization]
        PNET[pod_network_rx/tx_bytes]
        PRESTART[pod_restart_count]
    end
    
    subgraph "Node Metrics"
        NCPU[node_cpu_utilization]
        NMEM[node_memory_utilization]
        NDISK[node_filesystem_utilization]
    end
    
    subgraph "Cluster Metrics"
        CLUSTER[cluster_node_count]
        FAILED[cluster_failed_node_count]
    end
    
    CloudWatch[CloudWatch Metrics]
    
    CPU --> CloudWatch
    MEM --> CloudWatch
    DISK --> CloudWatch
    NET --> CloudWatch
    PCPU --> CloudWatch
    PMEM --> CloudWatch
    PNET --> CloudWatch
    PRESTART --> CloudWatch
    NCPU --> CloudWatch
    NMEM --> CloudWatch
    NDISK --> CloudWatch
    CLUSTER --> CloudWatch
    FAILED --> CloudWatch
{{< /mermaid >}}

#### 2.2 Custom CloudWatch Agent Configuration

```yaml
# cloudwatch-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cwagentconfig
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "logs": {
        "metrics_collected": {
          "kubernetes": {
            "cluster_name": "retail-forecast-cluster",
            "metrics_collection_interval": 60
          }
        },
        "force_flush_interval": 15
      },
      "metrics": {
        "namespace": "CWAgent",
        "metrics_collected": {
          "cpu": {
            "measurement": [
              "cpu_usage_idle",
              "cpu_usage_iowait",
              "cpu_usage_user",
              "cpu_usage_system"
            ],
            "metrics_collection_interval": 60
          },
          "disk": {
            "measurement": [
              "used_percent"
            ],
            "metrics_collection_interval": 60,
            "resources": [
              "*"
            ]
          },
          "diskio": {
            "measurement": [
              "io_time",
              "read_bytes",
              "write_bytes",
              "reads",
              "writes"
            ],
            "metrics_collection_interval": 60,
            "resources": [
              "*"
            ]
          },
          "mem": {
            "measurement": [
              "mem_used_percent"
            ],
            "metrics_collection_interval": 60
          },
          "netstat": {
            "measurement": [
              "tcp_established",
              "tcp_time_wait"
            ],
            "metrics_collection_interval": 60
          },
          "swap": {
            "measurement": [
              "swap_used_percent"
            ],
            "metrics_collection_interval": 60
          }
        }
      }
    }
---
```

## 3. Giám sát API Metrics

### 3.1 Cấu hình API Logger

Hãy tích hợp logging vào FastAPI application để ghi lại thông tin về requests và model inference:

```python
# logging_config.py
import logging
import json
import time
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

# Cấu hình logger
logging.basicConfig(
    format='%(asctime)s [%(levelname)s] %(message)s',
    level=logging.INFO
)
logger = logging.getLogger("retail-api")

class APILoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        
        # Xử lý request
        try:
            response = await call_next(request)
            status_code = response.status_code
            
            # Tính thời gian xử lý
            process_time = time.time() - start_time
            
            # Log thông tin
            log_dict = {
                "endpoint": request.url.path,
                "method": request.method,
                "status_code": status_code,
                "latency_ms": round(process_time * 1000, 2)
            }
            
            if status_code >= 500:
                logger.error(f"API Error: {json.dumps(log_dict)}")
            else:
                logger.info(f"API Request: {json.dumps(log_dict)}")
                
            return response
        except Exception as e:
            process_time = time.time() - start_time
            logger.error(f"Unhandled exception: {str(e)}, latency={round(process_time * 1000, 2)}ms")
            return JSONResponse(content={"error": "Internal server error"}, status_code=500)

# Wrapper cho model inference
def log_inference(func):
    def wrapper(*args, **kwargs):
        start_time = time.time()
        try:
            result = func(*args, **kwargs)
            inference_time = time.time() - start_time
            logger.info(f"Inference completed: latency={round(inference_time * 1000, 2)}ms, prediction={result}")
            return result
        except Exception as e:
            inference_time = time.time() - start_time
            logger.error(f"Inference failed: latency={round(inference_time * 1000, 2)}ms, error={str(e)}")
            raise
    return wrapper

# Áp dụng vào FastAPI app
app = FastAPI(title="Retail Prediction API")
app.add_middleware(APILoggingMiddleware)

# Áp dụng logger vào prediction endpoint
@app.post("/predict")
async def predict(data: dict):
    @log_inference
    def run_inference(input_data):
        # Giả lập inference
        time.sleep(0.05)  # 50ms latency
        return "High" if input_data["value"] > 50 else "Low"
    
    return {"prediction": run_inference(data)}
```

### 3.2 CloudWatch Metric Filters

Tạo metric filters để trích xuất thông tin từ logs:

```bash
# Tạo metric filter cho inference latency
aws logs put-metric-filter \
  --log-group-name "/aws/container/retail-api" \
  --filter-name "InferenceLatency" \
  --filter-pattern "\"Inference completed: latency=\"" \
  --metric-transformations \
      metricName=InferenceLatency,metricNamespace=RetailMLOps,metricValue="$.latency",unit=Milliseconds

# Tạo metric filter cho error rate
aws logs put-metric-filter \
  --log-group-name "/aws/container/retail-api" \
  --filter-name "ErrorRate" \
  --filter-pattern "\"API Error\"" \
  --metric-transformations \
      metricName=APIErrorCount,metricNamespace=RetailMLOps,metricValue=1,unit=Count

# Tạo metric filter cho API request count
aws logs put-metric-filter \
  --log-group-name "/aws/container/retail-api" \
  --filter-name "RequestCount" \
  --filter-pattern "\"API Request\"" \
  --metric-transformations \
      metricName=APIRequestCount,metricNamespace=RetailMLOps,metricValue=1,unit=Count
```

#### 3.2 Prometheus ServiceMonitor

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: retail-forecast-metrics
  namespace: mlops
  labels:
    app: retail-forecast-api
spec:
  selector:
    matchLabels:
      app: retail-forecast-api
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scrapeTimeout: 10s
---
apiVersion: v1
kind: Service
metadata:
  name: retail-forecast-metrics-service
  namespace: mlops
  labels:
    app: retail-forecast-api
spec:
  selector:
    app: retail-forecast-api
  ports:
  - name: metrics
    port: 8000
    targetPort: 8000
  - name: http
    port: 80
    targetPort: 8080
---
```

## 5. Cảnh báo (Alarms & SNS)

### 5.1 Tạo SNS Topic cho Cảnh báo

```bash
# Tạo SNS topic
aws sns create-topic \
  --name retail-prediction-alerts \
  --attributes DisplayName="Retail Prediction Alerts"

# Lưu ARN của topic cho việc sử dụng sau này
TOPIC_ARN=$(aws sns create-topic \
  --name retail-prediction-alerts \
  --query 'TopicArn' \
  --output text)
  
# Đăng ký email nhận cảnh báo
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com
```

### 5.2 Thiết lập CloudWatch Alarms

Theo yêu cầu của dự án, chúng ta cần thiết lập 3 cảnh báo chính:

```bash
# 1. API Latency Alarm - Cảnh báo khi độ trễ API P95 > 200ms
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailPrediction-HighLatency" \
  --alarm-description "API Latency P95 exceeds 200ms" \
  --metric-name "InferenceLatency" \
  --namespace "RetailMLOps" \
  --extended-statistic "p95" \
  --period 60 \
  --threshold 200 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 3 \
  --alarm-actions $TOPIC_ARN \
  --treat-missing-data "notBreaching"

# 2. Error Rate Alarm - Cảnh báo và kích hoạt rollback khi tỉ lệ lỗi > 5%
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailPrediction-HighErrorRate" \
  --alarm-description "API Error Rate exceeds 5%" \
  --metric-name "ErrorRate" \
  --namespace "RetailMLOps" \
  --statistic "Average" \
  --period 60 \
  --threshold 5 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions $TOPIC_ARN \
  --ok-actions $TOPIC_ARN \
  --treat-missing-data "notBreaching"

# 3. High CPU Utilization - Cảnh báo khi CPU > 85% để scale nodegroup
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailPrediction-HighCPU" \
  --alarm-description "EKS Node CPU Utilization exceeds 85%" \
  --metric-name "node_cpu_utilization" \
  --namespace "ContainerInsights" \
  --statistic "Average" \
  --period 300 \
  --threshold 85 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --dimensions Name=ClusterName,Value=retail-prediction-cluster \
  --alarm-actions $TOPIC_ARN \
  --ok-actions $TOPIC_ARN
```

### 5.3 Kích hoạt Auto-Scaling từ CloudWatch Alarm

```bash
# Tạo IAM role cho Lambda function
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name RetailPredictionAutoScaleRole \
  --assume-role-policy-document file://trust-policy.json

# Tạo policy cho phép Lambda thao tác với EKS
cat > autoscale-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:DescribeAutoScalingGroups"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name RetailPredictionAutoScaleRole \
  --policy-name RetailPredictionAutoScalePolicy \
  --policy-document file://autoscale-policy.json

# Tạo Lambda function
cat > auto-scale-function.py << EOF
import json
import boto3
import os

def lambda_handler(event, context):
    print("Received event:", json.dumps(event))
    
    # Phân tích SNS message
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    new_state = message['NewStateValue']
    
    cluster_name = os.environ['CLUSTER_NAME']
    asg_name = os.environ['ASG_NAME']
    
    autoscaling = boto3.client('autoscaling')
    
    # Chỉ xử lý nếu alarm là HighCPU và trạng thái là ALARM
    if alarm_name == 'RetailPrediction-HighCPU' and new_state == 'ALARM':
        print(f"Scaling up ASG {asg_name} due to high CPU utilization")
        
        # Lấy thông tin hiện tại của ASG
        response = autoscaling.describe_auto_scaling_groups(
            AutoScalingGroupNames=[asg_name]
        )
        
        if not response['AutoScalingGroups']:
            print(f"ASG {asg_name} not found")
            return {'statusCode': 404, 'body': 'ASG not found'}
        
        current_capacity = response['AutoScalingGroups'][0]['DesiredCapacity']
        max_capacity = response['AutoScalingGroups'][0]['MaxSize']
        
        # Tăng desired capacity lên 1 nếu chưa đạt max
        if current_capacity < max_capacity:
            new_capacity = current_capacity + 1
            autoscaling.set_desired_capacity(
                AutoScalingGroupName=asg_name,
                DesiredCapacity=new_capacity,
                HonorCooldown=True
            )
            print(f"Increased desired capacity from {current_capacity} to {new_capacity}")
            return {'statusCode': 200, 'body': f'Scaled up to {new_capacity} nodes'}
        else:
            print(f"Already at maximum capacity: {max_capacity}")
            return {'statusCode': 200, 'body': 'Already at maximum capacity'}
    
    return {'statusCode': 200, 'body': 'No scaling action needed'}
EOF

# Zip file để upload lên Lambda
zip auto-scale-function.zip auto-scale-function.py

# Tạo Lambda function
aws lambda create-function \
  --function-name RetailPredictionAutoScale \
  --zip-file fileb://auto-scale-function.zip \
  --handler auto-scale-function.lambda_handler \
  --runtime python3.9 \
  --role arn:aws:iam::$(aws sts get-caller-identity --query "Account" --output text):role/RetailPredictionAutoScaleRole \
  --environment "Variables={CLUSTER_NAME=retail-prediction-cluster,ASG_NAME=retail-prediction-nodegroup-asg}"

# Cho phép SNS gọi Lambda function
aws lambda add-permission \
  --function-name RetailPredictionAutoScale \
  --statement-id sns-invoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn $TOPIC_ARN

# Subscribe Lambda function vào SNS topic
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol lambda \
  --notification-endpoint $(aws lambda get-function --function-name RetailPredictionAutoScale --query "Configuration.FunctionArn" --output text)
```

{{< mermaid >}}
sequenceDiagram
    participant CloudWatch
    participant SNS as SNS Topic
    participant Lambda as AutoScale Lambda
    participant ASG as EKS NodeGroup ASG
    participant EKS as EKS Cluster
    
    CloudWatch->>SNS: Trigger ALARM (CPU > 85%)
    SNS->>Lambda: Notify about alarm
    Lambda->>ASG: Get current capacity
    ASG->>Lambda: Return current=2, max=5
    Lambda->>ASG: Set desired capacity to 3
    ASG->>EKS: Launch new node
    EKS->>CloudWatch: Report metrics
    CloudWatch->>SNS: Trigger OK (CPU < 85%)
    Note over Lambda,ASG: Scale down happens via HPA, not Lambda
{{< /mermaid >}}

## 6. Log Training (SageMaker Integration)

### 6.1 CloudWatch Integration với SageMaker

Amazon SageMaker tự động gửi logs và metrics của các training jobs tới CloudWatch. Thiết lập này đảm bảo quá trình huấn luyện mô hình được theo dõi và lưu trữ:

```bash
# Tạo SageMaker Training Job với logging configuration
aws sagemaker create-training-job \
  --training-job-name retail-prediction-training \
  --algorithm-specification TrainingImage=<your-ecr-image> \
  --role-arn arn:aws:iam::<account-id>:role/SageMakerExecutionRole \
  --input-data-config file://training-data-config.json \
  --output-data-config S3OutputPath=s3://retail-prediction-bucket/model-artifacts/ \
  --resource-config file://resource-config.json \
  --stopping-condition MaxRuntimeInSeconds=3600 \
  --hyper-parameters file://hyperparameters.json \
  --tags Key=Project,Value=RetailPrediction
```

Mẫu file `resource-config.json`:

```json
{
  "InstanceType": "ml.m5.xlarge",
  "InstanceCount": 1,
  "VolumeSizeInGB": 50
}
```

Logs từ SageMaker training job sẽ được gửi đến CloudWatch Log Group:

```
/aws/sagemaker/TrainingJobs/retail-prediction-training
```

### 6.2 Log Analysis với CloudWatch Insights

```bash
# Trích xuất loss values từ training logs
aws logs start-query \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --start-time $(date -d '24 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @message like "loss" 
    | parse @message "loss: *," as loss_value 
    | parse @message "epoch * " as epoch_num 
    | stats avg(loss_value) as avg_loss by bin(1h)'

# Trích xuất accuracy values từ evaluation logs  
aws logs start-query \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --start-time $(date -d '24 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @message like "accuracy" 
    | parse @message "accuracy: *," as accuracy_value 
    | parse @message "epoch * " as epoch_num 
    | stats avg(accuracy_value) as avg_accuracy by bin(1h)'
```

### 6.3 Metric Filters cho Training Metrics

```bash
# Tạo metric filter cho training loss
aws logs put-metric-filter \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --filter-name "TrainingLoss" \
  --filter-pattern "\"loss: \"" \
  --metric-transformations \
      metricName=TrainingLoss,metricNamespace=RetailMLOps,metricValue="$loss_value",unit=None

# Tạo metric filter cho validation accuracy
aws logs put-metric-filter \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --filter-name "ValidationAccuracy" \
  --filter-pattern "\"validation accuracy: \"" \
  --metric-transformations \
      metricName=ValidationAccuracy,metricNamespace=RetailMLOps,metricValue="$accuracy_value",unit=Percent

# Tạo metric filter cho F1 score
aws logs put-metric-filter \
  --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training" \
  --filter-name "F1Score" \
  --filter-pattern "\"f1 score: \"" \
  --metric-transformations \
      metricName=F1Score,metricNamespace=RetailMLOps,metricValue="$f1_score",unit=None
```

### 6.4 Biểu đồ Training Metrics

{{< mermaid >}}
graph TB
    subgraph "SageMaker Training"
        TRAIN[Train Model Script]
        LOG[Write Logs]
    end
    
    subgraph "CloudWatch"
        CWLOGS[CloudWatch Logs]
        METRIC[CloudWatch Metrics]
        ALARM[CloudWatch Alarms]
        DASH[CloudWatch Dashboard]
    end
    
    subgraph "Notification"
        SNS[SNS Topic]
        EMAIL[Email Notification]
        SLACK[Slack Notification]
    end
    
    TRAIN --> LOG
    LOG --> CWLOGS
    CWLOGS --> METRIC
    METRIC --> DASH
    METRIC --> ALARM
    ALARM --> SNS
    SNS --> EMAIL
    SNS --> SLACK
    
    class ALARM,SNS highlight
{{< /mermaid >}}

### 6.5 SageMaker Training Dashboard

```bash
# Tạo training metrics dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "Retail-Prediction-Training" \
  --dashboard-body file://training-dashboard.json
```

Nội dung file `training-dashboard.json`:

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "RetailMLOps", "TrainingLoss" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Training Loss",
        "period": 300,
        "stat": "Average"
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "RetailMLOps", "ValidationAccuracy" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Validation Accuracy",
        "period": 300,
        "stat": "Average",
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "RetailMLOps", "F1Score" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "F1 Score",
        "period": 300,
        "stat": "Average",
        "yAxis": {
          "left": {
            "min": 0,
            "max": 1
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/SageMaker", "CPUUtilization", "TrainingJobName", "retail-prediction-training" ],
          [ ".", "MemoryUtilization", ".", "." ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Training Resource Utilization",
        "period": 60,
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100
          }
        }
      }
    }
  ]
}
```

## 4. Dashboard Trực Quan

### 4.1 Tạo CloudWatch Dashboard "Retail Prediction KPI"

Dashboard này sẽ hiển thị 6 chỉ số KPI quan trọng theo yêu cầu của dự án:

```bash
# Tạo CloudWatch Dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "Retail-Prediction-KPI" \
  --dashboard-body file://retail-prediction-dashboard.json
```

Nội dung file `retail-prediction-dashboard.json`:

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ { "expression": "PERCENTILE(m1, 50)", "label": "P50", "id": "p50", "region": "us-east-1" } ],
          [ { "expression": "PERCENTILE(m1, 95)", "label": "P95", "id": "p95", "region": "us-east-1" } ],
          [ "RetailMLOps", "InferenceLatency", { "id": "m1", "visible": false } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "API Latency (P50/P95)",
        "period": 60,
        "stat": "Average",
        "yAxis": {
          "left": {
            "label": "Milliseconds",
            "showUnits": false
          }
        },
        "annotations": {
          "horizontal": [
            {
              "label": "SLA Threshold",
              "value": 200,
              "fill": "above"
            }
          ]
        }
      }
    },
    {
      "type": "metric",
      "x": 8,
      "y": 0,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ { "expression": "100 * m2 / (m1 + m2)", "label": "Error Rate (%)", "id": "e1" } ],
          [ "RetailMLOps", "APIErrorCount", { "id": "m2", "visible": false } ],
          [ "RetailMLOps", "APIRequestCount", { "id": "m1", "visible": false } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Error Rate (%)",
        "period": 300,
        "annotations": {
          "horizontal": [
            {
              "label": "Critical",
              "value": 5,
              "fill": "above"
            }
          ]
        },
        "yAxis": {
          "left": {
            "label": "Percent",
            "showUnits": false,
            "min": 0,
            "max": 10
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 16,
      "y": 0,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "ContainerInsights", "node_cpu_utilization", "ClusterName", "retail-prediction-cluster" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "CPU Utilization",
        "period": 60,
        "annotations": {
          "horizontal": [
            {
              "label": "Scale Up",
              "value": 85,
              "fill": "above"
            }
          ]
        },
        "yAxis": {
          "left": {
            "label": "Percent",
            "showUnits": false
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "ContainerInsights", "node_memory_utilization", "ClusterName", "retail-prediction-cluster" ],
          [ "ContainerInsights", "pod_memory_utilization", "ClusterName", "retail-prediction-cluster", "Namespace", "retail-prediction" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Memory Usage",
        "period": 60,
        "yAxis": {
          "left": {
            "label": "Percent",
            "showUnits": false
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 8,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/retail-prediction-alb/1234567890123456" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Request per Second (QPS)",
        "period": 60,
        "stat": "Sum",
        "yAxis": {
          "left": {
            "label": "Count/Minute",
            "showUnits": false
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 16,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/SageMaker", "TrainingElapsedTimeSeconds", "TrainingJobName", "retail-prediction-training" ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Training Duration (min)",
        "period": 60,
        "stat": "Maximum",
        "yAxis": {
          "left": {
            "label": "Minutes",
            "showUnits": false
          }
        }
      }
    }
  ]
}
```

### 4.2 Dashboard Visualization

{{< mermaid >}}
graph TB
    subgraph "CloudWatch Dashboard"
        subgraph "Row 1"
            P1["API Latency (P50/P95)"]
            P2["Error Rate (%)"]
            P3["CPU Utilization"]
        end
        subgraph "Row 2"
            P4["Memory Usage"]
            P5["Request per Second (QPS)"]
            P6["Training Duration (min)"]
        end
    end
    
    subgraph "Data Sources"
        DS1["API Logs"]
        DS2["Container Insights"]
        DS3["ALB Metrics"]
        DS4["SageMaker Metrics"]
    end
    
    DS1 --> P1
    DS1 --> P2
    DS2 --> P3
    DS2 --> P4
    DS3 --> P5
    DS4 --> P6
    
    style P1 fill:#f9f,stroke:#333,stroke-width:1px
    style P2 fill:#f9f,stroke:#333,stroke-width:1px
    style P3 fill:#bbf,stroke:#333,stroke-width:1px
    style P4 fill:#bbf,stroke:#333,stroke-width:1px
    style P5 fill:#bfb,stroke:#333,stroke-width:1px
    style P6 fill:#bfb,stroke:#333,stroke-width:1px
{{< /mermaid >}}

#### 6.2 Deploy Dashboard

```bash
# Create dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "RetailForecast-MLOps-Dashboard" \
  --dashboard-body file://dashboard.json

# Get dashboard URL
echo "Dashboard URL: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=RetailForecast-MLOps-Dashboard"
```

### 7. Log Analysis và Queries

#### 7.1 CloudWatch Insights Queries

```sql
-- Find errors in application logs
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Analyze request patterns
fields @timestamp, @message
| filter @message like /prediction/
| stats count() by bin(5m)

-- Monitor response times
fields @timestamp, @message
| filter @message like /response_time/
| parse @message "response_time: * ms" as response_time
| stats avg(response_time), max(response_time), min(response_time) by bin(5m)

-- Track model predictions
fields @timestamp, @message
| filter @message like /model_prediction/
| parse @message "prediction: *, confidence: *" as prediction, confidence
| stats count() by bin(1h)

-- Monitor memory usage patterns
fields @timestamp, @message
| filter @message like /memory_usage/
| parse @message "memory_usage: * MB" as memory
| stats avg(memory), max(memory) by bin(5m)
```

### 8. Testing và Validation

#### 8.1 Generate Test Load

```bash
# Generate load to trigger alarms
# Install siege for load testing
siege -c 50 -t 5m http://your-load-balancer-dns/predict

# Generate errors to test error rate alarm
for i in {1..20}; do
  curl -X POST http://your-load-balancer-dns/non-existent-endpoint
done

# Monitor alarm state changes
aws cloudwatch describe-alarms \
  --alarm-names "RetailForecast-HighErrorRate" "RetailForecast-HighLatency" \
  --query 'MetricAlarms[*].[AlarmName,StateValue,StateReason]' \
  --output table
```

#### 8.2 Validate Log Collection

```bash
# Check if logs are being collected
aws logs describe-log-streams \
  --log-group-name "/aws/eks/retail-forecast/application" \
  --order-by LastEventTime \
  --descending

# Query recent logs
aws logs start-query \
  --log-group-name "/aws/eks/retail-forecast/application" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc'
```

### 9. Automated Remediation

#### 9.1 Lambda Function for Auto-Scaling

```python
import json
import boto3

def lambda_handler(event, context):
    """
    Auto-scale EKS deployment based on CloudWatch alarms
    """
    
    # Parse SNS message
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    new_state = message['NewStateValue']
    
    if alarm_name == 'RetailForecast-HighCPU' and new_state == 'ALARM':
        # Scale up deployment
        scale_deployment('retail-forecast-api', 'mlops', replicas=6)
    elif alarm_name == 'RetailForecast-HighCPU' and new_state == 'OK':
        # Scale down deployment
        scale_deployment('retail-forecast-api', 'mlops', replicas=3)
    
    return {'statusCode': 200, 'body': json.dumps('Success')}

def scale_deployment(deployment_name, namespace, replicas):
    """Scale Kubernetes deployment"""
    import subprocess
    
    cmd = f"kubectl scale deployment {deployment_name} --replicas={replicas} -n {namespace}"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    
    if result.returncode == 0:
        print(f"Successfully scaled {deployment_name} to {replicas} replicas")
    else:
        print(f"Failed to scale deployment: {result.stderr}")
```

#### 9.2 Runbook Automation

```yaml
# automation-runbook.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: incident-response-runbook
  namespace: mlops
data:
  runbook.md: |
    # Incident Response Runbook
    
    ## High Error Rate (5xx)
    1. Check application logs for error patterns
    2. Verify deployment health: `kubectl get pods -n mlops`
    3. Check resource usage: `kubectl top pods -n mlops`
    4. If pods are unhealthy, restart: `kubectl rollout restart deployment/retail-forecast-api -n mlops`
    
    ## High Latency
    1. Check if HPA is scaling: `kubectl get hpa -n mlops`
    2. Monitor resource usage: `kubectl top nodes`
    3. Check if database/external services are responding
    4. Scale manually if needed: `kubectl scale deployment retail-forecast-api --replicas=5 -n mlops`
    
    ## High Resource Usage
    1. Check which pods are consuming resources: `kubectl top pods -n mlops --sort-by=cpu`
    2. Verify memory leaks in application logs
    3. Consider vertical scaling or pod limits adjustment
    4. Check for infinite loops or stuck processes
    
    ## Low Traffic/Potential Downtime
    1. Verify load balancer health: `aws elbv2 describe-target-health --target-group-arn <arn>`
    2. Check if service is accessible: `kubectl get endpoints -n mlops`
    3. Verify ingress configuration: `kubectl describe ingress -n mlops`
    4. Test connectivity: `curl http://your-endpoint/healthz`
---
```

## 7. Chi phí ước tính

Ước tính chi phí hàng tháng cho hệ thống monitoring:

| Thành phần | Ước tính | Ghi chú |
|------------|----------|---------|
| CloudWatch Logs | ~0.10 USD/tháng | Vài GB log dữ liệu |
| Dashboard & Alarms | 0 USD | Miễn phí trong giới hạn cơ bản |
| SNS Notification | 0 USD | Miễn phí nếu <100 email/tháng |
| Container Insights | ~0.30 USD/tháng | 5 node, metrics mỗi phút |
| **Tổng** | **~0.40 USD/tháng** | Chi phí thấp, chủ yếu do log lưu trữ |

{{% notice info %}}
CloudWatch có chi phí rất thấp so với giá trị mà nó mang lại cho việc monitoring và alert trong MLOps pipeline. Lưu ý rằng chi phí có thể tăng nếu số lượng log và số lượng metrics tăng đáng kể.
{{% /notice %}}

## 8. Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **CloudWatch Agent**: Cài đặt thành công CloudWatch Agent trên EKS cluster
- [ ] **Container Insights**: Thu thập metrics về CPU, memory, và network từ node và pod
- [ ] **API Metrics**: Log và metrics từ API được ghi nhận và hiển thị trong CloudWatch
- [ ] **Dashboard KPI**: Dashboard "Retail Prediction KPI" hiển thị 6 metrics chính
- [ ] **Alarms & Alerts**: Cảnh báo được thiết lập cho high latency, error rate, và CPU usage
- [ ] **Auto Scaling**: Lambda function được cấu hình để tự động scale nodegroup
- [ ] **Training Logs**: SageMaker logs được tích hợp vào CloudWatch
- [ ] **Cost Optimization**: Chi phí monitoring ước tính dưới 0.5 USD/tháng

### 📊 Verification Steps

1. **Xác nhận CloudWatch Agent đang hoạt động**
   ```bash
   # Kiểm tra CloudWatch Agent pods
   kubectl get pods -n amazon-cloudwatch
   # Expected: cloudwatch-agent pods should be running
   
   # Kiểm tra logs từ CloudWatch Agent
   kubectl logs -l k8s-app=cloudwatch-agent -n amazon-cloudwatch
   # Expected: No error messages, agent is collecting metrics
   ```

2. **Xác nhận logs từ API container được gửi đến CloudWatch**
   ```bash
   # Kiểm tra log groups
   aws logs describe-log-groups --log-group-name-prefix "/aws/container/retail-api"
   # Expected: Log group exists
   
   # Truy vấn log mẫu
   aws logs start-query \
     --log-group-name "/aws/container/retail-api" \
     --start-time $(date -d '1 hour ago' +%s) \
     --end-time $(date +%s) \
     --query-string 'fields @timestamp, @message | sort @timestamp desc | limit 10'
   # Expected: Should see API request logs with latency metrics
   ```

3. **Kiểm tra dashboard và metric filters**
   ```bash
   # Kiểm tra dashboard
   aws cloudwatch get-dashboard \
     --dashboard-name "Retail-Prediction-KPI"
   # Expected: Dashboard configuration should be returned
   
   # Kiểm tra metric filters
   aws logs describe-metric-filters \
     --log-group-name "/aws/container/retail-api"
   # Expected: Should see InferenceLatency, ErrorRate, and other filters
   ```

4. **Kiểm tra Alarms và SNS notifications**
   ```bash
   # Tạo tải thử nghiệm để kích hoạt alarm
   kubectl run load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://retail-api-service.retail-prediction.svc.cluster.local:8080/predict; done"
   
   # Kiểm tra trạng thái alarms
   aws cloudwatch describe-alarms \
     --alarm-names "RetailPrediction-HighLatency" "RetailPrediction-HighErrorRate" "RetailPrediction-HighCPU" \
     --query 'MetricAlarms[*].[AlarmName,StateValue,StateReason]' \
     --output table
   # Expected: Alarms should change state in response to load
   
   # Xóa pod tạo tải sau khi hoàn thành kiểm tra
   kubectl delete pod load-generator
   ```

5. **Kiểm tra logs từ SageMaker training**
   ```bash
   # Liệt kê log streams cho training job
   aws logs describe-log-streams \
     --log-group-name "/aws/sagemaker/TrainingJobs/retail-prediction-training"
   # Expected: Should see log streams for training job
   
   # Kiểm tra training metrics từ CloudWatch
   aws cloudwatch get-metric-statistics \
     --namespace "RetailMLOps" \
     --metric-name "TrainingLoss" \
     --statistics Average \
     --period 3600 \
     --start-time $(date -d '24 hours ago' +%s) \
     --end-time $(date +%s)
   # Expected: Should see training loss metrics
   ```

### 🔍 Monitoring Commands

```bash
# Kiểm tra tất cả alarms
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[?starts_with(AlarmName, `RetailPrediction`)].[AlarmName,StateValue,StateReason]' \
  --output table

# Theo dõi API hiệu năng real-time
watch -n 5 "aws cloudwatch get-metric-statistics \
  --namespace RetailMLOps \
  --metric-name InferenceLatency \
  --statistics Average p50 p95 p99 \
  --period 60 \
  --start-time \$(date -d '30 minutes ago' +%s) \
  --end-time \$(date +%s) \
  --output table"

# Kiểm tra CPU/Memory của EKS nodes
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name node_cpu_utilization \
  --statistics Average \
  --period 300 \
  --dimensions Name=ClusterName,Value=retail-prediction-cluster \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)

# Kiểm tra logs mới nhất từ API
aws logs get-log-events \
  --log-group-name "/aws/container/retail-api" \
  --log-stream-name $(aws logs describe-log-streams --log-group-name "/aws/container/retail-api" --order-by LastEventTime --descending --limit 1 --query 'logStreams[0].logStreamName' --output text) \
  --limit 10

# Dashboard và metrics
aws cloudwatch list-dashboards --dashboard-name-prefix "Retail-Prediction"
aws cloudwatch list-metrics --namespace "RetailMLOps"
```

## 9. Log Analysis và Troubleshooting

### 9.1 CloudWatch Logs Insights Queries

CloudWatch Logs Insights cho phép truy vấn logs một cách hiệu quả để phân tích và giải quyết vấn đề:

```sql
-- Phân tích latency theo percentile
fields @timestamp, @message
| filter @message like "API Request" 
| parse @message '"latency_ms": *,' as latency
| stats count() as requests, 
        min(latency) as min_latency,
        avg(latency) as avg_latency, 
        pct(latency, 50) as p50, 
        pct(latency, 95) as p95, 
        pct(latency, 99) as p99,
        max(latency) as max_latency by bin(5m)

-- Tìm các request có latency cao
fields @timestamp, @message
| filter @message like "API Request"
| parse @message '"latency_ms": *,' as latency
| filter latency > 200
| sort by latency desc
| limit 20

-- Phân tích error rate theo endpoint
fields @timestamp, @message
| filter @message like "API Error"
| parse @message '"endpoint": "*",' as endpoint
| parse @message '"status_code": *,' as status_code
| stats count(*) as error_count by endpoint, status_code
| sort by error_count desc

-- Theo dõi training metrics
fields @timestamp, @message
| filter @message like "loss" or @message like "accuracy"
| parse @message "loss: *," as loss
| parse @message "accuracy: *," as accuracy
| parse @message "epoch * " as epoch
| sort by @timestamp asc
| display @timestamp, epoch, loss, accuracy
```

### 9.2 Common Issues & Solutions

#### 1. CloudWatch Agent không thu thập metrics

**Vấn đề**: Metrics từ container không xuất hiện trong CloudWatch.

**Giải pháp**:
```bash
# Kiểm tra status của CloudWatch Agent
kubectl get pods -n amazon-cloudwatch
kubectl describe pod -n amazon-cloudwatch -l k8s-app=cloudwatch-agent

# Kiểm tra logs của CloudWatch Agent
kubectl logs -n amazon-cloudwatch -l k8s-app=cloudwatch-agent

# Kiểm tra IAM permissions
aws iam get-role --role-name EKS-CloudWatchAgentRole
```

#### 2. Alarm kích hoạt sai

**Vấn đề**: CloudWatch Alarm kích hoạt không mong muốn hoặc không kích hoạt khi cần.

**Giải pháp**:
```bash
# Kiểm tra metric data
aws cloudwatch get-metric-statistics \
  --namespace "RetailMLOps" \
  --metric-name "InferenceLatency" \
  --statistics Average \
  --period 60 \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)

# Kiểm tra và cập nhật alarm configuration
aws cloudwatch describe-alarms --alarm-names "RetailPrediction-HighLatency"
```

#### 3. SageMaker logs không xuất hiện

**Vấn đề**: Training logs từ SageMaker không được ghi nhận vào CloudWatch.

**Giải pháp**:
```bash
# Kiểm tra SageMaker training job
aws sagemaker describe-training-job --training-job-name retail-prediction-training

# Kiểm tra IAM role của SageMaker
aws iam get-role --role-name SageMakerExecutionRole

# Đảm bảo role có quyền CloudWatchLogsFullAccess
aws iam attach-role-policy \
  --role-name SageMakerExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

### 9.3 Best Practices

1. **Thiết lập log retention**: Giữ logs trong khoảng thời gian phù hợp để tiết kiệm chi phí
   ```bash
   aws logs put-retention-policy --log-group-name "/aws/container/retail-api" --retention-in-days 30
   ```

2. **Sử dụng metric filters hiệu quả**: Chỉ trích xuất các metrics thực sự cần thiết

3. **Sử dụng composite alarms**: Kết hợp nhiều điều kiện để giảm false positives
   ```bash
   aws cloudwatch put-composite-alarm \
     --alarm-name "RetailPrediction-CriticalIssue" \
     --alarm-rule "(ALARM(RetailPrediction-HighLatency) AND ALARM(RetailPrediction-HighErrorRate))"
   ```

4. **Log rotation**: Đảm bảo logs không chiếm quá nhiều disk space trên pods
   ```yaml
   # Trong Deployment
   volumeMounts:
     - name: varlog
       mountPath: /var/log
   volumes:
     - name: varlog
       emptyDir: {}
   ```

## 10. Kết luận

Hệ thống monitoring và logging được xây dựng cho MLOps Retail Prediction mang lại những lợi ích quan trọng:

✔️ **Giám sát toàn diện**: Theo dõi từ infrastructure (EKS), application (API) đến ML model (inference, training).

✔️ **Phát hiện sớm lỗi**: Cảnh báo kịp thời khi latency cao, error rate tăng hoặc resource bị cạn kiệt.

✔️ **Tối ưu hiệu năng**: Dữ liệu metrics giúp phân tích bottleneck và cải thiện API performance.

✔️ **Tự động hóa**: Auto-scaling dựa trên metrics giúp đáp ứng tải tự động và tiết kiệm chi phí.

✔️ **MLOps observability**: Theo dõi quá trình huấn luyện và hiệu suất model giúp cải thiện liên tục.

Hệ thống giám sát này là thành phần quan trọng của MLOps pipeline, giúp đảm bảo tính ổn định, độ tin cậy và hiệu suất của dịch vụ dự đoán trong môi trường production.

---

**Next Step**: [Task 13: CI/CD Pipeline Automation](../13-cicd-pipeline/)