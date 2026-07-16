---
title: "Worklog Tuần 11"
date: 2026-06-29
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

## Mục tiêu tuần 11:

* Tìm hiểu và nghiên cứu các công nghệ mới trên nền tảng Amazon Web Services.
* Đọc, tổng hợp các bài Blog kỹ thuật về Amazon EKS Pod Identity và các dịch vụ AWS.
* Phân tích, thiết kế giải pháp AWS Serverless cho dự án Game Backend API.
* Hoàn thiện kiến trúc, lộ trình triển khai, dự toán chi phí và đánh giá rủi ro của hệ thống.

## Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --- | --- | --- | --- |
| 2 | - Đọc và nghiên cứu các bài Blog về Amazon EKS Pod Identity <br> - Tìm hiểu Session Policies và nguyên tắc Least Privilege | 29/06/2026 | 29/06/2026 | AWS Blog |
| 3 | - Khảo sát hệ thống Game Backend API hiện tại <br> - Phân tích kiến trúc Monolith và các vấn đề tồn tại | 30/06/2026 | 30/06/2026 | Tài liệu dự án |
| 4 | - Thiết kế kiến trúc AWS Serverless <br> - Xây dựng sơ đồ kiến trúc và luồng xử lý hệ thống | 01/07/2026 | 01/07/2026 | AWS Architecture Center |
| 5 | - Xây dựng phương án triển khai kỹ thuật <br> - Lập lộ trình triển khai, dự toán chi phí và đánh giá rủi ro | 02/07/2026 | 03/07/2026 | AWS Documentation |
| 6 | - Hoàn thiện tài liệu giải pháp <br> - Rà soát kiến trúc và chuẩn bị báo cáo | 04/07/2026 | 05/07/2026 | Tài liệu dự án |

## Kết quả đạt được tuần 11:

* Nghiên cứu và tổng hợp các bài Blog kỹ thuật về Amazon EKS Pod Identity, Session Policies và các cơ chế phân quyền trên AWS.

* Phân tích hiện trạng hệ thống Game Backend API và xác định các hạn chế của kiến trúc Monolith về chi phí, khả năng mở rộng, triển khai và xử lý bất đồng bộ.

* Hoàn thành thiết kế giải pháp AWS Serverless cho hệ thống Game Backend API với các thành phần:
  * Amazon API Gateway.
  * AWS Lambda.
  * Amazon SQS FIFO.
  * Amazon Aurora PostgreSQL Serverless v2.
  * Amazon EventBridge.
  * AWS Backup.
  * Amazon CloudWatch.

* Xây dựng kiến trúc xử lý theo từng domain, sử dụng Lambda, SQS và EventBridge để tăng khả năng mở rộng, giảm chi phí và nâng cao độ tin cậy của hệ thống.

* Hoàn thiện phương án triển khai kỹ thuật, bao gồm:
  * Kiến trúc tổng thể.
  * Lộ trình triển khai theo từng giai đoạn.
  * Ước tính chi phí vận hành.
  * Đánh giá rủi ro và các phương án giảm thiểu.

* Hoàn thiện tài liệu đề xuất giải pháp AWS Serverless, sẵn sàng phục vụ cho quá trình triển khai và phát triển hệ thống trong các giai đoạn tiếp theo.
