# 2026-06-08 실습 — React Hooks

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-08_AI-Security-Course_Lecture_React-Hooks.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-08

- `setCount(prev => prev + 1)` 패턴을 이전 실습부터 계속 써왔는데, 오늘 `useEffect` + `setState` 무한루프 설명을 듣고 나서야 왜 이렇게 쓰는지 더 명확히 이해됐다.
- `StateCompo`는 지난 주에 강사 코드를 따라 친 것에서 불변 업데이트 패턴(`spread`, `filter`, `prev =>`)을 그대로 적용한 코드다. 강의 흐름에서 나온 구조.

---

## 실습 개요

- **목표**: React Hooks(`useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, Custom Hook) 동작 방식을 코드로 확인
- **방법**: 강의 예제 코드를 직접 작성하고, `StateCompo`는 기존 코드에서 불변 업데이트 패턴 적용 확인

---

## 진행 과정

### Step 1. StateCompo — useState 복습 (지난 실습 이어서)

강사 코드 기반으로 작성. 불변 업데이트 패턴 중심.

```jsx
import { useState } from 'react';

function StateCompo() {
    const [count, setCount] = useState(0);
    const [item, setItem] = useState({ name: '노트북', price: 1_200_000 });
    const [cart, setCart] = useState([]);

    const increment = () => setCount((prev) => prev + 1);
    const decrement = () => setCount((prev) => Math.max(0, prev - 1));

    // 불변형 프로그래밍(spread 문법 사용) - 추가, 삭제, 수정
    const discount = () => setItem((prev) => ({ ...prev, price: prev.price * 0.9 }));
    const addToCart = () => setCart((prev) => [...prev, { ...item, qty: count, id: Date.now() }]);
    const removeItem = (id) => setCart((prev) => prev.filter((c) => c.id !== id));

    return (
        <div>
            <h2>{item.name} — {item.price.toLocaleString()}원</h2>
            <button onClick={decrement}>-</button>
            <span> {count}개 </span>
            <button onClick={increment}>+</button>
            <button onClick={discount}>10% 할인</button>
            <button onClick={addToCart}>장바구니 추가</button>
            {cart.map((c) => (
                <p key={c.id}>
                    {c.name} x {c.qty}개, 가격: {c.price.toLocaleString()}원 x {c.qty} = {(c.price * c.qty).toLocaleString()}원
                    <button onClick={() => removeItem(c.id)}>삭제</button>
                </p>
            ))}
        </div>
    );
}
```

**포인트**
- `setCount((prev) => prev + 1)` — 현재 상태를 기반으로 업데이트 (비동기 배치 업데이트 안전)
- `setItem((prev) => ({ ...prev, price: ... }))` — spread로 객체 불변 업데이트
- `setCart((prev) => [...prev, { ...item }])` — spread로 배열 불변 추가
- `filter`로 불변 삭제

### Step 2. UseEffectCompo — 의존성 배열 4가지 패턴 실습

강의 예제 코드 그대로 작성 후 콘솔에서 실행 흐름 확인.

```jsx
// 1. 마운트 시 1회 실행
useEffect(() => {
    console.log('마운트될 때 딱 1번 실행');
    return () => console.log('언마운트될 때 정리(cleanup)');
}, []);

// 2. 특정 state 변경 시 실행
useEffect(() => {
    console.log('count 변경됨:', count);
    return () => console.log('이전 count cleanup:', count);
}, [count]);

// 3. 의존성 배열 없음 → 렌더링마다 실행
useEffect(() => {
    console.log('렌더링마다 실행');
    return () => console.log('다음 렌더 전 cleanup');
}); // 배열 자체 생략

// 4. 타이머 cleanup
useEffect(() => {
    const id = setInterval(() => setSec((s) => s + 1), 1000);
    return () => clearInterval(id); // cleanup: 언마운트 시 타이머 제거
}, []);
```

*작성일: 2026-06-08 | 인공지능 사관학교 AI 보안 과정*
