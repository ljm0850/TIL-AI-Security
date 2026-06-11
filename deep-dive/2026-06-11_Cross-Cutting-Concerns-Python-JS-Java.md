# 2026-06-11 심화 정리 — 유지보수를 위한 횡단 관심사 분리: Python · JavaScript · Java 비교

> **과정**: 인공지능 사관학교 AI 보안 과정
> **작성 계기**: Java, Python, JavaScript를 배우면서 각 언어의 유지보수 도구들(데코레이터, 프록시, 인터페이스 등)이 결국 같은 문제를 풀고 있다는 걸 깨달아 한 번에 정리

---

## 왜 정리했는가

Python 데코레이터, JavaScript 프록시, Java 인터페이스 — 각자 배울 때는 "이 언어의 기능"으로만 봤다. 그런데 되돌아보면 모두 **기존 코드를 건드리지 않고 공통 관심사를 끼워 넣는다**는 같은 문제를 풀고 있다. 언어마다 이름과 문법만 다를 뿐이다.

이걸 목적별로 묶어두면 새로운 언어나 프레임워크를 배울 때 "여기서는 이 문제를 어떻게 푸나?"를 빠르게 찾을 수 있다.

---

## 핵심 개념: 횡단 관심사 (Cross-Cutting Concerns)

핵심 비즈니스 로직과는 별개로, 여러 곳에서 반복되는 공통 관심사를 뜻한다.

```
핵심 로직:   "부서 정보를 조회한다"
횡단 관심사: 로그 기록 / 인증 확인 / 실행 시간 측정 / 캐싱 / 재시도
```

이걸 각 함수·메서드마다 직접 넣으면 코드가 오염된다. 분리하는 게 유지보수의 핵심이다.

---

## JavaScript 도구 비교 — Proxy vs 데코레이터 vs HOF

JavaScript에는 같은 횡단 관심사 문제를 푸는 도구가 세 가지 있다. 각각 적용 대상과 시점이 다르다.

### 세 가지 도구 한눈에 비교

| | Proxy | 데코레이터 (TC39) | HOF (고차함수) |
|---|---|---|---|
| 적용 대상 | 객체·함수 전체 | 클래스·메서드·속성 | 함수 |
| 적용 시점 | 런타임 (동적) | 선언 시점 (정적) | 호출 전 래핑 |
| 문법 | `new Proxy(target, handler)` | `@decorator` | `const fn = hof(원본함수)` |
| 표준 여부 | ES2015 표준 | TC39 Stage 3 (2024~, 빌드 도구 필요) | ES 표준 |
| 주요 용도 | 객체 접근 가로채기, 유효성 검사 | 클래스 기반 AOP, 프레임워크 | 함수 합성, 재사용 래퍼 |

---

### HOF (High-Order Function, 고차함수)

함수를 인자로 받아 새 함수를 반환하는 방식. 가장 단순하고 표준적인 래핑 방법이다.

```javascript
// 로깅 HOF
function withLogging(fn) {
    return function(...args) {
        console.log(`[LOG] ${fn.name} 호출`, args);
        const result = fn(...args);
        console.log(`[LOG] ${fn.name} 완료 →`, result);
        return result;
    };
}

// 인증 HOF
function withAuth(fn, isAuthenticated) {
    return function(...args) {
        if (!isAuthenticated()) throw new Error("인증 필요");
        return fn(...args);
    };
}

// 캐싱 HOF
function withCache(fn) {
    const cache = new Map();
    return function(...args) {
        const key = JSON.stringify(args);
        if (!cache.has(key)) cache.set(key, fn(...args));
        return cache.get(key);
    };
}

// HOF 조합 — 함수를 감싸는 순서대로 적용
const getDept = withLogging(withAuth(withCache(
    function getDept(deptno) {
        return db.query(deptno);
    }
), () => true));
```

**Python 데코레이터와 HOF 비교**
```python
# Python: @문법으로 선언 시점에 적용
@log
@require_auth
def get_dept(deptno): ...
```
```javascript
// JavaScript HOF: 같은 발상, 문법만 다름
const getDept = withLogging(withAuth(getDeptRaw, isAuth));
```

