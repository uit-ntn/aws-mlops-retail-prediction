---
title: "Giới thiệu AWS Infrastructure as Code MCP Server: Hỗ trợ CDK và CloudFormation được hỗ trợ bởi AI"
weight: 7
chapter: false
pre: " <b> 3.7. </b> "
---

_Tác giả: Idriss Laouali Abdou — 28 NOV 2025_

Tinh gọn quá trình phát triển hạ tầng AWS của bạn với tìm kiếm tài liệu, xác thực và khắc phục sự cố được hỗ trợ bởi AI

## Giới thiệu

Hôm nay, chúng tôi rất vui mừng giới thiệu AWS Infrastructure-as-Code (IaC) MCP Server, một công cụ mới giúp thu hẹp khoảng cách giữa các trợ lý AI và quy trình phát triển hạ tầng AWS của bạn. Được xây dựng trên Model Context Protocol (MCP), máy chủ này cho phép các trợ lý AI như Kiro CLI, Claude hoặc Cursor giúp bạn tìm kiếm tài liệu AWS CloudFormation và Cloud Development Kit (CDK), xác thực template, khắc phục sự cố triển khai và tuân theo các best practices – đồng thời vẫn duy trì tính bảo mật của việc thực thi cục bộ (local execution).

Dù bạn đang viết AWS CloudFormation templates hay mã AWS Cloud Development Kit (CDK), IaC MCP Server đóng vai trò như một người bạn đồng hành thông minh, hiểu nhu cầu về hạ tầng của bạn và cung cấp hỗ trợ theo ngữ cảnh xuyên suốt vòng đời phát triển của bạn.

Model Context Protocol (MCP) là một tiêu chuẩn mở cho phép các trợ lý AI kết nối an toàn với các nguồn dữ liệu và công cụ bên ngoài. Hãy hình dung nó như một “bộ chuyển đổi” phổ quát cho phép các mô hình AI tương tác với các công cụ phát triển của bạn trong khi vẫn giữ các thao tác nhạy cảm ở cục bộ và dưới sự kiểm soát của bạn.

IaC MCP Server cung cấp chín công cụ chuyên biệt, được tổ chức thành hai nhóm:

### Công cụ tìm kiếm tài liệu từ xa

Các công cụ này kết nối đến AWS Knowledge MCP backend để truy xuất thông tin phù hợp, cập nhật:

- `search_cdk_documentation`  
  Tìm kiếm cơ sở kiến thức AWS CDK cho APIs, khái niệm và hướng dẫn triển khai.
- `search_cdk_samples_and_constructs`  
  Khám phá các AWS CDK constructs và patterns dựng sẵn từ AWS Construct Library.
- `search_cloudformation_documentation`  
  Truy vấn tài liệu CloudFormation cho các loại tài nguyên, thuộc tính và các intrinsic functions.
- `read_iac_documentation_page`  
  Truy xuất và đọc đầy đủ các trang tài liệu CloudFormation và CDK được trả về từ các tìm kiếm hoặc từ các URL được cung cấp.

### Công cụ xác thực và khắc phục sự cố cục bộ

Các công cụ này chạy hoàn toàn trên máy của bạn

- `cdk_best_practices`  
  Truy cập một bộ sưu tập được tuyển chọn các best practices và nguyên tắc thiết kế của AWS CDK.
- `validate_cloudformation_template`  
  Thực hiện kiểm tra cú pháp và schema bằng cfn-lint để phát hiện lỗi trước khi triển khai.
- `check_cloudformation_template_compliance`  
  Chạy kiểm tra bảo mật và tuân thủ đối với templates của bạn bằng AWS Guard rules và cfn-guard.
- `troubleshoot_cloudformation_deployment`  
  Phân tích các lỗi triển khai CloudFormation stack với phân tích sự kiện CloudTrail được tích hợp. Công cụ này sẽ sử dụng thông tin xác thực AWS của bạn để phân tích trạng thái stack.
- `get_cloudformation_pre_deploy_validation_instructions`  
  Trả về hướng dẫn cho tính năng xác thực trước triển khai (pre-deployment validation) của CloudFormation, tính năng này xác thực templates trong quá trình tạo change set.

## Các trường hợp sử dụng chính

### 1. Trợ lý tài liệu thông minh

Thay vì tự tay tìm kiếm qua tài liệu, hãy hỏi trợ lý AI của bạn bằng ngôn ngữ tự nhiên:

“Làm thế nào để tạo một S3 bucket với bật mã hóa (encryption) trong CDK?”

Máy chủ sẽ tìm kiếm CDK best practice và samples, trả về các ví dụ mã và giải thích liên quan.

### 2. Xác thực template chủ động

Trước khi triển khai các thay đổi hạ tầng:

User: “Validate my CloudFormation template and check for security issues”

AI Agent: [Uses validate_cloudformation_template and check_cloudformation_template_compliance]

“Found 2 issues: Missing encryption on EBS volumes,

and S3 bucket lacks public access block configuration”

### 3. Khắc phục sự cố triển khai nhanh

Khi một stack triển khai thất bại:

User: “My stack ‘stack_03’ in us-east-1 failed to deploy. What happened?”

AI Agent: [Uses troubleshoot_stack_deployment with CloudTrail integration]

“The deployment failed due to insufficient IAM permissions.

CloudTrail shows AccessDenied for ec2:CreateVpc.

