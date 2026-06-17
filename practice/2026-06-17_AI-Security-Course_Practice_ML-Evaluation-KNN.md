# 2026-06-17 실습 — ML 분류평가 & KNN (ml_06, ml_07)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-17_AI-Security-Course_Lecture_ML-Evaluation-KNN-SVM.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-17

- **반복 코드를 보자마자 함수로 묶고 싶었다.** `ml_07_KNN.ipynb`를 보면 특징을 2개 쓸 때와 3개 쓸 때의 코드 블록이 거의 동일한데 그냥 복붙으로 진행하라고 하셨다. 강사님 의도는 "흐름을 몸으로 익혀라"인데, 나는 DRY 원칙이 먼저 튀어나와서 결국 Claude Code로 파이프라인 버전(`ml_07_KNN_pipeline.ipynb`)을 따로 만들었다. 어느 쪽이 더 낫다기보다 목적이 다른 거다 — 강사 코드는 학습용, 내 코드는 재사용 구조 탐구용.

- **Claude Code가 `StandardScaler`를 자동으로 끼워 넣었다.** KNN은 유클리드 거리 기반이라 피처 스케일이 다르면 왜곡이 생긴다. TRB(리바운드, 0~14 범위)와 AST(어시스트, 0~10 범위)는 단위가 비슷하지만, 이걸 퍼센트 데이터로 정규화하지 않은 상태다. 나라면 출전 게임 수로 나눠서 경기당 수치로 맞춘 뒤 실험했을 것 같다. Claude Code가 맥락(도메인 지식)을 모르니까 통계적 스케일링으로 처리한 거고, 맥락 이해는 여전히 사람의 몫이라는 걸 다시 확인했다.

- **ROC Curve에서 임계값을 움직이는 게 뭔지 이해했다.** `predict_proba()`로 "클래스 1일 확률"을 꺼내고, `roc_curve()`가 임계값을 0에서 1까지 바꿔가며 TPR/FPR 쌍을 전부 계산해준다. 실제 모델이 찍은 예측값(`predict()`)은 기본 임계값 0.5에서의 한 점에 불과하다 — 그 점을 ROC 그래프에 빨간 점으로 찍어 확인한 것이 인상적이었다.

---

## 실습 개요

- **목표**: 분류 평가 지표(F1, ROC, AUC) 코드 실습 + KNN으로 NBA 농구선수 포지션(C vs G) 분류
- **방법**: 강사 코드 따라가기 → 특징 추가 실험 → Claude Code로 파이프라인 리팩터링

---

## 진행 과정

### Step 1. 분류 평가 실습 (ml_06)

**데이터 생성 — `make_classification`**

```python
# 강사 코드
X, y = make_classification(
    n_samples=100, n_features=2, n_informative=1, n_redundant=0,
    n_clusters_per_class=1, flip_y=0.15, random_state=10
)
```

`flip_y=0.15`는 레이블의 15%를 의도적으로 뒤집어 노이즈를 주는 파라미터. 실제 데이터처럼 약간의 노이즈가 섞인 상황을 시뮬레이션.

**주요 평가 지표 코드**

```python
# 강사 코드
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score

accuracy  = accuracy_score(y_test, y_pred)
precision = precision_score(y_test, y_pred)
recall    = recall_score(y_test, y_pred)
f1        = f1_score(y_test, y_pred)
roc_auc   = roc_auc_score(y_test, y_pred)
```

**ROC Curve 그리기**

```python
# 강사 코드
from sklearn.metrics import confusion_matrix, roc_curve

tn, fp, fn, tp = confusion_matrix(y_test, y_pred).ravel()

# predict_proba: 각 클래스일 확률을 반환. [:, 1]은 클래스 1(양성)일 확률
y_proba = lr.predict_proba(x_test)[:, 1]
fpr, tpr, thresholds = roc_curve(y_test, y_proba)

plt.plot(fpr, tpr, 'o-')
plt.plot([0, 1], [0, 1], '--')  # 기준선
plt.plot(fp/(fp+tn), tp/(tp+fn), 'ro', ms=8)  # 현재 임계값(0.5)에서의 점
plt.xlabel('FPR'); plt.ylabel('TPR'); plt.title('ROC Curve')
plt.show()
```

`RocCurveDisplay.from_estimator(lr, x_test, y_test)`로 한 줄로도 그릴 수 있다.

