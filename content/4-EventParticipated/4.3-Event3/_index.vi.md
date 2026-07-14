---
title: "Event 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.3. </b> "
---
# Bài thu hoạch sự kiện FCAJ Community Day

| Thông tin   | Chi tiết                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------ |
| Ngày        | 27/6/2026                                                                                  |
| Địa điểm | Tầng 26 Tòa nhà Bitexco, 02 Hải Triều, Phường Bến Nghé, Quận 1, TP Hồ Chí Minh |
| Vai trò     | Người tham gia                                                                           |


## 4.3.1 Mục Đích Của Sự Kiện

Buổi chia sẻ tập trung vào việc giới thiệu các giải pháp ứng dụng AI trong môi trường doanh nghiệp, đặc biệt là trong lĩnh vực vận hành hạ tầng, tối ưu hệ thống đám mây và đảm bảo an toàn dữ liệu. Nội dung cũng đề cập đến các mô hình triển khai AI thực tế trên nền tảng AWS nhằm nâng cao hiệu quả vận hành và tự động hóa quy trình.

## 4.3.2 Danh Sách Diễn Giả

* **Mr. Steve Trần (Cloud Thinker):** Chuyên đề  *Vận hành Hạ tầng Đám mây và Kỷ nguyên Agentic Platform* .
* **Mr. Hiếu Nghị & Mr. Kiệt & Mr. Trung:** Chuyên đề  *Xây dựng Trợ lý AI Giọng nói cho Doanh nghiệp* .
* **Ms. Bảo & Mr. Nguyên Nguyễn (Cloud Kinetics):** Chuyên đề  *Triển khai DevOps AI Agent trên AWS* .
* **Mr. Trường & Ms. Minh Anh (Noventic):** Chuyên đề  *Tự động hóa Quy trình Nhân sự với Amazon Q* .
* **Mr. Toàn Nguyễn & Mr. Hiếu Nghị (Renova Cloud):** Chuyên đề  *Mô hình Bảo mật Private cho Amazon Q và MCP Server* .

## 4.3.3 Nội Dung Nổi Bật

### Vận hành hạ tầng đám mây trong kỷ nguyên Agentic Platform

Phần trình bày giới thiệu mô hình **Agentic Platform** với mục tiêu hỗ trợ vận hành hệ thống thông minh hơn thông qua AI. Nền tảng này tập trung giải quyết bốn bài toán phổ biến gồm điều tra sự cố, đánh giá mã nguồn, tối ưu chi phí hạ tầng (FinOps) và tăng cường bảo mật.

Diễn giả cũng so sánh hai mô hình triển khai AI Agent:

* **Single Agent** có thể xử lý hiệu quả phần lớn tác vụ nếu được thiết kế đầy đủ ngữ cảnh và quy trình.
* **Multi-Agent** chia nhỏ công việc cho nhiều Agent chuyên biệt nhằm tối ưu khả năng xử lý, quản lý context và phân quyền rõ ràng hơn.

Cách tiếp cận này giúp đội ngũ kỹ sư rút ngắn thời gian phân tích và xử lý sự cố trong môi trường vận hành thực tế.

### Xây dựng trợ lý AI giọng nói cho doanh nghiệp

Nội dung tiếp theo trình bày kiến trúc xây dựng trợ lý AI sử dụng ba thành phần chính:

* **Speech-to-Text (STT)** chuyển giọng nói thành văn bản.
* **Large Language Model (LLM)** xử lý nội dung và ngữ cảnh.
* **Text-to-Speech (TTS)** tạo phản hồi bằng giọng nói.

Việc sử dụng văn bản làm lớp trung gian giúp doanh nghiệp dễ dàng áp dụng Guardrails để kiểm soát nội dung cũng như tích hợp khả năng gọi API theo thời gian thực.

Ở môi trường Production, hệ thống cần bổ sung các cơ chế như Streaming nhằm giảm độ trễ, hỗ trợ nhiều vùng giọng nói, nhận diện giới tính, xử lý tình huống người dùng ngắt lời và chuyển tiếp cuộc gọi đến nhân viên khi cần thiết.

### DevOps AI Agent trên AWS

Chuyên đề giới thiệu giải pháp sử dụng AI Agent để hỗ trợ quy trình DevOps, đặc biệt trong việc tự động điều tra sự cố sau khi hệ thống phát sinh cảnh báo từ CloudWatch.

Giải pháp được xây dựng dựa trên sáu thành phần chính:

* Context Learning
* Control
* Integration
* Collaboration
* Convenient
* Cost Effective

Quy trình xử lý bao gồm tiếp nhận cảnh báo, phân loại lỗi, phân tích nguyên nhân gốc (Root Cause Analysis), đề xuất hướng khắc phục trước mắt và đưa ra các cải tiến dài hạn. AI Agent chỉ đóng vai trò hỗ trợ và đề xuất, không trực tiếp thực hiện thay đổi trên hệ thống.

### Ứng dụng Amazon Q trong quy trình nhân sự

Buổi chia sẻ giới thiệu Amazon Q như một trợ lý AI nội bộ phục vụ doanh nghiệp với khả năng đảm bảo tính bảo mật dữ liệu cao hơn so với các nền tảng AI công cộng.

Amazon Q có thể tích hợp với nhiều hệ thống như Google Workspace, Microsoft Workspace, Jira, Salesforce và GitHub thông qua MCP.

Trong quy trình tuyển dụng, hệ thống có thể tự động:

* Phân tích tài liệu hướng dẫn tuyển dụng.
* Trích xuất nội dung từ CV, bao gồm cả OCR.
* Đánh giá và phân loại ứng viên.
* Đề xuất mức lương cũng như xuất báo cáo thông qua các ứng dụng no-code.

### Mô hình bảo mật Private cho Amazon Q và MCP Server

Chuyên đề cuối cùng tập trung vào kiến trúc triển khai Amazon Q trong môi trường Private nhằm đảm bảo an toàn thông tin.

Giải pháp được xây dựng theo nguyên tắc  **Zero Trust** , giúp hạn chế các nguy cơ như tấn công DDoS hoặc Man-in-the-Middle.

Kiến trúc được đề xuất bao gồm:

* Đặt MCP Server trong Private Subnet của VPC.
* Amazon Q truy cập thông qua Interface Endpoint và VPC Connection.
* Mã hóa lưu lượng bằng TLS thông qua Internal ALB kết hợp ACM và Amazon Cognito.
* Sử dụng Route 53 Resolver để che giấu DNS nội bộ khỏi Internet.

## 4.3.4 Ứng Dụng Vào Công Việc

* Xem AI là công cụ hỗ trợ nhằm nâng cao hiệu quả công việc thay vì thay thế hoàn toàn vai trò của con người.
* Khi triển khai AI trong doanh nghiệp cần ưu tiên Data Governance, Guardrails và các mô hình bảo mật Private trước khi đưa hệ thống vào môi trường Production.
* Vận dụng tư duy Agentic Platform và Multi-Agent để hỗ trợ giám sát hệ thống, điều tra sự cố cũng như tối ưu chi phí vận hành trong các dự án sử dụng AWS.

![](blob:https://www.facebook.com/ebfcc44c-603b-4bb2-a1ce-9575e430ad4c)![1784013303527](image/_index.vi/1784013303527.png)
