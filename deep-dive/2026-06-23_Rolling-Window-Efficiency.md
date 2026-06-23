# 2026-06-23 심화 정리 — 롤링 윈도우 생성의 메모리·속도 효율

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 실습 파일**: `../practice/2026-06-23_AI-Security-Course_Practice_LSTM-Stock-Prediction.md`
> **작성 계기**: LSTM 실습에서 롤링 윈도우를 매 스텝 통째로 copy해 쌓는 게 비효율로 느껴짐. deque 같은 양방향 자료구조나 numpy stride 뷰로 줄일 수 있을지 직접 벤치마크.

---

## 왜 찾아봤는가

- dl_05 LSTM 실습의 `get_rolling_data`는 `for i in range(...)`로 매번 `data.iloc[i:i+window]`를 `np.array`로 떠서 리스트에 append하고, 마지막에 `np.array(X_list)`로 합친다.
- 윈도우가 20일씩 겹치는데 매번 통째로 복사하니 "메모리·속도 낭비 아닐까?"라는 의심이 들었다.
- 처음엔 "앞 pop / 뒤 append" 슬라이딩(특히 `deque` 양방향 연산)으로 줄일 수 있을 거라 생각 → 실제로 어떤지 측정.

---

## 핵심 내용

### 1. 벤치마크 (train 2092행 → 윈도우 2072개, window=20, feature=4)

| 방식 | 속도 | 추가 메모리(peak) | 비고 |
|---|---|---|---|
| ① pandas `.iloc` + `np.array` append (실습 코드) | **106.55 ms** | 3.23 MB | base=False (복사) |
| ② numpy 미리 할당 후 슬라이싱 | **1.76 ms** | 1.28 MB | base=False (복사) |
| ③ `np.lib.stride_tricks.sliding_window_view` (뷰) | **0.02 ms** | ~0 MB | base=True (복사 안 함) |

→ ②는 ①의 약 **60배**, ③은 약 **5000배** 빠르다. 메모리도 ③은 사실상 0.

### 2. 실습 코드가 느린 진짜 이유는 "copy"가 아니라 "pandas 경유"

- 범인은 윈도우 복사 자체가 아니라 **루프마다 `pandas.iloc` 슬라이싱 + `np.array` 변환**의 오버헤드.
- ②는 여전히 데이터를 *복사*하지만 pandas를 안 거치고 순수 numpy 슬라이싱만으로 60배 빨라졌다 → 병목이 pandas 변환이었다는 증거.
- `X_list.append(...)` 후 `np.array(X_list)`로 합칠 때 전체를 한 번 더 복사한다.

### 3. deque(양방향 pop/append)는 "요약 vs 보존" 갈림길에서 갈린다

- `collections.deque`의 양쪽 끝 O(1) pop/append는 **윈도우의 *요약값 하나*만 필요할 때** 위력을 낸다.
  - 예: 윈도우 최댓값/최솟값(sliding window maximum, monotonic deque로 O(N)), 윈도우 합/평균(나간 값 빼고 들어온 값 더하기).
  - 이땐 윈도우 내용물을 버려도 되므로, "굴리는 비용"만 O(1)로 줄이면 전체가 O(N).
- 그러나 LSTM 학습용은 **윈도우 (20, 4) 배열 전체를 2072개 모두 보존**해야 한다.
  - 덱으로 한 칸씩 굴려도, 매 스텝 현재 덱 상태를 `np.array(deque)`로 **스냅샷 복사**해야 한다 → 스냅샷 한 번이 O(W), 총 O(N·W).
  - 즉 복사 총량이 원래 copy 방식과 동일 → 덱의 O(1) 갱신 이점이 상쇄된다.
- **핵심 갈림길**: "자료구조가 양방향이냐"가 아니라 **"윈도우 내용물을 버려도 되느냐(요약) vs 다 들고 있어야 하느냐(보존)"**.
  - 요약형 → deque가 정답.
  - 보존형(LSTM 입력) → deque든 리스트든 복사 총량 동일 → 이득 없음.

### 4. 보존형의 진짜 해법 = `sliding_window_view` (복사 없는 뷰)

- 뷰가 거의 공짜인 이유: **데이터를 복사하지 않는다.** 결과의 `base`가 원본을 가리키고(`base=True`), numpy는 "원본을 이런 stride(보폭)로 건너뛰며 읽으면 (2072, 20, 4)로 보인다"는 **메타데이터만** 만든다.
- 겹치는 20일치를 물리적으로 20번 복제하지 않고 "겹쳐 보기"만 한다.
  - 윈도우로 펼친 논리적 크기 1.26 MB vs 원본 0.06 MB → copy 방식은 원본의 약 **20배(=window 크기배)** 를 중복 저장하고 있었다. 뷰는 그 중복을 안 만든다.
- **주의점**:
  - **읽기 전용 가정**: 뷰의 한 칸을 수정하면 겹치는 다른 윈도우도 같이 바뀐다(같은 메모리). 학습 입력으로 *읽기만* 하면 안전. → 스케일링 같은 값 변경은 윈도우 만들기 *전에* 원본에 해야 한다(실습 코드도 그렇게 함).
  - **`model.fit` 단계에서 결국 한 번은 복사**: 텐서로 올릴 때 연속 메모리가 필요. 그래도 "루프 2072번 + pandas 변환"을 "C 레벨 뷰 한 방 + 마지막 한 번 복사"로 바꾸는 이득은 그대로.
- **권장 형태**:
  ```python
  from numpy.lib.stride_tricks import sliding_window_view
  X = sliding_window_view(arr, (W, F))[:n_win, 0]   # (n_win, W, F) 뷰
  y = label[W:W+n_win]
  # 불안하면 X = X.copy() 로 연속 배열 명시
  ```

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 |
|---|
| 실습 코드의 병목은 copy보다 **pandas `.iloc` 반복 변환**이었다 (순수 numpy로 60배↑) |
| deque 양방향 O(1)은 **요약형 슬라이딩**에서만 이득, **보존형**에선 스냅샷 복사로 상쇄 |
| `sliding_window_view`는 stride 메타데이터만 만들어 복사 없이 겹쳐 보기(읽기 전용) |

---

## 출처
- Claude와 직접 벤치마크 (2026-06-23): 동일 형태 데이터(2072×20×4)로 3개 방식 `time.perf_counter` + `tracemalloc` 측정.
- numpy 문서: `numpy.lib.stride_tricks.sliding_window_view`.

---
*작성일: 2026-06-23 | 인공지능 사관학교 AI 보안 과정*
