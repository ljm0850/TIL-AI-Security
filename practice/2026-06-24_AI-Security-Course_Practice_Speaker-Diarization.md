# 2026-06-24 실습 — 화자분리(Speaker Diarization) & STT 병합

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 강의 교재 없이 실습 위주. ag_07_4는 강사 코드 거의 그대로)
> **관련 코드**: ag_07_3_화자분리(pyannote), ag_07_4_stt.py(통합 모듈)
> **참고**: 회의록 파이프라인 2단계. 이전 → `STT-Whisper` / 다음 → `Meeting-Summary`

---

## 느낀 것 / 배운 것

### 📅 2026-06-24

- **"누가 말했나(화자분리)"와 "뭐라고 했나(STT)"는 따로 구한 뒤 시간으로 합치는 거였다.** pyannote는 오디오에서 "언제~언제는 SPEAKER_00" 식의 **시간 구간만** 내놓고(텍스트는 모름), Whisper는 "언제~언제 이런 말"을 내놓는다(화자는 모름). 둘을 **시간 겹침(overlap)** 으로 짝지어야 "누가 무슨 말을 했는지"가 완성된다. 한 모델이 다 하는 게 아니라, 역할이 다른 모델의 출력을 후처리로 엮는 구조라는 게 핵심.

- **RTTM을 다루며 "연속된 같은 화자 구간을 하나로 묶는" 전형적인 그룹화 패턴을 썼다.** pyannote 출력(RTTM)은 한 사람이 말해도 잘게 쪼개져 여러 줄로 나온다. "이전 행과 speaker_id가 다르면 그룹 번호 +1, 같으면 유지"로 **run-length 식 번호**를 매긴 뒤 `groupby('number')`로 묶어 발화 단위의 (start=min, end=max)를 구했다. 데이터분석에서 본 groupby + NamedAgg가 음성 후처리에 그대로 쓰임.

- **공급망 보안 신호 재확인** — 화자분리 모델 로딩 중에도 `torch.load(weights_only=False)`의 "악성 pickle → 임의 코드 실행" 경고가 떴다. pyannote/transformers처럼 외부 가중치를 받아 쓰는 모든 지점이 잠재적 공급망 위험. 신뢰할 수 있는 출처·safetensors 포맷을 기본값으로 삼아야 한다.


---

## 실습 개요
- **목표**: 오디오에서 화자 구간을 분리(diarization)하고, STT 텍스트를 화자에 할당해 "화자별 발화" 표를 만든다.
- **도구**: pyannote `speaker-diarization-3.1`(HuggingFace, Access Token 필요), pandas.

---

## 진행 과정

### Step 1. 화자분리 → RTTM (ag_07_3)
```python
from pyannote.audio import Pipeline
pipeline = Pipeline.from_pretrained('pyannote/speaker-diarization-3.1', use_auth_token=api_key)
dia = pipeline('data/ch05/싼기타_비싼기타.mp3')
with open('...rttm', 'w', encoding='utf-8') as f:
    dia.write_rttm(f)   # 화자/시간 구간을 RTTM 포맷으로 기록
```
- RTTM을 `pd.read_csv(sep=' ')`로 읽어 `start`, `duration`, `speaker_id` 등 확보 → `end = start + duration` 추가.

### Step 2. 연속 동일 화자 그룹화
```python
df_rttm['number'] = None
df_rttm.at[0, 'number'] = 0
for i in range(1, len(df_rttm)):
    now, before = df_rttm.at[i, 'speaker_id'], df_rttm.at[i-1, 'speaker_id']
    isSameSpeaker = now == before
    cnt = 0 if isSameSpeaker else 1          # 화자 바뀌면 +1
    df_rttm.at[i, 'number'] = df_rttm.at[i-1, 'number'] + cnt

df_rttm_grouped = df_rttm.groupby('number').agg(
    start=pd.NamedAgg('start', 'min'),
    end=pd.NamedAgg('end', 'max'),
    speaker_id=pd.NamedAgg('speaker_id', 'first'))
```
> **본인 변형 vs 강사 코드**: 강사 원본은 같은 로직을 `if/else`로 직접 증감. 본인은 `isSameSpeaker`/`cnt`로 의도를 드러나게 정리(가독성 목적). 결과는 동일.

### Step 3. STT 텍스트 ↔ 화자 구간 병합 (ag_07_4, 강사 코드)
```python
# 각 STT 구간에 대해, 시간이 가장 많이 겹치는 화자 구간을 찾아 텍스트를 붙인다.
for i_stt, row_stt in df_stt.iterrows():
    overlap_dict = {}
    for i_rttm, row_rttm in df_rttm.iterrows():
        overlap = max(0, min(row_stt['end'], row_rttm['end']) - max(row_stt['start'], row_rttm['start']))
        overlap_dict[i_rttm] = overlap
    max_overlap = max(overlap_dict.values())
    max_overlap_idx = max(overlap_dict, key=overlap_dict.get)   # 최대 겹침 화자 구간의 인덱스
    if max_overlap > 0:
        df_rttm.at[max_overlap_idx, 'text'] += row_stt['text'] + '\n'
```
- 핵심은 **구간 교집합** `max(0, min(끝) - max(시작))` — 두 시간 구간이 겹치는 길이. 가장 많이 겹치는 화자에게 그 발화를 할당.
- `ag_07_4_stt.py`는 위 STT·화자분리·병합을 `whisper_stt` / `speaker_diarization` / `stt_to_rttm` 함수로 묶은 통합 모듈(강사 코드).

---

## 결과 / 산출물
- 화자별로 묶인 발화 표 (start, end, speaker_id, duration) → csv.
- STT 텍스트가 화자 구간에 할당된 최종 표 (start, end, speaker_id, text) → `..._final.csv` (다음 단계 입력).

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| "화자(구간) + 발화(텍스트)"를 시간 overlap으로 병합하는 구조 이해 | ag_07_4 병합은 강사 코드 — 직접 작성/변형은 적음 |
| run-length 그룹화 + groupby로 발화 단위 집계 | 화자가 겹쳐 말하는(overlap speech) 구간 처리는 미고려 |
| 토큰 출력 노출이라는 실제 시크릿 유출 사고를 인지 | 병합 정확도(잘못 할당된 발화)에 대한 평가는 안 함 |

---

## 다음에 이어서 할 것
- 노출된 HF 토큰 폐기(rotate) + 노트북 출력 셀 정리 후 커밋.
- 화자별 발화 표를 회의록으로 요약 → `Meeting-Summary` 단계.

## 다음에 파고들 만한 질문
> A유형만.

- STT 구간과 화자 구간이 1:1로 안 맞고 여러 화자에 걸칠 때(overlap이 비슷하게 분산될 때), "최대 겹침 하나에 통째 할당"이 최선일까? 발화를 시간 비율로 쪼개 나눠 붙이는 방식과 비교하면 어떨까?

---
*작성일: 2026-06-24 | 인공지능 사관학교 AI 보안 과정*
