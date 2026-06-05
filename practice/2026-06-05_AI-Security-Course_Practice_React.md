# 2026-06-05 실습 — React

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사 코드 따라치기 + 일부 변형)
> **관련 강의 파일**: `../lectures/2026-06-05_AI-Security-Course_Lecture_React.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-05

- `JSXBasic.jsx`에서 `||`, `??`, `?.` 세 가지 연산자를 나란히 쓰니 차이가 한눈에 비교됐다. `||`는 falsy 전체를 잡고, `??`는 null/undefined만 잡는다는 게 코드로 보니 명확했다.
- `PropsCompoChild.jsx`의 `DisplayPanel1`(props 객체 통째로)과 `DisplayPanel2`(구조 분해)를 나란히 만들어보니, 구조 분해 방식이 훨씬 읽기 편하다는 걸 직접 비교할 수 있었다.
- `ControlPanel`에서 부모가 내려준 함수를 자식이 버튼 클릭으로 호출하는 패턴을 보고, "자식→부모 방향은 콜백 함수로 구현한다"는 단방향 흐름의 원칙이 코드 수준에서 이해됐다.

---

## 실습 개요

- **목표**: React 기초 개념(컴포넌트, JSX, props) 코드로 확인
- **방법**: 강사 코드 따라치기
- **프로젝트**: `react-helloworld`, `react-components` 두 앱

---

## 진행 과정

### Step 1. react-helloworld — 강사 코드 따라치기

`npm create vite@latest`로 프로젝트 생성 후 `App.jsx`, `HelloWorld.jsx`, `JSXBasic.jsx` 구현.

**App.jsx — useState 카운터 + 컴포넌트 재사용**

```jsx
import { useState } from 'react'
import HelloWorld from './components/HelloWorld'
import JSXBasic from './components/JSXBasic'

function App() {
  const [count, setCount] = useState(0)

  return (
    <>
      <section id="center">
        <h1>Hello World!!!</h1>
        <button
          type="button"
          className="counter"
          onClick={() => setCount((count) => count + 1)}
        >
          Count is {count}
        </button>
        <HelloWorld />
        <HelloWorld />   {/* 같은 컴포넌트를 두 번 — 재사용성 확인 */}
        <JSXBasic />
      </section>
    </>
  )
}
```

`onClick={() => setCount((count) => count + 1)}` — 이전 값 기반 업데이트 패턴. `setCount(count + 1)` 방식보다 안전하다.

**HelloWorld.jsx — 기본 컴포넌트 구조**

```jsx
function HelloWorld(){
    return (
        <div>
            <h1 className="counter" style={{ fontStyle:"italic", fontWeight:"bold" }}>
                Hello World!!!
            </h1>
        </div>
    );
}
export default HelloWorld;
```

인라인 스타일은 `style={{ }}` — 바깥 `{}`는 JSX 표현식, 안쪽 `{}`는 JavaScript 객체.

**JSXBasic.jsx — JSX 표현식 종합 실습**

```jsx
// ||, ??, ?. 비교
{str}, {str || 'default'}, {str ?? 'default'}, {data?.c ?? 'default'}
// str = undefined 일 때:
// str         → (아무것도 출력 안 됨)
// str || 'default'  → 'default' (falsy 전체)
// str ?? 'default'  → 'default' (null/undefined만)
// data?.c ?? 'default' → optional chaining으로 c가 없으면 'default'

// 조건부 렌더링
{add(data) >= 10 ? <h3>10 이상</h3> : <h2>10 미만</h2>}

// 단축 평가
{add(data) % 2 === 0 && '짝수'}

// 배열 렌더링 — key 필수
{arrys.map((num, index) => (
    <span key={index}>{index}번째 값 : {num} {','}</span>
))}

// props 전달
function Add2({a, b}) {   // 객체 구조 분해
    return <h2>{a} + {b} = {a+b}</h2>;
}
<Add2 a={1} b={2} />
```

---

### Step 2. react-components — 강사 코드 따라치기

Props 단방향 흐름 집중 실습.

**PropsCompoChild.jsx — props 수신 방식 2가지 비교**

```jsx
// 방식 1: props 객체 통째로
function DisplayPanel1(props){
    return (
        <div>
            title: {props.title} / count: {props.count} / color: {props.color}
        </div>
    );
}

// 방식 2: 구조 분해 할당 — 더 간결하고 가독성 좋음
function DisplayPanel2({title, count, color}){
    return (
        <div>
            title: {title} / count: {count} / color: {color}
        </div>
    );
}
```

**ControlPanel — 자식→부모 콜백 패턴**

```jsx
function ControlPanel({ onIncrease, onDecrease, onReset, onChangeValue, label = '카운터 조작' }) {
    return (
        <div>
            <button onClick={onIncrease}>+1</button>
            <button onClick={onDecrease}>-1</button>
            <button onClick={onReset}>초기화</button>
            <button onClick={() => onChangeValue('안녕')}>부모값변경</button>
        </div>
    );
}
// label = '카운터 조작' : props 기본값 설정
```

자식은 `onClick={onIncrease}` 형태로 부모가 내려준 함수를 그대로 연결. 직접 상태를 바꾸지 않는다.

**named export 사용**

```jsx
// 내보낼 대상이 여러 개
export { DisplayPanel1, DisplayPanel2, ControlPanel };

// 가져올 때
import { DisplayPanel1, DisplayPanel2, ControlPanel } from "./PropsCompoChild";
```

---

## 결과 / 산출물

- `react-helloworld`: HelloWorld 컴포넌트 재사용, JSX 표현식 종합 실습, useState 카운터
- `react-components`: props 수신 방식 2가지 비교, 자식→부모 콜백 패턴

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 컴포넌트 재사용성을 코드로 체감 | ControlPanel의 콜백 함수를 부모에서 구현하는 실습까지 진행 못 함 |
| `||` vs `??` vs `?.` 차이 코드로 비교 확인 | useState와 props drilling 연결 실습 미진행 |
| named export / default export 차이 확인 | |
| props 구조 분해 vs 통째로 수신 직접 비교 | |

---

## 다음에 이어서 할 것

- `ControlPanel`을 실제로 부모(`PropsCompo.jsx`)에서 연결해서 state 끌어올리기(lifting state up) 직접 구현
- `StateCompo.jsx`, `CRUDCompo.jsx` 실습 (교재 실습 예제 구조 참고)

---

*작성일: 2026-06-05 | 인공지능 사관학교 AI 보안 과정*
