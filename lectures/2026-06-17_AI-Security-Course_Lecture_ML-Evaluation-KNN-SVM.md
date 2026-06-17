# 2026-06-17 강의 정리 — ML 성능평가 & 알고리즘(KNN, SVM)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 성능평가(분류·회귀·군집) 심화 + KNN·SVM 알고리즘
> **관련 실습 파일**: `../practice/2026-06-17_AI-Security-Course_Practice_ML-Evaluation-KNN.md`, `../practice/2026-06-17_AI-Security-Course_Practice_SVM.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-17

- **머신러닝은 패러다임의 전환이다.** 강사님이 오늘 오프닝에서 한 말인데 꽤 인상적이었다. 객체지향, 함수형 프로그래밍에 이어 ML은 또 다른 프로그래밍 패러다임이고, 유지보수 비용이 폭발적으로 늘어나는 복잡한 시스템 문제에 대한 또 다른 답이다. "조건이 너무 많아서 규칙을 직접 짤 수 없다면, 데이터로 규칙을 학습시킨다" — 이게 핵심이다.

- **F1-Score가 조화평균인 이유를 드디어 이해했다.** 정밀도 0.9, 재현율 0.1인 모델의 산술평균은 0.5지만 이 모델은 쓸모가 없다. 조화평균을 쓰면 0.18이 나오고, 이게 현실을 더 정직하게 반영한다. 100km/h와 80km/h 구간 평균 속도 예시로 이해하니 직관적이었다. 두 수 중 하나라도 낮으면 조화평균이 크게 낮아지는 성질 — 이것 때문에 불균형 데이터에 F1을 쓰는 거다.

- **KNN은 모델을 만들지 않는다.** Lazy learning이라는 표현이 적절하다. 학습 시점에 모델 파라미터를 갱신하지 않고, 예측 시점에 전체 훈련 데이터를 조회해서 답을 낸다. 따라서 훈련은 빠르지만 예측이 느리다. GridSearchCV가 "최적 파라미터를 찾는 실험"이라면, KNN의 k 탐색은 "최적 이웃 수를 찾는 실험"인데 둘 다 같은 구조다. 실제 서비스 사용에는 조심해야한다. 데이터 양이 많고 새로운 값을 자주 확인 하는 서비스에서는 비용문제가 발생한다.

- **SVM의 커널 트릭은 차원을 올려서 선형 분리를 가능하게 한다.** 1차원에서 분리 안 되는 데이터를 y=x² 변환으로 2차원에 올리면 수평선 하나로 분리된다. 핵심은 "실제로 고차원 공간에 올리지 않고, 내적만 계산해서 같은 효과를 낸다"는 것이다. 연산 비용을 획기적으로 줄이는 아이디어.

- **의도적 과대적합이 유효한 케이스가 있다.** 불량품 검사처럼 "정상"이 압도적으로 많고 "불량"을 절대 놓치면 안 되는 상황. 재현율(Recall)을 극대화하기 위해 FP는 허용하되 FN은 최소화하는 방향으로 모델을 세팅하는 것 — 이게 AI 보안 침입탐지에서 "미탐(FN)이 오탐(FP)보다 더 위험하다"는 원칙과 완전히 같은 구조다.

---

## 1. 지난 시간 리뷰

### 핵심 개념

머신러닝의 주요 과정:

1. 데이터 수집 (저작권 주의)
2. 탐색적 데이터 분석 — 요약 통계, 결측치·이상치 확인, 시각화(EDA)
3. 전처리
4. 훈련 → 모델 생성
5. 모델 평가
6. 모델 배포

### 주요 용어 정리

| 용어 | 설명 |
|---|---|
| 데이터 세트 | 학습에 사용되는 전체 데이터 집합 |
| 특징(Feature) | 입력 변수 = 독립 변수 |
| 레이블(Label) | 예측되는 값 = 종속 변수 |
| 모델(Model) | 특징과 레이블의 관계를 수식으로 표현한 것 |
| 학습(Training) | 데이터로 모델 파라미터를 갱신하는 과정 |
| 추론(Inference) | 학습된 모델로 새 데이터를 예측하는 과정 |

---

## 2. 성능평가 — 분류평가

### 핵심 개념

6/16에 배운 정확도·정밀도·재현율·오차행렬에 이어 오늘은 특이도·F1·ROC·AUC까지.

### 특이도 (Specificity)

실제 Negative 중 정확히 Negative로 예측한 비율.

```
Specificity = TN / (TN + FP)
```

재현율(TPR)과 짝을 이루는 지표. "실제 음성을 음성으로 잘 잡아내는가."

### F1-Score

정밀도와 재현율의 **조화평균**. 레이블이 불균일하게 분포된 경우 사용한다.

```
F1 = 2 * (P * R) / (P + R)
```

**왜 조화평균인가?**
두 수 중 하나라도 극단적으로 낮으면 조화평균은 크게 낮아진다. 산술평균은 이걸 희석시켜버린다.

- 예: P=0.9, R=0.1 → 산술평균 0.5 / 조화평균 ≈ 0.18
- 100km/h로 100km + 80km/h로 100km 이동 시 평균 속력 = 2×100×80/(100+80) ≈ 88.9km/h (시간을 기준으로 평균을 내야 정확하므로 조화평균 사용)

### ROC Curve & AUC

분류 임계값(Threshold)에 따라 TPR(=재현율)과 FPR이 어떻게 달라지는지를 그래프로 표현.

| 지표 | 공식 | 의미 |
|---|---|---|
| TPR | TP / (TP + FN) | 실제 양성을 양성으로 잡아낸 비율 |
| FPR | FP / (FP + TN) | 실제 음성을 양성으로 잘못 예측한 비율 |

- x축 FPR, y축 TPR. 기준선(대각선)보다 위에 있을수록 좋은 모델.
- **AUC**: ROC 곡선 아래 면적. 그래프 없이 숫자 하나로 모델 비교 가능. 1에 가까울수록 좋다.

```python
# 주요 평가 지표 한번에
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score

