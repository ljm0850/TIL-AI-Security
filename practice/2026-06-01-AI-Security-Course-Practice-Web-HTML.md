# 2026-06-01 실습 — Web HTML/CSS 기초

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-01-AI-Security-Course-Lecture-Web.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-01

> HTML 실습은 잊고 있던 지식의 리마인드였다. React를 쓰면서 직접 쓸 일이 없었던 태그 선택자(`h1`, `.class`, `#id`)와 인라인 스타일을 다시 써보니, 이게 결국 React의 `className`, `style={{ }}` 문법의 원형임이 보였다. 추상화 레이어만 다를 뿐 개념은 같다.

> `<font>` 태그가 HTML5에서 폐기된 이유를 직접 코드로 보니 납득이 갔다. 구조에 스타일이 섞여 있으면 나중에 수정이 힘들다. 바로 단일 책임 원칙이 HTML에도 적용되는 셈.

---

## 실습 개요

- **목표**: HTML 문서 기본 구조 이해 + CSS 선택자 우선순위 실습
- **방법**: `index.html`에서 같은 `<h1>` 태그에 4가지 방식으로 스타일 적용 비교

---

## 진행 과정

### Step 1. HTML 기본 구조 확인

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <!-- CSS는 head에서 정의 — 출력되지 않는 정보 -->
    <style>
        /* ... */
    </style>
</head>
<body>
    <!-- 실제 출력되는 내용 -->
</body>
</html>
```

### Step 2. CSS 선택자 우선순위 실험

같은 `<h1>` 태그에 4가지 방식을 적용해서 우선순위 직접 확인:

```html
<h1>HTML BASIC</h1>                         <!-- tag 선택자: coral, 50px -->
<h1 class="title">HTML BASIC</h1>           <!-- class 선택자: red, 60px -->
<h1 id="title">HTML BASIC</h1>              <!-- id 선택자: cornflowerblue, 70px -->
<h1 style="color: green; font-size: 80px;"> <!-- inline 스타일: green, 80px -->
```

```css
h1 { color:coral; font-size: 50px; }
.title { color:red; font-size: 60px; }
#title { color:cornflowerblue; font-size: 70px; }
```

**우선순위 확인**: inline style > #id > .class > tag

### Step 3. 폐기 태그 확인

```html
<!-- HTML5에서 폐기된 태그 — 실제로 브라우저가 렌더링은 하지만 비권장 -->
<font size="7" face="궁서체" color="blue">HTML BASIC</font>
```

구조(HTML)에 스타일 정보가 섞여 있어서 폐기됨. CSS의 `font-family`, `font-size`, `color`로 대체.

---

## 결과 / 산출물

- 같은 태그에 4가지 스타일이 각각 다르게 적용된 것 확인
- 선택자 우선순위: `inline > #id > .class > tag` 시각적으로 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 선택자 우선순위를 코드로 직접 눈으로 확인 | CSS 복합선택자, 가상클래스는 아직 실습 전 |
| `<head>` / `<body>` 역할 구분 재확인 | |

---

## 다음에 이어서 할 것

- CSS 레이아웃 (Box Model, Flexbox, Grid)
- Form 태그 실습

---
*작성일: 2026-06-01 | 인공지능 사관학교 AI 보안 과정*
