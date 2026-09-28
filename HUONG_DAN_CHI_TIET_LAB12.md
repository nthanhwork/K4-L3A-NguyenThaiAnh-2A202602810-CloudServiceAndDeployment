# HƯỚNG DẪN CHI TIẾT TOÀN BỘ CHECKPOINT — DAY 12: CLOUD SERVICES & DEPLOYMENT

> **Bài lab cá nhân — K4 Level 3A**  
> **Repository:** `K4-L3A-NguyenThaiAnh-2A202602810-Cloud-Service-And-Deployment`  
> **Thang điểm:** 100 điểm chính (85đ từ 5 Checkpoint + 15đ từ `exercises.md`) + 10đ Bonus CI/CD  
> **Môi trường Python:** Conda `lab` (`/home/sakana/miniconda3/envs/lab/bin/python`)

---

## MỤC LỤC
1. [Tổng Quan Kiến Trúc & Sơ Đồ Hệ Thống](#1-tổng-quan-kiến-trúc--sơ-đồ-hệ-thống)
2. [CP0 — Setup Môi Trường & Khởi Động Redis](#2-cp0--setup-môi-trường--khởi-động-redis)
3. [CP1 — 12-Factor Config, Health & Logging (15 điểm)](#3-cp1--12-factor-config-health--logging-15-điểm)
4. [CP2 — Docker Multi-Stage Build & Compose (15 điểm)](#4-cp2--docker-multi-stage-build--compose-15-điểm)
5. [CP3 — API Security: Auth, Rate Limit & Cost Guard (20 điểm)](#5-cp3--api-security-auth-rate-limit--cost-guard-20-điểm)
6. [CP4 — Scaling & Reliability: Stateless Store, Readiness & Shutdown (20 điểm)](#6-cp4--scaling--reliability-stateless-store-readiness--shutdown-20-điểm)
7. [CP5 — Cloud Deployment & DEPLOYMENT.md (15 điểm)](#7-cp5--cloud-deployment--deploymentmd-15-điểm)
8. [Exercises — Hướng Dẫn Trả Lời 10 Câu Hỏi Phản Ánh (15 điểm)](#8-exercises--hướng-dẫn-trả-lời-10-câu-hỏi-phản-ánh-15-điểm)
9. [Bonus — CI/CD Với GitHub Actions (+10 điểm)](#9-bonus--cicd-với-github-actions-10-điểm)
10. [Checklist Tổng Kết Trước Khi Nộp Bài](#10-checklist-tổng-kết-trước-khi-nộp-bài)

---

## 1. TỔNG QUAN KIẾN TRÚC & SƠ ĐỒ HỆ THỐNG

### Luồng xử lý một request tới `/ask`:
```
Client
  │
  ▼ [X-API-Key, X-User-Id]
1. verify_api_key (auth.py) ──────────────► 401 Unauthorized (sai / thiếu key)
  │
  ▼
2. RateLimiter.check (rate_limiter.py) ───► 429 Too Many Requests (ZSET sliding window)
  │
  ▼
3. CostGuard.check (cost_guard.py) ──────► 402 Payment Required (vượt budget tháng)
  │
  ▼
4. ConversationStore.get_history ────────► Đọc history từ Redis (stateless)
  │
  ▼
5. ask_llm (utils/mock_llm.py) ──────────► Sinh câu trả lời & tính chi phí tokens
  │
  ▼
6. ConversationStore.append (x2) ────────► Ghi câu hỏi + câu trả lời vào Redis
  │
  ▼
7. CostGuard.record ─────────────────────► Cộng dồn chi phí tháng vào Redis
  │
  ▼
8. log_event (logging_utils.py) ────────► In 1 dòng JSON log ra stdout
  │
  ▼
Trả về JSON Response 200 cho Client
```

---

## 2. CP0 — SETUP MÔI TRƯỜNG & KHỞI ĐỘNG REDIS

### Bước 1: Kích hoạt môi trường
```bash
conda activate lab
# Hoặc dùng đường dẫn trực tiếp: /home/sakana/miniconda3/envs/lab/bin/python
```

### Bước 2: Tạo file `.env`
Đảm bảo đã tạo `.env` từ `.env.example` và sinh `AGENT_API_KEY`:
```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_urlsafe(32))"
# Dán chuỗi sinh được vào biến AGENT_API_KEY trong file .env
```

### Bước 3: Chạy Redis
```bash
docker compose up -d redis
docker compose ps
# Trạng thái phải hiển thị: Up ... (healthy)
```

### Bước 4: Kiểm tra ban đầu
```bash
pytest tests/ -v -m "not docker"
# Hầu hết test sẽ FAIL — điều này hoàn toàn bình thường vì code chưa được viết.
```

---

## 3. CP1 — 12-FACTOR CONFIG, HEALTH & LOGGING (15 ĐIỂM)

### Mục tiêu
- `app/config.py`: Đọc cấu hình từ biến môi trường, fail-fast nếu thiếu API key.
- `app/logging_utils.py`: Xuất log dạng JSON đúng 1 dòng stdout.
- `app/main.py`: Endpoint `/health` độc lập, không phụ thuộc Redis.

### File 1: `app/config.py`
Thay thế phần khai báo trong class `Settings`:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    port: int = 8000
    agent_api_key: str  # BẮT BUỘC: Không gán mặc định để fail-fast
    redis_url: str = "redis://localhost:6379/0"
    rate_limit_per_minute: int = 10
    monthly_budget_usd: float = 10.0
    log_level: str = "INFO"
```
*Điểm lưu ý:* `agent_api_key: str` không có giá trị mặc định để nếu deploy thiếu biến môi trường, ứng dụng sẽ dừng ngay lập tức (`ValidationError`), tránh bị khai thác miễn phí.

### File 2: `app/logging_utils.py`
Hoàn thiện hàm `log_event`:
```python
def log_event(event: str, level: str = "info", **fields) -> str:
    """Ghi một dòng log JSON ra stdout."""
    payload = {
        "event": event,
        "level": level.lower(),
        "timestamp": utc_now_iso(),
        **fields,
    }
    line = json.dumps(payload, ensure_ascii=False)
    print(line, file=sys.stdout, flush=True)
    return line
```
*Điểm lưu ý:* `json.dumps` không dùng `indent` để đảm bảo log nằm trên đúng **1 dòng duy nhất**.

### File 3: `app/main.py` (`/health`)
Triển khai hàm `health()`:
```python
@app.get("/health")
def health():
    """Liveness probe — process còn sống không?"""
    if lifecycle.shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    return {"status": "ok", "service": SERVICE_NAME, "version": SERVICE_VERSION}
```
*Điểm lưu ý:* Tuyệt đối không nhận dependency injection (`Depends`) và không gọi Redis trong `/health`.

### Kiểm tra CP1:
```bash
pytest tests/test_cp1.py -v
# Kết quả mong đợi: 13/13 passed
```

---

## 4. CP2 — DOCKER MULTI-STAGE BUILD & COMPOSE (15 ĐIỂM)

### Mục tiêu
- Multi-stage build để giảm dung lượng image từ ~1GB xuống dưới 500MB (thường ~200MB).
- Chạy bằng user không có quyền root (`appuser`).
- Copy `requirements.txt` trước khi copy source code để tận dụng layer cache.
- `HEALTHCHECK` gọi `/health` và đọc cổng từ `$PORT`.
- Cấu hình `.dockerignore` và service `agent` trong `docker-compose.yml`.

### File 1: `Dockerfile`
Thay thế toàn bộ nội dung `Dockerfile`:
```dockerfile
# Stage 1: Builder
FROM python:3.11-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim

WORKDIR /app

# Copy các gói đã build từ builder
COPY --from=builder /install /usr/local

# Tạo non-root user
RUN useradd --create-home --uid 10001 appuser

# Copy mã nguồn ứng dụng
COPY app ./app
COPY utils ./utils

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()" || exit 1

CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

### File 2: `.dockerignore`
Cập nhật file `.dockerignore`:
```
.git
.gitignore
.env
.env.*
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
tests
screenshots
exercises.md
DEPLOYMENT.md
README.md
CHECKPOINTS.md
RUBRIC.md
RULES.md
SUBMISSION.md
```

### File 3: `docker-compose.yml`
Bổ sung service `agent` vào `docker-compose.yml`:
```yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      AGENT_API_KEY: ${AGENT_API_KEY}
      REDIS_URL: redis://redis:6379/0
      RATE_LIMIT_PER_MINUTE: 10
      MONTHLY_BUDGET_USD: 10.0
      LOG_LEVEL: INFO
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  redis-data:
```

### Kiểm tra CP2:
```bash
# Kiểm tra tĩnh
pytest tests/test_cp2.py -v -m "not docker"

# Kiểm tra build thật (cần bật Docker)
pytest tests/test_cp2.py -v
```

---

## 5. CP3 — API SECURITY: AUTH, RATE LIMIT & COST GUARD (20 ĐIỂM)

### Mục tiêu
- `app/auth.py`: Chống timing attack với `secrets.compare_digest`.
- `app/rate_limiter.py`: Sliding window 60s dùng Redis Sorted Set (ZSET).
- `app/cost_guard.py`: Chặn mã 402 khi chi tiêu vượt ngân sách tháng.
- `app/main.py`: Ghép pipeline trong endpoint `/ask`.

### File 1: `app/auth.py`
```python
def verify_api_key(
    x_api_key: str | None = Header(default=None),
    x_user_id: str | None = Header(default=None),
) -> str:
    """Kiểm tra header X-API-Key; trả về user_id nếu hợp lệ."""
    expected_key = get_settings().agent_api_key
    if not x_api_key or not secrets.compare_digest(x_api_key, expected_key):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="invalid or missing API key",
        )
    return x_user_id if x_user_id else ANONYMOUS_USER
```

### File 2: `app/rate_limiter.py`
```python
class RateLimiter:
    def __init__(self, client, limit_per_minute: int) -> None:
        self.client = client
        self.limit = limit_per_minute

    @staticmethod
    def _key(user_id: str) -> str:
        return f"ratelimit:{user_id}"

    def hit_count(self, user_id: str, now: float | None = None) -> int:
        now = now if now is not None else time.time()
        key = self._key(user_id)
        # Xóa các entry cũ hơn 60 giây trước
        self.client.zremrangebyscore(key, 0, now - WINDOW_SECONDS)
        return self.client.zcard(key)

    def check(self, user_id: str, now: float | None = None) -> None:
        now = now if now is not None else time.time()
        key = self._key(user_id)
        current_hits = self.hit_count(user_id, now)
        if current_hits >= self.limit:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail="rate limit exceeded",
                headers={"Retry-After": str(WINDOW_SECONDS)},
            )
        # Ghi nhận request (member phải là duy nhất)
        self.client.zadd(key, {f"{now}:{uuid.uuid4().hex}": now})
        self.client.expire(key, WINDOW_SECONDS)
```

### File 3: `app/cost_guard.py`
```python
class CostGuard:
    def __init__(self, client, monthly_budget_usd: float) -> None:
        self.client = client
        self.budget = monthly_budget_usd

    @staticmethod
    def current_month() -> str:
        return datetime.now(timezone.utc).strftime("%Y-%m")

    @classmethod
    def _key(cls, user_id: str, month: str | None = None) -> str:
        return f"cost:{user_id}:{month or cls.current_month()}"

    def spent(self, user_id: str, month: str | None = None) -> float:
        val = self.client.get(self._key(user_id, month))
        return float(val) if val is not None else 0.0

    def check(
        self,
        user_id: str,
        estimated_cost: float = 0.0,
        month: str | None = None,
    ) -> None:
        if self.spent(user_id, month) + estimated_cost > self.budget:
            raise HTTPException(
                status_code=status.HTTP_402_PAYMENT_REQUIRED,
                detail="monthly budget exceeded",
            )

    def record(self, user_id: str, cost: float, month: str | None = None) -> float:
        key = self._key(user_id, month)
        total = self.client.incrbyfloat(key, cost)
        self.client.expire(key, KEY_TTL_SECONDS)
        return float(total)
```

### File 4: `app/main.py` (Endpoint `/ask`)
```python
@app.post("/ask")
def ask(
    payload: AskRequest,
    user_id: str = Depends(verify_api_key),
    store: ConversationStore = Depends(get_store),
    limiter: RateLimiter = Depends(get_rate_limiter),
    guard: CostGuard = Depends(get_cost_guard),
):
    # 1. Rate limiter check (chặn 429 nếu gọi quá nhanh)
    limiter.check(user_id)

    # 2. Cost guard check (chặn 402 nếu hết tiền)
    guard.check(user_id)

    # 3. Lấy lịch sử hội thoại
    history = store.get_history(user_id)

    # 4. Gọi LLM
    result = ask_llm(payload.question, history)

    # 5. Lưu hội thoại vào Redis store
    store.append(user_id, "user", payload.question)
    store.append(user_id, "assistant", result["answer"])

    # 6. Ghi nhận chi phí token phát sinh
    guard.record(user_id, result["cost_usd"])

    # 7. Ghi structured log
    log_event(
        "ask_completed",
        user_id=user_id,
        tokens_in=result["tokens_in"],
        tokens_out=result["tokens_out"],
        cost_usd=result["cost_usd"],
    )

    # 8. Trả kết quả
    return {
        "answer": result["answer"],
        "user_id": user_id,
        "history_length": len(history),
        "cost_usd": result["cost_usd"],
        "tokens": {"in": result["tokens_in"], "out": result["tokens_out"]},
    }
```

### Kiểm tra CP3:
```bash
pytest tests/test_cp3.py -v
# Kết quả mong đợi: 22/22 passed
```

---

## 6. CP4 — SCALING & RELIABILITY: STATELESS STORE, READINESS & SHUTDOWN (20 ĐIỂM)

### Mục tiêu
- `app/store.py`: Đưa state ra Redis, giới hạn 20 message gần nhất, TTL 7 ngày.
- `app/main.py`: Endpoint `/ready` kiểm tra Redis connection.
- `app/lifecycle.py`: Bắt tín hiệu SIGTERM/SIGINT và nhường lại quyền cho uvicorn handler cũ.

### File 1: `app/store.py`
```python
class ConversationStore:
    def __init__(self, client) -> None:
        self.client = client

    @staticmethod
    def _key(user_id: str) -> str:
        return f"history:{user_id}"

    def ping(self) -> bool:
        """Kiểm tra kết nối Redis (nuốt toàn bộ Exception)."""
        try:
            return bool(self.client.ping())
        except Exception:
            return False

    def append(self, user_id: str, role: str, content: str) -> None:
        key = self._key(user_id)
        message = json.dumps({"role": role, "content": content}, ensure_ascii=False)
        self.client.rpush(key, message)
        # Giữ 20 message mới nhất
        self.client.ltrim(key, -HISTORY_MAX_MESSAGES, -1)
        # TTL 7 ngày
        self.client.expire(key, HISTORY_TTL_SECONDS)

    def get_history(self, user_id: str) -> list[dict]:
        key = self._key(user_id)
        raw_items = self.client.lrange(key, 0, -1)
        if not raw_items:
            return []
        return [json.loads(item) for item in raw_items]

    def clear(self, user_id: str) -> None:
        self.client.delete(self._key(user_id))
```

### File 2: `app/main.py` (`/ready`)
```python
@app.get("/ready")
def ready(store: ConversationStore = Depends(get_store)):
    """Readiness probe — đã sẵn sàng nhận traffic chưa?"""
    if lifecycle.shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    if not store.ping():
        return JSONResponse(status_code=503, content={"status": "not ready", "redis": False})
    return {"status": "ready", "redis": True}
```

### File 3: `app/lifecycle.py`
```python
class Lifecycle:
    def __init__(self) -> None:
        self.shutting_down = False
        self._previous: dict = {}

    def request_shutdown(self, signum=None, frame=None) -> None:
        self.shutting_down = True
        previous = self._previous.get(signum)
        if callable(previous):
            previous(signum, frame)

    def install(self) -> None:
        for sig in (signal.SIGTERM, signal.SIGINT):
            self._previous[sig] = signal.getsignal(sig)
            signal.signal(sig, self.request_shutdown)
```

### Kiểm tra CP4:
```bash
pytest tests/test_cp4.py -v
# Kết quả mong đợi: 16/16 passed
```

---

## 7. CP5 — CLOUD DEPLOYMENT & DEPLOYMENT.MD (15 ĐIỂM)

### Phương án A: Triển khai Cloud thật (Khuyến nghị: Railway)
1. Cài CLI hoặc dùng web [railway.app](https://railway.app):
   - Tạo Project mới -> Add Database -> **Redis**.
   - Thêm Service từ GitHub repo.
   - Thêm biến môi trường trong Service Variables:
     - `AGENT_API_KEY`: `<khóa bí mật>`
     - `RATE_LIMIT_PER_MINUTE`: `10`
     - `MONTHLY_BUDGET_USD`: `10.0`
     - `LOG_LEVEL`: `INFO`
     - `REDIS_URL`: lấy biến `${{Redis.REDIS_URL}}` hoặc chuỗi kết nối Redis.
2. Sinh public domain: `Settings -> Networking -> Generate Domain`.
3. Kiểm tra bằng curl:
   ```bash
   URL=https://<domain-cua-ban>.up.railway.app
   curl -i $URL/health
   curl -i $URL/ready
   curl -i -X POST $URL/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
   ```
4. Cập nhật `DEPLOYMENT.md` và chụp 2 ảnh vào `screenshots/dashboard.png` và `screenshots/health.png`.

### Phương án B: Local Fallback (Dự phòng tối đa 9/15 điểm)
Nếu không dùng cloud:
1. Thêm `LOCAL_FALLBACK=true` vào file `.env`.
2. Chạy `docker compose up -d`.
3. Chụp màn hình `docker compose ps` và curl localhost vào thư mục `screenshots/`.
4. Ghi lý do vào cuối file `DEPLOYMENT.md`.

### Kiểm tra CP5:
```bash
pytest tests/test_cp5.py -v
```

---

## 8. EXERCISES — HƯỚNG DẪN TRẢ LỜI 10 CÂU HỎI PHẢN ÁNH (15 ĐIỂM)

Mở file `exercises.md`, thay thế các dòng `> *Câu trả lời của bạn*`:

1. **Câu 1 (Fail fast):**  
   *Ý chính:* Nếu để mặc định `"changeme"`, khi deploy lên cloud mà quên cấu hình biến môi trường, app vẫn chạy bình thường. Kẻ xấu có thể quét và sử dụng API với key mặc định đó, gây tốn chi phí và lộ dịch vụ. Không có mặc định buộc app crash ngay lúc khởi động (`ValidationError`), giúp kỹ sư phát hiện ngay trên console deploy.
2. **Câu 2 (Log cho máy đọc):**  
   *Ý chính:* Dán dòng log JSON thu được từ `log_event`. Hai việc làm được: (1) Thiết lập metric lọc và cảnh báo tự động khi tổng `cost_usd` vượt ngưỡng trong 1 giờ; (2) Truy vấn tổng hợp (aggregation) xem user nào gửi nhiều request nhất hoặc tính latency trung bình qua công cụ phân tích log (CloudWatch, Datadog).
3. **Câu 3 (Kích thước image):**  
   *Ý chính:* Single-stage ~1GB, Multi-stage ~200MB. Phần dung lượng chênh lệch là toàn bộ các công cụ build (`build-essential`, gcc, compiler toolchain, pip cache, header files C/C++), những thứ chỉ cần lúc cài package chứ không cần lúc chạy.
4. **Câu 4 (Thứ tự lệnh trong Dockerfile):**  
   *Ý chính:* Khi sửa code `app/main.py`, các layer từ `COPY requirements.txt` và `RUN pip install` được lấy lại từ cache. Chỉ layer `COPY app` trở đi mới build lại. Nếu để `COPY . .` trước, mỗi lần sửa 1 dòng code, cache bị hủy và Docker phải tải/cài lại toàn bộ dependency từ đầu.
5. **Câu 5 (Không chạy bằng root):**  
   *Ý chính:* Nếu app chạy root và dính lỗ hổng Remote Code Execution (RCE) hoặc container breakout, kẻ tấn công chiếm quyền root trong container. Quyền root đó có UID=0 trùng với root trên Linux kernel của host, giúp kẻ tấn công đọc/ghi file hệ thống máy host. Lệnh `USER appuser` (UID=10001) cắt đứt chuỗi vì tiến trình chỉ có quyền user thường, không thể can thiệp tài nguyên hệ thống.
6. **Câu 6 (Cửa sổ trượt vs phút đồng hồ):**  
   *Ý chính:* Với hạn mức 10 req/phút theo phút đồng hồ, user có thể gửi 10 request lúc 10:00:59 và gửi tiếp 10 request lúc 10:01:00. Như vậy có tổng cộng 20 request trong 2 giây liên tiếp mà vẫn không bị chặn. Cửa sổ trượt (Sliding Window) 60 giây ngăn chặn được điều này vì nó tính đúng quãng 60s trôi ngược từ thời điểm hiện tại.
7. **Câu 7 (Rate limit vs Cost guard):**  
   *Ý chính:* Rate limit giới hạn *tần suất/số lượng request* (chống DDoS, nghẽn mạng). Cost guard giới hạn *tổng chi phí tiền token* (chống cháy túi tiền). Ví dụ 1: User chỉ gửi 1 request trong 10 phút (rate limit cho qua) nhưng prompt dài 500k token làm tốn 10 USD (cost guard chặn). Ví dụ 2: User gửi 20 câu "hi" trong 5 giây (cost guard cho qua vì tốn 0.0001 USD nhưng rate limit chặn 429).
8. **Câu 8 (/health khác /ready):**  
   *Ý chính:* Khi Redis mất kết nối 30s: (1) Nếu gộp làm một, `/health` báo lỗi 503; (2) Orchestrator hiểu nhầm container bị treo (deadlock) nên kill và restart cả 3 container; (3) Cả 3 container khởi động lại nhưng Redis vẫn chưa có nên lại bị restart liên tục (crash loop); (4) Hệ thống tê liệt hoàn toàn. Nếu tách riêng: `/health` vẫn 200 (process sống), chỉ `/ready` trả 503 để Load Balancer tạm ngắt traffic, khi Redis hồi phục cụm container sẵn sàng phục vụ ngay lập tức.
9. **Câu 9 (Chuyển tiếp signal shutdown):**  
   *Ý chính:* Nếu không gọi `previous(signum, frame)` của uvicorn, uvicorn sẽ không bao giờ nhận được tín hiệu tắt và tiếp tục chạy mãi dù cờ `shutting_down` đã bật. Orchestrator chờ hết thời gian grace period (10-30s) sẽ gửi `SIGKILL` cưỡng chế, khiến các request đang xử lý dở bị đứt gãy và user nhận lỗi 502 Bad Gateway.
10. **Câu 10 (Stateless service):**  
    *Ý chính:* Nếu lưu lịch sử trong RAM của container, request 1 user hỏi vào container A, request 2 load balancer điều hướng vào container B. Container B không có RAM của A nên không nhớ ngữ cảnh trước đó. Đưa lịch sử ra Redis tập trung giúp mọi container đều truy xuất được cùng một dữ liệu người dùng, cho phép scale ngang bao nhiêu container tùy ý.

---

## 9. BONUS — CI/CD VỚI GITHUB ACTIONS (+10 ĐIỂM)

Tạo file `.github/workflows/ci.yml`:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run unit & integration tests
        env:
          AGENT_API_KEY: test-key-ci-cd-secure-token-12345
          REDIS_URL: redis://localhost:6379/0
        run: |
          pytest tests/ -v -m "not docker and not deploy"

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: |
          docker build -t day12-agent:prod .

  deploy:
    needs: [test, build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy placeholder step
        run: |
          echo "Test passed and image built successfully. Ready for deployment."
```

Kiểm tra test bonus:
```bash
pytest tests/test_bonus_cicd.py -v
```

---

## 10. CHECKLIST TỔNG KẾT TRƯỚC KHI NỘP BÀI

Chạy lệnh chấm điểm tự động từ thư mục gốc repo:
```bash
python grade.py
```

### Tiêu chí đạt điểm tối đa:
- [x] CP1: `pytest tests/test_cp1.py -v` -> 13/13 passed (15đ)
- [x] CP2: `pytest tests/test_cp2.py -v` -> passed (15đ)
- [x] CP3: `pytest tests/test_cp3.py -v` -> 22/22 passed (20đ)
- [x] CP4: `pytest tests/test_cp4.py -v` -> 16/16 passed (20đ)
- [x] CP5: `pytest tests/test_cp5.py -v` -> passed (15đ)
- [x] `exercises.md`: Đã trả lời đủ 10 câu (không còn placeholder) (15đ)
- [x] Bonus (tùy chọn): `pytest tests/test_bonus_cicd.py -v` (+10đ)
- [x] Bảo mật: Kiểm tra không commit file `.env` hoặc API key thật:
  ```bash
  git status
  git ls-files | grep -E '(^|/)\.env$|\.(pem|key)$'
  ```
