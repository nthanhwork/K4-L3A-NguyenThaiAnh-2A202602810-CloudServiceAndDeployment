# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời trực tiếp bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thái Anh  Mã học viên: 2A202602810

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên nền tảng đám mây (Railway/Render) mà kỹ sư quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. Nếu để giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động thành công và mở cổng ra internet. Kẻ quét bot hoặc người lạ trên mạng có thể thử key mặc định `"changeme"` để gọi API, dẫn tới việc dùng cạn hạn mức token/tiền của ta mà ta không hay biết. Ngược lại, việc không để giá trị mặc định khiến Pydantic ném `ValidationError` và crash ngay lập tức lúc khởi động container; ta sẽ thấy cảnh báo lỗi đỏ ngay trên màn hình deploy console và bổ sung biến môi trường ngay trước khi có bất kỳ traffic nào đi vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:51:26.412351+00:00", "user_id": "sv-test", "tokens_in": 41, "tokens_out": 45, "cost_usd": 3.315e-05}`

Hai việc làm được mà `print` không làm được:
1. Hệ thống giám sát log tập trung (Datadog, CloudWatch, Railway Logs) có thể tự động parse các trường số có cấu trúc (như `cost_usd`, `tokens_out`, `tokens_in`) để kích hoạt cảnh báo (alert) tự động khi chi phí của một user vượt ngưỡng bất thường trong thời gian ngắn.
2. Có thể thực hiện các câu truy vấn tổng hợp (aggregation queries) trên log engine, ví dụ: thống kê tổng chi phí token của toàn bộ user trong ngày, tính thời gian phản hồi trung bình hoặc lọc danh sách top người dùng tiêu tốn token nhiều nhất mà không cần viết regex bóc tách xâu ký tự thủ công.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 195 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~825 MB) bao gồm toàn bộ công cụ biên dịch (build toolchain như gcc, g++, build-essential), các file header C/C++ của Linux kernel, bộ nhớ đệm pip wheel cache và các tệp tin tạm phát sinh trong quá trình cài đặt thư viện. Trong Dockerfile multi-stage, stage `builder` thực hiện toàn bộ việc biên dịch cài đặt vào thư mục `/install`, sau đó stage `runtime` chỉ sao chép kết quả đã cài đặt sang và loại bỏ hoàn toàn các trình biên dịch không cần thiết khi chạy ứng dụng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Các layer được dùng lại từ cache: `FROM python:3.11-slim`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`, và tạo user `appuser`.
- Các layer phải chạy lại: `COPY app ./app`, `COPY utils ./utils`, `USER appuser`, `CMD ...`.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa một ký tự trong code, layer copy source code bị thay đổi nội dung khiến Docker hủy bỏ toàn bộ cache từ layer đó trở đi. Kết quả là Docker buộc phải chạy lại lệnh `pip install`, tải và cài lại toàn bộ thư viện từ đầu, làm thời gian build kéo dài thêm vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Kẻ tấn công phát hiện và khai thác một lỗ hổng thực thi mã từ xa (RCE) trong ứng dụng Python hoặc thư viện phụ thuộc. Do container mặc định chạy bằng user root, tiến trình Python có UID 0 bên trong container. Vì Linux container chia sẻ chung nhân kernel với máy host, nếu container bị cấu hình sai (mount volume nhạy cảm, cấp cờ privileged, hoặc có lỗ hổng container breakout), tiến trình UID 0 của kẻ tấn công có thể thoát ra khỏi container và thao tác trực tiếp với quyền root trên kernel của máy host, chiếm quyền kiểm soát toàn bộ hệ thống.
- Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi sự kiện: Khi chuyển sang user thường không có đặc quyền, mã độc của kẻ tấn công chỉ chạy dưới quyền của `appuser`. Kẻ tấn công bị hệ điều hành chặn truy cập các file hệ thống, không thể ghi đè file cấu hình và không thể thực hiện các bước leo thang đặc quyền để thoát ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Giải thích: Với cách đếm theo phút đồng hồ, hạn mức được đặt lại vào giây 00 của mỗi phút mới. Kẻ tấn công có thể gửi 10 request vào giây cuối cùng của phút trước (ví dụ lúc `10:00:59`). Ngay giây tiếp theo (`10:01:00`), đồng hồ bước sang phút mới và bộ đếm reset về 0, kẻ tấn công gửi tiếp 10 request nữa. Tổng cộng có 20 request được gửi trong khoảng thời gian từ `10:00:59` đến `10:01:00` (2 giây) mà không bị hệ thống chặn. Thuật toán cửa sổ trượt (Sliding Window) 60 giây khắc phục triệt để lỗ hổng này bằng cách tính tổng số request trong đúng 60 giây trôi ngược từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau: Rate limit giới hạn *tần suất / số lượng request* trong một đơn vị thời gian ngắn (nhằm chống nghẽn mạng, chống tấn công DoS hạ tầng). Cost guard giới hạn *tổng chi phí tài chính / hạn mức token* lũy kế theo chu kỳ tháng (nhằm bảo vệ ngân sách, chống cháy tài khoản LLM).
- Rate limit cho qua nhưng Cost guard chặn: Một người dùng chỉ gửi duy nhất 1 request trong vòng 10 phút (tần suất cực thấp, Rate limit cho qua), nhưng request này yêu cầu xử lý tài liệu dài 500.000 tokens khiến chi phí dự kiến vượt quá ngân sách 10.0 USD của tháng -> Cost guard lập tức chặn và trả lỗi 402 Payment Required.
- Cost guard cho qua nhưng Rate limit chặn: Một người dùng gửi 15 request liên tiếp trong vòng 3 giây với câu hỏi "Hi" (mỗi request chỉ tốn vài token ~ 0.00001 USD, tổng chi phí chưa tới 0.01% ngân sách tháng -> Cost guard cho qua), nhưng do vượt quá 10 req/phút -> Rate limit chặn từ request thứ 11 và trả lỗi 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Khi Redis mất kết nối, cả 3 container agent đều nhận thấy Redis không phản hồi và đồng loạt trả về mã 503 tại endpoint kiểm tra sức khỏe gộp.
2. Container Orchestrator (Docker/Kubernetes/Cloud) coi mã 503 của liveness probe là ứng dụng bị treo (deadlock), lập tức ra lệnh kill và restart lại toàn bộ cả 3 container cùng lúc.
3. Trong suốt 30 giây Redis chưa hồi phục, các container vừa khởi động lại tiếp tục thấy Redis chết, lại báo 503 và lại bị restart liên tục (rơi vào trạng thái crash-loop backoff).
4. Khi Redis có kết nối trở lại, cả 3 container vẫn đang trong quá trình bị restart và khởi động lại dở dang, khiến toàn bộ hệ thống bị tê liệt hoàn toàn trong thời gian dài hơn thực tế sự cố. (Nếu tách riêng: `/health` vẫn báo 200 giúp container không bị restart vô ích, chỉ `/ready` báo 503 để Load Balancer tạm ngừng chuyển hướng traffic).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong một dict Python nội bộ của tiến trình: Do 3 container (A, B, C) chạy độc lập sau Load Balancer, mỗi request gửi đến sẽ được phân phối ngẫu nhiên (round-robin) tới một trong 3 container. Request 1 vào A (lưu câu hỏi 1, `history_length`=0). Request 2 vào B (B không có RAM của A, nên `history_length` vẫn là 0). Request 3 vào C (lại là 0). Request 4 vào lại A (thấy lại câu hỏi 1, `history_length`=2). Con số `history_length` sẽ nhảy lộn xộn (0, 0, 2, 0, 2...) và agent liên tục bị "mất trí nhớ" tùy thuộc vào việc request rơi vào container nào. Khi lưu trên Redis tập trung, cả 3 container cùng đọc ghi một nơi nên `history_length` luôn tăng đều đặn (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi gặp phải: Lỗi `HTTP/2 500 Internal Server Error` khi gọi `/ready` và `/ask` trên domain Railway, kèm lỗi SSL Certificate `curl: (60) SSL: no alternative certificate subject name matches target hostname` và `301 Moved Permanently`.
- Cách tìm ra nguyên nhân: Kiểm tra URL thấy tên miền đang dùng chứa `.railway.internal.up.railway.app` (nhầm lẫn giữa Private Network nội bộ của Railway và Public Domain), và lệnh curl thiếu `https://` nên bị redirect 301. Đồng thời khi xem tab Variables và Logs của service, biến `AGENT_API_KEY` và `REDIS_URL` lúc đầu ở dạng nháp chưa được deploy nên code FastAPI gọi `Settings()` ném `ValidationError`.
- Cách sửa: Vào mục Settings -> Public Networking trên Railway xóa domain cũ và bấm Generate Domain chuẩn dạng `...-production.up.railway.app`; trong tab Variables điền đầy đủ `AGENT_API_KEY` từ `.env`, gán `REDIS_URL = ${{Redis.REDIS_URL}}`, sau đó bấm nút tím `Deploy` để container khởi động lại với đầy đủ biến môi trường.
