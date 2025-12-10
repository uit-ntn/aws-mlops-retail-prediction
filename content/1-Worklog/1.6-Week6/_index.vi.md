---
title: "Tuần 6 - Dịch vụ cơ sở dữ liệu trên AWS"
weight: 7
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu Tuần 6:

- Triển khai và quản lý CSDL quan hệ (RDS) và NoSQL (DynamoDB).
- Xây dựng chiến lược migrate cơ sở dữ liệu.

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                       | Tài liệu tham khảo |
| ---- | -------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **RDS:** Khởi tạo RDS (PostgreSQL/MySQL) trong private subnets.<br>- Cấu hình Multi-AZ và Automated Backups. | AWS RDS Docs       |
| 3    | - **Truy cập:** Kết nối RDS an toàn qua SSM/Bastion.<br>- Theo dõi bằng Performance Insights.                  | AWS Docs           |
| 4    | - **DynamoDB:** Thiết kế bảng (Partition/Sort Keys, GSIs).<br>- Test CRUD operations và cấu hình TTL.          | DynamoDB Guide     |
| 5    | - **Migration (Tuỳ chọn):** Test AWS DMS (Database Migration Service).<br>- So sánh hiệu năng RDS vs DynamoDB. | AWS DMS Docs       |
| 6    | - **Ôn tập:** Xác minh failover và khôi phục từ backup.<br>- Báo cáo tuần.                                     | -                  |

### Kết quả đạt được Tuần 6:

- Triển khai RDS Multi-AZ an toàn kèm automated backups.
- Thiết kế DynamoDB tối ưu với keys và indexes phù hợp.
- Thiết lập truy cập CSDL an toàn, không public ra Internet.
