---
title: "Week 2 - Networking on AWS"
weight: 3
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

- Design and implement a Virtual Private Cloud (VPC).
- Configure routing, gateways (IGW, NAT), and security layers (SG, NACL).

### Tasks to be carried out this week:

| Day | Task                                                                                                                           | Reference           |
| --- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------- |
| 2   | - **VPC Design:** Plan CIDR blocks for 2 AZs (Public/Private Subnets).<br>- Create VPC, Subnets, and Route Tables.             | AWS VPC Docs        |
| 3   | - **Connectivity:** Attach Internet Gateway (IGW) for public subnets.<br>- Deploy NAT Gateway for private subnet egress.       | AWS Workshop        |
| 4   | - **Security:** Configure Security Groups (Stateful) vs NACLs (Stateless).<br>- Implement "Bastion Host" concept.              | FCJ Materials       |
| 5   | - **Access:** SSH into private EC2 via Bastion or Session Manager.<br>- **Optimization:** Setup VPC Endpoints for S3/DynamoDB. | AWS Systems Manager |
| 6   | - **Verification:** Test connectivity (Public -> Internet, Private -> NAT).<br>- Weekly Review.                                | -                   |

### Week 2 Achievements:

- Deployed a custom VPC with isolated Public/Private subnets across 2 AZs.
- Configured VPC Endpoints to secure internal AWS service traffic.
- Replaced SSH access with secure Session Manager.
