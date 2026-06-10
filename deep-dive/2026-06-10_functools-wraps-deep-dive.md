# 2026-06-10 심화 정리 — functools.wraps 없이 데코레이터를 쓰면 생기는 문제

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../practice/2026-06-10_AI-Security-Course_Practice_Python-Basics.md`
> **작성 계기**: `py_02_etc.py` 데코레이터 실습 후 `@wraps`가 왜 필요한지 궁금해서 찾아봄

---

## 왜 찾아봤는가

오늘 데코레이터 실습 코드에서 `@wraps(fn)`이 붙어 있었다. 없으면 어떤 문제가 생기는지 구체적으로 알고 싶었다.

---

## 핵심 내용

`@wraps` 없이도 데코레이터는 동작한다. 하지만 함수를 둘러싼 생태계(디버깅, 문서화, 리플렉션, 프레임워크)가 기대하는 메타데이터가 사라져 다양한 문제가 발생한다.

### 1. 로그와 디버깅이 어려워짐

```python
def timer(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@timer
def process_order():
    pass

print(process_order.__name__)  # 'wrapper' — 원본 이름 손실
```

여러 함수가 모두 `wrapper`로 표시되어 어떤 함수가 실행 중인지 로그에서 구분 불가.

### 2. help() 문서가 깨짐

```python
@auth
def get_user():
    '''사용자 정보를 반환'''
    pass

help(get_user)
# Help on function wrapper  ← 원본 docstring 사라짐
```

### 3. inspect.signature()가 올바르게 동작하지 않음

```python
@deco
def add(a: int, b: int):
    return a + b

print(inspect.signature(add))
# (*args, **kwargs)  ← 원래 시그니처 손실
# 기대값: (a: int, b: int)
```

### 4. FastAPI, Click 등 프레임워크와 충돌

FastAPI 등은 `inspect.signature()`로 함수 매개변수를 분석해 API 문서와 validation을 처리한다. `@wraps`가 없으면:
- API 문서 생성 오류
- Query Parameter 인식 실패
- Validation 오류

```python
@app.get("/users")
@auth               # @wraps 없으면 FastAPI가 파라미터를 읽지 못함
def get_users(limit: int):
    ...
```

### 5. `__wrapped__` 속성이 없어 원본 함수 접근 불가

`@wraps`는 원본 함수를 가리키는 `__wrapped__` 속성을 추가한다. 없으면:

```python
inspect.unwrap(func)      # 동작 불가
func.__wrapped__()        # AttributeError
```

테스트 코드에서 캐시 데코레이터를 건너뛰고 원본 함수를 직접 호출하고 싶을 때 문제가 된다.

### 6. 데코레이터 중첩 시 디버깅 난이도 증가

```python
@deco1
@deco2
@deco3
def my_func():
    pass

my_func.__name__     # 'wrapper'
inspect.unwrap(my_func)  # 원본 추적 불가
```

---

## `@wraps`가 복사하는 메타데이터

```python
from functools import wraps

def decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

| 속성 | 설명 |
|---|---|
| `__name__` | 함수 이름 |
| `__doc__` | docstring |
| `__module__` | 모듈 이름 |
| `__annotations__` | 타입 힌트 정보 |
| `__wrapped__` | 원본 함수 참조 |

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 |
|---|
| `@wraps` 없으면 `__name__`이 `wrapper`로 바뀌어 로그/디버깅이 어려워짐 |
| FastAPI 같은 프레임워크가 `inspect.signature()`에 의존하므로 `@wraps` 필수 |
| 테스트 코드에서 원본 함수를 `__wrapped__`로 직접 호출할 수 있음 |

---

## 출처

- 강의 실습 코드 (`py_02_etc.py`) + 자체 조사

---
*작성일: 2026-06-10 | 인공지능 사관학교 AI 보안 과정*
