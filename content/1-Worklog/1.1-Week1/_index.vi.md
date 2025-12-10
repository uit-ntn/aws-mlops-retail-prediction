---
title: "Tuần 1 - Giới thiệu về AWS"
weight: 2
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu Tuần 1:

- Làm quen với Hạ tầng Toàn cầu của AWS (Regions, AZs) và mô hình **Chia sẻ trách nhiệm** (Shared Responsibility Model).
- Thành thạo cấu hình AWS Console & AWS CLI.
- Thực hành các nguyên tắc bảo mật cơ bản (IAM User/Role, MFA).

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                                                    | Tài liệu tham khảo |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **Thiết lập:** Tạo tài khoản AWS, bật MFA cho root user.<br>- **IAM:** Tạo IAM Groups/Users theo nguyên tắc **Least Privilege**.                          | AWS IAM Docs       |
| 3    | - **CLI:** Cài AWS CLI v2, cấu hình profiles (`~/.aws/config`, `credentials`).<br>- **Kiểm tra:** Chạy `aws sts get-caller-identity` để xác thực đăng nhập. | AWS CLI Guide      |
| 4    | - **S3 cơ bản:** Tạo bucket với **Block Public Access** được bật.<br>- Thực hành upload/download object qua Console & CLI.                                  | AWS S3 Docs        |
| 5    | - **Bảo mật:** Học mô hình **Shared Responsibility Model**.<br>- Định nghĩa quy ước tagging (Owner, Project, Env).                                          | FCJ Materials      |
| 6    | - **Ôn tập:** Ghi lại quy trình setup và kiểm tra cảnh báo billing.<br>- Báo cáo tuần.                                                                      | -                  |

### Kết quả đạt được Tuần 1:

- Bảo vệ tài khoản AWS bằng MFA và không dùng root cho các tác vụ hằng ngày.
- Cấu hình AWS CLI profiles cho nhiều môi trường.
- Tạo S3 bucket an toàn, bật mã hoá và versioning.
