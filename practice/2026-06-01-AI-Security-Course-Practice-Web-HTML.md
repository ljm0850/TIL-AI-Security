# 2026-06-01~2 실습 — Web HTML/CSS 기초

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-01-AI-Security-Course-Lecture-Web.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-01

> HTML 실습은 잊고 있던 지식의 리마인드였다. React를 쓰면서 직접 쓸 일이 없었던 태그 선택자(`h1`, `.class`, `#id`)와 인라인 스타일을 다시 써보니, 이게 결국 React의 `className`, `style={{ }}` 문법의 원형임이 보였다. 추상화 레이어만 다를 뿐 개념은 같다.

> `<font>` 태그가 HTML5에서 폐기된 이유를 직접 코드로 보니 납득이 갔다. 구조에 스타일이 섞여 있으면 나중에 수정이 힘들다. 바로 단일 책임 원칙이 HTML에도 적용되는 셈.

### 📅 2026-06-02 (이어서)

> CSS 선택자를 코드로 직접 실습하면서, 이걸 jQuery가 그대로 가져다 쓴다는 게 보였다. `$('li[title$="품절"]')` — CSS 속성 선택자 문법 그 자체다. CSS를 제대로 알면 jQuery도 자연스럽게 읽힌다.

> `<datalist>`를 직접 써보니 확실히 인상적이었다. JavaScript로 구현하던 자동완성 기능이 HTML 태그 하나로 된다. `<input type="number" min max>`, `<input type="date" min max>`, `<input pattern>` 모두 마찬가지 — 입력 유효성 검사를 JavaScript 없이 HTML 속성으로 처리할 수 있다. 바닐라 HTML을 제대로 알았더라면 예전에 JavaScript로 직접 구현한 코드들 중 상당수가 불필요했을 것 같다.

> `index.html`에서 `document.getElementById("title").innerText`로 DOM을 직접 조작하는 코드를 써봤다. 버튼 클릭 한 번에 DOM이 변경되고 리렌더링이 일어나는 흐름 — React의 state 변경 → 리렌더링과 같은 원리인데, React가 그 과정을 얼마나 추상화해놓은 건지 다시 와 닿았다.

---

## 실습 개요

- **목표**: HTML 문서 기본 구조 이해 + CSS 선택자 전체 실습 + Form 태그 실습
- **방법**: `index.html` → `css_selector.html` → `form.html` 순서로 진행

---

## 진행 과정

### Step 1. HTML 기본 구조 + CSS 선택자 우선순위 (`index.html`)

같은 `<h1>` 태그에 4가지 방식을 적용해서 우선순위 직접 확인:

```html
<h1>HTML BASIC</h1>                              <!-- tag 선택자: coral, 50px -->
<h1 class="title">HTML BASIC</h1>               <!-- class 선택자: red, 60px -->
<h1 id="title">HTML BASIC</h1>                  <!-- id 선택자: cornflowerblue, 70px -->
<h1 style="color: green; font-size: 80px;">     <!-- inline 스타일: green, 80px -->
```

**우선순위 확인**: `inline > #id > .class > tag`

JavaScript로 DOM 직접 조작:

```javascript
function changeContent() {
    document.getElementById("title").innerText = "HTML 기본";
}
```

버튼 클릭 → `getElementById`로 DOM 요소 참조 → `innerText` 변경 → 리렌더링 발생.

---

### Step 2. CSS 선택자 전체 실습 (`css_selector.html`)

**외부 CSS + @import, @media 쿼리**

```css
@import url("style.css") screen and (min-width:200px) and (max-width:400px);

@media screen and (min-width:200px) and (max-width:400px) {
    #media_query { color: blue; font-weight: bold; }
}
```

화면 폭 200~400px 구간에서만 스타일 적용. 반응형의 기본 단위.

**복합 선택자 비교 실험**

```css
ul li { color:blue; }          /* 자손 선택자 — ul 하위 모든 li */
ul > li { background-color:pink; }  /* 자식 선택자 — ul의 직계 자식 li만 */
li#기준 ~ li { color:pink; }   /* 일반 형제 — #기준 뒤의 모든 형제 li */
li#기준 + li { color:green; }  /* 인접 형제 — #기준 바로 다음 li 하나만 */
```

