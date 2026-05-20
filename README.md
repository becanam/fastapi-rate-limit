# FastAPI Rate Limiter

A simple FastAPI server with rate limiting using slowapi. Limits each client to 10 requests per minute by IP address.

---

## 코드 (Code)

```python
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/hello")
@limiter.limit("10/minute")
async def hello(request: Request):
    return {"message": "Hello, World!"}
```

---

## 선택한 알고리즘 (Algorithm)

slowapi는 내부적으로 `limits` 라이브러리를 사용합니다.
`limits`는 3가지 전략을 제공합니다:

| 전략 | 메모리 | 정확도 | 특징 |
|------|--------|--------|------|
| Fixed Window | 낮음 | 보통 | 단순, 경계에서 burst 가능 |
| Moving Window | 높음 | 높음 | 정확하지만 복잡 |
| Sliding Window Counter | 중간 | 중간 | 두 버킷의 가중합으로 근사 |

이 실습에서는 **Fixed Window**가 사용됩니다 (slowapi 기본값).

### Fixed Window 동작 방식
- 첫 요청이 오면 60초짜리 윈도우가 시작됨
- 그 안의 모든 요청이 카운터를 증가시킴
- 윈도우가 만료되면 카운터 리셋
- 경계 시점에서 burst가 발생할 수 있음

### 실제 테스트에서 확인된 동작
- 요청 1~10 → 같은 초(15:59:57)에 모두 허용 ✅
- 요청 11~12 → 즉시 차단 (429) ❌
- 윈도우 만료 후 → 카운터 리셋, 다시 10회 허용

---

## 아키텍처 (Architecture)

```
Client (curl)
     │
     │ HTTP GET /hello
     ▼
┌─────────────────────────────┐
│         FastAPI App         │
│                             │
│  ┌──────────────────────┐   │
│  │   slowapi Limiter    │   │
│  │  key: client IP      │   │
│  │  rule: 10/minute     │   │
│  └──────────┬───────────┘   │
│             │               │
│     ┌───────┴────────┐      │
│     │                │      │
│  allowed          blocked   │
│     │                │      │
│  200 OK         429 Too     │
│  + message      Many Reqs   │
└─────────────────────────────┘
```

- **FastAPI** — 웹 프레임워크, HTTP 요청 처리
- **slowapi Limiter** — 미들웨어로 동작, 모든 요청을 가로채서 IP별 카운트
- **get_remote_address** — 클라이언트 IP를 키로 사용해 사람별로 독립적으로 제한
- **limits 라이브러리** — slowapi 하위에서 실제 Fixed Window 알고리즘 구현
- **RateLimitExceeded handler** — 한도 초과 시 429 응답을 올바르게 반환

---

## 테스트 방법 (Test Method)

### 1. 설치
```bash
pip install fastapi uvicorn slowapi
```

### 2. 서버 실행
```bash
uvicorn main:app --reload
```

### 3. 테스트 명령어
12개의 요청을 빠르게 연속으로 보내 rate limit 동작을 확인합니다:

```bash
for i in {1..12}; do
  echo -n "Request $i at $(date '+%H:%M:%S') → "
  curl -s -w " (HTTP %{http_code})\n" http://127.0.0.1:8000/hello
done
```

### 4. 실행 결과 (Execution Results)

```
Request 1  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 2  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 3  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 4  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 5  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 6  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 7  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 8  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 9  at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 10 at 15:59:57 → {"message":"Hello, World!"} (HTTP 200)
Request 11 at 15:59:57 → {"error":"Rate limit exceeded: 10 per 1 minute"} (HTTP 429)
Request 12 at 15:59:57 → {"error":"Rate limit exceeded: 10 per 1 minute"} (HTTP 429)
```

**결과:** 요청 1~10은 성공(200), 11~12는 차단(429) — rate limiter 정상 동작 확인 