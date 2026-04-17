# Day 12 Lab — Mission Answers

> **Student Name:** Phạm Đăng Phong  
> **Student ID:** 2A202600254  
> **Date :** 17/04/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

Tìm được **5 vấn đề chính**:

1. **API key hardcode trong code** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Nếu push lên GitHub → credentials bị lộ ngay lập tức.

2. **Debug mode bật cứng** — `DEBUG = True` và `reload=True` trong uvicorn. Trong production, reload mode làm chậm performance và có thể expose thông tin nhạy cảm.

3. **Dùng `print()` thay vì proper logging** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` vừa log ra secret, vừa không có timestamp, level, hay structured format để parse.

4. **Không có health check endpoint** — Platform cloud (Railway, Render) cần `/health` để biết lúc nào container crash và cần restart. Nếu không có → outage không được phát hiện.

5. **Port cứng và host localhost** — `host="localhost"` (chỉ bind internal), `port=8000` (cứng). Trên cloud, PORT được inject qua env var và app phải bind `0.0.0.0` để nhận traffic từ bên ngoài container.

**Bonus vấn đề thứ 6:** Không có graceful shutdown — khi container nhận SIGTERM, app tắt đột ngột, các request đang xử lý bị drop.

---

### Exercise 1.2: Chạy basic version

**Quan sát: Nó chạy! Nhưng có production-ready không?**
→ **KHÔNG.** Mặc dù code vẫn gọi được AI cục bộ, nhưng nó hoàn toàn không đạt chuẩn chạy thật (production-ready) vì: chứa mã bảo mật cứng (lộ Key), thiết lập Debugging chưa tối ưu dẫn đến rò rỉ log hệ thống, không có cơ chế quản lý sức khỏe App (Health Check) và chỉ khoá cố định một Port (`8000`) thay vì dùng cổng đám mây cấp phát linh hoạt.

---

### Exercise 1.3: Comparison table

| Feature | Basic (develop) | Advanced (production) | Tại sao quan trọng? |
|---------|-----------------|----------------------|---------------------|
| **Config** | Hardcode trong code | Env vars (`os.getenv`) | Tránh lộ secret; dễ thay đổi per-environment |
| **Secrets** | `OPENAI_API_KEY = "sk-..."` trong code | Đọc từ `.env` / platform secrets | Security: không commit creds lên Git |
| **Health check** | ❌ Không có | ✅ `GET /health` trả 200 + uptime | Platform tự động restart khi agent crash |
| **Readiness probe** | ❌ Không có | ✅ `GET /ready` trả 503 khi đang boot | Load balancer không route traffic sớm |
| **Logging** | `print()` debug | Structured JSON logging | Dễ parse bởi log aggregator (Datadog/Loki) |
| **Shutdown** | Đột ngột (Ctrl+C) | Graceful — xử lý SIGTERM | Request đang chạy không bị drop |
| **Port binding** | `localhost:8000` cứng | `0.0.0.0:$PORT` từ env | Chạy được trong Docker container & cloud |
| **Debug mode** | `reload=True` luôn luôn | `reload=settings.debug` — chỉ khi cần | Performance + security |
| **CORS** | ❌ Không có | ✅ Cấu hình qua env | Kiểm soát ai được gọi API từ browser |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)

1. **Base image là gì?** → `python:3.11` — full Python distribution (~1 GB). Bao gồm toàn bộ pip, build tools, v.v.

2. **Working directory là gì?** → `/app` — tất cả file được copy vào đây và lệnh chạy từ đây.

3. **Tại sao COPY requirements.txt trước?** → Docker layer caching. Nếu requirements không thay đổi, Docker reuse layer `pip install` đã cache → build nhanh hơn nhiều. Nếu copy toàn bộ code trước, mỗi lần thay 1 dòng code sẽ install lại toàn bộ dependencies.

4. **CMD vs ENTRYPOINT khác nhau thế nào?**
   - `ENTRYPOINT` — lệnh cố định, không override được dễ dàng khi chạy container
   - `CMD` — lệnh mặc định, có thể override bằng args khi `docker run`. Thường dùng `ENTRYPOINT ["python"]` + `CMD ["app.py"]` để linh hoạt hơn.
### Exercise 2.2: Build và run

**Quan sát: Image size là bao nhiêu?**
→ Khi kiểm tra bằng lệnh `docker images my-agent:develop`, dung lượng Image gốc (Single-stage Build) rơi vào khoảng **~1.01 GB**. Mức dung lượng khổng lồ này xảy ra vì nó cài đặt full phiên bản Python (với đầy đủ các package compile, build tools, linux packages) dù không cần thiết khi chạy thực tế. 

---
### Exercise 2.3: Image size comparison

| Image | Size | Ghi chú |
|-------|------|---------|
| `my-agent:develop` (single-stage) | ~1.0 GB | Base `python:3.11` đầy đủ |
| `my-agent:production` (multi-stage) | ~200-250 MB | Base `python:3.11-slim`, không có build tools |
| **Difference** | **~75-80%** | Multi-stage loại bỏ gcc, apt cache, và build artifacts |

**Tại sao multi-stage nhỏ hơn?**
- Stage 1 (builder): Dùng `python:3.11-slim`, cài gcc để compile binary packages
- Stage 2 (runtime): Copy chỉ wheels đã compile từ builder, không copy gcc/build tools
- Kết quả: Image runtime không chứa compiler, source headers, hay apt cache

### Exercise 2.4: Docker Compose Stack Architecture

```
                    ┌─────────────────────┐
    HTTP :8000      │   agent container   │
