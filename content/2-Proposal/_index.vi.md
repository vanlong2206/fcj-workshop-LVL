---
title: "Bản đề xuất"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
## Hệ thống lưu trữ và xử lý tiến trình Game Backend

### 2.1 Tóm tắt

Dự án Xây dựng hệ thống lưu trữ và xử lý tiến trình Game Backend nhằm mục tiêu chuyển đổi toàn diện kiến trúc hệ thống từ Monolith truyền thống sang hệ sinh thái Microservices Serverless trên AWS. Bằng việc phân tách các logic nghiệp vụ thành 6 cụm Lambda độc lập, kết hợp xử lý bất đồng bộ qua SQS và lưu trữ linh hoạt với Aurora Serverless v2, giải pháp này giải quyết triệt để vấn đề lãng phí tài nguyên vào giờ thấp điểm và rủi ro sập toàn cục. Kiến trúc mới cam kết cắt giảm ước tính 40% chi phí hạ tầng hàng tháng, cho phép khả năng mở rộng tức thời khi lượng người chơi tăng đột biến, và đẩy nhanh tốc độ triển khai tính năng mới một cách an toàn.

### 2.2 Vấn đề

**Vấn đề về chi phí vận hành:**
Kiến trúc monolith hiện tại chạy trên server 24/7, ngay cả khi không có người chơi nào hoạt động. Điều này gây lãng phí tài nguyên tính toán đáng kể. Server phải được cấp phát theo peak load (giờ cao điểm) nhưng lại chạy không tải vào giờ thấp điểm, dẫn đến chi phí vận hành không tối ưu.

**Vấn đề về khả năng mở rộng:**
Khi lượng người chơi tăng đột biến (sự kiện game, quảng cáo, mùa lễ), monolith không thể scale từng phần riêng lẻ. Ví dụ, chỉ riêng tính năng Auth hay Economy bị quá tải nhưng buộc phải scale toàn bộ hệ thống. Điều này vừa tốn kém vừa không hiệu quả. Kiến trúc monolith cũng không hỗ trợ auto-scaling linh hoạt theo từng domain.

**Vấn đề về triển khai và bảo trì:**
Mỗi lần cập nhật tính năng hoặc sửa lỗi đều phải deploy lại toàn bộ ứng dụng. Một lỗi nhỏ ở module Auth có thể làm sập toàn bộ game. Quy trình CI/CD phức tạp hơn và thời gian deploy lâu hơn do phải build và restart toàn bộ hệ thống. Không thể rollback độc lập từng tính năng.

**Vấn đề về xử lý bất đồng bộ:**
Các tác vụ quan trọng như cập nhật currency (earn/spend), thay đổi inventory (add/remove item), redeem gift code, lưu save data hiện đang được xử lý đồng bộ trong request-response cycle. Nếu server gặp sự cố giữa chừng, dữ liệu có thể bị mất hoặc không nhất quán. Thiếu cơ chế retry và hàng đợi để đảm bảo xử lý thành công.

**Vấn đề về backup và phục hồi sự cố:**
Dữ liệu game — tài khoản, tiền tệ, inventory, save data — là tài sản quan trọng nhất. Nếu game server chưa có chiến lược backup tự động, chưa có kế hoạch phục hồi sự cố rõ ràng, thì khi database gặp sự cố (hỏng ổ cứng, lỗi replication, tấn công), dữ liệu người chơi có thể bị mất vĩnh viễn.

### 2.3 Kiến trúc giải pháp

![1783961736805](image/diagram.png)

**Tổng quan luồng xử lý:**

Tổng quan luồng xử lý (Vòng đời dữ liệu):

Kiến trúc hệ thống được thiết kế theo nguyên tắc tách biệt các tầng (Edge, Compute, Data, Management) với các luồng tương tác được đánh số chi tiết trên sơ đồ:

[1] -> [2] -> [3]: Luồng Tiếp nhận và Điều hướng (Edge Layer)

- [1] Gửi request: Client khởi tạo các API request giao tiếp với backend.

