# 2026-06-24 실습 — STT (음성→텍스트): OpenAI Whisper API vs HuggingFace 로컬

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 강의 교재 없이 실습 위주)
> **관련 코드**: ag_07_1_whisper(OpenAI API), ag_07_2_허깅페이스(로컬 모델)
> **참고**: 회의록 자동 생성 파이프라인의 1단계. 다음 단계 → `Speaker-Diarization`, `Meeting-Summary`

---

## 느낀 것 / 배운 것

### 📅 2026-06-24

- **같은 Whisper인데 "API로 부르기"와 "로컬에 받아 돌리기"는 완전히 다른 경험이었다.** OpenAI API(ag_07_1)는 `client.audio.transcriptions.create(model='whisper-1', file=f)` 으로 끝 — 코드가 5줄. 반면 HuggingFace 로컬(ag_07_2)은 transformers `pipeline` 구성, torch device/dtype 설정, ffmpeg 경로 등록까지 셋업이 길다. "쉽게 쓰는 대신 데이터를 넘긴다 vs 번거로운 대신 내 손에 둔다"의 트레이드오프를 코드로 체감.

- **AI 보안 관점 — STT를 어디서 돌리느냐가 곧 데이터가 어디로 가느냐다.** OpenAI API는 오디오 파일이 **외부 서버로 전송**된다. 회의 녹음·상담 내용처럼 민감한 음성이면, 그 자체가 외부 유출 경로가 된다. HF 로컬은 모델을 받아 **데이터가 내 PC를 벗어나지 않는다(on-prem)** — 프라이버시가 중요한 도메인일수록 로컬 STT가 유리하다는 걸 "편의 vs 통제"의 실제 선택지로 이해.

- **긴 오디오를 "겹치는 청크"로 자르는 게 어제 롤링 윈도우와 같은 발상이었다.** HF 파이프라인에 `chunk_length_s=10, stride_length_s=2`를 줬는데, 10초 단위로 자르되 **2초씩 겹치게** 처리한다. 경계에서 단어가 잘려 인식이 깨지는 걸 막으려는 overlap — 어제 deep-dive에서 본 "윈도우를 겹쳐 미끄러뜨리는" 아이디어가 오디오 도메인에서 그대로 재등장. 도메인이 달라도 "겹치는 윈도우"는 같은 도구.


---

## 실습 개요
- **목표**: 음성 파일(mp3)을 텍스트로 변환(STT)하고, 이후 화자분리·요약에 쓸 수 있도록 (start, end, text) 형태로 정리.
- **두 방식**: (A) OpenAI Whisper API, (B) HuggingFace 로컬 whisper-large-v3-turbo.

---

## 진행 과정

### A. OpenAI Whisper API (ag_07_1)
```python
from openai import OpenAI
client = OpenAI(api_key=api_key)

audio_file = 'data/ch05/lsy_audio_2023_58s.mp3'
with open(audio_file, 'rb') as f:
    transcription = client.audio.transcriptions.create(
        model='whisper-1',
        file=f
    )
# transcription.text 에 전체 텍스트
```
- 매우 단순. 기본 호출은 전체 텍스트만 반환(타임스탬프 등은 옵션 필요).

### B. HuggingFace 로컬 (ag_07_2)
```python
import os, torch
from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor, pipeline

# ffmpeg 실행 경로 등록 (오디오 디코딩에 필요)
os.environ['PATH'] += os.pathsep + os.getcwd() + r'\ffmpeg_711\bin'

device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32
model_id = "openai/whisper-large-v3-turbo"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True)
model.to(device)
processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model, tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    torch_dtype=torch_dtype, device=device,
    return_timestamps=True,   # 청크별 시간 기록 → 화자분리에 활용
    chunk_length_s=10,        # 10초 단위 청크
    stride_length_s=2,        # 청크끼리 2초 겹침
)
result = pipe(sample, return_timestamps=True)
```
- `return_timestamps=True`로 청크별 (start, end, text)를 얻어, 다음 단계(화자분리)와 시간으로 정렬할 수 있게 함.
- 결과를 `pickle`로 백업하고, `chunks`를 순회해 DataFrame(start/end/text) → csv(`sep='|'`) 저장.

---

## 결과 / 산출물
- 58초 오디오 → 청크별 (start, end, text) DataFrame → `output/ch05/..._58s.csv`.
- 두 방식 모두 동일 오디오를 한국어로 정상 전사. (로컬은 타임스탬프까지 확보.)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| API/로컬 두 방식의 셋업·결과 차이를 직접 비교 | 로컬은 CPU라(device=cpu) 속도 한계 — GPU 환경 못 써봄 |
| `return_timestamps`로 다음 단계용 (start,end,text) 확보 | 청크 경계에서 문장이 잘리는 케이스 정밀 검증은 안 함 |
| API vs 로컬을 "데이터가 어디로 가나"의 보안 관점으로 해석 |  |

---

## 다음에 이어서 할 것
- 화자분리(pyannote) 결과와 STT 타임스탬프를 시간으로 정렬·병합 → `Speaker-Diarization` 단계.

---
*작성일: 2026-06-24 | 인공지능 사관학교 AI 보안 과정*
