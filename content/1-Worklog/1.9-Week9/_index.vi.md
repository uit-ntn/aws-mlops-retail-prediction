---
title: "Tuần 9 - Điện toán Container"
weight: 10
chapter: false
pre: " <b> 1.9. </b> "
---

### Mục tiêu Tuần 9:

- Nắm vững Containerization (Docker) và Orchestration (ECS/Fargate).
- Quản lý container images bằng ECR.

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                | Tài liệu tham khảo |
| ---- | ------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **Docker:** Tạo Multi-stage Dockerfiles để tối ưu ứng dụng.<br>- Build và test images trên máy local. | Docker Docs        |
| 3    | - **ECR:** Thiết lập Private ECR Repositories kèm Lifecycle policies.<br>- Push images lên ECR.         | AWS ECR Docs       |
| 4    | - **ECS:** Tạo ECS Cluster và Task Definitions (Fargate).<br>- Định nghĩa Task Role/Execution Role.     | AWS ECS Docs       |
| 5    | - **Service:** Triển khai ECS Service tích hợp với ALB.<br>- Cấu hình Auto Scaling.                     | AWS ELB/ASG        |
| 6    | - **Ôn tập:** Xác minh việc triển khai container và hành vi scaling.<br>- Báo cáo tuần.                 | -                  |

### Kết quả đạt được Tuần 9:

- Tối ưu container images bằng multi-stage builds.
- Triển khai ứng dụng container hóa có khả năng mở rộng trên ECS Fargate.
- Bảo mật giao tiếp container bằng Security Groups và IAM Roles.
