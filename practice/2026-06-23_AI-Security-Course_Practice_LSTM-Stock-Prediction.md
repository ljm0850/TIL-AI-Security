# 2026-06-23 실습 — LSTM 주가 예측 (롤링 윈도우)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (딥러닝, 오늘 DL 중 유일한 실습)
> **관련 강의 파일**: `../lectures/2026-06-23_AI-Security-Course_Lecture_RNN-LSTM-GRU.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-23

- **"시계열을 지도학습으로 바꾸는" 핵심이 롤링 윈도우였다.** 시계열은 그대로면 X/y가 없는데, **윈도우 크기(20일)만큼 잘라 X로, 그 다음날 종가를 y로** 만들면 평범한 지도학습 문제가 된다. 즉 "과거 20일 → 다음날 예측"을 한 칸씩 밀며(슬라이딩) 샘플을 대량 생성. RNN/LSTM이 "순서"를 다룬다는 강의 개념이, 데이터를 (samples, 20, 4) 모양으로 만드는 전처리로 구체화됐다.
- **윈도우를 만든 뒤엔 샘플을 섞어도 된다는 걸 이해했다.** `train_test_split`에 shuffle을 썼는데, 처음엔 "시계열인데 섞어도 되나?" 싶었다. 그런데 **각 샘플(윈도우) 안에 이미 20일치 순서 정보가 통째로 들어있어서**, 윈도우들끼리의 순서는 섞여도 학습엔 문제없다. 순서가 보존돼야 하는 단위가 "전체"가 아니라 "윈도우 내부"라는 게 포인트.
- **"데이터 누수" 의문이 실물로 보였다.** 이 코드는 `MinMaxScaler`를 **전체 데이터(df)에 fit한 뒤** train/test를 나눴다. 그러면 스케일러가 test 구간(미래 주가)의 min/max를 미리 본 셈이라, 엄밀히는 미래 정보가 스케일에 새어든 **데이터 누수**다. 특히 주가는 미래에 신고가가 나오면 과거 정규화 값이 통째로 바뀌니 더 민감하다. "Pipeline에서 스케일러가 train만 보고 fit돼야 누수가 없다"고 궁금해했던 게, 정확히 이 장면이었다. (강의용 단순화로 보이지만, 실무라면 train으로 fit → test에 transform이 맞다.)
- **회귀 평가는 정확도가 아니라 손실(mse)로 본다는 게 분류와 다른 점.** 종가는 연속값이라 `loss='mse'`만 쓰고 metrics는 안 걸었다. 시각화로 실제 vs 예측 곡선을 겹쳐 "추세를 따라가는지"를 눈으로 확인하는 흐름이 분류(정확도 숫자)와 달랐다.

---

## 실습 개요
- **목표**: NVDA 주가 시계열로 다음날 종가(Close)를 예측하는 LSTM 모델 제작.
- **데이터**: `NVDA.csv` (2014-01-02 ~ 2024-04-19, 2592행, 결측치 없음, Open/High/Low/Close/Adj Close/Volume).
- **방법**: 정규화 → 시간순 train/test 분리 → 롤링 윈도우로 (samples, 20, 4) 생성 → 스택 LSTM 학습 → 예측 곡선 비교.

---

## 진행 과정

### Step 1. 데이터 확인·분석
- `info()`: 시계열(DatetimeIndex), 결측치 없음. `corr()`: Open/High/Low/Close가 0.999로 거의 완전 상관(주가 컬럼들끼리), Volume은 약함.
- `lineplot`으로 High/Low/Close 추이 시각화.

### Step 2. 전처리 (정규화)
```python
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
scaler.fit(df)                 # ⚠️ 전체 데이터에 fit → test 정보 누수 소지 (위 인사이트 참고)
df_scaled = scaler.transform(df)
```

### Step 3. 시간순 분리 (셔플 X)
```python
TEST_SIZE = 500
df_train = df_data.iloc[:-TEST_SIZE].copy()   # 앞쪽
df_test  = df_data.iloc[-TEST_SIZE:].copy()   # 뒤쪽 500일 = 검증
```

### Step 4. 롤링 윈도우 → 지도학습 형태로 변환
```python
def get_rolling_data(data, label, window_size):
    X_list, y_list = [], []
    for i in range(len(data) - window_size):
        X_list.append(np.array(data.iloc[i:i+window_size]))  # 과거 window_size일
        y_list.append(np.array(label.iloc[i+window_size]))   # 그 다음날 = 정답
    return np.array(X_list), np.array(y_list)