자손(`공백`)과 자식(`>`)의 차이를 중첩 구조로 직접 확인.

**속성 선택자 — 실무 활용도 높음**

```css
li[title] { color: orange; }           /* title 속성 존재 */
li[title="할인"] { color: red; }        /* 정확히 일치 */
li[title~="무료배송"]::after { content: "(무료배송)"; }  /* 공백 구분 단어 포함 */
li[title$="품절"] { color: gray; }      /* "품절"로 끝남 */
li[title^="hot"] { color: red; font-weight: bold; }     /* "hot"으로 시작 */
li[title*="무료"] { text-decoration: underline; }       /* "무료" 포함 */
```

**가상 선택자**

```css
#fruit > ol > li:nth-child(2n) { background-color:lime; }  /* 짝수 번째 항목 */
#fruit > ol > li:hover { text-decoration: underline; }      /* 마우스 올림 */
```

**jQuery와 CSS 선택자의 관계**

```javascript
// CSS 속성 선택자 문법을 jQuery가 그대로 사용
$('li[title$="품절"]').on('click', function() {
    alert($(this).text() + "은 품절입니다.");
});
```

CSS `[title$="품절"]`과 jQuery `$('li[title$="품절"]')` — 완전히 같은 문법. CSS 선택자를 알면 jQuery 쿼리가 자연스럽게 읽힌다.

---

### Step 3. Form 태그 실습 (`form.html`)

**HTML이 기본 제공하는 기능들**

```html
<!-- 자동완성 입력 — JS 없이 -->
<input type="text" name="local" list="local" />
<datalist id="local">
    <option value="경기">경기도</option>
    <option value="서울">서울시</option>
</datalist>

<!-- 숫자 범위 제한 -->
<input type="number" min="0" max="120">

<!-- 날짜 범위 제한 -->
<input type="date" min="2023-01-01" max="2023-12-31">

<!-- 필수값 + 길이 제한 + 플레이스홀더 -->
<input type="text" name="name" maxlength="10" required placeholder="이름을 입력해 주세요." />

<!-- 체크박스: label for + input id 연결 -->
<input type="checkbox" name="취미" value="등산" id="a" checked>
<label for="a">등산</label>

<!-- hidden 필드 -->
<input type="hidden" name="hidden" value="">
```

**기타 태그 실습**

```html
<!-- details-summary: JS 없이 아코디언 -->
<details>
    <summary>질문 1</summary>
    <p>오늘 점심은 무엇을 먹을까요?</p>
</details>

<!-- meter: 값의 점유 정도 -->
<meter value="0.8">80%</meter>

<!-- progress: 진행률 -->
<progress value="2" max="10"></progress>

<!-- 다운로드 링크 -->
<a href="index.html" download="html_예제">HTML 기초 다운로드</a>
```

---

## 결과 / 산출물

- CSS 선택자 전체 유형(기본/복합/속성/가상) 코드로 직접 확인
- Form 태그에서 HTML 기본 기능만으로 처리 가능한 범위 파악
- jQuery 쿼리와 CSS 선택자의 관계 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 복합 선택자 자손/자식/형제 차이를 한 파일에서 비교 |
| jQuery가 CSS 선택자를 그대로 사용한다는 연결고리 확인 |
| Form 태그에서 JS 없이 처리 가능한 기능 범위 파악 |

---

## 다음에 이어서 할 것

- CSS 레이아웃 심화 (Box Model, `position`, `float`, Flexbox)
- JavaScript 기초 연결

## 아직 찾아보지 않은 것

- `::after { content: "..." }` — 가상 요소로 콘텐츠를 삽입하는 방식이 실제로 어떻게 DOM에 존재하는지 (스크린 리더, 접근성 영향)
- `@import` vs `<link>` — 외부 CSS를 불러오는 두 방법의 성능 차이

---
*작성일: 2026-06-01 ~ 2026-06-02 | 인공지능 사관학교 AI 보안 과정*