Python `@decorator`는 사실 `fn = decorator(fn)`의 문법 설탕이다. JS HOF와 본질적으로 동일하다.

---

### JavaScript 데코레이터 (TC39 Stage 3, 2024~)

클래스와 클래스 멤버에 `@` 문법으로 적용하는 방식. TypeScript나 Babel 환경에서 사용 가능하다. (현재 브라우저 네이티브 미지원, 빌드 도구 필요)

```javascript
// 메서드 데코레이터 정의
function log(fn, context) {
    return function(...args) {
        console.log(`[LOG] ${context.name} 호출`);
        const result = fn.apply(this, args);
        console.log(`[LOG] ${context.name} 완료`);
        return result;
    };
}

function readonly(_, context) {
    context.addInitializer(function() {
        Object.defineProperty(this, context.name, { writable: false });
    });
}

// 사용
class DeptService {
    @log
    getDept(deptno) {
        return db.query(deptno);
    }

    @readonly
    MAX_SIZE = 20;
}
```

**클래스 전체에 적용 — 싱글턴 패턴**
```javascript
function singleton(cls) {
    let instance;
    return class extends cls {
        constructor(...args) {
            if (instance) return instance;
            super(...args);
            instance = this;
        }
    };
}

@singleton
class AppConfig { ... }
```

---

### Proxy

객체나 함수 자체를 가로채는 방식. 13가지 trap으로 동작을 세밀하게 제어한다. HOF나 데코레이터로 처리하기 어려운 **객체 속성 접근·수정 제어**가 핵심 강점이다.

```javascript
// 속성 쓰기 검증 (set trap) — HOF로는 처리 불가한 영역
const validator = new Proxy({}, {
    set(target, prop, value) {
        if (prop === 'deptno' && (value < 1 || value > 99)) {
            throw new RangeError('deptno는 1~99 사이여야 합니다');
        }
        target[prop] = value;
        return true;
    }
});

validator.deptno = 10;   // OK
validator.deptno = 200;  // RangeError

// 존재하지 않는 속성 접근 방어 (get trap)
const safeObj = new Proxy(data, {
    get(target, prop) {
        if (!(prop in target)) throw new Error(`${prop} 없음`);
        return target[prop];
    }
});
```

**주요 trap**

| trap | 가로채는 동작 |
|---|---|
| `get` | 속성 읽기 |
| `set` | 속성 쓰기 |
| `apply` | 함수 호출 |
| `has` | `in` 연산자 |
| `deleteProperty` | `delete` 연산자 |
| `construct` | `new` 연산자 |

---

### 언제 무엇을 쓰나

| 상황 | 선택 | 이유 |
|---|---|---|
| 함수 하나에 공통 로직 추가 | **HOF** | 가장 단순, 빌드 도구 불필요 |
| 클래스 메서드 여러 개에 일괄 적용 | **데코레이터** | 선언적, Python/Java 스타일과 일치 |
| 객체 속성 접근·수정을 가로채야 할 때 | **Proxy** | 유일하게 `get`/`set` trap 지원 |
| 입력값 유효성 검사 (객체 단위) | **Proxy** | setter trap으로 모든 쓰기를 일괄 제어 |
| React 컴포넌트 래핑 | **HOF** | `React.memo(Component)`, `connect(Component)` |

---

## 목적 1. 로깅 (Logging)

실행 전후에 함수 이름, 인자, 결과를 기록한다.

**Python — 데코레이터**
```python
from functools import wraps

def log(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"[LOG] {fn.__name__} 호출")
        result = fn(*args, **kwargs)
        print(f"[LOG] {fn.__name__} 완료 → {result}")
        return result
    return wrapper

@log
def get_dept(deptno: int):
    return {"deptno": deptno, "dname": "SALES"}
```