# ROC Curve
from sklearn.metrics import roc_curve
y_proba = lr.predict_proba(x_test)[:, 1]  # 클래스 1일 확률
fpr, tpr, thresholds = roc_curve(y_test, y_proba)
```

---

## 3. 성능평가 — 회귀평가

### 핵심 개념

| 지표 | 공식 | 특징 |
|---|---|---|
| MAE | 평균(절대값) |실제 - 예측|
| MSE | 평균((실제 - 예측)²) | 오차 제곱으로 큰 오차에 민감 |
| RMSE | √MSE | MSE 단위 왜곡 완화 |
| R²(결정계수) | SSR/SST = 1 - SSE/SST | **0~1 사이 값, 현재 권장 기준** |

**MAE/MSE/RMSE의 한계**: 원래 데이터의 규모(단위)를 알아야 오차가 큰지 작은지 판단 가능. R²는 0~1 사이 값이라 규모에 무관하게 해석 가능 → 현재 권장되는 회귀 평가 기준.

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
import numpy as np

print('MAE:', mean_absolute_error(y_test, y_pred))
print('MSE:', mean_squared_error(y_test, y_pred))
print('RMSE:', np.sqrt(mean_squared_error(y_test, y_pred)))
print('R2:', r2_score(y_test, y_pred))
```

---

## 4. 성능평가 — 군집평가

### 핵심 개념

군집화(비지도학습)는 레이블이 없어 성능 평가가 어렵다. 대표 방법: **실루엣(Silhouette)**.

**실루엣 지수 공식**:
```
s_i = (b_i - a_i) / max(a_i, b_i)
```
- `a_i`: i번째 원소가 속한 군집 내 다른 원소들과의 평균 거리 (낮을수록 좋음)
- `b_i`: i번째 원소에서 가장 가까운 **다른** 군집 원소들과의 평균 거리 (높을수록 좋음)
- 값이 클수록 (1에 가까울수록) 군집화가 잘 된 것

**엘보우 기법(Elbow Method)**: k를 1~10으로 늘려가며 inertia(군집 내 거리 합)를 시각화하고, 꺾이는 지점이 최적 k.

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_samples

