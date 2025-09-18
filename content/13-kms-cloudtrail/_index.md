---
title: "KMS & CloudTrail (Encryption & Audit)"
date: 2024-01-01T00:00:00Z
weight: 13
chapter: false
pre: "<b>13. </b>"
---

## Mục tiêu

Đảm bảo bảo mật dữ liệu và theo dõi hoạt động trong hệ thống MLOps bằng mã hóa và ghi nhật ký truy cập.

## Nội dung chính

### 1. AWS KMS (Key Management Service) Setup

#### 1.1 Create Customer Managed KMS Keys

```bash
# Create KMS key for S3 encryption
aws kms create-key \
  --description "Retail Forecast S3 Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --origin AWS_KMS \
  --multi-region \
  --tags TagKey=Project,TagValue=RetailForecast TagKey=Environment,TagValue=Production TagKey=Purpose,TagValue=S3Encryption

# Get the key ID
S3_KEY_ID=$(aws kms list-keys --query 'Keys[0].KeyId' --output text)

# Create alias for S3 key
aws kms create-alias \
  --alias-name alias/retail-forecast-s3-key \
  --target-key-id $S3_KEY_ID

# Create KMS key for ECR encryption
aws kms create-key \
  --description "Retail Forecast ECR Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --origin AWS_KMS \
  --multi-region \
  --tags TagKey=Project,TagValue=RetailForecast TagKey=Environment,TagValue=Production TagKey=Purpose,TagValue=ECREncryption

# Get the ECR key ID
ECR_KEY_ID=$(aws kms list-keys --query 'Keys[1].KeyId' --output text)

# Create alias for ECR key
aws kms create-alias \
  --alias-name alias/retail-forecast-ecr-key \
  --target-key-id $ECR_KEY_ID

# Create KMS key for EKS encryption
aws kms create-key \
  --description "Retail Forecast EKS Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --origin AWS_KMS \
  --multi-region \
  --tags TagKey=Project,TagValue=RetailForecast TagKey=Environment,TagValue=Production TagKey=Purpose,TagValue=EKSEncryption

# Get the EKS key ID
EKS_KEY_ID=$(aws kms list-keys --query 'Keys[2].KeyId' --output text)

# Create alias for EKS key
aws kms create-alias \
  --alias-name alias/retail-forecast-eks-key \
  --target-key-id $EKS_KEY_ID
```

#### 1.2 KMS Key Policies

```json
{
  "Version": "2012-10-17",
  "Id": "retail-forecast-s3-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow S3 Service",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:ReEncrypt*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "Allow SageMaker Service",
      "Effect": "Allow",
      "Principal": {
        "Service": "sagemaker.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:CreateGrant"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "Allow EKS Service",
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:CreateGrant"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow CloudTrail Service",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

#### 1.3 Apply Key Policies

```bash
# Apply policy to S3 key
aws kms put-key-policy \
  --key-id $S3_KEY_ID \
  --policy-name default \
  --policy file://s3-key-policy.json

# Apply policy to ECR key
aws kms put-key-policy \
  --key-id $ECR_KEY_ID \
  --policy-name default \
  --policy file://ecr-key-policy.json

# Apply policy to EKS key
aws kms put-key-policy \
  --key-id $EKS_KEY_ID \
  --policy-name default \
  --policy file://eks-key-policy.json
```

### 2. S3 Bucket Encryption

#### 2.1 Enable SSE-KMS for Existing Buckets

```bash
# Enable encryption for data bucket
aws s3api put-bucket-encryption \
  --bucket retail-forecast-data-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "alias/retail-forecast-s3-key"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# Enable encryption for artifacts bucket
aws s3api put-bucket-encryption \
  --bucket retail-forecast-artifacts-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "alias/retail-forecast-s3-key"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# Create new encrypted bucket for CloudTrail logs
aws s3 mb s3://retail-forecast-cloudtrail-logs-$(date +%s)

CLOUDTRAIL_BUCKET="retail-forecast-cloudtrail-logs-$(date +%s)"

aws s3api put-bucket-encryption \
  --bucket $CLOUDTRAIL_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "alias/retail-forecast-s3-key"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'
```

#### 2.2 S3 Bucket Policy for Encryption Enforcement

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::retail-forecast-data-bucket",
        "arn:aws:s3:::retail-forecast-data-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::retail-forecast-data-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::retail-forecast-data-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id": "alias/retail-forecast-s3-key"
        }
      }
    }
  ]
}
```

