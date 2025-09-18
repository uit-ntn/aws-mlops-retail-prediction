---
title: "Task 12: CloudWatch Monitoring & Alerting"
date: 2024-01-01T00:00:00Z
weight: 12
chapter: false
pre: "<b>12. </b>"
---

## Mục tiêu

Thiết lập hệ thống giám sát và cảnh báo cho ứng dụng inference chạy trên EKS, đảm bảo theo dõi được hiệu năng, lỗi và tài nguyên sử dụng.

## Nội dung chính

### 1. CloudWatch Logs Setup

#### 1.1 Enable CloudWatch Logs for EKS

```bash
# Create CloudWatch Log Group for EKS cluster
aws logs create-log-group \
  --log-group-name /aws/eks/retail-forecast-cluster/cluster \
  --retention-in-days 30

# Create Log Group for application logs
aws logs create-log-group \
  --log-group-name /aws/eks/retail-forecast/application \
  --retention-in-days 30

# Create Log Group for load balancer logs
aws logs create-log-group \
  --log-group-name /aws/applicationloadbalancer/retail-forecast \
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

### 2. Container Insights Setup

#### 2.1 Enable Container Insights

```bash
# Install CloudWatch Insights using eksctl
eksctl utils install-vpc-controllers \
  --cluster retail-forecast-cluster \
  --approve

# Enable Container Insights
aws eks update-cluster-config \
  --region us-east-1 \
  --name retail-forecast-cluster \
  --logging '{"enable":["api","audit","authenticator","controllerManager","scheduler"]}'

# Deploy CloudWatch Agent
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | kubectl apply -f -
```

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

### 3. Custom Metrics từ Application

#### 3.1 Application Metrics với Prometheus

```python
# metrics.py - Add to your application
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time
import functools

# Define metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency', ['method', 'endpoint'])
ACTIVE_CONNECTIONS = Gauge('active_connections', 'Active connections')
MODEL_PREDICTION_TIME = Histogram('model_prediction_duration_seconds', 'Model prediction time')
MODEL_PREDICTIONS_TOTAL = Counter('model_predictions_total', 'Total model predictions', ['status'])
ERROR_RATE = Counter('application_errors_total', 'Total application errors', ['error_type'])

class MetricsMiddleware:
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        start_time = time.time()
        method = environ['REQUEST_METHOD']
        path = environ['PATH_INFO']
        
        def custom_start_response(status, headers):
            status_code = int(status.split()[0])
            REQUEST_COUNT.labels(method=method, endpoint=path, status=status_code).inc()
            REQUEST_LATENCY.labels(method=method, endpoint=path).observe(time.time() - start_time)
            return start_response(status, headers)
        
        return self.app(environ, custom_start_response)

