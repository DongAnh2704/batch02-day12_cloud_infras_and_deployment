# Deployment Information

## Public URL
https://myproject-production-3553.up.railway.app

## Platform
Railway

## Test Commands

### 🟢 1. Health Check
```bash
curl -i https://myproject-production-3553.up.railway.app/health
```
**Expected Response:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 38.9,
  "total_requests": 2,
  "checks": {
    "llm": "mock"
  },
  "timestamp": "2026-06-12T11:14:21.230565+00:00"
}
```

### 🔐 2. API Test (with X-API-Key)
```bash
curl -i -X POST https://myproject-production-3553.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: production-agent-key-2026" \
  -d '{"question": "How do you deploy an app?"}'
```
**Expected Response:**
```json
{
  "question": "How do you deploy an app?",
  "answer": "Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T11:14:35.079810+00:00"
}
```

### 🔐 3. Metrics (with X-API-Key)
```bash
curl -i https://myproject-production-3553.up.railway.app/metrics \
  -H "X-API-Key: production-agent-key-2026"
```

---

## Environment Variables Set
- `ENVIRONMENT=production`
- `AGENT_API_KEY=production-agent-key-2026`
- `JWT_SECRET=my-very-secret-jwt-key-2026`

---

## Interactive UI
You can open [https://myproject-production-3553.up.railway.app/](https://myproject-production-3553.up.railway.app/) in a browser to use the graphical chat interface and monitor live Gateway Dashboard metrics in real time.
