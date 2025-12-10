---
title: "CI/CD Pipeline"
date: 2024-01-01T00:00:00Z
weight: 12
chapter: false
pre: "<b>11. </b>"
---

{{% notice info %}}
**🎯 Mục tiêu Task 13:**
{{% /notice %}}

Thiết lập pipeline CI/CD tự động cho toàn bộ vòng đời dự án MLOps Retail Prediction:

- CI (Continuous Integration): build & test code, build Docker image, push ECR
- CD (Continuous Delivery): huấn luyện lại model, cập nhật Model Registry, deploy phiên bản mới lên EKS
- Monitoring hook: rollback khi API hoặc model có lỗi (CloudWatch trigger)

→ Đảm bảo triển khai liên tục, giảm lỗi thủ công, tiết kiệm thời gian và chi phí.

📥 **Input từ các Task trước:**
- **Task 6 (ECR Container Registry):** Repository URIs, lifecycle policies, image scanning and push commands; credentials and ECR access for CI runners
- **Task 8 (API Deployment on EKS):** Kubernetes manifests, service & deployment names, healthcheck endpoints, HPA and ServiceAccount/IRSA details used by CD stage
- **Task 10 (CloudWatch Monitoring):** Log groups, alarms, dashboards and Container Insights configuration used for deployment verification and automated rollback triggers
- **Task 2 (IAM Roles & Audit):** IAM roles, OIDC provider configuration and least-privilege policies for CI/CD runners (GitHub Actions / Jenkins) and SageMaker execution

## 1. Cấu trúc pipeline tổng quát

Pipeline CI/CD của dự án Retail Prediction sẽ tự động hóa toàn bộ quy trình từ commit code đến deploy lên production, bao gồm cả việc huấn luyện lại model khi cần thiết.

Pipeline chia thành 3 môi trường:
- **DEV**: Build, test và validate code
- **STAGING**: Huấn luyện và đánh giá model
- **PROD**: Deploy và monitor trên production

### 2. IAM Roles và Permissions

#### 2.1 Create CI/CD Service Role

```bash
# Create trust policy for CI/CD role
cat > cicd-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "ec2.amazonaws.com",
          "codebuild.amazonaws.com",
          "codepipeline.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:user/jenkins-user"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name RetailForecastCICDRole \
  --assume-role-policy-document file://cicd-trust-policy.json \
  --description "Role for Retail Forecast CI/CD Pipeline"
```

#### 2.2 CI/CD IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::retail-forecast-data-bucket",
        "arn:aws:s3:::retail-forecast-data-bucket/*",
        "arn:aws:s3:::retail-forecast-artifacts-bucket",
        "arn:aws:s3:::retail-forecast-artifacts-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:DescribeTrainingJob",
        "sagemaker:StopTrainingJob",
        "sagemaker:CreateModel",
        "sagemaker:CreateModelPackage",
        "sagemaker:CreateModelPackageGroup",
        "sagemaker:DescribeModelPackage",
        "sagemaker:UpdateModelPackage",
        "sagemaker:ListModelPackages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:CreateGrant"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:YOUR_ACCOUNT_ID:key/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 2.3 Attach Policies

```bash
# Create and attach the policy
aws iam create-policy \
  --policy-name RetailForecastCICDPolicy \
  --policy-document file://cicd-policy.json

aws iam attach-role-policy \
  --role-name RetailForecastCICDRole \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/RetailForecastCICDPolicy

# Attach additional managed policies
aws iam attach-role-policy \
  --role-name RetailForecastCICDRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy \
  --role-name RetailForecastCICDRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
```

### 3. Jenkins Pipeline Setup

#### 3.1 Jenkins Installation on EC2

```bash
# Launch EC2 instance for Jenkins
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1d0 \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --security-group-ids sg-jenkins \
  --subnet-id subnet-12345678 \
  --iam-instance-profile Name=JenkinsInstanceProfile \
  --user-data file://jenkins-install.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Jenkins-Server},{Key=Project,Value=RetailForecast}]'
```

#### 3.2 Jenkins Installation Script

```bash
#!/bin/bash
# jenkins-install.sh

# Update system
yum update -y

# Install Docker
yum install -y docker
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Install Jenkins
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum upgrade -y
yum install -y java-11-openjdk jenkins

# Start Jenkins
systemctl start jenkins
systemctl enable jenkins

# Install kubectl
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Install Python and dependencies
yum install -y python3 python3-pip
pip3 install boto3 sagemaker pandas scikit-learn

# Configure Jenkins user for Docker
usermod -a -G docker jenkins
systemctl restart jenkins

echo "Jenkins installation completed!"
echo "Access Jenkins at: http://$(curl -s http://169.254.169.254/latest/meta-data/public-ip):8080"
echo "Initial admin password: $(cat /var/lib/jenkins/secrets/initialAdminPassword)"
```

