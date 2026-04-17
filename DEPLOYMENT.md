# Deployment Information

> **Student:** Vũ Tiến Thành
> **Date:** 17/04/2026

---

## Public URL

**https://helloword-production-6719.up.railway.app**

---

## Platform

**Railway** (Free tier with $5 credit)

---

## Test Commands

### Health Check
```bash
curl https://helloword-production-6719.up.railway.app/health
# Expected: {"status": "ok", "version": "1.0.0", ...}
```

### API Test (with authentication)
```bash
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: {"question": "Hello", "answer": "...", "model": "gpt-4o-mini", ...}
```

### Test without API key (should return 401)
```bash
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: {"detail": "Invalid or missing API key. Include header: X-API-Key: <key>"}
```

### Test conversation continuity
```bash
# First request (auto-generate session_id)
SESSION=$(curl -s -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "My name is Thanh"}' | python -c "import sys,json; print(json.load(sys.stdin)['session_id'])")

# Second request (use session_id for continuity)
curl -X POST https://helloword-production-6719.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d "{\"question\": \"What is my name?\", \"session_id\": \"$SESSION\"}"
```

### Readiness Probe
```bash
curl https://helloword-production-6719.up.railway.app/ready
# Expected: {"ready": true}
```

---

## Environment Variables Set

| Variable | Value | Source |
|---------|-------|--------|
| `AGENT_API_KEY` | `abcd` | Railway Dashboard |
| `ENVIRONMENT` | `production` | Railway Dashboard |
| `PORT` | `$PORT` (auto) | Railway auto-inject |
| `REDIS_URL` | (not set) | App fallback to in-memory |

---

## Deployment Configuration

### railway.toml
```toml
[build]
builder = "DOCKERFILE"

[deploy]
healthcheckPath = "/health"
healthcheckTimeout = 30
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 3
```

### docker-compose.yml
- **agent**: FastAPI app (2 workers), depends on redis
- **redis**: Redis 7-alpine, healthcheck, 128MB maxmemory
- Health check: `GET /health` every 30s
- Port: `8000:8000`

---

## Architecture

```
Client
  │
  ▼
Railway (Cloud)
  │
  ├─ nginx/load balancer (if scaled)
  │
  ├─ agent-1 (FastAPI) ── session ──► Redis
  ├─ agent-2 (FastAPI) ── session ──► Redis
  └─ agent-3 (FastAPI) ── session ──► Redis
```

---

## Screenshots

Screenshots of working deployment available in:
- Railway Dashboard: https://railway.app/project/e65b96de-155e-496c-99dd-aedfbe12d641

---

## Build & Deploy Info

- **Docker Image:** 06-lab-complete-agent
- **Image Size:** 247MB (< 500MB requirement)
- **Base Image:** python:3.11-slim (multi-stage build)
- **Multi-stage:**  Builder (gcc, pip) + Runtime (slim, non-root)
- **Non-root:**  user `agent`
