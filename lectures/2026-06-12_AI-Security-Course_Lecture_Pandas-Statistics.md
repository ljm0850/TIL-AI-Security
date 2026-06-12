# 2026-06-12 강의 정리 — Pandas 심화 · 통계 · 시각화 입문

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Pandas 그룹·shift·rolling·외부 데이터·DB 연동, 기술 통계, 인코딩, Matplotlib 입문
> **관련 실습 파일**: `../practice/2026-06-12_AI-Security-Course_Practice_Pandas-Statistics.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-12

- Python 자료구조 처음 다룰 때처럼, 데이터를 원하는 대로 조작하는 게 처음엔 힘들다. 근데 그 단계가 지나면 갑자기 손에 익는다. 지금이 그 직전이라는 느낌이 든다.
- 통계를 모르고 만든 분석 결과물은 의미 없는 경우가 많고, 문제가 생기면 책임도 져야 한다. 숫자가 나왔다고 다 맞는 게 아니다. 결과를 해석할 수 있어야 한다.
- 결측치 처리는 단순한 기술 문제가 아니다. 0으로 채울지, 이전 값으로 채울지, 아예 제거할지는 데이터의 맥락에 따라 다르다. 성적 데이터에서 결석으로 인한 NaN과 기록 누락으로 인한 NaN은 처리 방식이 달라야 한다.
- 라벨 인코딩(`pd.factorize`)은 범주가 많아질수록 머신러닝에서 문제가 생긴다. 숫자의 크기가 의미 없는 순서를 암시하기 때문이다. 그래서 One Hot 인코딩(`pd.get_dummies`)이 필요하다 — 자연어 처리에서 단어 벡터화의 원리가 여기서 나온다.
- `df.shift()` + 증가량/증가율 계산 패턴이 실용적이었다. 주가 분석, 방문자 추이 같은 시계열 데이터에서 전일 대비 변화량을 구하는 방식이 딱 이것이다.
- `rolling(n).mean()`이 슬라이딩 윈도우 알고리즘을 생각하니 바로 이해됐다. 이동 평균선 같은 개념이 코드로 이렇게 간단하게 구현된다.

---

## 1. Pandas 그룹 연산 (groupby)

### 핵심 개념

- SQL의 `GROUP BY`와 동일한 발상 — 특정 열 기준으로 데이터를 묶어 집계
- 단일 키, 다중 키 모두 가능

### 상세 내용

```python
# 단일 키 그룹
sg1 = df['이용객'].groupby(df['공항'])
sg1.size()   # 그룹별 행 수
sg1.sum()    # 그룹별 합
sg1.mean()   # 그룹별 평균

# DataFrame 전체 그룹
df1 = df.groupby(df['공항']).sum()

# 다중 키 그룹
df1 = df.groupby(['공항', '연도']).sum()
df1 = df.groupby(['공항', '연도'])['이용객'].sum()['인천']  # 특정 공항만

# 특정 그룹 추출
df1 = df.groupby(['공항', '연도']).get_group(('인천', '2020'))
```

---

## 2. 시계열 데이터 조작 — shift · rolling

### 핵심 개념

- `shift(n)`: 데이터를 n행만큼 밀어냄 → 전일/전주 값과 비교 시 유용
- `rolling(n)`: n개 윈도우로 슬라이딩 집계 (이동 평균 등)

### 상세 내용

**shift — 전일 대비 증가량·증가율 계산**

```python
df = pd.DataFrame({'visitor': np.random.randint(1, 100, size=10)},
                  index=pd.date_range('2020-01-01', periods=10))

df['전일'] = df.shift(1, fill_value=0)    # 1행 뒤로 밀고 빈 값은 0으로 채움
df['증가량'] = df['visitor'] - df['전일']
df['증가율'] = ((df['증가량'] / df['전일']) * 100).round(1)
```

- 주가, 방문자 수, 매출 등 시계열 데이터에서 전일 대비 변화 분석에 바로 적용 가능

**rolling — 이동 평균 (슬라이딩 윈도우)**

```python
df = pd.DataFrame({'A': np.arange(5)},
                  index=pd.date_range('2020-01-01', periods=5))

df.rolling(2).mean()   # 2개 윈도우 이동 평균
df.rolling(3).mean()   # 3개 윈도우 이동 평균

import matplotlib.pyplot as plt
plt.plot(df.rolling(3).mean())
plt.show()
```

- `rolling(n)`: n개씩 묶어서 집계하는 슬라이딩 윈도우 알고리즘
- 주가 이동 평균선, 방문자 추세 완화 등에 사용

---

## 3. 외부 데이터 입출력

### 핵심 개념

- Pandas는 CSV, Excel, TXT, DB(SQL) 등 다양한 소스를 읽고 쓸 수 있음
- `read_csv` 시 `converters`로 읽는 시점에 타입 변환 가능

### 상세 내용

**파일 읽기**

```python
pd.read_csv('./datasets/airport_month.txt', delimiter='\s+')  # 공백 구분
pd.read_csv('./datasets/airport_month.csv')
pd.read_excel('./datasets/airport_month.xlsx')
```

**읽기 시점에 타입 변환 — converters**

```python
df = pd.read_csv('./datasets/지점별매출.csv', index_col='일자',
                 dtype={'D지점': int},
                 converters={
                     'A지점': convert_int,
                     'B지점': convert_int,
                     'C지점': lambda x: int(x.replace(',', '')),
                     '운영': lambda x: np.where(x == '직영', 1, 0)  # 범주 → 숫자
                 })
