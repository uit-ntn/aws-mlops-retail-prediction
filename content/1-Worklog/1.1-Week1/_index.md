---
title: "Week 1 - Introduction to AWS"
weight: 2
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives:

- Familiarize with AWS Global Infrastructure (Regions, AZs) and Shared Responsibility Model.
- Master AWS Console & CLI configuration.
- Implement basic security practices (IAM User/Role, MFA).

### Tasks to be carried out this week:

| Day | Task                                                                                                                                                  | Reference     |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| 2   | - **Setup:** Create AWS Account, enable MFA for root user.<br>- **IAM:** Create IAM Groups/Users with Least Privilege principles.                     | AWS IAM Docs  |
| 3   | - **CLI:** Install AWS CLI v2, configure profiles (`~/.aws/config`, `credentials`).<br>- **Verify:** Run `aws sts get-caller-identity` to check auth. | AWS CLI Guide |
| 4   | - **S3 Basics:** Create a bucket with "Block Public Access" enabled.<br>- Practice Upload/Download objects via Console & CLI.                         | AWS S3 Docs   |
| 5   | - **Security:** Study Shared Responsibility Model.<br>- Define tagging conventions (Owner, Project, Env).                                             | FCJ Materials |
| 6   | - **Review:** Document setup process and verify billing alerts.<br>- Weekly Report.                                                                   | -             |

### Week 1 Achievements:

- Secured AWS account with MFA and no root access for daily tasks.
- Configured AWS CLI profiles for different environments.
- Created secure S3 buckets with encryption and versioning enabled.
