# 2026-06-11 실습 — Python 모듈 & NumPy · Pandas 기초

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-11_AI-Security-Course_Lecture_Python-DataAnalysis.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-11

- NumPy 속도 비교 셀을 직접 실행해보고 나서 숫자로 실감했다. for 루프 62ms, 리스트 컴프리헨션 52ms, NumPy 3.5ms. 이전까지는 "빠르다"고만 알고 있었는데 실제 수치를 보니 체감이 달랐다.
- NumPy 슬라이싱이 view를 반환한다는 게 처음엔 직관적이지 않았다. `na3 = na2[1:5]`를 수정했더니 원본 `na2`가 바뀐 게 당연히 당황스러웠다. 반면 팬시 인덱싱 `na2[[0,1,2]]`는 복사본이라 원본이 안 바뀐다. 이 차이를 코드로 직접 확인하니 확실히 각인됐다.
- `np_01_기초.ipynb` 코드 안에 내가 직접 단 주석이 있었다: `# 이건 좀 신기하다, python에서 숫자는 0만 False, 나머지는 True 인게 생각났는데 전혀 다르다` — 부울린 인덱싱에서 True/False 배열로 요소를 선택하는 방식이 Python의 truthy/falsy와 다르다는 걸 처음 인지했다.
- Pandas `apply`와 `applymap`의 차이가 아직 헷갈린다. `apply`는 Series 또는 DataFrame 단위, `applymap`은 개별 요소 단위라고는 알겠는데 언제 어느 걸 써야 하는지 기준이 아직 불명확하다.
- `df.to_sql`을 강의에서 잠깐 보여줬는데, DataFrame을 DB 테이블로 바로 넣을 수 있다는 게 인상적이었다. RAG 서비스 토이 프로젝트에서 쓸 수 있겠다는 생각이 들었다.

---

## 실습 개요

- **목표**: Python 모듈 시스템 동작 확인, NumPy ndarray 기본 조작, Pandas DataFrame 생성·조작·타입 변환
- **방법**: 강사 코드 기반으로 직접 작성하며 출력 결과 확인

---

## 진행 과정

### Step 1. py_03_module.py + py_04_module_test.py — 모듈 시스템 (강사 코드)

```python
# py_03_module.py
PI = 3.1415

def print_helloworld():
    """Hello World를 출력하는 함수"""
    print('Hello World')

print("외부에서 호출해도 출력됨")    # import 시에도 실행됨
if __name__ == '__main__':
    print_helloworld()
    print("외부에선 출력되지 않음")  # import 시에는 실행 안 됨
```

```python
# py_04_module_test.py — import 3가지 방식
import py_03_module                         # 모듈 전체
py_03_module.print_helloworld()

from py_03_module import print_helloworld, PI  # 이름 직접 import
print_helloworld()

import py_03_module as md                   # 별칭
md.print_helloworld()
```

**확인된 동작**
- `import py_03_module` 시 `"외부에서 호출해도 출력됨"`이 출력됨 → `if __name__ == '__main__'` 블록 밖 코드는 실행됨
- `if __name__ == '__main__'` 블록 안 코드는 직접 실행 시에만 실행됨
- `__pycache__/py_03_module.cpython-312.pyc` 생성됨

---

### Step 2. np_01_기초.ipynb — NumPy 기초 (강사 코드 + 본인 주석)

**성능 비교 — 직접 실행해서 확인**

```python
%%time
# for 루프: CPU times total 62.5ms, Wall 61.7ms
a = list(range(1000000))
def test():
    for i in range(len(a)):
        a[i] += 1
test()

%time a = [n+1 for n in range(1000000)]
# 리스트 컴프리헨션: Wall 51.7ms

%%time
# NumPy 벡터화: Wall 3.48ms ← 약 15~18배 빠름
b = np.arange(1000000)
c = b + 1
```

**ndarray 생성과 기본 속성**

```python
data = [range(10)]
nd = np.array(data)
print(nd, type(nd), nd.dtype, nd.shape, nd.size, sep=' , ')
# [[0 1 2 3 4 5 6 7 8 9]] , <class 'numpy.ndarray'> , int32 , (1, 10) , 10
print(3 in nd)  # True

nd = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
print(nd.sum())          # 45
print(nd.sum(axis=0))    # [12 15 18] — 열 방향(↓)
print(nd.sum(axis=1))    # [ 6 15 24] — 행 방향(→)
```

**슬라이싱 = view, 팬시 인덱싱 = 복사본**

```python
na2 = np.array([1, 2, 3, 4, 5, 6, 7, 8])

# 슬라이싱 → view (원본 공유)
na3 = na2[1:5]
na3[0] = 20
print(na3, na2)  # [20 3 4 5] [1 20 3 4 5 6 7 8] ← 원본 바뀜

# .copy() → 독립적 복사본
na4 = na2[0:4].copy()
na4[0] = 1
print(na4, na2)  # [1 20 3 4] [1 20 3 4 5 6 7 8] ← 원본 그대로

# 팬시 인덱싱 → 복사본
b = na2[[0, 1, 2]]
b[0] = 10
print(b, na2)   # [10 20 3] [1 20 3 4 5 6 7 8] ← 원본 그대로
```

