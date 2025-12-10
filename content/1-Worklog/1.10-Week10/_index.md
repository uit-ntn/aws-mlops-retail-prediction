---
title: "Week 10 - DevOps & IaC"
weight: 11
chapter: false
pre: " <b> 1.10. </b> "
---

### Week 10 Objectives:

- Implement Infrastructure as Code (Terraform/CloudFormation).
- Setup CI/CD pipelines (CodePipeline/GitHub Actions).

### Tasks to be carried out this week:

| Day | Task                                                                                                                 | Reference        |
| --- | -------------------------------------------------------------------------------------------------------------------- | ---------------- |
| 2   | - **IaC:** Write Terraform code for VPC, SG, and IAM resources.<br>- Manage state with S3 Backend.                   | Terraform Docs   |
| 3   | - **CI/CD Setup:** Configure CodeCommit/GitHub repo.<br>- Setup CodeBuild projects for building Docker images.       | AWS CodeBuild    |
| 4   | - **Pipeline:** Create CodePipeline to automate Build -> Deploy.<br>- Implement Manual Approval stages.              | AWS CodePipeline |
| 5   | - **Security:** Integrate secrets management (SSM/Secrets Manager) into pipeline.<br>- Verify automated deployments. | FinOps Guide     |
| 6   | - **Review:** Test full CI/CD flow from commit to deploy.<br>- Weekly Report.                                        | -                |

### Week 10 Achievements:

- Automated infrastructure provisioning using Terraform.
- Built a fully automated CI/CD pipeline for container deployment.
- Eliminated hardcoded secrets in deployment scripts.
