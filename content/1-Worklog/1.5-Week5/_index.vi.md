---
title: "Tuần 5 - Dịch vụ bảo mật trên AWS"
weight: 6
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu Tuần 5:

- Triển khai các kiểm soát bảo mật toàn diện (IAM, KMS, WAF, Secrets).
- Tăng khả năng quan sát/giám sát với các công cụ audit (CloudTrail, Config, GuardDuty).

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                                                       | Tài liệu tham khảo |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **IAM & Secrets:** Rà soát IAM Roles theo nguyên tắc **Least Privilege**.<br>- Dùng Secrets Manager/Parameter Store để lưu thông tin xác thực (credentials). | AWS IAM/Secrets    |
| 3    | - **Mã hoá (Encryption):** Tạo Customer Managed Keys (CMK) trong KMS.<br>- Mã hoá S3/EBS bằng KMS keys.                                                        | AWS KMS Docs       |
| 4    | - **Bảo mật biên (Edge Security):** Triển khai AWS WAF (Web Application Firewall) cho ALB/CloudFront.<br>- Test rule chặn SQLi/XSS.                            | AWS WAF Docs       |
| 5    | - **Tuân thủ (Compliance):** Bật AWS Config và GuardDuty.<br>- Kiểm tra các phát hiện (findings) trên Security Hub.                                            | AWS Security Hub   |
| 6    | - **Ôn tập:** Thực hiện security audit và remediation.<br>- Báo cáo tuần.                                                                                      | -                  |

### Kết quả đạt được Tuần 5:

- Tập trung quản lý secrets bằng Secrets Manager/SSM.
- Bảo vệ endpoint bằng AWS WAF rules trước các tấn công phổ biến.
- Bật giám sát tuân thủ liên tục với AWS Config.
