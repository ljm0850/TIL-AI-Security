# 2026-06-16 실습 — 타이타닉 생존 예측 (전처리 → 학습 → 평가 완주)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (어제 중단분 이어서 완성)
> **관련 강의 파일**: `../lectures/2026-06-16_AI-Security-Course_Lecture_ML-Preprocessing-Validation.md`
> **이전 진행분**: `../practice/2026-06-15_AI-Security-Course_Practice_Machine-Learning.md` (6/15 결측치 진단까지)

---

## 느낀 것 / 배운 것

### 📅 2026-06-16

- 어제 결측치 heatmap까지만 보고 멈췄던 걸, 오늘 **`fillna` → 인코딩 → 학습 → 평가까지 끝까지 한 번에 완주**했다. 어제 "다음에 이어서 할 것"에 적어둔 항목(age 중앙값, embarked 최빈값, factorize, DT/LR 비교)을 그대로 따라가니 막힘이 없었다. **인수인계 메모대로 흘러가는 경험**이 좋았다.
- `age` 결측치를 채울 때 평균이 아니라 **중앙값**을 골랐다. histplot으로 분포를 보니 한쪽으로 치우쳐 있어서, 이런 경우 평균보다 중앙값이 "대표값"으로 더 적절하다는 걸 손으로 확인했다. 강의에서 들은 "통계 = 대표값 찾기"가 여기서 적용됐다.
- 결과가 흥미로웠다 — **DecisionTree 0.798 < LogisticRegression 0.834.** 같은 전처리 데이터인데 알고리즘에 따라 정확도가 크게 갈렸다. "모델을 바꾸는 건 fit/predict 골격 그대로 한 줄"이라 두 개를 나란히 돌려 비교하는 게 쉬웠고, **어떤 알고리즘이 이 데이터에 맞는지는 결국 돌려봐야 안다**는 걸 느꼈다.
- 결측치를 "행렬의 크기를 보장하기 위해" 채운다는 관점이 새로웠다. 행을 지우면 다른 열의 데이터까지 같이 날아가니, 함부로 `dropna`하기보다 채워서 행렬 크기를 유지하는 게 기본이라는 감각.

---

## 실습 개요
- **목표**: 타이타닉 생존 여부(survived) 이진 분류 — 전처리부터 두 모델 비교까지 완주.
- **방법**: 결측치 처리 → EDA → 범주형 인코딩 → 학습/평가 데이터 분리 → DecisionTree·LogisticRegression 학습 → accuracy 비교.

---

## 진행 과정

### Step 1. 결측치 처리 (어제 진단 → 오늘 처리)
```python
# age: 수치형 → 분포 보고 중앙값으로 (평균보다 적절)
sns.histplot(df1['age'], kde=True)
df1['age'] = df1['age'].fillna(df1['age'].median())   # 28.0

# embarked: 범주형 → 최빈값으로
df1['embarked'] = df1['embarked'].fillna(df1['embarked'].mode()[0])
```
- `deck`은 결측 비율이 커서 일찌감치 drop, `age`(약 20% 결측)는 채움, `embarked`(0.2%)도 최빈값으로.

### Step 2. EDA — 생존과의 관계 확인
```python
def plot_row(src, ref, col_list):       # 범주별 생존 분포 시각화
    for index, value in enumerate(col_list):
        plt.subplot(1, len(col_list), index+1)
        sns.countplot(x=value, hue=ref, data=src)   # ← x= 키워드 명시
        plt.title(value)
    plt.tight_layout(); plt.show()

plot_row(df1, 'survived', ['pclass','sex','age','embarked'])
plot_row(df1, 'survived', ['who','alone','family_size'])
```
- **본인 변형**: 강사/어제 코드의 `sns.countplot(value, ...)`를 `sns.countplot(x=value, ...)`로 수정. (seaborn 최신 버전은 위치 인자 대신 `x=` 키워드를 요구해서 안 그러면 에러)

### Step 3. 범주형 인코딩 + 타입 변환
```python
for col in ['sex','embarked','who','alone']:
    labels, uniques = pd.factorize(df1[col])   # 레이블 인코딩
    df1[col] = labels
df1['age'] = df1['age'].astype('int')
```
- 분류 개수가 적은 컬럼들이라 원-핫 대신 **레이블 인코딩(`factorize`)** 선택.

### Step 4. 학습/평가 분리 → 두 모델 학습 → 평가
```python
y = df1['survived']                 # 정답(1차원)
x_df = df1.drop(['survived'], axis=1)   # 피처(2차원)

X_train, X_test, y_train, y_test = train_test_split(
    x_df, y, test_size=0.25, random_state=15)

dt_clf = DecisionTreeClassifier(random_state=15).fit(X_train, y_train)
lr     = LogisticRegression(random_state=15).fit(X_train, y_train)

print(accuracy_score(y_test, dt_clf.predict(X_test)))  # 0.7982
print(accuracy_score(y_test, lr.predict(X_test)))      # 0.8341
```

---

## 결과 / 산출물
- 타이타닉 전처리 → 분류 모델 2종 학습·평가 **완주**.
- **DecisionTree 정확도 0.798 / LogisticRegression 정확도 0.834** → 이 데이터에선 로지스틱 회귀가 더 나았다.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 어제 중단분을 메모대로 완주 (결측치→인코딩→학습→평가) | 인코딩을 전부 레이블로만 함 — 원-핫과 비교는 못 해봄 |
| 두 모델(DT/LR)을 나란히 비교 | 스케일링/교차검증은 이 실습엔 미적용 (다음 실습에서 따로 학습) |
| seaborn 버전 이슈(`x=`)를 직접 잡아 해결 | 왜 LR이 DT보다 나았는지는 아직 추측만 |

---

## 다음에 이어서 할 것
- 개선 여지: 범주형을 원-핫으로 바꿔 비교 / 피처 스케일링 적용 / 단일 분할 대신 교차검증으로 평가해 점수 신뢰도 높이기.

## 다음에 파고들 만한 질문
- `who`(man/woman/child)는 `sex`(남/여)와 `age`(나이) 정보가 사실상 겹치는데, 이렇게 **정보가 중복되는 피처를 다 넣어도 되나?** (겹치는 피처가 모델에 해가 되는지 — 다중공선성 같은 게 궁금)

---
*작성일: 2026-06-16 | 인공지능 사관학교 AI 보안 과정*
