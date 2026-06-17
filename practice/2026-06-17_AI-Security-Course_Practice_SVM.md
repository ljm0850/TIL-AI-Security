# 2026-06-17 실습 — SVM (ml_08)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-17_AI-Security-Course_Lecture_ML-Evaluation-KNN-SVM.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-17

- **GridSearchCV를 SVM에 쓰니 비로소 그 가치가 실감났다.** KNN은 k 하나만 탐색하면 되는데, SVM은 kernel · C · gamma의 조합이 폭발적으로 많다. GridSearchCV + cv=10으로 돌리면 `5 × 6 × 10 = 300번`의 훈련이 자동으로 이뤄진다. 전날 배운 GridSearchCV가 단독으로 쓸 때보다 이런 다중 파라미터 알고리즘에서 진짜 빛을 발하는 걸 체감했다.

- **C와 gamma 조합을 시각화해보니 경계선 모양이 완전히 달랐다.** C=10/gamma=0.01이면 경계선이 완만한 곡선인데, C=1000/gamma=1.0이면 데이터 포인트 하나하나를 감싸는 울퉁불퉁한 경계가 된다. 같은 데이터, 같은 알고리즘인데 파라미터만 바꿨을 뿐인데 이 정도로 달라지는 건 직접 그래프로 봐야 실감이 난다.

- **SVM이 KNN보다 데이터가 적을 때 유리한 이유가 이해됐다.** KNN은 예측할 때마다 전체 훈련 데이터를 조회하는 Lazy learning인 반면, SVM은 훈련 완료 시점에 서포트 벡터(경계 근처 데이터 몇 개)만 남기면 예측이 가능하다. 데이터가 적어도 서포트 벡터를 정확히 찾으면 강한 일반화가 된다.

---

## 실습 개요

- **목표**: SVM으로 NBA 농구선수 포지션(C vs G) 분류. GridSearchCV로 최적 kernel·C·gamma 탐색
- **방법**: KNN 실습과 동일한 데이터셋(basketball.csv) 사용. TRB, AST 특징 사용

---

## 진행 과정

### Step 1. 데이터 준비 (KNN 실습과 동일)

```python
# 강사 코드
df_data = df[['Pos', 'TRB', 'AST']]
df_data = df_data.loc[df_data['Pos'].isin(['C', 'PG', 'SG'])]
df_data.loc[df_data['Pos'].isin(['PG', 'SG']), 'Pos'] = 'G'

y_df = df_data[['Pos']]
X_df = df_data[['TRB', 'AST']]

X_train, X_test, y_train, y_test = train_test_split(
    X_df.values, y_df.values.ravel(), test_size=0.25, random_state=15
)
```

---

### Step 2. GridSearchCV로 최적 하이퍼파라미터 탐색

```python
# 강사 코드
from sklearn.model_selection import GridSearchCV
from sklearn.svm import SVC

svm_params = [
    {
        'kernel': ['rbf'],
        'gamma': [0.00001, 0.0001, 0.001, 0.1, 1],
        'C': [0.001, 0.1, 1, 10, 100, 1000]
    }
]

clf = GridSearchCV(SVC(), svm_params, cv=10)  # 5 gamma × 6 C × 10 fold = 300회 훈련
clf.fit(X_train, y_train)
print(clf.best_params_)
```

`clf.best_params_`로 최적 조합 확인. `clf.best_score_`로 해당 조합의 CV 평균 정확도 확인.

---

### Step 3. 평가 및 예측

```python
# 강사 코드
y_pred = clf.predict(X_test)

from sklearn.metrics import accuracy_score
accuracy_score(y_test, y_pred)

# 새로운 데이터 예측 (TRB=9.0, AST=5.4인 선수의 포지션)
clf.predict([[9.0, 5.4]])
```

---

### Step 4. 시각화 — C·gamma 조합별 결정 경계선

강의 자료의 3×3 그리드 시각화:

- **행**: C = 10 / 100 / 1000
- **열**: gamma = 0.01 / 0.10 / 1.00

**관찰**:
- C=10, gamma=0.01 → 완만한 경계선, 넓은 마진
- C=10, gamma=1.0 → 데이터에 밀착된 울퉁불퉁한 경계 (과대적합 징후)
- C=1000, gamma=1.0 → 각 포인트를 섬처럼 감싸는 경계 (명백한 과대적합)

gamma를 올리면 경계선이 서포트 벡터에 더 밀착되고, C를 올리면 마진이 작아진다. 두 파라미터가 함께 과대적합을 만드는 방향으로 작용한다.

---

## 결과 / 산출물

- `ml_08_SVM.ipynb`: GridSearchCV + SVC로 농구선수 포지션 분류 완료
- C·gamma 조합별 결정 경계 시각화 (3×3 그리드)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| GridSearchCV가 다중 파라미터 탐색에서 진가를 발휘한다는 걸 체감 | best_params_ 실제 수치를 파일에 기록하지 않음 |
| C·gamma 조합 시각화로 파라미터 변화의 직관적 이해 | KNN 정확도와의 직접 비교 수치 부재 |
| SVM이 소수 데이터에 강한 이유를 구조적으로 이해 | |

---

## 다음에 이어서 할 것

- KNN vs SVM 동일 데이터 정확도 수치 비교
- SVM에 StandardScaler 추가 후 정확도 변화 확인

## 다음에 파고들 만한 질문

- SVM의 `predict_proba()`는 왜 기본적으로 비활성화되어 있고(`probability=True` 옵션 필요), 활성화하면 어떤 연산이 추가되는가?

---
*작성일: 2026-06-17 | 인공지능 사관학교 AI 보안 과정*