You need to add VPC permissions to your deployment role.”

### 4. Học tập và khám phá

Mới làm quen với AWS CDK? Máy chủ giúp bạn khám phá constructs và patterns:

User: “Show me how to build a serverless API”

AI Agent: [Searches CDK constructs and samples]

“Here are three approaches using API Gateway + Lambda…”

## Kiến trúc và bảo mật

### Thiết kế bảo mật

- **Thực thi cục bộ (Local Execution):** MCP server chạy hoàn toàn trên máy cục bộ của bạn bằng uv (trình quản lý gói Python nhanh). Không có mã hoặc templates nào được gửi đến các dịch vụ bên ngoài, ngoại trừ các truy vấn tìm kiếm tài liệu.
- **Thông tin xác thực AWS (AWS Credentials):** Máy chủ sử dụng thông tin xác thực AWS hiện có của bạn (từ ~/.aws/credentials, biến môi trường, hoặc IAM roles) để truy cập các CloudFormation và CloudTrail APIs. Điều này tuân theo cùng mô hình bảo mật như AWS CLI.
- **Giao tiếp stdio:** Máy chủ giao tiếp với các trợ lý AI qua standard input/output (stdio), không mở bất kỳ network ports nào.
- **Quyền tối thiểu (Minimal Permissions):** Để có đầy đủ chức năng, máy chủ cần quyền chỉ đọc (read-only) đối với CloudFormation stacks và CloudTrail events—không cần quyền ghi (write) cho các quy trình xác thực và khắc phục sự cố.

## Bắt đầu

### Điều kiện tiên quyết

- Python 3.10 hoặc mới hơn
- uv package manager
- AWS credentials được cấu hình cục bộ
- MCP-compatible AI client (ví dụ: Kiro CLI, Claude Desktop)

### Cấu hình

Cấu hình MCP server trong cấu hình MCP client của bạn. Với bài blog này, chúng tôi sẽ tập trung vào Kiro CLI. Chỉnh sửa .kiro/settings/mcp.json):

```json
{
  "mcpServers": {
    "awslabs.aws-iac-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.aws-iac-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "your-named-profile",
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

JSON

## Các cân nhắc về bảo mật

Privacy Notice: MCP server này thực thi các lệnh gọi AWS API bằng thông tin xác thực của bạn và chia sẻ dữ liệu phản hồi với nhà cung cấp mô hình AI bên thứ ba của bạn (ví dụ: Amazon Q, Claude Desktop, Cursor, VS Code). Người dùng chịu trách nhiệm hiểu các thực hành xử lý dữ liệu của nhà cung cấp AI của bạn và đảm bảo tuân thủ các yêu cầu bảo mật và quyền riêng tư của tổ chức bạn khi sử dụng công cụ này với các tài nguyên AWS.

## Quyền IAM

MCP server yêu cầu các quyền AWS sau:

**Cho xác thực template và tuân thủ:**

- Không yêu cầu quyền AWS (chỉ xác thực cục bộ)

**Cho khắc phục sự cố triển khai:**

- cloudformation:DescribeStacks
- cloudformation:DescribeStackEvents
- cloudformation:DescribeStackResources
- cloudtrail:LookupEvents (cho CloudTrail deep links)

Ví dụ IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents",
        "cloudformation:DescribeStackResources",
        "cloudtrail:LookupEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

JSON

## Ví dụ trường hợp sử dụng với Kiro CLI

IMPORTANT: Đảm bảo bạn đã đáp ứng tất cả các điều kiện tiên quyết trước khi thử các lệnh này.

1. Khi tệp mcp.json đã được thiết lập đúng, hãy thử chạy một prompt mẫu. Trong terminal của bạn, chạy kiro-cli chat để bắt đầu sử dụng Kiro-cli trong CLI.

![B7img](/images/blog7/b701.png "B7img")

Scenarios:

“What are the CDK best practices for Lambda functions?”
![B7img](/images/blog7/b702.png "B7img")

“Search for CDK samples that use DynamoDB with Lambda”
![B7img](/images/blog7/b703.png "B7img")

“Validate my CloudFormation template at ./template.yaml”
![B7img](/images/blog7/b704.png "B7img")

“Check if my template complies with security best practices”
![B7img](/images/blog7/b705.png "B7img")

## Thực tiễn tốt nhất

- Bắt đầu với tìm kiếm tài liệu: Trước khi viết mã, hãy tìm kiếm các constructs và patterns sẵn có
- Xác thực sớm và thường xuyên: Chạy các công cụ xác thực trước khi cố gắng triển khai
- Kiểm tra tuân thủ: Dùng check_template_compliance để phát hiện các vấn đề bảo mật trong quá trình phát triển
- Tận dụng CloudTrail: Khi khắc phục sự cố, tích hợp CloudTrail cung cấp ngữ cảnh thất bại chi tiết
- Tuân theo CDK Best Practices: Dùng công cụ cdk_best_practices để bám theo các khuyến nghị của AWS

## Tiếp theo là gì?

IAC MCP Server đại diện cho một mô hình mới trong quy trình làm việc AI agentic cho phát triển hạ tầng – nơi các trợ lý AI hiểu công cụ của bạn, giúp bạn điều hướng tài liệu phức tạp và cung cấp hỗ trợ thông minh xuyên suốt vòng đời phát triển.
