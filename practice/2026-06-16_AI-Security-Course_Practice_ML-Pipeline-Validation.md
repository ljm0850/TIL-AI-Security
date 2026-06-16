# 2026-06-16 실습 — 사이킷런 파이프라인 · 교차검증 · 하이퍼파라미터 튜닝

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사 코드 따라치기 + 일부 변형)
> **관련 강의 파일**: `../lectures/2026-06-16_AI-Security-Course_Lecture_ML-Preprocessing-Validation.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-16

- **파이프라인의 진짜 가치는 "실수 방지"**였다. scaler를 만들 때 `fit`은 train에만 하고 test엔 `transform`만 해야 하는데(test 정보가 학습에 새어들면 안 되니까), 손으로 하면 깜빡하기 쉽다. `Pipeline`으로 묶으면 이 순서를 자동으로 지켜준다. 강의에서 "부수효과(side effect)를 막는다"고 한 게 이거였다. **편해서가 아니라 안전해서 쓰는 도구**라는 게 인상적이었다.
- **k-fold와 StratifiedKFold의 차이를 숫자로 봤다.** iris는 품종별로 정렬돼 있어서, 그냥 k-fold로 자르면 한 조각에 특정 품종이 쏠릴 수 있다. 실제로 평균은 비슷했지만(0.946 vs 0.947), **fold별 정확도 분포가 Stratified 쪽이 더 안정적**이었다. "기계적으로 자르면 정답이 쏠린다"는 말이 데이터로 와닿았다.
- **GridSearchCV는 어제 내가 손으로 하던 걸 자동화한 상위 버전**이었다. 어제 `random_state`를 손으로 바꿔보며 정확도를 비교했는데, GridSearch는 `max_depth` 후보(3~9)를 다 돌려보고 교차검증으로 최적값(=4, 0.967)을 자동으로 골라줬다. **"여러 후보 × 여러 분할"을 한 번에** 처리하니, 하이퍼파라미터 튜닝이 노가다가 아니라 한 번의 호출이 됐다.

---

## 실습 개요
- **목표**: 사이킷런의 전처리 자동화(파이프라인) · 교차검증 · 하이퍼파라미터 튜닝을 iris로 직접 돌려보기.
- **방법**: `ml_03`(스케일링+파이프라인), `ml_04`(KFold/StratifiedKFold), `ml_05`(GridSearchCV).

---

## 진행 과정

### Step 1. 스케일링 → 파이프라인 (ml_03)
```python
# 수동: scaler는 train에만 fit, test엔 transform만
mimx_scaler = MinMaxScaler()
mimx_scaler.fit(X_train)              # 기준 생성 (train 기준)
X_train1 = mimx_scaler.transform(X_train)
X_test1  = mimx_scaler.transform(X_test)   # test엔 fit 안 함 (누수 방지)

# 파이프라인: 위 순서를 하나로 묶음
pipe = Pipeline([
    ('scaler', MinMaxScaler()),
    ('lr', LinearRegression()),
])
pipe.fit(X_train, y_train)
print(pipe.score(X_test, y_test))     # 0.9433 (수동과 동일)
```
- 수동 처리와 파이프라인 결과가 정확히 같음(0.9433) → 파이프라인이 같은 일을 더 안전하게 한다는 확인.

### Step 2. 교차검증 — KFold vs StratifiedKFold (ml_04)
```python
kfold = KFold(n_splits=7)
result = []
for train_idx, test_idx in kfold.split(iris_data):
    X_train, X_test = iris_data[train_idx], iris_data[test_idx]
    y_train, y_test = iris_label[train_idx], iris_label[test_idx]
    dt_clf.fit(X_train, y_train)
    result.append(accuracy_score(y_test, dt_clf.predict(X_test)))
# KFold 평균 ≈ 0.946

# StratifiedKFold: split에 레이블(y)도 같이 넘김 → 분포 유지
skfold = StratifiedKFold(n_splits=7)
for train_idx, test_idx in skfold.split(iris_data, iris_label):
    ...
# StratifiedKFold 평균 ≈ 0.947
```
- 차이의 핵심: `kfold.split(데이터)` vs `skfold.split(데이터, 레이블)` — **Stratified는 레이블을 받아 각 fold에 비율을 맞춘다.**

### Step 3. 하이퍼파라미터 튜닝 (ml_05)
```python
param_grid = {'max_depth': np.arange(3, 10)}
dt_clf = GridSearchCV(DecisionTreeClassifier(random_state=15),
                      param_grid, scoring='accuracy')
dt_clf.fit(iris_data, iris_label)

print(dt_clf.best_score_, dt_clf.best_params_)   # 0.9667 {'max_depth': 4}
dt_clf.predict([[4.4, 2.9, 1.4, 0.2]])           # → setosa
```
- `best_estimator_`가 자동으로 최적 모델이 되어 바로 `predict` 가능.

---

## 결과 / 산출물
- 파이프라인 score **0.9433** (수동 처리와 동일).
- 교차검증 평균 — KFold **0.946** / StratifiedKFold **0.947**.
- GridSearchCV best **0.967** (`max_depth=4`).

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 파이프라인이 막는 "데이터 누수"의 의미를 코드로 이해 | `ml_03`에서 분류 레이블에 `LinearRegression`을 쓴 점은 좀 어색 (아래 질문) |
| KFold↔StratifiedKFold를 split 인자 차이로 구분 | fold별 분포를 `np.unique`로 더 자세히 뜯어보진 못함 |
| GridSearch로 튜닝을 한 번의 호출로 자동화 | scoring/cv 옵션을 바꿔보는 실험까지는 못 감 |

---

## 다음에 이어서 할 것
- 타이타닉 모델에 교차검증/GridSearchCV를 적용해 단일 분할(0.79/0.83)보다 신뢰도 높은 점수로 다시 비교해보기.

## 다음에 파고들 만한 질문
- `ml_03`에서 `LinearRegression`을 쓰고 `score`가 0.943이 나왔는데, **분류 문제에 회귀 모델을 쓰면 이 점수를 어떻게 해석해야 하나?** (그리고 애초에 분류에 회귀를 쓰는 게 맞는 선택인지)

---
*작성일: 2026-06-16 | 인공지능 사관학교 AI 보안 과정*