#### 3.3 Jenkinsfile for ML Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '123456789012'
        ECR_REPOSITORY = 'retail-forecast'
        EKS_CLUSTER_NAME = 'retail-forecast-cluster'
        NAMESPACE = 'mlops'
        S3_DATA_BUCKET = 'retail-forecast-data-bucket'
        S3_ARTIFACTS_BUCKET = 'retail-forecast-artifacts-bucket'
        MODEL_PACKAGE_GROUP = 'retail-forecast-model-group'
    }
    
    parameters {
        choice(
            name: 'DEPLOY_ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target deployment environment'
        )
        booleanParam(
            name: 'RETRAIN_MODEL',
            defaultValue: false,
            description: 'Force model retraining'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
    }
    
    stages {
        stage('Setup') {
            steps {
                script {
                    // Clean workspace
                    cleanWs()
                    
                    // Checkout code
                    checkout scm
                    
                    // Set build info
                    env.BUILD_VERSION = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    env.IMAGE_TAG = "v${env.BUILD_VERSION}"
                    
                    echo "Build Version: ${env.BUILD_VERSION}"
                    echo "Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }
        
        stage('Environment Setup') {
            steps {
                script {
                    // Install Python dependencies
                    sh '''
                        python3 -m venv venv
                        source venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        pip install pytest pytest-cov flake8
                    '''
                    
                    // Configure AWS credentials
                    sh '''
                        aws sts get-caller-identity
                        aws configure set region $AWS_DEFAULT_REGION
                    '''
                    
                    // Configure kubectl
                    sh '''
                        aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_DEFAULT_REGION
                        kubectl cluster-info
                    '''
                }
            }
        }
        
        stage('Code Quality & Testing') {
            when {
                not { params.SKIP_TESTS }
            }
            parallel {
                stage('Linting') {
                    steps {
                        sh '''
                            source venv/bin/activate
                            flake8 src/ --max-line-length=88 --exclude=venv
                        '''
                    }
                }
                
                stage('Unit Tests') {
                    steps {
                        sh '''
                            source venv/bin/activate
                            pytest tests/unit/ -v --cov=src --cov-report=xml --cov-report=html
                        '''
                        
                        // Publish test results
                        publishTestResults testResultsPattern: 'test-results.xml'
                        publishCoverage adapters: [coberturaAdapter('coverage.xml')], sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        sh '''
                            source venv/bin/activate
                            pytest tests/integration/ -v
                        '''
                    }
                }
            }
        }
        
        stage('Data Validation') {
            steps {
                script {
                    sh '''
                        source venv/bin/activate
                        python scripts/validate_data.py \
                            --bucket $S3_DATA_BUCKET \
                            --key training-data/train.csv \
                            --output data-validation-report.json
                    '''
                    
                    // Archive validation report
                    archiveArtifacts artifacts: 'data-validation-report.json', fingerprint: true
                }
            }
        }
        
        stage('Model Training') {
            when {
                anyOf {
                    params.RETRAIN_MODEL
                    changeset "src/training/**"
                    changeset "data/**"
                }
            }
            steps {
                script {
                    sh '''
                        source venv/bin/activate
                        python scripts/trigger_training.py \
                            --job-name "retail-forecast-training-${BUILD_VERSION}" \
                            --data-bucket $S3_DATA_BUCKET \
                            --output-bucket $S3_ARTIFACTS_BUCKET \
                            --role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/SageMakerExecutionRole
                    '''
                    
                    // Wait for training completion
                    sh '''
                        source venv/bin/activate
                        python scripts/wait_for_training.py \
                            --job-name "retail-forecast-training-${BUILD_VERSION}" \
                            --timeout 3600
                    '''
                }
            }
        }
        
        stage('Model Validation & Registration') {
            when {
                anyOf {
                    params.RETRAIN_MODEL
                    changeset "src/training/**"
                }
            }
            steps {
                script {
                    // Validate model performance
                    sh '''
                        source venv/bin/activate
                        python scripts/validate_model.py \
                            --job-name "retail-forecast-training-${BUILD_VERSION}" \
                            --baseline-accuracy 0.85 \
                            --output model-validation-report.json
                    '''
                    
                    // Register model if validation passes
                    sh '''
                        source venv/bin/activate
                        python scripts/register_model.py \
                            --job-name "retail-forecast-training-${BUILD_VERSION}" \
                            --model-package-group $MODEL_PACKAGE_GROUP \
                            --approval-status "PendingManualApproval" \
                            --model-version $BUILD_VERSION
                    '''
                    
                    archiveArtifacts artifacts: 'model-validation-report.json', fingerprint: true
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    // Build Docker image
                    sh '''
                        # Get latest approved model
                        MODEL_URI=$(python scripts/get_latest_model.py --model-package-group $MODEL_PACKAGE_GROUP)
                        echo "Using model: $MODEL_URI"
                        
                        # Build image with model URI
                        docker build \
                            --build-arg MODEL_URI=$MODEL_URI \
                            --build-arg BUILD_VERSION=$BUILD_VERSION \
                            -t $ECR_REPOSITORY:$IMAGE_TAG \
                            -t $ECR_REPOSITORY:latest \
                            .
                    '''
                    
                    // Security scan
                    sh '''
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            -v $(pwd):/root/.cache/ \
                            aquasec/trivy:latest image \
                            --exit-code 1 \
                            --severity HIGH,CRITICAL \
                            $ECR_REPOSITORY:$IMAGE_TAG
                    '''
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    sh '''
                        # Login to ECR
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                            docker login --username AWS --password-stdin \
                            $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
                        
                        # Tag images
                        docker tag $ECR_REPOSITORY:$IMAGE_TAG \
                            $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG
                        
                        docker tag $ECR_REPOSITORY:latest \
                            $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY:latest
                        
                        # Push images
                        docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG
                        docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY:latest
                    '''
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update Kubernetes deployment
                    sh '''
                        # Update deployment image
                        kubectl set image deployment/retail-forecast-api \
                            retail-forecast-api=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG \
                            -n $NAMESPACE
                        
                        # Wait for rollout
                        kubectl rollout status deployment/retail-forecast-api -n $NAMESPACE --timeout=600s
                        
                        # Verify deployment
                        kubectl get pods -n $NAMESPACE -l app=retail-forecast-api
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh '''
                        # Get service endpoint
                        ENDPOINT=$(kubectl get service retail-forecast-service -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                        
                        if [ -z "$ENDPOINT" ]; then
                            ENDPOINT=$(kubectl get service retail-forecast-service -n $NAMESPACE -o jsonpath='{.spec.clusterIP}')
                            kubectl port-forward service/retail-forecast-service 8080:80 -n $NAMESPACE &
                            ENDPOINT="localhost:8080"
                            PORT_FORWARD_PID=$!
                        fi
                        
                        # Wait for service to be ready
                        echo "Waiting for service to be ready..."
                        for i in {1..30}; do
                            if curl -f http://$ENDPOINT/healthz; then
                                echo "Service is healthy!"
                                break
                            fi
                            echo "Attempt $i/30 failed, retrying in 10 seconds..."
                            sleep 10
                        done
                        
                        # Test prediction endpoint
                        curl -X POST http://$ENDPOINT/predict \
                            -H "Content-Type: application/json" \
                            -d '{"features": {"store_id": 1, "product_id": 123, "price": 29.99}}'
                        
                        # Kill port-forward if used
                        if [ ! -z "$PORT_FORWARD_PID" ]; then
                            kill $PORT_FORWARD_PID
                        fi
                    '''
                }
            }
        }
        
        stage('Performance Testing') {
            when {
                environment name: 'DEPLOY_ENVIRONMENT', value: 'prod'
            }
            steps {
                script {
                    sh '''
                        # Run load test
                        python scripts/load_test.py \
                            --endpoint http://$ENDPOINT \
                            --duration 300 \
                            --concurrent-users 10 \
                            --output load-test-report.json
                    '''
                    
                    archiveArtifacts artifacts: 'load-test-report.json', fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            // Clean up
            sh '''
                docker system prune -f
                rm -rf venv
            '''
            
            // Archive logs
            archiveArtifacts artifacts: 'logs/**', allowEmptyArchive: true
        }
        
        success {
            script {
                // Send success notification
                sh '''
                    aws sns publish \
                        --topic-arn arn:aws:sns:$AWS_DEFAULT_REGION:$AWS_ACCOUNT_ID:deployment-notifications \
                        --message "✅ Deployment successful for build $BUILD_VERSION" \
                        --subject "Retail Forecast Deployment Success"
                '''
            }
        }
        
        failure {
            script {
                // Send failure notification
                sh '''
                    aws sns publish \
                        --topic-arn arn:aws:sns:$AWS_DEFAULT_REGION:$AWS_ACCOUNT_ID:deployment-notifications \
                        --message "❌ Deployment failed for build $BUILD_VERSION. Check Jenkins logs." \
                        --subject "Retail Forecast Deployment Failed"
                '''
                
                // Rollback on production failure
                if (params.DEPLOY_ENVIRONMENT == 'prod') {
                    sh '''
                        echo "Rolling back production deployment..."
                        kubectl rollout undo deployment/retail-forecast-api -n $NAMESPACE
                        kubectl rollout status deployment/retail-forecast-api -n $NAMESPACE
                    '''
                }
            }
        }
    }
}
```

## 2. GitHub Actions CI/CD Pipeline

Chúng ta sẽ sử dụng GitHub Actions để xây dựng pipeline CI/CD cho dự án MLOps Retail Prediction vì khả năng tích hợp sẵn với GitHub repository và tính linh hoạt cao.

### 2.1 Cấu hình workflow file

Tạo file `.github/workflows/mlops-pipeline.yml`:

```yaml
# .github/workflows/mlops-pipeline.yml
name: MLOps Retail Prediction Pipeline

on:
  push:
    branches: [ main, develop ]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [ main ]
  schedule:
    # Chạy mỗi tuần vào thứ 2 để kiểm tra độ chính xác của model
    - cron: '0 2 * * 1'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Môi trường triển khai'
        required: true
        default: 'dev'
        type: choice
        options:
        - dev
        - staging
        - prod
      retrain_model:
        description: 'Huấn luyện lại model'
        required: false
        default: false
        type: boolean
      deploy_only:
        description: 'Chỉ triển khai, không build/huấn luyện mới'
        required: false
        default: false
        type: boolean

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: retail-forecast
  EKS_CLUSTER_NAME: retail-forecast-cluster
  S3_DATA_BUCKET: retail-forecast-data
  S3_MODEL_BUCKET: retail-forecast-models
  MODEL_PACKAGE_GROUP: retail-forecast-models

permissions:
  id-token: write   # Cần thiết cho OIDC với AWS
  contents: read    # Cần thiết để checkout code

jobs:
  setup:
    name: Setup Pipeline
    runs-on: ubuntu-latest
    outputs:
      build-id: ${{ steps.generate-id.outputs.build_id }}
      environment: ${{ steps.set-env.outputs.environment }}
      should-train: ${{ steps.set-env.outputs.should_train }}
      should-deploy: ${{ steps.set-env.outputs.should_deploy }}
    
    steps:
    - name: Generate build ID
      id: generate-id
      run: |
        BUILD_ID="build-${GITHUB_RUN_NUMBER}-${GITHUB_SHA::7}"
        echo "build_id=$BUILD_ID" >> $GITHUB_OUTPUT
        echo "Build ID: $BUILD_ID"
    
    - name: Set environment variables
      id: set-env
      run: |
        # Xác định môi trường
        if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
          ENV="${{ github.event.inputs.environment }}"
        elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
          ENV="prod"
        elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
          ENV="staging"
        else
          ENV="dev"
        fi
        echo "environment=$ENV" >> $GITHUB_OUTPUT
        
        # Xác định có nên huấn luyện model hay không
        if [[ "${{ github.event.inputs.retrain_model }}" == "true" ]] || \
           [[ "${{ github.event_name }}" == "schedule" ]]; then
          SHOULD_TRAIN="true"
        else
          SHOULD_TRAIN="false"
        fi
        echo "should_train=$SHOULD_TRAIN" >> $GITHUB_OUTPUT
        
        # Xác định có nên deploy hay không
        if [[ "${{ github.event.inputs.deploy_only }}" == "true" ]] || \
           [[ "$ENV" == "prod" && "${{ github.ref }}" == "refs/heads/main" ]] || \
           [[ "$ENV" == "staging" && "${{ github.ref }}" == "refs/heads/develop" ]]; then
          SHOULD_DEPLOY="true"
        else
          SHOULD_DEPLOY="false"
        fi
        echo "should_deploy=$SHOULD_DEPLOY" >> $GITHUB_OUTPUT
        
        # In thông tin
        echo "Environment: $ENV"
        echo "Should train model: $SHOULD_TRAIN"
        echo "Should deploy: $SHOULD_DEPLOY"

  test:
    name: Code Quality & Testing
    needs: setup
    runs-on: ubuntu-latest
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
        cache: 'pip'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r core/requirements.txt
        pip install -r server/requirements.txt
        pip install pytest pytest-cov pylint black
    
    - name: Code formatting check
      run: |
        black --check core/ server/
    
    - name: Lint code
      run: |
        pylint --disable=C0111,C0103 core/ server/
    
    - name: Run tests
      run: |
        pytest tests/ -v --cov=core --cov=server --cov-report=xml
    
    - name: Upload coverage report
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        fail_ci_if_error: false

  data_validation:
    name: Data Validation
    needs: [setup, test]
    runs-on: ubuntu-latest
    if: needs.setup.outputs.should_train == 'true'
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
        cache: 'pip'
    
    - name: Install dependencies
      run: |
        pip install pandas boto3 great_expectations
    
    - name: Validate training data
      run: |
        python aws/script/validate_data.py \
          --bucket ${{ env.S3_DATA_BUCKET }} \
          --key training/sales_data.csv \
          --output validation_report.json
    
    - name: Upload validation report
      uses: actions/upload-artifact@v3
      with:
        name: data-validation-report
        path: validation_report.json

  model_training:
    name: Model Training
    needs: [setup, test, data_validation]
    runs-on: ubuntu-latest
    if: needs.setup.outputs.should_train == 'true' && success()
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
        cache: 'pip'
    
    - name: Install dependencies
      run: |
        pip install boto3 sagemaker pandas scikit-learn
    
    - name: Create training job
      id: training
      run: |
        python aws/script/create_training_job.py \
          --job-name "retail-forecast-${{ needs.setup.outputs.build-id }}" \
          --data-bucket ${{ env.S3_DATA_BUCKET }} \
          --output-bucket ${{ env.S3_MODEL_BUCKET }} \
          --instance-type ml.m5.large \
          --hyperparameters "{\"n_estimators\":\"200\",\"max_depth\":\"10\"}"
      
    - name: Wait for training completion
      run: |
        aws sagemaker wait training-job-completed-or-stopped \
          --training-job-name "retail-forecast-${{ needs.setup.outputs.build-id }}"
        
        STATUS=$(aws sagemaker describe-training-job \
          --training-job-name "retail-forecast-${{ needs.setup.outputs.build-id }}" \
          --query 'TrainingJobStatus' --output text)
          
        if [ "$STATUS" != "Completed" ]; then
          echo "Training failed with status: $STATUS"
          exit 1
        fi

  model_evaluation:
    name: Model Evaluation & Registration
    needs: [setup, model_training]
    runs-on: ubuntu-latest
    outputs:
      model_approved: ${{ steps.evaluate.outputs.model_approved }}
      model_version: ${{ steps.register.outputs.model_version }}
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
        cache: 'pip'
    
    - name: Install dependencies
      run: |
        pip install boto3 sagemaker pandas scikit-learn matplotlib seaborn
    
    - name: Evaluate model
      id: evaluate
      run: |
        python aws/script/processing_evaluate.py \
          --job-name "retail-forecast-${{ needs.setup.outputs.build-id }}" \
          --evaluation-data s3://${{ env.S3_DATA_BUCKET }}/validation/sales_data.csv \
          --baseline-metrics s3://${{ env.S3_MODEL_BUCKET }}/baselines/metrics.json \
          --threshold 0.85
          
        # Check result
        if [ -f "model_approved.txt" ]; then
          MODEL_APPROVED=$(cat model_approved.txt)
          echo "model_approved=$MODEL_APPROVED" >> $GITHUB_OUTPUT
          echo "Model evaluation result: $MODEL_APPROVED"
        else
          echo "model_approved=false" >> $GITHUB_OUTPUT
          echo "Model evaluation failed"
          exit 1
        fi
    
    - name: Register model
      id: register
      if: steps.evaluate.outputs.model_approved == 'true'
      run: |
        MODEL_VERSION=$(python aws/script/register_model.py \
          --job-name "retail-forecast-${{ needs.setup.outputs.build-id }}" \
          --model-package-group ${{ env.MODEL_PACKAGE_GROUP }} \
          --approval-status "Approved")
          
        echo "model_version=$MODEL_VERSION" >> $GITHUB_OUTPUT
        echo "Model registered with version: $MODEL_VERSION"
    
    - name: Upload evaluation results
      uses: actions/upload-artifact@v3
      with:
        name: model-evaluation-report
        path: |
          evaluation_results.json
          plots/*.png

  build_docker:
    name: Build & Push Docker Image
    needs: [setup, test]
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.image_tag }}
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build and push Docker image
      id: build
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      run: |
        # Set tag
        IMAGE_TAG="${{ needs.setup.outputs.build-id }}"
        echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
        
        # Get latest model URI (or use placeholder for dev environment)
        if [[ "${{ needs.setup.outputs.environment }}" == "dev" ]]; then
          MODEL_URI="placeholder"
        else
          MODEL_URI=$(aws sagemaker list-model-packages \
            --model-package-group-name ${{ env.MODEL_PACKAGE_GROUP }} \
            --sort-by CreationTime --sort-order Descending --max-items 1 \
            --query 'ModelPackageSummaries[0].ModelPackageArn' --output text)
        fi
        
        echo "Using model URI: $MODEL_URI"
        
        # Build image
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
          --build-arg MODEL_URI=$MODEL_URI \
          --build-arg BUILD_ID=$IMAGE_TAG \
          --build-arg ENV=${{ needs.setup.outputs.environment }} \
          server/
        
        # Also tag as latest-{environment}
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
          $ECR_REGISTRY/$ECR_REPOSITORY:latest-${{ needs.setup.outputs.environment }}
        
        # Security scan
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image --severity HIGH,CRITICAL \
          $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        
        # Push images
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest-${{ needs.setup.outputs.environment }}
        
        echo "Image built and pushed: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

  deploy_eks:
    name: Deploy to EKS
    needs: [setup, test, build_docker, model_evaluation]
    runs-on: ubuntu-latest
    if: needs.setup.outputs.should_deploy == 'true'
    environment:
      name: ${{ needs.setup.outputs.environment }}
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER_NAME }} --region ${{ env.AWS_REGION }}
    
    - name: Check if need to deploy new model
      id: check-model
      run: |
        if [[ "${{ needs.model_evaluation.outputs.model_approved }}" == "true" ]]; then
          echo "deploy_new_model=true" >> $GITHUB_OUTPUT
        else
          echo "deploy_new_model=false" >> $GITHUB_OUTPUT
        fi
    
    - name: Deploy to EKS
      run: |
        # Set environment variables
        NAMESPACE="retail-forecast-${{ needs.setup.outputs.environment }}"
        IMAGE_TAG="${{ needs.build_docker.outputs.image_tag }}"
        ECR_REGISTRY="${{ steps.login-ecr.outputs.registry }}"
        
        # Update kustomization file with new image
        cd aws/k8s
        kustomize edit set image retail-forecast-api=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        
        # Apply changes
        kubectl apply -k overlays/${{ needs.setup.outputs.environment }}/ --namespace $NAMESPACE
        
        # Wait for deployment to complete
        kubectl rollout status deployment/retail-forecast-api -n $NAMESPACE --timeout=300s
    
    - name: Create SageMaker endpoint (if new model approved)
      if: steps.check-model.outputs.deploy_new_model == 'true'
      run: |
        python aws/script/deploy_endpoint.py \
          --model-package-arn "arn:aws:sagemaker:${{ env.AWS_REGION }}:${{ secrets.AWS_ACCOUNT_ID }}:model-package/${{ env.MODEL_PACKAGE_GROUP }}/${{ needs.model_evaluation.outputs.model_version }}" \
          --endpoint-name "retail-forecast-${{ needs.setup.outputs.environment }}" \
          --instance-type ml.t2.medium \
          --instance-count 1
    
    - name: Health check
      run: |
        # Get service endpoint
        NAMESPACE="retail-forecast-${{ needs.setup.outputs.environment }}"
        SERVICE_HOST=$(kubectl get ingress -n $NAMESPACE retail-forecast-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
        
        # If running locally, use port-forwarding
        if [ -z "$SERVICE_HOST" ]; then
          echo "No external hostname found, using port-forwarding"
          kubectl port-forward svc/retail-forecast-service 8080:80 -n $NAMESPACE &
          sleep 5
          SERVICE_HOST="localhost:8080"
        fi
        
        # Check health endpoint
        echo "Testing health endpoint: http://$SERVICE_HOST/health"
        HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://$SERVICE_HOST/health)
        
        if [ "$HEALTH_STATUS" -eq 200 ]; then
          echo "Health check passed: $HEALTH_STATUS"
        else
          echo "Health check failed: $HEALTH_STATUS"
          exit 1
        fi
        
        # Test prediction endpoint
        echo "Testing prediction endpoint"
        PREDICTION_RESULT=$(curl -s -X POST \
          -H "Content-Type: application/json" \
          -d '{"store_id": 1, "item_id": 123, "date": "2023-06-15"}' \
          http://$SERVICE_HOST/predict)
        
        echo "Prediction result: $PREDICTION_RESULT"
        
        # Check if response contains prediction
        if [[ $PREDICTION_RESULT == *"prediction"* ]]; then
          echo "Prediction endpoint working correctly"
        else
          echo "Prediction endpoint not returning expected response"
          exit 1
        fi

  monitoring:
    name: Setup Monitoring
    needs: [setup, deploy_eks]
    runs-on: ubuntu-latest
    if: always() && needs.deploy_eks.result == 'success'
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup CloudWatch alarms for new deployment
      run: |
        # Create/update CloudWatch alarms for API and model performance
        NAMESPACE="retail-forecast-${{ needs.setup.outputs.environment }}"
        DEPLOYMENT_ID="${{ needs.setup.outputs.build-id }}"
        
        aws cloudwatch put-metric-alarm \
          --alarm-name "RetailForecast-API-Error-Rate-$NAMESPACE" \
          --alarm-description "Monitor error rate for Retail Forecast API" \
          --metric-name "ErrorRate" \
          --namespace "RetailForecast/$NAMESPACE" \
          --statistic "Average" \
          --period 60 \
          --threshold 5 \
          --comparison-operator "GreaterThanThreshold" \
          --evaluation-periods 3 \
          --alarm-actions "arn:aws:sns:${{ env.AWS_REGION }}:${{ secrets.AWS_ACCOUNT_ID }}:retail-forecast-alerts" \
          --dimensions "Name=DeploymentId,Value=$DEPLOYMENT_ID"
        
        aws cloudwatch put-metric-alarm \
          --alarm-name "RetailForecast-API-Latency-$NAMESPACE" \
          --alarm-description "Monitor latency for Retail Forecast API" \
          --metric-name "Latency" \
          --namespace "RetailForecast/$NAMESPACE" \
          --statistic "Average" \
          --period 60 \
          --threshold 1000 \
          --comparison-operator "GreaterThanThreshold" \
          --evaluation-periods 3 \
          --alarm-actions "arn:aws:sns:${{ env.AWS_REGION }}:${{ secrets.AWS_ACCOUNT_ID }}:retail-forecast-alerts" \
          --dimensions "Name=DeploymentId,Value=$DEPLOYMENT_ID"
        
        echo "CloudWatch alarms configured successfully"

  notify:
    name: Send Notifications
    needs: [setup, deploy_eks, monitoring]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
    - name: Determine workflow status
      id: status
      run: |
        if [[ "${{ needs.deploy_eks.result }}" == "success" ]]; then
          echo "result=success" >> $GITHUB_OUTPUT
          echo "message=✅ Deployment successful for ${{ needs.setup.outputs.environment }} environment (Build ${{ needs.setup.outputs.build-id }})" >> $GITHUB_OUTPUT
        elif [[ "${{ needs.deploy_eks.result }}" == "failure" ]]; then
          echo "result=failure" >> $GITHUB_OUTPUT
          echo "message=❌ Deployment failed for ${{ needs.setup.outputs.environment }} environment (Build ${{ needs.setup.outputs.build-id }})" >> $GITHUB_OUTPUT
        else
          echo "result=skipped" >> $GITHUB_OUTPUT
          echo "message=ℹ️ Deployment skipped for ${{ needs.setup.outputs.environment }} environment (Build ${{ needs.setup.outputs.build-id }})" >> $GITHUB_OUTPUT
        fi
    
    - name: Configure AWS credentials
      if: steps.status.outputs.result != 'skipped'
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Send SNS notification
      if: steps.status.outputs.result != 'skipped'
      run: |
        aws sns publish \
          --topic-arn "arn:aws:sns:${{ env.AWS_REGION }}:${{ secrets.AWS_ACCOUNT_ID }}:retail-forecast-alerts" \
          --subject "Retail Forecast Deployment: ${{ steps.status.outputs.result }}" \
          --message "${{ steps.status.outputs.message }}"
```

## 3. IAM Role và Quyền hạn

Để GitHub Actions có thể tương tác với các dịch vụ AWS, chúng ta cần thiết lập IAM Role với quyền hạn phù hợp và sử dụng OIDC để xác thực.

### 3.1 Tạo IAM Role cho GitHub Actions

```bash
# Tạo IAM Role cho GitHub Actions
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:*"
        }
      }
    }
  ]
}
EOF

# Tạo role
aws iam create-role --role-name GitHubActionsRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for GitHub Actions CI/CD pipeline"
```

### 3.2 IAM Policy cho CI/CD Pipeline

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::retail-forecast-data*",
        "arn:aws:s3:::retail-forecast-data*/*",
        "arn:aws:s3:::retail-forecast-models*",
        "arn:aws:s3:::retail-forecast-models*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:DescribeTrainingJob",
        "sagemaker:StopTrainingJob",
        "sagemaker:ListTrainingJobs",
        "sagemaker:CreateProcessingJob",
        "sagemaker:DescribeProcessingJob",
        "sagemaker:StopProcessingJob",
        "sagemaker:CreateModel",
        "sagemaker:DeleteModel",
        "sagemaker:DescribeModel",
        "sagemaker:CreateEndpoint",
        "sagemaker:DescribeEndpoint",
        "sagemaker:DeleteEndpoint",
        "sagemaker:UpdateEndpoint",
        "sagemaker:CreateEndpointConfig",
        "sagemaker:DeleteEndpointConfig",
        "sagemaker:DescribeEndpointConfig",
        "sagemaker:CreateModelPackageGroup",
        "sagemaker:DescribeModelPackageGroup",
        "sagemaker:ListModelPackageGroups",
        "sagemaker:CreateModelPackage",
        "sagemaker:DescribeModelPackage",
        "sagemaker:ListModelPackages",
        "sagemaker:UpdateModelPackage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:ListImages",
        "ecr:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricData",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DeleteAlarms"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:*:*:retail-forecast-*"
    }
  ]
}
```

### 3.3 Gắn Policy cho Role

```bash
# Tạo và gắn policy
aws iam create-policy \
  --policy-name GitHubActionsPolicy \
  --policy-document file://github-actions-policy.json

aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/GitHubActionsPolicy

# Gắn thêm các managed policy cần thiết
aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonECR-FullAccess
```

## 4. Script hỗ trợ cho Pipeline

### 4.1 Script tạo Training Job

```python
# aws/script/create_training_job.py
import boto3
import argparse
import json
import os
from datetime import datetime

def create_training_job(job_name, data_bucket, output_bucket, 
                        instance_type='ml.m5.large', 
                        hyperparameters=None):
    """
    Tạo SageMaker training job cho Retail Forecast model
    """
    sagemaker = boto3.client('sagemaker')
    
    if hyperparameters is None:
        hyperparameters = {
            'n_estimators': '100',
            'max_depth': '10',
            'random_state': '42'
        }
    elif isinstance(hyperparameters, str):
        hyperparameters = json.loads(hyperparameters)
    
    # Lấy SageMaker execution role từ môi trường hoặc tạo mới
    role_arn = os.environ.get('SAGEMAKER_ROLE_ARN')
    if not role_arn:
        # Tìm role mặc định
        iam = boto3.client('iam')
        paginator = iam.get_paginator('list_roles')
        for page in paginator.paginate():
            for role in page['Roles']:
                if 'AmazonSageMaker-ExecutionRole' in role['RoleName']:
                    role_arn = role['Arn']
                    break
    
    if not role_arn:
        raise ValueError("Không tìm thấy SageMaker execution role")
    
    # Cấu hình training job
    training_params = {
        'TrainingJobName': job_name,
        'HyperParameters': hyperparameters,
        'AlgorithmSpecification': {
            'TrainingImage': '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-scikit-learn:1.0-1-cpu-py3',
            'TrainingInputMode': 'File'
        },
        'RoleArn': role_arn,
        'InputDataConfig': [
            {
                'ChannelName': 'train',
                'DataSource': {
                    'S3DataSource': {
                        'S3DataType': 'S3Prefix',
                        'S3Uri': f's3://{data_bucket}/training/',
                        'S3DataDistributionType': 'FullyReplicated'
                    }
                },
                'ContentType': 'text/csv'
            },
            {
                'ChannelName': 'validation',
                'DataSource': {
                    'S3DataSource': {
                        'S3DataType': 'S3Prefix',
                        'S3Uri': f's3://{data_bucket}/validation/',
                        'S3DataDistributionType': 'FullyReplicated'
                    }
                },
                'ContentType': 'text/csv'
            }
        ],
        'OutputDataConfig': {
            'S3OutputPath': f's3://{output_bucket}/models/'
        },
        'ResourceConfig': {
            'InstanceType': instance_type,
            'InstanceCount': 1,
            'VolumeSizeInGB': 30
        },
        'StoppingCondition': {
            'MaxRuntimeInSeconds': 3600
        },
        'Tags': [
            {'Key': 'Project', 'Value': 'RetailForecast'},
            {'Key': 'CreatedBy', 'Value': 'GitHubActions'},
            {'Key': 'Environment', 'Value': os.environ.get('ENVIRONMENT', 'dev')}
        ]
    }
    
    # Tạo training job
    response = sagemaker.create_training_job(**training_params)
    print(f"Training job created: {job_name}")
    print(f"ARN: {response['TrainingJobArn']}")
    
    return response['TrainingJobArn']

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Create SageMaker training job')
    parser.add_argument('--job-name', type=str, required=True, 
                        help='Name for the training job')
    parser.add_argument('--data-bucket', type=str, required=True,
                        help='S3 bucket containing training data')
    parser.add_argument('--output-bucket', type=str, required=True,
                        help='S3 bucket for output artifacts')
    parser.add_argument('--instance-type', type=str, default='ml.m5.large',
                        help='SageMaker instance type')
    parser.add_argument('--hyperparameters', type=str, default=None,
                        help='JSON string of hyperparameters')
    
    args = parser.parse_args()
    
    create_training_job(
        args.job_name,
        args.data_bucket,
        args.output_bucket,
        args.instance_type,
        args.hyperparameters
    )
```

### 4.2 Script đánh giá Model

```python
# aws/script/processing_evaluate.py
import boto3
import argparse
import json
import os
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score

def evaluate_model(job_name, evaluation_data, baseline_metrics=None, threshold=0.85):
    """Đánh giá model và so sánh với baseline metrics"""
    
    # Tạo thư mục cho plots
    os.makedirs('plots', exist_ok=True)
    
    sagemaker = boto3.client('sagemaker')
    s3 = boto3.client('s3')
    
    # Lấy thông tin training job
    response = sagemaker.describe_training_job(TrainingJobName=job_name)
    if response['TrainingJobStatus'] != 'Completed':
        raise Exception(f"Training job {job_name} chưa hoàn thành")
    
    model_artifacts = response['ModelArtifacts']['S3ModelArtifacts']
    print(f"Model artifacts: {model_artifacts}")
    
    # Download dữ liệu đánh giá từ S3
    if evaluation_data.startswith('s3://'):
        bucket, key = evaluation_data.replace('s3://', '').split('/', 1)
        local_path = 'evaluation_data.csv'
        s3.download_file(bucket, key, local_path)
    else:
        local_path = evaluation_data
    
    # Load dữ liệu đánh giá
    eval_data = pd.read_csv(local_path)
    
    # Trong dự án thực tế, ở đây sẽ load model từ S3 và đánh giá
    # Ví dụ đơn giản này giả định chúng ta đã có kết quả dự đoán
    
    # Giả định kết quả đánh giá
    # Trong thực tế, bạn sẽ load model và thực hiện dự đoán
    y_true = eval_data['target'].values
    y_pred = y_true * 0.9 + np.random.normal(0, 0.2, size=y_true.shape)
    
    # Tính toán metrics
    metrics = {
        'mse': mean_squared_error(y_true, y_pred),
        'rmse': np.sqrt(mean_squared_error(y_true, y_pred)),
        'mae': mean_absolute_error(y_true, y_pred),
        'r2_score': r2_score(y_true, y_pred),
        'accuracy': 0.88  # Giả định cho forecasting task
    }
    
    # So sánh với baseline metrics nếu có
    baseline = {}
    if baseline_metrics:
        if baseline_metrics.startswith('s3://'):
            bucket, key = baseline_metrics.replace('s3://', '').split('/', 1)
            local_baseline = 'baseline_metrics.json'
            try:
                s3.download_file(bucket, key, local_baseline)
                with open(local_baseline, 'r') as f:
                    baseline = json.load(f)
            except:
                print(f"Không tìm thấy baseline metrics: {baseline_metrics}")
        else:
            try:
                with open(baseline_metrics, 'r') as f:
                    baseline = json.load(f)
            except:
                print(f"Không tìm thấy baseline metrics: {baseline_metrics}")
    
    # Quyết định model có được chấp nhận hay không
    approved = True
    if baseline:
        print("So sánh với baseline:")
        for key in metrics:
            if key in baseline:
                improvement = (metrics[key] - baseline[key]) / baseline[key] * 100
                print(f"{key}: {metrics[key]:.4f} (baseline: {baseline[key]:.4f}, {improvement:+.2f}%)")
                
                # Kiểm tra tiêu chí cải thiện
                if key == 'accuracy' and metrics[key] < threshold:
                    approved = False
                elif key in ['mse', 'rmse', 'mae'] and metrics[key] > baseline[key] * 1.1:  # Cho phép tệ hơn 10%
                    approved = False
    else:
        print("Không có baseline để so sánh")
    
    # Tạo visualizations
    plt.figure(figsize=(10, 6))
    plt.scatter(y_true, y_pred, alpha=0.5)
    plt.plot([y_true.min(), y_true.max()], [y_true.min(), y_true.max()], 'r--')
    plt.xlabel('Actual')
    plt.ylabel('Predicted')
    plt.title('Actual vs Predicted Values')
    plt.savefig('plots/actual_vs_predicted.png')
    
    # Residual plot
    residuals = y_true - y_pred
    plt.figure(figsize=(10, 6))
    plt.scatter(y_pred, residuals, alpha=0.5)
    plt.hlines(y=0, xmin=y_pred.min(), xmax=y_pred.max(), colors='r', linestyles='--')
    plt.xlabel('Predicted')
    plt.ylabel('Residuals')
    plt.title('Residual Plot')
    plt.savefig('plots/residuals.png')
    
    # Lưu kết quả đánh giá
    result = {
        'job_name': job_name,
        'metrics': metrics,
        'baseline': baseline,
        'approved': approved,
        'model_uri': model_artifacts
    }
    
    with open('evaluation_results.json', 'w') as f:
        json.dump(result, f, indent=2)
    
    # Lưu kết quả approved để GitHub Actions có thể đọc
    with open('model_approved.txt', 'w') as f:
        f.write(str(approved).lower())
    
    print(f"Model approved: {approved}")
    return approved

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Evaluate trained model')
    parser.add_argument('--job-name', type=str, required=True,
                        help='Name of the SageMaker training job')
    parser.add_argument('--evaluation-data', type=str, required=True,
                        help='S3 URI or local path to evaluation data')
    parser.add_argument('--baseline-metrics', type=str, default=None,
                        help='S3 URI or local path to baseline metrics')
    parser.add_argument('--threshold', type=float, default=0.85,
                        help='Accuracy threshold for model approval')
    
    args = parser.parse_args()
    
    evaluate_model(
        args.job_name,
        args.evaluation_data,
        args.baseline_metrics,
        args.threshold
    )
```

### 4.3 Script đăng ký Model Registry

```python
# aws/script/register_model.py
import boto3
import argparse
import json
import time

def register_model(job_name, model_package_group, approval_status='PendingManualApproval'):
    """Đăng ký model vào Model Registry"""
    
    sagemaker = boto3.client('sagemaker')
    
    # Kiểm tra xem Model Package Group đã tồn tại chưa
    try:
        sagemaker.describe_model_package_group(ModelPackageGroupName=model_package_group)
        print(f"Model Package Group {model_package_group} đã tồn tại")
    except sagemaker.exceptions.ResourceNotFound:
        print(f"Tạo Model Package Group mới: {model_package_group}")
        sagemaker.create_model_package_group(
            ModelPackageGroupName=model_package_group,
            ModelPackageGroupDescription="Retail Forecast Models",
            Tags=[
                {'Key': 'Project', 'Value': 'RetailForecast'}
            ]
        )
    
    # Lấy thông tin training job
    training_job = sagemaker.describe_training_job(TrainingJobName=job_name)
    model_data_url = training_job['ModelArtifacts']['S3ModelArtifacts']
    image_uri = training_job['AlgorithmSpecification']['TrainingImage']
    
    # Đăng ký model package
    model_package_name = f"{model_package_group}-{int(time.time())}"
    
    create_model_package_input_dict = {
        'ModelPackageGroupName': model_package_group,
        'ModelPackageDescription': f"Model trained by job {job_name}",
        'InferenceSpecification': {
            'Containers': [
                {
                    'Image': image_uri,
                    'ModelDataUrl': model_data_url
                }
            ],
            'SupportedContentTypes': ['text/csv'],
            'SupportedResponseMIMETypes': ['text/csv']
        },
        'ModelApprovalStatus': approval_status,
        'CustomerMetadataProperties': {
            'TrainingJobName': job_name,
            'CreatedBy': 'GitHubActionsCI'
        },
        'Tags': [
            {'Key': 'Project', 'Value': 'RetailForecast'}
        ]
    }
    
    # Thêm model metrics nếu có
    try:
        with open('evaluation_results.json', 'r') as f:
            eval_results = json.load(f)
        
        model_metrics = []
        for metric_name, value in eval_results.get('metrics', {}).items():
            model_metrics.append({
                'Name': metric_name,
                'Value': float(value)
            })
        
        if model_metrics:
            create_model_package_input_dict['ModelMetrics'] = {
                'ModelQuality': {
                    'Statistics': {
                        'ContentType': 'application/json',
                        'Values': model_metrics
                    }
                }
            }
    except Exception as e:
        print(f"Không thể đọc kết quả đánh giá: {e}")
    
    response = sagemaker.create_model_package(**create_model_package_input_dict)
    
    print(f"Model package đã được tạo: {response['ModelPackageArn']}")
    
    # Lấy version từ ARN
    model_version = response['ModelPackageArn'].split('/')[-1]
    print(f"Model version: {model_version}")
    
    return model_version

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Register model to SageMaker Model Registry')
    parser.add_argument('--job-name', type=str, required=True,
                        help='Name of the training job')
    parser.add_argument('--model-package-group', type=str, required=True,
                        help='Model package group name')
    parser.add_argument('--approval-status', type=str, default='PendingManualApproval',
                        choices=['Approved', 'Rejected', 'PendingManualApproval'],
                        help='Initial model approval status')
    
    args = parser.parse_args()
    
    model_version = register_model(
        args.job_name,
        args.model_package_group,
        args.approval_status
    )
    
    print(model_version)
```

### 4.4 Script triển khai Endpoint

```python
# aws/script/deploy_endpoint.py
import boto3
import argparse
import time
import uuid

def deploy_endpoint(model_package_arn, endpoint_name, instance_type, instance_count=1):
    """Deploy model to SageMaker endpoint"""
    
    sagemaker = boto3.client('sagemaker')
    timestamp = int(time.time())
    
    # Tạo model
    model_name = f"{endpoint_name}-model-{timestamp}"
    print(f"Creating model {model_name} from {model_package_arn}")
    
    model_response = sagemaker.create_model(
        ModelName=model_name,
        PrimaryContainer={
            'ModelPackageName': model_package_arn
        },
        ExecutionRoleArn=sagemaker.get_caller_identity()['RoleArn']
    )
    
    # Tạo endpoint config
    config_name = f"{endpoint_name}-config-{timestamp}"
    print(f"Creating endpoint config {config_name}")
    
    endpoint_config_response = sagemaker.create_endpoint_config(
        EndpointConfigName=config_name,
        ProductionVariants=[
            {
                'VariantName': 'default',
                'ModelName': model_name,
                'InstanceType': instance_type,
                'InitialInstanceCount': instance_count,
                'InitialVariantWeight': 1.0
            }
        ],
        Tags=[
            {'Key': 'Project', 'Value': 'RetailForecast'}
        ]
    )
    
    # Kiểm tra xem endpoint đã tồn tại chưa
    endpoint_exists = False
    try:
        response = sagemaker.describe_endpoint(EndpointName=endpoint_name)
        endpoint_exists = True
    except sagemaker.exceptions.ClientError:
        endpoint_exists = False
    
    # Tạo hoặc cập nhật endpoint
    if endpoint_exists:
        print(f"Updating endpoint {endpoint_name}")
        endpoint_response = sagemaker.update_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=config_name
        )
    else:
        print(f"Creating endpoint {endpoint_name}")
        endpoint_response = sagemaker.create_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=config_name,
            Tags=[
                {'Key': 'Project', 'Value': 'RetailForecast'}
            ]
        )
    
    # Đợi endpoint sẵn sàng
    print(f"Waiting for endpoint {endpoint_name} to be in service...")
    waiter = sagemaker.get_waiter('endpoint_in_service')
    waiter.wait(EndpointName=endpoint_name)
    
    # Lấy thông tin endpoint
    endpoint_info = sagemaker.describe_endpoint(EndpointName=endpoint_name)
    print(f"Endpoint {endpoint_name} is ready (status: {endpoint_info['EndpointStatus']})")
    
    return {
        'endpoint_name': endpoint_name,
        'endpoint_arn': endpoint_info['EndpointArn'],
        'status': endpoint_info['EndpointStatus']
    }

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Deploy model to SageMaker endpoint')
    parser.add_argument('--model-package-arn', type=str, required=True,
                        help='ARN of model package to deploy')
    parser.add_argument('--endpoint-name', type=str, required=True,
                        help='Name of the endpoint')
    parser.add_argument('--instance-type', type=str, default='ml.t2.medium',
                        help='Instance type for inference')
    parser.add_argument('--instance-count', type=int, default=1,
                        help='Number of instances')
    
    args = parser.parse_args()
    
    result = deploy_endpoint(
        args.model_package_arn,
        args.endpoint_name,
        args.instance_type,
        args.instance_count
    )
    
    print(f"Endpoint deployment complete: {result}")
```

### 4.5 Script kiểm tra hiệu suất Endpoint

```python
# aws/script/autoscaling_endpoint.py
import boto3
import argparse
import json
import time
import datetime

def setup_autoscaling(endpoint_name, min_capacity=1, max_capacity=4, 
                      target_value=70.0, scale_in_cooldown=300, 
                      scale_out_cooldown=60):
    """Thiết lập autoscaling cho SageMaker endpoint"""
    
    # Lấy variant name và ARN của endpoint
    sm = boto3.client('sagemaker')
    endpoint = sm.describe_endpoint(EndpointName=endpoint_name)
    endpoint_arn = endpoint['EndpointArn']
    
    config_name = endpoint['EndpointConfigName']
    config = sm.describe_endpoint_config(EndpointConfigName=config_name)
    variant_name = config['ProductionVariants'][0]['VariantName']
    
    # Chuẩn bị resource ID cho autoscaling
    resource_id = f"endpoint/{endpoint_name}/variant/{variant_name}"
    
    # Thiết lập application autoscaling
    aas = boto3.client('application-autoscaling')
    
    # Đăng ký scalable target
    aas.register_scalable_target(
        ServiceNamespace='sagemaker',
        ResourceId=resource_id,
        ScalableDimension='sagemaker:variant:DesiredInstanceCount',
        MinCapacity=min_capacity,
        MaxCapacity=max_capacity
    )
    
    # Tạo scaling policy
    response = aas.put_scaling_policy(
        PolicyName=f"{endpoint_name}-scaling-policy",
        ServiceNamespace='sagemaker',
        ResourceId=resource_id,
        ScalableDimension='sagemaker:variant:DesiredInstanceCount',
        PolicyType='TargetTrackingScaling',
        TargetTrackingScalingPolicyConfiguration={
            'TargetValue': target_value,
            'PredefinedMetricSpecification': {
                'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
            },
            'ScaleInCooldown': scale_in_cooldown,
            'ScaleOutCooldown': scale_out_cooldown
        }
    )
    
    print(f"Autoscaling configured for endpoint {endpoint_name}")
    print(f"Min capacity: {min_capacity}, Max capacity: {max_capacity}")
    print(f"Target value: {target_value} invocations per instance")
    
    return {
        'endpoint_name': endpoint_name,
        'resource_id': resource_id,
        'scaling_policy_arn': response['PolicyARN']
    }

def load_test_endpoint(endpoint_name, test_data_path, duration=60, rate=10):
    """Test tải endpoint để kiểm tra hiệu suất và autoscaling"""
    
    sagemaker_runtime = boto3.client('sagemaker-runtime')
    
    # Load test data
    with open(test_data_path, 'r') as f:
        test_data = json.load(f)
    
    # Nếu test data là list, lấy phần tử đầu tiên
    if isinstance(test_data, list):
        sample = test_data[0]
    else:
        sample = test_data
    
    # Chuẩn bị test
    start_time = time.time()
    end_time = start_time + duration
    request_count = 0
    success_count = 0
    latencies = []
    
    print(f"Starting load test on endpoint {endpoint_name}")
    print(f"Duration: {duration} seconds, Rate: {rate} requests/second")
    
    # Thực hiện load test
    while time.time() < end_time:
        batch_start = time.time()
        for _ in range(rate):
            if time.time() >= end_time:
                break
                
            try:
                # Gửi request
                request_start = time.time()
                response = sagemaker_runtime.invoke_endpoint(
                    EndpointName=endpoint_name,
                    ContentType='application/json',
                    Body=json.dumps(sample)
                )
                
                # Đo latency
                latency = (time.time() - request_start) * 1000  # milliseconds
                latencies.append(latency)
                
                # Đọc kết quả
                result = json.loads(response['Body'].read().decode())
                
                # Cập nhật counter
                request_count += 1
                success_count += 1
                
                if request_count % 50 == 0:
                    print(f"Processed {request_count} requests...")
                
            except Exception as e:
                request_count += 1
                print(f"Error invoking endpoint: {e}")
        
        # Đợi đến đầu giây tiếp theo
        elapsed = time.time() - batch_start
        if elapsed < 1.0:
            time.sleep(1.0 - elapsed)
    
    # Tính toán kết quả
    total_time = time.time() - start_time
    avg_rate = request_count / total_time
    success_rate = (success_count / request_count) * 100 if request_count > 0 else 0
    
    if latencies:
        avg_latency = sum(latencies) / len(latencies)
        min_latency = min(latencies)
        max_latency = max(latencies)
        p95_latency = sorted(latencies)[int(len(latencies) * 0.95)]
        p99_latency = sorted(latencies)[int(len(latencies) * 0.99)]
    else:
        avg_latency = min_latency = max_latency = p95_latency = p99_latency = 0
    
    # In kết quả
    results = {
        'endpoint': endpoint_name,
        'duration_seconds': total_time,
        'request_count': request_count,
        'success_count': success_count,
        'success_rate_percent': success_rate,
        'avg_requests_per_second': avg_rate,
        'latency_ms': {
            'avg': avg_latency,
            'min': min_latency,
            'max': max_latency,
            'p95': p95_latency,
            'p99': p99_latency
        }
    }
    
    print("\nLoad Test Results:")
    print(json.dumps(results, indent=2))
    
    with open('load_test_results.json', 'w') as f:
        json.dump(results, f, indent=2)
    
    return results

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Setup autoscaling and test SageMaker endpoint')
    subparsers = parser.add_subparsers(dest='command')
    
    # Subparser cho autoscaling
    autoscaling_parser = subparsers.add_parser('autoscale')
    autoscaling_parser.add_argument('--endpoint-name', type=str, required=True)
    autoscaling_parser.add_argument('--min-capacity', type=int, default=1)
    autoscaling_parser.add_argument('--max-capacity', type=int, default=4)
    autoscaling_parser.add_argument('--target-value', type=float, default=70.0)
    
    # Subparser cho load testing
    loadtest_parser = subparsers.add_parser('loadtest')
    loadtest_parser.add_argument('--endpoint-name', type=str, required=True)
    loadtest_parser.add_argument('--test-data', type=str, required=True)
    loadtest_parser.add_argument('--duration', type=int, default=60)
    loadtest_parser.add_argument('--rate', type=int, default=10)
    
    args = parser.parse_args()
    
    if args.command == 'autoscale':
        setup_autoscaling(
            args.endpoint_name,
            args.min_capacity,
            args.max_capacity,
            args.target_value
        )
    elif args.command == 'loadtest':
        load_test_endpoint(
            args.endpoint_name,
            args.test_data,
            args.duration,
            args.rate
        )
    else:
        parser.print_help()
```

## 5. Giám sát và Thông báo

### 5.1 Thiết lập SNS Topic

```bash
# Tạo SNS topic cho thông báo deployment
aws sns create-topic --name retail-forecast-alerts

# Đăng ký email nhận thông báo
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:retail-forecast-alerts \
  --protocol email \
  --notification-endpoint team@example.com

# Đăng ký webhook Slack 
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:retail-forecast-alerts \
  --protocol https \
  --notification-endpoint https://hooks.slack.com/services/XXXX/YYYY/ZZZZ
```

### 5.2 CloudWatch Dashboard cho CI/CD Pipeline

```bash
# Tạo CloudWatch dashboard cho pipeline
aws cloudwatch put-dashboard \
  --dashboard-name RetailForecast-CI-CD-Pipeline \
  --dashboard-body '{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/SageMaker", "TrainingJobsCompleted", { "stat": "Sum", "period": 86400 } ],
                    [ "AWS/SageMaker", "TrainingJobsFailed", { "stat": "Sum", "period": 86400 } ]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "us-east-1",
                "title": "SageMaker Training Jobs",
                "period": 300,
                "stat": "Sum"
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
                    [ "AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/retail-forecast/abcdef" ],
                    [ ".", "HTTPCode_Target_5XX_Count", ".", "." ],
                    [ ".", "HTTPCode_Target_4XX_Count", ".", "." ]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "us-east-1",
                "title": "API Performance",
                "period": 300,
                "stat": "Average"
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
                    [ "AWS/SageMaker", "Invocations", "EndpointName", "retail-forecast-prod", { "stat": "Sum", "period": 60 } ],
                    [ ".", "InvocationsPerInstance", ".", "." ]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "us-east-1",
                "title": "SageMaker Endpoint Invocations",
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
                    [ "RetailForecast/Pipeline", "DeploymentFrequency" ],
                    [ ".", "FailureRate" ],
                    [ ".", "LeadTime" ]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "us-east-1",
                "title": "CI/CD Metrics",
                "period": 86400,
                "stat": "Average"
            }
        }
    ]
}'
```

### 5.3 Cấu hình Pipeline Metrics

```python
# aws/script/pipeline_metrics.py
import boto3
import json
import argparse
from datetime import datetime, timedelta

def publish_pipeline_metrics(deploy_success=True, lead_time=None):
    """Publish CI/CD pipeline metrics to CloudWatch"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Get date parts for daily metrics
    now = datetime.utcnow()
    today = now.strftime('%Y-%m-%d')
    
    # Increment deployment count
    cloudwatch.put_metric_data(
        Namespace='RetailForecast/Pipeline',
        MetricData=[
            {
                'MetricName': 'DeploymentCount',
                'Dimensions': [
                    {
                        'Name': 'Date',
                        'Value': today
                    }
                ],
                'Value': 1.0,
                'Unit': 'Count'
            }
        ]
    )
    
    # Add deployment success/failure
    cloudwatch.put_metric_data(
        Namespace='RetailForecast/Pipeline',
        MetricData=[
            {
                'MetricName': 'DeploymentSuccess',
                'Dimensions': [
                    {
                        'Name': 'Date',
                        'Value': today
                    }
                ],
                'Value': 1.0 if deploy_success else 0.0,
                'Unit': 'Count'
            }
        ]
    )
    
    # Add lead time if provided
    if lead_time is not None:
        cloudwatch.put_metric_data(
            Namespace='RetailForecast/Pipeline',
            MetricData=[
                {
                    'MetricName': 'LeadTimeMinutes',
                    'Dimensions': [
                        {
                            'Name': 'Date',
                            'Value': today
                        }
                    ],
                    'Value': lead_time,
                    'Unit': 'Minutes'
                }
            ]
        )
    
    print(f"Pipeline metrics published successfully")
    
def calculate_pipeline_kpis(days=30):
    """Calculate KPIs for CI/CD pipeline"""
    
    cloudwatch = boto3.client('cloudwatch')
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=days)
    
    # Get deployment frequency
    deploy_count_response = cloudwatch.get_metric_statistics(
        Namespace='RetailForecast/Pipeline',
        MetricName='DeploymentCount',
        StartTime=start_time,
        EndTime=end_time,
        Period=86400 * days,  # entire period
        Statistics=['Sum']
    )
    
    # Get deployment success/failure
    deploy_success_response = cloudwatch.get_metric_statistics(
        Namespace='RetailForecast/Pipeline',
        MetricName='DeploymentSuccess',
        StartTime=start_time,
        EndTime=end_time,
        Period=86400 * days,  # entire period
        Statistics=['Sum']
    )
    
    # Get lead time
    lead_time_response = cloudwatch.get_metric_statistics(
        Namespace='RetailForecast/Pipeline',
        MetricName='LeadTimeMinutes',
        StartTime=start_time,
        EndTime=end_time,
        Period=86400 * days,  # entire period
        Statistics=['Average']
    )
    
    # Calculate KPIs
    deploy_count = deploy_count_response['Datapoints'][0]['Sum'] if deploy_count_response['Datapoints'] else 0
    deploy_success = deploy_success_response['Datapoints'][0]['Sum'] if deploy_success_response['Datapoints'] else 0
    
    # Calculate metrics
    deployments_per_day = deploy_count / days if days > 0 else 0
    failure_rate = (deploy_count - deploy_success) / deploy_count * 100 if deploy_count > 0 else 0
    avg_lead_time = lead_time_response['Datapoints'][0]['Average'] if lead_time_response['Datapoints'] else 0
    
    kpis = {
        'period_days': days,
        'total_deployments': deploy_count,
        'successful_deployments': deploy_success,
        'deployments_per_day': deployments_per_day,
        'failure_rate_percent': failure_rate,
        'avg_lead_time_minutes': avg_lead_time
    }
    
    print(json.dumps(kpis, indent=2))
    return kpis

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Manage CI/CD pipeline metrics')
    subparsers = parser.add_subparsers(dest='command')
    
    # Subparser cho publish metrics
    publish_parser = subparsers.add_parser('publish')
    publish_parser.add_argument('--success', type=bool, default=True)
    publish_parser.add_argument('--lead-time', type=float, help='Lead time in minutes')
    
    # Subparser cho calculate KPIs
    kpi_parser = subparsers.add_parser('kpi')
    kpi_parser.add_argument('--days', type=int, default=30)
    
    args = parser.parse_args()
    
    if args.command == 'publish':
        publish_pipeline_metrics(args.success, args.lead_time)
    elif args.command == 'kpi':
        calculate_pipeline_kpis(args.days)
    else:
        parser.print_help()
```

## 6. Kiểm tra và Xác thực

### 6.1 Checklist Hoàn thành

Các thành phần chính đã triển khai:

- **Pipeline GitHub Actions**: Workflow đã được cấu hình với các job đầy đủ
- **IAM Role & Permissions**: Role cho GitHub Actions với OIDC xác thực
- **Automated Testing**: Unit tests, code quality check, data validation
- **Model Training**: Tự động trigger SageMaker training jobs 
- **Model Evaluation & Registry**: Đánh giá model và đăng ký vào Model Registry
- **Docker Build**: Tự động build và push image lên ECR
- **EKS Deployment**: Tự động cập nhật Kubernetes deployment
- **Health Check**: Kiểm tra và xác nhận API hoạt động sau deployment
- **CloudWatch Alarms**: Tự động cấu hình alert cho deployment mới
- **Notifications**: Thông báo kết quả deployment qua SNS

### 6.2 Kiểm tra Pipeline

Để xác nhận pipeline hoạt động chính xác, thực hiện các bước sau:

1. **Kích hoạt pipeline thông qua commit mới**
   ```bash
   # Commit code mới
   echo "# Test CI/CD pipeline" >> README.md
   git add README.md
   git commit -m "Test: Trigger CI/CD pipeline"
   git push origin main
   
   # Kiểm tra GitHub Actions
   # Pipeline sẽ tự động kích hoạt trong vòng 1 phút
   ```

2. **Kích hoạt pipeline thủ công với huấn luyện model**
   ```bash
   # Sử dụng GitHub UI để kích hoạt workflow với các tùy chọn:
   # - environment: prod
   # - retrain_model: true
   ```

3. **Kiểm tra ECR repository**
   ```bash
   # Kiểm tra image mới trên ECR
   aws ecr describe-images \
     --repository-name retail-forecast \
     --query 'sort_by(imageDetails,& imagePushedAt)[-5:]'
   
   # Kiểm tra tag mới nhất
   aws ecr list-images \
     --repository-name retail-forecast \
     --filter "tagStatus=TAGGED" \
     --query 'imageIds[?contains(imageTag, `latest`)]'
   ```

4. **Kiểm tra SageMaker model và deployment**
   ```bash
   # Kiểm tra training job mới nhất
   aws sagemaker list-training-jobs \
     --sort-by CreationTime \
     --sort-order Descending \
     --max-items 5
   
   # Kiểm tra model package mới nhất
   aws sagemaker list-model-packages \
     --model-package-group-name retail-forecast-models \
     --sort-by CreationTime \
     --sort-order Descending \
     --max-items 5
   
   # Kiểm tra endpoint
   aws sagemaker describe-endpoint \
     --endpoint-name retail-forecast-prod
   ```

5. **Kiểm tra EKS deployment**
   ```bash
   # Kiểm tra deployment mới
   kubectl get deployments -n retail-forecast-prod
   
   # Kiểm tra pods
   kubectl get pods -n retail-forecast-prod
   
   # Kiểm tra image được sử dụng
   kubectl describe deployment retail-forecast-api -n retail-forecast-prod | grep Image:
   
   # Kiểm tra history rollout
   kubectl rollout history deployment/retail-forecast-api -n retail-forecast-prod
   ```

6. **Test API endpoint**
   ```bash
   # Lấy endpoint URL
   ENDPOINT=$(kubectl get ingress -n retail-forecast-prod retail-forecast-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
   
   # Kiểm tra health endpoint
   curl -v http://$ENDPOINT/health
   
   # Test prediction endpoint
   curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"store_id": 1, "item_id": 123, "date": "2023-12-01"}' \
     http://$ENDPOINT/predict
   ```

### 6.3 Monitoring Commands

```bash
# Kiểm tra CloudWatch metrics của API
aws cloudwatch get-metric-statistics \
  --namespace "RetailForecast/prod" \
  --metric-name "Latency" \
  --dimensions Name=DeploymentId,Value=<latest-deployment-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average

# Kiểm tra SageMaker endpoint metrics
aws cloudwatch get-metric-statistics \
  --namespace "AWS/SageMaker" \
  --metric-name "ModelLatency" \
  --dimensions Name=EndpointName,Value=retail-forecast-prod \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average

# Kiểm tra Pipeline metrics
aws cloudwatch get-metric-statistics \
  --namespace "RetailForecast/Pipeline" \
  --metric-name "DeploymentSuccess" \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Sum

# Kiểm tra CloudWatch Logs cho pipeline
aws logs get-log-events \
  --log-group-name "/aws/sagemaker/TrainingJobs" \
  --log-stream-name <latest-training-job-name>
```

## 7. Best Practices & Tối ưu

### 7.1 Pipeline Optimization

- **Parallel Execution**: Chạy song song các job không phụ thuộc nhau
- **Caching**: Sử dụng caching cho pip và Docker layers
- **Conditional Execution**: Chỉ chạy các job cần thiết dựa trên loại thay đổi
- **Matrix Builds**: Test trên nhiều phiên bản Python hoặc môi trường
- **Artifact Sharing**: Sử dụng artifacts để chia sẻ dữ liệu giữa các job

### 7.2 Security Best Practices

- **OIDC Integration**: Sử dụng federation thay vì access keys
- **Least Privilege IAM**: Role với quyền hạn tối thiểu cần thiết
- **Secret Management**: Sử dụng GitHub Secrets hoặc AWS Secrets Manager
- **Image Scanning**: Kiểm tra lỗi bảo mật trong container images
- **Network Security**: Giới hạn network access trong pipeline

### 7.3 Quality Gates

- **Code Quality Checks**: Linting, formatting, static analysis
- **Unit & Integration Tests**: Test tự động cho mọi thay đổi code
- **Data Validation**: Kiểm tra tính toàn vẹn và chất lượng dữ liệu
- **Model Evaluation**: Đánh giá hiệu suất model với baseline metrics
- **Performance Testing**: Kiểm tra độ trễ và khả năng xử lý tải của API

### 7.4 Observability

- **Pipeline Metrics**: Theo dõi tần suất deployment, thời gian thực hiện
- **Model Monitoring**: Theo dõi model drift và data drift
- **API Metrics**: Latency, error rate, throughput của endpoints
- **Logging**: Log tập trung với cấu trúc thống nhất
- **Alerting**: Cảnh báo sớm khi phát hiện vấn đề

## Tổng kết

CI/CD pipeline đã được thiết lập đầy đủ sử dụng GitHub Actions để tự động hóa toàn bộ quy trình MLOps cho dự án Retail Prediction:

1. **Tích hợp liên tục (CI)**: Tự động test, lint, và build Docker image.
2. **Huấn luyện tự động**: Kích hoạt SageMaker training jobs khi cần thiết.
3. **Model Registry**: Đánh giá và đăng ký model với quản lý version.
4. **Deployment liên tục (CD)**: Tự động triển khai lên EKS cluster.
5. **Giám sát**: CloudWatch metrics và alerts cho toàn bộ hệ thống.

Pipeline này đảm bảo quy trình MLOps đáng tin cậy, tự động, và có khả năng mở rộng, giúp team có thể tập trung vào việc cải thiện model và tính năng thay vì công việc vận hành thủ công.

{{% notice success %}}
**🎯 Task 11 Complete - CI/CD Pipeline**
- **GitHub Actions workflow** configured với đầy đủ CI/CD stages
- **IAM Role & OIDC** setup cho secure authentication
- **Automated testing** và quality gates
- **SageMaker integration** cho model training & registry
- **EKS deployment** automation với health checks
- **CloudWatch monitoring** và notifications
{{% /notice %}}

## 8. Clean Up CI/CD Resources

### 8.1 Xóa GitHub Actions Workflow

```bash
# Disable GitHub Actions workflows
# Thực hiện trực tiếp trên GitHub repository hoặc qua GitHub CLI

# Nếu sử dụng GitHub CLI
gh workflow disable mlops-pipeline.yml

# List tất cả workflow runs
gh run list --limit 50

# Cancel running workflows
gh run list --status in_progress --json databaseId --jq '.[].databaseId' | xargs -I {} gh run cancel {}

# Xóa workflow artifacts (có thể tốn phí storage)
gh api repos/:owner/:repo/actions/artifacts --paginate | jq -r '.artifacts[] | select(.expired == false) | .id' | xargs -I {} gh api --method DELETE repos/:owner/:repo/actions/artifacts/{}
```

### 8.2 Xóa IAM Roles và Policies

```bash
# List tất cả IAM roles liên quan đến CI/CD
aws iam list-roles \
  --query 'Roles[?contains(RoleName, `GitHub`) || contains(RoleName, `CICD`) || contains(RoleName, `RetailForecast`)].RoleName' \
  --output table

# Detach policies trước khi xóa role
aws iam list-attached-role-policies \
  --role-name GitHubActionsRole \
  --query 'AttachedPolicies[].PolicyArn' \
  --output text | tr '\t' '\n' | while read policy_arn; do
    echo "Detaching policy: $policy_arn"
    aws iam detach-role-policy --role-name GitHubActionsRole --policy-arn "$policy_arn"
done

# Xóa custom policies
aws iam delete-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/GitHubActionsPolicy

aws iam delete-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/RetailForecastCICDPolicy

# Xóa roles
aws iam delete-role --role-name GitHubActionsRole
aws iam delete-role --role-name RetailForecastCICDRole

# Xóa instance profiles (nếu có)
aws iam remove-role-from-instance-profile \
  --instance-profile-name JenkinsInstanceProfile \
  --role-name JenkinsRole || true

aws iam delete-instance-profile \
  --instance-profile-name JenkinsInstanceProfile || true

aws iam delete-role --role-name JenkinsRole || true
```

### 8.3 Xóa Jenkins Infrastructure (nếu sử dụng)

```bash
# Terminate Jenkins EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Jenkins-Server" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text | tr '\t' '\n' | while read instance_id; do
    echo "Terminating Jenkins instance: $instance_id"
    aws ec2 terminate-instances --instance-ids "$instance_id"
done

# Xóa Jenkins security group
JENKINS_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=sg-jenkins" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

if [ "$JENKINS_SG_ID" != "None" ]; then
    aws ec2 delete-security-group --group-id "$JENKINS_SG_ID"
fi

# Xóa Jenkins key pair
aws ec2 delete-key-pair --key-name jenkins-key-pair || true
```

### 8.4 Xóa SNS Topics và Subscriptions

```bash
# List tất cả SNS topics liên quan
aws sns list-topics \
  --query 'Topics[?contains(TopicArn, `retail-forecast`) || contains(TopicArn, `mlops`)].TopicArn' \
  --output table

# Xóa subscriptions trước
SNS_TOPIC_ARN=$(aws sns list-topics \
  --query 'Topics[?contains(TopicArn, `retail-forecast-alerts`)].TopicArn' \
  --output text)

if [ "$SNS_TOPIC_ARN" != "" ]; then
    # List và xóa subscriptions
    aws sns list-subscriptions-by-topic \
      --topic-arn "$SNS_TOPIC_ARN" \
      --query 'Subscriptions[].SubscriptionArn' \
      --output text | tr '\t' '\n' | while read sub_arn; do
        if [ "$sub_arn" != "PendingConfirmation" ]; then
            echo "Unsubscribing: $sub_arn"
            aws sns unsubscribe --subscription-arn "$sub_arn"
        fi
    done
    
    # Xóa topic
    aws sns delete-topic --topic-arn "$SNS_TOPIC_ARN"
fi
```

### 8.5 Xóa CloudWatch Resources cho CI/CD

```bash
# Xóa CloudWatch dashboard cho CI/CD
aws cloudwatch delete-dashboards \
  --dashboard-names "RetailForecast-CI-CD-Pipeline" || true

# Xóa custom metrics namespace
aws cloudwatch list-metrics \
  --namespace "RetailForecast/Pipeline" \
  --query 'Metrics[].MetricName' \
  --output text | tr '\t' '\n' | while read metric; do
    echo "Found pipeline metric: $metric"
    # Note: CloudWatch metrics tự động expire sau 15 tháng
done

# Xóa alarms liên quan đến CI/CD
aws cloudwatch describe-alarms \
  --alarm-name-prefix "Pipeline-" \
  --query 'MetricAlarms[].AlarmName' \
  --output text | tr '\t' '\n' | while read alarm; do
    echo "Deleting pipeline alarm: $alarm"
    aws cloudwatch delete-alarms --alarm-names "$alarm"
done

# Xóa log groups cho CI/CD
aws logs delete-log-group \
  --log-group-name "/aws/codebuild/retail-forecast-build" || true

aws logs delete-log-group \
  --log-group-name "/aws/codepipeline/retail-forecast-pipeline" || true
```

### 8.6 Xóa ECR Images và Tags

```bash
# List tất cả images trong ECR repository
aws ecr describe-images \
  --repository-name mlops/retail-api \
  --region ap-southeast-1 \
  --query 'imageDetails[].imageTags[]' \
  --output table

# Xóa specific CI/CD tags (giữ lại production tags)
aws ecr batch-delete-image \
  --repository-name mlops/retail-api \
  --region ap-southeast-1 \
  --image-ids imageTag=dev imageTag=staging imageTag=feature-* || true

# Xóa untagged images
aws ecr describe-images \
  --repository-name mlops/retail-api \
  --region ap-southeast-1 \
  --filter tagStatus=UNTAGGED \
  --query 'imageDetails[].imageDigest' \
  --output text | tr '\t' '\n' | while read digest; do
    echo "Deleting untagged image: $digest"
    aws ecr batch-delete-image \
      --repository-name mlops/retail-api \
      --region ap-southeast-1 \
      --image-ids imageDigest="$digest"
done
```

### 8.7 Clean Up SageMaker Resources

```bash
# List training jobs created by CI/CD
aws sagemaker list-training-jobs \
  --name-contains "retail-forecast" \
  --max-results 50 \
  --query 'TrainingJobSummaries[].TrainingJobName' \
  --output table

# Stop running training jobs
aws sagemaker list-training-jobs \
  --status-equals InProgress \
  --name-contains "retail-forecast" \
  --query 'TrainingJobSummaries[].TrainingJobName' \
  --output text | tr '\t' '\n' | while read job_name; do
    echo "Stopping training job: $job_name"
    aws sagemaker stop-training-job --training-job-name "$job_name"
done

# Xóa model versions trong Model Registry (giữ approved models)
aws sagemaker list-model-packages \
  --model-package-group-name "retail-forecast-models" \
  --model-approval-status PendingManualApproval \
  --query 'ModelPackageSummaryList[].ModelPackageArn' \
  --output text | tr '\t' '\n' | while read package_arn; do
    echo "Deleting pending model package: $package_arn"
    aws sagemaker delete-model-package --model-package-name "$package_arn"
done

# Xóa endpoints từ failed deployments
aws sagemaker list-endpoints \
  --name-contains "retail-forecast-dev" \
  --query 'Endpoints[?EndpointStatus==`Failed`].EndpointName' \
  --output text | tr '\t' '\n' | while read endpoint; do
    echo "Deleting failed endpoint: $endpoint"
    aws sagemaker delete-endpoint --endpoint-name "$endpoint"
done
```

### 8.8 Verification

```bash
# Verify IAM resources đã bị xóa
aws iam get-role --role-name GitHubActionsRole 2>/dev/null || echo "GitHubActionsRole deleted"

# Verify SNS topics đã bị xóa
aws sns list-topics \
  --query 'Topics[?contains(TopicArn, `retail-forecast-alerts`)]' || echo "SNS topics cleaned"

# Verify CloudWatch resources
aws cloudwatch describe-dashboards \
  --dashboard-name-prefix "RetailForecast-CI-CD" || echo "Dashboards cleaned"

# Verify no running training jobs
aws sagemaker list-training-jobs \
  --status-equals InProgress \
  --name-contains "retail-forecast" \
  --query 'TrainingJobSummaries' || echo "No running training jobs"

# Check GitHub Actions status
gh run list --limit 5 --status completed
```

## 9. Bảng giá CI/CD Pipeline (ap-southeast-1)

### 9.1. GitHub Actions Pricing

| Plan | Included Minutes | Price per minute | Storage |
|------|------------------|------------------|---------|
| **Free (Public repos)** | Unlimited | $0 | 500MB |
| **Free (Private repos)** | 2,000 min/month | $0.008 | 500MB |
| **Pro** | 3,000 min/month | $0.008 | 1GB |
| **Team** | 10,000 min/month | $0.008 | 2GB |
| **Enterprise** | 50,000 min/month | $0.008 | 50GB |

**Runner costs:**
- Ubuntu: Standard rate
- macOS: 10x Standard rate  
- Windows: 2x Standard rate
- Self-hosted: Free compute, infrastructure cost only

### 9.2. AWS IAM và Security Costs

| Service | Cost | Description |
|---------|------|-------------|
| **IAM Roles & Policies** | Free | Unlimited roles and policies |
| **STS AssumeRole calls** | $0.002/1000 calls | OIDC authentication |
| **AWS Config (compliance)** | $0.003/configuration item | Policy compliance tracking |

**Example calculation:**
- 100 CI/CD runs/month × 5 STS calls = 500 calls = $0.001/month

### 9.3. SageMaker Training Costs trong CI/CD

| Instance Type | Cost per Hour | Typical Job Duration | Cost per Run |
|---------------|---------------|---------------------|--------------|
| **ml.m5.large** | $0.134 | 15 minutes | $0.034 |
| **ml.m5.xlarge** | $0.269 | 10 minutes | $0.045 |
| **ml.c5.xlarge** | $0.238 | 8 minutes | $0.032 |
| **ml.p3.2xlarge** | $4.284 | 5 minutes | $0.357 |

**Monthly costs by frequency:**
- Daily training: 30 runs × $0.045 = $1.35
- Weekly training: 4 runs × $0.045 = $0.18  
- On-demand training: 2 runs × $0.045 = $0.09

### 9.4. ECR Storage và Transfer Costs

| Component | Cost | Volume | Monthly Cost |
|-----------|------|---------|--------------|
| **Storage** | $0.10/GB/month | 5GB images | $0.50 |
| **Data Transfer IN** | Free | Upload images | $0 |
| **Data Transfer OUT** | $0.12/GB | Download to EKS | Variable |

**Image management costs:**
```bash
# Example: 10 images × 500MB each = 5GB storage
# Monthly cost: 5GB × $0.10 = $0.50
# Transfer to EKS: 5GB × $0.12 = $0.60 (one-time per deployment)
```

### 9.5. CloudWatch Monitoring cho CI/CD

| Metric Type | Quantity | Unit Cost | Monthly Cost |
|-------------|----------|-----------|--------------|
| **Custom Metrics** | 20 metrics | $0.30/metric | $6.00 |
| **API Calls** | 100K calls | $0.01/1K calls | $1.00 |
| **Alarms** | 10 alarms | $0.10/alarm | $1.00 |
| **Dashboard** | 1 dashboard | $3.00/dashboard | $3.00 |
| **Total Monitoring** | | | **$11.00** |

### 9.6. SNS Notification Costs

| Notification Type | Volume | Cost per Message | Monthly Cost |
|-------------------|---------|------------------|--------------|
| **Email** | 200 notifications | $0.75/million | $0.0002 |
| **SMS** | 50 notifications | $0.8/message | $40.00 |
| **Slack Webhook** | 200 notifications | $0.75/million | $0.0002 |
| **Push Mobile** | 100 notifications | $0.75/million | $0.0001 |

### 9.7. Jenkins Infrastructure Costs (nếu self-hosted)

| Component | Instance Type | Monthly Hours | Monthly Cost |
|-----------|---------------|---------------|--------------|
| **Jenkins Master** | t3.medium | 730 hours | $30.37 |
| **Build Agents** | t3.large (2 agents) | 100 hours | $13.25 |
| **EBS Storage** | 100GB gp3 | - | $8.00 |
| **Data Transfer** | 50GB/month | $0.12/GB | $6.00 |
| **Total Jenkins** | | | **$57.62** |

### 9.8. CI/CD Pipeline Scenarios

**Scenario 1: Small Team (GitHub Actions)**

| Component | Usage | Monthly Cost |
|-----------|-------|--------------|
| GitHub Actions (private) | 2,000 min included | $0 |
| SageMaker training | 4 runs/month | $0.18 |
| ECR storage | 2GB images | $0.20 |
| CloudWatch basic | 5 metrics, 3 alarms | $1.80 |
| SNS notifications | Email only | $0.0002 |
| **Total Small Team** | | **$2.18/month** |

**Scenario 2: Medium Team (GitHub Actions Pro)**

| Component | Usage | Monthly Cost |
|-----------|-------|--------------|
| GitHub Actions Pro | 3,000 min + 500 extra | $4.00 |
| SageMaker training | 12 runs/month | $0.54 |
| ECR storage | 8GB images | $0.80 |
| CloudWatch full | 15 metrics, 8 alarms | $5.30 |
| SNS notifications | Email + Slack | $0.0004 |
| **Total Medium Team** | | **$10.64/month** |

**Scenario 3: Enterprise (Self-hosted Jenkins)**

| Component | Usage | Monthly Cost |
|-----------|-------|--------------|
| Jenkins infrastructure | t3.medium + agents | $57.62 |
| SageMaker training | 60 runs/month | $2.70 |
| ECR storage | 20GB images | $2.00 |
| CloudWatch enterprise | 50 metrics, 25 alarms | $17.50 |
| SNS notifications | Multi-channel | $40.50 |
| **Total Enterprise** | | **$120.32/month** |

### 9.9. Cost Optimization Strategies

**GitHub Actions Optimization:**
```yaml
# Use matrix strategy để giảm runtime
strategy:
  matrix:
    python-version: [3.8, 3.9, 3.10]
    
# Cache dependencies
- uses: actions/cache@v3
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    
# Conditional jobs
if: github.ref == 'refs/heads/main'
```

**SageMaker Training Optimization:**
```python
# Sử dụng Spot instances cho training
training_params = {
    'TrainingJobName': job_name,
    'ResourceConfig': {
        'InstanceType': 'ml.m5.large',
        'InstanceCount': 1,
        'VolumeSizeInGB': 30,
        'UseSpotInstances': True,  # 90% cost savings
        'MaxRuntimeInSeconds': 3600
    }
}
```

**ECR Cost Optimization:**
```bash
# Lifecycle policy để tự động xóa old images
aws ecr put-lifecycle-policy \
  --repository-name mlops/retail-api \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": {
          "type": "expire"
        }
      }
    ]
  }'
```

### 9.10. ROI Analysis cho CI/CD Investment

| Benefit | Manual Process | Automated CI/CD | Time Saved | Cost Benefit |
|---------|----------------|-----------------|------------|--------------|
| **Code Testing** | 2 hours/week | 5 minutes | 1.9 hours | $95/week |
| **Model Training** | 30 min setup | Automatic | 2 hours/month | $100/month |
| **Deployment** | 1 hour/deploy | 5 minutes | 55 min/deploy | $45/deploy |
| **Rollback** | 2 hours | 5 minutes | 1.9 hours | $95/incident |

**Annual ROI calculation:**
- **Investment:** $127.68/month × 12 = $1,532
- **Savings:** (2 hours/week × 52 weeks + 2 hours/month × 12) × $50/hour = $6,400
- **ROI:** 322% return on investment

### 9.11. Monitoring CI/CD Costs

```bash
# Track GitHub Actions usage
gh api /repos/:owner/:repo/actions/billing/usage

# Monitor SageMaker training costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon SageMaker"]}}'

# ECR storage costs
aws ecr describe-registry-statistics --region ap-southeast-1

# CloudWatch costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon CloudWatch"]}}'
```

{{% notice info %}}
**💰 Cost Summary cho Task 11:**
- **Small Team (GitHub Free):** $2.18/month
- **Medium Team (GitHub Pro):** $10.64/month  
- **Enterprise (Self-hosted):** $120.32/month
- **ROI:** 322% với automation benefits
- **Break-even point:** ~3 months cho medium team setup
{{% /notice %}}

---

**Next Step**: [Task 12: Cost Optimization & Teardown](../12-cost-&-teardown/)
