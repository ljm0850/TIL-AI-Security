# 2026-06-04 강의 정리 — JavaScript

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: JavaScript 기초 ~ Ajax / Vue.js 입문
> **관련 실습 파일**: `../practice/2026-06-04_AI-Security-Course_Practice_JavaScript.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-04

- JS의 소스코드는 브라우저에 그대로 노출된다는 점이 보안 관점에서 핵심이다. 프론트엔드 검증은 UX용이고, 실제 보안은 서버에서 해야 한다는 원칙이 여기서 나온다.
- 클로저를 예전엔 "이런 것도 있구나" 하고 넘겼는데, AI가 코드를 생성하는 시대에 전역변수 사이드 이펙트가 훨씬 많아질 것 같다는 생각이 들었다. 클로저, class의 #(private)가 다시 중요해질 수 있겠다.
- 함수를 매개변수로 넘기는 패턴(콜백)이 익숙하지 않았는데, 이게 함수형 프로그래밍의 핵심이라는 걸 강의를 통해 다시 인식했다. 과거 코드들이 생각났다.
- Vue.js에서 `v-model`로 입력값이 화면에 즉시 반영되는 걸 보면서, MVVM 패턴이 단방향이 아니라 뷰↔뷰모델 양방향 바인딩이라는 점을 다시 확인했다.

---

## 1. JavaScript 개요

### 핵심 개념

- 1996년 Netscape에서 브라우저 반응형 문서 지원을 위해 도입 (초기명: LiveScript)
- C언어 기반. **일급시민(First Class Citizen)은 함수(function)와 객체(object)**
- 객체 지향형(OOP) + 함수 지향형(FOP) 동시 지원
- ECMAScript(ECMA 표준)를 따르며, ES6(2015) 이후 1년 주기로 표준 발표

### 상세 내용

**JS의 구조적 한계 — 소스코드 노출 문제**

브라우저에 소스코드를 그대로 전송해야 하므로 로직을 숨길 수 없다. 이를 해결하기 위한 대안으로 **Wasm(WebAssembly)** 이 존재한다. Java의 `.class` 파일처럼 컴파일된 바이너리 형태로 브라우저에 전달되어 소스를 숨길 수 있다.

**JS 특징 요약**

| 특징 | 설명 |
|---|---|
| 인터프리터 / JIT | 브라우저 엔진이 실행 (V8 등) |
| Dynamic Type | 변수 타입을 선언하지 않음, 값으로 결정 |
| 비동기 처리 | 코드 실행이 끝날 때까지 기다리지 않음 |
| GC | 가비지 컬렉터가 메모리 관리 |
| OS 독립 | 어떤 OS에서도 브라우저만 있으면 실행 가능 |

---

## 2. 변수와 자료형

### 핵심 개념

- `var`: 전역/지역변수, 재선언 가능, 호이스팅 대상
- `let`: 블록 지역변수 (ES6), 재선언 불가
- `const`: 상수 (ES6), 재선언 불가, 재할당 불가
- 타입은 값(리터럴)으로 결정됨 — **loose typed**

### 상세 내용

```javascript
var a = 'abcd';     // 재선언 가능 (권장하지 않음)
let b = 'abcd';     // 재선언 불가, 재할당 가능
const c = 'abcd';   // 재선언·재할당 불가

// 타입 강제변환 주의
console.log(1 + '2');   // '12' (문자열 연결)
console.log('1' * 2);   // 2   (* 는 숫자 연산자라 변환)
console.log('1'*1 + 2); // 3

// 동등 vs 일치
console.log(123 == '123');   // true  (값만 비교, 타입 변환 발생)
console.log(123 === '123');  // false (값 + 타입 모두 비교)
```

**기본 자료형 (Primitive Type)**

| 타입 | 설명 |
|---|---|
| Boolean | true / false. 0도 false |
| Null | 빈 값으로 초기화된 변수 |
| Undefined | 초기화하지 않은 변수 |
| Number | 정수(8/10/16진수), 실수 |
| String | `""`, `''` 로 선언 |

**참조형 (Reference Type)**: 객체 `{}`, 배열 `[]`, 함수 `function(){}`

---

## 3. 함수 (Function)

### 핵심 개념

- 함수도 값이다 — 변수에 할당 가능, 매개변수로 전달 가능 (일급시민)
- 함수선언식은 호이스팅 대상, 함수표현식은 아님
- 클로저(Closure)

### 상세 내용

**함수 선언 방법**

```javascript
// 1. 함수선언식 (호이스팅 O)
function add(a, b) { return a + b; }

