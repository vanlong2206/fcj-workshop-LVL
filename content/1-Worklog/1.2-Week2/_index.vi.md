---
title: "Worklog Tuần 2"
date: 2026-05-11
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2:

* Tìm hiểu các dịch vụ AWS và các module miễn phí có thể ứng dụng trực tiếp vào đồ án.
* Triển khai dự án thực tế trên máy ảo EC2 và cơ sở dữ liệu RDS.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --- | --- | --- | --- |
| 2  | - Tìm hiểu các dịch vụ AWS và module miễn phí có thể ứng dụng vào đồ án: <br>&emsp; + EC2 (t3.micro) <br>&emsp; + RDS (db.t4g.micro) <br>&emsp; + Amazon Gamelift <br>&emsp; + ElastiCache (cache.t3.micro) | 11/05/2026 | 11/05/2026 |
| 3 - 5 | - Khởi tạo tài nguyên cơ bản: <br>&emsp; + Tạo EC2 (t3.micro) với hệ điều hành Ubuntu Linux. <br>&emsp; + Tạo RDS (db.t4g.micro) chạy MySQL. <br>&emsp; + Thiết lập IP tĩnh cho server bằng Elastic IP. | 12/05/2026 | 14/05/2026 |
| 6 - 7 | - Cấu hình và liên kết tài nguyên: <br>&emsp; + Liên kết db RDS đến EC2. <br>&emsp; + Thiết lập inbound rule (launch-wizard-x). <br>&emsp; + Tận dụng ổ cứng dư thừa để cắt làm RAM ảo. | 15/05/2026 | 16/05/2026 |
| CN | - Deploy dự án: <br>&emsp; + Sử dụng pm2 để deploy ngay trên máy ảo EC2. <br>&emsp; + Mở cổng để máy khác có thể truy cập vào public IP. | 17/05/2026 | 17/05/2026 |

### Kết quả đạt được tuần 2:

* Nắm rõ và xác định được các dịch vụ/module AWS miễn phí phù hợp cho đồ án: EC2, RDS, Amazon Gamelift, ElastiCache.
* Triển khai thành công máy ảo EC2 (Ubuntu Linux, t3.micro) và cơ sở dữ liệu RDS (MySQL, db.t4g.micro).
* Đã thiết lập thành công IP tĩnh cho server thông qua Elastic IP.
* Hoàn tất liên kết cơ sở dữ liệu RDS vào EC2 và cấu hình Inbound rule (launch-wizard-x) để cho phép kết nối.
* Tối ưu hóa được máy ảo bằng cách tận dụng ổ cứng dư thừa làm RAM ảo.
* Đã sử dụng pm2 để deploy dự án trực tiếp trên EC2 và mở cổng thành công, cho phép truy cập từ public IP.