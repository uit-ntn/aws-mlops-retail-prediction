---
title: "Tuần 2 - Mạng (Networking) trên AWS"
weight: 3
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu Tuần 2:

- Thiết kế và triển khai Virtual Private Cloud (VPC).
- Cấu hình định tuyến (routing), các gateway (IGW, NAT) và lớp bảo mật (Security Group, NACL).

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                                | Tài liệu tham khảo  |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| 2    | - **Thiết kế VPC:** Lập kế hoạch CIDR cho 2 AZ (Public/Private Subnets).<br>- Tạo VPC, Subnets và Route Tables.                         | AWS VPC Docs        |
| 3    | - **Kết nối:** Gắn Internet Gateway (IGW) cho public subnets.<br>- Triển khai NAT Gateway để private subnet truy cập Internet (egress). | AWS Workshop        |
| 4    | - **Bảo mật:** Cấu hình Security Groups (stateful) vs NACLs (stateless).<br>- Áp dụng mô hình **Bastion Host**.                         | FCJ Materials       |
| 5    | - **Truy cập:** SSH vào private EC2 thông qua Bastion hoặc Session Manager.<br>- **Tối ưu:** Thiết lập VPC Endpoints cho S3/DynamoDB.   | AWS Systems Manager |
| 6    | - **Xác minh:** Kiểm tra kết nối (Public -> Internet, Private -> NAT).<br>- Tổng kết tuần.                                              | -                   |

### Kết quả đạt được Tuần 2:

- Triển kha