// 2. 함수표현식 (호이스팅 X)
var add = function(a, b) { return a + b; }

// 3. 화살표 함수 (ES6) — 익명함수의 단축표기법, this 바인딩 없음
const add = (a, b) => a + b;

// 4. 즉시 실행 함수 — 실행 후 사라짐, 전역변수 충돌 방지
(function(a, b) { console.log(a + b); })(1, 2);
```

**기본값 매개변수 / 나머지 매개변수**

```javascript
function add2(a, b = 0) { return a + b; }    // 기본값
function add3(...a) {                          // 나머지값 (rest parameter)
    let sum = 0;
    a.forEach((n) => sum += n);
    return sum;
}
```

**클로저 (Closure) — 사이드 이펙트 제거**

외부함수의 실행이 끝나도 내부함수가 외부함수의 변수를 계속 참조하는 구조. 전역변수 없이 상태를 유지할 수 있다.

```javascript
// 전역변수 방식 — 외부에서 cnt를 건드릴 수 있음 (side effect 발생 가능)
let cnt = 0;
function count1() { return cnt++; }

// 클로저 방식 — cnt를 캡슐화, 외부에서 접근 불가
function count2() {
    let cnt = 0;
    return function() { return cnt++; }
}
const cnt1 = count2();
cnt1(); // 0
cnt1(); // 1
```

---

## 4. 객체 (Object)

### 핵심 개념

- JS는 **프로토타입(Prototype) 기반** 언어
- 모든 객체는 프로토타입 체인(Prototype Chain)을 통해 상속 구현
- ES6 이후 `class`, `extends` 키워드 사용 가능
- `this`는 호출 위치에 따라 달라짐

### 상세 내용

**객체 생성 방법 3가지**

```javascript
// 1. 리터럴
const calc = { a: 1, b: 2, add: function() { return this.a + this.b; } };

// 2. 생성자 함수
function Calc(a, b) { this.a = a; this.b = b; }
Calc.prototype.add = function() { return this.a + this.b; }
const c = new Calc(5, 6);

// 3. ES6 class
class Polygon {
    constructor(height, width) {
        this.height = height;
        this.width = width;
    }
    get area() { return this.height * this.width; }
}
class Square extends Polygon {
    constructor(length) { super(length, length); }
}
```

**`this` 동작 규칙**

| 컨텍스트 | this가 가리키는 것 |
|---|---|
| 전역 컨텍스트 | `window` (브라우저) |
| 일반 함수 | 호출한 객체 (없으면 `window`) |
| 화살표 함수 | 상위 스코프의 `this` (lexical scope) |
| 이벤트 핸들러 | 이벤트가 발생한 DOM 요소 |

`bind(this)`로 명시적으로 `this`를 고정할 수 있다.

**프로토타입 기반 상속**

```javascript
var a = { property_a: "A", method_a: function() { console.log("A!"); } };
var b = { __proto__: a, property_b: "B" };
var c = { __proto__: b, property_c: "C" };

console.log(c.property_a); // "A" — 프로토타입 체인 타고 올라감
```

---

## 5. 실행 컨텍스트와 스코프

### 핵심 개념

- **실행 컨텍스트**: JS 엔진이 코드를 실행하기 위해 관리하는 환경 (변수, 함수, 스코프, `this` 포함)
- **호이스팅(Hoisting)**: `var` 변수와 함수선언식을 해당 스코프 최상위로 미리 올려두는 동작
- **스코프 체인**: 하위 → 상위 → 전역 순으로 변수를 검색

### 상세 내용

```javascript
// 호이스팅 예시 — add2가 선언 전에 호출 가능한 이유
console.log(add2(1));   // NaN (b=undefined) — 호이스팅으로 함수가 이미 메모리에 있음
function add2(a, b = 0) { return a + b; }
```

**JS 런타임 구조**

- **Call Stack**: 함수 실행 컨텍스트가 쌓이는 곳
- **Heap**: 객체가 저장되는 메모리
- **Web APIs**: Timer, Ajax, DOM Events 등 비동기 작업
- **Callback Queue**: 완료된 비동기 콜백이 대기하는 곳
- **Event Loop**: Call Stack이 비면 Callback Queue에서 꺼내 실행

---

## 6. 배열 고차함수

### 핵심 개념

함수를 매개변수로 받는 배열 메서드들. 함수형 프로그래밍의 핵심 패턴.

```javascript
const ary = [1, 2, 3, 4, 5];

