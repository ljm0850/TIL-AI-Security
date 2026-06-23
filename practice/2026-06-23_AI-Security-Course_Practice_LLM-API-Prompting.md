# 2026-06-23 실습 — LLM API 활용 & 프롬프트 엔지니어링

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 강의 교재 없이 실습 위주)
> **관련 코드**: ag_01_gpt_basic, ag_02_역할부여, ag_03_맥락학습, ag_04_싱글턴대화, ag_05_멀티턴대화

---

## 느낀 것 / 배운 것

### 📅 2026-06-23

- **오늘 오후는 "모델을 만든다"에서 "거대 모델을 부린다"로의 전환이었다.** 오전까지는 직접 신경망을 설계·학습했는데, 오후엔 이미 학습된 LLM(gpt-4o-mini)을 API로 호출해 *프롬프트로* 행동을 유도했다. 학습이 아니라 **지시(프롬프트)** 로 결과를 바꾼다는 게 핵심 — 5/14 AI 엔지니어링 기초에서 본 "모델을 도구로 쓰는" 감각이 코드로 구체화됐다.

- **`messages`의 role 3종이 대화의 문법이다.** `system`(대화 전체 지침/페르소나), `user`(사용자 입력), `assistant`(모델 답변). 역할 부여(ag_02)는 결국 system에 "너는 조커야" 같은 지침을 박는 것이고, few-shot(ag_03)은 user/assistant 쌍으로 *예시 대화*를 미리 깔아 패턴을 보여주는 것. 즉 프롬프트 엔지니어링 = 이 messages 배열을 어떻게 구성하느냐의 문제.

- **맥락학습(In-Context Learning)의 zero/one/few-shot을 직접 비교했다.** 예시 0개(zero) → 1개(one) → 여러 개(few)로 갈수록, 모델이 원하는 출력 형식("유치원생처럼 동물 울음소리")을 더 잘 따라간다. 가중치를 안 바꾸고 *예시만으로* 행동을 잡는다는 게 신기 — 어제까지의 "학습으로 가중치 조정"과 완전히 다른 길.

- **싱글턴 vs 멀티턴의 차이는 "대화 이력을 들고 가느냐"였다.** ag_04(싱글턴)는 매 호출마다 system+현재 입력만 보내서 모델이 직전 대화를 기억 못 한다. ag_05(멀티턴)는 `messages` 리스트에 user/assistant를 **계속 append**해서 통째로 다시 보낸다 → 맥락 유지. LLM은 상태가 없고(stateless), "기억"은 매번 전체 이력을 다시 보내서 흉내 내는 것임을 코드로 이해했다.

- **AI 보안 관점 — 멀티턴 이력 관리가 곧 데이터 보관 문제다.** ag_05는 대화 전체를 메모리에 쌓아 매번 외부 API로 전송한다. 실제 서비스라면 (1) 민감정보가 이력에 누적돼 매 턴 외부로 나가고, (2) 이력이 길어질수록 비용·컨텍스트 한계 문제가 생긴다. "맥락 유지"의 편의 뒤에 데이터 노출·보관이라는 보안 비용이 붙는다는 걸 기록.

---

## 실습 개요
- **목표**: OpenAI API로 LLM을 호출하고, 프롬프트 엔지니어링(역할 부여·맥락학습)과 대화 관리(싱글/멀티턴)를 직접 구현.
- **공통 셋업**: `.env`에서 `OPENAI_API_KEY` 로딩(dotenv) → `OpenAI(api_key=...)` 클라이언트 → `chat.completions.create(model='gpt-4o-mini', temperature, messages=[...])`.

---

## 진행 과정

### Step 1. 기본 호출 (ag_01)
- `messages`에 system(지침)+user(질문)를 넣어 호출, `response.choices[0].message.content`로 답 추출.
- **temperature**: 0에 가까울수록 같은 답(결정적), 1에 가까울수록 창의적/다양. (재현이 필요하면 낮게.)

### Step 2. 역할 부여 (ag_02)
- system에 "너는 다크나이트의 조커야" 같은 페르소나를 지정 → 답변 톤·내용이 캐릭터에 맞게 변함. temperature=0.9로 창의성↑.

### Step 3. 맥락학습 zero/one/few-shot (ag_03)
- **zero-shot**: 예시 없음 (system 지침만).
- **one-shot**: 예시 1쌍 제공.
- **few-shot**: 예시 여러 쌍 제공 — user(질문)/assistant(모범답) 쌍을 깔고 마지막에 진짜 질문(user).
```python
messages=[
    {'role':'system', 'content':'너는 유치원 학생이야. 유치원생처럼 답변해줘'},
    {'role':'user', 'content':'참새'}, {'role':'assistant', 'content':'짹짹'},   # 예시 쌍
    {'role':'user', 'content':'오리'}, {'role':'assistant', 'content':'꽥꽥'},   # 예시 쌍
    {'role':'user', 'content':'개구리'}, {'role':'assistant', 'content':'개굴개굴'},
    {'role':'user', 'content':'뱀'},   # ← 실제 질문
]
```
> ⚠️ **메모 정정**: 처음 메모엔 예시를 `user:'오리'` / `user:'꽥꽥'`처럼 둘 다 user로 적었는데, 이러면 few-shot이 안 된다. **모범답은 `assistant` 역할**로 줘야 모델이 "이 입력엔 이렇게 답한다"는 패턴을 학습한다. 마지막 한 줄만 진짜 질문(user)으로 비워둠.

### Step 4. 싱글턴 대화 (ag_04)
- `while` 루프로 입력받아 매번 호출하지만 **이력 없음** → 직전 대화를 기억 못 함.

### Step 5. 멀티턴 대화 (ag_05)
```python
messages = [{'role':'system', 'content':'...'}]
while True:
    user_input = input("사용자 : ")
    if user_input == 'exit': break
    messages.append({'role':'user', 'content':user_input})
    ai = get_ai_response(messages)                    # 전체 이력을 통째로 전송
    messages.append({'role':'assistant', 'content':ai})  # 답도 이력에 누적
```
- 매 턴 `messages`에 user/assistant를 append → 다음 호출에 전체 맥락 전달 → 대화 기억 유지.

---

## 결과 / 산출물
- 역할 부여로 페르소나 답변, few-shot로 출력 형식 유도, 멀티턴으로 맥락 유지 대화 — 4가지 프롬프트 패턴을 동작 확인.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| role(system/user/assistant) 구조와 프롬프트 패턴을 코드로 체득 | 토큰/비용·컨텍스트 길이 한계는 다뤄보지 않음 |
| LLM이 stateless이고 "기억"은 이력 재전송임을 이해 | 멀티턴 이력이 길어질 때의 관리(요약/잘라내기) 미적용 |
| zero/one/few-shot 차이를 직접 비교 | 출력 형식 강제(JSON 등 구조화) 까진 안 감 |

---
*작성일: 2026-06-23 | 인공지능 사관학교 AI 보안 과정*