────────────────>   │  (FastAPI + uvicorn) │
                    └──────────┬──────────┘
                               │ redis://redis:6379
                               ▼
                    ┌─────────────────────┐
                    │   redis container   │
                    │  (redis:7-alpine)   │
                    │   maxmem: 128MB     │
                    └─────────────────────┘
```

Services được start:
- **agent**: FastAPI app, depends on redis (health check pass trước)
- **redis**: In-memory store cho rate limiting, conversation history, cost tracking

Communication: Qua Docker internal network, agent kết nối redis qua hostname `redis:6379`.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- **Platform:** Railway
- **URL:** Xem `DEPLOYMENT.md`
- **Railway config:** `railway.toml` — dùng Dockerfile, healthcheck `/health`, restart on failure

### Exercise 3.2: Deploy Render

**So sánh config files (render.yaml vs railway.toml):**

| Tính năng | `railway.toml` | `render.yaml` |
|-----------|---------------|--------------|
| Build | `builder = "DOCKERFILE"` | `type: web` + Dockerfile |
| Start command | Cấu hình trong toml | `startCommand` trong yaml |
| Health check | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| Env vars | Set qua `railway variables set` | Set trong dashboard hoặc yaml |
| Auto-deploy | Khi push lên Git branch | Khi push lên Git branch |

### Exercise 3.3: (Optional) GCP Cloud Run

**Nhiệm vụ: Hiểu CI/CD pipeline qua file `cloudbuild.yaml` và `service.yaml`**
→ **Giải thích cơ chế:**
- `cloudbuild.yaml`: Cấu hình hệ thống CI/CD của Google. Khi phát hiện code mới trên Git, hệ thống sẽ tự động chạy lệnh Docker build, dán Tag phiên bản v1, v2... và đẩy Image đó lên kho chứa (Registry).
- `service.yaml`: File triển khai tự động (Tương tự cấu hình của Kubernetes). Nó dùng để định nghĩa cho Cloud Run biết rằng: Dùng image nào, cấp bao nhiêu RAM/CPU, và cấu hình Scale thế nào (vd: scale từ 0 đến 10 instances).

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication

**API key được check ở đâu?**
- Trong `verify_api_key()` function, dùng `APIKeyHeader(name="X-API-Key")` — FastAPI tự extract header
- So sánh với `settings.agent_api_key` (đọc từ env var `AGENT_API_KEY`)

**Điều gì xảy ra nếu sai key?**
- Raise `HTTPException(status_code=401, detail="Invalid or missing API key")`
- Client nhận HTTP 401 Unauthorized

**Làm sao rotate key?**
- Thay giá trị `AGENT_API_KEY` trong env vars trên Railway/Render
- Restart service — không cần deploy lại code

### Exercise 4.2: JWT authentication (Advanced)

**Nhiệm vụ: Phân tích luồng (flow) xác thực bằng JWT (JSON Web Token)**
→ **Mô tả Flow:** 
1. Thay vì dùng thẳng chuỗi API Key cứng, Client gửi Request POST lên endpoint `/token` kèm theo `username` và `password`.
2. Hệ thống kiểm tra thông tin. Nếu chính xác, nó sinh ra một chuỗi JWT Token có chứa Payload (thông tin auth, hạn dùng expire time) và cấp cho Client.
3. Ở các lần gọi endpoint `/ask` tiếp theo, Client bắt buộc kẹp Token này vào thuộc tính Header: `Authorization: Bearer <token>`.
4. FastAPI Middleware sẽ tự động kiểm tra và giải mã Token. Nếu ký hiệu hợp lệ và chưa hết hạn, Request mói được đi tiếp tới Agent.

### Exercise 4.1-4.3: Test results

```bash
# Test 1: Không có key → 401
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Response: {"detail":"Invalid or missing API key..."}  HTTP 401

