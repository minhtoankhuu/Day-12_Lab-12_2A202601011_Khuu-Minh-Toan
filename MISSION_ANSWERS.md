# Day 12 Lab - Mission Answers

> **Student Name:** Khưu Minh Toàn  
> **Student ID:** 2A202601011  
> **Date:** 2026-06-12

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `develop/app.py`

1. **Hardcoded API key** (dòng 17): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` — nếu push lên GitHub, key bị lộ ngay
2. **Hardcoded DATABASE_URL** (dòng 18): `postgresql://admin:password123@localhost:5432/mydb` — password DB bị lộ trong source code
3. **Dùng `print()` thay vì logging** (dòng 33-35): log ra cả secret key, không có log level, không thể parse trong production log aggregator
4. **Không có `/health` endpoint**: Cloud platform (Railway/Render/K8s) không biết container có còn sống không để restart
5. **Port cứng `8000`** (dòng 53): Railway/Render inject `PORT` qua env var; app không lắng nghe đúng port
6. **`host="localhost"`** (dòng 52): Trong container, `localhost` chỉ bind loopback — phải dùng `0.0.0.0` để nhận traffic từ bên ngoài container

### Exercise 1.2: Kết quả chạy basic version

```
GET  /          → {"message":"Hello! Agent is running on my machine :)"}
POST /ask       → {"answer":"Agent đang hoạt động tốt! (mock response)"}
GET  /health    → 404 Not Found   ← không tồn tại!
```

Server log lộ secret:
```
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
```

### Exercise 1.3: Comparison table

| Feature | Develop (sai) | Production (đúng) | Tại sao quan trọng? |
|---------|---------------|-------------------|---------------------|
| Config | Hardcode trong code | Đọc từ env vars (`os.getenv`) | Bảo mật secret, dễ thay đổi giữa môi trường |
| Secrets | `OPENAI_API_KEY = "sk-..."` trong code | Không có secret nào trong code | Tránh lộ key khi push GitHub |
| Health check | Không có (404) | `/health` trả `{"status":"ok","uptime_seconds":...}` | Platform biết khi nào restart container |
| Readiness check | Không có (404) | `/ready` trả `{"ready":true}` | Load balancer không route traffic khi app chưa sẵn sàng |
| Logging | `print("[DEBUG] Using key: ...")`| JSON structured logging, không log secret | Dễ parse, có log level, không lộ thông tin nhạy cảm |
| Shutdown | Đột ngột | `handle_sigterm()` + lifespan cleanup | Requests đang xử lý được hoàn thành trước khi tắt |
| Host | `"localhost"` | `"0.0.0.0"` | Container cần `0.0.0.0` để nhận kết nối từ bên ngoài |
| Port | Cứng `8000` | `int(os.getenv("PORT", "8000"))` | Cloud platform inject PORT tự động |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile cơ bản (develop)

1. **Base image là gì?** `python:3.11` — full Python distribution (~1GB)
2. **Working directory là gì?** `/app`
3. **Tại sao COPY requirements.txt trước?** Docker layer cache — requirements ít thay đổi hơn code. Nếu copy requirements trước, bước `pip install` được cache, chỉ re-run khi requirements thay đổi. Giúp rebuild nhanh hơn rất nhiều.
4. **CMD vs ENTRYPOINT?**
   - `CMD` = lệnh mặc định, có thể override khi `docker run my-image <other-cmd>`
   - `ENTRYPOINT` = lệnh cố định, không thể bị override (chỉ thêm arguments). Dùng ENTRYPOINT khi app là "executable" chính; CMD khi muốn cho phép override.

### Exercise 2.2: Build và chạy develop image

```bash
docker build -f 02-docker/develop/Dockerfile -t agent-develop .
docker run -p 8000:8000 agent-develop
```

**Image size của develop:** 1.66 GB

### Exercise 2.3: Multi-stage build (production)

- **Stage 1 (builder):** Cài `gcc`, `libpq-dev`, build tools, rồi `pip install --user`. Dùng `python:3.11-slim` làm base.
- **Stage 2 (runtime):** Bắt đầu từ `python:3.11-slim` sạch, chỉ copy `/root/.local` (packages đã cài) từ builder. Không có pip, gcc, hay build artifacts.
- **Tại sao image nhỏ hơn?** Stage 2 không chứa build tools, pip cache, hay metadata từ quá trình compile. Chỉ giữ code + runtime dependencies.

**So sánh image size:**

| Image | Size | Ghi chú |
|-------|------|---------|
| `agent-develop` (single-stage) | 1.66 GB | `python:3.11` full image |
| `agent-production` (multi-stage) | 236 MB | `python:3.11-slim` + chỉ runtime deps |
| **Chênh lệch** | **~86% nhỏ hơn** | Production nhỏ hơn ~7 lần |

Ngoài ra, production image chạy với **non-root user** (`appuser`) — security best practice.

### Exercise 2.4: Docker Compose stack

Services trong `docker-compose.yml`:
- **agent** — FastAPI AI agent (có thể scale với `--scale agent=N`)
- **redis** — Cache cho session và rate limiting
- **qdrant** — Vector database cho RAG
- **nginx** — Reverse proxy, load balancer (expose port 80/443)