**부울린 인덱싱 — 처음엔 헷갈렸던 부분**

```python
b = na2[[0, 1, 2, 3, 4]]       # [1 2 3 4 5]
c = b[[True, False, True, False, True]]  # [1 3 5]
# Python truthy/falsy(0만 False)와 다름 — 위치별로 True인 것만 선택

print(b > 2)    # [False False  True  True  True]
d = b[b > 2]    # [3 4 5]
print(d.sum())  # 12
```

**유니버설 함수: np.where, np.select**

```python
a = np.arange(10)
b = np.where(a % 2 == 0, '짝수', '홀수')
# ['짝수' '홀수' '짝수' ... ]

con_list = [a < 3, a > 7]
con_exe = [a + 1, a * 2]
print(np.select(con_list, con_exe))
# 조건 리스트와 결과 리스트를 쌍으로 전달 — 해당 조건이면 해당 값, 아무것도 해당 없으면 0
```

---

### Step 3. pd_01_기초.ipynb — Pandas 기초 (강사 코드)

강의에서 시간 관계상 일부만 실습했음. 핵심 패턴 위주로 기록.

**DataFrame 생성 — 날짜 인덱스**

```python
df = pd.DataFrame(
    [[10, 20, 30, 40], [50, 60, 70, 80], [90, 100, 110, 120]],
    columns=list('ABCD'),
    index=pd.date_range(start='2019-01-01', periods=3)
)
```

**loc vs iloc**

```python
df.loc['2019-01-01']          # 라벨로 행 선택
df.loc['2019-01-01', 'A']     # 라벨로 행+열 선택
df.loc[:, 'A']                # 전체 행의 A열
df.loc['2019-01-01':'2019-01-03', 'A':'C']  # 범위 (끝 포함)

df.iloc[0]                    # 번호로 행 선택
df.iloc[0, 0]                 # 번호로 행+열 선택
df.iloc[0:2, 0:2]             # 범위 (끝 미포함)
```

---

### Step 4. pd_02_응용.ipynb — Pandas 응용 (강사 코드)

**조건부 값 수정**

```python
df1 = df.copy()

# A열의 값이 20보다 큰 행 전체를 +1
df1[df1['A'] > 20] = df1[df1['A'] > 20] + 1

# A열만 조건부 수정 — apply + lambda
df2 = df1['A'].apply(lambda n: n + 1 if n > 20 else n)
```

**문자열 숫자 → 정수 변환 (실무에서 자주 만나는 패턴)**

```python
# '1,000' 형태의 문자열 숫자가 담긴 DataFrame
df = pd.DataFrame({'A지점': ['1,000', '2,000', '3,000'],
                   'B지점': ['4,000', '5,000', '6,000'],
                   'C지점': ['7,000', '8,000', '9,000']},
                   index=pd.date_range(start='2020-01-01', periods=3, freq='D'))

def to_int(num: str) -> int:
    return int(num.replace(',', ''))

# 방법 1: 열별로 apply
df1['A지점'] = df1['A지점'].apply(to_int)

# 방법 2: DataFrame 전체에 map (pandas 2.1+)
# df1 = df1.map(to_int)
```

**행/열 합계 추가**

```python
df1['일별총계'] = df1.sum(axis=1)        # 행 방향 합 → 새 열
df1.loc['지점별총계'] = df1.sum(axis=0)  # 열 방향 합 → 새 행
```

결과:
```
              A지점    B지점    C지점   일별총계
2020-01-01   1000   4000   7000  12000
2020-01-02   2000   5000   8000  15000
2020-01-03   3000   6000   9000  18000
지점별총계    6000  15000  24000  45000
```

**to_sql — DataFrame을 DB로 내보내기**

```python
df1.to_sql  # SQLAlchemy 엔진과 함께 DataFrame → DB 테이블 저장 가능
```

---

## 결과 / 산출물

- `np_01_기초.ipynb` — NumPy ndarray 생성·조작·인덱싱·유니버설 함수 실습
- `pd_01_기초.ipynb` — Pandas Series·DataFrame 생성·loc/iloc 실습
- `pd_02_응용.ipynb` — 조건부 값 수정, 문자열 → 정수 변환, 합계 추가

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| NumPy view vs 복사본 차이를 코드로 직접 확인 | `apply` vs `applymap` vs `map` 구분이 아직 불명확 |
| 성능 비교 수치 직접 측정해서 체감 |  |

---
*작성일: 2026-06-11 | 인공지능 사관학교 AI 보안 과정*
