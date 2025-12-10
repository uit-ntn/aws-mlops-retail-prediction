---
title: "Công bố AWS Partner Central trong AWS Management Console"
weight: 10
chapter: false
pre: " <b> 3.10. </b> "
---

_Tác giả: Raj Kandaswamy, Nicole Schreiber, và Christin Voytko — 30 NOV 2025_

Bởi Raj Kandaswamy, Principal PMT-ES – AWS  
Bởi Nicole Schreiber, Partner Experience Lead – AWS  
Bởi Christin Voytko, Partner Experience Lead – AWS

Chúng tôi rất vui mừng thông báo rằng AWS Partner Central hiện đã có sẵn trong AWS Management Console, giúp đơn giản hóa việc truy cập Partner Central và AWS Marketplace Management Portal. Lần ra mắt này giới thiệu các API mạnh mẽ cung cấp khả năng tự động hóa quy trình và tích hợp, giúp Partners kết nối trực tiếp các hệ thống kinh doanh của họ với Partner Central, đồng thời tinh gọn cách họ phát triển và mở rộng cùng Amazon Web Services (AWS).

Được xây dựng trên AWS Identity and Access Management (IAM), trải nghiệm nâng cao này cung cấp các tính năng bảo mật cấp doanh nghiệp và khả năng quản lý người dùng linh hoạt cùng khả năng single sign-on (SSO)—giúp bạn dễ dàng hơn trong việc cung cấp cho đội ngũ quyền truy cập vào các công cụ họ cần để vận hành hoạt động kinh doanh AWS của bạn.

## Những điểm mới trong trải nghiệm Partner Central nâng cao

Trải nghiệm Partner Central mới được xây dựng để hỗ trợ các đội nhóm khi họ mở rộng cùng AWS. Bằng cách hợp nhất quyền truy cập và các hệ thống, các đội nhóm có thể điều hướng nhanh hơn đến các công cụ họ cần để đồng bán (co-sell) hiệu quả với AWS và các Partners khác. Trải nghiệm nâng cao này cho phép bạn:

### Chuyển đổi vận hành với tích hợp và tự động hóa mạnh mẽ

Kết nối trực tiếp các hệ thống kinh doanh hiện có của bạn với Partner Central thông qua các API được mở rộng, cho phép tích hợp liền mạch và tự động hóa luồng công việc. Loại bỏ việc nhập dữ liệu trùng lặp giữa đồng bán (co-selling) và các giao dịch AWS Marketplace, và tinh gọn vận hành trên toàn bộ quản lý giải pháp, đăng ký quyền lợi, và yêu cầu tài trợ. Các API của Partner Central bao gồm:

- **Account and Connections APIs** cho đăng ký Partner theo lập trình, quản lý hồ sơ, và cộng tác đa Partner thông qua các cơ hội chung (joint opportunities).
- **Solution APIs** để tạo và quản lý danh sách giải pháp (solution listings) trong AWS Marketplace.
- **Benefits APIs** cho việc khám phá quyền lợi tập trung và áp dụng quyền lợi xuyên suốt các chương trình Partner.
- **Selling and Leads APIs** hiện hỗ trợ toàn bộ vòng đời đồng bán—từ tạo lead và đánh giá đủ điều kiện đến chuyển đổi thành opportunity—với theo dõi tiến trình deal theo thời gian thực.

Các Partners đã và đang tận dụng các API này để tinh gọn vận hành. “Leveraging the Benefits API, we’ve streamlined the application process for Marketplace Private Offer Promotion Program (MPOPP) applications,” says Adam Boyle, VP of Product & Engineering at Tackle.io. “With this integration, users can now access pre-populated data, use AI-assisted justifications, and submit applications with a single click. These API capabilities allow sellers to manage fund requests from within the systems they are already working in on a day-to-day basis.”

Truy cập API Reference để bắt đầu tự xây dựng các tích hợp tùy chỉnh hoặc kết nối với một nhà cung cấp có thể giúp bạn triển khai tự động hóa cho Partner Central và AWS Marketplace.

### Nhận hướng dẫn cá nhân hóa bằng Amazon Q chat

