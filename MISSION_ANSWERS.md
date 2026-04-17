#  Delivery Checklist — Day 12 Lab Submission

> **Student Name:** Vũ Tiến Thành 
> **Student ID:** 2A202600443
> **Date:** 17/04/2026

---

# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. API key hardcode trong code
2. Không có config management
3. Print thay vì proper logging
4. log ra secret key
5. Không có health check endpoint
6. Port cố định — không đọc từ environment
7. debug reload trong production
...

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config  | cứng trong cod     | lấy từ env | Có thể thay đổi dễ từ 1 file env duy nhất, không động đến code           |
| Logging  | print    | Structured JSON logging | dễ parse trong log aggregator (Datadog, Loki...) theo dõi tốt hơn và có cấu trúc hơn|
| Health check  | Không có    | Có  | Dễ check các service không chạy khoanh vùng lỗi và chuyển hướng call service|
| Graceful shutdown  | Không có    | Có  | Xử lý hết dữ liệu trước khi kill đảm bảo không  hỏng hoặc mất dữ liệu đang xử lý|
| Readiness probe  | Không có    | Có  | Đảm bảo Service đã sẵn sàng nhận request trước khi gửi vào để tránh lỗi hoặc quá tải|
| CORS  | Không có    | Có  | config chỉ cho phép các origins được cấu hình call vào service|
| Lifecycle management  | Không có    | Có  | Quản lý được lifecycle của service dễ hơn và đảm bảo vòng đời không bị crack đột ngột gây lỗi ảnh hưởng đến dữ liệu đang được xử lý|

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: môi trường hệ điều hành hoặc runtime đã được cấu hình sẵn(Node, PHP, Python,…)
2. Working directory: à thư mục mặc định bên trong container mà mọi lệnh sẽ được thực thi tại đó.
3. Tại sao COPY requirements.txt TRƯỚC COPY . ?
- Docker cache theo layers. Khi code thay đổi nhưng requirements không
đổi, Docker tái sử dụng layer pip install, không phải chạy lại.
- Tiết kiệm thời gian build đáng kể.

4. CMD vs ENTRYPOINT khác nhau gì?
- CMD: có thể bị override bởi docker run arguments
- ENTRYPOINT: cố định, arguments thêm vào sau
- VD: ENTRYPOINT ["python"] + CMD ["app.py"] → python app.py

### Exercise 2.3: Image size comparison
- Develop: 1.66 GB
- Production: 236MB
- Difference: giảm ~86.12%

2.4 Architecture Diagram:

┌──────────┐      ┌─────────┐      ┌─────────┐      ┌───────┐
│  Client  │──────│  Nginx  │──────│  Agent  │──────│ Redis │
│          │ :80  │ :80     │:8000 │         │:6379  │       │
└──────────┘      └─────────┘      └─────────┘      └───────┘

Flow: Client → Nginx (port 80) → Agent (port 8000) → Redis (port 6379)
Nginx làm: reverse proxy + load balancing + rate limiting
Redis làm: cache session + rate limit tracking
Agent: FastAPI app nhận /ask requests

Test script (test_stack.sh):
```bash
# Test health
curl http://localhost/health

# Test agent directly
curl -X POST http://localhost/ask \
  -H "Content-Type: application/json" \
  -d '{"user_id":"test","question":"hello"}'

# Test rate limiting (gửi 20 requests)
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost/ask \
    -H "X-API-Key: secret" \
    -H "Content-Type: application/json" \
    -d "{\"user_id\":\"test\",\"question\":\"test $i\"}"
```


## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://demo-lab11-production.up.railway.app
- Screenshot: [\[Link to screenshot in repo\]](https://drive.google.com/file/d/1hdcyEfRzQ_WnEOu0v4_6ZhuOyCcUf4Eh/view?usp=sharing)

### Exercise 3.2 Railway vs Render Config Comparison:

| Aspect          | Railway               | Render                   |
|----------------|----------------------|--------------------------|
| Config file    | railway.toml         | render.yaml              |
| Build command  | Auto-detect hoặc tùy chỉnh | pip install -r...     |
| Start command  | uvicorn --host 0.0.0.0 | uvicorn --host 0.0.0.0   |
| Health check   | /health (builtin)    | healthCheckPath: /health |
| Redis          | Plugin marketplace   | Redis add-on type        |
| Environment    | Variables dashboard  | envVars trong YAML       |
| Auto-deploy    | On git push        | On git push            |

Kết luận: Railway dễ hơn cho MVP, Render tốt hơn cho IaC (Infrastructure as Code).


## Part 4: API Security

### Exercise 4.1-4.3: Test results
- curl.exe -X POST "http://localhost:8000/ask?question=hello" -H "X-API-Key: abc"

>>>{"question":"hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}

- curl.exe -X POST "http://localhost:8000/ask?question=hello"                    
>>> {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

- Invoke-RestMethod -Uri "http://localhost:8000/auth/token" -Method POST -ContentType "application/json" -Body '{"username":"student","password":"demo123"}'

>>>access_token                       
>>>------------                              
>>>eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MTgxODksImV4cCI6MTc3NjQyMTc4...

- Invoke-RestMethod -Uri "http://localhost:8000/ask" -Method POST -Headers @{ "Authorization"="Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MTg2MjIsImV4cCI6MTc3NjQyMjIyMn0.letxFI2PE6UKsyrOygrkhvnGiG28CJ3aIL5IHwWpJ9M" } -ContentType "application/json" -Body '{"question":"what is docker?"}'

>>>question        answer                         usage          
>>>--------        ------                         -----          
>>>what is docker? Container lÃ  cÃ¡ch ÄÃ³ng gÃ³i app Äá» cháº¡y á»i nÆ¡i. Build once, run anywhere! requests_r...

- Invoke-RestMethod -Uri "http://localhost:8000/ask" -Method POST -Headers @{ "Authorization"="Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MTg2MjIsImV4cCI6MTc3NjQyMjIyMn0.letxFI2PE6UKsyrOygrkhvnGiG28CJ3aIL5IHwWpJ9M" } -ContentType "application/json" -Body '{"question":"what is docker?"}'

>>> {"detail":"Invalid token."}


(Curl test theo hướng dẫn trên vscode sai vì vậy đổi sang các lệnh trên)
#### 4.2: JWT

- JWT Flow:
1. POST /auth/token với username/password → nhận JWT access_token
2. Gửi JWT trong header: Authorization: Bearer <token>
3. Server verify signature (HS256) + expiry → extract user info
4. Token hết hạn sau 60 phút → phải login lại

JWT Payload: {"sub": "student", "role": "user", "iat": ..., "exp": ...}

So sánh API Key vs JWT:
| Aspect       | API Key        | JWT                    |
|-------------|----------------|------------------------|
| Stateless  | Không           |  Có                  |
| Expiry      | Không có       |  60 phút              |
| Role-based  | Không           |  Có (user/admin)      |
| Best for    | Internal API   | Public/multi-user API  |

#### 4.3 Rate Limiting

Algorithm: Sliding Window Counter
- User (student): 10 requests / 60 giây
- Admin (teacher): 100 requests / 60 giây
- Vượt quá → HTTP 429 + header Retry-After

Test results:
- Request 1-10:   200 OK
- Request 11:     429 Too Many Requests
  {"detail":{"error":"Rate limit exceeded","retry_after_seconds":58}}

Response headers khi bị limit:
  X-RateLimit-Limit: 10
  X-RateLimit-Remaining: 0
  Retry-After: 58

Giải thích: Sliding window đếm timestamps của 10 request gần nhất.
Mỗi request mới loại bỏ timestamp cũ (>60s). Vượt 10 → block
### Exercise 4.4: Cost guard implementation
**Approach:**
Dùng CostGuard class để track chi phí LLM theo user mỗi ngày.

**Algorithm:**
1. Mỗi request → estimate input_tokens + output_tokens
2. Tính cost = (input/1000 × $0.00015) + (output/1000 × $0.0006)
3. check_budget() chạy TRƯỚC khi gọi LLM
4. record_usage() chạy SAU khi có response

**Cấu hình:**
- Per-user daily budget: $1.00/ngày
- Global daily budget: $10.00/ngày
- Warning ở 80% budget

**Logic chi tiết:**
```python
# Trước khi gọi LLM:
cost_guard.check_budget(username)
  → 402 Payment Required nếu user vượt $1/ngày
  → 503 nếu global vượt $10/ngày

# Sau khi có response:
usage = cost_guard.record_usage(username, input_tokens, output_tokens)
  → Cập nhật _records dict
  → Trả về usage stats

# Kiểm tra:
GET /me/usage  → Xem budget còn lại của bản thân
GET /admin/stats → Xem tổng chi phí toàn hệ thống
```

- Results:
```python
# Sau ~50 requests (mock tokens):
GET /me/usage
→ {
    "user_id": "student",
    "date": "2026-04-17",
    "requests": 50,
    "cost_usd": 0.012,
    "budget_usd": 1.0,
    "budget_remaining_usd": 0.988,
    "budget_used_pct": 1.2
  }

# Khi vượt $1 budget:
POST /ask
→ 402: {
    "detail": {
      "error": "Daily budget exceeded",
      "used_usd": 1.003,
      "budget_usd": 1.0,
      "resets_at": "midnight UTC"
    }
  }
```

Production recommendation: 

- Lưu UsageRecord vào Redis thay vì in-memory dict
- Sync cost tracking với OpenAI billing API thực tế
- Thêm alert khi global cost đạt 50%, 80%, 100%

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- curl http://localhost:8000/health
>> RawContent        : HTTP/1.1 200 OK
>>Content-Length: 186
>>Content-Type: application/json
>>Date: Fri, 17 Apr 2026 10:56:22 GMT
>>Server: uvicorn
>>
>>{"status":"ok","uptime_seconds":39.4,"version":"1.0.0","environment":"deve...

- curl http://localhost:8000/ready

>>RawContent        : HTTP/1.1 200 OK
>>Content-Length: 37
>>Content-Type: application/json
>>Date: Fri, 17 Apr 2026 10:57:49 GMT
>>Server: uvicorn
>>
>>{"ready":true,"in_flight_requests":1}

### Exercise 5.1: Health & Readiness Checks

/health — Liveness Probe:
- Trả về 200 → container còn sống, không restart
- Kiểm tra: memory, uptime, version, environment
- Cloud platform gọi định kỳ (mỗi 30s)

/ready — Readiness Probe:
- Trả về 200 → ready nhận traffic
- Trả về 503 → đang startup/shutdown, load balancer không route vào
- Trả về: {"ready": true, "in_flight_requests": N}
- in_flight_requests = số request đang xử lý → readiness = True khi = 0

Sự khác biệt:
- /health: "Có còn chạy không?" → Restart nếu fail
- /ready: "Sẵn sàng nhận việc chưa?" → Stop routing nếu 503

### Exercise 5.2: Graceful Shutdown

Flow khi nhận SIGTERM:
1. uvicorn nhận signal → set _is_ready = False
2. /ready trả về 503 → load balancer stop routing
3. Chờ in-flight requests hoàn thành (max 30 giây)
4. Log " Shutdown complete"
5. Container exit cleanly

Nếu không graceful:
- Request đang xử lý bị kill → user nhận lỗi
- Health check fail → platform restart liên tục
- Session data có thể mất

Code chính:
signal.signal(signal.SIGTERM, handle_sigterm)
uvicorn.run(app, timeout_graceful_shutdown=30)

### Exercise 5.3: Stateless Design
- Invoke-RestMethod `-Uri "http://localhost:8000/chat"   -Method POST -ContentType "application/json" `-Body '{"question":"My name is Alice","session_id":"abc-123-xyz"}'
>>session_id : abc-123-xyz
>>question   : My name is Alice
>>answer     : Agent Äang hoáº¡t Äá»ng tá»t! (mock response) Há»i thÃªm cÃ¢u há»i Äi nhÃ©.
>>turn       : 2
>>served_by  : instance-56bf5e
>>storage    : redis

- Invoke-RestMethod -Uri "http://localhost:8000/chat" -Method POST -ContentType "application/json" -Body '{"question":"What is my name?","session_id":"abc-123-xyz"}'

>>session_id : abc-123-xyz
>>question   : What is my name?
>>answer     : Agent Äang hoáº¡t Äá»ng tá»t! (mock response) Há»i thÃªm cÃ¢u há»i Äi nhÃ©.
>>turn       : 3
>>served_by  : instance-56bf5e
>>storage    : redis

Tại sao cần stateless?
- Scale: nếu state trong memory, request 2 đến instance khác → mất session
- Reliability: restart instance → mất tất cả data

Giải pháp:
- Session + history lưu trong Redis (key: session:{id})
- TTL: 3600 giây (1 tiếng không hoạt động → tự xóa)
- Giữ tối đa 20 messages per session (tiết kiệm memory)

Code flow:
1. load_session(session_id) → đọc từ Redis
2. Gọi LLM với context
3. append_to_history(session_id, "user", question)
4. append_to_history(session_id, "assistant", answer)
5. save_session(session_id, session) → ghi lại vào Redis

Redis fallback: Nếu không có Redis → dùng in-memory dict (⚠️ not scalable)

### Exercise 5.4: Load Balanced Stack

docker-compose scale:
- Chạy 3 agent instances cùng lúc
- Nginx làm load balancer (round-robin)
- Mỗi request tự động distribute qua các instances

Nginx upstream config:
upstream agent_backend {
    server agent:8000;   # instance 1
    # instance 2, 3 được add khi scale
    keepalive 32;
}

Test: Round-robin → served_by luân phiên giữa các instance

### Exercise 5.5: Test Stateless

Test 1: Gửi request → tạo session
Test 2: Kill một instance đang chạy
Test 3: Gửi request với session_id đó → vẫn có context

→ Stateless: session không phụ thuộc instance cụ thể

---

## Part 6: Final Project (06-lab-complete)

###  Deployment

- **Public URL:** https://helloword-production-6719.up.railway.app
- **railway.toml:**  Multi-stage Dockerfile, healthcheck, restart policy
- **render.yaml:**  Infrastructure as Code với web service + Redis
- **Platform:** Railway

###  Verification Tests

**Health endpoint:**
```bash
curl https://helloword-production-6719.up.railway.app/health
→ 200 OK
→ {"status":"ok","version":"1.0.0","environment":"development",
   "uptime_seconds":388.6,"total_requests":5,"checks":{"llm":"mock"},
   "timestamp":"2026-04-17T16:32:25.014159+00:00"}
```

**Readiness endpoint:**
```bash
curl https://helloword-production-6719.up.railway.app/ready
→ 200 OK
→ {"ready":true}
```

**Authentication (without key → 401):**
```bash
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"test"}'
→ 401: {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
```

**Agent works with API key:**
```bash
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question":"My name is Thanh"}'
→ 200 OK
→ {"question":"My name is Thanh",
   "answer":"Agent đang hoạt động tốt! (mock response)",
   "model":"gpt-4o-mini",
   "timestamp":"2026-04-17T16:33:31.970122+00:00",
   "session_id":"88f4faf5-b1d6-45bc-b28c-d1e698d3a95b",
   "storage":"in-memory"}
```

**Conversation history (session_id):**
```bash
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is Docker?","session_id":"88f4faf5-b1d6-45bc-b28c-d1e698d3a95b"}'
→ 200 OK
→ {"question":"What is Docker?",...,"session_id":"88f4faf5-b1d6-45bc-b28c-d1e698d3a95b",
   "storage":"in-memory"}
```

**Rate limiting (20 req/min):**
```bash
# Request 1-20: 200 OK
# Request 21+: 429 Too Many Requests
{"detail":"Rate limit exceeded: 20 req/min"}
```

###  Docker & Configuration

| Tiêu chí | Kết quả |
|----------|---------|
| Multi-stage Dockerfile |  Stage 1 (builder) + Stage 2 (runtime) |
| Image size |  **247MB** (< 500MB) |
| Non-root user |  `useradd -r -g agent agent` |
| docker-compose.yml |  agent + redis, healthcheck, depends_on |
| Environment config |  Tất cả từ `app/config.py` (12-factor) |

###  Security

| Tiêu chí | Kết quả |
|----------|---------|
| API Key auth |  `X-API-Key: dev-key-change-me` → 200, không có → 401 |
| Rate limiting |  In-memory sliding window (20 req/min) |
| Cost guard |  Track tokens + daily budget $5.00 |
| No hardcoded secrets |  Không tìm thấy `sk-` trong code |

###  Reliability

| Tiêu chí | Kết quả |
|----------|---------|
| `/health` endpoint |  200 OK, liveness probe |
| `/ready` endpoint |  200 OK, readiness probe |
| Graceful shutdown |  SIGTERM handler + `timeout_graceful_shutdown=30` |
| Stateless design |  Session trong Redis (hoặc in-memory fallback) |

###  Architecture Summary (06-lab-complete/)

```
06-lab-complete/
├── app/
│   ├── main.py       /ask, /health, /ready + auth + rate limit + cost guard + session
│   └── config.py    12-factor: tất cả từ env vars
├── utils/            mock_llm.py
├── Dockerfile        Multi-stage, non-root, slim
├── docker-compose.yml  agent + redis
├── railway.toml      Railway deployment config
├── render.yaml      Render IaC config
├── requirements.txt  fastapi, uvicorn, redis, pydantic, pyjwt
├── .gitignore       .env, __pycache__, .venv/
├── .dockerignore    __pycache__, .env
└── .env.example     Template không có secrets
```

###  Production Readiness Check

```bash
cd 06-lab-complete
python check_production_ready.py
```

Result:
-  Dockerfile exists
-  Multi-stage build
-  Non-root user
-  HEALTHCHECK instruction
-  Slim base image
-  docker-compose.yml with redis
-  /health endpoint defined
-  /ready endpoint defined
-  Authentication implemented
-  Rate limiting implemented
-  Graceful shutdown (SIGTERM)
-  Structured logging (JSON)
-  No hardcoded secrets
-  .env in .gitignore