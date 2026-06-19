# 2026-06-19 실습 — 선형회귀 (보스턴 집값 예측)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-19_AI-Security-Course_Lecture_Regression.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-19

- **회귀는 "예측값"이 아니라 "회귀계수"를 읽을 때 진짜 이해된다.** 학습 후 `coef_`, `intercept_`로 식을 복원하니 `MEDV ≈ -0.58·LSTAT + 4.48·RM - 0.94·PTRATIO + 0.02·INDUS + 18.92`가 나왔다. 부호가 곧 방향이다 — RM(방 개수)은 +로 집값을 올리고, LSTAT(하위계층 비율)·PTRATIO(학생/교사 비율)는 -로 내린다. 강의에서 "기울기=가중치, 절편=편향"이라고 외운 게 실제 숫자로 해석되는 순간. (이 해석 가능성이 강의 인사이트에 적은 "선형 모델은 감사 가능"의 실체.)

- **상관계수로 feature를 고르고 들어간 게 분류 과제(펭귄)의 importance 접근과 같은 결**이었다. `corr()['MEDV'].abs().sort_values()`로 타겟과 상관 높은 LSTAT/RM/PTRATIO/INDUS를 골랐다. 분류에선 feature importance, 회귀에선 상관계수 — "어떤 feature가 결과를 설명하나"를 미리 보는 도구가 알고리즘 종류만 다를 뿐 역할은 같다.

- **R²와 MSE를 같이 봐야 한다.** R²=0.648은 "타겟 분산의 약 65%를 설명"이라 절대적으로 높진 않고, MSE=29.4(=$1000 단위 제곱)도 작지 않다. feature 4개만 골랐으니 당연한 결과지만, "정확도 한 숫자"가 아니라 설명력(R²) + 오차크기(MSE)를 함께 봐야 모델 상태가 보인다. (6/17 회귀 평가지표 강의가 여기서 쓰였다.)

---

## 실습 개요
- **목표**: 보스턴 주택 데이터로 집값 중앙값(`MEDV`)을 예측하는 다중 선형회귀 모델 제작·해석.
- **데이터**: `boston.csv` (506행, 14개 수치 컬럼, 결측치 없음).
- **방법**: 분포·상관 확인 → 상관 높은 feature 선정 → 학습 → R²/MSE 평가 → 회귀계수 해석.

---

## 진행 과정

### Step 1. 데이터 확인
- `info()`: 506행, 전부 float, 결측치 0.
- `describe()`: mean과 50%(중앙값) 차이로 분포의 치우침 확인.

### Step 2. feature 분석 (시각화 + 상관)
- `regplot`으로 주요 feature와 `MEDV`의 선형 경향 확인.
- `heatmap(df.corr())`으로 feature 간 비례/반비례 관계 확인.
- `df.corr()['MEDV'].abs().sort_values(ascending=False)` → 상관 상위: **LSTAT(0.74) > RM(0.70) > PTRATIO(0.51) > INDUS(0.48)** ...

### Step 3. feature 선정 · 데이터 분리
- `df[['LSTAT','RM','PTRATIO','INDUS','MEDV']]`로 추림.
- `histplot(MEDV, kde=True)`으로 정답 분포 확인.
- `X = [LSTAT,RM,PTRATIO,INDUS]`, `y = MEDV`로 분리.
- `train_test_split(test_size=0.25, random_state=15)` (※ `y.values.ravel()`로 1차원 변환).

### Step 4. 학습 · 평가
- `LinearRegression().fit(X_train, y_train)`.
- `predict(X_test)` 후 `r2_score`, `mean_squared_error`로 평가.

### Step 5. 회귀계수 해석 · 시각화
- `lr.coef_`, `lr.intercept_`로 식 복원.
- `X_test`의 LSTAT축 기준 `y_test` vs `y_pred` 산점도 비교.

---

## 결과 / 산출물
- **R² ≈ 0.648, MSE ≈ 29.44**
- **복원된 회귀식**: `MEDV ≈ -0.579·LSTAT + 4.478·RM - 0.943·PTRATIO + 0.022·INDUS + 18.917`
- 산점도: LSTAT가 낮을수록 예측·실제 집값이 함께 높아지는 경향 확인.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 상관계수로 feature를 근거 있게 선정 | INDUS는 계수(0.02)가 거의 0 — 굳이 넣을 필요 있었나 (정리 여지) |
| 회귀계수를 식으로 복원해 방향·영향력까지 해석 | R²=0.65로 설명력 보통 — feature를 4개로 줄인 trade-off |
| R²와 MSE를 함께 보고 모델 상태 판단 | 다항회귀/규제 모델과 성능 비교는 안 해봄 |



## 다음에 파고들 만한 질문
- (오늘은 따로 없음)

---
*작성일: 2026-06-19 | 인공지능 사관학교 AI 보안 과정*
