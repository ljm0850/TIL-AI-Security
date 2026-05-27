# 2026-05-27 심화 정리 — Java switch문의 String 비교 방식

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-05-27_AI-Security-Course_Lecture_Java-Basics.md`
> **작성 계기**: 강의 중 `switch`의 `case`에 String을 쓸 때 내부적으로 `==`인지 `equals()`인지 궁금해서 찾아봄

---

## 왜 찾아봤는가

`switch`문에서 String을 case 값으로 쓸 때, 내부적으로 어떻게 비교하는지 궁금했다.
`==`는 주소 비교, `equals()`는 값 비교라는 걸 알고 있었기 때문에, `switch`가 둘 중 어느 쪽인지가 중요했다.
`new String("Gold")`처럼 새로 생성한 객체를 넣어도 제대로 동작하는 걸 보고, `==`는 아닐 것 같다고 생각했다.

---

## 핵심 내용

**결론: switch는 `==`도 아니고, 단순 `equals()`만도 아니다. `hashCode()` → `equals()` 2단계 비교다.**

### 내부 동작 방식

Java 컴파일러는 `switch(String)`을 두 단계로 변환한다.

**1단계 — `hashCode()`로 후보 좁히기**
```java
// 컴파일러가 내부적으로 이렇게 변환
switch (medal.hashCode()) {
    case 2180746:  // "Gold".hashCode()
        ...
    case 2390505:  // "Silver".hashCode()
        ...
}
```
해시코드가 같은 case만 후보로 좁힌다.

**2단계 — `equals()`로 최종 확인**
```java
// 해시 충돌 가능성 때문에 2차로 검증
if (medal.equals("Gold")) { ... }
```

### 비교 방식 정리

| 비교 방식 | switch 내부 사용 여부 |
|---|---|
| `==` (주소 비교) | ❌ 사용 안 함 |
| `hashCode()` | ✅ 1차 필터링 |
| `equals()` | ✅ 2차 최종 확인 |

### `==`와 `equals()`의 차이 확인

```java
String a = new String("Gold");
String b = new String("Gold");

System.out.println(a == b);       // false (주소가 다름)
System.out.println(a.equals(b));  // true  (값이 같음)
```

switch가 `==`를 쓴다면 `new String("Gold")`를 넣었을 때 매칭 실패 버그가 생긴다.
Java가 `equals()` 기반으로 설계한 이유다.

### Java 버전별 String switch 지원

- **Java 7 이전**: int, char, enum 등 primitive 타입만 가능
- **Java 7 이상**: String을 switch 조건으로 사용 가능

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 | 아직 모르는 것 |
|---|---|
| switch가 `hashCode()` + `equals()` 2단계로 동작한다는 것 | 해시 충돌이 실제로 발생할 가능성이 얼마나 되는지 |
| `==`가 주소 비교라 new String()에서 버그가 생길 수 있다는 것 |  |
| Java 7부터 String case 사용이 가능해졌다는 것 |  |

---

## 출처

- 강의 중 직접 궁금증 발생 후 정리
- Java Language Specification (JLS) — switch 관련 섹션 참고

---
*작성일: 2026-05-27 | 인공지능 사관학교 AI 보안 과정*
