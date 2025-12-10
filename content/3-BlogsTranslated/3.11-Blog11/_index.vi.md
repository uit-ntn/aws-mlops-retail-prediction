---
title: "Công bố Amazon EKS Capabilities cho điều phối workload và quản lý tài nguyên đám mây"
weight: 11
chapter: false
pre: " <b> 3.11. </b> "
---

_Tác giả: Channy Yun (윤석찬) — 30 NOV 2025_

Hôm nay, chúng tôi công bố Amazon Elastic Kubernetes Service (Amazon EKS) Capabilities, một tập hợp có thể mở rộng các giải pháp gốc Kubernetes (Kubernetes-native) giúp đơn giản hóa việc điều phối workload, quản lý tài nguyên đám mây Amazon Web Services (AWS), cũng như việc kết hợp (composition) và điều phối tài nguyên Kubernetes. Các năng lực nền tảng được quản lý toàn phần (fully managed) và tích hợp này bao gồm các giải pháp Kubernetes mã nguồn mở mà nhiều khách hàng đang sử dụng hiện nay, chẳng hạn như Argo CD, AWS Controllers for Kubernetes và Kube Resource Orchestrator.

Với EKS Capabilities, bạn có thể xây dựng và mở rộng các ứng dụng Kubernetes mà không cần quản lý cơ sở hạ tầng giải pháp phức tạp. Khác với các cài đặt thông thường trong cụm (in-cluster installations), các capabilities này thực tế chạy trong các tài khoản do dịch vụ EKS sở hữu và được trừu tượng hóa hoàn toàn khỏi khách hàng.

Vì AWS quản lý việc mở rộng quy mô hạ tầng, vá lỗi và cập nhật các năng lực của cụm này, bạn có thể tận dụng độ tin cậy và bảo mật cấp doanh nghiệp mà không cần phải duy trì và quản lý các thành phần nền tảng bên dưới.

Dưới đây là các capabilities có sẵn tại thời điểm ra mắt:

- **Argo CD** – Đây là một công cụ GitOps dạng khai báo (declarative) cho Kubernetes, cung cấp các khả năng triển khai liên tục (continuous deployment, CD) cho Kubernetes. Công cụ này được áp dụng rộng rãi, với hơn 45% người dùng cuối Kubernetes báo cáo đang dùng trong môi trường production hoặc dự định dùng production trong Khảo sát Cloud Native Computing Foundation (CNCF) năm 2024.
- **AWS Controllers for Kubernetes (ACK)** – ACK rất phổ biến với các nhóm nền tảng doanh nghiệp trong môi trường production. ACK cung cấp các tài nguyên tùy chỉnh (custom resources) cho Kubernetes, cho phép quản lý các tài nguyên AWS Cloud trực tiếp từ bên trong các cụm của bạn.
- **Kube Resource Orchestrator (KRO)** – KRO cung cấp một cách thức tinh gọn để tạo và quản lý các tài nguyên tùy chỉnh trong Kubernetes. Với KRO, các nhóm nền tảng có thể tạo các gói tài nguyên có thể tái sử dụng (reusable resource bundles) nhằm trừu tượng hóa sự phức tạp trong khi vẫn giữ tính native với hệ sinh thái Kubernetes.

Với các tính năng này, bạn có thể tăng tốc và mở rộng việc sử dụng Kubernetes của mình với các capabilities được quản lý toàn phần, sử dụng các tính năng mang tính định hướng (opinionated) nhưng linh hoạt để xây dựng cho khả năng mở rộng ngay từ đầu. Bộ này được thiết kế để cung cấp một tập các năng lực nền tảng của cụm có thể xếp lớp (layer) với nhau một cách liền mạch, mang lại các tính năng tích hợp cho triển khai liên tục, điều phối tài nguyên và kết hợp. Bạn có thể tập trung vào việc quản lý và phát hành phần mềm mà không cần phải dành thời gian và tài nguyên để xây dựng và quản lý các thành phần nền tảng của nền tảng (platform) này.

## Cách thức hoạt động

Các kỹ sư nền tảng (platform engineers) và quản trị viên cụm (cluster administrators) có thể thiết lập EKS Capabilities để giảm tải việc xây dựng và quản lý các giải pháp tùy chỉnh nhằm cung cấp các dịch vụ nền tảng phổ biến, nghĩa là họ có thể tập trung vào các tính năng khác biệt hơn quan trọng đối với doanh nghiệp của bạn.

