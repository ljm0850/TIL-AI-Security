# 2026-06-22 강의 정리 — 케라스(Keras)와 CNN

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 케라스 모델 API, 드롭아웃/조기종료, Layers(Dense/Conv/Pooling/Flatten/RNN/Embedding), CNN(합성곱·필터·스트라이드·피쳐맵·풀링·패딩)
> **관련 실습 파일**: `../practice/2026-06-22_AI-Security-Course_Practice_MNIST-DNN-vs-CNN.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-22

- **케라스는 학습 메커니즘을 "조립식"으로 추상화한 것** 손실함수·옵티마이저·층을 직접 구현하는 대신 `model.add(Dense(...))` → `compile(loss, optimizer)` → `fit()`으로 끝난다. 
- **드롭아웃·조기종료는 과적합 문제에 대한 딥러닝식 규제** : 회귀에서 과적합을 Lasso/Ridge(가중치 페널티)로 눌렀다면, 딥러닝은 **드롭아웃**(학습 중 뉴런을 무작위로 끔)과 **조기종료**(검증 성능이 patience 동안 안 오르면 중단)로 누른다. 도구는 달라도 "모델이 학습 데이터에 과하게 맞는 걸 막는다"는 의도는 회귀의 규제와 똑같다
- **CNN은 "이미지를 1차원으로 펴면 사라지는 2차원 정보"를 지키려는 구조다.** 일반 DNN은 이미지를 Flatten해서 픽셀 간 공간 관계(특징)를 잃는다. 게다가 고해상도면 입력·가중치가 폭발하고, 픽셀에 민감해 조금만 달라도 다른 이미지로 본다. CNN은 합성곱(필터)으로 **공간 구조를 유지한 채 특징만 추출**하고, 풀링으로 크기를 줄인다. "전부 연결"이 아니라 "필터가 훑고 지나간다"는 발상의 전환.
- **필터는 처음엔 무작위였다가 학습으로 "특징 검출기"가 된다.** 필터=합성곱에 쓰는 가중치 집합, 초기엔 랜덤이고 학습이 진행되며 엣지·곡선 같은 특징을 잡는 값으로 수렴한다. 즉 CNN의 학습 = "어떤 필터를 가질지"를 경사하강으로 찾는 것. 어제 배운 가중치 학습이 여기선 "필터 학습"으로 나타난다.

---

## 1. 케라스(Keras) 모델

### 핵심 개념
- 모델(model)과 계층(Layer)을 모듈 방식으로 조립해 직관적으로 딥러닝 모델을 만드는 라이브러리. `tensorflow.keras` 사용 권장.
- 모델 생성 방식 3가지: **Sequential / Functional / SubClass(상속)**.

### 상세 내용
- **주요 메서드**: `compile`(학습 환경 설정 - optimizer/loss/metrics), `fit`(학습, History 반환), `evaluate`(성능 평가), `predict`(예측), `summary`(구조 요약).
  - `compile(optimizer, loss, metrics)`: optimizer 기본 rmsprop / loss는 mse·categorical_crossentropy·sparse_categorical_crossentropy·binary_crossentropy 등 / metrics는 보통 `['accuracy']`.
  - `fit(x, y, batch_size, epochs, verbose, callbacks, validation_split, shuffle)`: batch_size 미지정 시 32, validation_split로 검증 비율 지정.
- **SubClass**: `tf.keras.Model` 상속, `call(self, inputs)`이 객체 호출 시 자동 실행 (Python `__call__` 발상과 동일).
- **저장/불러오기**: `model.save('model.h5')` / `tf.keras.models.load_model(...)`. 가중치만: `save_weights`/`load_weights`.

---

## 2. 과적합 대응 (드롭아웃 · 조기종료)

### 핵심 개념
- **드롭아웃(Dropout)**: 학습 시 완전연결층의 뉴런 일부를 **무작위로 제거**해 과적합을 막는 방법. (테스트/예측 때는 적용 안 함.) `Dropout(rate)`, rate 0~1.
- **조기종료(Early Stopping)**: 학습을 계속해도 성능이 안 오르면 중단. 과도한 에포크=과적합, 부족=과소적합.

### 상세 내용
- **EarlyStopping**: `monitor`(기본 val_loss), `patience`(정체 허용 횟수), `restore_best_weights`(최적 가중치 복원), `mode`(min/max/auto).
- **ModelCheckpoint**: 모델/가중치를 주기적으로 저장. `save_best_only=True`면 최고 성능만 저장.
- 둘 다 `fit(callbacks=[...])`로 함께 사용.

---

## 3. Layers

### 핵심 개념
- 케라스가 제공하는 다양한 층: Dense, Conv, Pooling, Flatten, LSTM, Embedding 등.

### 상세 내용
- **Dense**: 전결합층 생성. 출력 유닛 수, 활성화 함수, 가중치/편향 초기화, 입력 차원 지정.
- **Conv(Convolution)**: CNN 구현용(주로 비전). `Conv2D(filters, kernel_size, strides, padding, activation, input_shape)`. input_shape=(행,열,채널), 채널 1=흑백/3=RGB.
- **Pooling**: 영역에서 최대값(MaxPooling)이나 평균(AveragePooling)으로 크기 축소. 주로 MaxPooling2D.
- **Flatten**: 다차원을 1차원(전결합층 입력용)으로 변환. 일반 적용 순서: `[Conv2D → MaxPooling2D] × n → Flatten → Dense`.
- **Recurrent(RNN)**: 시계열/시퀀스 처리용. 대표 LSTM/GRU. (LSTM은 RNN의 기울기 소실 문제 해결용 — *오늘은 도입부만, 다음 강의에서 이어짐*)
- **Embedding**: NLP에서 토큰화된 단어를 실수 밀집행렬로 변환. one-hot 없이 처리해 고차원 문제 회피.

---

## 4. CNN (합성곱 신경망, Convolutional Neural Network)

### 핵심 개념
- 컴퓨터 비전에 강한 신경망. 일반 DNN(전결합)의 한계를 해결:
  - 고해상도 이미지면 입력·가중치가 폭발적으로 증가(1000×1000×3…).
  - 픽셀에 민감해 조금만 달라도 다른 이미지로 인식.
  - 이미지를 1차원으로 펴면 2차원 정보(공간적 관계성·특징)가 사라짐.
- **합성곱**으로 이미지 특징을 유지한 채 원하는 데이터로 변환.

### 상세 내용
- **필터(Filter)/커널(Kernel)**: 합성곱에 적용할 **가중치 집합**. 초기엔 무작위, 학습이 진행될수록 특징 정보를 갖게 됨.
- **수용영역(Receptive field)**: 필터가 적용되는 영역.
- **스트라이드(Stride)**: 필터를 오른쪽·아래로 순차 이동하는 것(이동 폭).
- **피쳐맵(Feature map)**: 필터를 스트라이드한 결과(합성곱 출력).
- **풀링(Pooling)**: 과적합·연산을 줄이려 영역의 최대/평균값으로 크기 축소.
- **제로 패딩(Zero Padding)**: 모서리는 필터가 한 번만 적용되는 불균형을 막으려, 상하좌우에 0을 덧대 적용을 균일하게. (`padding='same'`)
- **일반 구조**: `[Conv → ReLU → Pool] × n → Flatten → Fully-Connected → softmax`.

---
*작성일: 2026-06-22 | 인공지능 사관학교 AI 보안 과정*
