# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phùng Hồng Phước  Mã học viên: 2A202601215

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên cấu hình AGENT_API_KEY thì việc app dừng ngay lúc khởi động giúp tôi phát hiện lỗi cấu hình trước khi service nhận traffic thật. Tôi có thể nhìn log deploy và sửa biến môi trường ngay. Nếu code dùng mặc định "changeme", service vẫn có thể khởi động và public ra Internet. Khi đó bất kỳ ai biết hoặc đoán được key mặc định đều có thể gọi /ask, làm phát sinh request LLM và chi phí. Vì vậy fail fast biến một lỗi bảo mật khó phát hiện thành một lỗi deploy rõ ràng và dễ sửa.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Status: 401
Body: {"detail":"invalid or missing API key"}

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 173 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Khi build thực tế, image 1-stage của tôi có dung lượng 1.73 GB, còn image multi-stage chỉ còn 271 MB. Như vậy bản production nhỏ hơn khoảng 1.46 GB, tức giảm hơn 80% dung lượng.

Phần dung lượng chênh lệch chủ yếu đến từ hai thay đổi. Thứ nhất, bản đầu dùng python:3.11 đầy đủ, còn bản production dùng python:3.11-slim, nên loại bỏ nhiều package hệ thống không cần thiết. Thứ hai, multi-stage build tách phần cài/build dependency sang stage builder, còn stage runtime chỉ copy những thành phần cần để chạy ứng dụng. Vì vậy image cuối không phải mang theo toàn bộ môi trường build và các công cụ không cần thiết khi chạy service.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi copy requirements.txt và chạy pip install trước khi copy source code. Vì vậy khi tôi chỉ sửa một ký tự trong app/main.py, các layer từ base image, WORKDIR, COPY requirements.txt và layer cài dependency vẫn có thể dùng lại từ cache. Docker chỉ cần chạy lại từ layer COPY . . trở đi.

Nếu đặt COPY . . trước RUN pip install, chỉ cần thay đổi một file Python thì layer COPY . . đã thay đổi. Các layer sau nó sẽ mất cache, bao gồm cả pip install, dù requirements.txt hoàn toàn không đổi. Kết quả là thời gian build lâu hơn và phải cài lại dependency không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ ứng dụng Python có một lỗ hổng cho phép kẻ tấn công thực thi lệnh trong container. Nếu process ứng dụng đang chạy bằng root, mã của kẻ tấn công cũng có quyền root bên trong container. Khi kết hợp với một lỗi cấu hình như mount tài nguyên nhạy cảm hoặc một lỗ hổng container runtime/kernel, quyền cao đó làm hậu quả nghiêm trọng hơn và có thể trở thành bước đệm để tác động tới host.

Lệnh USER appuser làm cho process ứng dụng chạy với user thường thay vì root. Vì vậy ngay cả khi kẻ tấn công chiếm được process Python, họ chỉ nhận quyền của appuser, bị giới hạn quyền truy cập file và thao tác hệ thống. USER không tự loại bỏ mọi khả năng container escape, nhưng nó giảm quyền của kẻ tấn công và giảm mức độ thiệt hại nếu ứng dụng bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây. Ví dụ họ gửi 10 request vào khoảng 10:00:59, vẫn thuộc phút 10:00, nên chưa vượt giới hạn 10 request/phút. Ngay sau khi đồng hồ sang 10:01:00, bộ đếm của phút mới được reset và họ gửi tiếp 10 request nữa vào 10:01:00 hoặc 10:01:01.

Như vậy trong khoảng thời gian thực tế chỉ khoảng 2 giây, server đã nhận 20 request dù giới hạn được mô tả là 10 request/phút. Sliding window tránh lỗ hổng này vì tại thời điểm request mới, nó luôn đếm các request trong 60 giây gần nhất chứ không phụ thuộc vào ranh giới phút trên đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tần suất/số lượng request trong một khoảng thời gian, còn cost guard kiểm soát tổng số tiền đã tiêu hoặc dự kiến tiêu. Hai cơ chế bảo vệ hai loại rủi ro khác nhau.

Ví dụ rate limit là 10 request/phút và user chỉ gửi 2 request trong phút đó, nên rate limiter cho qua. Tuy nhiên nếu user đã gần hoặc vượt ngân sách tháng, cost guard phải chặn request với HTTP 402 dù tốc độ gửi request rất thấp.

Trường hợp ngược lại: user còn rất nhiều ngân sách trong tháng nên cost guard vẫn cho phép, nhưng họ gửi hơn 10 request trong vòng 60 giây. Khi đó rate limiter phải trả HTTP 429 mặc dù ngân sách vẫn còn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu /health cũng kiểm tra Redis thì khi Redis mất kết nối, cả 3 container đều gọi health check và cùng nhận kết quả thất bại. Orchestrator có thể hiểu rằng cả 3 process ứng dụng đều bị lỗi và bắt đầu restart chúng. Các container mới khởi động lên vẫn gặp Redis đang mất kết nối, health check lại fail và tiếp tục bị restart. Như vậy một lỗi tạm thời của dependency Redis bị biến thành việc cả cụm ứng dụng liên tục restart.

Khi tách riêng hai endpoint, /health chỉ kiểm tra process ứng dụng còn sống nên vẫn trả 200. /ready kiểm tra Redis và trả 503, nên load balancer tạm ngừng đưa request mới vào instance nhưng không nhất thiết restart container. Khi Redis hoạt động lại, /ready trở lại 200 và các instance có thể nhận traffic tiếp.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

>Khi các instance cùng dùng Redis, dù request tiếp theo được load balancer đưa tới container nào thì chúng đều đọc cùng key lịch sử của X-User-Id. Vì mỗi lần /ask lưu một message của user và một message của assistant, history_length có thể tăng theo dạng 0, 2, 4, 6... cho tới giới hạn lưu lịch sử.

Nếu dùng một dict Python trong RAM, mỗi container sẽ có một bản lịch sử riêng. Khi request được phân phối lần lượt giữa 3 instance, có lúc tôi sẽ thấy history_length tăng, nhưng request tiếp theo sang container khác lại có thể trở về 0 hoặc một giá trị nhỏ hơn. Người dùng sẽ có cảm giác agent lúc nhớ, lúc quên vì mỗi instance chỉ thấy state trong RAM của chính nó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi deploy lên Render, /health và /ready đều trả 200 nhưng khi tôi kiểm tra /ask bằng PowerShell, request trả về:

HTTP/1.1 422 Unprocessable Entity
JSON decode error
Expecting property name enclosed in double quotes

Ban đầu tôi kiểm tra service vì nghĩ có thể endpoint /ask bị lỗi, nhưng /health và /ready vẫn hoạt động bình thường. Nội dung lỗi JSON decode error cho thấy request đã tới FastAPI nhưng body không phải JSON hợp lệ. Tôi nhận ra nguyên nhân là cách escape dấu nháy khi dùng curl.exe trong PowerShell làm chuỗi {"question":"Hello"} bị biến đổi trước khi gửi.

Tôi đổi sang Invoke-WebRequest/Invoke-RestMethod và gửi body JSON đúng định dạng. Sau khi sửa, request không có API key trả đúng 401, còn request có X-API-Key hợp lệ trả 200 với answer, user_id, history_length, cost_usd và thông tin token. Sau đó tôi cũng kiểm tra rate limit và nhận đúng 10 lần 200 rồi 5 lần 429.