Các nhà phát triển ứng dụng của bạn chủ yếu làm việc với EKS Capabilities giống như với các tính năng Kubernetes khác. Họ làm điều này bằng cách áp dụng cấu hình dạng khai báo (declarative configuration) để tạo các tài nguyên Kubernetes bằng các công cụ quen thuộc, chẳng hạn như kubectl hoặc thông qua tự động hóa từ git commit đến mã chạy.
![B11img](/images/blog11/b1101.png "B11img")

## Bắt đầu với EKS Capabilities

Để bật EKS Capabilities, bạn có thể dùng EKS console, AWS Command Line Interface (AWS CLI), eksctl hoặc các công cụ ưa thích khác. Trong EKS console, chọn Create capabilities trong tab Capabilities trên cụm EKS hiện có của bạn. EKS Capabilities là các tài nguyên AWS, và chúng có thể được gắn thẻ (tagged), quản lý và xóa.
![B11img](/images/blog11/b1102.png "B11img")
Bạn có thể chọn một hoặc nhiều capabilities để phối hợp cùng nhau. Tôi đã chọn cả ba capabilities: ArgoCD, ACK và KRO. Tuy nhiên, các capabilities này hoàn toàn độc lập và bạn có thể chọn bật capabilities nào bạn muốn trên các cụm của mình.
![B11img](/images/blog11/b1103.png "B11img")
Bây giờ bạn có thể cấu hình các capabilities đã chọn. Bạn nên tạo các vai trò AWS Identity and Access Management (AWS IAM) để cho phép EKS vận hành các capabilities này trong cụm của bạn. Xin lưu ý rằng bạn không thể sửa đổi tên capability, namespace, vùng xác thực (authentication region) hoặc phiên bản AWS IAM Identity Center sau khi tạo capability. Chọn Next, xem lại các thiết lập và bật capabilities.
![B11img](/images/blog11/b1104.png "B11img")
Giờ đây bạn có thể xem và quản lý các capabilities đã tạo. Chọn ArgoCD để cập nhật cấu hình của capability.
![B11img](/images/blog11/b1105.png "B11img")
Bạn có thể xem chi tiết capability ArgoCD. Chọn Edit để thay đổi các thiết lập cấu hình hoặc Monitor ArgoCD để hiển thị trạng thái sức khỏe của capability cho cụm EKS hiện tại.
![B11img](/images/blog11/b1106.png "B11img")
Chọn Go to Argo UI để trực quan hóa và theo dõi trạng thái triển khai và tình trạng ứng dụng.
![B11img](/images/blog11/b1107.png "B11img")
Để tìm hiểu thêm về cách thiết lập và sử dụng chi tiết từng capability, hãy truy cập Getting started with EKS Capabilities trong Amazon EKS User Guide.

## Những điều cần biết

Dưới đây là các điểm cần lưu ý chính về tính năng này:

- **Quyền (Permissions)** – EKS Capabilities là các tài nguyên quản trị phạm vi cụm (cluster-scoped administrator resources), và quyền của tài nguyên được cấu hình thông qua AWS IAM. Với một số capabilities, có thêm cấu hình cho single sign-on. Ví dụ, cấu hình single sign-on của Argo CD được bật trực tiếp trong EKS với tích hợp trực tiếp với IAM Identity Center.
- **Nâng cấp (Upgrades)** – EKS tự động cập nhật các cluster capabilities bạn bật và các phụ thuộc liên quan của chúng. Nó tự động phân tích các thay đổi có thể gây lỗi tương thích (breaking changes), vá và cập nhật các thành phần khi cần, và thông báo cho bạn về xung đột hoặc vấn đề thông qua EKS cluster insights.
- **Tiếp nhận (Adoptions)** – ACK cung cấp các tính năng tiếp nhận tài nguyên (resource adoption) cho phép di chuyển các tài nguyên AWS hiện có vào phạm vi quản lý của ACK. ACK cũng cung cấp các tài nguyên chỉ đọc (read-only resources) có thể giúp hỗ trợ một lộ trình di chuyển theo từng bước từ các tài nguyên được cấp phát bằng Terraform, AWS CloudFormation sang EKS Capabilities.

## Hiện đã có sẵn

Amazon EKS Capabilities hiện đã có sẵn tại các AWS Regions thương mại. Để biết khả dụng theo vùng và lộ trình tương lai, hãy truy cập AWS Capabilities by Region. Không có cam kết trả trước hoặc phí tối thiểu, và bạn chỉ trả tiền cho EKS Capabilities và các tài nguyên mà bạn sử dụng. Để tìm hiểu thêm, hãy truy cập trang EKS pricing.

Hãy thử trong Amazon EKS console và gửi phản hồi đến AWS re:Post cho EKS hoặc thông qua các kênh AWS Support thông thường của bạn.
