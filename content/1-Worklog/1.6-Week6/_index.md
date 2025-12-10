---
title: "Week 6 - Database Service on AWS"
weight: 7
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

- Deploy and manage Relational (RDS) and NoSQL (DynamoDB) databases.
- Implement database migration strategies.

### Tasks to be carried out this week:

| Day | Task                                                                                                             | Reference      |
| --- | ---------------------------------------------------------------------------------------------------------------- | -------------- |
| 2   | - **RDS:** Launch RDS (PostgreSQL/MySQL) in private subnets.<br>- Configure Multi-AZ and Automated Backups.      | AWS RDS Docs   |
| 3   | - **Access:** Connect to RDS via SSM/Bastion securely.<br>- Monitor using Performance Insights.                  | AWS Docs       |
| 4   | - **DynamoDB:** Design tables (Partition/Sort Keys, GSIs).<br>- Test CRUD operations and TTL settings.           | DynamoDB Guide |
| 5   | - **Migration (Optional):** Test AWS DMS (Database Migration Service).<br>- Compare RDS vs DynamoDB performance. | AWS DMS Docs   |
| 6   | - **Review:** Validate database failover and backup recovery.<br>- Weekly Report.                                | -              |

### Week 6 Achievements:

- Deployed secure Multi-AZ RDS with automated backups.
- Designed optimized DynamoDB tables with appropriate keys and indexes.
- Implemented secure database access without public exposure.
