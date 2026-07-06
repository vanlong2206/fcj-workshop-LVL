---
title : "Amazon SQS FIFO"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---
#### Amazon SQS FIFO

#### 5.8.1 Kiến trúc hệ thống

![1783096083070](image/_index.vi/1783096083070.png)

<div align="center"><i>Hình 5.8.1:Kiến trúc hệ thống.</i></div>

ví dụ với luồng xử lí POST /Economy/earn :

* Client gửi `POST /Economy/earn`.
* API Gateway chuyển request đến Producer Lambda.
* Producer Lambda tạo message và gửi vào Amazon SQS FIFO, không ghi trực tiếp vào cơ sở dữ liệu.
* Producer Lambda phản hồi ngay cho Client với trạng thái `queued`.
* Amazon SQS kích hoạt Consumer Lambda khi có message.
* Consumer Lambda mở transaction và sử dụng `SELECT ... FOR UPDATE` để khóa bản ghi, tránh xung đột dữ liệu.
* Consumer Lambda xử lý nghiệp vụ và ghi dữ liệu vào Amazon Aurora/RDS.
* Sau khi xử lý thành công, message được xóa khỏi SQS; nếu thất bại sẽ được retry hoặc chuyển sang DLQ.