# Test 2: Có key đúng → 200
curl -X POST http://localhost:8000/ask \
  -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Response: {"question":"Hello","answer":"...","model":"gpt-4o-mini","timestamp":"..."}  HTTP 200

# Test 3: Rate limit (gọi > 20 lần/phút → 429)
# Response sau lần 21: {"detail":"Rate limit exceeded: 20 req/min"} HTTP 429
# Header: Retry-After: 60
```

### Exercise 4.3: Rate limiting analysis

- **Algorithm:** Sliding window — dùng `deque` lưu timestamp của các request trong 60 giây gần nhất
- **Limit:** 20 requests/minute (cấu hình qua `RATE_LIMIT_PER_MINUTE` env var)
- **Bypass cho admin:** Không implement bypass; tất cả key đều áp dụng limit như nhau. Có thể mở rộng bằng cách kiểm tra prefix key để phân loại admin vs regular user.

### Exercise 4.4: Cost guard implementation

```python
# Trong app/main.py — simplified in-memory implementation
_daily_cost = 0.0
_cost_reset_day = time.strftime("%Y-%m-%d")

def check_and_record_cost(input_tokens: int, output_tokens: int):
    global _daily_cost, _cost_reset_day
    today = time.strftime("%Y-%m-%d")
    if today != _cost_reset_day:          # Reset hàng ngày
        _daily_cost = 0.0
        _cost_reset_day = today
    if _daily_cost >= settings.daily_budget_usd:
        raise HTTPException(503, "Daily budget exhausted. Try tomorrow.")
    # Tính cost theo GPT-4o-mini pricing
    cost = (input_tokens / 1000) * 0.00015 + (output_tokens / 1000) * 0.0006
    _daily_cost += cost
```

**Approach:** Estimate số tokens từ word count (×2), track trong memory, reset hàng ngày. Trong production nên dùng Redis để persist qua restart và share giữa multiple instances.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks implementation

```python
@app.get("/health")
def health():
    """Liveness probe — container còn sống không?"""
    return {
        "status": "ok",
        "version": settings.app_version,
        "uptime_seconds": round(time.time() - START_TIME, 1),
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }

@app.get("/ready")
def ready():
    """Readiness probe — sẵn sàng nhận traffic không?"""
    if not _is_ready:
        raise HTTPException(503, "Not ready")
    return {"ready": True}
