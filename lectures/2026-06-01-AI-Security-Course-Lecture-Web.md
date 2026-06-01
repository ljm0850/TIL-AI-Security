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

*작성일: 2026-06-01 | 인공지능 사관학교 AI 보안 과정*
