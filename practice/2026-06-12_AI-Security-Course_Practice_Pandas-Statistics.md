# 2026-06-12 실습 — Pandas 심화 · 통계 · 시각화 입문

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-12_AI-Security-Course_Lecture_Pandas-Statistics.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-12

- 처음 Python 자료구조 다룰 때처럼, 원하는 대로 데이터를 조작하는 게 아직 힘들다. 메서드가 너무 많아서 어느 걸 써야 할지 바로 떠오르지 않는다. 근데 그 단계가 지나면 갑자기 손에 익는다는 걸 경험으로 알기 때문에 일단 많이 쳐보는 수밖에 없다.
- `shift(1)`로 전일 값을 만들고 증가량·증가율까지 이어지는 흐름이 깔끔했다. 주가 차트나 방문자 추이 분석에서 쓰는 패턴이 이거였구나 싶었다.
- `rolling(n).mean()`을 보고 슬라이딩 윈도우라는 게 바로 연결됐다. 알고리즘에서 봤던 개념이 데이터 분석에서 이동 평균으로 쓰인다.
- `read_csv`에서 `converters`로 읽는 시점에 타입 변환까지 끝낼 수 있다는 게 실용적이었다. 콤마가 포함된 매출 데이터를 따로 `apply`로 처리하지 않아도 된다.
- `pd.melt`로 wide 형태 로또 데이터를 long 형태로 바꾸고 `value_counts()`로 빈도를 구하는 흐름이 인상적이었다. 데이터 구조를 먼저 적절하게 바꿔야 원하는 분석이 가능하다는 것.
- 라벨 인코딩과 One Hot 인코딩의 차이를 코드로 직접 보고 나서 왜 구분이 필요한지 이해됐다. 라벨 인코딩은 숫자 크기가 순서를 암시해서 모델이 오해할 수 있다.

---

## 실습 개요

- **목표**: Pandas 그룹·시계열·외부 데이터·통계·인코딩 전체 흐름 직접 실습
- **방법**: 강사 코드 기반으로 작성하며 출력 결과 확인 (`pd_03`, `pd_04`, `pd_05`, `plt_01`)

---

## 진행 과정

### Step 1. pd_03_응용.ipynb — 그룹 연산 · shift · rolling (강사 코드)

**groupby — 공항별 이용객 통계**

```python
df = pd.DataFrame({
    "공항": ["김포", "인천", "인천", "김포", "제주", "제주", "김해", "인천"],
    "연도": ["2020", "2020", "2019", "2018", "2020", "2019", "2020", "2020"],
    "이용객": [1444483, 182523, 214252, 244448, 1785615, 278561, 554583, 224252],
    "운항편수": [483, 623, 680, 428, 515, 571, 383, 452]
})

# 단일 키
sg1 = df['이용객'].groupby(df['공항'])
sg1.size()    # 공항별 행 수
sg1.sum()     # 공항별 합계
sg1.mean()    # 공항별 평균

# DataFrame 전체 그룹
df.groupby(df['공항']).sum()

# 다중 키 — 공항 + 연도 조합
df.groupby(['공항', '연도']).sum()
df.groupby(['공항', '연도'])['이용객'].sum()['인천']    # 인천만
df.groupby(['공항', '연도']).get_group(('인천', '2020'))  # 특정 그룹 추출
```

**shift + 증가량·증가율**

```python
df = pd.DataFrame({'visitor': np.random.randint(1, 100, size=10)},
                  index=pd.date_range('2020-01-01', periods=10))

df['전일'] = df.shift(1, fill_value=0)
df['증가량'] = df['visitor'] - df['전일']
df['증가율'] = ((df['증가량'] / df['전일']) * 100).round(1)
```

**rolling — 이동 평균 (슬라이딩 윈도우)**

```python
df = pd.DataFrame({'A': np.arange(5)},
                  index=pd.date_range('2020-01-01', periods=5))

df.rolling(2).mean()    # 2개 단위 이동 평균
df.rolling(3).mean()    # 3개 단위 이동 평균

# 시각화
import matplotlib.pyplot as plt
plt.plot(df.rolling(3).mean())
plt.show()
```

---

### Step 2. pd_04_외부소스.ipynb — 파일·DB 입출력 (강사 코드)

**다양한 파일 형식 읽기**

```python
pd.read_csv('./datasets/airport_month.txt', delimiter='\s+')   # 공백 구분
pd.read_csv('./datasets/airport_month.csv')
pd.read_excel('./datasets/airport_month.xlsx')
```

**read_csv converters — 읽기 시점에 타입 변환**