- [2] Lọc request xấu: Lưu lượng đi vào CloudFront và ngay lập tức bị giám sát bởi AWS WAF để chặn đứng các request dị dạng, botnet, hoặc các cuộc tấn công DDoS ở tầng Application.

- [3] Tăng tốc và Định tuyến: CloudFront tối ưu hóa đường truyền toàn cầu, chuyển tiếp lưu lượng an toàn về API Gateway. Tại đây, API Gateway áp dụng rate limiting và định tuyến chính xác request đến cụm xử lý Lambda tương ứng.

[4a] & [4b, 4c]: Luồng Xử lý Nghiệp vụ (Compute & Database Layer)

- [4a] Đọc/ghi đồng bộ (Sync): Đối với các tác vụ yêu cầu phản hồi tức thời (như đăng nhập, tra cứu túi đồ ở cụm auth, inventory, transaction, economy), Lambda xử lý trực tiếp và thực hiện thao tác Đọc/Ghi thẳng vào Aurora RDS.

- [4c] Đọc dữ liệu tức thời (SELECT/JOIN): Với các request truy vấn thông tin, Lambda có thể lấy dữ liệu từ database và trả kết quả về ngay lập tức cho client.

- [4b] Ghi/Chỉnh sửa dữ liệu (UPDATE/INSERT): Đối với các tác vụ có tính chất cập nhật, chỉnh sửa dữ liệu game, request bắt buộc phải được đẩy vào xếp hàng tại SQS FIFO. Các Lambda DB Writer sau đó sẽ xử lý tuần tự để ghi vào cơ sở dữ liệu, đảm bảo tính toàn vẹn và ngăn chặn nguy cơ xung đột dữ liệu khi có nhiều thao tác diễn ra cùng lúc.

[5] -> [6] -> [7] -> [8]: Luồng Bảo trì và Sao lưu định kỳ (Maintenance Flow)

- [5] Bắt đầu bảo trì: Theo lịch trình định kỳ, hệ thống sẽ tự động cập nhật biến isMaintenance trên API Gateway để chặn toàn bộ request từ bên ngoài vào. Việc này giúp hệ thống đóng băng và không phát sinh giao dịch mới.

- [6] Thực hiện bảo trì & Reset: Các tiến trình chạy nền bắt đầu kích hoạt reset daily (làm mới các hoạt động hàng ngày trong gameplay như hồi sinh quái vật, đặt lại các sự kiện...). Đồng thời, hệ thống chạy lệnh vacuum analyze dọn dẹp nhẹ nhàng giúp sắp xếp lại các khoảng trống trong cơ sở dữ liệu, đảm bảo việc truy xuất luôn nhanh và mượt mà.

- [7] Sao lưu dữ liệu: Ngay trong lúc hệ thống đang ở trạng thái đóng băng bảo trì, quá trình export và backup sẽ được tiến hành để sao lưu toàn bộ dữ liệu an toàn sang Amazon S3.

- [8] Kết thúc bảo trì: Sau khi mọi tác vụ hoàn tất, biến isMaintenance được gỡ bỏ trên API Gateway, hệ thống mở cửa trở lại để tiếp nhận request từ người chơi bình thường.

**Chi tiết các thành phần:**

API Gateway đóng vai trò cổng duy nhất nhận request từ client, chịu trách nhiệm xác thực, rate limiting, và routing đến đúng Lambda. 6 Lambda API được triển khai độc lập, mỗi Lambda là một Express app được wrap bởi `@vendia/serverless-express`, chạy trên Node.js 20, xử lý một domain cụ thể của game. Các Lambda này scale độc lập dựa trên số request đến từng domain.

5 SQS FIFO queue đảm bảo message được xử lý đúng thứ tự và không bị mất. Mỗi queue có DLQ riêng để thu gom message thất bại sau 3 lần retry, kèm CloudWatch alarm để cảnh báo kịp thời. SQS Consumer Lambda xử lý message theo batch (tối đa 10 message/batch) với cơ chế partial failure.

Aurora RDS Serverless PostgreSQL là database chính, tự động scale ACU theo tải, hỗ trợ IAM authentication cho bảo mật. TypeORM tự động tạo/cập nhật schema khi khởi động (synchronize: true). Có 18 entities chia làm 5 domain (User, Forum, GiftCode, Game, System).

