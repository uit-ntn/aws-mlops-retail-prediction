---
title: "Self-Assessment"
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

Trong quá trình thực hiện workshop **Retail MLOps trên AWS**, tôi có cơ hội áp dụng kiến thức Cloud/DevOps/MLOps vào một pipeline thực tế end-to-end: từ tổ chức dữ liệu theo chuẩn S3, huấn luyện và đánh giá mô hình trên SageMaker, quản lý phiên bản mô hình bằng SageMaker Model Registry, đóng gói dịch vụ inference FastAPI bằng Docker, triển khai lên EKS, giám sát qua CloudWatch, tự động hoá CI/CD, và tối ưu chi phí bằng các quy trình teardown sau mỗi buổi demo.

Thông qua workshop này, tôi đã cải thiện các kỹ năng sau:

- Thiết kế kiến trúc MLOps theo hướng production (networking, security, scalability)
- Xây dựng pipeline từ dữ liệu → mô hình với tiêu chí chất lượng rõ ràng và bước kiểm chứng cụ thể
- Làm việc với các dịch vụ AWS: S3, SageMaker, ECR, EKS, IAM/IRSA, CloudWatch, VPC endpoints
- Container hoá dịch vụ inference (multi-stage build, non-root, healthcheck, scan image)
- Triển khai và vận hành workload Kubernetes (manifests, Service/Ingress, HPA, debug)
- Tối ưu chi phí và vận hành “gọn sạch” (Spot, lifecycle policies, log retention, schedule start/stop, teardown scripts)

Để phản ánh khách quan mức độ tiến bộ sau khi hoàn thành workshop, tôi tự đánh giá theo các tiêu chí dưới đây:

| STT | Tiêu chí                           | Mô tả                                                                                        | Tốt | Khá | Trung bình |
| --- | ---------------------------------- | -------------------------------------------------------------------------------------------- | --- | --- | ---------- |
| 1   | **Kiến thức & kỹ năng chuyên môn** | Hiểu khái niệm MLOps, triển khai pipeline train/deploy, chất lượng đầu ra                    | ✅  | ☐   | ☐          |
| 2   | **Khả năng học hỏi**               | Học và áp dụng AWS/EKS/SageMaker/ECR/CloudWatch nhanh                                        | ✅  | ☐   | ☐          |
| 3   | **Tính chủ động**                  | Chủ động debug (pods, image pull, IRSA), cải tiến cấu hình và cleanup                        | ✅  | ☐   | ☐          |
| 4   | **Tinh thần trách nhiệm**          | Hoàn thành theo checklist, đạt tiêu chí metric mô hình và yêu cầu verify                     | ✅  | ☐   | ☐          |
| 5   | **Tính kỷ luật**                   | Tuân thủ naming conventions, nhất quán region, thực hiện teardown để tránh phát sinh chi phí | ☐   | ✅  | ☐          |
| 6   | **Tinh thần cầu tiến**             | Tiếp nhận phản hồi và cải tiến (Spot, lifecycle, log retention, scheduling)                  | ✅  | ☐   | ☐          |
| 7   | **Giao tiếp**                      | Trình bày lựa chọn kiến trúc, báo cáo tiến độ, ghi chú bước verify rõ ràng                   | ☐   | ✅  | ☐          |
| 8   | **Làm việc nhóm (nếu áp dụng)**    | Phối hợp các phần (data/train/deploy/monitor/cost) và tích hợp deliverables                  | ☐   | ✅  | ☐          |
| 9   | **Tác phong chuyên nghiệp**        | Làm việc có hệ thống, tôn trọng quy ước, ưu tiên least privilege và bảo mật                  | ✅  | ☐   | ☐          |
| 10  | **Kỹ năng giải quyết vấn đề**      | Tìm root cause và đề xuất fix (VPC endpoints, IRSA/RBAC, LB/SG, logs)                        | ☐   | ✅  | ☐          |
| 11  | **Đóng góp cho dự án/nhóm**        | Tạo pipeline end-to-end có thể tái sử dụng kèm script vận hành/teardown                      | ✅  | ☐   | ☐          |
| 12  | **Tổng quan**                      | Mức độ hoàn thành tổng thể và khả năng tự tái hiện hệ thống                                  | ✅  | ☐   | ☐          |

## Cần cải thiện

- **Nâng chất lượng tài liệu hoá:** mô tả kiến trúc rõ hơn, checklist prerequisite, và bộ hướng dẫn troubleshooting có cấu trúc.
- **Tăng độ “chặt” cho CI/CD:** bổ sung các cổng kiểm soát (tests, policy checks, rollback rõ ràng), tách dev/stage/prod.
- **Nâng mức trưởng thành quan sát (observability):** dashboard/cảnh báo theo SLO thực tế (latency, error rate), tối ưu log và (tuỳ chọn) tracing.
- **Kiểm soát chi phí tốt hơn:** schedule theo khung giờ demo, kiểm soát log ingestion, dọn ECR định kỳ.