```python
def convert_int(p: str) -> int:
    return int(p.replace(',', ''))

df = pd.read_csv('./datasets/지점별매출.csv', index_col='일자',
                 dtype={'D지점': int},
                 converters={
                     'A지점': convert_int,
                     'B지점': convert_int,
                     'C지점': lambda x: int(x.replace(',', '')),
                     '운영': lambda x: np.where(x == '직영', 1, 0)
                 })

df['일별 총계'] = df.drop('운영', axis=1).sum(axis=1)
df.loc['지점별 총계'] = df.sum()
```

**melt — wide → long 변환 (로또 당첨번호 빈도 분석)**

```python
df = pd.read_csv('./datasets/로또당첨번호_수정.csv')

# wide: 각 회차의 당첨번호가 1번~6번 열에 가로로 펼쳐져 있음
# → long: (회차, 추첨순서, 당첨번호) 형태로 녹임
df1 = pd.melt(df,
              id_vars=['회차'],
              value_vars=['1번', '2번', '3번', '4번', '5번', '6번'],
              var_name='추첨 순서',
              value_name='당첨번호')

# 번호별 빈도수
s = df1['당첨번호'].value_counts().sort_index()
s.to_excel('로또당첨번호빈도수.xlsx', header=False)
```

**전치(Transpose)로 행·열 뒤집기**

```python
df2 = df.T                   # 행↔열 전치
df2.columns = df2.loc['일자']  # 전치 후 날짜를 열 이름으로 설정
df2.drop(index=['일자'], inplace=True)
```

**DB 연동 — SQLAlchemy**

```python
from sqlalchemy import create_engine

# SQLite (로컬 테스트)
engine = create_engine("sqlite+pysqlite:///labs.db", echo=True, future=True)
# Oracle: create_engine("oracle+oracledb://SCOTT:TIGER@localhost:1521/xe")

# DB → DataFrame
df = pd.read_sql('SELECT * FROM t_airport_month', engine, index_col='reg_month')

# DataFrame → DB
df.to_sql(name='t_airport_month', con=engine, if_exists='append')
```

---

### Step 3. pd_05_통계.ipynb — 기술 통계 · 인코딩 (강사 코드)

**성적 데이터 기술 통계**

```python
df = pd.read_csv('./datasets/성적통계.csv', encoding='euc-kr')  # python 기본 utf-8 → 한글 파일은 euc-kr 지정

df1 = df.drop(['학년', '반', '번호', '성별'], axis=1)
df1.set_index('성명', inplace=True)
df1.fillna(0, inplace=True)   # 결측치 → 0으로 채움
df1 = df1.astype('int')

# 기술 통계
print(df1.count(), df1.sum(), df1.mean())
print(df1.var(), df1.std())    # 분산, 표준편차
print(df1.max(), df1.min())
print(df1.median())
df1['물리'].value_counts()     # 물리 점수 빈도수

df1.describe()   # count / mean / std / min / 25% / 50% / 75% / max 한눈에
df1.corr()       # 과목 간 상관계수 행렬
```

**타이타닉 데이터 — 범주형 인코딩**

```python
df = pd.read_csv('./datasets/seaborn-data/titanic.csv')

df_class = df[['class']].copy()
print(df_class.value_counts())   # First: 216 / Second: 184 / Third: 491

# 라벨 인코딩 — 범주 → 정수
codes, uniques = pd.factorize(df_class['class'])
df_class['label_encoding'] = codes
# First→0, Second→1, Third→2 (등장 순서대로)

# One Hot 인코딩 — 범주 → 독립적 열
df_class2 = pd.get_dummies(df_class, columns=['class'])
# class_First | class_Second | class_Third 열 생성 (0 또는 1)
```

---

### Step 4. plt_01_시각화-1.ipynb — Matplotlib 입문 (강사 코드)

```python
import matplotlib.pyplot as plt

x = [0, 1, 2, 3, 4]
y = [0, 4, 9, 16, 20]

plt.plot(x, y)
plt.show()
```

- 오늘은 기본 선 그래프만 확인. 시각화는 다음 시간에 이어서 진행 예정.

---

## 결과 / 산출물

- `pd_03_응용.ipynb` — groupby, shift, rolling 패턴 실습
- `pd_04_외부소스.ipynb` — CSV/Excel/DB 입출력, melt, converters 실습
- `pd_05_통계.ipynb` — 기술 통계, 결측치 처리, 라벨/One Hot 인코딩 실습
- `plt_01_시각화-1.ipynb` — 기본 선 그래프 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| shift + 증가량 + 증가율 흐름을 직접 만들어봄 | groupby 결과에서 MultiIndex 다루는 법이 아직 헷갈림 |
| converters로 읽기 시점 변환 패턴 이해 |  |
| 라벨 인코딩과 One Hot 인코딩의 차이를 코드로 확인 |  |

---

## 다음에 이어서 할 것

- Matplotlib 시각화 이어서 (선 그래프, 막대 그래프, 히스토그램 등)

---
*작성일: 2026-06-12 | 인공지능 사관학교 AI 보안 과정*
