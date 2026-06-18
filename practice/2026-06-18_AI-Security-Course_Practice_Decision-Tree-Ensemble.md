# 2026-06-18 실습 — 결정트리 · 앙상블 (ml_09, ml_10, ml_11)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-18_AI-Security-Course_Lecture_Decision-Tree-Ensemble-Clustering-PCA.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-18

- **결정트리를 시각화했더니 왜 과적합이 나는지 바로 보였다.** `tree.plot_tree()`로 트리를 그려보니 깊이가 아주 깊어지면서 리프 노드 하나가 샘플 1~2개를 담고 있는 경우가 나왔다. 이게 바로 훈련 데이터의 노이즈까지 학습한 상태다. `max_depth`를 제한하니 시각화가 훨씬 단순해지고 성능은 오히려 올라갔다. "덜 배우는 게 더 잘 맞는" 역설이 눈에 보이는 경험이었다.

- **랜덤 포레스트는 n_estimators를 홀수로 쓰는 이유가 있다.** 실습에서 `n_estimators=[101, 111]`을 썼는데, KNN의 k를 홀수로 하는 것과 같은 이유다. 분류 결과를 다수결로 결정하는데, 짝수면 동수가 나올 수 있다. 알고리즘마다 이런 암묵적인 관례들이 있다는 걸 실습을 통해 체감했다.

- **XGBoost는 코드 자체는 간단한데 내부는 복잡하다.** `XGBClassifier()`로 호출하면 끝이지만, 학습 과정에서 경사 하강법, 트리 가지치기, 조기 종료 등이 자동으로 동작한다. `eval_metric='mlogloss'`를 명시해야 경고가 없어지는 부분에서 내부 동작에 신경 써야 한다는 걸 느꼈다.

- **feature importance가 알고리즘마다 달랐다.** 결정트리, 랜덤포레스트 모두 붓꽃 데이터에서 `petal width`와 `petal length`가 중요하게 나왔다. 이건 알고리즘이 다르더라도 데이터의 실제 구조를 반영한다는 의미다. 반면 `sepal length`는 두 알고리즘에서 모두 중요도가 낮았다.

---

## 실습 개요

- **목표**: 결정트리 → 랜덤포레스트 → XGBoost 순으로 앙상블 흐름 체험. 동일 데이터로 성능 비교.
- **데이터**: 결정트리·랜덤포레스트 = 붓꽃(iris) / XGBoost = 손글씨(digits)

---

## 진행 과정

### Step 1. 결정트리 실습 (ml_09)

**기본 훈련 및 평가**

```python
# 강사 코드
from sklearn.tree import DecisionTreeClassifier
dt_clf = DecisionTreeClassifier(random_state=15)
dt_clf.fit(X_train, y_train)
y_pred = dt_clf.predict(X_test)
accuracy_score(y_test, y_pred)
```

**트리 시각화**

```python
# 강사 코드
from sklearn import tree
plt.figure(figsize=(16, 8))
tree.plot_tree(dt_clf,
               feature_names=iris.feature_names,
               class_names=iris.target_names,
               fontsize=10,
               filled=True)
plt.show()
```

각 노드에 `gini`, `samples`, `value`, `class`가 표시됨. `filled=True`로 클래스별 색이 입혀져 시각적으로 순도를 파악 가능.

**피처 중요도 시각화**

```python
# 강사 코드
n_features = X.shape[1]
plt.barh(range(n_features), dt_clf.feature_importances_, align='center')
plt.yticks(np.arange(n_features), iris.feature_names)
plt.xlabel('중요도')
plt.show()
```

붓꽃 결과: `petal width > petal length >> sepal width ≈ sepal length`

**GridSearchCV로 최적 하이퍼파라미터 탐색**

```python
# 강사 코드
dt_params = [
    {'criterion': ['gini'], 'max_depth': [2, 3, 4, 5, 6, 7, 8],
     'min_samples_split': [2, 3, 4]}
]
dt_clf2 = GridSearchCV(DecisionTreeClassifier(random_state=15), dt_params, cv=10)
dt_clf2.fit(X_train, y_train)
print(dt_clf2.best_params_)
```

`gs.cv_results_['mean_test_score']`로 파라미터 조합별 성능 시각화 가능.

---

### Step 2. 랜덤포레스트 실습 (ml_10)

결정트리와 코드 구조가 동일. `DecisionTreeClassifier` → `RandomForestClassifier`로 교체하고, `n_estimators` 파라미터 추가.

```python
# 강사 코드 (본인이 n_estimators를 101, 111로 조정)
from sklearn.ensemble import RandomForestClassifier

dt_params = [
    {'n_estimators': [101, 111], 'max_depth': [2, 3, 4, 5, 6, 7, 8],
     'min_samples_split': [2, 3, 4]}
]
dt_clf2 = GridSearchCV(RandomForestClassifier(random_state=15), dt_params, cv=10)
dt_clf2.fit(X_train, y_train)
print(dt_clf2.best_params_, dt_clf2.best_score_)
```

> **주의**: `n_estimators`는 홀수 권장 (다수결 동수 방지). 강사 코드의 `[100]`을 `[101, 111]`로 변경.

피처 중요도는 `gs.best_estimator_.feature_importances_`로 확인.

---

### Step 3. XGBoost 실습 (ml_11)

데이터: 손글씨(digits) — 8×8 픽셀 이미지를 64차원 벡터로 표현한 0~9 숫자 데이터.

```python
# 강사 코드
from xgboost import XGBClassifier

# eval_metric 명시 안 하면 경고 발생
xgb_clf = XGBClassifier(random_state=15, eval_metric='mlogloss')
xgb_clf.fit(X_train, y_train)
y_pred = xgb_clf.predict(X_test)
accuracy_score(y_test, y_pred)
```

**GridSearchCV**

```python
# 강사 코드
dt_params = [
    {'n_estimators': [101, 111], 'learning_rate': [0.05, 0.1]}
]
dt_clf2 = GridSearchCV(XGBClassifier(random_state=15, eval_metric='mlogloss'),
                       dt_params, cv=10)
dt_clf2.fit(X_train, y_train)
print(dt_clf2.best_params_, dt_clf2.best_score_)
```

---

## 결과 / 산출물

| 알고리즘 | 데이터 | 비고 |
|---|---|---|
| DecisionTree | iris | 기본 + GridSearch 비교 |
| RandomForest | iris | n_estimators 홀수 적용 |
| XGBoost | digits | eval_metric='mlogloss' 명시 |

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 트리 시각화로 과적합 원리를 눈으로 확인 |  |
| 랜덤포레스트에서 n_estimators 홀수 관례 적용 |  |
| 결정트리·랜덤포레스트 피처 중요도 비교 | |

## 

## 다음에 파고들 만한 질문

- 하이퍼파라미터 탐색에서 GridSearch 대신 이진탐색 방식이 가능할까? 파라미터 반응이 단조적이라는 전제가 필요한데, 실전에서 그 전제를 확신할 수 있는 경우가 있을까?

---
*작성일: 2026-06-18 | 인공지능 사관학교 AI 보안 과정*
