# 2026-06-18 실습 — 군집화(K-Means) · PCA (ml_12, ml_13)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-18_AI-Security-Course_Lecture_Decision-Tree-Ensemble-Clustering-PCA.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-18

- **엘보우 기법은 "그래프를 보고 판단하는" 주관적 기법이다.** inertia 꺾임 지점을 찾는다고 하지만 실습에서 그래프를 보면 딱 떨어지지 않는 경우가 많다. 어디서 "꺾였다"고 볼지는 분석자의 판단이다. 반면 실루엣은 수치로 나오니까 비교적 객관적이다. 두 지표를 같이 보는 게 맞다는 걸 실습으로 확인했다.

- **PCA에서 차원축소본과 복원본이 다른 이유가 명확해졌다.** `fit_transform()`으로 64→25차원으로 줄인 데이터를 시각화하면 5×5 흐릿한 이미지가 나온다. 그걸 `inverse_transform()`으로 복원하면 8×8로 돌아오는데 원본보다는 뭉개져 있다. 손실된 차원의 정보는 복원 불가 — 이게 차원 축소의 본질적 트레이드오프다. 그런데도 숫자가 식별되는 수준으로 복원된다는 것이 인상적이었다.

- **n_components=5²(=25)로 설정한 이유** :  `5**2`로 쓴 건 복원 시각화를 5×5로 `reshape`하기 위한 의도적 선택이다. 단순 숫자로 쓰지 않고 `5**2`로 표현해서 "이 숫자가 왜 25인지"를 코드에 담아보았다.

---

## 실습 개요

- **목표**: K-Means로 붓꽃 데이터 군집화 + 엘보우/실루엣으로 최적 k 탐색. PCA로 손글씨 차원 축소 및 복원 시각화.
- **데이터**: K-Means = 붓꽃(iris) / PCA = 손글씨(digits)

---

## 진행 과정

### Step 1. K-Means 실습 (ml_12)

**엘보우 기법으로 최적 k 탐색**

```python
# 강사 코드
from sklearn.cluster import KMeans

inertias = []
for i in range(1, 11):
    kms = KMeans(n_clusters=i, init='k-means++', random_state=15)
    kms.fit(X)
    inertias.append(kms.inertia_)

plt.plot(range(1, 11), inertias, marker='o')
plt.xlabel('k')
plt.ylabel('inertias')
plt.show()
```

붓꽃 데이터에서 k=3 부근에서 꺾임 → 실제 붓꽃 품종이 3개이므로 일치.

**실루엣으로 최적 k 탐색**

```python
# 강사 코드
from sklearn.metrics import silhouette_samples

silhouettes = []
for i in range(2, 11):
    kms = KMeans(n_clusters=i, init='k-means++', random_state=15)
    pred = kms.fit_predict(X)
    samples = np.mean(silhouette_samples(X, pred, metric='euclidean'))
    silhouettes.append(samples)

plt.plot(range(2, 11), silhouettes, marker='o')
plt.xlabel('k')
plt.ylabel('silhouettes')
plt.show()
```

실루엣은 k=2에서 최고값이 나오는 경우도 있어 엘보우와 같이 봐야 한다. 붓꽃은 setosa가 다른 두 품종과 명확히 분리되어 k=2로도 실루엣이 높게 나온다.

---

### Step 2. PCA 실습 (ml_13)

**데이터**: 손글씨(digits) — 8×8 픽셀 = 64차원

**차원 축소 및 복원**

```python
# 강사 코드
from sklearn.decomposition import PCA

pca = PCA(n_components=5**2)        # 64차원 → 25차원 (5×5로 시각화 목적)
X_reduced = pca.fit_transform(X)    # (1797, 64) → (1797, 25)

X_recovered = pca.inverse_transform(X_reduced)  # (1797, 25) → (1797, 64)
```

**3단계 시각화 비교**

```python
# 강사 코드
# 원본 (8×8)
plt.imshow(X[1].reshape(8, 8), cmap='gray')

# 차원축소본 (5×5) — 흐릿하고 뭉개짐
plt.imshow(X_reduced[1].reshape(5, 5), cmap='gray')

# 복원본 (8×8) — 원본보다 약간 흐리지만 숫자 식별 가능
plt.imshow(X_recovered[1].reshape(8, 8), cmap='gray')
```

원본 64차원 → 25차원으로 줄여도 복원 시 숫자가 식별됨. 39개 차원(약 61%)을 버려도 핵심 특징은 유지된다는 것.

---

## 결과 / 산출물

- `ml_12_Kmeans.ipynb`: 붓꽃 K-Means 군집화, 엘보우·실루엣 최적 k 탐색
- `ml_13_pca.ipynb`: 손글씨 64→25차원 축소, 차원축소본·복원본 시각화

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 엘보우 vs 실루엣의 차이를 실제 데이터로 비교 | 엘보우 꺾임 지점이 주관적임을 체감 (수치 기준 없음) |
| PCA n_components=5**2의 의도를 코드에서 바로 파악 | 차원별 설명분산비율(explained_variance_ratio_) 확인 안 함 |
| 원본·축소본·복원본 3단계 시각화로 PCA 직관 확보 | |

---

## 다음에 이어서 할 것

- 누적 분산비율 그래프로 "몇 차원까지 줄여도 되는가" 탐색

## 다음에 파고들 만한 질문

- 소리 데이터에서 푸리에 변환(주파수 성분 분해)과 PCA(분산 최대화 방향 분해)는 어떤 차이가 있나? 소리 특성 분석에 PCA가 의미 있는 경우와 푸리에 변환이 더 적합한 경우는 각각 언제인가?

---
*작성일: 2026-06-18 | 인공지능 사관학교 AI 보안 과정*