X_feature = ['High','Low','Open','Volume']; y_feature = ['Close']
X_data, y_data = get_rolling_data(df_train[X_feature], df_train[y_feature], 20)
# X_data.shape = (2072, 20, 4), y_data.shape = (2072, 1)
```
- **본인 의문(메모)**: 이 방식은 윈도우를 매번 통째로 copy해서 쌓는데(앞뒤 pop/append가 아니라), 메모리·복사 비용이 크지 않을까? → "다음에 파고들 질문"으로 남김.

### Step 5. 모델 설계 (스택 LSTM)
```python
model = Sequential([
    Input(shape=(20, 4)),
    LSTM(64, return_sequences=True),   # 시퀀스 전체를 다음 층에 전달
    LSTM(32, return_sequences=False),  # 마지막 출력만
    Dense(1)
])
model.compile(loss='mse', optimizer=Adam())
```
- `return_sequences=True`: 매 시점 출력을 다음 LSTM에 넘김(LSTM 쌓을 때 필요). 마지막 LSTM은 False로 최종 벡터 하나만. 파라미터 30,113개.

### Step 6. 학습·예측·시각화
- `EarlyStopping(monitor='val_loss', patience=5)`, `fit(epochs=200, batch_size=100, validation_data=(X_test,y_test))`.
- `val_loss`가 빠르게 감소(6e-4 → 8e-5 수준). `predict(X_valid)` 후 실제 vs 예측 곡선 비교.

---

## 결과 / 산출물
- 검증 손실(val_loss)이 안정적으로 낮아짐 (≈1e-4 수준).
- 예측 곡선이 실제 종가 추세를 대체로 따라감 (시각화로 확인).

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 롤링 윈도우로 시계열→지도학습 변환을 직접 구현 | MinMaxScaler를 전체에 fit — 데이터 누수 소지(train fit이 맞음) |
| "윈도우 내부 순서 보존 → 윈도우끼리는 셔플 OK" 이해 | 단순 베이스라인(어제값=예측) 대비 성능 비교는 안 함 |
| return_sequences로 LSTM을 쌓는 법 습득 | 예측이 "지연(lag) 패턴"인지 검증 못 함 |

---

## 다음에 이어서 할 것
- 스케일러를 train에만 fit → test엔 transform으로 바꿔 누수 제거 후 성능 변화 비교.
- "어제 종가 = 오늘 예측" 단순 베이스라인과 mse 비교해 LSTM이 실제로 더 나은지 확인.

## 파고든 것 (deep-dive)

> 아래 질문은 실제로 벤치마크해서 파고들었음 → 별도 deep-dive로 분리.

- **롤링 윈도우 생성의 메모리·속도 효율** → `../deep-dive/2026-06-23_Rolling-Window-Efficiency.md`
  - 요지: 병목은 copy가 아니라 **pandas `.iloc` 반복**(순수 numpy로 60배↑). deque 양방향은 *요약형* 슬라이딩에서만 이득이고, *윈도우 보존형*(LSTM 입력)은 `sliding_window_view`(복사 없는 뷰)가 정답(속도 ~5000배·메모리 ~0, 단 읽기 전용 가정).

---
*작성일: 2026-06-23 | 인공지능 사관학교 AI 보안 과정*