---

### Step 2. KNN 실습 — 농구선수 포지션 분류 (ml_07)

**데이터 및 전처리**

NBA 2021 시즌 선수 데이터. 포지션 단순화: PG·SG → G (가드), C (센터)만 남겨 이진 분류.

```python
# 강사 코드
df_data = df[['Pos', 'TRB', '3P', 'AST', 'BLK']]
df_data = df_data.loc[df_data['Pos'].isin(['C', 'SG', 'PG'])]
df_data.loc[df_data['Pos'].isin(['SG', 'PG']), 'Pos'] = 'G'
```

**최적 k 탐색**

```python
# 강사 코드
k_values = range(3, len(X_train) // 2, 2)  # 홀수만
cv_scores = []

for k in k_values:
    knn = KNeighborsClassifier(n_neighbors=k)
    scores = cross_val_score(knn, X_train, y_train, cv=10, scoring='accuracy')
    cv_scores.append(scores.mean())

max_idx = np.argmax(cv_scores)
k = k_values[max_idx]
```

**특징 2개(TRB, AST)로 훈련 후 특징 3개(TRB, AST, 3P)로 재실험**

코드 블록이 거의 동일하게 반복됨. 특징 추가 시 정확도가 소폭 변화.

---

### Step 3. KNN 파이프라인 리팩터링 (ml_07_KNN_pipeline — 본인 작업)

강사 코드의 반복 구조를 보고 Pipeline으로 재구성. Claude Code를 이용해 생성.

> **이 코드의 95% 이상은 Claude Code에 의해 만들어졌습니다.**  
> 기존 ml_07_KNN.ipynb의 반복 코드를 Pipeline으로 재사용성을 높이기 위해 만듦.

**핵심 구조**

```python
# 본인이 Claude Code로 생성한 코드
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

def run_knn_pipeline(df, features: list, target: str = 'Pos',
                     test_size: float = 0.25, random_state: int = 15, cv: int = 10):
    X = df[features].values
    y = df[[target]].values.ravel()
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=test_size, random_state=random_state)

    k_values = range(3, len(X_train) // 2, 2)
    cv_scores = []
    for k in k_values:
        pipe = Pipeline([
            ('scaler', StandardScaler()),   # Claude Code가 자동 추가 (KNN은 거리 기반이므로 스케일링 필요)
            ('knn', KNeighborsClassifier(n_neighbors=k))
        ])
        scores = cross_val_score(pipe, X_train, y_train, cv=cv, scoring='accuracy')
        cv_scores.append(scores.mean())

    best_k = list(k_values)[np.argmax(cv_scores)]
    # ... 최종 훈련 및 평가
    return best_k, acc, best_pipe
```

**실험 결과**

| Features | Best K | Accuracy |
|---|---|---|
| TRB, AST | 3 | 0.8649 |
| TRB, AST, 3P | 5 | 0.9054 |
| TRB, AST, 3P, BLK | 3 | 0.8919 |

> **도메인 관점에서**: 출전 게임 수(G)로 나눠 경기당 수치로 정규화하는 것이 더 공정한 비교일 수 있다. 많이 뛴 선수와 적게 뛴 선수의 절대 수치를 그대로 비교하면 출전 시간 효과가 섞인다. AI는 이 맥락을 모르고 통계적 StandardScaler만 적용했다.

---

## 결과 / 산출물

- `ml_06_분류평가.ipynb`: F1, ROC Curve, AUC 실습 완료
- `ml_07_KNN.ipynb`: 농구선수 포지션 분류 (특징 2→3개 실험)
- `ml_07_KNN_pipeline.ipynb`: Claude Code 활용 파이프라인 리팩터링 (특징 조합 3가지 실험)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| ROC Curve에서 임계값/predict_proba 관계를 시각적으로 이해 |  |
| 반복 코드를 Pipeline으로 구조화하는 시도 |  |
| AI가 놓친 도메인 맥락(경기당 수치 정규화)을 스스로 발견 | |

## 다음에 파고들 만한 질문

- Pipeline 안에서 `cross_val_score`를 쓸 때 `StandardScaler`의 `fit`이 각 fold마다 훈련 데이터만으로 이뤄지는지, 아니면 전체 데이터로 이뤄지는지? (데이터 누수 방지 관점)

---
*작성일: 2026-06-17 | 인공지능 사관학교 AI 보안 과정*
