# 2026-06-10 심화 정리 — 비동기 데코레이터와 LLM 서비스 활용

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../practice/2026-06-10_AI-Security-Course_Practice_Python-Basics.md`
> **작성 계기**: 데코레이터 deep-dive 중 "LLM 서비스에서 비동기 처리를 데코레이터로 어떻게 다룰까?" 라는 의문에서 출발

---

## 왜 찾아봤는가

데코레이터 deep-dive를 정리 하면서 든 생각 — LLM을 이용한 서비스에서 클라이언트가 API 요청을 하면 LLM이 응답할 때까지 기다리는 것은 UX 측면에서 좋지 않다. 비동기로 처리해야 할 텐데, 그렇다면 데코레이터를 어떻게 만들어야 하고 어떻게 활용할 수 있을까?

---

## 핵심 내용

### 1. 일반 데코레이터를 async 함수에 쓰면 망가진다

Python에서 비동기 함수(`async def`)에 일반 데코레이터를 그대로 쓰면 동작이 깨진다.

```python
from functools import wraps

def log_decorator(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"{fn.__name__} 시작")
        result = fn(*args, **kwargs)   # ← 문제: coroutine을 실행하지 않고 반환
        print(f"{fn.__name__} 완료")  # ← LLM 응답 전에 찍힘
        return result
    return wrapper

@log_decorator
async def call_llm(prompt: str):
    ...
```

**무슨 일이 생기냐면:**
- `async def` 함수를 일반 함수처럼 호출하면 coroutine 객체가 반환될 뿐, 실제로 실행되지 않는다
- "완료" 로그가 LLM 응답이 오기 전에 찍힌다
- 반환값이 coroutine 객체라 이후 코드에서 실제 데이터를 쓸 수 없다

---

### 2. 비동기 데코레이터 — wrapper도 async로 만들면 된다

해결 방법은 간단하다. `wrapper`도 `async def`로 선언하고, 원본 함수 호출 앞에 `await`을 붙이면 된다.

```python
from functools import wraps
import asyncio

def async_log_decorator(fn):
    @wraps(fn)
    async def wrapper(*args, **kwargs):     # async wrapper
        print(f"{fn.__name__} 시작")
        result = await fn(*args, **kwargs)  # await으로 실제 실행 대기
        print(f"{fn.__name__} 완료")
        return result
    return wrapper

@async_log_decorator
async def call_llm(prompt: str):
    await asyncio.sleep(2)   # LLM 응답 대기 시뮬레이션
    return f"응답: {prompt}"
```

---

### 3. LLM 서비스에서 실제로 쓸 수 있는 비동기 데코레이터 패턴

#### 패턴 1. 응답 시간 측정 (성능 모니터링)

```python
def measure_time(fn):
    @wraps(fn)
    async def wrapper(*args, **kwargs):
        start = asyncio.get_event_loop().time()
        result = await fn(*args, **kwargs)
        elapsed = asyncio.get_event_loop().time() - start
        print(f"{fn.__name__}: {elapsed:.2f}초")
        return result
    return wrapper
```

#### 패턴 2. 재시도 (LLM API는 간헐적으로 실패하므로 필수)

```python
def retry(max_attempts=3):
    def decorator(fn):
        @wraps(fn)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await fn(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise   # 마지막 시도도 실패하면 예외 그대로 전파
                    wait = 2 ** attempt   # 지수 백오프: 1초 → 2초 → 4초
                    await asyncio.sleep(wait)
        return wrapper
    return decorator
```

#### 패턴 3. 인증 확인 (API 요청 전 토큰 검증)

```python
def require_auth(fn):
    @wraps(fn)
    async def wrapper(request, *args, **kwargs):
        if not request.headers.get("Authorization"):
            raise PermissionError("인증 필요")
        return await fn(request, *args, **kwargs)
    return wrapper
```

#### 데코레이터 조합 — 실제 엔드포인트에 적용

```python
@require_auth          # 3. 인증 확인 (가장 바깥)
@retry(max_attempts=3) # 2. 실패 시 재시도
@measure_time          # 1. 응답 시간 측정 (가장 안쪽)
async def call_llm(request, prompt: str):
    # OpenAI / Anthropic API 호출
    response = await openai_client.chat.completions.create(...)
    return response
```

데코레이터는 아래에서 위로 쌓인다. 실행 순서는 `require_auth` → `retry` → `measure_time` → `call_llm`.

---

### 4. 비동기만으로는 부족하다 — 스트리밍이 함께 필요한 이유

비동기 처리만으로는 "기다리는 느낌"을 완전히 없애기 어렵다.

```
비동기 처리 → 서버가 블로킹되지 않음 (다른 요청을 동시에 처리 가능)
스트리밍    → 클라이언트가 첫 토큰부터 바로 화면에 표시
```

ChatGPT에서 글자가 하나씩 나오는 것처럼 보이는 게 바로 스트리밍이다. LLM이 토큰을 생성하는 순서대로 클라이언트에 흘려보낸다.

**구현 스택**

| 레이어 | 기술 |
|---|---|
| 서버 (Python) | FastAPI `StreamingResponse` |
| 전송 프로토콜 | SSE (Server-Sent Events) |
| 클라이언트 | `EventSource` 또는 `fetch` + `ReadableStream` |

```python
# FastAPI 스트리밍 응답 예시
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def generate_tokens(prompt: str):
    async for chunk in openai_client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}],
        stream=True   # 스트리밍 활성화
    ):
        token = chunk.choices[0].delta.content or ""
        yield f"data: {token}\n\n"   # SSE 형식

@app.post("/chat")
@require_auth
async def chat(request, prompt: str):
    return StreamingResponse(
        generate_tokens(prompt),
        media_type="text/event-stream"
    )
```

```javascript
// 클라이언트 (EventSource)
const source = new EventSource('/chat');
source.onmessage = (e) => {
    document.getElementById('response').textContent += e.data;
};
```

---

### 5. 비동기 vs 스트리밍 — 역할 정리

| | 비동기 (`async/await`) | 스트리밍 (SSE / WebSocket) |
|---|---|---|
| 해결하는 문제 | 서버 블로킹 방지 | 클라이언트 대기 시간 체감 감소 |
| 적용 위치 | 서버 내부 로직 | 서버 ↔ 클라이언트 통신 |
| 없으면 | 다른 요청이 LLM 응답까지 대기 | 전체 응답 완료까지 화면이 비어 있음 |
| 함께 쓰면 | 서버 처리 효율 + 사용자 체감 속도 모두 개선 | |

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 |
|---|
| 일반 데코레이터를 async 함수에 쓰면 coroutine이 실행되지 않는 이유 |
| wrapper를 async로 선언하고 await을 붙이면 해결된다 |
| 재시도 데코레이터에서 지수 백오프(`2 ** attempt`)가 필요한 이유 |
| 비동기와 스트리밍이 해결하는 문제가 다르고, 실제 LLM 서비스에서는 둘 다 필요하다 |

---

## 출처

- Python 데코레이터 deep-dive 정리 중 발생한 의문 → 자체 정리

---
*작성일: 2026-06-10 | 인공지능 사관학교 AI 보안 과정*