km = KMeans(n_clusters=3, init='k-means++', random_state=7).fit(X)
silt_vals = silhouette_samples(X, km.labels_)
```

---

## 5. 알고리즘 — KNN

### 핵심 개념

**K-최근접 이웃(K-Nearest Neighbor)**: 새로운 데이터가 들어오면 유클리드 거리 기준으로 가장 가까운 k개 이웃의 레이블 투표로 분류를 결정하는 지도학습 알고리즘.

### 특징

- 알고리즘이 단순하고 구현이 쉬움. 결과 해석도 직관적.
- 최적 k를 찾는 것이 목적. k값 연산 결과로 분류하기 때문에 **모델이 생성되지 않는 게으른 학습(Lazy learning)**
- 데이터셋 편중·이상치가 있으면 일반화 실패 확률 높음 → **정규화(Normalization)로 해결**
- k는 홀수로 지정 (동수 투표 방지), 3~10 사이 값 많이 사용
- 다중 분류에서도 사용 가능

### 하이퍼파라미터

| 파라미터 | 설명 |
|---|---|
| n_neighbors (k) | 이웃 수. 클수록 과소적합, 작을수록 과대적합 |

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import cross_val_score

# 최적 k 탐색
k_values = range(3, len(X_train) // 2, 2)
cv_scores = []
for k in k_values:
    knn = KNeighborsClassifier(n_neighbors=k)
    scores = cross_val_score(knn, X_train, y_train, cv=10, scoring='accuracy')
    cv_scores.append(scores.mean())

best_k = list(k_values)[np.argmax(cv_scores)]
```

---

## 6. 알고리즘 — SVM

### 핵심 개념

**Support Vector Machine**: 데이터를 레이블별로 구분하는 결정 경계선(Decision Boundary)을 찾아 분류하는 지도학습 알고리즘.

### 주요 개념

| 개념 | 설명 |
|---|---|
| 결정 경계(Decision Boundary) | 레이블을 구분하는 선(2D) 또는 초평면(고차원). 특징이 n차원이면 결정 경계는 n-1차원 |
| 서포트 벡터(Support Vector) | 결정 경계에 가장 가까운 데이터 포인트 |
| 마진(Margin) | 결정 경계와 서포트 벡터 사이 거리. **마진을 최대화**하여 일반화 |
| 하드 마진(Hard Margin) | 마진 내 오류 벡터 0개 허용 → 이상치에 민감, 과대적합 위험 |
| 소프트 마진(Soft Margin) | 마진 내 일부 오류 허용 → 일반화에 유리 |

### 비용(C, Cost)

마진 내 허용하는 오류 벡터의 정도를 조정하는 하이퍼파라미터. 기본값 1.0.

```
Margin ∝ 1/Cost
```

- C가 클수록 → 오류 허용 적음 → 하드 마진 → 과대적합 위험
- C가 작을수록 → 오류 허용 많음 → 소프트 마진 → 과소적합 위험

### 커널 트릭(Kernel Trick)

저차원에서 선형 분리가 불가능한 데이터를 **고차원으로 변환**하여 선형 분리 가능하게 만드는 기법.

실제로 고차원으로 올리지 않고 **내적 연산만으로** 동일 효과를 냄 → 연산 비용 절감.

```
Φ(xᵢ) · Φ(x) ⇒ (xᵢ · x)²
```

| 커널 종류 | 특징 |
|---|---|
| linear | 선형 분리 가능한 경우. C만 조정 |
| rbf (Gaussian) | 비선형 분리. 이론적으로 무한 차원 적용 가능. C + gamma 조정 |
| poly | 다항식 |
| sigmoid | 시그모이드 |

### 감마(gamma)

RBF 커널에서 결정 경계선의 곡률을 결정하는 파라미터.

- gamma 낮음 → 경계선이 완만 (일반화 잘 됨)
- gamma 높음 → 경계선이 서포트 벡터에 딱 붙음 (과대적합 위험)

### SVM 특징 요약

- 분류·수치 예측 모두 가능, 비교적 정확도 높음
- 이상치·오류 데이터에 의한 영향력이 적음
- 학습 데이터가 적어도 높은 성능 가능
- 데이터 전처리가 중요
- 결정 경계가 많으면 시각화·설명이 어려움

```python
from sklearn.svm import SVC
from sklearn.model_selection import GridSearchCV

svm_params = [
    {'kernel': ['rbf'], 'gamma': [0.00001, 0.0001, 0.001, 0.1, 1],
     'C': [0.001, 0.1, 1, 10, 100, 1000]}
]
clf = GridSearchCV(SVC(), svm_params, cv=10)
clf.fit(X_train, y_train)
print(clf.best_params_)
```

---

## 다음에 파고들 만한 질문

- AUC가 0.5(기준선)보다 **낮은** 모델이 나왔다면? 그게 정말로 랜덤보다 못한 건가, 아니면 뭔가 데이터 처리 실수인 건가? (레이블이 반전된 경우)

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| | | | |

---
*작성일: 2026-06-17 | 인공지능 사관학교 AI 보안 과정*