EventBridge đóng vai trò scheduler với 4 rule: bật chế độ bảo trì, tắt chế độ bảo trì, vacuum analyze và reindex tất cả bảng, reset daily (reset stamina, dọn expired codes, logs, stale data...). Maintenance Lambda nhận event từ EventBridge và thực thi các tác vụ tương ứng.

AWS Backup thực hiện backup database tự động hàng ngày hoặc hàng tuần.

Code dùng chung được đóng gói qua npm workspace `@gameapi/shared` bao gồm: 18 TypeORM entities, 3 middlewares (auth, admin, maintenance), SQS producer với 8 static methods, utils (JwtHelper, PasswordHasher, TimeHelper, ItemGenerationHelper, logger), services (GiftCodeService, GameLogicValidator), và CloudWatch metrics helper.

### 2.4 Các quyết định đánh đổi kiến trúc
#### 2.4.1 Đánh đổi giữa bảo mật tuyệt đối và chi phí vận hành
Vấn đề: Để đặt Aurora RDS vào Private Subnet một cách chuẩn mực, các Lambda Functions sẽ mất quyền truy cập Internet. Để Lambda có thể giao tiếp với các dịch vụ AWS khác hoặc bên thứ ba, hệ thống bắt buộc phải trang bị thêm NAT Gateway hoặc hệ thống VPC Endpoints.

Quyết định: Khởi tạo Aurora RDS trong Public Subnet để Lambda có thể chạy trong môi trường có sẵn Internet Access (Default Subnet) mà không cần NAT Gateway.

Lợi ích thực tế: Cắt giảm ngay lập tức chi phí cố định cho NAT Gateway (khoảng ~$32/tháng + phí truyền tải dữ liệu) và VPC Endpoints. Đối với giai đoạn đầu của dự án khi lượng người chơi và ngân sách còn hạn hẹp, việc gánh thêm chi phí mạng này là không khả thi về mặt tài chính.

#### 2.4.2 Đánh đổi giữa serverless compute (AWS Lambda) và server truyền thống (EC2)
Vấn đề: Ứng dụng game có đặc thù lưu lượng biến động cực kỳ mạnh. Lượng người chơi thường tăng vọt vào buổi tối, cuối tuần hoặc khi có sự kiện, và giảm sâu vào nửa đêm. Nếu sử dụng EC2 truyền thống, chúng ta đối mặt với bài toán: cấp phát tài nguyên theo mức cao điểm sẽ gây lãng phí, nhưng nếu cấu hình Auto Scaling Group, tốc độ khởi tạo một máy chủ EC2 mới mất từ 2-5 phút, không thể bắt kịp những đợt bùng nổ traffic tức thời, dẫn đến rớt kết nối của người chơi.

Quyết định: Loại bỏ EC2, chuyển đổi toàn bộ logic xử lý sang AWS Lambda.

Lợi ích thực tế và Đánh đổi:

Đánh đổi: Chấp nhận mất quyền can thiệp vào hệ điều hành, giới hạn thời gian thực thi và phải đối mặt với độ trễ khởi tạo môi trường ở những request đầu tiên.

Lợi ích vượt trội: Mô hình tính tiền theo mili-giây triệt tiêu hoàn toàn chi phí chạy không tải. Khi lượng người chơi = 0, chi phí compute = 0. Quan trọng hơn, hệ thống đạt được khả năng mở rộng tức thời và mở rộng độc lập từng module. Gánh nặng quản trị hạ tầng được chuyển giao hoàn toàn cho AWS, giúp đội ngũ phát triển tập trung 100% vào nghiệp vụ game.

#### 2.4.3 Đánh đổi giữa Aurora Serverless và Amazon RDS truyền thống
Vấn đề: Cơ sở dữ liệu quan hệ luôn là nút thắt cổ chai khó giải quyết nhất khi scale hệ thống. Với RDS truyền thống, nhà phát triển buộc phải chọn trước kích thước máy chủ. Việc thay đổi cấu hình máy chủ khi tải tăng đột biến yêu cầu phải khởi động lại và mất vài phút để hoàn thành. Điều này là rủi ro chí mạng đối với trải nghiệm game liên tục.

