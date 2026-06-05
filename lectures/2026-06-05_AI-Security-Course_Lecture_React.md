# 2026-06-05 강의 정리 — React

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: ReactJS 소개 · 모듈 시스템 · JSX · 컴포넌트 · useState
> **관련 실습 파일**: ../practice/2026-06-05_AI-Security-Course_Practice_React.md

---

## 핵심 인사이트 모음

### 📅 2026-06-05

- 바이브코딩으로 시작한 프로젝트의 프론트엔드를 완성하고 나서 보니 구조가 난잡했고, 리팩토링에 많은 자원이 들고 있다. 컴포넌트 분리와 상태관리가 처음부터 설계되지 않으면 나중에 고치는 비용이 크다는 걸 몸으로 겪었다. React가 강제하는 컴포넌트 단위 설계가 왜 중요한지 그 경험이 있어서 더 와닿았다.
- `onClick` 같은 이벤트 핸들러가 콜백함수라는 걸 예전엔 그냥 넘겼는데, 이번에 JS의 일급시민 개념을 제대로 이해하고 나서 보니 다르게 보인다. "함수를 값으로 넘긴다"는 관점이 생기니까 코드가 읽히는 방식이 달라졌다.
- Virtual DOM의 핵심은 "변경된 부분만 실제 DOM에 반영"이다. `innerHTML`로 전체를 갈아치우는 방식과 비교하면, 왜 React가 성능상 이점이 있는지 바로 이해된다.
- `useState`의 상태값 변경함수를 호출하면 컴포넌트가 리렌더링된다는 구조가, Vue의 `v-model` 양방향 바인딩과 비교하면 React는 단방향이라는 차이가 선명해진다.

---

## 1. ReactJS 개요

### 핵심 개념

- Facebook 주도로 2011년부터 제공된 JavaScript UI 오픈소스 경량 라이브러리 (MIT)
- MVC 패턴에서 **View(V) 담당** — UI 이외 기능은 외부 라이브러리 사용
- 핵심 패러다임: **컴포넌트 단위 UI 조립**

### 상세 내용

**React 주요 특징**

| 주요특징 | 설명 | 장점 |
|---|---|---|
| 컴포넌트 기반 | UI를 독립적 컴포넌트 단위로 분리·조합 | 재사용성 극대화, 팀 협업 효율 향상 |
| 선언형 UI | '어떻게(How)' 아닌 '무엇(What)'을 기술 | 코드 가독성·예측성 향상 |
| Virtual DOM | 가상 DOM과 실제 DOM 비교(Diffing)하여 변경된 최소 부분만 반영 | 불필요한 DOM 조작 제거로 빠른 렌더링 |
| 단방향 데이터 흐름 | 데이터는 부모→자식(Props) 방향으로만 흐름 | 데이터 흐름 추적 용이, 버그 감소 |

**Virtual DOM이 왜 빠른가 — innerHTML vs textContent 비교**

```html
<!-- 전체 재렌더링: innerHTML -->
app.innerHTML = `<h3>Count</h3><div><strong>${n}개</strong></div><hr/>`;
// → 숫자 하나 바뀌었는데 h3, div, strong, hr 노드 전부 재생성

<!-- 부분 업데이트: textContent -->
document.getElementById('count').textContent = `${n}개`;
// → 텍스트 노드 1개만 교체
```

React의 Virtual DOM은 이전 상태와 현재 상태를 비교(Diffing)하여 실제로 바뀐 노드만 DOM에 반영한다. `textContent` 방식을 React가 자동으로 해준다고 이해하면 된다.

**React 생태계**

| 역할 | 라이브러리 |
|---|---|
| 서버 상태 관리 | React Query |
| 라우팅 | React Router |
| 전역 상태 관리 | Zustand / Redux |
| SSR | Next.js |
| 모바일 앱 | React Native |

---

## 2. 모듈 시스템

### 핵심 개념

- 모듈: 파일 단위의 기능적 코드 단위. `export`로 지정하지 않으면 외부에서 접근 불가
- **CJS(CommonJS)**: Node.js 기본. `require()` / `module.exports`. 동기식. `.cjs`
- **ESM(ECMAScript Module)**: ECMA 표준. `import` / `export`. 비동기식. `.mjs`

### 상세 내용

**CJS vs ESM 비교**

