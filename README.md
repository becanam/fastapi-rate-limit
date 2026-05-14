# FastAPI Rate Limiter

A simple FastAPI server with rate limiting using slowapi.
Limits each client to **10 requests per minute** by IP address.

## Setup

### Install dependencies
pip install fastapi uvicorn slowapi

### Run the server
uvicorn main:app --reload

## How It Works

- Every request to `/hello` is tracked by the client's IP address
- First 10 requests per minute → `200 OK`
- After 10 requests → `429 Too Many Requests`
- Counter resets after 1 minute

## Test Command

Simulation to run 12 rapid requests and verify rate limiting:

```bash
for i in {1..12}; do
  echo -n "Request $i at $(date '+%H:%M:%S') → "
  curl -s -w " (HTTP %{http_code})\n" http://127.0.0.1:8000/hello
done
```

## Test Results

Sending 12 requests back-to-back to `GET /hello`:

| Request | Time     | Response                                      | Status |
|---------|----------|-----------------------------------------------|--------|
| 1       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 2       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 3       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 4       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 5       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 6       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 7       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 8       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 9       | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 10      | 15:59:57 | {"message":"Hello, World!"}                  | ✅ 200 |
| 11      | 15:59:57 | {"error":"Rate limit exceeded: 10 per 1 minute"} | ❌ 429 |
| 12      | 15:59:57 | {"error":"Rate limit exceeded: 10 per 1 minute"} | ❌ 429 |

**Result: Rate limiter working correctly.**
Requests 1–10 succeeded, requests 11–12 were blocked.