Quyết định: Sử dụng Amazon Aurora Serverless v2 PostgreSQL thay vì RDS PostgreSQL/MySQL truyền thống.

Lợi ích thực tế và Đánh đổi:

Đánh đổi: Đơn giá cho 1 đơn vị dung lượng ACU của Aurora Serverless thực tế cao hơn so với tài nguyên tính toán tương đương trên RDS Provisioned. Hệ thống cũng đòi hỏi cấu hình logic quản lý kết nối khắt khe hơn do đặc thù kết nối từ hàng trăm Lambda instances.

Lợi ích vượt trội: Sự chênh lệch về đơn giá hoàn toàn được bù đắp bởi khả năng tự động mở rộng theo phương thẳng đứng tức thời từ 0.5 ACU lên tới hàng chục ACU dựa trên tải thực tế mà không hề gây ra downtime. Aurora tự động thu hẹp tài nguyên khi ít người chơi, giúp đường cong chi phí bám sát hoàn hảo với đường cong lưu lượng, tránh lãng phí tiền bạc vào việc duy trì một database cấu hình cao hoạt động 24/7 chỉ để đề phòng giờ cao điểm.

### 2.5 Triển khai kỹ thuật

**6 Lambda API** chia theo domain — Auth (đăng ký/đăng nhập/dashboard), Economy (balance/earn/spend), Inventory (CRUD đồ + storage), Transaction (shop + gift code), Progression (stats + nông trại), Loot-Reward (leaderboard + diễn đàn + save data + game data).

**5 SQS FIFO queue** — economy, inventory, giftcode, stats, save-data. Mỗi queue có DLQ riêng với CloudWatch alarm. Message được xử lý bất đồng bộ bởi các consumer Lambda tương ứng.

**Database** — Aurora RDS PostgreSQL với TypeORM (synchronize: true). 18 entities chia làm 5 domain. Hỗ trợ IAM authentication cho production, password cho local dev.

**Bảo mật** — JWT cho API, admin secret cho endpoint admin, rate limiting, IAM authentication.

**Shared code** — npm workspace `@gameapi/shared` chứa models, middlewares, utils, SQS producer, services dùng chung.

**Maintenance** — 4 EventBridge rules: bật/tắt bảo trì (thứ Hai), vacuum analyze (3h sáng), reset daily (0h sáng).

**Triển khai** — Docker Compose cho local dev, Serverless Framework + esbuild cho AWS production.

### 2.6 Lộ trình

Tuần 4: Khảo sát và thiết kế giải pháp.

Tuần 5: Thiết lập hạ tầng AWS (RDS, API Gateway, SQS, EventBridge, IAM).

Tuần 6: Triển khai Lambda Auth + Economy + consumer tương ứng.

Tuần 7: Triển khai Lambda Inventory + Transaction + consumer tương ứng.

Tuần 8: Triển khai Lambda Progression + Loot-Reward + consumer tương ứng.

Tuần 9: Triển khai Maintenance Lambda, backup, monitoring.

Tuần 10: Kiểm thử tích hợp, load testing, cutover, go-live.

### 2.7 Ước tính ngân sách

**Giả định lượng tải và thông số hệ thống:**

* **Lượng người dùng:** 500 DAU (Daily Active Users).
* **Tổng lưu lượng:** ~5,000,000 requests/tháng qua API Gateway.
* **Kích thước & Mạng:** Kích thước dữ liệu trung bình 34 KB mỗi request/response. Lưu lượng truyền tải ra Internet (Data Transfer Out) khoảng 50 GB/tháng.
* **AWS Lambda:** Phân bổ bộ nhớ 512 MB RAM, thời gian thực thi trung bình 100 ms/request, sử dụng kiến trúc ARM để tối ưu chi phí.
* **Amazon SQS:** ~2,000,000 requests bất đồng bộ, phát sinh tổng cộng ~6,000,000 tác vụ. Tỷ lệ lỗi (chuyển vào DLQ) ước tính  **<0.1%** .
* **Lưu trữ:** Amazon S3 (~10 GB) và Amazon CloudWatch Logs (~10 GB nạp/lưu trữ mỗi tháng).
* **Cơ sở dữ liệu:** Amazon Aurora PostgreSQL chạy trung bình ở mức 2 ACU hoạt động 24/7.

