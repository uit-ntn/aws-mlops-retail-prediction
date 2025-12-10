---
title: "Đơn giản hóa việc tạo chính sách IAM với IAM Policy Autopilot, một MCP server mã nguồn mở mới dành cho builders"
weight: 12
chapter: false
pre: " <b> 3.12. </b> "
---

_Tác giả: Micah Walter — 30 NOV 2025_

Hôm nay, chúng tôi công bố IAM Policy Autopilot, một máy chủ Model Context Protocol (MCP) mã nguồn mở mới phân tích mã ứng dụng của bạn và giúp các trợ lý lập trình AI của bạn tạo ra các chính sách dựa trên danh tính (identity-based) của AWS Identity and Access Management (IAM). IAM Policy Autopilot tăng tốc phát triển ban đầu bằng cách cung cấp cho builders một điểm khởi đầu mà họ có thể xem xét và tinh chỉnh thêm. Nó tích hợp với các trợ lý lập trình AI như Kiro, Claude Code, Cursor và Cline, và cung cấp cho chúng kiến thức về AWS Identity and Access Management (IAM) cũng như sự hiểu biết về các dịch vụ và tính năng AWS mới nhất. IAM Policy Autopilot có sẵn mà không tính thêm phí, chạy cục bộ, và bạn có thể bắt đầu bằng cách truy cập kho GitHub của chúng tôi.

Các ứng dụng Amazon Web Services (AWS) yêu cầu các chính sách IAM cho các vai trò (roles) của chúng. Builders trên AWS, từ nhà phát triển đến lãnh đạo doanh nghiệp, tương tác với IAM như một phần trong quy trình làm việc của họ. Các nhà phát triển thường bắt đầu với các quyền rộng hơn và tinh chỉnh chúng theo thời gian, cân bằng giữa phát triển nhanh và bảo mật. Họ thường sử dụng các trợ lý lập trình AI với hy vọng tăng tốc phát triển và soạn thảo các quyền IAM. Tuy nhiên, các công cụ AI này không hiểu đầy đủ các sắc thái của IAM và có thể bỏ sót quyền hoặc gợi ý các hành động không hợp lệ. Builders tìm kiếm các giải pháp cung cấp kiến thức IAM đáng tin cậy, tích hợp với các trợ lý AI và giúp họ bắt đầu việc tạo chính sách, để họ có thể tập trung xây dựng ứng dụng.

## Tạo chính sách hợp lệ với kiến thức AWS

IAM Policy Autopilot giải quyết các thách thức này bằng cách tạo các chính sách IAM dựa trên danh tính trực tiếp từ mã ứng dụng của bạn. Sử dụng phân tích mã mang tính xác định (deterministic code analysis), nó tạo ra các chính sách đáng tin cậy và hợp lệ, để bạn dành ít thời gian hơn cho việc viết và gỡ lỗi quyền. IAM Policy Autopilot tích hợp kiến thức AWS, bao gồm triển khai tham chiếu (reference implementation) của dịch vụ AWS đã được công bố, để luôn cập nhật. Nó sử dụng thông tin này để hiểu cách mã và các lệnh gọi SDK ánh xạ sang các hành động IAM và luôn theo kịp các dịch vụ và thao tác (operations) AWS mới nhất.

Các chính sách được tạo ra cung cấp một điểm khởi đầu để bạn xem xét và thu hẹp phạm vi nhằm triển khai quyền theo nguyên tắc đặc quyền tối thiểu (least privilege). Khi bạn sửa đổi mã ứng dụng của mình—dù là thêm tích hợp dịch vụ AWS mới hay cập nhật các tích hợp hiện có—bạn chỉ cần chạy IAM Policy Autopilot lại để nhận các quyền đã được cập nhật.

## Bắt đầu với IAM Policy Autopilot

Các nhà phát triển có thể bắt đầu với IAM Policy Autopilot chỉ trong vài phút bằng cách tải xuống và tích hợp nó vào quy trình làm việc của họ.

Là một máy chủ MCP, IAM Policy Autopilot vận hành ở chế độ nền khi builders trò chuyện với các trợ lý lập trình AI của họ. Khi ứng dụng của bạn cần các chính sách IAM, các trợ lý lập trình có thể gọi IAM Policy Autopilot để phân tích các lệnh gọi AWS SDK trong ứng dụng của bạn và tạo các chính sách IAM dựa trên danh tính cần thiết, cung cấp cho bạn các quyền cần có để bắt đầu. Sau khi tạo quyền, nếu bạn vẫn gặp lỗi Access Denied trong quá trình kiểm thử, trợ lý lập trình AI sẽ gọi IAM Policy Autopilot để phân tích sự từ chối và đề xuất các bản sửa chính sách IAM có mục tiêu. Sau khi bạn xem xét và phê duyệt các thay đổi được đề xuất, IAM Policy Autopilot sẽ cập nhật các quyền.

Bạn cũng có thể dùng IAM Policy Autopilot như một công cụ giao diện dòng lệnh (CLI) độc lập để tạo chính sách trực tiếp hoặc khắc phục các quyền bị thiếu. Cả công cụ CLI và máy chủ MCP đều cung cấp cùng các khả năng tạo chính sách và khắc phục sự cố, vì vậy bạn có thể chọn cách tích hợp phù hợp nhất với quy trình làm việc của mình.