```

- `dtype`: 단순 타입 지정
- `converters`: 열별로 변환 함수 지정 — 콤마 제거, 범주 인코딩 등 복잡한 변환에 사용

**melt — wide → long 변환**

```python
# 로또 당첨 번호: 열로 펼쳐진 데이터를 행으로 녹임
df1 = pd.melt(df, id_vars=['회차'],
              value_vars=['1번', '2번', '3번', '4번', '5번', '6번'],
              var_name='추첨 순서', value_name='당첨번호')

s = df1['당첨번호'].value_counts()  # 번호별 빈도수
s.sort_index().to_excel('로또당첨번호빈도수.xlsx', header=False)
```

**DB 연동 — SQLAlchemy + Pandas**

```python
from sqlalchemy import create_engine

engine = create_engine("sqlite+pysqlite:///labs.db", echo=True, future=True)
# Oracle: create_engine("oracle+oracledb://SCOTT:TIGER@localhost:1521/xe")

# DB → DataFrame
sql = 'SELECT * FROM t_airport_month'
df = pd.read_sql(sql, engine, index_col='reg_month')

# DataFrame → DB
df.to_sql(name='t_airport_month', con=engine, if_exists='append')
```

- `if_exists`: `'replace'`(테이블 재생성), `'append'`(행 추가), `'fail'`(오류)

---

## 4. 기술 통계

### 핵심 개념

- 데이터의 분포와 특성을 수치로 요약하는 방법
- **통계를 모르고 만든 결과물은 의미 없는 경우가 많다** — 수치가 나왔다고 해석이 맞는 게 아님

### 상세 내용

**주요 통계 메서드**

```python
df1.count()    # 유효값 개수 (NaN 제외)
df1.sum()      # 합계
df1.mean()     # 평균
df1.var()      # 분산
df1.std()      # 표준편차
df1.max()      # 최대값
df1.min()      # 최소값
df1.median()   # 중앙값
df1.value_counts()          # 값별 빈도수
df1['물리'].value_counts()  # 특정 열 빈도수
df1.describe()              # 주요 통계 한눈에 (count, mean, std, min, 25%, 50%, 75%, max)
df1.corr()                  # 상관계수 행렬
```

**결측치 처리**

```python
df1.fillna(0, inplace=True)   # NaN → 0으로 채움
# 기타: fillna(method='ffill') — 이전 값으로 채움
#       dropna() — NaN 행/열 제거
```

- 결측치를 어떻게 처리할지는 **데이터의 맥락**에 따라 다름
  - 성적 데이터에서 결석(=0점)인지, 기록 누락인지에 따라 처리 방식이 달라져야 함

---

## 5. 범주형 데이터 인코딩

### 핵심 개념

- 머신러닝 모델은 숫자만 입력받을 수 있음 → 범주형 데이터를 숫자로 변환해야 함
- 두 가지 방법: 라벨 인코딩, One Hot 인코딩

### 상세 내용

**라벨 인코딩 (Label Encoding) — `pd.factorize`**

```python
codes, uniques = pd.factorize(df_class['class'])
df_class['label_encoding'] = codes
# First → 0, Second → 1, Third → 2 (또는 등장 순서대로)
```

- 문제점: 숫자 크기에 순서 의미가 생김 → 모델이 "Third(2)가 First(0)보다 더 좋다"고 오해할 수 있음
- 범주가 많을수록 이 문제가 심해짐

**One Hot 인코딩 — `pd.get_dummies`**

```python
df_class2 = pd.get_dummies(df_class, columns=['class'])
# class_First | class_Second | class_Third 열이 각각 생성됨 (0 또는 1)
```

- 각 범주를 독립적인 열로 분리 → 크기 관계 없음
- 자연어 처리(NLP)에서 단어 벡터화의 원리

**언제 무엇을 쓰나**

| | 라벨 인코딩 | One Hot 인코딩 |
|---|---|---|
| 적합한 경우 | 순서 있는 범주 (수/우/미/양/가) | 순서 없는 범주 (등급, 성별, 공항명) |
| 문제점 | 크기 순서 암시 | 범주 수만큼 열이 늘어남 (고차원 문제) |
| 자연어 | — | 단어 → 벡터 변환의 기원 |

---

## 6. Matplotlib 시각화 입문

### 핵심 개념

- Python 대표 시각화 라이브러리
- 오늘은 기본 선 그래프만 확인 (다음 시간 이어서 진행 예정)

### 상세 내용

```python
import matplotlib.pyplot as plt

x = [0, 1, 2, 3, 4]
y = [0, 4, 9, 16, 20]

plt.plot(x, y)
plt.show()
```

*작성일: 2026-06-12 | 인공지능 사관학교 AI 보안 과정*
