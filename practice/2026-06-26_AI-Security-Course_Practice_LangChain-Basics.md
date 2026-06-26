# 2026-06-26 실습 — LangChain 기초 (모델·히스토리·LCEL·Tools)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 강의가 실습에 녹아있어 실습 위주)
> **관련 코드**: ag_10_1_langchain, ag_10_2_멀티턴대화, ag_10_3_히스토리관리, ag_10_4_LCEL, ag_10_5_Tools
> **참고**: 에이전트 트랙은 강사 일정상 오늘까지 진행 후 **2주 뒤 재개** 예정.

---

## 느낀 것 / 배운 것

### 📅 2026-06-26

- **LangChain은 "어제 내가 손으로 짠 것"을 그대로 부품화한 프레임워크였다.** 어제(6/25) 펑션 콜링에서 `messages.append`로 직접 관리하던 대화 이력은 오늘 `RunnableWithMessageHistory`(세션별 자동 관리)로, 직접 분기하던 `tool_calls` 처리는 `bind_tools` + `@tool`로 추상화됐다. 어제 *바닥부터 해봤기에* LangChain이 마법이 아니라 "그 반복 작업의 포장지"로 보인다 — 프레임워크를 쓰되 안에서 무슨 일이 일어나는지 아는 상태로 쓰게 된 게 가장 큰 수확.

- **LCEL의 `|`는 Unix 파이프 / 함수 합성이다.** `chain = prompt_template | model | parser`처럼, **앞 단계의 출력이 뒤 단계의 입력으로 자동 전달**된다. 내가 "yield 느낌"이라 메모했는데 절반 맞다 — 연결 자체는 파이프(합성)이고, `.stream()`으로 토큰을 흘려보낼 때 그 yield 같은 지연 흐름이 드러난다. 핵심 이득: 글루 코드 없이 "프롬프트→모델→파싱"을 선언적으로 엮어 **프롬프팅에만 집중**할 수 있다.

- **구조화 출력(structured output)은 "틀은 강제, 내용은 모델 몫"이었다.** 내 질문이 "답변이 어느 정도나 강제되나?"였는데, `with_structured_output(Adlib)`을 써보니 **형식·타입은 확실히 강제**된다 — `answer:str`, `main_emotion`은 Literal 6종(기쁨/분노/…) 중 하나, `intensity:float`. 모델이 이 스키마를 벗어난 형태로는 못 답한다. 다만 **어떤 감정을 고르는지(의미적 정답)는 보장 안 됨**. 즉 "그릇은 고정, 담기는 내용은 모델 판단". 6/23 PDF 요약에서 포맷을 프롬프트로 부탁하던 걸, 이제 타입으로 못 박는 정식 방법.

- **`@tool`이 docstring을 "도구 설명서"로 읽는다 — 내 'googlestyle docstring ???' 의문 해결.** `@tool` 데코레이터는 함수의 **docstring(Google 스타일 `Args:`)을 파싱해 LLM에게 줄 description/parameters로 사용**한다. 즉 docstring이 "사람용 주석"이 아니라 **LLM이 도구를 언제·어떻게 쓸지 판단하는 실제 스펙**이 된다. 어제 손으로 짜던 tools 스키마(name/description/parameters)를 docstring 한 곳으로 모은 셈. docstring을 잘 쓰는 게 곧 도구 품질이 된다.

- **AI 보안 — Pydantic 스키마/구조화 출력은 어제 말한 "가드레일"의 실체다.** 어제 "tool 인자를 실행 전 검증해야 한다"고 적었는데, 오늘 `StockHistoryInput(BaseModel)`로 tool 입력을 **타입 강제**하고, `with_structured_output`로 출력을 **스키마 강제**하는 게 정확히 그 방어다. 모델 출력을 자유 텍스트로 두지 않고 타입·열거값으로 좁히면 인젝션·형식 공격 표면이 준다. 반면 세션 히스토리(`store[session_id]`)는 **세션별 격리**가 되지만 in-memory·서버 보관이라, session_id가 추측 가능하면 교차 사용자 유출 위험이 있다 — 멀티유저 서비스의 프라이버시 포인트.

---

## 실습 개요
- **목표**: LangChain으로 모델 호출·메시지 타입·멀티턴·세션 히스토리·LCEL 체인·구조화 출력·도구 바인딩을 한 바퀴 익히기.
- **도구**: langchain-openai(ChatOpenAI), langchain-core(messages/prompts/output_parsers/tools/history), pydantic.

---

## 진행 과정

### Step 1. 모델 호출 + 메시지 타입 (ag_10_1)
```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage  # SystemMessage/AIMessage/ToolMessage
model = ChatOpenAI(model='gpt-4o-mini', api_key=api_key)
model.invoke([HumanMessage('안녕? 나는 홍길동이야.')])   # → AIMessage
```
- 메시지 타입 매핑: SystemMessage=system(지침), HumanMessage=user, AIMessage=assistant, ToolMessage=tool(도구 결과). 어제 dict role과 1:1 대응.

### Step 2. 멀티턴 (ag_10_2)
- `messages` 리스트에 `HumanMessage`/`AIMessage`를 append하며 맥락 유지 — 어제 OpenAI SDK로 한 멀티턴의 LangChain 버전.

