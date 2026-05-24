---
title: "Worklog Tuần 3"
date: 2026-05-18
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3:

* Thiết lập quản lý chi phí AWS và cấu hình bảo mật, phân quyền truy cập cơ bản.
* Tìm hiểu và thực hành cấu hình mạng (VPC), DNS (Route53) và các công cụ quản lý qua CLI.
* Ứng dụng tự động hóa trong việc quản lý tài nguyên (tự động bật/tắt EC2, RDS) và tìm hiểu dịch vụ AI mới của AWS.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu / Ghi chú |
| --- | --- | --- | --- | --- |
| 2 | - Thiết lập quản lý tài khoản và chi phí bằng AWS Budgets. <br> - Cài đặt 2 cảnh báo miễn phí: Actual 80% và Forecasted 100% trong ngân sách 10 USD. | 18/05/2026 | 18/05/2026 |
| 3 | - Thiết lập Amazon VPC. <br> - Tạo và quản lý IAM User/Role để lấy AWS Access Key ID và AWS Secret Access Key. | 19/05/2026 | 19/05/2026 |
| 4 - 5 | - Tìm hiểu và cấu hình AWS CLI trên môi trường Ubuntu EC2 thông qua IAM User. <br> - Tìm hiểu và thử triển khai dịch vụ Route53. | 20/05/2026 | 21/05/2026 | Quên tắt Route53 và bị trừ 20 đô |
| 6 | - Tìm hiểu về dịch vụ AI của AWS: AWS Bedrock. | 22/05/2026 | 22/05/2026 |
| 7 - CN | - Thiết lập tự động bật/tắt EC2 và RDS bằng cách sử dụng kết hợp Lambda Function và Amazon EventBridge. | 23/05/2026 | 24/05/2026 |

### Kết quả đạt được tuần 3:

* Đã cấu hình thành công công cụ cảnh báo chi phí AWS Budgets (ngưỡng 80% và 100% cho ngân sách 10$).
* Thiết lập xong mạng Amazon VPC và phân quyền IAM, lấy thành công Access Key ID và Secret Access Key.
* Cấu hình và sử dụng được AWS CLI trên môi trường máy ảo Ubuntu EC2.
* Đã có kinh nghiệm thực tế triển khai Route53 (và rút ra bài học quản lý tài nguyên khi bị trừ 20$).
* Nắm được tổng quan về dịch vụ AI AWS Bedrock.
* Tối ưu hoá được chi phí bằng cách tự động hoá việc bật/tắt EC2 và RDS thông qua Lambda Function và Amazon EventBridge.