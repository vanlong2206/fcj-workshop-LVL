---
title: "Kiểm tra kết quả và thực nghiệm"
date: 2026-07-13
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---
## 5.10.1 Tiến hành demo game sau khi đã hoàn thành hệ thống lưu trữ và xử lý tiến trình Game Backend

<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
  <iframe src="https://www.youtube.com/embed/bxgfVyOcEXw" 
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" 
          allowfullscreen 
          title="Kiểm tra kết quả và thực nghiệm">
  </iframe>
</div>

## 5.10.2 Kiểm tra hệ thống SQS DLQ và Bảo Trì cho game 2D

<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
  <iframe src="https://www.youtube.com/embed/Vmwd8U29pL0" 
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" 
          allowfullscreen 
          title="Kiểm tra kết quả và thực nghiệm">
  </iframe>
</div>

##### * Bối cảnh:

Test với **50 users ảo** chạy song song trong **5 phút**, mỗi user thực hiện: Register → Login → [ Earn(100 coin) → Spend(random) ] loop

**30% spend** dùng amount=999.999.999 — cố tình lớn hơn số dư để gây fail dẫn đến dlq
