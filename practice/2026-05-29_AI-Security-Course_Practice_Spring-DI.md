# 2026-05-29 실습 — Spring DI (의존성 주입)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-05-29_AI-Security-Course_Lecture_Spring-Basics.md` (작성 예정)

---

## 느낀 것 / 배운 것

### 📅 2026-05-29

- **DI가 왜 필요한지** di01 예제에서 눈으로 확인했다. `new MessageKo()`, `new MessageEn()`처럼 직접 객체를 만드는 순간 그 클래스에 의존성이 생긴다. 언어를 바꾸려면 소스 코드를 직접 찾아서 수정해야 한다. XML 설정 파일에서 클래스만 바꾸면 소스 코드는 건드릴 필요가 없다는 게 핵심
- **`@Qualifier` 문제가 흥미로웠다**: `HelloService`를 구현한 클래스가 `BasicHelloService`와 `NiceService` 두 개라서 `@Autowired`만 쓰면 어느 걸 주입할지 몰라 에러가 난다. `@Qualifier("basicHelloService")`로 해결할 수 있지만 그러면 소스 코드에 특정 구현체 이름이 박혀서 결국 DI의 장점이 사라진다. XML에서 `helloService` id로 `NiceService`를 명시적으로 등록하는 방식으로 소스 코드 변경 없이 해결
- **Python 데코레이터와 연결**: `@Service`, `@Controller` 어노테이션이 Python의 데코레이터와 비슷한 결이다. 클래스에 메타 정보를 붙여서 Spring이 알아서 처리하게 만드는 방식
- **POJO 철학**: 프레임워크에 종속되지 말자는 이야기가 인상 깊었다. `BasicHelloService`는 Spring 어노테이션(`@Service`)이 붙어 있긴 하지만 비즈니스 로직 자체는 순수 Java다. 프레임워크가 바뀌어도 핵심 로직은 재사용할 수 있어야 한다

---

## 실습 개요

- **목표**: Spring DI의 두 가지 방식 이해 — XML 수동 등록(di01) / 어노테이션 자동 등록(di02)
- **방법**: 강사 실습 코드 따라 작성 후 동작 확인
- **패키지**: `app.lab.di01`, `app.lab.di02`

---

## 진행 과정

### Step 1. di01 — XML 기반 수동 DI (의존성 문제 확인)

**문제 상황 — 소스 코드에 의존성이 박혀 있음**

```java
MessageKo msg = new MessageKo();   // MessageKo에 직접 의존
msg.getMessage();

MessageEn msg1 = new MessageEn();  // MessageEn에 직접 의존
msg1.getMessage();

Message msg2 = new MessageKo();    // 인터페이스 타입이지만 여전히 new MessageKo()
msg2.getMessage();
// 위 세 줄 모두 MessageKo, MessageEn에 의존성을 가짐
// 언어를 바꾸려면 소스 코드를 찾아서 수정해야 함
```

**Spring으로 해결 — XML에서 구현체를 설정**

```xml
<!-- application-config.xml -->
<bean id="MessageService" class="app.lab.di01.MessageKo"/>
<!-- class 부분만 MessageEn으로 바꾸면 소스 코드 수정 없이 언어 전환 -->
```

```java
// Spring 컨테이너에서 빈을 가져옴
AbstractApplicationContext context =
    new GenericXmlApplicationContext("classpath:app/lab/di01/application-config.xml");

Message msg3 = context.getBean("MessageService", Message.class);
msg3.getMessage();  // XML에서 MessageKo로 설정 → 안녕하세요. 여러분
```

- `context.getBean()`으로 꺼낼 때 타입은 `Message` 인터페이스 → 구현체가 뭔지 소스 코드는 모름
- XML의 `class`를 `MessageEn`으로 바꾸면 `msg3.getMessage()`가 "Hello~~~"를 출력
- **인터페이스가 없으면 이 구조 자체가 성립 안 됨** — DI와 인터페이스는 세트

---

### Step 2. di02 — 어노테이션 기반 자동 DI

**구조**

```
HelloService (인터페이스)
  ├─ BasicHelloService (@Service)
  └─ NiceService (@Service)

HelloController (@Controller)
  └─ @Autowired HelloService helloService  ← Spring이 주입
```

**어노테이션으로 빈 자동 등록**

```java
@Service  // Spring 빈으로 자동 등록. id는 클래스명 앞글자 소문자 → "basicHelloService"
public class BasicHelloService implements HelloService {
    @Override
    public String sayHello(String name) {
        System.out.println("BasicHelloService.sayHello() 메서드 실행");
        return "Hello " + name;
    }
}

@Service  // id → "niceService"
public class NiceService implements HelloService {
    @Override
    public String sayHello(String name) {
        System.out.println("NiceService.sayHello() 메서드 실행");
        return "Hello " + name;
    }
}
```

```xml
<!-- component-scan으로 패키지 내 @Service, @Controller 자동 탐색 -->
<context:component-scan base-package="app.lab.di02"/>
```

**@Autowired 주입 방식의 진화 과정 (HelloController 주석에서 확인)**

```java
// 1단계: 변수명으로 id를 맞춰서 주입
@Autowired
HelloService basicHelloService;  // 변수명 = 빈 id → basicHelloService 주입

// 2단계: @Qualifier로 명시적 지정 → 소스 코드에 구현체 이름이 박힘 (DI 장점 반감)
@Autowired
@Qualifier("basicHelloService")
HelloService helloService;

// 3단계: XML에서 helloService id로 NiceService를 명시 등록 → 소스 코드 변경 없이 해결
@Autowired
HelloService helloService;  // XML에서 id="helloService"로 NiceService 지정
```

```xml
<!-- application-config.xml (di02) -->
<!-- NiceService를 helloService id로 명시 등록 → @Autowired가 이 빈을 주입 -->
<bean id="helloService" class="app.lab.di02.NiceService"/>
<context:component-scan base-package="app.lab.di02"/>
```

**실행**

```java
AbstractApplicationContext context =
    new GenericXmlApplicationContext("classpath:app/lab/di02/application-config.xml");

HelloController controller = context.getBean("helloController", HelloController.class);
controller.hello("홍 길 동");
```

**출력 결과**
```
NiceService.sayHello() 메서드 실행
HelloController: Hello 홍길동
```

- `BasicHelloService` → `NiceService`로 바꾸려면 XML의 `class` 한 줄만 수정
- `HelloController` 소스 코드는 전혀 건드릴 필요 없음

---

## 결과 / 산출물

- `app.lab.di01`: `Message.java`, `MessageKo.java`, `MessageEn.java`, `MessageTest.java`, `application-config.xml`
- `app.lab.di02`: `HelloService.java`, `BasicHelloService.java`, `NiceService.java`, `HelloController.java`, `HelloTest.java`, `application-config.xml`

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| di01에서 의존성 문제를 코드로 직접 확인하고 DI로 해결하는 흐름 파악 | `@Qualifier` 문제가 왜 DI 장점을 반감시키는지 처음엔 바로 이해 안 됨 |
| `@Autowired` 주입 방식 진화 과정을 주석 코드로 추적 | Spring 컨테이너가 내부적으로 어떻게 동작하는지는 아직 블랙박스 |
| 인터페이스 + DI 조합이 왜 세트인지 이해 | |

---

## 다음에 이어서 할 것
- Spring 추가 실습 (강의 이어지는 내용)
- Spring 컨테이너 내부 동작 원리 (IoC 컨테이너, 빈 생명주기)

---
*작성일: 2026-05-29 | 인공지능 사관학교 AI 보안 과정*
