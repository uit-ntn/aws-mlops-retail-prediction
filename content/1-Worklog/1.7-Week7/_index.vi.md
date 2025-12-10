---
title: "Tuần 7 - Dịch vụ Data Lake trên AWS"
weight: 8
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu Tuần 7:

- Xây dựng Data Lake serverless trên S3.
- Triển khai pipeline ETL với Glue và phân tích dữ liệu bằng Athena.

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                             | Tài liệu tham khảo |
| ---- | -------------------------------------------------------------------------------------------------------------------- | ------------------ |
| 2    | - **Data Lake:** Thiết kế các lớp S3 (Raw, Staging, Curated).<br>- Xây dựng chiến lược partitioning.                 | Task 3 Guide       |
| 3    | - **Glue Catalog:** Thiết lập Glue Crawlers để tự động phát hiện schema.<br>- Xây dựng Glue Data Catalog.            | AWS Glue Docs      |
| 4    | - **ETL:** Tạo Glue Jobs (Visual/Spark) để chuyển CSV sang Parquet.<br>- Triển khai data partitioning.               | AWS Glue Studio    |
| 5    | - **Phân tích (Analytics):** Query dữ liệu bằng Amazon Athena.<br>- Trực quan hoá kết quả với QuickSight (tuỳ chọn). | AWS Athena Docs    |
| 6    | - **Ôn tập:** Kiểm tra hiệu quả pipeline và hiệu năng truy vấn.<br>- Báo cáo tuần.                                   | -                  |

### Kết quả đạt được Tuần 7:

- Thiết lập kiến trúc Data Lake nhiều lớp (multi-layer).
- Tối ưu lưu trữ và hiệu năng truy vấn bằng định dạng Parquet.
- Cho phép truy vấn SQL linh hoạt (ad-hoc) trực tiếp trên dữ liệu S3 bằng Athena.
