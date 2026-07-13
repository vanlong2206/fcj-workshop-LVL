---
title: "Khó khăn và hướng phát triển"
date: 2026-07-13
weight: 12
chapter: false
pre: " <b> 5.12. </b> "
---
#### 5.12.1 Khó khăn

* Thiết kế kiến trúc Serverless phù hợp giữa các dịch vụ AWS (API Gateway, Lambda, SQS, Aurora, Backup...) làm tăng độ phức tạp của kiến trúc hệ thống.
* Quản lý quyền truy cập giữa các dịch vụ AWS (IAM Role, IAM Policy) khá phức tạp và dễ phát sinh lỗi `AccessDenied`.
* Phải bảo tính nhất quán dữ liệu khi nhiều yêu cầu cập nhật đồng thời. Có nguy cơ xảy ra xung đột dữ liệu, ghi đè giao dịch hoặc mất tính chính xác của dữ liệu trong hệ thống game.
* Theo dõi và xử lý lỗi trong môi trường Serverless gặp nhiều khó khăn do không có máy chủ cố định.

#### 5.12.2 Hướng phát triển

* Mở rộng hệ thống theo kiến trúc  **Event-Driven Architecture** , sử dụng EventBridge để kết nối các microservices thay vì giao tiếp trực tiếp.
* Triển khai **CI/CD** bằng GitHub Actions kết hợp Serverless Framework để tự động kiểm thử và triển khai hạ tầng AWS.
* Bổ sung **AWS WAF** và **Amazon Cognito** nhằm tăng cường bảo mật cho API Gateway và xác thực người dùng.
* Mở rộng khả năng giám sát bằng **AWS X-Ray** và xây dựng Dashboard trên CloudWatch để theo dõi hiệu năng toàn hệ thống.
* Thiết lập  **Cross-Region Backup** , **Cross-Account Backup** và **Backup Vault Lock** nhằm nâng cao khả năng khôi phục sau thảm họa (Disaster Recovery).
* Tối ưu chi phí bằng cách áp dụng chính sách Lifecycle cho Backup, tối ưu thời gian thực thi Lambda và sử dụng các gói dịch vụ phù hợp với lưu lượng truy cập.
* Mở rộng kiến trúc sang nhiều môi trường ( **Development** ,  **Staging** ,  **Production** ) với cấu hình và tài nguyên độc lập, giúp thuận tiện cho kiểm thử và vận hành.