ary.forEach((v) => console.log(v));           // 각 요소 순회
ary.map((n) => n * 2);                        // 새 배열 반환 [2,4,6,8,10]
ary.filter((x) => x % 2 === 0);              // 짝수만 [2,4]
ary.reduce((tot, x) => tot + x);             // 누적 합산 15
ary.some((x) => x < 3);                      // 하나라도 만족 → true
ary.every((x) => x < 3);                     // 모두 만족 → false

// for...in : 객체의 key(인덱스)를 순회
// for...of : 요소값을 직접 순회 (ES6)
```

---

## 7. Ajax와 비동기 처리

### 핵심 개념

- **Ajax**: 페이지 새로고침 없이 서버와 비동기 통신
- **Promise**: 콜백 지옥(Callback Hell) 해결을 위한 비동기 처리 패턴
- **async/await**: Promise를 동기식처럼 작성할 수 있는 ES8 문법

### 상세 내용

**Ajax 구현 방법 4가지 (js05.html 기준)**

```javascript
// 1. jQuery (콜백)
$.get('msg.json', function(data) { result = data; });

// 2. jQuery + Promise 체이닝
$.get('msg.json').done(function(data) {
    document.getElementById('msg').innerText = data.text;
}).fail(function(data) { console.log('오류', data); });

// 3. axios
axios.get('msg.json').then(function(data) {
    document.getElementById('msg').innerText = data.data.text;
});

// 4. fetch (ECMA 표준)
fetch('msg.json', { method: 'GET' })
    .then((res) => { if (res.ok) return res.json(); })
    .then((data) => { document.getElementById('msg').innerText = data.text; });
```

**비동기 순서 주의**

```javascript
let result = '';
$.get('msg.json', function(data) { result = data; console.log('1', result); });
console.log('2', result);  // 2가 먼저 출력됨 — result는 아직 ''
```

비동기 특성상 `$.get` 콜백이 완료되기 전에 다음 줄이 실행된다.

**Promise 상태**

| 상태 | 설명 |
|---|---|
| pending | 비동기 작업 진행 중 |
| fulfilled | 완료 (resolve 호출) |
| rejected | 실패 (reject 호출) |

---

## 8. BOM / DOM

### 핵심 개념

- **BOM**: 브라우저 기능을 객체로 제공 (`window`, `location`, `history` 등), 표준 없음
- **DOM**: HTML 문서를 트리 구조로 표현, JS로 요소 선택·생성·수정·삭제 가능

### 상세 내용

```javascript
// DOM 요소 선택
document.getElementById("msg")
document.querySelector(".class")
document.querySelectorAll("li")

// DOM 조작
el.innerText = "변경";
el.style.color = "blue";
el.style.display = "none";

// 이벤트
el.addEventListener('click', function(e) { ... });
e.stopPropagation();   // 이벤트 버블링 차단
e.preventDefault();    // 기본 동작 취소
```

**이벤트 버블링**: 자식 요소 이벤트가 부모로 전파되는 현상
**이벤트 캡처**: 부모 이벤트가 자식으로 전파되는 현상 (기본값 false)

---

## 9. Vue.js 입문

### 핵심 개념

- 2014년 Evan You가 AngularJS 참고하여 제작한 경량 JS 프레임워크
- **MVVM 패턴**: View ↔ ViewModel (양방향 바인딩) ↔ Model
- 가상 DOM(Virtual DOM) 사용으로 성능 향상
- SPA(Single Page Application) 개발에 적합

### 상세 내용

```html
<div id="app">
    <div>{{ message }}</div>
    <input v-model="message">  <!-- 양방향 바인딩 -->
    <li v-for="todo in todos">{{todo.text}}</li>
</div>

<script>
const app = new Vue({
    el: '#app',
    data: {
        message: '안녕하세요. Vue!',
        todos: [
            { text: 'JavaScript 배우기' },
            { text: 'Vue 배우기' },
        ],
    }
});
</script>
```

입력창에 타이핑하는 즉시 `{{ message }}`가 실시간으로 바뀐다. DOM을 직접 건드리지 않아도 된다.

---

## 아직 찾아보지 않은 것

- Wasm(WebAssembly)의 실제 활용 사례 및 보안 관점 장단점

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| 프론트엔드 JS 조작 공격과 방어 전략 | 2026-06-04 | 가격 조작·Race Condition·권한 우회·API 직접 호출 등 클라이언트 신뢰의 위험성과 서버 측 방어 원칙 | `../deep-dive/2026-06-04_Frontend-Security-JS-Manipulation.md` |

---

*작성일: 2026-06-04 | 인공지능 사관학교 AI 보안 과정*
