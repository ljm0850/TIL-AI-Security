# 2026-07-24 실습 — MLP 기반 침입 탐지 다중 분류 (ex06)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사님 코드 따라 작성)
> **관련 강의 파일**: `../lectures/2026-07-24_AI-Security-Course_Lecture_Anomaly-Detection-and-Deep-Learning.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-24

- **은닉층 하나짜리 얕은 MLP인데도 성능이 이 정도라는 건 데이터가 쉽다는 뜻이다.** 구조는 `Dense(32) → Dropout(0.2) → Dense(5)`가 전부다. 모델이 좋아서가 아니라 클래스별 분포가 갈려 있어서 나온 숫자로 봐야 한다.
- **Softmax 확률이 "얼마나 확신하는지"를 보여준다.** DoS·U2R은 1.000으로 단정했지만 Probe는 0.941, 정상은 0.974로 상대적으로 덜 확신했다. 이진 판정만 보면 안 보이는 정보라, 확률이 애매한 구간을 따로 다루는 게 실무적으로 의미 있어 보인다.

---

## 실습 개요
- **목표**: 침입 탐지를 이진 분류에서 5개 클래스 다중 분류로 확장하고, 딥러닝(MLP) 기반 분류 모델을 구축한다.
- **분류 대상**: Normal(정상) / DoS(서비스 거부) / Probe(정찰) / R2L(원격 접속) / U2R(권한 상승)
- **방법**: NSL-KDD 모방 데이터 + Keras MLP (Softmax + Categorical Cross-Entropy).

---

## 진행 과정

```
Step 1. 데이터 생성   # make_nsl_kdd(2000)
                     # 비율: Normal 45% / DoS 30% / Probe 12% / R2L 8% / U2R 5%
Step 2. 특성 선택      # 8개: duration, src_bytes, dst_bytes, hot,
                     #      num_failed_logins, num_compromised, count, serror_rate
Step 3. 라벨 인코딩    # LabelEncoder → 정수, 이후 to_categorical로 원-핫
Step 4. 분할·정규화    # test_size=0.3, stratify / StandardScaler(train에만 fit)
Step 5. 모델 구성      # Dense(32, relu) → Dropout(0.2) → Dense(5, softmax)
Step 6. 학습          # Adam(1e-3), categorical_crossentropy
                     # epochs=40, batch=64, validation_split=0.15
                     # EarlyStopping(patience=6, restore_best_weights)
Step 7. 평가          # classification_report + confusion_matrix
Step 8. 5개 유형 샘플에 대한 클래스별 확률 확인
```

**실제 출력 (tf.random.set_seed(42), np.random.seed(42) 고정)**
```
              precision    recall  f1-score   support
         DoS      1.00      1.00      1.00       180
      Normal      1.00      1.00      1.00       270
       Probe      1.00      1.00      1.00        72
         R2L      0.98      1.00      0.99        48
         U2R      1.00      0.97      0.98        30
    accuracy                           1.00       600

혼동행렬 (행=실제, 열=예측 / 순서: DoS, Normal, Probe, R2L, U2R)
  [[180   0   0   0   0]
   [  0 270   0   0   0]
   [  0   0  72   0   0]
   [  0   0   0  48   0]
   [  0   0   0   1  29]]   ← U2R 1건을 R2L로 오분류 (유일한 오류)

학습 추이 (총 40 에폭)
  초반 (ep  1): train=0.2924  val=0.4286
  중반 (ep 20): train=0.9832  val=0.9952
  최종 (ep 40): train=0.9941  val=0.9952

유형별 예측 확률 (Softmax)
  정상 트래픽  → Normal 0.974  (Probe 0.018)
  DoS 공격    → DoS    1.000
  Probe 스캔  → Probe  0.941  (Normal 0.048)
  R2L 공격    → R2L    0.999
  U2R 공격    → U2R    1.000
```

> ※ 강의 "핵심 구현 요소"에는 Dense·**Batch Normalization**·Dropout 레이어 구성이 적혀 있으나, 실제 실습 코드에는 BatchNormalization이 없다. Dropout만 적용됨.

---

## 결과 / 산출물
- 5클래스 침입 탐지 MLP 모델과 클래스별 성능 리포트.
- 혼동 행렬로 확인한 유일한 혼동 지점(U2R ↔ R2L).
- 공격 유형별 Softmax 확률 분포.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 이진 → 다중 분류 확장의 전체 흐름(원-핫, softmax, CCE)을 끝까지 따라감 | 데이터가 쉬워 모델 구조·튜닝의 효과를 확인할 수 없음 |
| 소수 클래스(U2R)에서 오류가 난다는 걸 혼동 행렬로 확인 |  |
| Softmax 확률로 모델의 확신 정도를 읽음 |  |

---
*작성일: 2026-07-24 | 인공지능 사관학교 AI 보안 과정*
