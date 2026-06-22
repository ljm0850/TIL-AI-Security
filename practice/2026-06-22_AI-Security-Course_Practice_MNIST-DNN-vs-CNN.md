# 2026-06-22 실습 — MNIST 손글씨 분류: DNN vs CNN 비교

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (딥러닝)
> **관련 강의 파일**: `../lectures/2026-06-22_AI-Security-Course_Lecture_DNN-Training.md`, `../lectures/2026-06-22_AI-Security-Course_Lecture_Keras-CNN.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-22

- **같은 MNIST를, 같은 학습 설정으로 DNN과 CNN에 돌려 비교하니 "왜 이미지엔 CNN인가"가 숫자로도 보였다.** DNN test accuracy **0.9814**(파라미터 235,146개), CNN **0.9890**(파라미터 215,370개). **CNN이 파라미터를 더 적게 쓰고도 더 정확**했다. DNN은 이미지를 Flatten해서 픽셀 공간관계를 잃지만, CNN은 합성곱으로 공간 구조를 유지한 채 특징을 뽑기 때문 — 강의에서 들은 "2차원 정보 보존"이 정확도 차이로 드러났다.
- **정규화(/255)의 의미가 와닿았다.** 0~255 픽셀을 255로 나눠 0~1로 만드는데, 이게 어제 배운 활성화함수(sigmoid/softmax가 0~1 다루는 것)·경사하강 안정성과 맞물린다. 입력 스케일을 맞춰야 학습이 안정적으로 수렴한다.
- **조기종료 + 체크포인트로 "필요한 만큼만 학습"하는 패턴을 익혔다.** `epochs=500`을 걸어도 EarlyStopping(patience=5)이 검증 성능 정체를 감지해 멈추고, ModelCheckpoint가 최고 성능 모델만 저장. 어제 배운 과적합/조기종료가 실제 콜백 코드로 들어오니 "에포크를 크게 잡고 알아서 멈추게" 하는 게 표준 흐름임을 이해.
- **AI 보안 관점 — test accuracy 98~99%가 "안전"을 뜻하진 않는다.** 두 모델 다 깨끗한 MNIST에선 매우 정확하지만, 강의에서 본 "픽셀에 민감" 성질을 생각하면 미세하게 조작된 입력(적대적 예제)엔 무너질 수 있다. 정확도 한 숫자로 모델을 신뢰하기보다, 입력을 흔들었을 때의 강인함을 따로 봐야 한다

---

## 실습 개요
- **목표**: MNIST 손글씨(0~9) 분류를 DNN과 CNN 두 구조로 만들어 성능·구조·전처리 차이를 비교.
- **데이터**: MNIST (train 60000, test 10000, 28×28 흑백).
- **방법**: 정규화 → 모델 설계(DNN/CNN) → compile → EarlyStopping+Checkpoint로 학습 → evaluate.

---

## 진행 과정

### Step 1. 공통 — 데이터 로딩·확인·정규화
- `tf.keras.datasets.mnist.load_data()` → shape (60000,28,28).
- `imshow`로 샘플 확인, `astype('float32')` 후 `/255`로 0~1 정규화.

### Step 2. DNN (dl_03)
```python
model = Sequential([
    Input(shape=(28,28)),
    Flatten(),                      # 28x28 → 784 (1차원)
    Dense(256, activation='relu'),
    Dense(128, activation='relu'),
    Dense(10),
    Activation('softmax')
])
model.compile(loss='sparse_categorical_crossentropy', optimizer='adam', metrics=['accuracy'])
```
- 정답 정수 그대로 → `sparse_categorical_crossentropy`. 파라미터 235,146개.

### Step 3. CNN (dl_04)
```python
X_train = np.reshape(X_train, (-1, 28, 28, 1))   # 채널 축 추가
y_train = keras.utils.to_categorical(y_train, 10) # one-hot

model = Sequential([
    Input(shape=(28,28,1)),
    Conv2D(16, kernel_size=(5,5), activation='relu', padding='same'),
    MaxPooling2D(pool_size=(2,2)),
    Conv2D(32, kernel_size=(5,5), activation='relu', padding='same'),
    MaxPooling2D(pool_size=(2,2)),
    Flatten(),
    Dense(128, activation='relu'),
    Dense(10, activation='softmax')
])
model.compile(loss=categorical_crossentropy, optimizer=Adam(), metrics=['accuracy'])
```
- one-hot 정답 → `categorical_crossentropy`. `padding='same'`으로 모서리 적용 불균형 보정. 파라미터 215,370개.

### Step 4. 공통 — 학습·평가
- `callbacks=[EarlyStopping(monitor='val_accuracy', patience=5), ModelCheckpoint('bestmodel.keras', save_best_only=True)]`
- `fit(epochs=500, batch_size=1000, validation_split=0.1, callbacks=...)` → accuracy/val_accuracy 곡선 시각화 → `evaluate(X_test, y_test)`.

---

## 결과 / 산출물

| 모델 | 파라미터 수 | Test Accuracy | 정답 인코딩 / 손실함수 |
|---|---|---|---|
| DNN | 235,146 | **0.9814** | 정수 / sparse_categorical_crossentropy |
| CNN | 215,370 | **0.9890** | one-hot / categorical_crossentropy |

- CNN이 **더 적은 파라미터로 더 높은 정확도** → 이미지엔 공간 구조를 보존하는 합성곱이 유리.
- CNN 예측 시각화: test 앞 16개 라벨 vs 예측 비교 → 대부분 일치.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 동일 데이터로 DNN/CNN을 비교해 "이미지엔 CNN" 근거를 숫자로 확보 | 하이퍼파라미터(필터 수·커널 크기·학습률) 튜닝은 안 함 |
| 구조별 전처리 차이(채널 추가/Flatten, one-hot/정수)를 정리 | 오분류 사례를 들여다보며 원인 분석까진 못 감 |
| 조기종료+체크포인트로 학습 자동 중단·최적 저장 패턴 습득 | 적대적 입력 등 강인성 관점 실험은 못 해봄(인식만) |

---
*작성일: 2026-06-22 | 인공지능 사관학교 AI 보안 과정*
