# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thái Anh |
| Mã học viên | 2A202602810 |
| Repo | https://github.com/nthanhwork/K4-L3A-NguyenThaiAnh-2A202602810-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (${{Redis.REDIS_URL}}) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-l3a-nguyenthaianh-2a202602810-cloudserviceand-production.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 09:51:23 GMT
server: railway-hikari
x-railway-request-id: qVO1bDD2ThmWQLKhwUFZXw
content-length: 57
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 09:51:23 GMT
server: railway-hikari
x-railway-request-id: 3p5nlfHPQxCPK9qc9o6EoQ
content-length: 31
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1

{"status":"ready","redis":true}

# 3. Không có API key
HTTP/2 401 
content-type: application/json
date: Mon, 28 Sep 2026 09:51:25 GMT
server: railway-hikari
x-railway-request-id: vK5DxijDSIqwyLmm9o6EoQ
content-length: 39
x-hikari-trace: hkg1.hn7d
x-railway-edge: hkg1

{"detail":"invalid or missing API key"}

# 4. Có API key
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 09:51:26 GMT
server: railway-hikari
x-railway-request-id: pN58dbqhSbOofg-V2prcFg
content-length: 337
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1
vary: accept-encoding

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":2,"cost_usd":3.315e-05,"tokens":{"in":41,"out":45}}

# 5. Rate limit (15 lần)
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
