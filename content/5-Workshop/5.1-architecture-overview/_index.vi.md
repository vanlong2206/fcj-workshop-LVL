---
title : "Tổng quan về kiến trúc hệ thống"
date : 2026-06-18 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Giới thiệu về Kiến trúc Game Backend Server

#### 5.1.1 Tổng quan về mô hình triển khai
Edge & Security Layer:
   * AWS Route 53: Phân giải và điều phối tên miền hệ thống.
   * Amazon CloudFront: Mạng lưới phân phối nội dung (CDN) giúp tối ưu hóa tốc độ đường truyền và giảm độ trễ cho Client.
   * AWS WAF: Gắn trực tiếp vào CloudFront để lọc các request độc hại, ngăn chặn DDoS, Botnet và các cuộc tấn công từ vòng ngoài.

API & Sync Compute Layer:
   * Amazon API Gateway: Tiếp nhận và định tuyến các RESTful endpoint đến trực tiếp các hàm xử lý logic.
   * Compute Sync: Nhóm hàm xử lý logic thời gian thực và phản hồi trực tiếp cho Client, bao gồm các module cốt lõi: Xác thực, Kho đồ/Rương độc lập, Giao dịch...

Async Processing Layer:
   * Compute Async: Nhóm hàm xử lý các tác vụ nền tốn tài nguyên như Tiến trình thế giới và Hệ thống Reward. Sau khi tính toán, dữ liệu sẽ được đẩy vào hàng đợi xử lý.
   * Amazon SQS FIFO: Hàng đợi đảm bảo thứ tự của các sự kiện ghi dữ liệu quan trọng, tránh xung đột trạng thái dữ liệu.
   * Amazon SQS DLQ: Bắt và cô lập các message lỗi bị thất bại để xử lý sau.
   * Amazon CloudWatch: Giám sát trạng thái của DLQ để gửi cảnh báo lập tức ngay khi xuất hiện Exception hoặc Crash.

Database Layer:
   * Amazon Aurora DB: Hệ quản trị cơ sở dữ liệu chính với khả năng tự động mở rộng (Auto-scaling).
   * AWS Backup & Amazon S3: Cấu hình quy trình tự động snapshot và sao lưu dữ liệu an toàn định kỳ.

Automation & Maintenance Layer:
   * Amazon EventBridge: Kích hoạt theo lịch trình hoặc sự kiện cho các hàm Lambda đặc thù:
     * Hạ tầng API: Gọi các hàm `start_maintenance` và `stop_maintenance` để cập nhật trạng thái hệ thống `stageVariables` của API Gateway thông qua AWS SDK.
     * Bảo trì CSDL: Định kỳ kích hoạt hàm `vacuum & analyze` để tối ưu hóa hiệu năng DB và hàm `reset_daily` để làm mới tiến trình thế giới hàng ngày.

![overview](/images/diagram.png)
<div align="center"><i>Hình 5.1.1: Sơ đồ kiến trúc tổng thể.</i></div>