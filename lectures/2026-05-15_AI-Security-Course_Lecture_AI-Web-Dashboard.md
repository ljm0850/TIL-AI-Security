# 2026-05-15 강의 정리 — AI 웹 대시보드 (Streamlit)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 데이터 시각화와 AI 웹 대시보드 — 파이썬 기반 AI 웹 대시보드 제작 (Streamlit 활용)
> **관련 실습 파일**: `../practice/2026-05-15_AI보안과정_실습_AI웹대시보드.md` (있을 경우)

---

## 1. Streamlit이란?

### 핵심 개념
- Python만으로 대화형 웹 앱과 데이터 대시보드를 만들 수 있는 오픈소스 라이브러리
- Flask/Django처럼 HTML·CSS·JS 지식 없이, `.py` 파일 하나로 프런트엔드 + 백엔드 동시 완성

### 상세 내용
- **동작 원리 (Rerun 방식)**: 사용자가 버튼 클릭, 슬라이더 조절 등 위젯을 조작할 때마다 스크립트 전체를 위에서 아래로 재실행한다. 이벤트 리스너 없이도 실시간 반응형 인터페이스 구현 가능 — 단, 이 Rerun 방식이 단점이기도 함 (아래 인사이트 참고)
- **실행**: `streamlit run app.py` → `localhost:8501` 자동 오픈
- **핫 리로드**: 코드 저장 시 브라우저에 즉시 반영
- **주요 활용 사례**: AI 챗봇, 데이터 분석 대시보드, ML 모델 데모, MVP 프로토타입

---

## 2. 핵심 UI 요소

### 핵심 개념
- 텍스트 출력, 입력 위젯, 알림 메시지, 로딩 표시, 파일 업로드를 몇 줄의 Python으로 구현

### 상세 내용

**텍스트 출력**
- `st.title()` / `st.header()` / `st.subheader()` / `st.write()` / `st.markdown()`
- `st.write()`는 텍스트·숫자·DataFrame·차트·딕셔너리 모두 자동 감지 출력

**입력 위젯**
- `st.text_input()` : 한 줄 텍스트 입력
- `st.text_area()` : 여러 줄 텍스트 입력
- `st.selectbox()` : 드롭다운
- `st.slider()` : 슬라이더
- `st.checkbox()` : 체크박스

**알림 메시지**
- `st.success()` (초록) / `st.error()` (빨강) / `st.warning()` (노랑) / `st.info()` (파랑)

**기타**
- `with st.spinner('처리 중...')` : API 호출 중 로딩 표시
- `st.file_uploader()` : 파일 업로드 처리

---

## 3. 레이아웃 시스템

### 핵심 개념
- `columns`, `sidebar`, `tabs`, `expander`로 대시보드 레이아웃 구성
- `st.set_page_config(layout='wide')` : 스크립트 최상단에 배치, 전체 화면 너비 사용

### 상세 내용

```python
# 균등 3분할 / 3:1 비율 분할
col1, col2, col3 = st.columns(3)
col1, col2 = st.columns([3, 1])

# 사이드바 (필터·설정 영역)
with st.sidebar:
    model = st.selectbox("모델", ["gpt-4o-mini", "gpt-4o"])

# 탭 UI
tab1, tab2 = st.tabs(["데이터 분석", "AI 챗봇"])

# 접고 펼치는 영역
with st.expander("설정 가이드", expanded=True):
    st.info("페이지 로드 시 자동 펼쳐짐")
```

---

## 4. 데이터 시각화

### 핵심 개념
- Streamlit 내장 차트(간단·빠름) vs Plotly·Matplotlib 연동(인터랙티브·정밀)

### 상세 내용

**Streamlit 내장 차트** (pandas DataFrame과 연동)

- `st.line_chart(df)` : 선 그래프 (시계열)
- `st.bar_chart(df)` : 막대 그래프
- `st.area_chart(df)` : 영역 차트
- `st.map(df)` : 위도/경도 기반 지도

**Plotly Express 연동** (확대/축소, 호버, 범례 토글 가능)
```python
import plotly.express as px
fig = px.bar(df, x='날짜', y='매출', color='카테고리')
st.plotly_chart(fig, use_container_width=True)
```
- `use_container_width=True` : 컨테이너 너비에 맞춰 차트 크기 자동 조절 → 모바일/데스크탑 일관된 UX

**Matplotlib 연동**

```python
fig, ax = plt.subplots()
ax.hist(data, bins=10)
st.pyplot(fig)
plt.close(fig)  # 메모리 해제
```

---

## 5. KPI 대시보드 — st.metric

### 핵심 개념
- `st.metric()` : 현재값 + 변화량(Delta)을 BI 도구 수준의 카드로 시각화
- Delta 양수(+) → 초록 상승 화살표 / 음수(-) → 빨강 하락 화살표 자동 표시

### 상세 내용

```python
col1, col2, col3, col4 = st.columns(4)
col1.metric("총 API 호출", "15,240건", "+12%")
col2.metric("평균 응답속도", "142ms", "-15ms")
col3.metric("오류율", "0.2%", "-0.05%", delta_color="inverse")  # 낮을수록 좋은 지표
col4.metric("활성 모델 버전", "v2.5.1", "Stable")
```

**데이터 출력 방식 비교**

| 위젯 | 형태 | 특징 |
|---|---|---|
| `st.dataframe()` | 인터랙티브 | 컬럼 정렬, 검색, 크기 조절 가능 |
| `st.table()` | 정적(Static) | 전체 데이터를 그대로 출력 |

---

## 6. session_state — 상태 관리의 핵심

