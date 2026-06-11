# 2026-06-11 강의 정리 — Python 오류/모듈 & 데이터 분석 (NumPy · Pandas)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Python 오류 처리 · 모듈 시스템, 데이터 분석 개요(EDA), NumPy, Pandas
> **관련 실습 파일**: `../practice/2026-06-11_AI-Security-Course_Practice_Python-DataAnalysis.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-11

- API 요청·DB 조회에서는 예외처리가 필수다. 네트워크 장애, 타임아웃, 서버 오류는 언제든 발생할 수 있고 프로그램이 그냥 죽으면 안 된다.
- `if __name__ == '__main__'`은 모듈로 import될 때와 직접 실행될 때를 구분하는 장치다. 이걸 안 쓰면 import 시점에 실행하지 않아야 할 코드가 돌아간다.
- 확증적 데이터 분석(CDA)은 "선입견"이 문제다. 가설을 먼저 세우면 그 가설을 확인하는 방향으로만 데이터를 해석하게 된다. 탐색적 데이터 분석(EDA)은 그 반대 — 데이터 자체를 먼저 보고 귀납적으로 의미를 찾는다.
- NumPy의 속도 이점이 실감됐다. 100만 개 요소에 for 루프는 62ms, 리스트 컴프리헨션은 52ms, NumPy 벡터화 연산은 3.5ms. 약 15~18배 빠르다. C로 구현된 내부 구조와 벡터화 연산 덕분이다.
- NumPy 슬라이싱은 view를 반환한다는 게 핵심이다. 슬라이싱한 결과를 수정하면 원본도 바뀐다. `.copy()`나 팬시 인덱싱을 써야 독립적인 복사본이 생긴다.
- Pandas의 핵심은 다양한 index 지원이다. 날짜를 index로 쓰는 시계열 데이터 처리, 라벨 기반 접근이 NumPy ndarray와의 가장 큰 차이다.

---

## 1. Python 오류 처리

### 핵심 개념

Python의 오류는 발생 시점과 원인에 따라 세 가지로 구분된다.

| 오류 종류 | 발생 시점 | 예시 |
|---|---|---|
| 문법 오류 (SyntaxError) | 실행 전 파싱 단계 | `:` 누락, 들여쓰기 오류 |
| 런타임 오류 (RuntimeError) | 실행 중 | 0 나누기, 없는 인덱스 접근 |
| 예외 상황 (Exception) | 실행 중 예측 가능한 상황 | 파일 없음, 네트워크 오류 |

### 상세 내용

**API 요청 · DB 조회에서 예외처리 필수**

```python
import requests

try:
    response = requests.get("https://api.example.com/data", timeout=5)
    response.raise_for_status()  # 4xx, 5xx 오류 시 예외 발생
    data = response.json()
except requests.exceptions.Timeout:
    print("요청 시간 초과")
except requests.exceptions.HTTPError as e:
    print(f"HTTP 오류: {e}")
except Exception as e:
    print(f"예상치 못한 오류: {e}")
```

- 네트워크 장애, 서버 다운, 타임아웃은 언제든 발생 가능 → try-except 없으면 프로그램이 그냥 죽음
- DB 조회도 동일 — 연결 실패, 쿼리 오류, 권한 오류 등을 반드시 처리

---

## 2. Python 모듈 시스템

### 핵심 개념

- 모듈: 함수, 변수, 클래스를 담은 `.py` 파일
- `import` 시 `__pycache__` 폴더에 CPython으로 컴파일된 `.pyc` 파일이 생성됨 → 재import 시 컴파일 단계 생략으로 속도 향상

### 상세 내용

**`__name__ == '__main__'` 패턴**

```python
# py_03_module.py
PI = 3.1415

def print_helloworld():
    print('Hello World')

print("외부에서 호출해도 출력됨")   # import 시에도 실행됨

if __name__ == '__main__':
    print_helloworld()
    print("외부에선 출력되지 않음") # import 시에는 실행 안 됨
```

- `__name__`: 파일을 직접 실행하면 `'__main__'`, import되면 모듈 이름(`'py_03_module'`)
- 테스트 코드나 실행 진입점을 이 블록 안에 넣어 모듈로 재사용할 때 부작용 방지

**import 방식 3가지**

```python
import py_03_module                    # 모듈 전체 import
py_03_module.print_helloworld()

from py_03_module import print_helloworld, PI  # 특정 이름만 import
print_helloworld()

import py_03_module as md              # 별칭(alias) 사용
md.print_helloworld()
```

---

## 3. 데이터 분석 개요

### 핵심 개념

**데이터 과학 진행 단계**
```
문제 정의 → 데이터 수집 → 전처리·데이터 분석 → 모델링 → 평가 및 공유
```

**확증적 데이터 분석(CDA) vs 탐색적 데이터 분석(EDA)**

| | CDA (Confirmatory) | EDA (Exploratory) |
|---|---|---|
| 방식 | 연역적 — 가설 먼저, 검증 나중 | 귀납적 — 데이터 먼저, 의미 나중 |
| 주요 도구 | 통계 검정 | 기술 통계 + 시각화 |
| 강점 | 문제가 명확하면 빠르고 효율적 | 예상치 못한 인사이트 발견 가능 |
| 약점 | **선입견**에 의한 편향 발생 위험 | 방향성 없이 시간이 오래 걸릴 수 있음 |

### 상세 내용

**탐색적 데이터 분석(EDA) 주요 절차**

1. 데이터 구조 파악 — `df.info()`, `df.describe()`, 시각화
2. 결측치 및 이상치 처리
   - 결측치(NaN): 제거 또는 보간
   - 이상치: 통계적으로 문제 있는 값 제거 또는 수정
3. 데이터 관계 분석

**데이터 타입 분류**

```
범주형 (Categorical)
  ├── 순서형: 수, 우, 미, 양, 가 (순서 있음)
  └── 명목형: Yes/No, 남/여 (순서 없음)

