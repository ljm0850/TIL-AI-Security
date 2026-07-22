# 2026-07-22 실습 — 스팸 메일 · 피싱 URL 탐지 (ex01, ex02)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사님 코드 따라 작성)
> **관련 강의 파일**: `../lectures/2026-07-22_AI-Security-Course_Lecture_ML-Threat-Detection.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-22

> `ex0X_data.py`는 강사님이 통째로 주신 데이터, `ex0X_분류.py`는 강사님 코드를 따라(복붙 포함) 작성했다. 코드 자체보다 흐름과 로직 중심으로 이해하는 방식이었다.

- **ex01이 정확도 100%로 나온 게 오히려 신호였다.** 데이터가 `topic × action` 템플릿으로 증강돼서, 모델이 '실제 스팸'이 아니라 '템플릿 패턴'을 외운 것. 핵심 판별 단어가 offer/limited/urgent/notice로 죄다 증강 문장의 부품이었다. 현실 데이터라면 이렇게 깔끔하게 안 나뉜다.
- **ex02는 판별력이 모델이 아니라 특징 공학에서 나온다는 걸 체감했다.** 실제 판별은 `extract_url_features` 안의 사람이 짠 규칙(IP 정규식, 의심 TLD, 피싱 키워드)이 하고, 랜덤포레스트는 그 위에서 투표만 한다.
- **ex02 데이터에 HTTPS 피싱 / HTTP 정상 샘플을 일부러 섞은 설계가 인상적이었다.** 'HTTPS면 안전'이라는 지름길을 모델이 학습하지 못하게 막으려는 의도. 실제로 특징 중요도에서 `is_https`가 1등이 아니라 3등(0.181)으로 눌린 걸 보면 그 의도가 먹혔다.
- **팀 프로젝트 관점**: 모델 성능만 보지 말고 오탐·미탐 관점으로 평가 지표를 봐야겠다. 애매한 예측(확률 경계)을 어떻게 다룰지도 설계에 넣을 지점.

---

## 실습 개요
- **목표**: ML로 스팸 메일·피싱 URL 탐지기를 구축하고, 코드보다 전처리→벡터화/특징추출→학습→평가의 흐름을 이해한다.
- **방법**: scikit-learn.
  - ex01: `TfidfVectorizer` + `LogisticRegression`
  - ex02: 수동 특징 추출 + `RandomForestClassifier`

---

## 진행 과정

### ex01. 스팸 메일 분류 (TF-IDF + 로지스틱 회귀)

```
Step 1. 데이터 준비   # spam/ham 원본 + topic×action 템플릿으로 증강
Step 2. 학습/테스트 분할  # test_size=0.3, stratify=label (클래스 비율 유지)
Step 3. TF-IDF 벡터화  # max_features=500, stop_words="english", lowercase
Step 4. 로지스틱 회귀 학습  # max_iter=1000
Step 5. 평가        # classification_report + confusion_matrix
Step 6. 핵심 단어    # 계수 상위 5(스팸) / 하위 5(정상)
Step 7. 새 메일 예측  # predict + predict_proba(스팸 확률)
```

**실제 출력 (random_state=42 고정)**
```
학습데이터:71개 | 테스트데이터:31개
TF-IDF 행렬크기: (71, 133)

              precision    recall  f1-score   support
     Ham(정상)      1.00      1.00      1.00        16
    Spam(스팸)      1.00      1.00      1.00        15
    accuracy                           1.00        31

혼동 행렬
  실제 \ 예측   정상(Ham)  스팸(Spam)
  정상(Ham)       16          0   ← 오탐 0
  스팸(Spam)       0         15   ← 미탐 0

핵심 판별 단어(계수)
  스팸 방향: offer(1.06), limited(1.06), receive(1.02), urgent(1.02), notice(1.02)
  정상 방향: business(-1.11), normal(-1.06), regarding(-1.06), hours(-0.96), update(-0.48)

새 메일 예측
  "Win a free gift card! Click here..."       → 스팸 (64.8%)
  "Please review the attached proposal..."    → 정상 (38.4%)
```

### ex02. 피싱 URL 탐지 (특징 공학 + 랜덤포레스트)

```
Step 1. 데이터 로딩   # 피싱/정상 URL + [편향 방지] HTTPS 피싱·HTTP 정상 샘플 포함
Step 2. 특징 추출     # extract_url_features()로 17개 특징 생성
Step 3. 학습 특성 선택  # feature_cols = 6개만 사용
Step 4. 학습/테스트 분할  # test_size=0.3, stratify, class_weight="balanced"
Step 5. 랜덤포레스트 학습  # n_estimators=100, max_depth=10
Step 6. 평가 + 특징 중요도  # ※ 제공 코드가 predict에서 끊겨 있어, 강의 '주요 단계'대로 이어붙여 실행
```

**실제 출력**
```
              precision    recall  f1-score   support
   Legit(정상)      1.00      0.86      0.92         7
Phishing(피싱)      0.88      1.00      0.93         7
    accuracy                           0.93        14

혼동행렬
  [[6 1]     ← 정상 7개 중 1개를 피싱으로 오탐
   [0 7]]    ← 피싱 7개 전부 탐지 (미탐 0)

특징 중요도(feature_importances_)
  url_length         0.388
  num_hyphens        0.277
  is_https           0.181
  num_digits         0.066
  phishing_keywords  0.064
  has_ip             0.024
```

---

## 결과 / 산출물
- ex01: 성능 리포트, 혼동 행렬, 스팸/정상 핵심 단어, 새 메일 확률 예측.
- ex02: 성능 리포트, 혼동 행렬, 특징 중요도 순위.
- 공통: 판별의 근거(단어 계수 / 특징 중요도)를 눈으로 확인 → 왜 그렇게 분류했는지 감사 가능.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 전처리→벡터화/특징추출→학습→평가 흐름을 끝까지 이해 | ex01 정확도 100%는 데이터가 너무 쉬움(현실성 부족) |
| 특징 공학이 탐지 성능의 실제 주역임을 체감 | ex02 추출 특징 17개 중 6개만 학습에 사용(나머지 미활용) |
| 편향 방지 샘플 설계 의도를 특징 중요도로 확인 |  |

---
*작성일: 2026-07-22 | 인공지능 사관학교 AI 보안 과정*