Communication: Tất cả service giao tiếp qua **internal bridge network** (`internal`). Nginx nhận request từ bên ngoài, forward vào `agent` (không expose port agent trực tiếp).

```
Client → Nginx:80 → agent:8000 (internal)
                  → redis:6379 (internal)
                  → qdrant:6333 (internal)
```

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- URL: https://lab12deplou-production.up.railway.app
- Health check: `curl https://lab12deplou-production.up.railway.app/health` 
- Screenshot: ![Demo](image.png)

### Exercise 3.2: Render vs Railway config

**So sánh sự khác biệt:**

| Tiêu chí | Railway (`railway.toml`) | Render (`render.yaml`) |
|----------|---------------------------|-------------------------|
| **Định dạng** | TOML | YAML |
| **Phạm vi** | Thường cấu hình cho 1 service cụ thể (build, deploy). | Cấu hình toàn bộ hạ tầng (Blueprint), có thể khai báo nhiều services (Web, Redis) cùng lúc. |
| **Build** | Dùng `builder = "NIXPACKS"` để tự động detect và build (không cần định nghĩa build command). | Cần định nghĩa rõ `runtime: python` và `buildCommand: pip install...`. |
| **Env Vars** | Gen/set thông qua Railway CLI hoặc Dashboard, ít khi hardcode biến môi trường vào `.toml`. | Cho phép cấu hình linh hoạt trong file: hardcode (`value`), không đồng bộ (`sync: false`) để bảo mật, hoặc tự sinh (`generateValue: true`). |

Nhìn chung, `render.yaml` thích hợp cho việc định nghĩa Infrastructure as Code (IaC) chuẩn xác với nhiều services phụ thuộc nhau, trong khi `railway.toml` gọn nhẹ và tự động hoá nhiều hơn ở khâu build.

---

## Part 4: API Security

### Exercise 4.1: API Key authentication

API key được check tại hàm `verify_api_key()` trong `develop/app.py` (dùng FastAPI `Security` dependency):
- API key đọc từ env var `AGENT_API_KEY` (không hardcode)
- Nếu thiếu header → **401 Unauthorized**
- Nếu sai key → **403 Forbidden**
- Để rotate key: đổi giá trị `AGENT_API_KEY` env var, restart service

**Test kết quả:**

```bash
# Không có key → 401
curl -X POST http://localhost:8001/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
# → {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}  HTTP 401

# Sai key → 403
curl -H "X-API-Key: wrong-key" -X POST http://localhost:8001/ask ...
# → {"detail":"Invalid API key."}  HTTP 403

# Đúng key → 200
curl -H "X-API-Key: secret-key-123" -X POST http://localhost:8001/ask ...
# → {"question":"Hello","answer":"..."}  HTTP 200
```

### Exercise 4.2: JWT authentication

**JWT Flow:**
1. `POST /auth/token` với `{username, password}` → server trả JWT token (hết hạn 60 phút)
2. Client gửi token trong header: `Authorization: Bearer <token>`
3. Server decode token, verify chữ ký bằng `SECRET_KEY` → extract `{username, role}`
4. Không cần database lookup mỗi request → stateless

**Test kết quả:**
```bash
# Lấy token
TOKEN=$(curl -X POST http://localhost:8002/auth/token \
  -d '{"username":"student","password":"demo123"}' | jq -r .access_token)

# Không có token → 401
curl -X POST http://localhost:8002/ask ...
# → {"detail":"Authentication required..."}  HTTP 401

# Có token → 200
curl -H "Authorization: Bearer $TOKEN" -X POST http://localhost:8002/ask ...
# → {"question":"...","answer":"...","usage":{"requests_remaining":9}}  HTTP 200
```

### Exercise 4.3: Rate limiting

**Algorithm: Sliding Window Counter**
- Mỗi user có 1 deque lưu timestamps của các request
- Mỗi lần check: loại bỏ timestamps cũ (> 60 giây), đếm còn lại
- Nếu `count >= max_requests` → raise 429 với `Retry-After` header

**Limits:**
- `user` role: 10 req/phút
- `admin` role: 100 req/phút
- Bypass cho admin: dùng account `teacher/teach456`

**Test kết quả:**
```
Request 1-9:  HTTP 200 — lần lượt còn 8, 7, 6... req
Request 10:   HTTP 429 → Rate limit exceeded
Request 11+:  HTTP 429 → Rate limit exceeded
```

### Exercise 4.4: Cost guard implementation

**Cách hoạt động:**
- Mỗi request ước tính số tokens (input + output)
- Cộng dồn chi phí theo công thức: `cost = tokens/1000 * price_per_1k`
- Budget $1/ngày per user, reset midnight UTC
- Budget $10/ngày toàn hệ thống (global guard)
- 80% budget → log warning
- 100% budget → 402 Payment Required
- Global budget hết → 503 Service Unavailable