| 항목 | CommonJS (CJS) | ES Module (ESM) |
|---|---|---|
| 문법 | `require()` / `module.exports` | `import` / `export` |
| 처리 방식 | 동기식 | 비동기식 |
| 위치 제약 | 어디서나 사용 가능 (`if`문 내부도 가능) | 최상위 스코프(Top Level)에만 선언 |
| 기본 확장자 | `.js` (type: commonjs) / `.cjs` | `.js` (type: module) / `.mjs` |
| strict mode | 선택 | 기본 적용 |
| 현재 권장 | Node.js 레거시 호환 | 신규 프로젝트 권장 |

```javascript
// CJS
const { add, sub } = require('./math');
module.exports = { add, sub };

// ESM
import { add, sub } from './math.js';
export { add, sub };
export default { add };  // 피하는 것 권장 (모호성 발생)
```

**배럴 패턴 (Barrel Pattern)**

모듈이 많아지면 `import`가 반복된다. 폴더에 `index.js`를 두고 모듈을 한데 모아 내보내는 패턴.

```javascript
// modules/stores/index.js
export * from './ModuleA';
export * from './ModuleB';
export * from './ModuleC';

// 사용 측 — 경로가 단순해짐
import { ModuleA, ModuleB, ModuleC } from 'modules/stores';
```

단점: 모듈 수가 많아지면 출처 파악이 어렵고, Webpack의 Tree Shaking에 문제가 생길 수 있다.

---

## 3. 개발 환경 — Vite

### 핵심 개념

- **Vite**: 프론트엔드 빌드 도구. ES Module 기반으로 필요한 파일만 로딩하여 Webpack 대비 빠른 개발 서버 시작
- Webpack은 전체 앱을 번들링 후 서버를 띄우지만, Vite는 요청 시점에 필요한 모듈만 변환

```bash
npm create vite@latest
# → Project name, framework(React), variant(JavaScript) 선택

cd react-helloworld
npm install
npm run dev
```

**React 프로젝트 기본 구조**

```
react-helloworld/
├── src/
│   ├── App.jsx          ← 루트 컴포넌트
│   ├── App.css
│   ├── main.jsx         ← 진입점 (ReactDOM.render)
│   └── components/      ← 컴포넌트 폴더
│       └── HelloWorld.jsx
├── public/
├── index.html
└── package.json
```

---

## 4. JSX

### 핵심 개념

- **JSX**: XML 형식으로 React 요소 트리를 생성하는 자바스크립트 확장 표현식
- 문법 설탕(Syntactic Sugar) — Babel이 빌드 시 `React.createElement()` 호출로 변환

### 상세 내용

**JSX 주요 규칙**

| 규칙 | 설명 | 예시 |
|---|---|---|
| 단일 루트 요소 | 반환값은 하나의 루트 태그로 감싸야 함 | `<div>...</div>` 또는 `<>` (Fragment) |
| className | HTML의 `class` 대신 `className` 사용 | `<div className="btn">` |
| 표현식 삽입 | `{}`로 JavaScript 표현식 삽입 | `{user.name}` `{2+3}` `{fn()}` |
| 셀프 클로징 | 내용 없는 태그는 `/`로 닫기 | `<Input />` `<br />` |
| camelCase 속성 | HTML 속성명은 camelCase | `onClick` `onChange` `htmlFor` |
| 인라인 스타일 | style은 JavaScript 객체로 | `style={{ color: 'red', fontSize: 16 }}` |
| 주석 | `{}`로 감싼 블록 주석 | `{/* JSX 주석 */}` |

```jsx
// JSX와 React.createElement() 는 동일하다
const greeting = <h1 className="class" style={style}>{name}</h1>;
const greeting = React.createElement('h1', { className: 'class', style }, name);

// 조건부 렌더링
{add(data) >= 10 ? <h3>10 이상</h3> : <h2>10 미만</h2>}

// 단축 평가 (&&)
{add(data) % 2 === 0 && '짝수'}

// null 병합 연산자
{str ?? 'default'}        // str이 null/undefined이면 'default'
{data?.c ?? 'default'}    // optional chaining + null 병합

// Boolean, null, undefined는 렌더링되지 않음
```

---

## 5. 컴포넌트 (Component)

### 핵심 개념

- React 앱의 UI를 구성하는 재사용 가능한 기본 단위
- 자바스크립트 함수로 작성 → `props`로 데이터를 받고 → React 엘리먼트를 반환
- 컴포넌트명은 **PascalCase** (HTML 태그와 구분)
- 현재는 **함수 컴포넌트** 권장 (클래스 컴포넌트는 v16.8 이전 레거시)

### 상세 내용

**컴포넌트 구조 예시**