#### 2.3 Apply Bucket Policies

```bash
# Apply encryption enforcement policy
aws s3api put-bucket-policy \
  --bucket retail-forecast-data-bucket \
  --policy file://s3-encryption-policy.json

aws s3api put-bucket-policy \
  --bucket retail-forecast-artifacts-bucket \
  --policy file://s3-encryption-policy.json
```

### 3. ECR Repository Encryption

#### 3.1 Create Encrypted ECR Repository

```bash
# Create ECR repository with KMS encryption
aws ecr create-repository \
  --repository-name retail-forecast \
  --encryption-configuration '{
    "encryptionType": "KMS",
    "kmsKey": "alias/retail-forecast-ecr-key"
  }' \
  --image-scanning-configuration '{
    "scanOnPush": true
  }' \
  --tags Key=Project,Value=RetailForecast Key=Environment,Value=Production

# Enable lifecycle policy for cost optimization
aws ecr put-lifecycle-policy \
  --repository-name retail-forecast \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {
          "type": "expire"
        }
      },
      {
        "rulePriority": 2,
        "description": "Delete untagged images older than 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": {
          "type": "expire"
        }
      }
    ]
  }'
```

#### 3.2 ECR Repository Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEKSAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/NodeInstanceRole"
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ]
    },
    {
      "Sid": "AllowCICDAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/RetailForecastCICDRole"
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ]
    },
    {
      "Sid": "DenyUnencryptedAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### 4. EKS Encryption

#### 4.1 Enable EKS Cluster Encryption

```bash
# Update EKS cluster with encryption
aws eks update-cluster-config \
  --region us-east-1 \
  --name retail-forecast-cluster \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {
      "keyArn": "arn:aws:kms:us-east-1:YOUR_ACCOUNT_ID:key/'$EKS_KEY_ID'"
    }
  }]'

# Check encryption status
aws eks describe-cluster \
  --name retail-forecast-cluster \
  --query 'cluster.encryptionConfig'
```

#### 4.2 EBS Volume Encryption for EKS Nodes

```yaml
# ebs-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: alias/retail-forecast-eks-key
  fsType: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
```

### 5. CloudTrail Setup

#### 5.1 Create CloudTrail

```bash
# Create CloudTrail bucket policy first
cat > cloudtrail-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::$CLOUDTRAIL_BUCKET"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::$CLOUDTRAIL_BUCKET/AWSLogs/YOUR_ACCOUNT_ID/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::$CLOUDTRAIL_BUCKET",
        "arn:aws:s3:::$CLOUDTRAIL_BUCKET/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
EOF

# Apply bucket policy
aws s3api put-bucket-policy \
  --bucket $CLOUDTRAIL_BUCKET \
  --policy file://cloudtrail-bucket-policy.json

# Create CloudTrail
aws cloudtrail create-trail \
  --name retail-forecast-audit-trail \
  --s3-bucket-name $CLOUDTRAIL_BUCKET \
  --s3-key-prefix "retail-forecast-logs" \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id alias/retail-forecast-s3-key \
  --tags-list '[
    {"Key": "Project", "Value": "RetailForecast"},
    {"Key": "Environment", "Value": "Production"},
    {"Key": "Purpose", "Value": "AuditLogging"}
  ]'

# Start logging
aws cloudtrail start-logging \
  --name retail-forecast-audit-trail
```

#### 5.2 CloudTrail Event Selectors

```bash
# Configure advanced event selectors for specific services
aws cloudtrail put-event-selectors \
  --trail-name retail-forecast-audit-trail \
  --advanced-event-selectors '[
    {
      "Name": "EKS API Calls",
      "FieldSelectors": [
        {
          "Field": "eventCategory",
          "Equals": ["Management"]
        },
        {
          "Field": "eventName",
          "StartsWith": ["CreateCluster", "DeleteCluster", "UpdateCluster", "CreateNodegroup", "DeleteNodegroup", "UpdateNodegroup"]
        }
      ]
    },
    {
      "Name": "S3 Data Events",
      "FieldSelectors": [
        {
          "Field": "eventCategory",
          "Equals": ["Data"]
        },
        {
          "Field": "resources.type",
          "Equals": ["AWS::S3::Object"]
        },
        {
          "Field": "resources.ARN",
          "StartsWith": ["arn:aws:s3:::retail-forecast-data-bucket/", "arn:aws:s3:::retail-forecast-artifacts-bucket/"]
        }
      ]
    },
    {
      "Name": "SageMaker Events",
      "FieldSelectors": [
        {
          "Field": "eventCategory",
          "Equals": ["Management"]
        },
        {
          "Field": "eventSource",
          "Equals": ["sagemaker.amazonaws.com"]
        }
      ]
    },
    {
      "Name": "KMS Key Usage",
      "FieldSelectors": [
        {
          "Field": "eventCategory",
          "Equals": ["Management"]
        },
        {
          "Field": "eventSource",
          "Equals": ["kms.amazonaws.com"]
        },
        {
          "Field": "eventName",
          "StartsWith": ["Decrypt", "Encrypt", "GenerateDataKey", "CreateGrant"]
        }
      ]
    }
  ]'
```

### 6. CloudWatch Integration for Security Monitoring

#### 6.1 CloudTrail Log Metric Filters

```bash
# Create log group for CloudTrail if not exists
aws logs create-log-group \
  --log-group-name /aws/cloudtrail/retail-forecast-audit \
  --retention-in-days 90

# Update CloudTrail to send logs to CloudWatch
aws cloudtrail update-trail \
  --name retail-forecast-audit-trail \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:YOUR_ACCOUNT_ID:log-group:/aws/cloudtrail/retail-forecast-audit:*" \
  --cloud-watch-logs-role-arn "arn:aws:iam::YOUR_ACCOUNT_ID:role/CloudTrailLogsRole"

# Create metric filters for security events
# Failed login attempts
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --filter-name "FailedConsoleLogins" \
  --filter-pattern '{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }' \
  --metric-transformations \
    metricName="FailedConsoleLogins",metricNamespace="SecurityMetrics",metricValue="1"

# Root account usage
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName="RootAccountUsage",metricNamespace="SecurityMetrics",metricValue="1"

# Unauthorized API calls
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --filter-name "UnauthorizedAPICalls" \
  --filter-pattern '{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }' \
  --metric-transformations \
    metricName="UnauthorizedAPICalls",metricNamespace="SecurityMetrics",metricValue="1"

# IAM policy changes
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --filter-name "IAMPolicyChanges" \
  --filter-pattern '{ ($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=AttachRolePolicy)||($.eventName=DetachRolePolicy)||($.eventName=AttachUserPolicy)||($.eventName=DetachUserPolicy)||($.eventName=AttachGroupPolicy)||($.eventName=DetachGroupPolicy) }' \
  --metric-transformations \
    metricName="IAMPolicyChanges",metricNamespace="SecurityMetrics",metricValue="1"
```

#### 6.2 Security Alarms

```bash
# Create SNS topic for security alerts
aws sns create-topic --name security-alerts

SECURITY_TOPIC_ARN=$(aws sns get-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:security-alerts \
  --query 'Attributes.TopicArn' --output text)

# Subscribe security team
aws sns subscribe \
  --topic-arn $SECURITY_TOPIC_ARN \
  --protocol email \
  --notification-endpoint security-team@company.com

# Create alarms
aws cloudwatch put-metric-alarm \
  --alarm-name "Security-FailedLogins" \
  --alarm-description "Multiple failed login attempts detected" \
  --metric-name "FailedConsoleLogins" \
  --namespace "SecurityMetrics" \
  --statistic "Sum" \
  --period 300 \
  --threshold 3 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \
  --alarm-actions $SECURITY_TOPIC_ARN

aws cloudwatch put-metric-alarm \
  --alarm-name "Security-RootAccountUsage" \
  --alarm-description "Root account usage detected" \
  --metric-name "RootAccountUsage" \
  --namespace "SecurityMetrics" \
  --statistic "Sum" \
  --period 300 \
  --threshold 1 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \
  --alarm-actions $SECURITY_TOPIC_ARN

aws cloudwatch put-metric-alarm \
  --alarm-name "Security-UnauthorizedAPICalls" \
  --alarm-description "Unauthorized API calls detected" \
  --metric-name "UnauthorizedAPICalls" \
  --namespace "SecurityMetrics" \
  --statistic "Sum" \
  --period 300 \
  --threshold 5 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \
  --alarm-actions $SECURITY_TOPIC_ARN
```

### 7. Data Encryption in Transit

#### 7.1 EKS Security Groups for HTTPS Only

```bash
# Create security group for HTTPS only traffic
aws ec2 create-security-group \
  --group-name retail-forecast-https-only \
  --description "Security group allowing HTTPS traffic only"

SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --group-names retail-forecast-https-only \
  --query 'SecurityGroups[0].GroupId' --output text)

# Allow HTTPS inbound
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow internal cluster communication
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 8080 \
  --source-group $SECURITY_GROUP_ID
```

#### 7.2 TLS Configuration for Applications

```yaml
# tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: retail-forecast-tls
  namespace: mlops
type: kubernetes.io/tls
data:
  tls.crt: # Base64 encoded certificate
  tls.key: # Base64 encoded private key
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-forecast-ingress-tls
  namespace: mlops
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/YOUR-CERT-ID
spec:
  tls:
  - hosts:
    - api.retail-forecast.com
    secretName: retail-forecast-tls
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
---
```

### 8. Compliance and Audit Reports

#### 8.1 Automated Compliance Checks

```python
# compliance_checker.py
import boto3
import json
from datetime import datetime, timedelta

class ComplianceChecker:
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.kms = boto3.client('kms')
        self.ecr = boto3.client('ecr')
        self.cloudtrail = boto3.client('cloudtrail')
    
    def check_s3_encryption(self, bucket_name):
        """Check if S3 bucket has encryption enabled"""
        try:
            response = self.s3.get_bucket_encryption(Bucket=bucket_name)
            encryption_config = response['ServerSideEncryptionConfiguration']
            
            for rule in encryption_config['Rules']:
                sse_algorithm = rule['ApplyServerSideEncryptionByDefault']['SSEAlgorithm']
                if sse_algorithm == 'aws:kms':
                    return {'status': 'COMPLIANT', 'encryption': 'KMS'}
                elif sse_algorithm == 'AES256':
                    return {'status': 'PARTIAL', 'encryption': 'AES256'}
            
            return {'status': 'NON_COMPLIANT', 'encryption': 'None'}
        except Exception as e:
            return {'status': 'ERROR', 'error': str(e)}
    
    def check_ecr_encryption(self, repository_name):
        """Check if ECR repository has encryption enabled"""
        try:
            response = self.ecr.describe_repositories(repositoryNames=[repository_name])
            repository = response['repositories'][0]
            
            if 'encryptionConfiguration' in repository:
                encryption_type = repository['encryptionConfiguration']['encryptionType']
                if encryption_type == 'KMS':
                    return {'status': 'COMPLIANT', 'encryption': 'KMS'}
                else:
                    return {'status': 'PARTIAL', 'encryption': encryption_type}
            
            return {'status': 'NON_COMPLIANT', 'encryption': 'None'}
        except Exception as e:
            return {'status': 'ERROR', 'error': str(e)}
    
    def check_cloudtrail_status(self, trail_name):
        """Check if CloudTrail is enabled and logging"""
        try:
            # Check trail status
            response = self.cloudtrail.get_trail_status(Name=trail_name)
            is_logging = response['IsLogging']
            
            # Check trail configuration
            trail_response = self.cloudtrail.describe_trails(trailNameList=[trail_name])
            trail = trail_response['trailList'][0]
            
            has_encryption = 'KMSKeyId' in trail
            is_multi_region = trail['IsMultiRegionTrail']
            has_log_validation = trail['LogFileValidationEnabled']
            
            compliance_score = sum([is_logging, has_encryption, is_multi_region, has_log_validation])
            
            return {
                'status': 'COMPLIANT' if compliance_score == 4 else 'PARTIAL',
                'logging': is_logging,
                'encryption': has_encryption,
                'multi_region': is_multi_region,
                'log_validation': has_log_validation,
                'score': f"{compliance_score}/4"
            }
        except Exception as e:
            return {'status': 'ERROR', 'error': str(e)}
    
    def generate_compliance_report(self):
        """Generate comprehensive compliance report"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'checks': {}
        }
        
        # Check S3 buckets
        buckets = ['retail-forecast-data-bucket', 'retail-forecast-artifacts-bucket']
        for bucket in buckets:
            report['checks'][f's3_{bucket}'] = self.check_s3_encryption(bucket)
        
        # Check ECR repositories
        repositories = ['retail-forecast']
        for repo in repositories:
            report['checks'][f'ecr_{repo}'] = self.check_ecr_encryption(repo)
        
        # Check CloudTrail
        report['checks']['cloudtrail'] = self.check_cloudtrail_status('retail-forecast-audit-trail')
        
        return report

# Usage
if __name__ == "__main__":
    checker = ComplianceChecker()
    report = checker.generate_compliance_report()
    
    # Save report
    with open(f"compliance_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json", 'w') as f:
        json.dump(report, f, indent=2)
    
    print(json.dumps(report, indent=2))
```

#### 8.2 Audit Log Analysis

```bash
# Query CloudTrail logs for specific activities
# Recent S3 access attempts
aws logs start-query \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, sourceIPAddress, userIdentity.type, eventName
    | filter eventSource = "s3.amazonaws.com"
    | filter eventName like /Put|Get|Delete/
    | sort @timestamp desc'

# Failed authentication attempts
aws logs start-query \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --start-time $(date -d '7 days ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, sourceIPAddress, userIdentity, eventName, errorCode
    | filter errorCode exists
    | filter errorCode like /UnauthorizedOperation|AccessDenied/
    | sort @timestamp desc'

# KMS key usage patterns
aws logs start-query \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --start-time $(date -d '30 days ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, userIdentity.type, eventName, resources
    | filter eventSource = "kms.amazonaws.com"
    | filter eventName like /Decrypt|Encrypt|GenerateDataKey/
    | stats count() by eventName, userIdentity.type'
```

### 9. Backup and Recovery for Security Components

#### 9.1 KMS Key Backup Strategy

```bash
# Create key material backup (for customer-managed keys)
aws kms get-parameters-for-import \
  --key-id $S3_KEY_ID \
  --wrapping-algorithm RSAES_OAEP_SHA_256 \
  --wrapping-key-spec RSA_2048

# Schedule automated key rotation
aws kms enable-key-rotation --key-id $S3_KEY_ID
aws kms enable-key-rotation --key-id $ECR_KEY_ID
aws kms enable-key-rotation --key-id $EKS_KEY_ID

# Verify rotation is enabled
aws kms get-key-rotation-status --key-id $S3_KEY_ID
```

#### 9.2 CloudTrail Log Backup

```bash
# Create lifecycle policy for CloudTrail logs
aws s3api put-bucket-lifecycle-configuration \
  --bucket $CLOUDTRAIL_BUCKET \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "CloudTrailLogRetention",
        "Status": "Enabled",
        "Filter": {
          "Prefix": "retail-forecast-logs/"
        },
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "STANDARD_IA"
          },
          {
            "Days": 90,
            "StorageClass": "GLACIER"
          },
          {
            "Days": 365,
            "StorageClass": "DEEP_ARCHIVE"
          }
        ]
      }
    ]
  }'

# Enable cross-region replication for disaster recovery
aws s3api put-bucket-replication \
  --bucket $CLOUDTRAIL_BUCKET \
  --replication-configuration file://replication-config.json
```

### 10. Security Validation and Testing

#### 10.1 Encryption Validation

```bash
# Test S3 encryption
echo "test data" > test-file.txt
aws s3 cp test-file.txt s3://retail-forecast-data-bucket/test/ \
  --server-side-encryption aws:kms \
  --ssekms-key-id alias/retail-forecast-s3-key

# Verify encryption
aws s3api head-object \
  --bucket retail-forecast-data-bucket \
  --key test/test-file.txt \
  --query 'ServerSideEncryption'

# Test ECR encryption
docker pull hello-world
docker tag hello-world YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-forecast:test
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-forecast:test

# Verify ECR encryption
aws ecr describe-images \
  --repository-name retail-forecast \
  --query 'imageDetails[0].imageScanFindingsSummary'
```

#### 10.2 Audit Trail Validation

```bash
# Generate test events
aws s3 ls s3://retail-forecast-data-bucket/
aws sagemaker list-training-jobs
kubectl get pods -n mlops

# Wait for logs to appear in CloudTrail (usually 5-15 minutes)
sleep 900

# Query for recent events
aws logs start-query \
  --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
  --start-time $(date -d '30 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, eventName, eventSource, sourceIPAddress
    | sort @timestamp desc
    | limit 20'
```

## Kết quả kỳ vọng

### ✅ Checklist Hoàn thành

- [ ] **KMS Keys**: Customer-managed KMS keys được tạo cho S3, ECR, EKS
- [ ] **S3 Encryption**: SSE-KMS được enable cho tất cả S3 buckets
- [ ] **ECR Encryption**: ECR repositories được mã hóa với KMS
- [ ] **EKS Encryption**: EKS secrets được mã hóa
- [ ] **CloudTrail**: Multi-region CloudTrail được enable với KMS encryption
- [ ] **Security Monitoring**: CloudWatch alarms cho security events
- [ ] **Compliance Checks**: Automated compliance validation
- [ ] **Audit Capabilities**: Log analysis và investigation tools
- [ ] **Backup Strategy**: Key rotation và log retention policies

### 📊 Verification Steps

1. **Các bucket S3 và repository ECR hiển thị trạng thái encryption = Enabled**
   ```bash
   # Check S3 encryption
   aws s3api get-bucket-encryption --bucket retail-forecast-data-bucket
   # Expected: SSE-KMS configuration with retail-forecast-s3-key
   
   # Check ECR encryption
   aws ecr describe-repositories --repository-names retail-forecast \
     --query 'repositories[0].encryptionConfiguration'
   # Expected: encryptionType: KMS
   ```

2. **CloudTrail có thể ghi nhận sự kiện mới**
   ```bash
   # Check CloudTrail status
   aws cloudtrail get-trail-status --name retail-forecast-audit-trail
   # Expected: IsLogging: true
   
   # Generate test event and verify
   aws s3 ls s3://retail-forecast-data-bucket/
   
   # Wait and check logs (15 minutes delay typical)
   aws logs start-query \
     --log-group-name "/aws/cloudtrail/retail-forecast-audit" \
     --start-time $(date -d '1 hour ago' +%s) \
     --end-time $(date +%s) \
     --query-string 'fields @timestamp, eventName | filter eventName = "ListObjects" | sort @timestamp desc | limit 5'
   ```

3. **Hệ thống đáp ứng yêu cầu bảo mật, audit, và điều tra sự cố**
   ```bash
   # Run compliance check
   python compliance_checker.py
   # Expected: All checks show COMPLIANT status
   
   # Test security alarms
   # Generate unauthorized access attempt
   aws s3 ls s3://some-restricted-bucket --region us-west-2
   
   # Check alarm status
   aws cloudwatch describe-alarms \
     --alarm-names "Security-UnauthorizedAPICalls" \
     --query 'MetricAlarms[0].StateValue'
   # Expected: ALARM state should trigger
   ```

### 🔍 Monitoring Commands

```bash
# Monitor encryption status
aws kms list-keys --query 'Keys[?KeyUsage==`ENCRYPT_DECRYPT`]'

# Check CloudTrail log delivery
aws cloudtrail get-trail-status --name retail-forecast-audit-trail \
  --query '[IsLogging,LatestDeliveryTime,LatestDeliveryError]'

# Monitor security metrics
aws cloudwatch get-metric-statistics \
  --namespace "SecurityMetrics" \
  --metric-name "UnauthorizedAPICalls" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --period 3600 \
  --statistics Sum

# Check key rotation status
aws kms get-key-rotation-status --key-id alias/retail-forecast-s3-key

# Verify backup retention
aws s3api get-bucket-lifecycle-configuration --bucket $CLOUDTRAIL_BUCKET
```

## Security Best Practices Summary

### 🔐 **Encryption Best Practices**
- Use customer-managed KMS keys for full control
- Enable automatic key rotation annually
- Implement least-privilege access to keys
- Monitor key usage patterns

### 📋 **Audit Best Practices**  
- Enable multi-region CloudTrail for comprehensive coverage
- Use advanced event selectors for granular logging
- Implement real-time security monitoring
- Regular compliance validation and reporting

### 🛡️ **Access Control**
- Enforce HTTPS-only communication
- Implement network segmentation
- Use IAM roles instead of access keys
- Regular access reviews and cleanup

### 📊 **Monitoring & Response**
- Automated security alerting
- Incident response playbooks
- Regular security assessments
- Backup and recovery procedures

---

**Next Step**: [Task 14: Final Integration & Testing](../14-integration-testing/)