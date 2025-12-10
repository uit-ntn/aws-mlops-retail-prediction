---
title: "Tuần 8 - Điện toán Serverless"
weight: 9
chapter: false
pre: " <b> 1.8. </b> "
---

### Mục tiêu Tuần 8:

- Xây dựng ứng dụng hướng sự kiện (event-driven) bằng Lambda và API Gateway.
- Triển khai xử lý bất đồng bộ với SQS/SNS.

### Các nhiệm vụ thực hiện trong tuần này:

| Ngày | Nhiệm vụ                                                                                                                       | Tài liệu tham khảo |
| ---- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------ |
| 2    | - **Lambda:** Phát triển functions (Python/Node.js) có dùng environment variables.<br>- Cấu hình memory và timeouts.           | AWS Lambda Docs    |
| 3    | - **API Gateway:** Tạo HTTP/REST APIs tích hợp với Lambda.<br>- Thiết lập CORS và Stages.                                      | AWS APIGW Docs     |
| 4    | - **Event-Driven:** Cấu hình S3 Event Notifications để trigger Lambda.<br>- Thiết lập EventBridge rules.                       | AWS EventBridge    |
| 5    | - **Messaging:** Dùng SQS để tách rời (decoupling) và thiết lập DLQ để xử lý lỗi.<br>- Dùng SNS cho mô hình fan-out messaging. | AWS SQS/SNS        |
| 6    | - **Ôn tập:** Test luồng serverless end-to-end.<br>- Báo cáo tuần.                                                             | -                  |

### Kết quả đạt được Tuần 8:

- Phát triển serverless API bằng API Gateway và Lambda.
- Triển khai xử lý lỗi ổn định với SQS Dead Letter Queues.
- Giảm overhead vận hành nhờ tận dụng các managed services.