Partner Assistant được nâng cấp, nay đã tích hợp với Amazon Q, cung cấp các phản hồi được “đo ni đóng giày” để giúp bạn điều hướng hành trình AWS Partner của mình. Nhận các khuyến nghị và insight cá nhân hóa dựa trên Partner Path, tier, giải pháp, pipeline opportunity, và danh sách sản phẩm (product listings). Hỏi câu hỏi bằng ngôn ngữ địa phương bạn ưu tiên về bất cứ điều gì, từ tối đa hóa việc sử dụng các quyền lợi AWS Partner đến kiểm tra trạng thái của một customer opportunity.

![B10img](/images/blog10/b1001.png "B10img")

### Mở rộng vận hành với kiểm soát truy cập linh hoạt

Thông qua AWS IAM, bạn có thể quản lý quyền một cách chính xác và sử dụng Workforce Identity Provider hiện có của bạn cho SSO. IAM cho bạn quyền kiểm soát chính xác đối với các thiết lập truy cập của người dùng mà không có các giới hạn truyền thống—như bị giới hạn chỉ một Alliance Lead. Triển khai các kiểm soát truy cập chi tiết (granular) để phù hợp với nhu cầu tổ chức của bạn, chẳng hạn như giới hạn các opportunity cụ thể cho các đội nhóm được chỉ định.

Với SSO, các thành viên trong nhóm của bạn có thể truy cập các công cụ và tài nguyên Partner bằng một bộ thông tin đăng nhập duy nhất. Truy cập hướng dẫn AWS IAM Identity Center để có hướng dẫn từng bước về việc kết nối với các identity providers. Các Partners sử dụng Okta có thể tận dụng tích hợp AWS Identity Center của Okta để tinh gọn onboarding người dùng và bật SSO trên toàn tổ chức của họ.

Các Partners đã và đang trải nghiệm những lợi ích này. “Arctic Wolf’s onboarding to AWS Partner Central in the AWS Console was incredibly smooth using AWS Identity Center with our existing identity provider—our IT department was able to provision users in minutes,” says Sean Phillips, VP of Strategic Alliances at Arctic Wolf. “I’m excited to no longer manage individual user permissions myself, knowing our IT team has the security controls and streamlined administration they need through IAM. We’re looking forward to leveraging the new API capabilities and automated workflows to transform how we manage our AWS partnership.”

### Mở rộng phạm vi tiếp cận với khả năng khám phá giải pháp trong AWS Marketplace

Trong Partner Central, giờ đây bạn có thể xuất bản các giải pháp đa sản phẩm, đa nhà cung cấp để công khai khám phá trong AWS Marketplace. Ngoài việc sử dụng Partner Connections để tìm các Partners bổ trợ cho hoạt động đồng bán, giờ đây bạn có thể tạo các giải pháp đa sản phẩm bao gồm nhiều thành phần phần mềm và dịch vụ từ nhiều nhà cung cấp, và niêm yết công khai trong AWS Marketplace. Khách hàng có thể khám phá các giải pháp của bạn trong AWS Marketplace bằng cách tìm kiếm theo vấn đề kinh doanh (chẳng hạn như “data migration”) thay vì theo các sản phẩm cụ thể, giúp họ dễ dàng hơn trong việc tìm đúng giải pháp.

Trong tương lai, khi chúng tôi tiếp tục nâng cao trải nghiệm cho Partners, chúng tôi đang đầu tư vào các khả năng ứng dụng AI để tăng tốc đánh giá kỹ thuật, luồng phê duyệt, và các chuyển động đồng bán—giúp bạn cải thiện hiệu quả vận hành và mở rộng hoạt động kinh doanh nhanh hơn với AWS.

## Bắt đầu

Truy cập trang web AWS Partner Central để tìm hiểu thêm về trải nghiệm mới. Các Partners mới có thể bắt đầu bằng cách đăng ký trở thành AWS Partner trong AWS Console.

Các Partners hiện có cần thực hiện hành động để chuyển sang trải nghiệm Partner Central mới:

- Phối hợp với bộ phận IT của bạn để liên kết tài khoản APN với một tài khoản AWS và lập kế hoạch chiến lược onboarding người dùng. Để onboard người dùng, hãy kết nối với Identity Provider hiện có của công ty bạn, sử dụng AWS IAM Identity Center, hoặc quản lý quyền trực tiếp trong IAM console.
- Lên lịch migration của bạn trong Partner Centralridge, Cloudsoft, CloudZone, Innovative Solutions, Labra, Rackspace, Spektra Systems, Suger, Tackle.io, và WorkSpan.
