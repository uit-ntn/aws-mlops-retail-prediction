---
title: "Cách tùy chỉnh phản hồi đối với các cuộc tấn công DDoS tầng 7 bằng AWS WAF Anti-DDoS AMR"
weight: 2
chapter: false
date: "2025-12-10"
author: "Achraf Souk"
categories: ["Advanced", "AWS WAF", "Security", "Identity & Compliance"]
pre: " <b> 3.2. </b> "
---

*Bởi Achraf Souk | 09 tháng 12 năm 2025 | trong Advanced (300), AWS WAF, Security, Identity, & Compliance*

Trong nửa đầu năm nay, AWS WAF đã giới thiệu các biện pháp bảo vệ tầng ứng dụng mới để giải quyết xu hướng gia tăng của các cuộc tấn công từ chối dịch vụ phân tán (DDoS) tầng 7 (L7) có thời gian tồn tại ngắn và thông lượng cao. Các biện pháp bảo vệ này được cung cấp thông qua nhóm quy tắc AWS WAF Anti-DDoS AWS Managed Rules (Anti-DDoS AMR). Mặc dù cấu hình mặc định có hiệu quả đối với hầu hết các khối lượng công việc, bạn có thể muốn điều chỉnh phản hồi để phù hợp với mức độ chấp nhận rủi ro của ứng dụng.

Trong bài viết này, bạn sẽ học cách Anti-DDoS AMR hoạt động và cách bạn có thể tùy chỉnh hành vi của nó bằng cách sử dụng nhãn và các quy tắc AWS WAF bổ sung. Bạn sẽ được hướng dẫn qua ba tình huống thực tế, mỗi tình huống minh họa một kỹ thuật tùy chỉnh khác nhau.

## Cách thức hoạt động của Anti-DDoS AMR

Hình 1, khi Anti-DDoS AMR phát hiện một cuộc tấn công DDoS, nó thêm nhãn `event-detected` vào tất cả các yêu cầu đến và nhãn `ddos-request` vào các yêu cầu đến được nghi ngờ góp phần vào cuộc tấn công. Nó cũng thêm một nhãn dựa trên mức độ tin cậy bổ sung, chẳng hạn như `high-suspicion-ddos-request`, khi yêu cầu được nghi ngờ góp phần vào cuộc tấn công. Trong AWS WAF, nhãn là siêu dữ liệu được thêm vào một yêu cầu bởi một quy tắc khi quy tắc phù hợp với yêu cầu. Sau khi được thêm, nhãn có sẵn cho các quy tắc tiếp theo, có thể sử dụng nó để làm phong phú logic đánh giá của chúng. Anti-DDoS AMR sử dụng các nhãn được thêm để giảm thiểu cuộc tấn công DDoS.

![Hình 1 - Quy trình Anti-DDoS AMR](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/11/25/Customize-response-AWS-WAF-1.jpeg)

*Hình 1 – Quy trình của Anti-DDoS AMR*

Các biện pháp giảm thiểu mặc định dựa trên sự kết hợp của các hành động `Block` và `JavaScript Challenge`. Hành động `Challenge` chỉ có thể được xử lý đúng cách bởi một máy khách đang mong đợi nội dung HTML. Vì lý do này, bạn cần loại trừ các đường dẫn của các yêu cầu không thể thách thức (chẳng hạn như lấy API) trong cấu hình Anti-DDoS AMR. Anti-DDoS AMR áp dụng nhãn `challengeable-request` cho các yêu cầu không khớp với các loại trừ thách thức được cấu hình. Theo mặc định, các quy tắc giảm thiểu sau được đánh giá theo thứ tự:

• **ChallengeAllDuringEvent**, tương đương với logic sau: IF `event-detected` AND `challengeable-request` THEN challenge.

• **ChallengeDDoSRequests**, tương đương với logic sau: If (`high-suspicion-ddos-request` OR `medium-suspicion-ddos-request` OR `low-suspicion-ddos-request`) AND `challengeable-request` THEN challenge. Độ nhạy của nó có thể được thay đổi để phù hợp với nhu cầu của bạn, chẳng hạn như chỉ thách thức các yêu cầu DDoS có mức độ nghi ngờ trung bình và cao.

• **DDoSRequests**, tương đương với logic sau: IF `high-suspicion-ddos-request` THEN block. Độ nhạy của nó có thể được thay đổi để phù hợp với nhu cầu của bạn, chẳng hạn như chặn các yêu cầu DDoS có mức độ nghi ngờ trung bình ngoài việc có mức độ nghi ngờ cao.

## Tùy chỉnh phản hồi đối với các cuộc tấn công DDoS tầng 7

Việc tùy chỉnh này có thể được thực hiện bằng hai cách tiếp cận khác nhau. Trong cách tiếp cận đầu tiên, bạn cấu hình Anti-DDoS AMR để thực hiện hành động mà bạn muốn, sau đó bạn thêm các quy tắc tiếp theo để củng cố thêm phản hồi của bạn trong các điều kiện nhất định. Trong cách tiếp cận thứ hai, bạn thay đổi một số hoặc tất cả các quy tắc của Anti-DDoS AMR thành chế độ đếm, sau đó tạo các quy tắc bổ sung xác định phản hồi của bạn đối với các cuộc tấn công DDoS.

