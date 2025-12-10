---
title: "Week 3 - Compute VM Service on AWS"
weight: 4
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

- Master EC2 deployment and management.
- Implement High Availability with Auto Scaling (ASG) and Load Balancing (ALB).

### Tasks to be carried out this week:

| Day | Task                                                                                                                      | Reference    |
| --- | ------------------------------------------------------------------------------------------------------------------------- | ------------ |
| 2   | - **EC2:** Create Launch Templates with User Data scripts.<br>- Configure IAM Roles for EC2 (S3 access).                  | AWS EC2 Docs |
| 3   | - **Load Balancing:** Deploy Application Load Balancer (ALB).<br>- Configure Target Groups and Health Checks (`/health`). | AWS ELB Docs |
| 4   | - **Auto Scaling:** Create Auto Scaling Group (ASG) across Multi-AZ.<br>- Set scaling policies (e.g., CPU > 50%).         | AWS ASG Docs |
| 5   | - **Monitoring:** Install CloudWatch Agent via User Data.<br>- Test scaling events (stress test).                         | Hands-on Lab |
| 6   | - **Review:** Verify ALB routing and ASG dynamic scaling.<br>- Weekly Report.                                             | -            |

### Week 3 Achievements:

- Automated EC2 provisioning using Launch Templates and User Data.
- Deployed a highly available web app with ALB and ASG.
- Secured EC2 access using IAM Roles instead of hardcoded keys.
