# 2026-06-01 강의 정리 — Web Programming (WWW / HTML / CSS)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: WWW, HTTP, HTML5, CSS 기초
> **관련 실습 파일**: `../practice/2026-06-01-AI-Security-Course-Practice-Web-HTML.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-01

> 바이브 코딩으로 구현 난이도가 낮아진 지금, 제대로 만들어졌는지 검증할 수 있는 엔지니어가 되어야 한다는 말이 가장 인상 깊었다. "만드는 능력"보다 "판단하는 능력"이 더 중요해지는 흐름.

> HTML은 잊고 있었던 지식을 리마인드하는 기회였다. React 쓰면서 쓰지 않던 용어들(`<head>`, `<body>`, `<!DOCTYPE html>`)을 다시 들었는데, 이 개념들이 실제로 어떻게 렌더링에 영향을 미치는지(DOM Tree → CSSOM Tree → Rendering Tree → Layout → Painting) 브라우저 렌더링 흐름으로 연결되니 왜 중요한지 다시 보였다.

> `<head>`는 정의하는 곳, `<body>`는 출력하는 곳 — 단순한 구분처럼 보이지만, 이게 바로 HTML이 "마크업과 출력을 분리한다"는 설계 원칙의 핵심이다. CSS도 마찬가지 — 구조(HTML)와 표현(CSS)을 분리하는 게 목적.

### 📅 2026-06-02 (이어서)

> 시맨틱 태그의 의미를 다시 생각하게 된 날이다. 기존엔 `<div>`에 CSS 입히면 브라우저에 보이는 결과는 같으니 태그의 의미가 크지 않다고 생각했다. 그런데 지금 AI가 HTML을 읽는 시대가 됐다는 걸 생각하면 — 시맨틱 태그가 단순히 "개발자 컨벤션"이 아니라 기계가 문서를 이해하는 근거가 된다. 관점이 바뀌었다.

> `<datalist>` 같은 태그를 보면서 과거에 JavaScript로 직접 구현했던 기능들이 이미 HTML 스펙에 있다는 걸 알게 됐다. 그때 바닐라 CSS/HTML을 제대로 알았더라면 라이브러리 없이도 됐을 것들이 있다. "Tailwind 쓰지 말고 바닐라 CSS 쓰자"고 주장하는 이유를 이제야 이해했다 — 표준을 모른 채 도구에 의존하면 표준이 이미 해결한 것도 못 쓰게 된다.

> `for`문으로 DOM에 요소를 반복 추가하면 매번 리렌더링이 일어난다는 것, 네이버가 HTML 변경 이슈를 감지하기 위해 주기적으로 체크 코드를 심어둔다는 것 — 이런 실무 디테일이 강의에서 나왔다. 보안 관점에서도 DOM 조작 감지는 중요한 이야기다.

---

## 1. WWW & HTTP

### 핵심 개념

- WWW: 팀 버너스리가 1989년 제안. HTML + URI + HTTP 세 기술의 조합
- HTTP: 웹의 통신 규약. **Request/Response 한 쌍**, 동기식(Synchronous)
- **Connectionless**: 응답이 끝나면 연결을 끊는다
- **Stateless**: 클라이언트 상태를 서버가 저장하지 않는다

### 상세 내용

Connectionless + Stateless 조합 → 서버 자원 효율적 사용. 그러나 로그인 상태 유지 등이 필요할 때는 Cookie/Session/Token으로 별도 처리 필요.

**WWW 서비스 구조**

```
Web Client → (HTTP) → Web Server → Web Application Server → DB Server
                           ↓
                    Static Resource
```

---

## 2. HTML5

### 핵심 개념

- HTML = HyperText Markup Language. 태그(Tag)로 문서 구조를 표현
- `<!DOCTYPE html>` — HTML5 버전 선언. 과거 버전보다 단순해짐
- `<head>` — 브라우저가 참조할 부가 정보 선언. **화면에 출력되지 않음**
- `<body>` — 실제 출력할 내용

### 상세 내용

**HTML5의 주요 특징**
- Semantic Tag 도입 (`<header>`, `<section>`, `<nav>`, `<article>`, `<footer>` 등) — 구조에 의미를 부여
- 마크업(구조)과 출력(스타일)의 분리 → CSS 역할 강화
- `<font>`, `<center>` 등 이전 태그 폐기 → CSS로 대체

**HTML 문서 구조 기본**

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="utf-8">
    <title>페이지 제목</title>
  </head>
  <body>
    <!-- 출력 내용 -->
  </body>
</html>
```

**주요 태그 분류**

| 분류 | 태그 |
|---|---|
| 문서/메타 | `<html>`, `<head>`, `<body>`, `<meta>`, `<title>`, `<link>` |
| 글자모양 | `<h1~6>`, `<b>`, `<strong>`, `<i>`, `<em>`, `<mark>` |
| 문단 | `<p>`, `<div>`, `<span>`, `<pre>` |
| 구조(시맨틱) | `<header>`, `<footer>`, `<nav>`, `<section>`, `<article>`, `<aside>` |
| 미디어 | `<video>`, `<audio>`, `<img>` |
| 폼 | `<form>`, `<input>`, `<textarea>`, `<select>`, `<button>` |
| 그래픽 | `<canvas>` (픽셀 기반), `<svg>` (벡터 기반) |

**Block vs Inline 요소**
- Block: `<div>`, `<p>`, `<h1>` 등 → 줄바꿈 발생, 인라인 요소에 중첩 불가
- Inline: `<span>`, `<a>`, `<strong>` 등 → 줄바꿈 없음, 블록 안에 중첩 가능