Trong cả hai cách tiếp cận, các quy tắc tiếp theo được cấu hình bằng cách sử dụng các điều kiện mà bạn xác định, kết hợp với các điều kiện dựa trên nhãn áp dụng cho các yêu cầu bởi Anti-DDoS AMR. Phần sau bao gồm ba ví dụ về việc tùy chỉnh phản hồi của bạn đối với các cuộc tấn công DDoS. Hai ví dụ đầu tiên dựa trên cách tiếp cận đầu tiên, trong khi ví dụ cuối cùng dựa trên cách tiếp cận thứ hai.

## Ví dụ 1: Giảm thiểu nhạy cảm hơn bên ngoài các quốc gia cốt lõi

Giả sử rằng hoạt động kinh doanh chính của bạn được thực hiện ở hai quốc gia chính, UAE và KSA. Bạn hài lòng với hành vi mặc định của Anti-DDoS AMR ở các quốc gia này, nhưng bạn muốn chặn tích cực hơn bên ngoài các quốc gia này. Bạn có thể triển khai điều này bằng cách sử dụng các quy tắc sau:

• Anti-DDoS AMR với cấu hình mặc định
• Một quy tắc tùy chỉnh chặn nếu các điều kiện sau được đáp ứng: Yêu cầu được khởi tạo từ bên ngoài UAE hoặc KSA VÀ yêu cầu có nhãn `high-suspicion-ddos-request` hoặc `medium-suspicion-ddos-request`

### Cấu hình

Sau khi Anti-DDoS AMR của bạn với cấu hình mặc định, tạo một quy tắc tùy chỉnh tiếp theo với định nghĩa JSON sau:

**Lưu ý**: Bạn cần sử dụng trình chỉnh sửa quy tắc JSON của AWS WAF hoặc các công cụ infrastructure-as-code (IaC) để định nghĩa quy tắc này. Bảng điều khiển AWS WAF hiện tại không cho phép tạo quy tắc với logic AND/OR lồng nhau.

