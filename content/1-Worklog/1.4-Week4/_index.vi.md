---
title: "Tuần 4 - Dịch vụ lưu trữ trên AWS"
weight: 5
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu Tuần 4:

- Khám phá các lựa chọn lưu trữ nâng cao: S3, EFS và EBS.
- Triển khai phân phối nội dung với CloudFront.

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                   | Tài liệu tham khảo |
| ---- | -------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **S3 nâng cao:** Cấu hình Versioning, Lifecycle Policies và Encryption.<br>- Thiết lập Pre-signed URL để upload an toàn. | AWS S3 Docs        |
| 3    | - **CDN:** Thiết lập CloudFront với Origin Access Control (OAC) cho S3.<br>- Kiểm tra caching và invalidation.             | AWS CloudFront     |
| 4    | - **Lưu trữ file:** Tạo và mount Amazon EFS vào các EC2 instances.<br>- Cấu hình Access Points và Mount Targets.           | AWS EFS Docs       |
| 5    | - **Lưu trữ block:** Quản lý EBS Volumes (Snapshot, Restore, Resize).<br>- Triển khai kế hoạch Backup.                     | AWS Backup         |
| 6    | - **Ôn tập:** Xác thực phân phối nội dung an toàn và tính bền vững (persistence) của lưu trữ.<br>- Báo cáo tuần.           | -                  |

### Kết quả đạt được Tuần 4:

- Phục vụ nội dung tĩnh an toàn thông qua CloudFront + S3 OAC.
- Triển khai shared storage bằng EFS cho nhiều instances.
- Tự động hoá backup EBS bằng Data Lifecycle Manager.
