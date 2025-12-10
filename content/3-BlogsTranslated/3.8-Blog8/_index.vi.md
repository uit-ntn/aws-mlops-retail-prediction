---
title: "AWS Clean Rooms ra mắt tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư để huấn luyện mô hình ML"
weight: 8
chapter: false
pre: " <b> 3.8. </b> "
---

_Tác giả: Micah Walter — 30 NOV 2025_

Hôm nay, chúng tôi công bố tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư cho AWS Clean Rooms, một khả năng mới mà các tổ chức và đối tác của họ có thể sử dụng để tạo các bộ dữ liệu tổng hợp tăng cường quyền riêng tư từ dữ liệu chung của họ nhằm huấn luyện các mô hình máy học (ML) hồi quy và phân loại. Bạn có thể sử dụng tính năng này để tạo các bộ dữ liệu huấn luyện tổng hợp vẫn giữ lại các mẫu thống kê của dữ liệu gốc, mà mô hình không cần truy cập vào các bản ghi gốc, mở ra những cơ hội mới cho việc huấn luyện mô hình vốn trước đây không thể thực hiện do lo ngại về quyền riêng tư.

Khi xây dựng mô hình ML, các nhà khoa học dữ liệu và nhà phân tích thường đối mặt với một mâu thuẫn nền tảng giữa tính hữu ích của dữ liệu và bảo vệ quyền riêng tư. Việc truy cập dữ liệu chất lượng cao, chi tiết là điều thiết yếu để huấn luyện các mô hình chính xác có thể nhận diện xu hướng, cá nhân hóa trải nghiệm và thúc đẩy kết quả kinh doanh. Tuy nhiên, việc sử dụng dữ liệu chi tiết như dữ liệu sự kiện ở mức người dùng từ nhiều bên làm phát sinh các lo ngại lớn về quyền riêng tư và các thách thức tuân thủ. Các tổ chức muốn trả lời những câu hỏi như, “Những đặc điểm nào cho thấy khả năng chuyển đổi khách hàng cao?”, nhưng việc huấn luyện dựa trên các tín hiệu ở mức cá nhân thường mâu thuẫn với chính sách quyền riêng tư và các yêu cầu pháp lý.

## Tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư cho ML tùy chỉnh

Để giải quyết thách thức này, chúng tôi giới thiệu tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư trong AWS Clean Rooms ML, cho phép các tổ chức tạo ra các phiên bản tổng hợp của các bộ dữ liệu nhạy cảm để có thể được sử dụng an toàn hơn cho việc huấn luyện mô hình ML. Khả năng này sử dụng các kỹ thuật ML tiên tiến để tạo ra các bộ dữ liệu mới duy trì các thuộc tính thống kê của dữ liệu gốc trong khi loại bỏ định danh (de-identifying) các đối tượng từ dữ liệu nguồn ban đầu.

Các kỹ thuật ẩn danh truyền thống như che (masking) vẫn mang rủi ro tái nhận diện cá nhân trong một bộ dữ liệu—biết các thuộc tính về một người như mã bưu chính và ngày sinh có thể đủ để xác định họ khi đối chiếu với dữ liệu điều tra dân số. Tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư giải quyết rủi ro này bằng một cách tiếp cận khác về bản chất. Hệ thống huấn luyện một mô hình học các mẫu thống kê cốt lõi của bộ dữ liệu gốc, sau đó tạo các bản ghi tổng hợp bằng cách lấy mẫu các giá trị từ bộ dữ liệu gốc và dùng mô hình để dự đoán cột giá trị được dự đoán (predicted value column). Thay vì chỉ sao chép hoặc làm nhiễu dữ liệu gốc, hệ thống sử dụng một kỹ thuật giảm năng lực mô hình (model capacity reduction) để giảm thiểu rủi ro mô hình ghi nhớ thông tin về các cá nhân trong dữ liệu huấn luyện. Bộ dữ liệu tổng hợp thu được có cùng schema và đặc điểm thống kê như dữ liệu gốc, khiến nó phù hợp để huấn luyện các mô hình phân loại và hồi quy. Cách tiếp cận này giảm định lượng (quantifiably) rủi ro tái nhận diện.

Các tổ chức sử dụng khả năng này có quyền kiểm soát các tham số quyền riêng tư, bao gồm lượng nhiễu (noise) được áp dụng và mức bảo vệ chống lại các cuộc tấn công suy luận thành viên (membership inference attacks), trong đó kẻ đối địch cố gắng xác định liệu dữ liệu của một cá nhân cụ thể có được đưa vào tập huấn luyện hay không. Sau khi tạo bộ dữ liệu tổng hợp, AWS Clean Rooms cung cấp các chỉ số chi tiết để giúp khách hàng và các nhóm tuân thủ của họ hiểu chất lượng của bộ dữ liệu tổng hợp trên hai chiều quan trọng: độ trung thực so với dữ liệu gốc và mức bảo toàn quyền riêng tư. Điểm độ trung thực (fidelity score) sử dụng KL-divergence để đo mức độ giống nhau giữa dữ liệu tổng hợp và bộ dữ liệu gốc, và điểm quyền riêng tư (privacy score) định lượng mức độ bộ dữ liệu được bảo vệ trước các cuộc tấn công suy luận thành viên.

## Làm việc với dữ liệu tổng hợp trong AWS Clean Rooms

Việc bắt đầu với tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư tuân theo quy trình làm việc custom models của AWS Clean Rooms ML hiện có, với các bước mới để chỉ định yêu cầu quyền riêng tư và xem các chỉ số chất lượng. Các tổ chức bắt đầu bằng cách tạo các bảng đã cấu hình (configured tables) với các quy tắc phân tích (analysis rules) bằng các nguồn dữ liệu ưa thích của họ, sau đó join hoặc tạo một collaboration với các đối tác của họ và liên kết (associate) các bảng của họ với collaboration đó.

