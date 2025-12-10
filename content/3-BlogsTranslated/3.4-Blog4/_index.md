---
title: "Tích hợp dữ liệu S&P Global mở rộng khả năng Amazon QuickSight Research"
weight: 4
chapter: false
date: 2025-12-08
author: "Jon Haubauf, Brandon Portnivitte, và Prasanth Ponnoth"
categories: ["Artificial Intelligence"]
pre: " <b> 3.4. </b> "
---

*Bài viết gốc được đăng vào ngày 08 tháng 12 năm 2025*

Hôm nay, chúng tôi hân hạnh thông báo về tích hợp mới giữa Amazon QuickSight Research và S&P Global. Tích hợp này kết hợp tin tức năng lượng toàn cầu, nghiên cứu và thông tin chi tiết của S&P Global với khả năng thị trường toàn cầu S&P Global để cung cấp cho khách hàng QuickSight Research một đại lý nghiên cứu sâu.

Tích hợp S&P Global mở rộng khả năng của QuickSight Research để các chuyên gia kinh doanh có thể phân tích nhiều nguồn dữ liệu - bao gồm tin tức năng lượng toàn cầu và thông tin tài chính cao cấp - trong một không gian làm việc, loại bỏ nhu cầu chuyển đổi giữa các nền tảng và chuyển đổi hàng tuần nghiên cứu thành phút thông tin tập trung. QuickSight Suite kết nối thông tin trên các kho lưu trữ nội bộ, ứng dụng phổ biến, dịch vụ AWS và thông qua Giao thức Ngữ cảnh Mô hình (MCP) tích hợp với hơn 1.000 ứng dụng. Ứng dụng AI agentic này đang định hình lại cách thức công việc được thực hiện bằng cách chuyển đổi cách các nhóm tìm thông tin chi tiết, tiến hành nghiên cứu, tự động hóa nhiệm vụ, trực quan hóa dữ liệu và tìm thông tin chi tiết từ hàng terabyte ứng dụng và dữ liệu.

Trong bài viết này, chúng ta khám phá các bộ dữ liệu của S&P Global và kiến trúc giải pháp của tích hợp với QuickSight Research.

## Tổng quan giải pháp

S&P Global đã tiên phong hai triển khai máy chủ MCP trên AWS để các tổ chức có thể dễ dàng tích hợp các dịch vụ tài chính đáng tin cậy và nội dung năng lượng vào quy trình làm việc được hỗ trợ bởi AI trong khi duy trì chất lượng, bảo mật và độ tin cậy mà các nhà lãnh đạo kinh doanh yêu cầu.

*"Sự hợp tác của chúng tôi với AWS mở rộng cách S&P Global cung cấp thông tin đáng tin cậy thông qua thế hệ tiếp theo của trải nghiệm AI agentic. Bằng cách làm việc cùng với các công ty AI hàng đầu, mục tiêu của chúng tôi là đảm bảo khách hàng có thể truy cập dữ liệu đáng tin cậy và thông tin chi tiết bất cứ nơi nào quy trình làm việc của họ diễn ra."*

**— Bhavesh Dayalji, Giám đốc Công nghệ của S&P Global và CEO của Kensho.**

## S&P Global Energy: Thông tin toàn diện về hàng hóa và năng lượng

Tích hợp S&P Global Energy, hiện có sẵn trong Amazon QuickSight Research, sử dụng máy chủ MCP AI Ready Data để cung cấp quyền truy cập toàn diện vào thông tin thị trường năng lượng bao gồm Crude, Clean Fuels, Natural Gas, Power, Coal, Metals, Chemicals, LNG, Petrochemicals, Clean Energy, Nông nghiệp và Vận chuyển trên các thị trường toàn cầu. Được xây dựng dựa trên danh tiếng của S&P Global như một cơ quan thị trường đáng tin cậy, máy chủ MCP sử dụng hàng trăm nghìn tài liệu do chuyên gia tạo bao gồm phân tích, bình luận, tin tức và nghiên cứu.

