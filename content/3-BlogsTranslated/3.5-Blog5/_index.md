---
title: "Cách Văn phòng Tổng chưởng lý Quận Contra Costa hiện đại hóa quy trình xử lý trát đòi hầu tòa với AWS và CC Tech Digital"
weight: 5
chapter: false
date: 2025-12-08
author: "Ekta Vasa, Khalil Ahmed, Nati Abebe, và Saurabh Kapoor"
categories: ["AWS Public Sector Blog"]
pre: " <b> 3.5. </b> "
---

*Bài viết gốc được đăng vào ngày 08 tháng 12 năm 2025*

Chuyển đổi số trong chính phủ không còn là tùy chọn - đó là điều bắt buộc. Văn phòng Tổng chưởng lý Quận Contra Costa đã hợp tác với AWS (Amazon Web Services) và CC Tech Digital, một Đối tác Cấp cao AWS, để hiện đại hóa quy trình xử lý trát đòi hầu tòa bằng giải pháp serverless, cloud-native trên AWS.

Bằng cách tự động hóa việc tải lên tài liệu, tích hợp trực tiếp với Microsoft Word, và định tuyến an toàn trát đòi hầu tòa đến các cơ quan đúng, hệ thống mới hiện quản lý hơn 17.000 trát đòi hầu tòa hàng năm - với tốc độ, độ chính xác và tuân thủ được cải thiện.

## Thách thức: Quy trình trát đòi hầu tòa thủ công làm chậm hoạt động

Trước khi hiện đại hóa, việc xử lý trát đòi hầu tòa ở Quận Contra Costa chủ yếu dựa vào quy trình làm việc thủ công gây ra sự kém hiệu quả tại nhiều điểm tiếp xúc:

- **Trát đòi hầu tòa được tải lên các chia sẻ tệp tại chỗ**
- **Tệp PDF được chia tách và xem xét thông qua các bước thủ công**
- **Giao tiếp email dựa trên danh sách liên hệ tĩnh**
- **Khả năng theo dõi và báo cáo tối thiểu**
- **Phối hợp liên hệ thống đòi hỏi sự tham gia đáng kể của IT**

Mặc dù cách tiếp cận này hoạt động trong nhiều năm, nhưng nó tốn thời gian và dễ có sự không nhất quán. Khi nhu cầu minh bạch và giao tiếp pháp lý kịp thời tăng lên, những quy trình làm việc thủ công này trở nên khó duy trì hơn.

## Giải pháp: Nền tảng trát đòi hầu tòa serverless, cloud-native được xây dựng trên AWS

Để giải quyết những thách thức này, CC Tech Digital và AWS đã cung cấp một ứng dụng hoàn toàn tự động, cloud-native được hỗ trợ bởi kiến trúc serverless của AWS. Giải pháp này tích hợp liền mạch với các hệ thống quận hiện có, bao gồm Microsoft Word, và chỉ yêu cầu đào tạo tối thiểu cho nhân viên pháp lý.

### Gửi trát đòi hầu tòa được hợp lý hóa qua Microsoft Word

Nhân viên pháp lý hiện có thể tải lên trát đòi hầu tòa trực tiếp từ Word bằng Add-In Microsoft Office được tùy chỉnh. Chỉ với một cú nhấp chuột, người dùng gửi tài liệu vào pipeline xử lý - mà không cần rời khỏi không gian làm việc quen thuộc của họ. Add-in xác thực định dạng tài liệu, số trang và các trường bắt buộc theo thời gian thực, giảm các lần gửi thất bại và đảm bảo tuân thủ ban đầu.

