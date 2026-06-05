# 2026-W23 주간 회고

> **과정**: 인공지능 사관학교 AI 보안 과정
> **기간**: 2026-06-02 ~ 2026-06-05

---

## 이번 주 배운 것

- **JavaScript의 언어적 특성** — `var`/`let`/`const` 스코프 차이, 타입 강제변환(`==` vs `===`), 호이스팅, 실행 컨텍스트까지 javascript deepdive 책에서 배웠던 내용이 떠오르며 하나의 흐름으로 연결됐다. 특히 클로저는 "이런 것도 있구나" 수준에서 "바이브코딩 시대에 사이드 이펙트를 막는 중요한 패턴"으로 다시 보이기 시작했다.
- **비동기 처리 진화** — 콜백 → Promise → async/await 흐름을 코드로 직접 비교하면서, 왜 각 방식이 등장했는지 맥락이 생겼다.
- **프론트엔드 보안 관점** — JS 소스코드가 브라우저에 그대로 노출된다는 점에서 출발해서, 가격 조작·Race Condition·IDOR·토큰 재사용까지 직접 찾아봤다. "Never Trust the Client"가 단순한 원칙이 아니라 구체적인 공격 시나리오와 연결된다는 걸 이해했다.

---

## 이번 주 만든 것 / 기여한 것

- `react-helloworld`: HelloWorld 컴포넌트 재사용, JSX 표현식 종합 실습, useState 카운터
- `react-components`: props 수신 방식 2가지 비교(`props` 통째 vs 구조 분해), 자식→부모 콜백 패턴
- deep-dive 2개 자체 작성: 프론트엔드 JS 조작 공격·방어, React/Vue 사용 시 Reflow·Repaint

---

## 아직 부족한 것

- "찾아보지 않은 것" 목록이 계속 쌓이고 있다. 강의를 따라가는 속도는 괜찮은데, 각 주제를 깊이 파고드는 시간이 부족하다. Vite 내부 동작, Tree Shaking, React Reconciliation 알고리즘, `useEffect`/`useCallback`/`useMemo` 등 당장 쓰지 않더라도 원리 수준의 이해가 필요한 것들이 밀려 있다.
- React 실습에서 `ControlPanel`의 state 끌어올리기(lifting state up)를 직접 완성하지 못했다. 다음 주 실습 이어서 처리 필요.

---

## 다음 주 목표

- 쌓인 "찾아보지 않은 것" 중 React 관련 항목(`useEffect`, props drilling 해결 패턴) 최소 2개 deep-dive 작성
- 강의 진도 따라가면서 새로 나오는 개념은 당일 정리 유지

---

*작성일: 2026-06-05 | 인공지능 사관학교 AI 보안 과정*