Giải pháp cung cấp góc nhìn đa chiều độc đáo, cung cấp thông tin từ cập nhật thị trường hàng ngày đến triển vọng một năm và mở rộng đến phân tích kịch bản 20+ năm. Với dữ liệu được làm mới mỗi 30 phút, các nhà lãnh đạo kinh doanh có quyền truy cập gần thời gian thực vào thông tin hàng hóa và năng lượng, tăng tốc đáng kể tốc độ quyết định khi khám phá các thách thức quy định, cơ hội đầu tư hoặc tác động môi trường.

## S&P Global Market Intelligence: Thông tin tài chính đáng tin cậy

Tích hợp S&P Global Market Intelligence, hiện có sẵn trong Amazon QuickSight Research, sử dụng máy chủ MCP sẵn sàng API Kensho LLM để cung cấp quyền truy cập vào trung tâm tài chính đổi mới của S&P Global. Máy chủ MCP làm cho dữ liệu tài chính đáng tin cậy có thể truy cập thông qua các truy vấn ngôn ngữ tự nhiên, tích hợp liền mạch với Amazon QuickSight Research. Các chuyên gia tài chính có thể truy cập S&P Capital IQ Financials, bản ghi cuộc gọi thu nhập, thông tin công ty, giao dịch và nhiều hơn nữa, chỉ bằng cách đặt câu hỏi.

Giải pháp Kensho giải quyết một thách thức quan trọng trong dịch vụ tài chính: làm cho các kho lưu trữ rộng lớn của dữ liệu tài chính có thể truy cập ngay lập tức mà không yêu cầu hàng giờ truy vấn hoặc chuyên môn kỹ thuật. Các nhóm kỹ thuật, sản phẩm và kinh doanh có thể tiết kiệm thời gian và tài nguyên đáng kể bằng cách chuyển đổi những gì từng yêu cầu hàng giờ trích xuất dữ liệu thành các cuộc trò chuyện trả về thông tin chính xác, đáng tin cậy trong vài giây.

## Kiến trúc giải pháp

Kiến trúc máy chủ MCP của S&P Global được hiển thị trong sơ đồ sau. Khi sử dụng một trong các tích hợp S&P, lưu lượng chảy từ QuickSight Research thông qua Amazon API Gateway đến AWS Application Load Balancer với các dịch vụ MCP được lưu trữ trên Amazon Elastic Kubernetes Service (Amazon EKS). Máy chủ MCP sử dụng dữ liệu được lưu trữ trong Amazon S3 và AWS Relational Database Service cho PostgreSQL cho dữ liệu có cấu trúc, và Amazon OpenSearch Service cho lưu trữ vector. Kiến trúc này cung cấp các máy chủ MCP sẵn sàng cho doanh nghiệp với bảo mật chuyên sâu, tự động mở rộng và khả năng quan sát toàn diện.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2025/11/21/ML-20016-image-1.png)

MCP là một tiêu chuẩn mở hỗ trợ giao tiếp liền mạch giữa các đại lý AI và nguồn dữ liệu bên ngoài, công cụ và dịch vụ. MCP hoạt động trên kiến trúc client-server nơi máy chủ MCP xử lý các cuộc gọi công cụ, thường bao gồm nhiều cuộc gọi API và hiển thị triển khai logic kinh doanh như các hàm có thể gọi. Điều này cho phép các đại lý AI khám phá khả năng một cách động, đàm phán các tính năng và chia sẻ ngữ cảnh một cách an toàn với tất cả các yêu cầu quan trọng cho các ứng dụng cấp doanh nghiệp.

Giải pháp của S&P Global có các khối xây dựng chính sau:

