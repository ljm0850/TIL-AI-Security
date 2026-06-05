# 2026-06-05 심화 정리 — React/Vue를 사용하면 Reflow·Repaint에 신경 안 써도 될까?

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-06-05_AI-Security-Course_Lecture_React.md`
> **작성 계기**: React 강의에서 Virtual DOM의 Diffing 개념을 들으면서, 이전에 "아직 찾아보지 않은 것"으로 미뤄뒀던 Reflow/Repaint 문제를 React/Vue 맥락에서 다시 생각하게 됐다.

---

## 왜 찾아봤는가

Web/CSS 강의 때 Reflow vs Repaint를 "아직 찾아보지 않은 것"에 남겨뒀었다. React의 Virtual DOM이 DOM 변경을 최소화한다는 설명을 들으면서, "그럼 React를 쓰면 Reflow/Repaint는 신경 안 써도 되는 건가?"라는 의문이 생겨 찾아봤다.

---

## 핵심 내용

### 결론부터

React/Vue는 Reflow/Repaint를 **줄여주는 안전망**을 제공하지만, 완전히 신경을 끊어도 되는 건 아니다.

---

### React/Vue가 해주는 것 — Virtual DOM

```
실제 DOM 변경  → reflow/repaint 발생 (비용 큼)
Virtual DOM 변경 → 메모리에서만 계산 (비용 없음)
```

상태(state)가 바뀌면 새 Virtual DOM 트리를 생성하고, 이전 Virtual DOM과 Diffing(비교)하여 실제로 바뀐 부분만 실제 DOM에 반영한다(Reconciliation). 결과적으로 reflow/repaint 횟수가 줄어든다.

---

### 그래도 신경 써야 하는 경우

**1. key 없는 리스트 렌더링 — Reconciliation 비효율**

```jsx
// 나쁜 예: key가 없으면 리스트 전체를 다시 렌더링
items.map(item => <Item item={item} />)

// 좋은 예: key로 변경된 항목만 추적
items.map(item => <Item key={item.id} item={item} />)
```

React가 어떤 항목이 바뀌었는지 추적하지 못하면 전체를 다시 그린다. key가 최적화의 핵심이다.

**2. 불필요한 리렌더링 → reflow/repaint 유발**

```jsx
// 좋은 예: React.memo로 props가 바뀌지 않으면 리렌더링 건너뜀
const HeavyChild = React.memo(() => <div>...</div>);
```

**3. 레이아웃 강제 읽기 — reflow 강제 발생**

```jsx
useEffect(() => {
    // offsetWidth 읽기 → reflow 강제 발생
    const width = el.current.offsetWidth;
    el.current.style.width = width + 10 + 'px'; // 다시 reflow
}, []);
```

DOM의 레이아웃 관련 속성(`offsetWidth`, `scrollTop` 등)을 읽으면 브라우저가 최신 레이아웃을 계산하기 위해 강제로 reflow를 발생시킨다.

**4. 인라인 스타일 남용 — 스타일 재계산**

```jsx
// 렌더링마다 새 객체 생성 → 스타일 재계산
<div style={{ color: 'red', fontSize: 16 }}>
```

렌더링마다 새 객체가 생성되므로 React는 이전과 다른 값으로 인식해 스타일을 재계산할 수 있다.

---

### 비교 정리

| 항목 | 순수 JS/jQuery | React/Vue |
|---|---|---|
| DOM 직접 조작 | 매번 reflow 가능 | Virtual DOM이 배치 처리 |
| 불필요한 업데이트 | 개발자가 직접 제어 | 자동 최소화 (but 실수 가능) |
| 리스트 렌더링 | 직접 최적화 필요 | key로 제어 |
| 대규모 DOM 변경 | 위험 | 비교적 안전 |

---

### 특히 주의해야 하는 상황

`key`, `React.memo`, `useCallback`, `useMemo` 등을 제대로 쓰지 않으면 **Virtual DOM 비용 + reflow 이중 부담**이 생길 수 있다. 애니메이션, 대용량 리스트, 실시간 데이터 상황에서는 여전히 의식적인 최적화가 필요하다.

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 | 아직 모르는 것 |
|---|---|
| Virtual DOM이 reflow/repaint를 줄이는 원리 | `useCallback`, `useMemo`의 구체적인 사용 기준 |
| key가 없으면 왜 리스트 전체를 다시 그리는지 |  |
| 레이아웃 속성 읽기가 강제 reflow를 발생시키는 이유 |  |
| `React.memo`의 역할 |  |

---

## 출처

- 강의 중 직접 조사 및 정리 (2026-06-05)

---

*작성일: 2026-06-05 | 인공지능 사관학교 AI 보안 과정*
