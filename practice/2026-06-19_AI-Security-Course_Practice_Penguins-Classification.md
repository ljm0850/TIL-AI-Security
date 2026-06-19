# 2026-06-19 실습 — 펭귄 3종 분류 모델 만들기 (종합 과제)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 과제 (지금까지 배운 분류 알고리즘 종합)
> **관련 강의 파일**: 6/16~6/18 ML 전처리·평가·결정트리·앙상블 정리 참조

---

## 느낀 것 / 배운 것

### 📅 2026-06-19

- **"강해 보이던 feature가 importance에서는 약하게 나올 수 있다"가 이번 과제의 가장 큰 배움.** `island`는 시각화에서 "Torgersen엔 Adelie만 산다"처럼 강력한 구분자로 보였는데, 막상 feature importance는 낮았다. 이유는 수치 feature(부리 길이·깊이 등)들이 이미 island가 주는 정보를 상당 부분 담고 있어서 — 즉 **feature 간 상관/중복** 때문에 island의 "고유 기여"가 줄어든 것. 단변량으로 강해 보이는 것과 다변량 모델 안에서의 기여는 다르다. (6/16에 적어둔 "다중공선성 / 정보 중복 feature를 다 넣어도 되나?" 호기심이 실제 데이터에서 눈에 보인 순간.)

- **시각화로 한 사전 분석이 importance 결과와 대체로 맞아떨어졌다.** `culmen_length_mm`가 종 구분 핵심일 거라고 pairplot에서 짐작했고, importance 1위로 확인됐다. EDA가 모델 결과를 미리 읽는 "가설"이 된다는 감을 얻음. 다만 island 건처럼 빗나간 부분이 있어서, 데이터 분석을 더 촘촘히 해야겠다는 반성.

- **`sex`는 "직접 구분자는 아니지만 다른 feature에 영향을 줄 것"이라 판단해 살렸는데, importance 거의 0이었다.** 가설은 합리적이었지만 모델은 보조 효과를 거의 잡아내지 못했다. 가설이 맞는지는 "제거 후 성능 비교"로 검증해야 하는데 이번엔 거기까진 못 감 → 개선 방향으로 남김.

- **모델 선택을 "느낌"이 아니라 교차검증 수치로 했다.** Decision Tree(96.23% ±1.70%) vs Random Forest(98.50% ±1.41%)를 StratifiedKFold로 비교해 정확도·안정성(표준편차) 모두 나은 RF 선택. 6/18에 배운 "앙상블이 단일 트리보다 분산을 줄인다"가 표준편차 숫자로 드러남.

---

## 실습 개요
- **목표**: 펭귄 3종(Adelie, Chinstrap, Gentoo)을 신체 측정값으로 분류하는 모델 제작 (그동안 배운 전처리→EDA→인코딩→모델선택→평가 전 과정 직접 수행).
- **데이터**: seaborn `penguins.csv` (344행, feature: 부리 길이/깊이, 지느러미 길이, 체중, 성별, 섬).
- **방법**: 결측치/이상치 처리 → EDA → 인코딩 → DT vs RF 교차검증 비교 → RF 최종 학습·평가 → feature importance 해석.

---

## 진행 과정

### Step 1. 데이터 확인 · 결측치 파악
- `info()` / `describe()`로 확인: 수치 feature 4개에 null 2행, `sex`에 null 11행.
- `describe()`에서 수치 feature 간 단위 차이가 커서 정규화 필요성 판단(단, 최종 모델이 트리 계열이라 스케일링은 결과에 영향 적음).

### Step 2. 전처리
- 수치 feature null 2행 → `dropna(subset=[...4개])`로 제거.
- `sex` null 9행 → countplot/pairplot으로 "성별이 종을 직접 가르진 않지만 다른 수치에 영향을 줄 수 있음" 판단 후, 전체 대비 3% 미만이라 행 제거 선택(컬럼 자체는 유지).
- 이상치: IQR 기준으로 4개 수치 feature 모두 0건 → 별도 처리 없음.

### Step 3. EDA (시각화로 구분력 가설 세우기)
- `pairplot(hue='species')`: 종별 분포 분리 정도 확인.
- `violinplot` (종 × 성별): `flipper_length_mm`는 단독 구분력이 생각보다 애매.
- `scatterplot(culmen_length × flipper_length)`: 조합하면 종이 분리됨 → flipper를 보조 feature로 유지 결정.
- `body_mass_g`: Gentoo 분리에 유효, culmen_length와 조합 시 Chinstrap/Adelie 분리.
- `countplot(island × species)`: Torgersen엔 Adelie만 → island가 강해 보임(나중에 importance에서 뒤집힘).

### Step 4. 인코딩
- **Label Encoding**: `sex`(이진), `species`(타겟).
- **One-Hot Encoding**: `island`(3개 범주, 순서 없음) → `pd.get_dummies(drop_first=False)`.

### Step 5. 분리 · 모델 비교
- `train_test_split(test_size=0.2, stratify=y)` — 클래스 비율 유지.
- `StratifiedKFold(n_splits=5)`로 train 내부 교차검증.
- `cross_val_score`로 DT vs RF 비교 → **RF 선택**.

### Step 6. 최종 학습 · 평가 · 해석
- `RandomForestClassifier(random_state=15)` 학습.
- `classification_report` 출력, feature importance 막대그래프로 해석.

---

## 결과 / 산출물
- **교차검증**: Decision Tree `0.9623 ± 0.0170`, Random Forest `0.9850 ± 0.0141`.
- **최종 Test Accuracy: 1.0000** (test 67개 전부 정확 분류, precision/recall/f1 모두 1.00).
- **Feature importance 순위(요지)**: `culmen_length_mm`(1위) > `culmen_depth_mm` / `body_mass_g` > `flipper_length_mm` > `island_*` > `sex`(≈0).

> ⚠️ Test Accuracy 100%는 데이터셋이 작고(test 67개) 종 간 분리가 비교적 뚜렷해서 나온 결과. "완벽한 모델"이라기보다 "이 데이터에선 분리가 쉬웠다"로 읽는 게 맞다.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| EDA 가설(핵심=culmen_length)이 importance와 일치 | island를 강한 feature로 과대평가 — 다변량 안 기여를 못 읽음 |
| 모델 선택을 교차검증 수치(평균±표준편차)로 근거 있게 결정 | sex 유지 가설을 "제거 후 비교"로 검증 못 함 |
| 인코딩을 범주 성질(이진=Label, 무순서=One-Hot)에 맞게 구분 | 하이퍼파라미터 기본값 사용 — 튜닝 안 함 |

---

## 다음에 이어서 할 것
- `sex` 제거 후 성능 비교 → 보조 feature 가설 검증.
- `island` 제외 후 성능 변화 확인 → "수치 feature가 island 정보를 이미 담고 있다"는 추론을 인과적으로 확인.
- `GridSearchCV`로 RF 하이퍼파라미터 튜닝.

## 다음에 파고들 만한 질문
> 

---
*작성일: 2026-06-19 | 인공지능 사관학교 AI 보안 과정*
