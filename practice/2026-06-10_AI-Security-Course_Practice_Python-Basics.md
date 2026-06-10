# 2026-06-10 실습 — Python 기초 & 데코레이터

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-10_AI-Security-Course_Lecture_SpringBoot-Review-Python.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-10

- `py_01_basic.py`는 SSAFY 때 다 해봤던 내용이라 별다른 감흥이 없었다. 빨리 끝났으면 했다.
- 그런데 `py_02_etc.py`의 데코레이터는 달랐다. API 요청 시 로그인 여부 확인 같은 공통 로직을 함수마다 반복하지 않고 데코레이터 하나로 처리할 수 있겠다는 게 바로 보였다.
- 데코레이터를 보면서 Spring AOP가 떠올랐다. `@Around`로 메서드 앞뒤에 공통 로직을 끼워 넣는 것과 Python 데코레이터가 하는 일이 본질적으로 같다. 언어는 달라도 문제를 해결하는 방식은 수렴한다.
- `sys.maxsize`가 Python `int` 오버플로우 한계가 아니라 리스트 등 자료구조의 크기 한계라는 점이 새로웠다. Python `int`는 메모리 허용 범위 내에서 무한히 커질 수 있다.
- Django 팀 프로젝트 이후로 Python 가상환경을 쓸 일이 없었는데, 오늘 다시 언급이 나오면서 그 시절이 생각났다.

---

## 실습 개요

- **목표**: Python 기본 문법 복습, 데코레이터 패턴 이해
- **방법**: 강사 제공 예제 코드 작성 및 확인

---

## 진행 과정

### Step 1. py_01_basic.py — Python 기초 문법 (강사 코드)

**숫자형 & 부동소수점**
```python
import sys
import decimal

# sys.maxsize: 자료구조의 크기 한계 (int 오버플로우 한계가 아님)
a = sys.maxsize
b = sys.maxsize + 100  # 오버플로우 없음 — Python int는 메모리 허용 범위 내 무한 확장

# 부동소수점 오차
d = 0.1 + 0.1 + 0.1 - 0.3
print(d)  # 5.551115123125783e-17 (0이 아님)

# decimal 모듈로 정밀도 확보
e = decimal.Decimal('0.1') + decimal.Decimal('0.1') + decimal.Decimal('0.1') - decimal.Decimal('0.3')
print(e)  # 0.0
```

**문자열 슬라이싱**
```python
str1 = 'Hello Python'
print(str1[0:5])    # 'Hello'
print(str1[6:])     # 'Python'
print(str1[::-1])   # 'nohtyP olleH' (역순)
```

**자료구조**
```python
# 리스트 컴프리헨션
a = [1, 2, 3]
c = [v * 3 for v in a]          # [3, 6, 9]
d = [n for n in a if n % 2 == 0]  # [2] (짝수만)

# tuple unpacking / packing
a = (1, 2, 3)
b, c, d = a     # unpacking
e = (c, d)      # packing

# dictionary
a = {'name': 'TV', 'price': 1598000}
print(a.get('제조국', '없음'))  # 키 없을 때 기본값 반환
a['제조국'] = '대한민국'        # 없는 키 → 추가, 있는 키 → 수정

# dict(zip()) 패턴
keys = ("name", "price", "제조국")
values = ("Full HD TV", 1000000, "대한민국")
c = dict(zip(keys, values))

# set — 가변형, 중복 제거, hash 정렬
a = {2, 5, 3, 1, 4}
b = {5, 6, 7}
print(a | b)        # 합집합
print(a & b)        # 교집합
print(a - b)        # 차집합
print(a >= {2, 3})  # 부분집합 여부
```

**함수 — 패킹/언패킹**
```python
def param5(a, b, c):
    print(a, b, c)

param5(*[4, 5, 6])              # 리스트 언패킹
param5(**{'a': 1, 'b': 2, 'c': 3})  # 딕셔너리 언패킹

def param8(**args):             # 키워드 인자 패킹 → dict
    print(args['a'], args['b'])

def param9(*args):              # 위치 인자 패킹 → tuple
    for i in args: print(i)

def param10(a, *b, **c):        # 혼합
    for i in b:
        print('{} ({}{})'.format(a * i, i, c['단위']))

param10('', 7, 20, 10, 3, 15, 단위='대')
param10('#', *[7, 20, 10, 3, 15], **{'단위': '대'})
```

**클래스**
```python
class Calc():
    def __init__(self):
        self.a = 0
        self.b = 0

    def add(self):
        return self.a + self.b

    def multi(self):
        return self.a * self.b

calc = Calc()
calc.a, calc.b = 1, 2
print(calc.add(), calc.multi())  # 3, 2
```

**lambda**
```python
def fn_calc(a, b, fn):
    print(fn(a, b))

fn_calc(2, 3, lambda x, y: x * y)  # 6
```

---

### Step 2. py_02_etc.py — 데코레이터 (강사 코드)

```python
from functools import wraps

def log_decorator(fn):
    @wraps(fn)  # 원본 함수의 __name__, __doc__ 메타데이터 보존
    def wrapper_fn():
        print('=' * 10)
        print(fn.__name__)  # 'msg' 출력
        fn()                # 원본 함수 실행
        print('=' * 10)
    return wrapper_fn

@log_decorator   # msg = log_decorator(msg) 와 동일
def msg():
    print('Hello World!!!')

msg()
# 출력:
# ==========
# msg
# Hello World!!!
# ==========
```

**데코레이터 동작 원리**

```
@log_decorator       ┐
def msg(): ...       ┘  →  msg = log_decorator(msg)
                               ↓
                         wrapper_fn 반환
                               ↓
msg() 호출 시         →  wrapper_fn() 실행
                         (앞뒤 로직 + fn() 호출)
```

**`@wraps` 역할**
```python
# @wraps 없는 경우
print(msg.__name__)  # 'wrapper_fn' — 원본 이름 손실

# @wraps 있는 경우
print(msg.__name__)  # 'msg' — 원본 이름 유지
```

**Spring AOP `@Around`와 대응**

```java
// Spring AOP
@Around("execution(* app.labs..*(..))")
public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("==========");
    System.out.println(pjp.getSignature().getName());
    Object result = pjp.proceed();  // 원본 메서드 실행
    System.out.println("==========");
    return result;
}
```

```python
# Python 데코레이터 — 같은 발상
def log_decorator(fn):
    @wraps(fn)
    def wrapper_fn():
        print('==========')
        print(fn.__name__)
        fn()               # 원본 함수 실행
        print('==========')
    return wrapper_fn
```

- 기존 코드를 수정하지 않고 공통 관심사(로깅, 인증, 타이밍)를 분리하는 방식이 동일

---

## 결과 / 산출물

- `py_01_basic.py` — Python 기초 문법 전체 실습
- `py_02_etc.py` — 데코레이터 패턴 실습

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 데코레이터와 Spring AOP의 연결고리 파악 | basic 내용은 새로운 게 없어 집중도가 낮았음 |
| `@wraps` 필요성 이해 |  |
| `decimal` 모듈로 부동소수점 문제 해결 방법 확인 |  |

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| `functools.wraps` 없이 데코레이터를 쓸 때 생기는 문제 | 2026-06-10 | `__name__` 손실로 로그 혼란, `inspect.signature()` 깨짐, FastAPI 등 프레임워크 파라미터 분석 실패, `__wrapped__` 없어 원본 함수 접근 불가 | → `../deep-dive/2026-06-10_functools-wraps-deep-dive.md` |

---
*작성일: 2026-06-10 | 인공지능 사관학교 AI 보안 과정*