```

**Tại sao cần 2 endpoints khác nhau?**
- `/health` (Liveness): "Container có crash không?" → Restart nếu fail
- `/ready` (Readiness): "App có ready nhận traffic không?" → Stop routing nếu fail. App có thể alive nhưng chưa load xong model/DB connection.

### Exercise 5.2: Graceful shutdown

```python
import signal

def _handle_signal(signum, _frame):
    logger.info(json.dumps({"event": "signal", "signum": signum}))
    # uvicorn timeout_graceful_shutdown=30 sẽ cho 30s để hoàn thành requests

signal.signal(signal.SIGTERM, _handle_signal)

# Trong uvicorn.run():
uvicorn.run(..., timeout_graceful_shutdown=30)
```

**Test graceful shutdown:** Gửi request dài, sau đó kill -TERM → request hoàn thành, sau đó app tắt.

### Exercise 5.3: Stateless design

**Anti-pattern (in-memory state):**
```python
# ❌ Mỗi instance có memory riêng → data không consistent khi scale
conversation_history = {}  # mất khi restart hoặc route sang instance khác
```

**Correct (Redis state):**
```python
# ✅ Shared state — tất cả instances đọc cùng một nơi
history = r.lrange(f"history:{user_id}", 0, -1)
```

**Tại sao quan trọng khi scale:** Khi chạy `--scale agent=3`, 3 instances có 3 memory độc lập. Request 1 vào instance A, request 2 vào instance B → instance B không có history từ instance A → conversation bị mất. Redis đóng vai trò shared memory cho toàn bộ cluster.

### Exercise 5.4: Load balancing test results

```bash
# Khởi động 3 instances
docker compose up --scale agent=3

# Test: 10 requests được phân tán
for i in {1..10}; do
  curl http://localhost/ask -X POST \
    -H "Content-Type: application/json" \
    -d "{\"question\": \"Request $i\"}"
done

# Xem logs: requests được round-robin qua agent_1, agent_2, agent_3
docker compose logs agent
```

### Exercise 5.5: Stateless test

Chạy `python check_production_ready.py` → Kiểm tra tất cả checklist items và báo cáo những gì còn thiếu.

---

## Part 6: Final Project

### Architecture

```
┌──────────┐     X-API-Key header
│ Client   │ ──────────────────────────────────> ┌──────────────────────────┐
└──────────┘                                     │  Production AI Agent     │
                                                 │  FastAPI + uvicorn       │
                                                 │                          │
                                                 │  Layers:                 │
                                                 │  1. verify_api_key()     │
                                                 │  2. check_rate_limit()   │
                                                 │  3. check_budget()       │
                                                 │  4. llm_ask()            │
                                                 └──────────┬───────────────┘
                                                            │ (optional)
                                                            ▼
                                                  ┌─────────────────┐
                                                  │     Redis       │
                                                  │  rate windows   │
                                                  │  conversation   │
                                                  └─────────────────┘
```

### Production-ready checklist

- ✅ Multi-stage Dockerfile (image < 500 MB)
- ✅ Config từ environment variables (12-factor)
- ✅ API key authentication (`X-API-Key` header)
- ✅ Rate limiting: 20 req/min (sliding window algorithm)
- ✅ Cost guard: $5/day budget với auto-reset
- ✅ Health check: `GET /health` → 200 OK với uptime
- ✅ Readiness check: `GET /ready` → 503 khi đang boot
- ✅ Graceful shutdown: SIGTERM handler + `timeout_graceful_shutdown=30`
- ✅ Stateless design: state trong Redis (khi dùng docker-compose)
- ✅ Structured JSON logging: format `{"ts":"...","lvl":"...","msg":"..."}`
- ✅ Security headers: `X-Content-Type-Options`, `X-Frame-Options`
- ✅ Không có hardcoded secrets — tất cả từ env vars
- ✅ Deploy Railway/Render: xem `DEPLOYMENT.md`