**JavaScript — HOF (기본) / Proxy (객체 단위) / 데코레이터 (클래스)**
```javascript
// HOF — 가장 단순
const getDept = withLogging(function getDept(deptno) {
    return { deptno, dname: "SALES" };
});

// Proxy — 함수 호출 가로채기
const getDept = new Proxy(getDeptRaw, {
    apply(fn, thisArg, args) {
        console.log(`[LOG] ${fn.name} 호출`, args);
        const result = Reflect.apply(fn, thisArg, args);
        console.log(`[LOG] ${fn.name} 완료 →`, result);
        return result;
    }
});

// 데코레이터 — 클래스 메서드에 선언적으로 적용
class DeptService {
    @log
    getDept(deptno) { return db.query(deptno); }
}
```

**Java — Spring AOP**
```java
@Aspect @Component
public class LogAspect {
    @Around("execution(* app.labs.dept.service.*.*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("[LOG] " + pjp.getSignature().getName() + " 호출");
        Object result = pjp.proceed();
        System.out.println("[LOG] 완료 → " + result);
        return result;
    }
}
```

---

## 목적 2. 인증 확인 (Authentication)

핵심 로직 실행 전에 로그인 여부 또는 권한을 확인한다.

**Python — 데코레이터**
```python
def require_auth(fn):
    @wraps(fn)
    def wrapper(request, *args, **kwargs):
        if not request.get("token"):
            raise PermissionError("인증 필요")
        return fn(request, *args, **kwargs)
    return wrapper

@require_auth
def get_secret(request):
    return "비밀 데이터"
```

**JavaScript — HOF / Proxy**
```javascript
// HOF — 함수 단위 인증
const getSecret = withAuth(
    function getSecret(request) { return "비밀 데이터"; },
    (req) => !!req.token
);

// Proxy — 객체 속성 접근 자체를 차단
const secretData = new Proxy({ value: "비밀 데이터" }, {
    get(obj, prop) {
        if (!isAuthenticated()) throw new Error("인증 필요");
        return obj[prop];
    }
});
```

**Java — Spring AOP + 커스텀 어노테이션**
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequireAuth {}

@Around("@annotation(RequireAuth)")
public Object checkAuth(ProceedingJoinPoint pjp) throws Throwable {
    if (!isAuthenticated()) throw new SecurityException("인증 필요");
    return pjp.proceed();
}

@RequireAuth
public Dept getDeptInfo(int deptno) { ... }
```

---

## 목적 3. 캐싱 (Caching)

동일한 입력에 대해 재계산 없이 이전 결과를 반환한다.

**Python — `@lru_cache` / 데코레이터**
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_dept(deptno: int):
    return db.query(deptno)
```

**JavaScript — HOF / Proxy**
```javascript
// HOF — 직접 구현
const getDept = withCache(function getDept(deptno) {
    return db.query(deptno);
});

// Proxy — apply trap으로 동일하게 구현 가능
const getDept = new Proxy(getDeptRaw, {
    apply(target, thisArg, args) {
        const key = JSON.stringify(args);
        if (!cache.has(key)) cache.set(key, Reflect.apply(target, thisArg, args));
        return cache.get(key);
    }
});
```

**Java — `@Cacheable`**
```java
@Cacheable("depts")
public Dept getDeptInfo(int deptno) {
    return deptRepository.findById(deptno);
}

@CacheEvict("depts")
public void updateDept(Dept dept) { ... }
```

---

## 목적 4. 재시도 (Retry)

일시적 오류 발생 시 자동으로 재시도한다. LLM API, 외부 HTTP 호출 등에 필수.

**Python — async 데코레이터 (지수 백오프)**
```python
def retry(max_attempts=3):
    def decorator(fn):
        @wraps(fn)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await fn(*args, **kwargs)
                except Exception:
                    if attempt == max_attempts - 1: raise
                    await asyncio.sleep(2 ** attempt)
        return wrapper
    return decorator

@retry(max_attempts=3)
async def call_llm(prompt: str): ...
```

**JavaScript — HOF**
```javascript
function withRetry(fn, maxAttempts = 3) {
    return async function(...args) {
        for (let attempt = 0; attempt < maxAttempts; attempt++) {
            try {
                return await fn(...args);
            } catch (e) {
                if (attempt === maxAttempts - 1) throw e;
                await new Promise(res => setTimeout(res, 2 ** attempt * 1000));
            }
        }
    };
}

const callLlm = withRetry(async function callLlm(prompt) { ... });
```