| Service Name                           |            Monthly Cost (USD)            | Annual Cost (USD) |
| :------------------------------------- | :--------------------------------------: | :---------------: |
| Amazon Aurora PostgreSQL-Compatible DB |                  74.68                  |      896.16      |
| AWS Web Application Firewall (WAF)     |                  11.00                  |      132.00      |
| Amazon CloudFront                      |                  10.25                  |      123.00      |
| Amazon CloudWatch                      |                   7.05                   |       84.54       |
| Amazon API Gateway                     |                   6.25                   |       75.00       |
| AWS Lambda                             |                   4.33                   |       51.96       |
| Amazon Simple Queue Service (SQS)      |                   3.00                   |       36.00       |
| S3 Standard                            |                   0.51                   |       6.12       |
| Data Transfer                          |                   0.00                   |       0.00       |
| **Total**                        | **$117.07** |  **$1,404.78** |                  |

*(Ghi chú: Hình ảnh biểu đồ tỉ lệ chi phí dịch vụ tham chiếu tại Hình 5.2.10)*

**Đánh giá tổng quan:**
Với mức chi phí $117.07/tháng, hệ thống đáp ứng tốt cho tệp 500 người chơi. Điểm nổi bật của kiến trúc này là chi phí cho hạ tầng tính toán (Lambda + API Gateway + SQS) chỉ chiếm chưa tới $15/tháng, trong khi vẫn đảm bảo khả năng tự động mở rộng linh hoạt. Thành phần tốn kém nhất là Aurora Database và RDS Proxy (~$75/tháng), đây là khoản đầu tư bắt buộc để duy trì tính ACID, quản lý kết nối an toàn và hiệu năng lưu trữ cho game.

### 2.8 Đánh giá rủi ro

**Cold start Lambda:**
Khi không có request trong thời gian dài, Lambda bị thu hồi tài nguyên. Request đầu tiên sau đó sẽ bị chậm do phải khởi tạo runtime và kết nối database. Mức độ ảnh hưởng trung bình — người chơi có thể bị lag nhẹ ở lần request đầu tiên. Giảm thiểu bằng cách sử dụng Provisioned Concurrency cho các Lambda quan trọng để giữ sẵn môi trường chạy.

**Rủi ro bị tấn công CSDL:**
Việc đặt Aurora RDS tại Public Subnet để tiết kiệm chi phí NAT Gateway sẽ mở ra một public endpoint, khiến cơ sở dữ liệu có nguy cơ bị dò IP hoặc brute-force trực tiếp từ Internet. Mức độ ảnh hưởng: Rất cao. Giảm thiểu bằng cách siết chặt tối đa Security Group, triển khai IAM Database Authentication để loại bỏ mật khẩu tĩnh, và đưa cấu hình Private Subnet vào lộ trình nâng cấp ưu tiên khi ngân sách cho phép.

**FIFO queue throughput giới hạn:**
SQS FIFO giới hạn 3000 TPS — nếu vượt quá, message sẽ bị throttling. Mức độ ảnh hưởng thấp vì với quy mô dưới 1000 người chơi cùng lúc, throughput này là quá đủ. Nếu cần mở rộng sau này, có thể tăng số lượng queue và sharding theo account.

**Mất dữ liệu hoặc duplicate message:**
SQS FIFO đảm bảo exactly-once processing nhưng consumer có thể crash sau khi xử lý nhưng trước khi acknowledge, dẫn đến duplicate. Mức độ ảnh hưởng thấp — các handler cần được thiết kế idempotent. DLQ với maxReceiveCount=3 đảm bảo message không bao giờ bị mất, và đội ngũ vận hành được cảnh báo qua CloudWatch alarm khi DLQ có message.