Khi sử dụng IAM Policy Autopilot, bạn cũng nên hiểu các thực tiễn tốt nhất để tối đa hóa lợi ích của nó. IAM Policy Autopilot tạo các chính sách dựa trên danh tính và không tạo các chính sách dựa trên tài nguyên (resource-based policies), ranh giới quyền (permission boundaries), chính sách kiểm soát dịch vụ (SCPs) hay chính sách kiểm soát tài nguyên (RCPs). IAM Policy Autopilot tạo các chính sách ưu tiên chức năng hơn là quyền tối thiểu. Bạn nên luôn xem xét các chính sách được tạo ra và tinh chỉnh nếu cần để chúng phù hợp với các yêu cầu bảo mật của bạn trước khi triển khai.

## Hãy thử xem

Để thiết lập IAM Policy Autopilot, trước tiên tôi cần cài đặt nó trên hệ thống của mình. Để làm vậy, tôi chỉ cần chạy một lệnh một dòng:

```bash
curl -sSL https://github.com/awslabs/iam-policy-autopilot/raw/refs/heads/main/install.sh | sudo sh
```

Sau đó, tôi có thể làm theo hướng dẫn để cài đặt bất kỳ máy chủ MCP nào cho IDE mà tôi chọn. Hôm nay, tôi đang dùng Kiro!
![B12img](/images/blog12/b1201.png "B12img")
Trong một phiên chat mới trong Kiro, tôi bắt đầu với một prompt đơn giản, trong đó tôi yêu cầu Kiro đọc các tệp trong thư mục file-to-queue của tôi và tạo một tệp AWS CloudFormation mới để tôi có thể triển khai ứng dụng. Thư mục này chứa một bộ định tuyến tệp Amazon Simple Storage Service (Amazon S3) tự động, quét một bucket và gửi thông báo đến các hàng đợi Amazon Simple Queue Service (Amazon SQS) hoặc Amazon EventBridge dựa trên các quy tắc khớp tiền tố (prefix-matching) có thể cấu hình, cho phép các quy trình làm việc hướng sự kiện (event-driven) được kích hoạt bởi vị trí của tệp.

Phần cuối cùng yêu cầu Kiro đảm bảo tôi đang bao gồm các chính sách IAM cần thiết. Điều này sẽ đủ để Kiro sử dụng máy chủ MCP IAM Policy Autopilot.
![B12img](/images/blog12/b1202.png "B12img")
Tiếp theo, Kiro sử dụng máy chủ MCP IAM Policy Autopilot để tạo một tài liệu chính sách mới, như được minh họa trong hình ảnh sau. Sau khi xong, Kiro sẽ tiếp tục xây dựng mẫu CloudFormation của chúng ta và một số tài liệu bổ sung cũng như các tệp mã liên quan.
![B12img](/images/blog12/b1203.png "B12img")

Cuối cùng, chúng ta có thể thấy mẫu CloudFormation được tạo ra với một tài liệu chính sách mới, tất cả đều được tạo bằng máy chủ MCP IAM Policy Autopilot!
![B12img](/images/blog12/b1204.png "B12img")

## Quy trình phát triển được cải thiện

IAM Policy Autopilot tích hợp với các dịch vụ AWS trên nhiều lĩnh vực. Với các dịch vụ AWS cốt lõi, IAM Policy Autopilot phân tích việc ứng dụng của bạn sử dụng các dịch vụ như Amazon S3, AWS Lambda, Amazon DynamoDB, Amazon Elastic Compute Cloud (Amazon EC2) và Amazon CloudWatch Logs, sau đó tạo ra các quyền cần thiết mà mã của bạn cần dựa trên các lệnh gọi SDK mà nó phát hiện. Sau khi các chính sách được tạo, bạn có thể sao chép chính sách trực tiếp vào mẫu CloudFormation, stack AWS Cloud Development Kit (AWS CDK) hoặc cấu hình Terraform của bạn. Bạn cũng có thể prompt các trợ lý lập trình AI của mình để tích hợp nó cho bạn.

IAM Policy Autopilot cũng bổ trợ cho các công cụ IAM hiện có như AWS IAM Access Analyzer bằng cách cung cấp các chính sách mang tính chức năng như một điểm khởi đầu, sau đó bạn có thể xác thực bằng tính năng xác thực chính sách (policy validation) của IAM Access Analyzer hoặc tinh chỉnh theo thời gian với phân tích truy cập không sử dụng (unused access analysis).

## Hiện đã có sẵn

IAM Policy Autopilot hiện có sẵn dưới dạng một công cụ mã nguồn mở trên GitHub mà không tính thêm phí. Công cụ hiện hỗ trợ các ứng dụng Python, TypeScript và Go.

Những khả năng này đại diện cho một bước tiến đáng kể trong việc đơn giản hóa trải nghiệm phát triển trên AWS để các builders ở các mức kinh nghiệm khác nhau có thể phát triển và triển khai ứng dụng một cách hiệu quả hơn.