### Step 3. 세션별 히스토리 관리 (ag_10_3)
```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

store = {}
def get_session_history(session_id):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

with_message_history = RunnableWithMessageHistory(model, get_session_history)
config = {'configurable': {'session_id': 'abc1'}}
with_message_history.invoke([HumanMessage('안녕 난 홍길동이야')], config=config)
# 같은 session_id면 이름 기억, 다른 session_id(abc2)는 기억 못 함 → 세션 격리
```
- 이력 관리를 프레임워크에 위임. `.stream()`으로 토큰 단위 스트리밍 출력도 확인.

### Step 4. LCEL — 파이프 체인 (ag_10_4)
```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

parser = StrOutputParser()
chain = model | parser                    # 출력에서 .content만 깔끔히 추출

# 프롬프트 템플릿: 변수화 → 재사용
prompt = ChatPromptTemplate([('system','너는 {story}의 {character_a} 역할이다...'),
                             ('user','안녕? 저는 {character_b}입니다. {activity} 같이 할까요?')])
chain = prompt | model | parser
chain.invoke({'story':'미녀와 야수','character_a':'미녀','character_b':'야수','activity':'저녁'})
# 같은 체인에 '흥부와 놀부' 등 다른 값만 넣어 재사용
```

### Step 5. 구조화 출력 (ag_10_4)
```python
from typing import Literal
from pydantic import BaseModel, Field

class Adlib(BaseModel):
    """스토리 설정과 사용자 입력에 반응하는 대사를 만드는 클래스"""
    answer: str = Field(description='생성된 대사')
    main_emotion: Literal['기쁨','분노','슬픔','공포','불쾌','중립'] = Field(description='주요 감정')
    main_emotion_intensity: float = Field(description='감정의 정도')

structured_model = model.with_structured_output(Adlib)
st_chain = prompt | structured_model
st_chain.invoke({...})   # → Adlib(answer=..., main_emotion='기쁨', main_emotion_intensity=8.0)
```
- 출력이 자유 텍스트가 아니라 **타입이 박힌 객체**로 나온다 → 후처리·검증 쉬움.

### Step 6. Tools — @tool 데코레이터 (ag_10_5)
```python
from langchain_core.tools import tool

@tool
def get_current_time(timezone: str, location: str) -> str:
    """현재 시간을 반환하는 함수
    Args:
        timezone (str): 타임존 (예: Asia/Seoul). 실제 존재하는 타임존이어야 함.
        location (str): 지역명. 타임존이 모든 지명에 대응되지 않아 이후 답변 생성에 사용됨.
    """
    ...   # docstring이 곧 LLM에게 줄 도구 설명서

tools = [get_current_time]
tool_dict = {'get_current_time': get_current_time}      # 이름→함수 매핑 디스패처
model_with_tools = model.bind_tools(tools=tools)

response = model_with_tools.invoke(messages)            # response.tool_calls 에 호출 지시
for tool_call in response.tool_calls:
    selected_tool = tool_dict[tool_call['name']]
    tool_msg = selected_tool.invoke(tool_call)          # 실행 → ToolMessage 반환
    messages.append(tool_msg)
model_with_tools.invoke(messages)                        # 도구 결과로 최종 답변
```
- 어제 `if/elif` 디스패치가 오늘 `tool_dict[name]` 매핑으로 깔끔해짐(어제 떠올린 그 방식).
- **Pydantic 입력 스키마**로 도구 인자도 타입화: `get_yf_stock_history(stock_history_input: StockHistoryInput)` → "테슬라 한 달 전 대비?" 질문에 ticker=TSLA, period=1mo로 호출 → 답변 생성.

---

## 결과 / 산출물
- LangChain으로 멀티턴·세션 히스토리·LCEL 체인·구조화 출력·도구 호출까지 한 바퀴 동작 확인.
- "광주는 지금 몇시야?" → 도구 호출 → 시각 반영 답변 / "테슬라 한 달 전 대비?" → 주가기록 도구 호출 → 비교 답변.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 어제 수동 구현(이력·도구)과 LangChain 추상화의 1:1 대응을 이해 | 에이전트(자동 루프)까지는 못 감 — 2주 후 재개 |
| LCEL 파이프·프롬프트템플릿·구조화 출력을 직접 엮음 | 체인 중간 단계 디버깅/관찰(LangSmith 등)은 안 봄 |
| `@tool` docstring=도구 스펙, Pydantic=가드레일을 연결 | 구조화 출력이 실패(스키마 위반)할 때의 처리 미확인 |

---

## 다음에 이어서 할 것
- (2주 후 에이전트 재개 시) bind_tools + 루프로 "도구 호출이 끝날 때까지 반복"하는 자동 에이전트 구성.
- 구조화 출력이 스키마를 못 맞출 때의 재시도/검증 처리 확인.

## 다음에 파고들 만한 질문
> A유형만.

- `with_structured_output`은 내부적으로 함수 호출(tool) 메커니즘을 쓴다고 들었는데, 그렇다면 한 번의 호출에서 **구조화 출력과 일반 도구 호출을 동시에** 쓸 수 있을까, 아니면 둘 중 하나만 되는 구조일까?

---
*작성일: 2026-06-26 | 인공지능 사관학교 AI 보안 과정*