**Database connection pool exhaustion:**
Mỗi Lambda instance tạo connection pool riêng đến database. Nếu có nhiều Lambda instance chạy đồng thời, số lượng connection có thể vượt quá giới hạn của RDS. Mức độ ảnh hưởng trung bình-cao. Giảm thiểu bằng cách giới hạn pool size mỗi Lambda (tối đa 2-5 connections), sử dụng RDS Proxy để quản lý connection pool tập trung, và tận dụng IAM authentication để tránh lưu password.

**Chi phí AWS tăng đột biến:**
Khi game viral hoặc có sự kiện lớn, lượng request tăng đột biến kéo theo chi phí Lambda, API Gateway, và database tăng theo. Mức độ ảnh hưởng cao — có thể dẫn đến hóa đơn AWS bất ngờ. Giảm thiểu bằng cách thiết lập AWS Budget Alert ở 3 ngưỡng (50%, 80%, 100%), giới hạn API Gateway usage plan, rate limiting, và monitoring qua CloudWatch dashboard.

**Lambda timeout với dữ liệu lớn:**
Save data của người chơi có thể rất lớn, dẫn đến Lambda timeout (mặc định 30s, tối đa 900s). Mức độ ảnh hưởng trung bình. Giảm thiểu bằng cách tăng Lambda timeout lên phù hợp, chia nhỏ save data thành nhiều phần, hoặc xử lý bất đồng bộ qua SQS cho các tác vụ nặng.

**Phụ thuộc vào AWS:**
Toàn bộ hệ thống chạy trên AWS — nếu AWS gặp sự cố regional (outage ở ap-southeast-1), game sẽ ngừng hoạt động hoàn toàn. Mức độ ảnh hưởng cao nhưng xác suất thấp. Giảm thiểu bằng cách thiết lập multi-AZ cho RDS, có kế hoạch DR với cross-region backup, và tài liệu hướng dẫn recovery chi tiết.

### 2.9 Kết quả kỳ vọng

**Giảm chi phí vận hành:**
Với Lambda pay-per-use, chi phí tính toán chỉ phát sinh khi có request từ người chơi. Không còn lãng phí tài nguyên chạy server 24/7. Dự kiến tiết kiệm 40-60% chi phí vận hành so với monolith chạy server cố định, đặc biệt là giai đoạn đầu khi lượng người chơi còn thấp.

**Khả năng mở rộng linh hoạt:**
Mỗi domain scale độc lập dựa trên tải thực tế. Auth Lambda có thể scale lên 1000 instance trong khi Inventory Lambda chỉ cần 10 instance. API Gateway auto-scale theo số request. Aurora Serverless v2 tự động điều chỉnh ACU theo tải database. Không còn bottleneck toàn hệ thống.

**Triển khai nhanh và an toàn:**
Deploy từng Lambda riêng lẻ, không ảnh hưởng đến các domain khác. Thời gian deploy giảm từ 10-15 phút xuống còn 1-2 phút cho mỗi lambda. Rollback độc lập nếu có sự cố. CI/CD đơn giản hơn với Serverless Framework.

**Xử lý bất đồng bộ tin cậy:**
FIFO queue đảm bảo message được xử lý đúng thứ tự và không bị mất. DLQ + CloudWatch alarm cho phép phát hiện và xử lý kịp thời các message thất bại. Retry tự động tối đa 3 lần. Idempotent handler tránh duplicate do retry.

**Sẵn sàng production:**
AWS Backup tự động (daily + weekly) với retention policy rõ ràng. CloudWatch monitoring và alarm cho tất cả các thành phần (DLQ, Lambda errors, API Gateway 5xx, database connections). Bảo mật đa lớp: JWT, admin secret, rate limiting, IAM authentication. Maintenance mode cho phép bảo trì có kiểm soát mà không ảnh hưởng đến dữ liệu.

**Dễ dàng chuyển đổi:**
Monolith có thể chạy song song trong quá trình cutover. Có thể chuyển dần từng domain từ monolith sang Lambda mà không cần dừng game. Rollback nhanh chóng bằng cách chuyển API Gateway về monolith nếu phát hiện vấn đề.
