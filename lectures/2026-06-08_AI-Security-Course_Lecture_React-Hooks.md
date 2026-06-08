# 2026-06-08 강의 정리 — ReactJS Hooks

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: React Hooks (useEffect, useRef, useMemo, useCallback, Custom Hook)
> **관련 실습 파일**: `../practice/2026-06-08_AI-Security-Course_Practice_React-Hooks.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-08

- Hook은 클래스 컴포넌트 없이도 함수형 컴포넌트에서 state와 lifecycle을 쓸 수 있게 해주는 API다. 클래스 기반 코드의 복잡성을 줄이기 위해 React 16.8에 도입됐다.
- `useEffect`의 의존성 배열을 이해하는 게 핵심이다. `[]`이면 마운트 1회, `[value]`면 해당 값이 바뀔 때마다, 생략하면 렌더링마다 실행된다. 생략 + setState 조합은 무한루프이므로 조심해야 한다.
- `useRef`는 "렌더링에 영향 주지 않는 변수"가 필요할 때 쓴다. DOM 직접 접근(포커스, 스크롤)도 이걸로 처리한다. `useState`로 잘못 쓰면 불필요한 리렌더링이 발생한다.
- `useMemo`는 값을, `useCallback`은 함수 참조를 캐싱한다. 의존성이 안 바뀌면 재연산하지 않는다. 이게 렌더링 최적화의 핵심 도구다.
- Custom Hook은 `use`로 시작하는 이름을 가진 함수이고, 같은 훅을 쓰는 컴포넌트들은 상태를 완전히 독립적으로 유지한다.
- 상태관리 라이브러리는 현재 **Zustand**가 대세다. Context API, Redux보다 가볍고 사용법이 단순하다.

---

## 1. React Hooks 개요

### 핵심 개념

- Hook은 React 16.8에서 도입된 함수
- 클래스 컴포넌트 없이 함수형 컴포넌트에서 state와 lifecycle 기능을 사용할 수 있게 해주는 API
- 함수형 컴포넌트의 Life Cycle별 트리거로 동작하여 Side Effect 처리를 가능하게 함

### 상세 내용

**Side Effect (사이드 이펙트)**
- 리액트에서 컴포넌트와 외부 사이에 일어나는 비동기 작업 (API 통신, DOM 수동 수정, 브라우저 API)
- 렌더링을 먼저 하고 비동기로 데이터를 받아오는 방식으로 효율적 실행 가능

**Hook 사용 규칙**
- 함수형 컴포넌트에서만 사용 가능 (일반 JS 함수 불가)
- 최상위 레벨에서만 호출 가능 (조건문/반복문 내부 불가)
- `use`로 시작하는 네이밍 컨벤션

---

## 2. useEffect

### 핵심 개념

- 컴포넌트가 렌더링된 이후 실행되는 사이드 이펙트 처리 Hook
- API 호출, 이벤트 구독, 타이머, DOM 직접 조작 등을 처리

### 상세 내용

**기본 형식**
```js
useEffect(callback[, dependencies]);
useEffect(() => { 사이드 이펙트; return () => { 클린업 함수 }; }, [의존성 배열]);
```

**라이프 사이클별 동작**

| 상황 | 형식 |
|---|---|
| 렌더링마다 매번 실행 | `useEffect(() => { ... })` |
| 마운트 시 1회 실행 | `useEffect(() => { ... }, [])` |
| 특정 값 변화에 반응 | `useEffect(() => { ... }, [value])` |
| 언마운트 시 cleanup | `return () => { 해제코드 }` |

**주의: 무한루프**
- 의존성 배열 생략 + `setState` 조합 → 렌더링 → effect 실행 → setState → 렌더링 ... 무한루프
- 의존성 배열 생략은 신중하게

**Cleanup 함수**
- 컴포넌트 언마운트 또는 effect 재실행 전에 호출
- 이벤트 구독 해제, 타이머 제거 등 리소스 정리 목적

---

## 3. useRef

### 핵심 개념

- 함수형 컴포넌트에서 변경 가능한 참조 객체를 관리하는 Hook
- 값이 바뀌어도 **리렌더링이 발생하지 않음**
- `.current` 속성으로 값에 접근

### 상세 내용

**기본 형식**
```js
import { useRef } from 'react';
const ref = useRef(초기값);
ref.current = 1;
```

**주요 사용 패턴**

| 패턴 | 설명 |
|---|---|
| DOM 직접 접근 | `ref={inputRef}` → `inputRef.current.focus()` |
| 렌더링 없이 값 유지 | 리렌더링과 무관한 변수 저장 |
| 이전 값 기억 | `prevRef.current = count` (useEffect와 함께) |

- 렌더링 때마다 내부 변수는 초기화되지만 `useRef` 값은 언마운트 전까지 유지됨

---

## 4. useMemo & useCallback

### 핵심 개념

- **메모이제이션**: 이전 계산 결과를 저장해두고, 동일한 입력이 들어올 때 재계산 없이 저장된 값 반환
- `useMemo`: 연산 결과값 캐싱
- `useCallback`: 함수 참조 캐싱

### 상세 내용

**기본 형식**
```js
import { useMemo, useCallback } from 'react';
useMemo(() => { return 계산처리; }, [의존성]);
useCallback(() => { 실행처리; }, [의존성]);
```

**비교**

| Hook | 캐싱 대상 | 리렌더링 | 주 사용처 |
|---|---|---|---|
| `useMemo` | 연산 결과값 | 없음 (캐시 반환) | 고비용 계산, 필터링/정렬, 파생 데이터 |
| `useCallback` | 함수 참조 | 없음 (캐시 반환) | React.memo 자식에 전달하는 콜백 함수 |
| `React.memo` | 컴포넌트 렌더 결과 | Props 변경 시만 | 같은 Props면 리렌더링 건너뜀 |

---

## 5. Custom Hook

### 핵심 개념

- 내장 훅을 조합하거나 로직을 재사용하기 위해 직접 만든 Hook
- `use`로 시작하는 이름 필수
- 같은 Custom Hook을 사용하는 각 컴포넌트의 상태는 **완전히 독립적**

### 상세 내용

```js
// 예: useCounter 커스텀 훅
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(initialValue);
  return { count, increment, decrement, reset };
}
```

- 같은 `useCounter`를 사용하는 `CounterA(초기값 0)`, `CounterB(초기값 100)`은 상태를 서로 공유하지 않음

---

## 6. React Hooks Lifecycle 전체 흐름

- **Render phase**: `function(){}` → `useMemo()` → `return()`  — 순수 함수, 사이드 이펙트 없음
- **Commit phase**: React가 DOM과 refs를 업데이트 → `useEffect()`, `useLayoutEffect()` 실행
- **Cleanup phase**: 컴포넌트 제거 전 실행, 메모리 누수 방지



*작성일: 2026-06-08 | 인공지능 사관학교 AI 보안 과정*