**Implementation (production/cost_guard.py):**
```python
def check_budget(user_id: str) -> None:
    record = self._get_record(user_id)      # Lấy record ngày hôm nay
    if self._global_cost >= global_budget:  # Check global budget
        raise HTTPException(503, "Service unavailable")
    if record.total_cost_usd >= self.daily_budget_usd:  # Check per-user
        raise HTTPException(402, {"error": "Daily budget exceeded", ...})
```

**Giá tham khảo (GPT-4o-mini):** $0.15/1M input tokens, $0.60/1M output tokens

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

**Liveness probe (`/health`)** — "Agent còn sống không?"
- Trả về 200 với uptime, version, memory check
- Cloud platform gọi định kỳ, nếu non-200 → restart container

**Readiness probe (`/ready`)** — "Agent sẵn sàng nhận request chưa?"
- Trả về 503 khi đang startup hoặc đang shutdown
- Load balancer dùng để quyết định có route traffic không

```bash
curl http://localhost:8003/health
# → {"status":"ok","uptime_seconds":1.7,"checks":{"memory":{"used_percent":76.4}}}

curl http://localhost:8003/ready
# → {"ready":true,"in_flight_requests":1}
```

### Exercise 5.2: Graceful shutdown

Có 2 cơ chế:
1. **`lifespan` context manager** — chờ `_in_flight_requests == 0` (tối đa 30 giây) trước khi exit
2. **`signal.signal(SIGTERM)`** — log khi nhận signal, uvicorn tự handle

```python
@asynccontextmanager
async def lifespan(app):
    _is_ready = True
    yield
    _is_ready = False
    while _in_flight_requests > 0 and elapsed < 30:
        time.sleep(1)  # chờ request hiện tại xong
```

### Exercise 5.3: Stateless design

**Anti-pattern (stateful — không scale được):**
```python
conversation_history = {}  # ❌ chỉ có trong instance này

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])  # ❌ instance khác không thấy
```

**Correct (stateless với Redis):**
```python
def load_session(session_id):
    data = redis.get(f"session:{session_id}")   # ✅ bất kỳ instance nào đều đọc được
    return json.loads(data) if data else {}
```

Khi có Redis: `storage: "redis"`. Khi không có Redis: fallback `storage: "in-memory"` (cảnh báo không scale được).

### Exercise 5.4: Load balancing

**Kết quả test 6 requests liên tiếp:**
```
Request 1 → served by: instance-d21f4d
Request 2 → served by: instance-fc8921
Request 3 → served by: instance-c7ee6b
Request 4 → served by: instance-d21f4d
Request 5 → served by: instance-fc8921
Request 6 → served by: instance-c7ee6b
```

Nginx dùng **round-robin** để phân tán requests đều sang 3 instance. Vì state lưu trong Redis, mọi instance đều xử lý được mọi request.

**Chạy với scale:**
```bash
docker compose up --scale agent=3
```

Stack: Nginx (port 8080) → 3 agent instances → Redis

### Exercise 5.5: Stateless test

Session data tồn tại xuyên suốt nhiều request, bất kể instance nào serve:
```
Turn 1 (instance-d21f4d): "Hello, my name is Toan"
Turn 2 (instance-fc8921): "What is Docker?"
History: 4 messages (user + assistant × 2)
```

---

## Part 6: Final Project

### Architecture

```
Client → Docker Container (agent-final:247MB)
           ├── FastAPI + uvicorn (2 workers)
           ├── API Key Authentication
           ├── Rate Limiting (sliding window, 20 req/min)
           ├── Cost Guard ($5/day budget)
           ├── /health + /ready probes
           ├── JSON structured logging
           └── Graceful SIGTERM handler
         → Redis (session, rate limit shared state)
```

### Production Readiness Check: 20/20 ✅

```
📁 Required Files:      6/6 ✅
🔒 Security:            2/2 ✅
🌐 API Endpoints:       6/6 ✅
🐳 Docker:              6/6 ✅
Result: 20/20 (100%) — 🎉 PRODUCTION READY!
```

### Final Validation Test Results

```bash
# 1. /health → {"status":"ok","uptime_seconds":22.7,"environment":"staging"}
# 2. /ready  → {"ready":true}
# 3. No API key → HTTP 401 "Invalid or missing API key"
# 4. Correct key → HTTP 200 {"answer":"...","model":"gpt-4o-mini","timestamp":"..."}
# 5. Rate limit 20/min → 20 × HTTP 200, sau đó HTTP 429
# 6. /metrics → {"total_requests":21,"daily_cost_usd":0.0003,"budget_used_pct":0.0}
```

### Image size: 247 MB (< 500 MB yêu cầu) ✅

### Bug fixed trong quá trình làm:
`MutableHeaders.pop()` không tồn tại trong FastAPI/Starlette → fix thành:
```python
# ❌ response.headers.pop("server", None)
# ✅
if "server" in response.headers:
    del response.headers["server"]
```

### Deploy (Railway)

```bash
cd 03-cloud-deployment/railway
railway login
railway init
railway variables set AGENT_API_KEY=<your-key>
railway up
```

Config trong `railway.toml`:
- `builder = "NIXPACKS"` — auto-detect Python
- `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"`
- `healthcheckPath = "/health"` — Railway restart khi fail
