---
title: "Workshop"
date: 2025-08-30T00:00:00+00:00
weight: 5
draft: false
chapter: true
pre: "<b>5. </b>"
---

<h2 style="margin: 20px 0 30px 0; font-size: 2.5rem; color: #131314ff; text-align: center;">Nền tảng Dự đoán Bán lẻ AWS MLOps</h2>

<div style="display: flex; flex-wrap: wrap; gap: 10px; margin: 20px 0;">
    <div style="background: #3b82f6; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🏗️ Cơ sở hạ tầng</div>
    <div style="background: #10b981; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🤖 Huấn luyện ML</div>
    <div style="background: #8b5cf6; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🚀 Triển khai</div>
    <div style="background: #ef4444; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">📊 Giám sát</div>
    <div style="background: #06b6d4; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">🔄 CI/CD</div>
    <div style="background: #f59e0b; color: white; padding: 8px 16px; border-radius: 20px; font-size: 14px; font-weight: 500;">💰 Tối ưu hóa chi phí</div>
</div>

<p style="text-align: center; font-style: italic; color: #64748b; font-size: 1.1rem; margin-bottom: 40px;">Pipeline MLOps từ đầu đến cuối cho Dự đoán Bán lẻ với Infrastructure as Code và Triển khai Mô hình</p>

<!-- Authors Section -->
<div class="authors-grid" style="display: grid; grid-template-columns: 1fr 1fr; gap: 30px; max-width: 800px; margin: 40px auto; padding: 0 20px;">
  <div style="background: white; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);">
    <h3 style="margin: 0 0 12px 0; color: #1e293b; font-size: 1.2rem;">👨‍💻 Tác giả 1</h3>
    <p style="margin: 8px 0; color: #475569; font-weight: 600;">Nguyễn Thanh Nhân</p>
    <p style="margin: 4px 0; color: #64748b; font-size: 14px;">Kỹ sư Đám mây</p>
    <div style="margin-top: 12px;">
      <p style="margin: 4px 0; font-size: 14px; display: flex; align-items: center; gap: 8px;">
        <span style="color: #3b82f6;">📧</span> 
        <span style="color: #64748b;">nhan.nguyen@example.com</span>
      </p>
      <p style="margin: 4px 0; font-size: 14px; display: flex; align-items: center; gap: 8px;">
        <span style="color: #059669;">🌐</span> 
        <span style="color: #64748b;">github.com/nhan-nguyen</span>
      </p>
    </div>
  </div>

  <div style="background: white; border: 1px solid #e2e8f0; border-radius: 12px; padding: 20px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);">
    <h3 style="margin: 0 0 12px 0; color: #1e293b; font-size: 1.2rem;">👨‍💻 Tác giả 2</h3>
    <p style="margin: 8px 0; color: #475569; font-weight: 600;">Người đóng góp</p>
    <p style="margin: 4px 0; color: #64748b; font-size: 14px;">Chuyên gia MLOps</p>
    <div style="margin-top: 12px;">
      <p style="margin: 4px 0; font-size: 14px; display: flex; align-items: center; gap: 8px;">
        <span style="color: #3b82f6;">📧</span> 
        <span style="color: #64748b;">contributor@example.com</span>
      </p>
      <p style="margin: 4px 0; font-size: 14px; display: flex; align-items: center; gap: 8px;">
        <span style="color: #059669;">🌐</span> 
        <span style="color: #64748b;">github.com/contributor</span>
      </p>
    </div>
  </div>
</div>

---

## 🎯 Tổng quan Workshop

Workshop này sẽ hướng dẫn bạn xây dựng một **nền tảng MLOps hoàn chỉnh trên AWS** để dự đoán xu hướng bán lẻ. Bạn sẽ học cách:

### 🏗️ **Cơ sở hạ tầng (Infrastructure)**
- Thiết lập VPC và networking cho môi trường MLOps
- Cấu hình IAM roles và security policies
- Triển khai container registry với ECR

### 🤖 **Huấn luyện & Quản lý Mô hình**  
- Xây dựng pipeline dữ liệu với S3 và data processing
- Huấn luyện mô hình ML với Amazon SageMaker
- Versioning và quản lý mô hình với Model Registry

### 🚀 **Triển khai & Scaling**
- Deploy mô hình trên Amazon EKS (Kubernetes)
- Cấu hình Load Balancer cho high availability
- Auto-scaling dựa trên traffic patterns

### 📊 **Giám sát & Monitoring**
- Thiết lập CloudWatch metrics và dashboards
- Monitoring model performance và data drift
- Alerting và notification system

### 🔄 **CI/CD & Automation**
- Jenkins pipeline cho automated deployment
- Infrastructure as Code với Terraform/CloudFormation
- Automated testing và validation

### 💰 **Tối ưu hóa Chi phí**
- Cost monitoring và optimization strategies
- Resource cleanup và cost analysis
- Best practices cho cost-effective MLOps

---

## 📋 **Yêu cầu trước khi bắt đầu**

- **AWS Account** với appropriate permissions
- **Basic knowledge** về machine learning concepts
- **Familiar** với Docker và Kubernetes
- **Understanding** của CI/CD principles

---

## 🚀 **Kết quả đạt được**

Sau khi hoàn thành workshop, bạn sẽ có:

✅ **Production-ready MLOps pipeline** trên AWS  
✅ **Automated model training** và deployment process  
✅ **Monitoring và alerting** system  
✅ **Cost-optimized** infrastructure setup  
✅ **Best practices** cho enterprise MLOps  

---

{{% notice info %}}
🎓 **Học tập hiệu quả:** Workshop này được thiết kế theo phương pháp hands-on learning. Mỗi task đều có examples thực tế và code samples để bạn có thể áp dụng ngay vào dự án của mình.
{{% /notice %}}

{{% notice tip %}}
💡 **Tip:** Hãy làm theo từng bước một cách tuần tự. Mỗi task đều build upon kiến thức từ task trước đó.
{{% /notice %}}