![](https://d2908q01vomqb2.cloudfront.net/9e6a55b6b4563e652a23be9d623ca5055c356940/2025/12/02/Picture16.png)
*Hình 1. Mẫu tài liệu trát đòi hầu tòa và cổng tải lên Add-in Microsoft*

### Xử lý PDF thông minh và gửi email trên AWS

Khi một tài liệu được tải lên, một pipeline serverless hoàn toàn sẽ đảm nhận:

1. **Phát hiện tài liệu**  
   Các tệp được tải lên Amazon Simple Storage Service (Amazon S3) sẽ được phát hiện bởi AWS Lambda, kích hoạt quy trình làm việc xử lý.

2. **Chia tách PDF tự động**  
   Trát đòi hầu tòa nhiều trang (20-30 trang) được chia thành nhiều PDF, với mỗi cơ quan chỉ nhận hai trang liên quan, theo chính sách pháp lý.

3. **Trích xuất trường dựa trên AI**  
   Sử dụng Amazon Textract, các trường chính như số vụ án, tên cơ quan và metadata trát đòi hầu tòa được trích xuất từ tài liệu.

4. **Định tuyến động và gửi email**  
   Một tra cứu được thực hiện trong Amazon DynamoDB để xác định người nhận cơ quan đúng. PDF tương ứng sau đó được gửi email tự động qua Amazon Simple Email Service (Amazon SES), loại bỏ định tuyến thủ công.

5. **Xử lý lỗi và thông báo quản trị viên**  
   Nếu hệ thống gặp phải sự không khớp hoặc lỗi, nó tự động thông báo cho admin với chẩn đoán lỗi và hướng dẫn.

![](https://d2908q01vomqb2.cloudfront.net/9e6a55b6b4563e652a23be9d623ca5055c356940/2025/12/02/Picture17.png)
*Hình 2. Mẫu email với thông tin đính kèm được cá nhân hóa cho người nhận*

## Bảo mật, tuân thủ và tối ưu hóa chi phí được tích hợp sẵn

Bảo mật và tuân thủ là nền tảng cho thiết kế của giải pháp:

- **AWS WAF**: Quyền truy cập tải lên được hạn chế đối với các địa chỉ IP được ủy quyền của Quận Contra Costa, giảm rủi ro gửi không được ủy quyền.

- **Amazon S3**: Tất cả tài liệu được mã hóa khi lưu trữ bằng AWS Key Management Service (KMS) với các mô-đun mã hóa đã được xác thực FIPS 140-2 và được mã hóa trong quá trình truyền tải qua TLS bằng các điểm cuối tuân thủ FIPS.

- **AWS Identity and Access Management (IAM)**: Các chính sách dựa trên vai trò đảm bảo rằng chỉ nhân viên được ủy quyền mới có thể tải lên tệp trát đòi hầu tòa.

- **Chính sách vòng đời Amazon S3**: Tài liệu tự động chuyển đổi sang lưu trữ lạnh tiết kiệm chi phí (ví dụ: S3 Glacier) sau 90 ngày để đáp ứng nhu cầu lưu giữ dữ liệu và ngân sách.

Những tính năng này giúp Văn phòng Tổng chưởng lý Quận Contra Costa đáp ứng các tiêu chuẩn bảo vệ dữ liệu nghiêm ngặt mà không làm phức tạp cho nhân viên hoặc nhóm IT.

## Kết quả: Tác động có thể đo lường trong toàn tổ chức

Bảng dưới đây nổi bật các kết quả được Văn phòng Tổng chưởng lý Quận Contra Costa đạt được sau khi triển khai giải pháp serverless, cloud-native mới:

| Danh mục | Tác động |
|----------|----------|
| **Hiệu quả** | Giảm các tác vụ thủ công hơn 80% |
| **Độ chính xác** | Cải thiện định tuyến tài liệu với AI và tra cứu động |
| **Trải nghiệm người dùng** | Tải lên một cú nhấp trực tiếp từ Word với phản hồi thời gian thực |
| **Tuân thủ & Bảo mật** | Kiểm soát truy cập được thực thi, lưu trữ được mã hóa và khả năng hiển thị kiểm toán đầy đủ |
| **Khả năng mở rộng** | Không có cơ sở hạ tầng để quản lý; xử lý hơn 17.000 trát đòi hầu tòa hàng năm - và con số này dự kiến sẽ tăng |

*"CC Tech đã giúp chúng tôi chuyển đổi cách chúng tôi xử lý trát đòi hầu tòa - cải thiện hiệu quả, minh bạch và tuân thủ với giải pháp cloud-native hiện đại trên AWS."*  
**— James Mount, Giám đốc IT, Văn phòng Tổng chưởng lý Quận Contra Costa**

## Kết luận: Một bản thiết kế có thể mở rộng cho hiện đại hóa khu vực công

Dự án này là một mô hình về cách các cơ quan chính phủ địa phương có thể hiện đại hóa quy trình làm việc tài liệu bằng AWS. Văn phòng Tổng chưởng lý Quận Contra Costa hiện được hưởng lợi từ một nền tảng trát đòi hầu tòa hoàn toàn tự động, được tăng cường bởi AI, loại bỏ các bước thủ công, cải thiện tuân thủ và mở rộng dễ dàng theo nhu cầu.

Bằng cách hợp tác với CC Tech Digital và tận dụng các dịch vụ AWS như Amazon Textract, AWS Lambda, Amazon SES và AWS WAF, các tổ chức khu vực công có thể hiện đại hóa các hoạt động quan trọng trong tương lai mà không làm gián đoạn các công cụ hoặc nhóm hiện có của họ.

**Link bài viết gốc** : https://aws.amazon.com/blogs/publicsector/how-contra-costa-county-district-attorneys-office-modernized-subpoena-processing-with-aws-and-cc-tech-digital/ 