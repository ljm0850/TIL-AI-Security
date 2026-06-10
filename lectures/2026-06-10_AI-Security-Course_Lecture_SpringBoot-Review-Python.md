# 2026-06-10 강의 정리 — Spring Boot 리뷰 & Python 기초

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 지난 시간 리뷰 (MVC 패턴, RESTful), Python 기초 & 데코레이터
> **관련 실습 파일**:
> - `../practice/2026-06-10_AI-Security-Course_Practice_VibeCoding-SpringBtWeb4.md`
> - `../practice/2026-06-10_AI-Security-Course_Practice_Python-Basics.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-10

- 오늘 강사님이 강조한 것: "중요한 건 Spring이 아니라 MVC 패턴이다." 프레임워크는 바뀌어도 패턴은 남는다. Django를 배울 때도, Spring을 배울 때도, 결국 같은 MVC 구조였다.
- 바이브 코딩에서 발생한 오류의 근본 원인은 Skill 문서에 환경 정보가 불정확했던 것이다. Oracle 12c+ 기준으로 스펙을 썼는데 실제 환경은 11g였고, Eclipse의 `-parameters` 미적용 특성도 초기에 고려하지 않았다. AI는 문서에 적힌 대로만 만든다. 문서가 틀리면 코드도 틀린다.
- `CLAUDE.md`(공통 환경/원칙)와 `skill.md`(개별 작업 스펙)를 분리하는 구조가 인상적이었다. 바이브 코딩을 팀으로 하면 이 문서를 맞추는 것 자체가 큰 작업이겠다 싶었다. 강사님이 제공해주신 양식을 기반으로 내 것으로 커스터마이징해봐야겠다.
- Python basic은 SSAFY 때 다 해본 내용이라 감흥이 없었다. 그런데 데코레이터는 달랐다. API 요청 시 로그인 여부 확인 같은 곳에 쓰면 딱이겠다는 생각이 들었다. AOP랑 같은 발상이다 — Spring에서 `@Around`로 메서드 앞뒤에 공통 로직을 끼워 넣는 것과 Python 데코레이터가 하는 일이 본질적으로 같다.
- 바이브 코딩에서 책임 문제도 짚었다. AI가 생성한 코드의 품질 보증, 오버스펙 여부 판단은 결국 사람이 해야 한다. 도구가 빠를수록 판단하는 사람의 책임이 더 커진다.

---

## 1. 지난 시간 리뷰 — Spring MVC & RESTful

### 핵심 개념

- **중요한 건 Spring이 아니라 MVC 패턴** — 프레임워크는 바뀌어도 패턴은 남는다
- 바이브 코딩으로 발생하는 문제: 오버스펙, 품질 보증, 환경 불일치

### 상세 내용

**MVC 패턴 재정리**

```
Client
  ↕ HTTP
Controller  — 요청 수신, 응답 반환 (라우팅)
  ↕
Service     — 비즈니스 로직 (핵심 책임)
  ↕
Repository  — 데이터 접근 (DB 쿼리)
  ↕
DB
```

- Django: `urls.py → views.py → models.py`
- Spring: `@Controller → @Service → @Repository(@Mapper)`
- 구조가 같다. 이름과 문법만 다를 뿐.

**RESTful 표준**

| HTTP Method | 의미 | 성공 상태코드 |
|---|---|---|
| GET | 조회 | 200 OK |
| POST | 생성 | 201 Created |
| PUT | 수정 | 200 OK |
| DELETE | 삭제 | 204 No Content |
| — | 없는 리소스 | 404 Not Found |
| — | 중복 | 409 Conflict |
| — | 입력값 오류 | 400 Bad Request |

---

## 2. Python 기초 리뷰

### 핵심 개념

- SSAFY에서 다룬 내용 복습 수준 — 자료구조, 제어문, 함수, 클래스, lambda
- 새로웠던 것: `decimal` 모듈을 이용한 부동소수점 정밀도 처리, 타입 힌트(type hint)

### 상세 내용

**부동소수점 문제 & decimal 모듈**
```python
# 일반 float 연산 — 부동소수점 오차 발생
d = 0.1 + 0.1 + 0.1 - 0.3
print(d)  # 5.551115123125783e-17 (0이 아님)

# decimal 모듈 — 정밀한 소수 연산
from decimal import Decimal
e = Decimal('0.1') + Decimal('0.1') + Decimal('0.1') - Decimal('0.3')
print(e)  # 0.0 (정확)
```

**Python `int`는 오버플로우가 없다**
```python
import sys
a = sys.maxsize          # 플랫폼 최대 정수 (list 등 자료구조의 크기 한계)
b = sys.maxsize + 100    # 오버플로우 없음 — 메모리 허용 한도 내 무한 확장
```

**타입 힌트 (Type Hint)**
```python
a: str = 'abcde'
b: int = 1
test: list[int] = [1, 2, 3]
# 강제 타입 검사 아님 — 가독성과 IDE 지원 목적
```

**함수 패킹/언패킹**
```python
def param10(a, *b, **c):        # *b: 위치 인자 패킹(리스트), **c: 키워드 인자 패킹(딕셔너리)
    for i in b:
        print('{} ({}{})'.format(a*i, i, c['단위']))

param10('', 7, 20, 10, 3, 15, 단위='대')
param10('#', *[7, 20, 10, 3, 15], **{'단위': '대'})  # 언패킹 전달
```

**lambda와 고차함수**
```python
def fn_calc(a, b, fn):
    print(fn(a, b))

fn_calc(2, 3, lambda x, y: x * y)  # 익명 함수를 인자로 전달
```

---

## 3. Python 데코레이터

### 핵심 개념

- 함수를 인자로 받아 새로운 함수를 반환하는 고차함수 패턴
- 기존 함수 코드 수정 없이 공통 로직(로깅, 인증, 타이밍 등)을 앞뒤에 끼워 넣는 방식
- **Spring AOP의 `@Around` Advice와 본질적으로 같은 발상**

### 상세 내용

```python
from functools import wraps

def log_decorator(fn):
    @wraps(fn)          # fn의 __name__, __doc__ 등 메타데이터를 wrapper에 복사
    def wrapper_fn():
        print('=' * 10)
        print(fn.__name__)  # 원본 함수 이름 출력
        fn()                # 원본 함수 실행
        print('=' * 10)
    return wrapper_fn

@log_decorator  # msg = log_decorator(msg) 와 동일
def msg():
    print('Hello World!!!')

msg()
```

**`@wraps`가 필요한 이유**
- `@wraps` 없이 데코레이터를 쓰면 `msg.__name__`이 `wrapper_fn`으로 바뀜
- `@wraps(fn)`을 붙이면 원본 함수의 이름·docstring이 유지됨 → 디버깅, 로깅 시 혼란 방지

**활용 예시 (발상)**
```python
# API 요청 시 로그인 여부 확인 — 데코레이터로 공통화
def login_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        if not is_logged_in():
            return redirect('/login')
        return fn(*args, **kwargs)
    return wrapper

@login_required
def my_page():
    ...
```

**Spring AOP `@Around`와 비교**

| | Python 데코레이터 | Spring AOP `@Around` |
|---|---|---|
| 공통 로직 위치 | 함수 앞뒤 | 메서드 앞뒤 |
| 적용 방법 | `@데코레이터명` | `@Around("execution(...)")` |
| 주요 사용처 | 로깅, 인증, 타이밍 | 로깅, 트랜잭션, 성능 측정 |
| 핵심 발상 | 동일 — 기존 코드 수정 없이 횡단 관심사 분리 | |

*작성일: 2026-06-10 | 인공지능 사관학교 AI 보안 과정*