```json
{
  "Action": {
    "Block": {}
  },
  "Name": "more-sensitive-ddos-mitigation-outside-of-core-countries",
  "Priority": 1,
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          "NotStatement": {
            "Statement": {
              "GeoMatchStatement": {
                "CountryCodes": [
                  "AE",
                  "SA"
                ]
              }
            }
          }
        },
        {
          "OrStatement": {
            "Statements": [
              {
                "LabelMatchStatement": {
                  "Key": "awswaf:managed:aws:anti-ddos:high-suspicion-ddos-request",
                  "Scope": "LABEL"
                }
              },
              {
                "LabelMatchStatement": {
                  "Key": "awswaf:managed:aws:anti-ddos:medium-suspicion-ddos-request",
                  "Scope": "LABEL"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

## Ví dụ 2: Ngưỡng giới hạn tốc độ thấp hơn trong các cuộc tấn công DDoS

Giả sử rằng ứng dụng của bạn có các URL nhạy cảm tốn nhiều tài nguyên tính toán. Để bảo vệ tính khả dụng của ứng dụng, bạn đã áp dụng một quy tắc giới hạn tốc độ cho các URL này được cấu hình với ngưỡng 100 yêu cầu trong cửa sổ 2 phút. Bạn có thể củng cố phản hồi này trong một cuộc tấn công DDoS bằng cách áp dụng ngưỡng tích cực hơn. Bạn có thể triển khai điều này bằng cách sử dụng các quy tắc sau:

1. Một Anti-DDoS AMR với cấu hình mặc định
2. Một quy tắc giới hạn tốc độ, có phạm vi cho các URL nhạy cảm, được cấu hình với ngưỡng 100 yêu cầu trong cửa sổ 2 phút
3. Một quy tắc giới hạn tốc độ, có phạm vi cho các URL nhạy cảm và nhãn `event-detected`, được cấu hình với ngưỡng 10 yêu cầu trong cửa sổ 10 phút

### Cấu hình

Sau khi thêm Anti-DDoS AMR của bạn với cấu hình mặc định và quy tắc giới hạn tốc độ của bạn cho các URL nhạy cảm, tạo một quy tắc giới hạn tốc độ mới tiếp theo với định nghĩa JSON sau:

```json
{
  "Action": {
    "Block": {}
  },
  "Name": "ip-rate-limit-10-10mins-under-ddos",
  "Priority": 2,
  "Statement": {
    "RateBasedStatement": {
      "AggregateKeyType": "IP",
      "EvaluationWindowSec": 600,
      "Limit": 10,
      "ScopeDownStatement": {
        "AndStatement": {
          "Statements": [
            {
              "ByteMatchStatement": {
                "FieldToMatch": {
                  "UriPath": {}
                },
                "PositionalConstraint": "STARTS_WITH",
                "SearchString": "/sensitive-path",
                "TextTransformations": [
                  {
                    "Priority": 0,
                    "Type": "LOWERCASE"
                  }
                ]
              }
            },
            {
              "LabelMatchStatement": {
                "Key": "awswaf:managed:aws:anti-ddos:event-detected",
                "Scope": "LABEL"
              }
            }
          ]
        }
      }
    }
  }
}
```

## Ví dụ 3: Phản hồi thích ứng theo khả năng mở rộng ứng dụng của bạn

Giả sử rằng bạn đang vận hành một ứng dụng kế thừa có thể mở rộng an toàn đến một ngưỡng nhất định về khối lượng lưu lượng, sau đó nó bị suy giảm. Nếu tổng khối lượng lưu lượng, bao gồm cả lưu lượng DDoS, nằm dưới ngưỡng này, bạn quyết định không thách thức yêu cầu trong cuộc tấn công DDoS để tránh làm suy giảm trải nghiệm người dùng. Trong tình huống này, bạn chỉ dựa vào hành động chặn mặc định của các yêu cầu DDoS có mức độ nghi ngờ cao. Nếu tổng khối lượng lưu lượng vượt quá ngưỡng an toàn của ứng dụng kế thừa để xử lý lưu lượng, thì bạn quyết định sử dụng tương đương với biện pháp giảm thiểu ChallengeDDoSRequests mặc định của Anti-DDoS AMR. Bạn có thể triển khai điều này bằng cách sử dụng các quy tắc sau:

1. Một Anti-DDoS AMR với quy tắc `ChallengeAllDuringEvent` và `ChallengeDDoSRequests` được cấu hình ở chế độ đếm.
2. Một quy tắc giới hạn tốc độ đếm lưu lượng của bạn và được cấu hình với ngưỡng tương ứng với khả năng của ứng dụng để xử lý lưu lượng bình thường. Như một hành động, nó chỉ đếm yêu cầu và áp dụng nhãn tùy chỉnh—ví dụ, `CapacityExceeded`—khi đạt ngưỡng.
3. Một quy tắc mô phỏng `ChallengeDDoSRequests` nhưng chỉ khi nhãn `CapacityExceeded` có mặt: Thách thức nếu các nhãn `ddos-request`, `CapacityExceeded`, và `challengeable-request` có mặt

### Cấu hình

Đầu tiên, cập nhật Anti-DDoS AMR của bạn bằng cách thay đổi các hành động Challenge thành các hành động Count.

![Hình 2 – Quy tắc Anti-DDoS AMR đã cập nhật trong ví dụ 3](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/11/25/Customize-response-AWS-WAF-2.png)

*Hình 2 – Quy tắc Anti-DDoS AMR đã cập nhật trong ví dụ 3*

Sau đó tạo quy tắc giới hạn tốc độ `capacity-exceeded-detection` ở chế độ đếm, sử dụng định nghĩa JSON sau:

```json
{
  "Priority": 7,
  "RuleLabels": [
    {
      "Name": "mycompany:capacityexceeded"
    }
  ],
  "Statement": {
    "RateBasedStatement": {
      "AggregateKeyType": "IP",
      "EvaluationWindowSec": 120,
      "Limit": 10000
    }
  },
  "VisibilityConfig": {
    "CloudWatchMetricsEnabled": true,
    "MetricName": "capacity-exceeded-detection",
    "SampledRequestsEnabled": true
  }
}
```

Cuối cùng, tạo quy tắc `challenge-if-ddos-and-capacity-exceeded` challenge bằng định nghĩa JSON sau:

```json
{
  "Action": {
    "Challenge": {}
  },
  "Name": "challenge-if-ddos-and-capacity-exceeded",
  "Priority": 2,
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          "LabelMatchStatement": {
            "Key": "mycompany:capacityexceeded",
            "Scope": "LABEL"
          }
        },
        {
          "LabelMatchStatement": {
            "Key": "awswaf:managed:aws:anti-ddos:ddos-request",
            "Scope": "LABEL"
          }
        },
        {
          "LabelMatchStatement": {
            "Key": "awswaf:managed:aws:anti-ddos:challengeable-request",
            "Scope": "LABEL"
          }
        }
      ]
    }
  }
}
```

## Kết luận

Bằng cách kết hợp các biện pháp bảo vệ tích hợp của Anti-DDoS AMR với logic tùy chỉnh, bạn có thể điều chỉnh các biện pháp phòng thủ để phù hợp với hồ sơ rủi ro, mô hình lưu lượng và khả năng mở rộng ứng dụng độc đáo của mình. Các ví dụ trong bài viết này minh họa cách bạn có thể tinh chỉnh độ nhạy, thực thi các biện pháp giảm thiểu mạnh mẽ hơn trong các điều kiện cụ thể và thậm chí xây dựng các biện pháp phòng thủ thích ứng phản ứng động với khả năng của hệ thống.

Bạn có thể sử dụng hệ thống gắn nhãn động trong AWS WAF để triển khai tùy chỉnh một cách chi tiết. Bạn cũng có thể sử dụng nhãn AWS WAF để loại trừ việc ghi log tốn kém của lưu lượng tấn công DDoS.

**Link bài viết gốc** : https://aws.amazon.com/blogs/security/how-to-customize-your-response-to-layer-7-ddos-attacks-using-aws-waf-anti-ddos-amr/
