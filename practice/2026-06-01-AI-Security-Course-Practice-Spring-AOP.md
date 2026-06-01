# 2026-06-01 실습 — Spring AOP (프록시 패턴)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: 강의 자료 없이 실습 위주 진행

---

## 느낀 것 / 배운 것

### 📅 2026-06-01

> 오늘 AOP를 보면서, 이전 면접에서 "디버깅 어떻게 하나요?"라고 물어봤을 때 `console.log` 찍었다고 답했던 게 생각났다. 면접관은 단순히 디버깅 방법만이 아니라, 문제를 분석하고 구조적으로 접근하는 과정, "어디서 로그를 찍냐"보다 "어떤 구조로 로그를 관리하냐"를 물어본 것이였는데.

> XML에서 부가 기능을 설정하는 방식을 보면서, 소스코드를 건드리지 않고 외부에서 동작을 주입할 수 있다는 게 Spring이 강력한 이유인 걸 다시 체감했다. "다른 팀원의 코드에 손대지 않고 내 관심사를 끼워넣을 수 있다"는 발상이 신선했다.

> 프록시 패턴 → AOP의 흐름을 코드로 직접 따라가니, 프론트엔드에서 컴포넌트 재사용하던 감각과 비슷하다는 느낌이 들었다. 공통 관심사를 분리해서 재사용하는 것 — 방향은 같다.

---

## 실습 개요

- **목표**: 로깅이라는 공통 관심사를 핵심 비즈니스 로직에서 분리하는 AOP 구조 이해
- **방법**: 수동 프록시(상속 방식) → XML 기반 AOP 순서로 진화 과정을 코드로 추적

---

## 진행 과정

### Step 1. 로그를 직접 소스코드에 넣는 방식 (Before — 문제 확인)

`HelloService.sayHello()` 안에 로그 코드를 직접 작성하면, 모든 Service마다 같은 코드를 반복해야 한다. 단일 책임 원칙 위반.

```java
// HelloService.java — 주석 처리된 흔적이 진화 과정을 보여줌
// 1) 직접 코드 삽입
System.out.println(">>> Log : " + new java.util.Date());

// 2) 객체지향적 분리 (재사용은 되지만 여전히 소스코드에 침투)
HelloLog.log();
```

### Step 2. 프록시 패턴으로 분리 (수동 AOP)

`HelloService`를 상속한 `HelloServiceProxy`가 로그(`HelloLog.log()`)를 호출한 뒤 원본 메서드(`super.sayHello()`)를 실행. 핵심 로직에 손대지 않고 부가 기능 삽입.

```java
// HelloServiceProxy.java
public class HelloServiceProxy extends HelloService {
    @Override
    public String sayHello(String name) {
        HelloLog.log();               // Aspect, Advice — 부가 기능
        String result = super.sayHello(name);  // PointCut — 핵심 로직 호출
        return result;
    }
}
```

- **메서드 전후로 시간 계산**하면 성능 측정도 가능 → APM(Application Performance Monitoring)의 원리

### Step 3. XML 기반 AOP — 소스코드 완전 분리

`application-config.xml`에서 어느 메서드(PointCut)에 어떤 부가 기능(Advice)을 적용할지 선언. 소스코드를 전혀 건드리지 않아도 됨.

```xml
<!-- application-config.xml 핵심 구조 -->
<!-- bean 등록 → aop:config → aop:aspect → aop:before/after 등 -->
```

**AOP 핵심 용어 정리**

| 용어 | 역할 | 이 실습에서 해당하는 것 |
|---|---|---|
| Aspect | 공통 관심사 모듈 | `HelloLog` 클래스 |
| Advice | 실제 실행되는 부가 기능 | `HelloLog.log()` |
| PointCut | Advice를 적용할 지점 | `sayHello()` 메서드 |
| JoinPoint | PointCut이 될 수 있는 후보들 | 모든 메서드 호출 지점 |
| Weaving | Advice를 PointCut에 끼워넣는 과정 | XML 설정 or Spring 컨테이너 |

### Step 4. 전체 실행 흐름

```
HelloMain
  → Spring 컨테이너에서 HelloController 빈 획득
  → controller.hello("홍길동") 호출
  → HelloController → IHelloService.sayHello() 호출
  → (AOP 개입) HelloLog.log() 실행
  → HelloService.sayHello() 실행
  → 결과 반환
```

---

## 결과 / 산출물

- 로그 출력 → 핵심 로직 실행 → 결과 반환 흐름 확인
- `HelloService` 소스코드 무수정으로 로깅 동작 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 수동 프록시 → XML AOP 진화 흐름을 코드로 직접 추적 |
| AOP 핵심 용어(Aspect, Advice, PointCut)를 코드와 1:1 매핑 |

---
*작성일: 2026-06-01 | 인공지능 사관학교 AI 보안 과정*