숫자형 (Numeric)
  ├── 연속형: 실수. 1.00001, 1.00002 ...
  └── 이산형: 정수. 1, 2, 3, 4, 5 ...
```

**데이터 관계 분석 방법**
- 범주형 × 범주형: 값의 개수, 분포
- 이산형 × 이산형: 상관계수
- 범주형 × 이산형: 카테고리별 통계치

---

## 4. NumPy

### 핵심 개념

- 대용량 다차원 배열과 행렬 연산을 위한 Python 패키지
- 내부 구조를 C로 구현 → Python list보다 압도적으로 빠름
- **벡터화 연산(vectorized operation)**: 모든 요소에 루프 없이 한 번에 연산 적용
- 하나의 자료형만 요소로 가질 수 있음 (list와의 핵심 차이)
- Pandas, Matplotlib, scikit-learn 등의 기반 패키지

### 상세 내용

**성능 비교 (100만 개 요소 +1 연산)**

| 방식 | 실행 시간 |
|---|---|
| for 루프 | ~62ms |
| 리스트 컴프리헨션 | ~52ms |
| NumPy 벡터화 | **~3.5ms** |

**ndarray 핵심 속성**

```python
nd = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
nd.dtype   # int32
nd.shape   # (3, 3) — (행, 열)
nd.size    # 9 — 전체 요소 수
nd.sum()           # 45 — 전체 합
nd.sum(axis=0)     # [12 15 18] — 열 방향 합 (행을 따라 내려가며)
nd.sum(axis=1)     # [6 15 24]  — 행 방향 합 (열을 따라 가로로)
```

**슬라이싱은 view를 반환한다**

```python
na2 = np.array([1, 2, 3, 4, 5, 6, 7, 8])
na3 = na2[1:5]   # view (참조) — na3 수정 시 na2도 바뀜
na3[0] = 20
print(na2)       # [1 20 3 4 5 6 7 8] ← 원본 바뀜

na4 = na2[0:4].copy()  # 독립적인 복사본
na4[0] = 1
print(na2)       # 원본 그대로
```

**팬시 인덱싱은 복사본 반환**

```python
b = na2[[0, 1, 2]]  # 팬시 인덱싱 → 새로운 객체 생성
b[0] = 10
print(na2)       # 원본 변경 없음
```

**boolean 인덱싱**

```python
a = np.arange(10)
print(a > 7)           # [False ... True True] — 조건 배열 반환
print(a[a > 7])        # [8 9]
print(np.where(a % 2 == 0, '짝수', '홀수'))  # 조건부 값 교체
```

**reshape와 -1**

```python
na2 = np.array([1, 2, 3, 4, 5, 6, 7, 8])
na3 = na2.reshape(4, 2)      # (4행, 2열)
na3.reshape(-1, 8)            # (-1 = 자동 계산) → (1, 8)
na3.reshape(1, -1)            # → (1, 8)
```

---

## 5. Pandas

### 핵심 개념

- NumPy 기반의 데이터 분석 라이브러리
- **다양한 index 지원** — 정수, 라벨, 날짜(시계열) 인덱스
- CSV, Excel, DB 등 다양한 입출력 지원
- 주요 자료구조: `Series`(1차원), `DataFrame`(2차원)

### 상세 내용

**Series vs DataFrame**

| | Series | DataFrame |
|---|---|---|
| 차원 | 1차원 | 2차원 |
| 구조 | index + values | index + columns + values |
| 반환 조건 | 단일 행/열 선택 시 | 다중 행/열 선택 시 |

**DataFrame 생성**

```python
df = pd.DataFrame(
    [[10, 20, 30, 40], [50, 60, 70, 80], [90, 100, 110, 120]],
    columns=list('ABCD'),
    index=pd.date_range(start='2019-01-01', periods=3)  # 날짜 인덱스
)
```

**loc vs iloc**

| | `loc` | `iloc` |
|---|---|---|
| 기준 | 라벨(이름) | 정수 인덱스(번호) |
| 범위 | 끝 포함 | 끝 미포함 |
| 예시 | `df.loc['2019-01-01', 'A']` | `df.iloc[0, 0]` |

**apply — 함수 적용**

```python
# Series에 함수 적용
df['A'].apply(lambda n: n + 1 if n > 20 else n)

# DataFrame 전체에 함수 적용 (열 단위)
df.apply(lambda n: n / 10)

# 전체 요소에 함수 적용
df.applymap(to_int)   # pandas 2.1 이후 df.map(to_int)
```

**to_sql — DB로 내보내기**

```python
df.to_sql   # DataFrame → DB 테이블로 저장 가능
            # SQLAlchemy 엔진과 함께 사용
```

---

## 아직 찾아보지 않은 것

- NumPy `axis=0`이 "행 방향"인지 "열 방향"인지 항상 헷갈림 — 직관적으로 외우는 방법

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| Pandas `loc` 슬라이싱에서 끝이 포함되는 이유 | 2026-06-11 | `pandas`의 `loc` 슬라이싱은 **레이블(label) 기반 인덱싱**이기 때문에, 일반적인 Python 슬라이싱과 달리 **끝값(stop)이 포함**<br />레이블 기반 슬라이싱에서는 다음 레이블이 무엇인지 항상 알 수 없기 때문 | 검색 |

---
*작성일: 2026-06-11 | 인공지능 사관학교 AI 보안 과정*