def track_prediction_metrics(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        try:
            result = func(*args, **kwargs)
            MODEL_PREDICTIONS_TOTAL.labels(status='success').inc()
            return result
        except Exception as e:
            MODEL_PREDICTIONS_TOTAL.labels(status='error').inc()
            ERROR_RATE.labels(error_type=type(e).__name__).inc()
            raise
        finally:
            MODEL_PREDICTION_TIME.observe(time.time() - start_time)
    return wrapper

# Start metrics server
def start_metrics_server(port=8000):
    start_http_server(port)
    print(f"Metrics server started on port {port}")
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

### 4. CloudWatch Alarms Configuration

#### 4.1 Application Performance Alarms

```bash
# High Error Rate Alarm (5xx errors)
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-HighErrorRate" \
  --alarm-description "High 5xx error rate for Retail Forecast API" \
  --metric-name "HTTPCode_Target_5XX_Count" \
  --namespace "AWS/ApplicationELB" \
  --statistic "Sum" \
  --period 300 \
  --threshold 10 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=LoadBalancer,Value=app/retail-forecast-alb/1234567890123456

# High Response Time Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-HighLatency" \
  --alarm-description "High response time for Retail Forecast API" \
  --metric-name "TargetResponseTime" \
  --namespace "AWS/ApplicationELB" \
  --statistic "Average" \
  --period 300 \
  --threshold 2.0 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=LoadBalancer,Value=app/retail-forecast-alb/1234567890123456

# Low Request Rate (potential downtime)
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-LowTraffic" \
  --alarm-description "Unusually low traffic to Retail Forecast API" \
  --metric-name "RequestCount" \
  --namespace "AWS/ApplicationELB" \
  --statistic "Sum" \
  --period 600 \
  --threshold 5 \
  --comparison-operator "LessThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=LoadBalancer,Value=app/retail-forecast-alb/1234567890123456
```

#### 4.2 Infrastructure Resource Alarms

```bash
# High CPU Usage Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-HighCPU" \
  --alarm-description "High CPU usage in EKS cluster" \
  --metric-name "node_cpu_utilization" \
  --namespace "ContainerInsights" \
  --statistic "Average" \
  --period 300 \
  --threshold 80 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=ClusterName,Value=retail-forecast-cluster

# High Memory Usage Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-HighMemory" \
  --alarm-description "High memory usage in EKS cluster" \
  --metric-name "node_memory_utilization" \
  --namespace "ContainerInsights" \
  --statistic "Average" \
  --period 300 \
  --threshold 85 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=ClusterName,Value=retail-forecast-cluster

# Pod Restart Rate Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-PodRestarts" \
  --alarm-description "High pod restart rate" \
  --metric-name "pod_restart_count" \
  --namespace "ContainerInsights" \
  --statistic "Sum" \
  --period 600 \
  --threshold 5 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts" \
  --dimensions Name=ClusterName,Value=retail-forecast-cluster Name=Namespace,Value=mlops
```

#### 4.3 Custom Metrics Alarms

```bash
# Model Prediction Error Rate
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-ModelErrorRate" \
  --alarm-description "High model prediction error rate" \
  --metric-name "model_predictions_total" \
  --namespace "CustomMetrics/RetailForecast" \
  --statistic "Average" \
  --period 300 \
  --threshold 0.1 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts"

# Model Prediction Latency
aws cloudwatch put-metric-alarm \
  --alarm-name "RetailForecast-ModelLatency" \
  --alarm-description "High model prediction latency" \
  --metric-name "model_prediction_duration_seconds" \
  --namespace "CustomMetrics/RetailForecast" \
  --statistic "Average" \
  --period 300 \
  --threshold 5.0 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts"
```

### 5. SNS Notification Setup

#### 5.1 Create SNS Topic và Subscriptions

```bash
# Create SNS topic
aws sns create-topic \
  --name retail-forecast-alerts \
  --attributes DisplayName="Retail Forecast Alerts"

# Get topic ARN
TOPIC_ARN=$(aws sns get-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts \
  --query 'Attributes.TopicArn' \
  --output text)

# Subscribe email addresses
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint devops-team@company.com

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint ml-team@company.com

# Subscribe Slack webhook (optional)
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol https \
  --notification-endpoint https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

#### 5.2 SNS Topic Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudWatchAlarmsToPublish",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudwatch.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts"
    },
    {
      "Sid": "AllowAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": [
        "SNS:GetTopicAttributes",
        "SNS:SetTopicAttributes",
        "SNS:AddPermission",
        "SNS:RemovePermission",
        "SNS:DeleteTopic",
        "SNS:Subscribe",
        "SNS:ListSubscriptionsByTopic",
        "SNS:Publish"
      ],
      "Resource": "arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts"
    }
  ]
}
```

### 6. CloudWatch Dashboard

#### 6.1 Create Comprehensive Dashboard

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
          ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/retail-forecast-alb/1234567890123456"],
          [".", "TargetResponseTime", ".", "."],
          [".", "HTTPCode_Target_2XX_Count", ".", "."],
          [".", "HTTPCode_Target_4XX_Count", ".", "."],
          [".", "HTTPCode_Target_5XX_Count", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Application Load Balancer Metrics",
        "period": 300
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
          ["ContainerInsights", "node_cpu_utilization", "ClusterName", "retail-forecast-cluster"],
          [".", "node_memory_utilization", ".", "."],
          [".", "node_network_total_bytes", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "EKS Cluster Resource Utilization",
        "period": 300
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
          ["ContainerInsights", "pod_cpu_utilization", "ClusterName", "retail-forecast-cluster", "Namespace", "mlops"],
          [".", "pod_memory_utilization", ".", ".", ".", "."],
          [".", "pod_network_rx_bytes", ".", ".", ".", "."],
          [".", "pod_network_tx_bytes", ".", ".", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Pod Metrics",
        "period": 300
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
          ["CustomMetrics/RetailForecast", "model_predictions_total"],
          [".", "model_prediction_duration_seconds"],
          [".", "application_errors_total"]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Custom Application Metrics",
        "period": 300
      }
    },
    {
      "type": "log",
      "x": 0,
      "y": 12,
      "width": 24,
      "height": 6,
      "properties": {
        "query": "SOURCE '/aws/eks/retail-forecast/application'\n| fields @timestamp, log\n| filter log like /ERROR/\n| sort @timestamp desc\n| limit 100",
        "region": "us-east-1",
        "title": "Recent Error Logs",
        "view": "table"
      }
    }
  ]
}
```

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

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **CloudWatch Logs**: Logs từ pods được thu thập và hiển thị
- [ ] **Container Insights**: CPU, memory, network metrics được thu thập
- [ ] **Custom Metrics**: Application metrics được gửi đến CloudWatch
- [ ] **Alarms**: CloudWatch alarms được cấu hình cho các scenarios quan trọng
- [ ] **SNS Notifications**: Email/Slack notifications được thiết lập
- [ ] **Dashboard**: Comprehensive dashboard được tạo
- [ ] **Log Queries**: CloudWatch Insights queries hoạt động
- [ ] **Automated Testing**: Load testing để trigger alarms
- [ ] **Automated Remediation**: Lambda functions cho auto-scaling

### 📊 Verification Steps

1. **Log từ các pod inference hiển thị trong CloudWatch**
   ```bash
   aws logs describe-log-groups --log-group-name-prefix "/aws/eks/retail-forecast"
   # Expected: Log groups are created and receiving data
   
   aws logs start-query \
     --log-group-name "/aws/eks/retail-forecast/application" \
     --start-time $(date -d '1 hour ago' +%s) \
     --end-time $(date +%s) \
     --query-string 'fields @timestamp, @message | sort @timestamp desc | limit 10'
   ```

2. **Khi tạo tải thử nghiệm, alarm chuyển trạng thái (OK → ALARM)**
   ```bash
   # Generate load
   siege -c 20 -t 2m http://your-load-balancer/healthz
   
   # Check alarm states
   aws cloudwatch describe-alarms \
     --alarm-names "RetailForecast-HighLatency" "RetailForecast-HighErrorRate" \
     --query 'MetricAlarms[*].[AlarmName,StateValue,StateChangeTime]' \
     --output table
   # Expected: Alarms should transition from OK to ALARM
   ```

3. **Có thể giám sát toàn bộ hệ thống từ CloudWatch console**
   ```bash
   # Access dashboard
   echo "Dashboard: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=RetailForecast-MLOps-Dashboard"
   
   # Check Container Insights
   echo "Container Insights: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#container-insights:performance/EKS:Cluster/retail-forecast-cluster"
   ```

### 🔍 Monitoring Commands

```bash
# Check all alarms status
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[?starts_with(AlarmName, `RetailForecast`)].[AlarmName,StateValue,StateReason]' \
  --output table

# Monitor log ingestion
aws logs describe-metric-filters \
  --log-group-name "/aws/eks/retail-forecast/application"

# Check SNS subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456789012:retail-forecast-alerts

# Monitor custom metrics
aws cloudwatch list-metrics \
  --namespace "CustomMetrics/RetailForecast"

# Check dashboard
aws cloudwatch list-dashboards \
  --dashboard-name-prefix "RetailForecast"
```

## Troubleshooting

### Common Issues

1. **Logs not appearing in CloudWatch**
   - Check Fluent Bit pod status: `kubectl get pods -n amazon-cloudwatch`
   - Verify IAM permissions for CloudWatch Logs
   - Check Fluent Bit configuration

2. **Alarms not triggering**
   - Verify metric data exists: `aws cloudwatch get-metric-statistics`
   - Check alarm threshold and evaluation periods
   - Ensure sufficient data points

3. **SNS notifications not received**
   - Check email subscription confirmation
   - Verify SNS topic permissions
   - Check spam folder for notifications

---

**Next Step**: [Task 13: Security & Compliance](../13-security-compliance/)