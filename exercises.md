# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền trực tiếp nội dung vào bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phan Danh Dat  Mã học viên: 02627

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên môi trường Production (như Render, Railway, K8s), lập trình viên vô tình quên không cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. Nếu để mặc định là `"changeme"`, service vẫn khởi động thành công và mở cổng ra internet. Kẻ tấn công hoặc bot quét API có thể dễ dàng đoán ra khóa mặc định `"changeme"` và gọi API thoải mái, dẫn đến bị lạm dụng tài nguyên hoặc làm tiêu tốn toàn bộ ngân sách OpenAI/Anthropic API hàng nghìn USD mà ta không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, việc "fail-fast" khiến app crash ngay lúc khởi động (`ValidationError`), buộc deploy thất bại ngay lập tức để lập trình viên nhìn thấy lỗi ngay trên console/log deploy và khắc phục trước khi service nhận bất kỳ traffic nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log mẫu thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:00:00.000000+00:00", "user_id": "sv-123", "tokens_in": 12, "tokens_out": 24, "cost_usd": 0.000072}`

Hai việc làm được:
1. **Truy vấn, lọc và phân tích dữ liệu có cấu trúc (Structured Query & Aggregation):** Có thể tích hợp với các hệ thống gom log (Datadog, CloudWatch, Elasticsearch/Kibana) để lọc chính xác tất cả request của riêng `user_id = 'sv-123'` hoặc tính tổng `cost_usd` đã tiêu thụ theo ngày/tháng mà không cần viết regex bóc tách chuỗi phức tạp.
2. **Thiết lập giám sát và cảnh báo tự động theo ngưỡng (Automated Alerting & Monitoring):** Có thể dễ dàng cấu hình metric tự động kích hoạt cảnh báo (alert) khi số lượng `level = "error"` tăng đột biến trong 5 phút, hoặc khi `cost_usd` của một request vượt quá hạn mức an toàn.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | ~271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~780 MB) bao gồm hệ điều hành Debian đầy đủ đi kèm các công cụ phát triển và biên dịch C/C++ (`build-essential`, `gcc`, `g++`, header files), các thư viện tĩnh/động không dùng ở runtime, pip build cache, và các tiện ích hệ thống thừa thãi. Stage runtime của Multi-stage build dùng base `python:3.11-slim` chỉ copy duy nhất thư mục dependencies đã được cài đặt (`/usr/local`), loại bỏ toàn bộ compiler và công cụ build khỏi image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile chuẩn: các layer từ đầu cho đến `RUN pip install` và `RUN useradd` đều được lấy lại nguyên vẹn từ cache (`CACHED`). Chỉ có layer `COPY app ./app` và các chỉ thị bên dưới nó là phải build lại, quá trình build chỉ mất 1-2 giây.
Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa dù chỉ một ký tự trong `app/main.py`, cache của layer `COPY . .` sẽ bị mất hiệu lực, kéo theo lệnh `RUN pip install` ở ngay sau buộc phải chạy lại từ đầu, tải và cài đặt lại toàn bộ thư viện tốn rất nhiều thời gian và băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python tồn tại lỗ hổng thực thi mã từ xa (RCE, command injection hoặc deserialization).
2. Kẻ tấn công kích hoạt lỗ hổng để thực thi shellcode trong container. Vì container mặc định chạy bằng root (UID 0), kẻ tấn công chiếm được quyền root bên trong container.
3. Kẻ tấn công khai thác tiếp một lỗ hổng container escape (hoặc kernel exploit, misconfigured capabilities, docker socket mount). Do UID 0 trong container map trực tiếp với UID 0 trên máy host, kẻ tấn công lập tức có toàn quyền root trên máy host thật.
Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 2: Tiến trình Python chỉ chạy với quyền của user không đặc quyền (`UID 10001`). Ngay cả khi bị RCE, kẻ tấn công không có quyền root, không thể sửa đổi file hệ thống và bị tước hầu hết Linux capabilities nguy hiểm, chặn đứng khả năng escape ra host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Giải thích: Người dùng gửi 10 request vào giây 59 của phút trước (10:00:59). Khi đồng hồ chuyển sang giây 00 của phút tiếp theo (10:01:00), bộ đếm theo phút đồng hồ tự động reset về 0, cho phép người dùng gửi thêm 10 request nữa ngay tại giây này (10:01:00). Như vậy trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải nhận tổng cộng 20 request, vượt gấp đôi hạn mức 10/phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau: Rate limit kiểm soát tần suất số lượng request trong một khoảng thời gian ngắn (chống nghẽn hạ tầng, spam request); Cost guard kiểm soát tổng ngân sách tài chính tích lũy qua các request trong tháng (chống cạn kiệt ngân sách do chi phí token của LLM).
- Tình huống Rate limit cho qua nhưng Cost guard chặn: User gọi API rất chậm rãi, cả tháng chỉ gửi 1 request mỗi 10 phút (dưới xa ngưỡng 10 request/phút), nhưng mỗi request có prompt dài 100k tokens hoặc user đã tiêu hết sạch ngân sách $10 của tháng -> Cost guard chặn với HTTP 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: User mới kích hoạt tài khoản trong tháng và còn nguyên $10 ngân sách, nhưng gửi liên tiếp 15 request chỉ trong vòng 3 giây -> Rate limit kích hoạt và chặn với HTTP 429 Too Many Requests, dù user vẫn còn tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mạng hoặc khởi động lại, không phản hồi trong 30 giây.
2. Bộ dò liveness probe `/health` của cả 3 container gọi kiểm tra Redis và đồng loạt thất bại.
3. Orchestrator (Docker/Kubernetes) nhận định cả 3 instance đều bị lỗi/deadlock không thể phục hồi và tự động kill rồi restart liên tục cả 3 container.
4. Trong suốt 30 giây sự cố, toàn bộ cụm container rơi vào vòng lặp restart (CrashLoop), tiêu tốn tài nguyên và cắt đứt toàn bộ request của người dùng.
5. Khi Redis hoạt động trở lại, các container vẫn đang dở dang trong quá trình khởi động lại, biến một lỗi tạm thời của dịch vụ phụ thuộc thành sự cố sụp đổ toàn diện (cascading failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi dùng Redis (stateless), `history_length` tăng đều đặn theo mỗi lượt hỏi (0, 2, 4, 6...) vì cả 3 instance đều truy xuất vào cùng một kho dữ liệu trung tâm.
Nếu lưu trong dict Python trong bộ nhớ process: Do load balancer phân phối các request luân phiên qua lại giữa 3 container khác nhau, mỗi container chỉ lưu lịch sử của riêng nó. Con số `history_length` sẽ nhảy thất thường (ví dụ: lượt 1 vào container A -> 0; lượt 2 vào B -> 0; lượt 3 vào C -> 0; lượt 4 vào A -> 2; lượt 5 vào B -> 2). Kết quả là agent sẽ liên tục bị "mất trí nhớ" và không thể duy trì ngữ cảnh trò chuyện mạch lạc với người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp: Health check timeout / Port binding failure khi deploy lên Cloud Platform (Render/Railway).
- Thông báo lỗi: `Port binding failed: Service failed to listen on assigned PORT` hoặc `Deployment failed: Healthcheck timed out after 300s`.
- Cách tìm ra nguyên nhân: Xem trực tiếp Deploy log và Runtime log trên dashboard của platform, phát hiện Uvicorn cố định lắng nghe cổng 8000 (`Uvicorn running on http://0.0.0.0:8000`) trong khi nền tảng đám mây lại tự động cấp một cổng động qua biến môi trường `$PORT` (ví dụ `PORT=10000`).
- Cách sửa: Cập nhật lệnh khởi động trong Dockerfile thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` và đảm bảo cấu hình `Settings` đọc biến môi trường `PORT`, giúp container tự động bind vào đúng cổng mà platform cấp phát.
