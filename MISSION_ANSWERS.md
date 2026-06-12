# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. **Hardcoded Secrets:** API key (`sk-hardcoded-fake-key-never-do-this`) and Database URL (`postgresql://admin:password123...`) are hardcoded directly in the source file, which risk leaking if pushed to version control.
2. **No Config Management:** App settings like `DEBUG = True` and `MAX_TOKENS = 500` are hardcoded instead of being loaded dynamically from `.env` files or system environment variables.
3. **Improper Logging:** Uses standard `print()` statements for logging instead of a structured logging library, and logs the sensitive API Key (`print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`).
4. **No Health Checks:** There is no `/health` or `/ready` endpoint, preventing platforms from monitoring the status of the container.
5. **Hardcoded Server Bindings:** Host is hardcoded to `localhost` (making it inaccessible from outside the container) and port `8000` is hardcoded instead of reading the dynamic `$PORT` injected by host platforms. Additionally, `reload=True` is enabled which is an anti-pattern in production.

### Exercise 1.3: Comparison table

| Feature | Develop / Basic | Production / Advanced | Why Important? |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcoded directly inside script. | Loaded from environment variables (`pydantic` BaseSettings / `os.getenv`). | Decouples code from config, allowing deployment to dev/staging/prod without changing code or leaking secrets. |
| **Health check** | None. | Dedicated `/health` and `/ready` endpoints. | Allows orchestrators (Railway, Docker Compose, K8s) to automatically monitor liveness and restart failing services. |
| **Logging** | Standard `print()` statements. | Structured JSON logs outputted to stdout. | Enables log aggregators (ELK, Datadog) to index and query logs easily by key-value fields. |
| **Shutdown** | Process is abruptly terminated. | Handled gracefully via signal capture (SIGTERM). | Allows processing of in-flight requests and cleanly closing connections (e.g. Redis, DB) before exit. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base Image:** `python:3.11` (full Python image, which is around 1 GB).
2. **Working Directory:** `/app`.
3. **Why COPY requirements.txt first?** Docker builds images layer by layer. If `requirements.txt` is copied first and packages are installed, Docker caches this layer. Unless `requirements.txt` changes, Docker will skip installing packages on subsequent builds, speeding up compilation significantly.
4. **CMD vs ENTRYPOINT:**
   - `CMD` defines the default command and/or arguments that can be easily overridden during `docker run`.
   - `ENTRYPOINT` defines the executable command container will always run; arguments passed during `docker run` are appended to it.

### Exercise 2.3: Image size comparison
- **Develop (Single-stage python:3.11):** ~1.01 GB
- **Production (Multi-stage python:3.11-slim):** ~148 MB
- **Difference:** ~85% reduction.
- **Why:** The builder stage uses build tools (`gcc`, compilers) to compile wheel files, while the runtime stage starts with a clean `python:3.11-slim` base and only copies the generated site-packages and source code, completely excluding compiler tools and package caches.

### Exercise 2.4: Docker Compose Stack Architecture
```
                     ┌──────────────────┐
                     │   Client (HTTP)  │
                     └────────┬─────────┘
                              │ Port 80
                              ▼
                     ┌──────────────────┐
                     │    Nginx (LB)    │
                     └────────┬─────────┘
                              │ Routing (Round Robin)
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
     │ Agent (1)   │   │ Agent (2)   │   │ Agent (3)   │
     └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │ Port 6379
                              ▼
                     ┌──────────────────┐
                     │   Redis Cache    │
                     └──────────────────┘
```
- **Services:** `nginx` (acting as load-balancer), `agent` (FastAPI app instances scaled to 3), and `redis` (data store).
- **Communication:** External clients hit Nginx on port 80. Nginx proxies requests to the agents on port 8000. Agents communicate with Redis on port 6379 to check rate limits, session history, and cost budgets.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **Public URL:** [https://myproject-production-3553.up.railway.app](https://myproject-production-3553.up.railway.app)
- **Status:** Fully online, serving the premium HTML interface at `/`, checking health at `/health`, and agent asking at `/ask`.

---

## Part 4: API Security

### Exercise 4.1: API Key questions
- **Where checked:** Handled by the FastAPI security dependency `verify_api_key` utilizing `APIKeyHeader(name="X-API-Key")`.
- **Wrong Key response:** Aborts with a `401 Unauthorized` status code and details payload: `{"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}`.
- **Key Rotation:** Change the value of the `AGENT_API_KEY` environment variable in Railway variables or `.env`. The app reads the new key at startup without any code adjustments.

### Exercise 4.2-4.3: Test outputs
- **API Request with correct key:**
  ```json
  {"question":"What is Docker?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","model":"gpt-4o-mini","timestamp":"2026-06-12T11:14:35.079810+00:00"}
  ```
- **Rate limiting response:** Once requests exceed the `RATE_LIMIT_PER_MINUTE` limit, it responds with `429 Too Many Requests` and details payload:
  ```json
  {"detail":"Rate limit exceeded: 20 req/min"}
  ```

### Exercise 4.4: Cost Guard implementation
Cost guard uses an in-memory cost accumulator `_daily_cost` combined with `_cost_reset_day` to reset cost metrics daily:
```python
def check_and_record_cost(input_tokens: int, output_tokens: int):
    global _daily_cost, _cost_reset_day
    today = time.strftime("%Y-%m-%d")
    if today != _cost_reset_day:
        _daily_cost = 0.0
        _cost_reset_day = today
    if _daily_cost >= settings.daily_budget_usd:
        raise HTTPException(503, "Daily budget exhausted. Try tomorrow.")
    cost = (input_tokens / 1000) * 0.00015 + (output_tokens / 1000) * 0.0006
    _daily_cost += cost
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- **Health check (`/health`):** Verifies liveness of the application container.
- **Readiness check (`/ready`):** Returns 200 once app setup is complete.
- **Graceful Shutdown:** Configured via process signals handling `SIGTERM` to allow finishing in-flight requests and terminating connections correctly.
- **Stateless design:** All user configuration, metrics, and application logs are stateless. A Redis database container can be hooked up to sync rate limiter buckets and user histories across multiple scaled application nodes.