### 핵심 개념
- Streamlit은 위젯 조작 시마다 전체 스크립트를 재실행 → 일반 변수는 매번 초기화됨
- `st.session_state` : 재실행에도 값이 유지되는 딕셔너리 형태의 저장소

### 상세 내용

```python
# 초기화 패턴 (없을 때만 생성)
if "count" not in st.session_state:
    st.session_state.count = 0

if st.button("카운트 증가"):
    st.session_state.count += 1
```

**AI 챗봇에서의 핵심 역할**
- 대화 기록을 `st.session_state.messages` 리스트에 누적 → 재실행 후에도 이전 문맥 유지

**주요 활용 사례**

- 사용자 인증 상태 유지
- 필터 선택값·위젯 설정 보존
- 파일 업로드 결과·DataFrame 수정사항 저장
- 복잡한 연산의 중간 결과 캐싱

---

## 7. AI 챗봇 UI 구현

### 핵심 개념
- `st.chat_input()` + `st.chat_message()` + `session_state` 조합으로 채팅 UI 구현

### 상세 내용

**채팅 전용 위젯**
- `st.chat_input()` : 화면 하단 고정 입력창, 엔터/전송 시 텍스트 반환
- `st.chat_message("user")` / `st.chat_message("assistant")` : 역할별 아이콘·말풍선 렌더링

**Walrus 연산자 패턴** (`:=`)

```python
# 입력값 할당과 조건 판단을 한 줄에
if prompt := st.chat_input("무엇이든 물어보세요"):
    st.chat_message("user").markdown(prompt)
```

**UX 향상 기술**

- `st.write_stream()` : 타자기 효과 (스트리밍 응답)
- `with st.spinner("생각 중...")` : API 호출 중 로딩 표시

---

## 8. 통합 아키텍처 및 성능 최적화

### 핵심 개념
- 멀티페이지 구조 + API 로직 분리로 확장성 확보
- `@st.cache_data` 데코레이터로 불필요한 재연산 방지

### 상세 내용

**권장 파일 구조**
```
project_root/
├── app.py                # 메인 진입점
├── pages/
│   ├── 01_dashboard.py   # 데이터 시각화
│   ├── 02_chatbot.py     # AI 챗봇
│   └── 03_settings.py    # 설정
└── utils/
    └── api.py            # API 호출·비즈니스 로직 분리
```
- `pages/` 내 파일명 앞 숫자가 사이드바 메뉴 순서를 결정

**캐싱 전략**
```python
@st.cache_data
def load_data():
    return pd.read_csv("large_data.csv")  # 최초 1회만 실행

@st.cache_data(ttl=3600)  # 1시간 후 캐시 만료
def fetch_api_data(query):
    return requests.get(f"...?q={query}").json()
```
- `@st.cache_resource` : DB 연결, ML 모델 로딩 등 앱 전체에서 공유하는 객체용

---

## 9. Streamlit Community Cloud 배포

### 핵심 개념
- 무료 플랜으로 최대 3개 앱 배포 (1GB RAM), GitHub 연동 CI/CD 내장

### 상세 내용

**5단계 배포 절차**
1. GitHub public 저장소 생성 → `app.py` 업로드, `requirements.txt` 작성
2. `share.streamlit.io` 가입 → GitHub 계정 연동
3. "New app" → 저장소·브랜치·파일 선택
4. Advanced settings에서 환경변수(Secrets) 입력
5. Deploy! → `yourapp.streamlit.app` URL 자동 생성

**Secrets 관리** (`.streamlit/secrets.toml`)

```toml
OPENAI_API_KEY = "sk-xxxxxxxxxxxx"
```
코드에서는 `st.secrets["OPENAI_API_KEY"]`로 접근

---

## 오늘의 핵심 인사이트

> 강의를 듣고 직접 재해석한 내용.

- **Streamlit의 Rerun 구조는 편리함이자 제약이다.** 버튼 하나 누를 때마다 전체 스크립트가 재실행되는 방식은 간단한 앱에서는 괜찮지만, 복잡한 UI에서는 성능 저하와 UX 불일치가 생길 수 있다. `session_state`와 `@st.cache_data`는 이 구조적 한계를 보완하기 위한 도구다.
- **Streamlit은 "내부용 도구"에 최적화되어 있다.** 일반 사용자 대상 서비스 품질을 기대하기는 어렵고, 데이터를 다루는 본인이나 팀 내부용(모니터링 대시보드, ML 데모, 빠른 프로토타입 검증)에 적합한 도구다. 바이브코딩이나 Next.js 같은 도구가 있는 상황에서 Streamlit의 포지션은 "HTML/CSS/JS 없이 Python만으로 빠르게 시각화하고 싶은 사람"을 위한 것임을 이번 강의에서 명확히 이해했다.
- **Python 문법 이해가 Streamlit 실습의 진짜 장벽이다.** 들여쓰기 오류, 주석 처리 미숙 등 기초 문법 문제로 강의 진행이 자주 끊겼다. Python 사전 교육의 격차가 현장에서 바로 드러나는 상황이다. 도움을 드리면서 이 부분이 팀원들의 실질적인 병목임을 체감했다.
- **캐싱 전략은 단순 최적화가 아니라 아키텍처 설계다.** `@st.cache_data`와 `@st.cache_resource`를 어디에 쓰느냐는 "무엇이 자주 바뀌고 무엇이 공유되어야 하는가"를 묻는 구조적 결정이다.

---

## 아직 찾아보지 않은 것

- Streamlit의 Rerun 방식이 React의 상태관리(useState)와 어떻게 다른지
- `st.write_stream()`의 실제 구현 방식

---

*작성일: 2026-05-15 | 인공지능 사관학교 AI 보안 과정*
