# 2026-06-25 실습 — Function Calling (함수 콜링) & 에이전트 구조

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 강의가 실습에 녹아있어 실습 위주)
> **관련 코드**: ag_09_1_gpt_fn(함수+tools 정의), ag_09_2_fn_calling(본인 코드), ag_09_2_fn_calling2(강사 배포 코드)

---

## 느낀 것 / 배운 것

### 📅 2026-06-25

- **펑션 콜링의 정체를 직접 뜯어 이해했다 — LLM은 함수를 실행하지 않는다.** 처음엔 "함수 코드 전체를 넘기나, 호출 결과를 넘기나?" 헷갈렸는데, 동작을 보니 **LLM은 함수를 직접 실행하지 않고, 받은 *함수 설명(tools)* 만 보고 "이 함수를 이런 인자로 쓰면 되겠다"를 판단**해서 알려줄 뿐이다. 실제 실행은 내가 로컬에서 하고, 그 **결과를 다시 LLM에 전달**해 최종 답을 만든다. 즉 LLM은 "어떤 도구를 어떻게 쓸지 정하는 두뇌"고, 손발은 내 코드다.

- **전체 구조가 한 줄로 잡혔다.** `사용자 질문 → tools 정보 포함해 요청 → LLM이 필요한 tool과 인자 반환 → 로컬에서 tool 실행 → 결과를 포함해 다시 요청 → 최종 답변`. 한 번의 호출이 아니라 **"LLM ↔ 내 코드"가 핑퐁**하는 구조라는 게 핵심.

- **MCP가 떠올랐고, 실제로 맞는 연상이었다.** "함수 설명을 풀컨텍스트로 주고, 요청에 따라 도구를 안내한다"는 이 패턴이 MCP가 표준화하려는 바로 그것이다. 펑션 콜링이 *모델 한 개에 도구를 붙이는 방법*이라면, MCP는 *그 도구 연결을 프로토콜로 규격화*한 것 — 오늘 배운 게 MCP를 이해하는 토대가 됐다.

- **에이전트로 가려면 "한 번"이 아니라 "루프"가 필요하다는 걸 스스로 도출했다.** 도구가 순서대로 필요하거나(A 결과로 B를 호출) 여러 단계를 거쳐야 하면, **"더 호출할 tool이 없어질 때까지 반복"** 하는 루프가 있어야 진짜 에이전트가 된다. 오늘 코드는 1~2회 핑퐁이지만, 이 루프를 일반화하면 자율 에이전트의 뼈대다. (에이전트 = 펑션 콜링 + 종료 조건까지의 반복.)

- **AI 보안(오늘의 핵심) — 펑션 콜링은 프롬프트 인젝션의 반경을 "말"에서 "행동"으로 넓힌다.** 지금까지 과정의 인젝션은 기껏해야 "엉뚱한 텍스트 출력"이었다. 그런데 펑션 콜링에선 **LLM이 사용자 입력에 따라 어떤 도구를 어떤 인자로 부를지 결정**한다. 악성 입력이 끼면 의도치 않은 도구 호출·인자로 이어져 실제 부작용(파일 삭제, 외부 요청, 송금 등 도구에 따라)이 날 수 있다. 더구나 tool 인자는 모델 출력(`json.loads(tool_calls[0].function.arguments)`)이라, **실행 전에 인자를 검증·화이트리스트**하지 않으면 모델 말을 그대로 실행하는 셈. "LLM에 손을 쥐여주는 순간 인젝션은 곧 행동이 된다" — 에이전트 보안의 핵심 축.

---

## 실습 개요
- **목표**: LLM이 외부 도구(현재시각, 주식정보)를 호출하도록 tools를 정의하고, 멀티턴 대화에서 도구 호출→결과 반영→최종 답변까지 구현.
- **도구**: OpenAI API(gpt-4o-mini), pytz, yfinance.

---

## 진행 과정