---

## 3. CSS

### 핵심 개념

- CSS = Cascading Style Sheets. HTML의 출력 방법을 기술하는 언어
- 구조(HTML)와 표현(CSS) 분리 → 유지보수 효율 향상

### 상세 내용

**CSS 선언 방법 (우선순위: Inline > Internal > External)**

```css
/* CSS Rule 구조 */
선택자(selector) { 속성(property): 값(value); }

h1 { color: blue; font-size: 50px; }
.title { color: red; }      /* class 선택자 */
#title { color: blue; }     /* id 선택자 */
```

**선택자 종류**
- 일반: Universal(`*`), Type(태그명), Class(`.`), Id(`#`), 속성
- 복합: `+`, `>`, `~` 조합
- 가상클래스: 요소의 상태 (`:hover`, `:focus` 등)

---

## 4. 브라우저 렌더링 과정

### 핵심 개념

HTML과 CSS가 화면에 표시되기까지의 과정 (Webkit 기준):

```
HTML → HTML Parser → DOM Tree ─┐
                                 ├→ Rendering Tree → Layout → Painting → Display
CSS  → CSS Parser  → CSSOM Tree ─┘
```

`<head>`에서 CSS를 로드하는 이유: CSSOM Tree 구성이 먼저 완료되어야 Rendering Tree를 만들 수 있기 때문.

---

---

## 📅 2026-06-02 이어서

### 5. 시맨틱 태그의 재발견

기존엔 `<div>` + CSS와 시맨틱 태그의 렌더링 결과가 같으니 큰 차이가 없다고 생각했다. 그런데 AI가 HTML을 파싱하는 시대에는 태그가 가진 의미가 기계의 문서 이해에 직접 영향을 미친다. SEO도, 접근성도, AI 파싱도 — 결국 다 같은 이유다.

```html
<!-- 의미 없는 구조 -->
<div class="header">...</div>
<div class="nav">...</div>

<!-- 시맨틱 구조 — 기계가 읽을 수 있다 -->
<header>...</header>
<nav>...</nav>
```

### 6. HTML이 이미 해결한 것들

JavaScript로 직접 구현했던 기능들이 HTML 표준에 이미 있다.

**`<datalist>` — 자동완성 입력 필드**

```html
<input type="text" name="local" list="local" />
<datalist id="local">
    <option value="경기">경기도</option>
    <option value="서울">서울시</option>
    <option value="인천">인천시</option>
</datalist>
```

JavaScript로 구현하던 검색/필터 기능을 HTML 한 줄로 처리. 정규식 검증(`<input type="email">`, `pattern` 속성)도 마찬가지.

**CSS3 고급 기능들**

강의 자료에서 다룬 범위 중 주요한 것들:

| 기능 | 속성 | 설명 |
|---|---|---|
| CSS 우선순위 | `!important > inline > #id > .class > tag > 상속` | 동일 속성은 마지막 선언이 override |
| Box Model | `margin / border / padding / content` | 모든 요소의 공간 계산 기준 |
| 배치 | `position: static\|relative\|absolute\|fixed` | 레이아웃 흐름 제어 |
| 미디어 쿼리 | `@media screen and (min-width: 401px)` | 반응형 레이아웃 |
| 애니메이션 | `@keyframes`, `animation-*` | CSS만으로 동적 효과 |
| 변환 | `transform: rotate\|scale\|translate\|skew` | 2D/3D 요소 변형 |

### 7. DOM과 렌더링 성능

**DOM 조작 시 리렌더링 발생**

```javascript
// 나쁜 패턴: 반복마다 DOM 변경 → 매번 리렌더링
for (let i = 0; i < 1000; i++) {
    document.body.appendChild(document.createElement('div'));
}

// 좋은 패턴: DocumentFragment로 묶어서 한 번에 삽입
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
    fragment.appendChild(document.createElement('div'));
}
document.body.appendChild(fragment);
```

**브라우저 자료저장소 (Application 탭)**

- Cookie / localStorage: 기본 클라이언트 저장
- IndexedDB: 오프라인 지원 / 모바일 PWA에서 주로 사용
- Service Worker: 멀티스레드, 오프라인 캐싱 처리
- Manifest: PWA 앱 메타 정보

**DOM 변경 감지 — 보안 관점**

네이버는 HTML 변경으로 인한 이슈를 줄이기 위해 일정 시간마다 DOM 상태를 체크하는 코드를 심어둔다고 한다. 이건 보안 관점에서도 중요한 패턴 — 페이지 위변조 감지에 DOM 감지 로직이 쓰인다.

---

## 아직 찾아보지 않은 것

- Reflow vs Repaint 차이 — DOM 변경 시 항상 둘 다 발생하는 건지, 언제 Repaint만 일어나는지
- `<canvas>` vs `<svg>` 성능 비교 — 실시간 데이터 시각화는 canvas, 정밀 고화질은 svg라는 기준의 구체적 근거
- CSS `!important` 사용이 안티패턴으로 여겨지는 이유
- Service Worker와 Web Worker의 차이 — 둘 다 멀티스레드인데 어떻게 다른지
- MutationObserver API — 네이버처럼 DOM 변경을 감지하는 표준 방법

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| | | | |

---
*작성일: 2026-06-01 ~ 2026-06-02 | 인공지능 사관학교 AI 보안 과정*
