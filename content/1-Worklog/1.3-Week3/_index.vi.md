---
title: "Tuần 3 - Dịch vụ máy ảo/Compute (VM) trên AWS"
weight: 4
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu Tuần 3:

- Thành thạo triển khai và quản lý EC2.
- Triển khai tính sẵn sàng cao (High Availability) với Auto Scaling (ASG) và Load Balancing (ALB).

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                                   | Tài liệu tham khảo |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------ |
| 2    | - **EC2:** Tạo Launch Templates kèm User Data scripts.<br>- Cấu hình IAM Roles cho EC2 (truy cập S3).                                      | AWS EC2 Docs       |
| 3    | - **Cân bằng tải (Load Balancing):** Triển khai Application Load Balancer (ALB).<br>- Cấu hình Target Groups và Health Checks (`/health`). | AWS ELB Docs       |
| 4    | - **Auto Scaling:** Tạo Auto Scaling Group (ASG) trên Multi-AZ.<br>- Thiết lập scaling policies (ví dụ: CPU > 50%).                        | AWS ASG Docs       |
| 5    | - **Giám sát (Monitoring):** Cài CloudWatch Agent qua User Data.<br>- Test các sự kiện scaling (stress test).                              | Hands-on Lab       |
| 6    | - **Ôn tập:** Xác minh ALB routing và ASG auto scaling động.<br>- Báo cáo tuần.                                                            | -                  |

### Kết quả đạt được Tuần 3:

- Tự động hoá provisioning EC2 bằng Launch
