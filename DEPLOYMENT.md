# Deployment Information — Day 12 Lab

> **Student:** Pham Dang Phong  
> **Platform:** Railway  
> **Status:** ⏳ Pending deployment 

---

## Public URL

```
https://day12ai20-production.up.railway.app
```

---

## Platform

**Railway** — Dùng Dockerfile (multi-stage), auto-deploy từ GitHub

---

## Test Commands

### Health Check
```bash
curl https://day12ai20-production.up.railway.app/health
# Expected:
# {
#   "status": "ok",
#   "version": "1.0.0",
#   "environment": "production",
#   "uptime_seconds": 42.3,
#   "total_requests": 5,
#   "checks": {"llm": "mock"},
#   "timestamp": "2026-04-17T09:00:00+00:00"
# }
```

### Authentication Required (401)
```bash
curl https://day12ai20-production.up.railway.app/ask
# Expected: HTTP 401 - {"detail":"Invalid or missing API key..."}
```

### API Test (with key)
```bash
curl -X POST https://day12ai20-production.up.railway.app/ask \
  -H "X-API-Key: YOUR_AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello! What is deployment?"}'
# Expected: HTTP 200
# {
#   "question": "Hello! What is deployment?",
#   "answer": "Deployment là quá trình đưa code...",
#   "model": "gpt-4o-mini",
#   "timestamp": "2026-04-17T09:00:00+00:00"
# }
```

### Rate Limit Test (429 after 20 requests)
```bash
for i in $(seq 1 25); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST https://day12ai20-production.up.railway.app/ask \
    -H "X-API-Key: YOUR_AGENT_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"question\": \"Test $i\"}"
done
# Expected: 200 200 200 ... (first 20) then 429 429 429
```

### Readiness Check
```bash
curl https://day12ai20-production.up.railway.app/ready
# Expected: {"ready": true}
```

---

## Environment Variables Set on Railway

| Variable | Value |
|----------|-------|
| `PORT` | (auto-set by Railway) |
| `ENVIRONMENT` | `production` |
| `AGENT_API_KEY` | (secret — set manually) |
| `RATE_LIMIT_PER_MINUTE` | `20` |
| `DAILY_BUDGET_USD` | `5.0` |
| `LOG_LEVEL` | `INFO` |

---

## Screenshots

| Screenshot | Link |
|------------|------|
| Deployment dashboard | `screenshots/dashboard.png` |
| Service running | `screenshots/running.png` |
| Health check test | `screenshots/health_check.png` |
| API test result | `screenshots/api_test.png` |

---

## Local Test (Docker Compose)

```bash
cd 06-lab-complete

# Copy env template
cp .env.example .env.local

# Start stack (agent + redis)
docker compose up -d

# Test health
curl http://localhost:8000/health

# Test API
curl -X POST http://localhost:8000/ask \
  -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

---

## Hướng Dẫn Deploy Lên Railway

### Bước 1: Cài Railway CLI
```bash
npm i -g @railway/cli
```

### Bước 2: Login Railway
```bash
railway login
```

### Bước 3: Init project (từ thư mục 06-lab-complete)
```bash
cd 06-lab-complete
railway init
```

### Bước 4: Set environment variables
```bash
railway variables set ENVIRONMENT=production
railway variables set AGENT_API_KEY=my-super-secret-key-2026
railway variables set RATE_LIMIT_PER_MINUTE=20
railway variables set DAILY_BUDGET_USD=5.0
```

### Bước 5: Deploy
```bash
railway up
```

### Bước 6: Lấy public URL
```bash
railway domain
```

### Bước 7: Cập nhật file này với URL thực tế và chụp screenshots
