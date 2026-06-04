# 2026-06-04 실습 — JavaScript

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사 코드 따라치기 + 일부 변형)
> **관련 강의 파일**: `../lectures/2026-06-04_AI-Security-Course_Lecture_JavaScript.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-04

- 함수를 매개변수로 넘기는 패턴(`forEach`, `map`, 콜백)이 익숙하지 않아서 아직도 신기하다. 과거에 반복문으로 풀었던 것들이 이렇게 간결해질 수 있다는 걸 다시 확인했다.
- 강의가 "JavaScript만의 특징" 위주로 진행돼서 전에 알던 것들이 리마인드되는 느낌이었다. 그냥 문법을 나열하는 게 아니라 왜 이렇게 설계됐는지가 보였다.

---

## 실습 개요

- **목표**: JavaScript 핵심 개념(변수/타입/함수/객체/클로저/Ajax) 코드로 확인, Vue.js 기초 체험
- **방법**: 강사 코드 따라치기 후 일부 변형 테스트

---

## 진행 과정

### Step 1. 변수와 타입 (js01.html)

`var` / `let` / `const` 차이와 타입 강제변환 동작 확인.

```javascript
// 타입 추론 — 값이 바뀌면 타입도 바뀜
var a = 'abcd';
console.log(typeof(a));  // string
var a = 12;
console.log(typeof(a));  // number

// 타입 강제변환 주의
console.log(1 + '2');    // '12'
console.log('1' * 2);   // 2
console.log('1'*1 + 2); // 3

// 동등(==) vs 일치(===)
console.log(123 == '123');   // true
console.log(123 === '123');  // false
```

배열 순회 4가지 방법 비교:

```javascript
const ary = [100, 200, 300, 400, 500];

for (let i = 0; i < ary.length; i++) { ... }  // 전통적 방식
for (const i in ary) { ... }                   // key(인덱스) 순회
for (const v of ary) { ... }                   // 값 직접 순회
ary.forEach((v) => console.log(v));            // 콜백 방식
```

### Step 2. 함수 (js02.html)

호이스팅, 기본값 매개변수, 나머지값 매개변수, 클로저 확인.

```javascript
// 호이스팅 — 선언 전에 호출해도 동작
console.log(add2(1));   // NaN이 아니라 1이 나옴 (b=0 기본값)
function add2(a, b = 0) { return a + b; }

// 나머지값 매개변수
function add3(...a) {
    let sum = 0;
    a.forEach((n) => sum += n);
    return sum;
}
console.log(add3(1,2,3,4,5)); // 15

// 클로저 — cnt를 캡슐화
function count2() {
    let cnt = 0;
    return function() { return cnt++; }
}
const cnt1 = count2();
console.log(cnt1()); // 0
console.log(cnt1()); // 1  ← 상태 유지됨
```

### Step 3. 객체 (js03.html)

객체 리터럴, `this` 동작, 프로토타입 상속 확인.

```javascript
const calc = {
    a: 1, b: 2,
    add: function() { return this.a + this.b; }
};

// const로 선언해도 내부 프로퍼티는 변경 가능 (주소값만 const)
calc.a = 3;
console.log(calc.add()); // 5

// 동적 메서드 추가
calc.multi = function() { return this.a * this.b; }
console.log(calc.multi()); // 6
```

`this` 혼란스러운 케이스:

```javascript
var name = "1가";
function test() {
    var name = "2나";
    console.log(name);        // "2나" — 지역변수
    console.log(this.name);   // "1가" — window.name
}

// bind(this)로 this 고정
var test2 = {
    name: "3다",
    test: function() {
        function a() {
            console.log("a 호출", this.name); // bind 없으면 window.name
        }
        a();
    }.bind(this)
}
```

### Step 4. 클래스 (js04.js)

```javascript
class Polygon {
    constructor(height, width) {
        this.height = height;
        this.width = width;
    }
    get area() { return this.height * this.width; }
    print() { console.log(this.area); }
}

// 강사 코드: 정사각형 클래스 만들기 실습
class Square extends Polygon {
    constructor(length) {
        super(length, length);
    }
}
const square = new Square(2);
console.log(square.area); // 4

// 동적 메서드 추가 (인스턴스에만 추가)
const poly = new Polygon(10, 20);
poly.print2 = function() { console.log(this.area); }
poly.print2(); // 200
```

### Step 5. Ajax 비동기 (js05.html)

비동기 실행 순서 직접 확인.

```javascript
// 비동기 순서 체감
let result = '';
$.get('msg.json', function(data) {
    result = data;
    console.log('1', result);  // 나중에 출력
});
console.log('2', result);      // 먼저 출력 — result는 아직 ''
```

jQuery Promise → axios → fetch 순서로 같은 동작을 4가지 방법으로 구현. 어떤 방법이든 결과적으로 `document.getElementById('msg').innerText = data.text`로 DOM에 반영.

### Step 6. Vue.js (js06_vue.html)

```html
<div id="app">
    <div>{{ message }}</div>
    <input v-model="message">
    <li v-for="todo in todos">{{todo.text}}</li>
</div>
```

```javascript
const app = new Vue({
    el: '#app',
    data: {
        message: '안녕하세요. Vue!',
        todos: [
            { text: 'JavaScript 배우기' },
            { text: 'Vue 배우기' },
            { text: '무언가 멋진 것을 만들기' }
        ],
    }
});
```

입력창에 타이핑하면 `{{ message }}`가 즉시 바뀐다. `v-model`이 뷰↔뷰모델 양방향 바인딩을 처리한다.

---

## 결과 / 산출물

- `js01.html` ~ `js06_vue.html` 코드 실습 완료
- `js04.js`: `Square extends Polygon` 클래스 직접 작성

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 비동기 실행 순서를 콘솔로 직접 확인 | `this` 동작 규칙이 아직 직관적이지 않음 |
| 클로저의 상태 유지 동작을 코드로 확인 | `bind(this)` 사용 경험 |
| Vue.js 양방향 바인딩 체험 |  |
| 배열 고차함수 4가지 방법 비교 확인 | |

---

*작성일: 2026-06-04 | 인공지능 사관학교 AI 보안 과정*
