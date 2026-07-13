# 2026-07-13 실습 — RAG 리트리버 & Streamlit 챗봇

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-13_AI-Security-Course_Lecture_RAG-Intro.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-13
> 직접 해보면서 느낀 점. 개념으로 알던 것과 실제로 해본 것의 차이 등.

- 청킹 → 임베딩 → 벡터DB 저장 → 리트리버 → 답변 체인까지 전체 파이프라인을 처음부터 끝까지 다뤄봤다.
- "페이지 경계 수동 오버랩" 부분이 조금 인상깊었다. 청크가 페이지 경계에서 잘리면 문맥이 끊기는데, 다음 청크의 앞 100자를 현재 청크 뒤에 붙여서 손실을 보완하는 방식이었다. 현재는 기계적으로 잘랐지만, 좋은 RAG 시스템을 만들려면 이 또한 고려해야 하는 사항이라고 생각된다.

---

## 실습 개요
- **목표**: PDF 문서(뉴욕/서울 도시계획) 기반 RAG 챗봇을 노트북 프로토타입 → 모듈화 → Streamlit 앱 순으로 구현
- **방법**: LangChain + Chroma + OpenAI Embedding/Chat 모델

---

## 진행 과정

### Step 1. 노트북 프로토타입 (`ag_11_1_rag_practice.ipynb`)
- `PyMuPDFLoader`로 NYC 2050 전략계획, 서울 2040 계획 PDF 로드
- `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)`로 청킹
- 페이지 경계 수동 오버랩 처리 후 두 문서의 청크를 `extend`로 병합
- `OpenAIEmbeddings(text-embedding-3-large)`로 임베딩, Chroma에 저장(이미 저장돼 있으면 로드만)
- `as_retriever(k=3)`으로 유사 청크 3개 검색
- `create_stuff_documents_chain`으로 답변 체인 구성, `ChatMessageHistory`로 대화 이력 관리
- 질의증강 체인 추가: 이전 대화 맥락을 참고해 모호한 질문("뉴욕은?")을 명확한 한 문장으로 변환

### Step 2. 모듈화 (`ag_11_2_retriever.py`)
- 노트북의 벡터스토어 로드, 답변 체인, 질의증강 체인을 재사용 가능한 모듈로 분리
- 이후 `ag_11_3_rag.py`에서 `import ag_11_2_retriever as retriever`로 불러다 씀

### Step 3. Streamlit 챗봇 (`ag_11_3_rag.py`)
- `st.session_state`에 대화 이력 저장(SystemMessage로 "도시 정책 전문가" 페르소나 설정)
- 사용자 입력 시 `retriever.query_augmentation_chain`으로 질의증강 실행
- `get_ai_response`로 `llm.stream()` 결과를 스트리밍 출력

---

## 결과 / 산출물
- 동작하는 Streamlit RAG 챗봇 UI (질의증강 로직까지는 실행됨)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 청킹 → 임베딩 → 벡터DB → 리트리버 파이프라인을 노트북에서 끝까지 구성 성공 |
| `retriever.py` 모듈화로 노트북 로직을 재사용 가능한 형태로 정리 |

---
*작성일: 2026-07-13 | 인공지능 사관학교 AI 보안 과정*