Khả năng mới giới thiệu một mẫu phân tích (analysis template) nâng cao, nơi các chủ sở hữu dữ liệu không chỉ xác định truy vấn SQL tạo bộ dữ liệu mà còn chỉ định rằng bộ dữ liệu kết quả phải là tổng hợp. Trong mẫu này, các tổ chức phân loại các cột để chỉ ra cột nào mô hình ML sẽ dự đoán và cột nào chứa giá trị dạng phân loại (categorical) so với dạng số (numerical). Một điểm quan trọng là mẫu cũng bao gồm các ngưỡng quyền riêng tư mà dữ liệu tổng hợp được tạo ra phải đáp ứng để có thể được cung cấp cho việc huấn luyện. Các ngưỡng này bao gồm một giá trị epsilon chỉ định lượng nhiễu cần có trong dữ liệu tổng hợp để bảo vệ chống tái nhận diện, và một điểm bảo vệ tối thiểu chống lại các cuộc tấn công suy luận thành viên. Việc đặt các ngưỡng này một cách phù hợp đòi hỏi phải hiểu các yêu cầu quyền riêng tư và tuân thủ cụ thể của tổ chức bạn, và chúng tôi khuyến nghị phối hợp với các nhóm pháp lý và tuân thủ trong quá trình này.

Sau khi tất cả các chủ sở hữu dữ liệu xem xét và phê duyệt mẫu phân tích, một thành viên collaboration tạo một kênh đầu vào máy học (machine learning input channel) tham chiếu đến mẫu đó. AWS Clean Rooms sau đó bắt đầu quy trình tạo bộ dữ liệu tổng hợp, thường hoàn tất trong vài giờ tùy thuộc vào kích thước và độ phức tạp của bộ dữ liệu. Nếu bộ dữ liệu tổng hợp được tạo ra đáp ứng các ngưỡng quyền riêng tư bắt buộc được xác định trong mẫu phân tích, một kênh đầu vào máy học tổng hợp (synthetic machine learning input channel) sẽ trở nên khả dụng cùng với các chỉ số chất lượng chi tiết. Các nhà khoa học dữ liệu có thể xem lại điểm bảo vệ thực tế đạt được trước một cuộc tấn công suy luận thành viên mô phỏng.

Khi đã hài lòng với các chỉ số chất lượng, các tổ chức có thể tiếp tục huấn luyện các mô hình ML của họ bằng bộ dữ liệu tổng hợp trong collaboration của AWS Clean Rooms. Tùy thuộc vào trường hợp sử dụng, họ có thể xuất các trọng số mô hình đã huấn luyện hoặc tiếp tục chạy các tác vụ suy luận (inference jobs) ngay trong chính collaboration.

## Hãy thử xem

Khi tạo một collaboration AWS Clean Rooms mới, giờ đây tôi có thể đặt ai là người trả tiền cho việc tạo bộ dữ liệu tổng hợp.
![B8img](/images/blog8/b801.png "B8img")
Sau khi Collaboration của tôi được cấu hình, tôi có thể chọn Require analysis template output to be synthetic khi tạo một analysis template mới.
![B8img](/images/blog8/b802.png "B8img")
Sau khi mẫu phân tích tổng hợp của tôi sẵn sàng, tôi có thể dùng nó khi chạy các truy vấn được bảo vệ (protected queries) và xem tất cả các chi tiết kênh đầu vào ML liên quan.
![B8img](/images/blog8/b803.png "B8img")
Clean Rooms Synthetic Data Console

## Hiện đã có sẵn

Bạn có thể bắt đầu sử dụng tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư thông qua AWS Clean Rooms ngay hôm nay. Tính năng này có sẵn ở tất cả các AWS Regions thương mại nơi AWS Clean Rooms có sẵn. Tìm hiểu thêm trong tài liệu AWS Clean Rooms.

Tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư được tính phí riêng dựa trên mức sử dụng. Bạn chỉ trả tiền cho phần compute được sử dụng để tạo bộ dữ liệu tổng hợp của bạn, được tính phí theo Synthetic Data Generation Units (SDGUs). Số lượng SDGUs thay đổi tùy theo kích thước và độ phức tạp của bộ dữ liệu gốc. Khoản phí này có thể được cấu hình như một thiết lập payer, nghĩa là bất kỳ thành viên collaboration nào cũng có thể đồng ý trả chi phí. Để biết thêm thông tin về giá, hãy tham khảo trang giá AWS Clean Rooms.

Bản phát hành ban đầu hỗ trợ huấn luyện các mô hình phân loại và hồi quy trên dữ liệu dạng bảng (tabular). Các bộ dữ liệu tổng hợp hoạt động với các framework ML tiêu chuẩn và có thể tích hợp vào các pipeline phát triển mô hình hiện có mà không yêu cầu thay đổi quy trình làm việc của bạn.

Khả năng này đại diện cho một bước tiến đáng kể trong máy học tăng cường quyền riêng tư. Các tổ chức có thể khai phá giá trị của dữ liệu nhạy cảm ở mức người dùng cho việc huấn luyện mô hình trong khi giảm thiểu rủi ro thông tin nhạy cảm về từng người dùng có thể bị rò rỉ. Dù bạn đang tối ưu hóa các chiến dịch quảng cáo, cá nhân hóa báo giá bảo hiểm hay nâng cao các hệ thống phát hiện gian lận, tính năng tạo bộ dữ liệu tổng hợp tăng cường quyền riêng tư giúp việc huấn luyện các mô hình chính xác hơn thông qua hợp tác dữ liệu trở nên khả thi, đồng thời tôn trọng quyền riêng tư cá nhân.
