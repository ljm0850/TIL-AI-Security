# 2026-06-15 실습 — 머신러닝 기초 & 사이킷런 (붓꽃 분류 / 타이타닉)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사 코드 따라치기 + 일부 변형)
> **관련 강의 파일**: `../lectures/2026-06-15_AI-Security-Course_Lecture_Machine-Learning.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-15

- sklearn을 처음 써봤는데 진짜로 **`fit` → `predict` → `accuracy_score` 3단계**가 끝이었다. 알고리즘(DecisionTree)을 바꿔도 이 골격이 안 바뀐다는 걸 손으로 확인하니, 강의에서 들은 "일관된 API"가 무슨 말인지 체감됐다.
- 붓꽃은 거의 손 안 대도 정확도가 잘 나왔는데, **타이타닉은 시작하자마자 결측치/범주형 때문에 전처리가 일**이었다. "ML은 모델보다 데이터(전처리)가 8할"이라는 말이 왜 나오는지 알았다. 붓꽃 = 깨끗한 정형 데이터, 타이타닉 = 현실 데이터.
- `ml_00`에서 **유클리드 거리로 "가장 가까운 후보 고르기"를 직접 함수로 짜봤는데, 이게 결국 KNN(최근접 이웃)의 직관**이었다. 라이브러리 쓰기 전에 거리 계산을 손으로 한 번 해본 게 분류의 감을 잡는 데 도움이 됐다.
- `value_counts()`로 정답 비율을 먼저 확인하는 습관 — **정답이 한쪽으로 쏠려 있으면(불균형) 정확도 숫자가 거짓말을 할 수 있으니** 학습 전에 분포부터 보는 게 맞다.

---

## 실습 개요
- **목표**: 사이킷런으로 지도학습(분류)의 전체 흐름을 직접 돌려보기.
- **방법**:
  1. `ml_00` — 유클리드 거리 함수로 분류 직관 잡기 (라이브러리 없이)
  2. `ml_01` — 붓꽃(iris) 품종 분류 (DecisionTree, **완주**)
  3. `ml_02` — 타이타닉 생존 예측 전처리 (**결측치 진단까지, 이후 내일로 미뤄짐**)

---

## 진행 과정

### Step 1. 거리 기반 분류 직관 (ml_00)
라이브러리 없이 유클리드 거리만으로 "가장 가까운 후보"를 고르는 연습.

```python
def distance(x, x1):
    n  = np.array(x)
    n1 = np.array(x1)
    diff = n - n1
    return np.sqrt(np.sum(diff**2))   # 벡터 뺄셈 → 제곱합 → 제곱근

# 위/경도로 가장 가까운 공항 찾기
locs = [[37.46,126.45],[36.43,127.30],[33.30,126.29]]
loc  = [35.00, 126.55]
err  = [distance(n, loc) for n in locs]   # 가장 작은 값이 정답 후보
```
- 같은 함수로 점수 벡터 `[2,2,5]`에 가장 가까운 레벨도 찾음. → **거리가 작을수록 비슷하다**는 분류의 기본 직관.

### Step 2. 붓꽃 분류 — 사이킷런 전체 흐름 (ml_01, 완주)
로딩 → DataFrame화 → EDA → 분할 → 학습 → 평가 → 적용 → 저장/로드.

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.25, random_state=15)

dt_clf = DecisionTreeClassifier(random_state=15)
dt_clf.fit(X_train, y_train)          # fit = 알고리즘을 데이터에 맞게 조정 => 모델
y_pred = dt_clf.predict(X_test)
accuracy_score(y_test, y_pred)
```

**강사 코드와 다르게 한 부분 (내 변형):**
- `random_state=15` (강의자료는 `5`) — seed만 바꿔도 train/test 분할이 달라져 정확도가 조금 흔들리는 걸 확인.
- `plt.scatter`에서 `cmap`/`s` 옵션을 뺀 기본형으로 작도.
- 코드 옆에 내 말로 주석을 달아둠: `"라벨을 알고 있으므로 지도학습"`, `"fit = 알고리즘을 데이터에 맞게 조정 => 모델"`.
- 모델을 `'붓꽃 모델.pkl'`로 `joblib.dump` → 다시 `joblib.load`해서 같은 예측이 나오는지 확인.

### Step 3. 타이타닉 전처리 (ml_02, 진행 중 ⏸)
```python
df['family_size'] = df['sibsp'] + df['parch']     # 파생변수
df1 = df.drop(['sibsp','parch','fare','class',
               'adult_male','deck','embark_town','alive'], axis=1)

df1.isnull().sum() / len(df1)     # 결측치 비율
sns.heatmap(df1.isnull())         # 결측 위치 시각화
plt.show()
```
- 강사 흐름보다 먼저 `deck`까지 초기 drop에 포함시킴 (강의에선 결측 40% 확인 후 따로 제거).
- **여기까지 하고 시간이 다 됨.** `fillna`(age 중앙값 / embarked 최빈값), `factorize`(범주형 인코딩), `corr` 히트맵, 학습/평가는 **내일**.

---

## 결과 / 산출물
- 붓꽃 분류 모델(DecisionTree) — 학습/평가/저장(`.pkl`)/로드까지 **완주**.
- 타이타닉 전처리 절반 — 결측치 진단(heatmap)까지.
- 거리 기반 분류 직관 함수(`ml_00`).

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 붓꽃 분류 `fit→predict→평가→저장/로드` 전체 완주 | 타이타닉 전처리 시간 부족으로 중단(결측치 확인까지) |
| 거리 함수를 직접 짜서 분류 직관 확보 | `random_state`에 따른 정확도 변동의 원인은 아직 직관만 있음 |
| 강사 코드와 내 변형을 구분해 기록 | |

---

## 다음에 이어서 할 것
- 타이타닉 전처리 마무리: `age` 결측 → 중앙값, `embarked` 결측 → 최빈값(C) `fillna`.
- 범주형 인코딩: `pd.factorize`로 `sex`/`embarked`/`who`/`alone` 변환.
- `corr()` 히트맵으로 `survived`와의 상관 확인 → **DecisionTree / LogisticRegression 두 모델 학습·비교**.

## 다음에 파고들 만한 질문
> 실습하다 생긴 곁가지 궁금증만.

- `train_test_split`의 `random_state`(seed)에 따라 정확도가 출렁이는데, "이 모델 정확도"를 숫자 하나로 믿어도 되나? (교차검증이 이 문제를 푸는 건지 궁금)

---
*작성일: 2026-06-15 | 인공지능 사관학교 AI 보안 과정*