### Đường ống dữ liệu tự động với Amazon Bedrock
Trọng tâm của giải pháp là một đường ống Retrieval Augmented Generation (RAG) tự động hóa việc nhập dữ liệu bằng Amazon Textract. Điều này chuyển đổi dữ liệu thị trường thô thành các nhúng vector AI Ready sử dụng mô hình Cohere Embed được lưu trữ trên Bedrock. Các tài liệu từ kho lưu trữ độc quyền của S&P Global trải qua xử lý trước, phân tách và làm giàu trước khi được chuyển đổi thành các nhúng vector bằng cách sử dụng mô hình Cohere Embed được lưu trữ trên Bedrock. Đường ống nhập chạy theo lịch trình, làm mới kho lưu trữ vector OpenSearch mỗi 30 phút để truy cập gần thời gian thực vào dữ liệu năng lượng.

### Tìm kiếm vector và ngữ nghĩa
Amazon OpenSearch phục vụ như cơ sở dữ liệu vector, lưu trữ các nhúng được tạo bởi Bedrock và cho phép khả năng tìm kiếm ngữ nghĩa trên dữ liệu năng lượng của S&P Global. Kho lưu trữ vector OpenSearch được tối ưu hóa cho các tìm kiếm tương tự cao chiều cho phép tìm kiếm tương tự nhanh chóng mang lại khả năng truy xuất thông tin liên quan theo ngữ cảnh để phản hồi các truy vấn ngôn ngữ tự nhiên.

### Khả năng phục hồi và quy mô
Giải pháp này sử dụng Amazon EKS để lưu trữ tất cả các giải pháp máy chủ MCP với hai cụm sản xuất cho phép phân chia lưu lượng và khả năng chuyển đổi dự phòng. Cách tiếp cận cụm kép này cung cấp tính khả dụng liên tục ngay cả trong các lỗi không mong đợi. Cả Cluster Autoscaler và Horizontal Pod Autoscaler đều cho phép mở rộng động dựa trên nhu cầu. Các máy chủ tự động mở rộng với khung FastMCP, cung cấp hiệu suất cao và các endpoint có khả năng phục hồi tuân thủ với đặc tả Streamable HTTP Transport được yêu cầu bởi giao thức MCP.

### Bảo mật
Bảo mật được tích hợp vào mọi lớp của giải pháp. API Gateway phục vụ như điểm cuối cho truy cập máy chủ MCP. Danh tính doanh nghiệp của S&P Global được cung cấp bởi OAuth authentication. API Gateway được bảo mật hơn nữa với AWS Web Application Firewall (WAF) với phát hiện mối đe dọa nâng cao. Các vai trò và chính sách AWS IAM thực thi đặc quyền ít nhất, để mỗi thành phần chỉ có các quyền mà nó yêu cầu. AWS Secrets Manager lưu trữ an toàn thông tin đăng nhập, chứng chỉ và dịch vụ API. AWS Security Groups và VPC cung cấp cách ly mạng, trong khi TLS 1.2+ với AWS Certificate Manager xác thực tất cả dữ liệu trong quá trình truyền vẫn được mã hóa. Bảo mật đa lớp này bao gồm các điều khiển bảo mật chuyên sâu.

### Khả năng quan sát
Amazon CloudWatch cung cấp ghi nhật ký tập trung, thu thập số liệu và giám sát thời gian thực của toàn bộ đường ống từ việc nhập dữ liệu thông qua phản hồi máy chủ MCP. AWS CloudTrail ghi lại các nhật ký hoạt động API chi tiết và dấu vết kiểm toán, cần thiết cho việc tuân thủ trong các ngành được quy định.

## Kết luận

Cùng nhau, những máy chủ MCP này được xây dựng trên AWS và tích hợp vào Amazon QuickSight Research thể hiện tầm nhìn của S&P Global cho tương lai của dịch vụ tài chính và thông tin năng lượng: duy trì sự tin cậy, chính xác và độ sâu mà các nhà lãnh đạo kinh doanh yêu cầu trong khi nắm bắt tiềm năng chuyển đổi của AI để làm cho thông tin đó dễ tiếp cận, có thể thực hiện và tích hợp vào quy trình làm việc hiện đại.

**Link bài viết gốc** : https://aws.amazon.com/blogs/machine-learning/sp-global-data-integration-expands-amazon-quick-research-capabilities/