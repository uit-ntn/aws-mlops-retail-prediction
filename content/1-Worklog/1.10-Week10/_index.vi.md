---
title: "Tuần 10 - DevOps & IaC"
weight: 11
chapter: false
pre: " <b> 1.10. </b> "
---

### Mục tiêu Tuần 10:

- Triển khai Infrastructure as Code (Terraform/CloudFormation).
- Thiết lập CI/CD pipelines (CodePipeline/GitHub Actions).

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                          | Tài liệu tham khảo |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **IaC:** Viết Terraform cho các tài nguyên VPC, SG và IAM.<br>- Quản lý state bằng S3 Backend.                  | Terraform Docs     |
| 3    | - **Thiết lập CI/CD:** Cấu hình CodeCommit/GitHub repo.<br>- Thiết lập CodeBuild projects để build Docker images. | AWS CodeBuild      |
| 4    | - **Pipeline:** Tạo CodePipeline để tự động hoá Build -> Deploy.<br>- Thêm các bước Manual Approval.              | AWS CodePipeline   |
| 5    | - **Bảo mật:** Tích hợp quản lý secrets (SSM/Secrets Manager) vào pipeline.<br>- Xác minh việc deploy tự động.    | FinOps Guide       |
| 6    | - **Ôn tập:** Test toàn bộ luồng CI/CD từ commit đến deploy.<br>- Báo cáo tuần.                                   | -                  |

### Kết quả đạt được Tuần 10:

- Tự động hoá provisioning hạ tầng bằng Terraform.
- Xây dựng CI/CD pipeline tự động cho việc triển khai container.
- Loại bỏ hardcoded secrets trong các script triển khai.
