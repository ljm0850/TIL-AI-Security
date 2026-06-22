# 2026-06-22 실습 — XOR: 게이트 조합 vs MLP 학습

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (딥러닝)
> **관련 강의 파일**: `../lectures/2026-06-22_AI-Security-Course_Lecture_DNN-Training.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-22

- **XOR을 푸는 두 길을 나란히 해보니 "수동 설계 → 학습"의 전환이 보였다.** dl_01은 NAND·OR·AND를 *내가 직접 조합*해서 XOR을 만들었고(가중치를 사람이 정함), dl_02는 같은 XOR을 *MLP가 학습으로* 풀게 했다(가중치를 경사하강이 찾음). 금요일에 "단일 퍼셉트론은 XOR 못 푼다 → 다층으로"라고 적어둔 게, 오늘 두 방식으로 실제로 닫혔다.

- **dl_02 MLP가 100 epoch으로는 XOR을 못 배웠다**. 예측이 전부 ≈0.5(0.4949, 0.4981, 0.5044, 0.5059), accuracy도 0.5. 손실도 0.6924에서 거의 안 움직였다. 강의 슬라이드는 같은 구조에 **50000 epoch**을 썼다. 즉 모델 구조가 맞아도 "충분히 학습"되지 않으면 못 푼다 — (구조의 표현력 ≠ 실제 학습 완료.)

- **`compile`이 "손실함수 + 옵티마이저"의 두 축.** `compile(loss='binary_crossentropy', optimizer='sgd')` — 손실(무엇을 줄일지)과 옵티마이저(어떻게 줄일지)를 한 줄에 지정. 별개 개념으로 정리한 두 축이 케라스 API에선 인자 두 개로 바로 나타난다.

---

## 실습 개요
- **목표**: XOR 문제를 (1) 퍼셉트론 게이트 수동 조합과 (2) MLP 학습 두 방식으로 풀어보고, 단층의 한계와 다층의 해법을 직접 확인.
- **방법**: dl_01 — `pr_nand`/`pr_or`/`pr_and` 조합으로 `pr_xor` 구현. dl_02 — keras Sequential로 2-2-1 MLP 구성 후 학습.

---

## 진행 과정

### Step 1. dl_01 — 게이트 조합으로 XOR (수동)
```python
def pr_xor(x1, x2):
    r1, _ = pr_nand(x1, x2)   # NAND
    r2, _ = pr_or(x1, x2)     # OR
    result = pr_and(r1, r2)   # AND(NAND, OR)
    return (result, r1, r2)
```
- 결과: (0,0)→0, (0,1)→1, (1,0)→1, (1,1)→0 ✅ XOR 정확히 구현.
- 핵심: 가중치를 사람이 미리 정한 NAND·OR·AND를 **은닉층처럼 조합** → 선형 분리 불가 문제를 층으로 우회.

### Step 2. dl_02 — MLP로 XOR (학습)
```python
model = Sequential()
model.add(Input(shape=(2,)))
model.add(Dense(units=2, activation='sigmoid'))   # 은닉층 (OR/NAND 역할 학습)
model.add(Dense(units=1, activation='sigmoid'))   # 출력층 (AND 역할 학습)
model.compile(loss='binary_crossentropy', optimizer='sgd', metrics=['accuracy'])
model.fit(X, y, epochs=100, batch_size=4, verbose=1)
```
- 파라미터 수: 9개 (2-2-1 망). 총 4개 샘플을 batch_size=4로 학습.
- `get_weights()`로 은닉층/출력층 가중치 확인.

---

## 결과 / 산출물
- **dl_01(수동)**: XOR 진리표 완벽 재현.
- **dl_02(학습, 100 epoch)**: 예측 ≈ [0.49, 0.50, 0.50, 0.51], accuracy 0.5, loss ≈ 0.692 → **사실상 미학습**. (슬라이드는 50000 epoch 사용.)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 게이트 조합으로 XOR을 정확히 구현 (다층 = 비선형 분리 직관 확보) | MLP가 100 epoch으론 미학습 — 예측이 전부 ≈0.5 |
| `compile`의 loss/optimizer가 두 축과 직접 매핑됨을 확인 |  |

---
*작성일: 2026-06-22 | 인공지능 사관학교 AI 보안 과정*