### Step 1. 함수 + tools 스키마 정의 (ag_09_1)
```python
def get_current_time(timezone:str='Asia/Seoul'): ...      # 실제 실행될 로컬 함수
def get_yf_stock_info(ticker:str)->str: ...               # 야후 주식정보 (작성만, 아직 미연결)

tools = [{
    'type':'function',
    'function':{
        'name':'get_current_time',
        'description':'현재 날짜와 시간을 반환합니다.',
        'parameters':{
            'type':'object',
            'properties':{'timezone':{'type':'string','description':'예: Asia/Seoul'}},
            'required':['timezone'],
        }
    }
}]
```
- 핵심: `description`과 `parameters`가 **LLM이 "언제/어떻게 이 함수를 쓸지" 판단하는 유일한 단서**. 함수 본문은 LLM에 안 간다.
- `get_yf_stock_info`(yfinance)는 함수만 작성하고 tools 연결·사용 전에 수업 종료 → 다음에 이어서.

### Step 2. 펑션 콜링 — 본인 코드 (ag_09_2_fn_calling)
```python
ai_message = get_ai_response(messages, tools=tools).choices[0].message
tool_calls = ai_message.tool_calls          # LLM이 "이 도구 쓰자"고 반환
if tool_calls:
    tool_name = tool_calls[0].function.name
    tool_call_id = tool_calls[0].id
    arguments = json.loads(tool_calls[0].function.arguments)
    if tool_name == 'get_current_time':
        messages.append({'role':'tool','tool_call_id':tool_call_id,
                         'name':tool_name,'content':get_current_time(**arguments)})
    ai_message = get_ai_response(messages, tools=tools).choices[0].message  # 결과 반영해 재요청
```
- 본인 주석: "tool별로 if 말고 **반복문 + match-case로 dispatch하면 가독성 좋으려나?**" → 도구가 늘 때의 확장성 고민(좋은 방향).
- 진행 중 에러가 많이 나서, 강사가 자기 코드를 배포 → 이후는 그 코드 기준으로 진행(ag_09_2_fn_calling2).

### Step 3. 펑션 콜링 — 강사 배포 코드 (ag_09_2_fn_calling2)
```python
if tool_calls:
    messages.append(ai_message.model_dump())     # ① 메시지를 dict로 직렬화해 추가
    for tool_call in tool_calls:                 # ② tool_calls 전체를 순회(여러 도구 대응)
        ...
        messages.append({'role':'tool', 'tool_call_id':..., 'name':..., 'content':...})
    messages.append({'role':'system','content':'이제 주어진 결과를 바탕으로 답변할 차례다.'})  # ③ 최종답변 유도
    ai_message = get_ai_response(messages, tools=tools).choices[0].message
messages.append(ai_message.model_dump())
```
> **본인 코드 vs 강사 코드 차이**: ① 강사는 메시지를 `model_dump()`로 **dict 직렬화**해 이력에 넣음(tool_calls 포함 메시지를 안전하게 재전송). ② `tool_calls[0]`만이 아니라 **전체를 순회**(한 턴에 여러 도구 호출 대응). ③ 도구 결과 뒤 **system 한 줄로 최종 답변을 유도**. 본인 버전은 단일 도구·`ai_message` 직접 append라 메시지 중복/직렬화 이슈가 있었음 → 강사 버전이 멀티 도구·직렬화에 더 견고.

---

## 결과 / 산출물
- "지금 몇 시야?" 류 질문에 LLM이 `get_current_time` 호출을 결정 → 로컬 실행 결과를 받아 자연어로 답하는 멀티턴 대화 동작 확인.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 펑션 콜링 동작 구조(LLM=판단/로컬=실행/결과 재전달)를 스스로 도출 | tool 인자 검증/화이트리스트 등 보안 처리 미적용 |
| MCP·에이전트 루프와의 연결을 인사이트로 포착 | yfinance 주식 도구는 연결·사용 전 종료 |
| 도구 dispatch 확장성(match-case)까지 고민 |  |

---

## 다음에 이어서 할 것
- `get_yf_stock_info`를 tools에 등록해 "AAPL 주가 알려줘" → 도구 호출까지 연결.

## 다음에 파고들 만한 질문
> A유형만.

- 도구가 순서 의존적일 때(A 결과로 B 호출) "필요한 tool이 없어질 때까지 루프"가 표준 에이전트 패턴인 것 같은데, 이 루프가 무한히 돌거나 잘못된 도구를 반복 호출하는 걸 막는 종료/안전장치는 보통 어떻게 설계할까?

---
*작성일: 2026-06-25 | 인공지능 사관학교 AI 보안 과정*