**Java — `@Retryable`**
```java
@Retryable(
    value = RuntimeException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public String callExternalApi(String prompt) { ... }

@Recover
public String fallback(RuntimeException e, String prompt) {
    return "서비스 일시 불가";
}
```

---

## 목적 5. 구현 교체 (Strategy / 인터페이스)

구체적인 구현을 숨기고, 인터페이스만 보고 코드를 작성한다. 구현체를 바꿔도 호출 코드는 수정하지 않는다.

**Python — ABC + 덕 타이핑**
```python
from abc import ABC, abstractmethod

class DeptRepository(ABC):
    @abstractmethod
    def find_all(self) -> list: ...

class MybatisDeptRepository(DeptRepository):
    def find_all(self): return db.query("SELECT * FROM DEPT")

class InMemoryDeptRepository(DeptRepository):
    def find_all(self): return [{"deptno": 10, "dname": "SALES"}]

repo: DeptRepository = MybatisDeptRepository()
repo.find_all()
```

**JavaScript — Proxy (메서드 존재 여부 런타임 강제)**
```javascript
function createRepository(impl) {
    const required = ['findAll', 'findById', 'save', 'delete'];
    return new Proxy(impl, {
        get(target, prop) {
            if (required.includes(prop) && typeof target[prop] !== 'function') {
                throw new Error(`${prop} 미구현`);
            }
            return target[prop];
        }
    });
}
```

**Java — 인터페이스 + DI**
```java
public interface DeptRepository {
    List<Dept> findAll();
    Dept findById(int deptno);
}

@Configuration
public class RepositoryConfig {
    @Primary @Bean
    DeptRepository active(DeptMapper mapper) {
        return new DeptMyBatisRepository(mapper);  // @Primary 이동으로 JPA 전환
    }
}
```

Spring AOP 내부도 이 프록시 패턴이다. 인터페이스가 있으면 JDK Dynamic Proxy, 없으면 CGLIB 프록시를 사용한다.

---

## 전체 비교 요약

| 목적 | Python | JavaScript | Java |
|---|---|---|---|
| 로깅 | 데코레이터 | HOF / Proxy / 데코레이터 | Spring AOP `@Around` |
| 인증 | 데코레이터 | HOF / Proxy (get trap) | AOP + 커스텀 어노테이션 |
| 캐싱 | `@lru_cache` / 데코레이터 | HOF / Proxy (apply trap) | `@Cacheable` |
| 재시도 | async 데코레이터 | HOF | `@Retryable` |
| 구현 교체 | ABC + 덕 타이핑 | Proxy (get trap) | 인터페이스 + DI |

**공통 발상**: 기존 코드를 건드리지 않고 앞뒤 또는 접근 시점에 끼워 넣는다.

**언어별 차이**:
- Java: 컴파일 타임에 인터페이스로 강제, 어노테이션으로 선언적 AOP
- Python: `@` 문법 설탕 + 런타임 유연성, async 지원 자연스러움
- JavaScript: HOF(가장 단순) / 데코레이터(클래스 기반) / Proxy(객체 접근 제어) 세 가지 중 상황에 맞게 선택

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 | 아직 모르는 것 |
|---|---|
| JS의 HOF, 데코레이터, Proxy가 각각 다른 적용 대상과 시점을 갖는다 | TypeScript에서 `experimentalDecorators`(구) vs TC39 데코레이터(신)의 차이 |
| HOF와 Python 데코레이터가 본질적으로 동일한 패턴이다 (`fn = decorator(fn)`) | JavaScript Proxy의 나머지 trap(`has`, `deleteProperty`, `construct` 등) 실제 활용 사례 |
| Proxy만이 `get`/`set` trap으로 객체 속성 접근 자체를 가로챌 수 있다 |  |

---

## 출처

- Java, Python, JavaScript 강의 실습 복기 + 자체 정리
- 관련 deep-dive:
  - `../deep-dive/2026-06-10_functools-wraps-deep-dive.md`
  - `../deep-dive/2026-06-10_Async-Decorator-LLM-Streaming.md`

---
*작성일: 2026-06-11 | 인공지능 사관학교 AI 보안 과정*