```jsx
// App.jsx — 루트 컴포넌트
import HelloWorld from './components/HelloWorld';

function App() {
    return (
        <>
            <h1>Hello World!!!</h1>
            <HelloWorld />   {/* 컴포넌트 사용 */}
        </>
    );
}
export default App;

// components/HelloWorld.jsx — 자식 컴포넌트
function HelloWorld() {
    return (
        <div>
            <h1 style={{ fontStyle: 'italic' }} className="counter">
                Hello World!!
            </h1>
        </div>
    );
}
export default HelloWorld;
```

**컴포넌트 계층 구조**

```
App
├── Header
│   ├── Nav
│   └── Logo
├── Main
│   ├── PostList
│   └── UserInfo
└── Footer
    └── Copyright
        
공통: Button, List, Paging  ← 여러 컴포넌트에서 재사용
```

**Props — 부모 → 자식 단방향 데이터 전달**

```jsx
// 부모
function App() {
    return <Welcome name="홍길동" />;
}

// 자식 — props는 읽기 전용(immutable)
function Welcome(props) {
    return <h3>{props.name}</h3>;
}

// 객체 구조 분해로 간결하게
function Add2({ a, b }) {
    return <h2>{a} + {b} = {a + b}</h2>;
}
<Add2 a={1} b={2} />
```

**배열 렌더링 시 key 필수**

```jsx
topics.map((t) =>
    <li key={t.id}><a href={'/read/' + t.id}>{t.title}</a></li>
)
// key: React가 리렌더링 시 어떤 요소가 바뀌었는지 추적하는 고유 식별자
// 형제 노드 내에서만 유일하면 됨
```

---

## 6. Virtual DOM

### 핵심 개념

- React DOM은 브라우저 DOM을 투영한 **Virtual DOM**(메모리상 가상 DOM 트리)을 유지
- 상태 변경 시: 새 Virtual DOM 생성 → 이전 Virtual DOM과 **Diffing** → 변경된 노드만 실제 DOM에 반영
- React 엘리먼트는 **불변(immutable)**. 수정이 아니라 새 엘리먼트를 생성해서 갱신

---

## 7. useState

### 핵심 개념

- 컴포넌트 내부 상태를 동적으로 관리하는 기본 React Hook
- `props`는 외부에서 읽기 전용으로 들어오지만, `state`는 컴포넌트 내부에서 변경 가능
- 상태값 변경함수를 호출하면 컴포넌트가 **비동기적으로 리렌더링**됨

### 상세 내용

```jsx
import { useState } from 'react';

const [mode, setMode] = useState('READ');          // 기본형
const [user, setUser] = useState({ mode: 'READ', level: 'ADMIN' }); // 객체형

// 상태 변경
setMode('WRITE');                                  // 기본형
setCount(prev => prev + 1);                        // 이전 값 기반
setUser(prv => ({ ...prv, mode: 'WRITE' }));       // 객체 — 스프레드로 불변성 유지
setUsers(prv => [...prv, { ...item, id: Date.now() }]); // 배열 추가
setUsers(prv => prv.filter(c => c.id !== id));     // 배열 삭제
```

**단방향 데이터 흐름과 state 끌어올리기 (Lifting State Up)**

React는 부모 → 자식 방향 단방향이 기본. 형제 컴포넌트 간 직접 데이터 전달 불가. 자식의 상태를 형제와 공유해야 할 경우, 공통 부모로 state를 끌어올려 props로 내려준다.

트리 깊이가 깊어지면 props를 계속 내려줘야 하는 **props drilling** 문제 발생 → Context, Redux 등 전역 상태관리 라이브러리로 해결.

**이벤트 처리 관례**

```jsx
// 이벤트명: on + 동작 (camelCase)
// 핸들러명: handle + 동작
const handleBoardInsert = (event) => {
    event.preventDefault();
    console.log('글이 등록되었습니다.');
};

<button onClick={handleBoardInsert}>글등록</button>
```

이벤트 핸들러는 React 컴포넌트에서는 props로 전달되고, 실제 처리는 DOM 엘리먼트에서 이루어진다. 브라우저 이벤트가 필요하면 `event.nativeEvent`로 접근.

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| React/Vue 사용 시 Reflow·Repaint에 신경 안 써도 될까? | 2026-06-05 | Virtual DOM이 안전망을 제공하지만 key, memo, useCallback 미사용 시 이중 부담 가능 | `../deep-dive/2026-06-05_React-Vue-Reflow-Repaint.md` |

---

*작성일: 2026-06-05 | 인공지능 사관학교 AI 보안 과정*
