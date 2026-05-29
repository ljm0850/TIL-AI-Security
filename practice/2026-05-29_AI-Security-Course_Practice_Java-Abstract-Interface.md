# 2026-05-29 실습 — Java 추상 클래스 & 인터페이스

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-05-29_AI-Security-Course_Lecture_Java-Abstract-Interface.md`

---

## 느낀 것 / 배운 것

### 📅 2026-05-29

- `RunningRace` 예제를 직접 치면서 템플릿 메서드 패턴이 왜 강력한지 체감했다. `race()`를 `final`로 막아두니 어떤 종목(Run100, Run200...)을 만들어도 준비 → 달리기 → 결승선 흐름이 깨질 수가 없다
- `ready()`와 `finish()`가 `private`인 점이 인상적이었다. 하위 클래스는 흐름 자체에 관여할 수 없고, 오직 `run()`만 구현하면 된다. 책임의 경계가 명확하다
- `Message` 인터페이스 예제에서 `Message msg2 = new MessageEn()` 형태로 업캐스팅하는 코드를 직접 써봤다. 타입은 `Message`이지만 실제 실행은 `MessageEn`의 `getMessage()`가 호출된다 — 어제 배운 가상 메서드 원리와 동일하게 작동
- 인터페이스의 실용성은 아직 완전히 감이 안 잡혔지만, Spring DI 예제와 연결되면서 `Message msg = new MessageKo()` 대신 설정 파일에서 구현체를 주입받는 구조가 인터페이스 없이는 불가능하다는 걸 이해했다

---

## 실습 개요

- **목표**: 추상 클래스로 템플릿 메서드 패턴 구현 / 인터페이스로 다형성 구현
- **방법**: 강사 실습 코드 따라 작성 후 동작 확인
- **패키지**: `oop.abstract_ex`, `oop.interface_ex`

---

## 진행 과정

### Step 1. 추상 클래스 — RunningRace (템플릿 메서드 패턴)

**패키지**: `oop.abstract_ex`

```java
// RunningRace.java — 추상 클래스
public abstract class RunningRace {
    int i;  // 변수 값을 미래에 정하겠다 (주석에서 의도 명확히 드러남)

    private void ready() { System.out.println("준비"); }

    protected abstract void run();  // 달리기 방식은 하위 클래스가 결정

    private void finish() { System.out.println("결승선"); }

    public final void race() {  // 흐름 고정 — 재정의 불가
        ready();
        run();
        finish();
    }
}
```

```java
// Run100.java — 구체 클래스
public class Run100 extends RunningRace {
    @Override
    protected void run() {
        System.out.println("100M 달리기");
    }
}
```

```java
// RunApp.java — 실행
public class RunApp {
    public static void main(String[] args) {
        Run100 race = new Run100();
        race.race();  // 준비 → 100M 달리기 → 결승선
    }
}
```

**출력 결과**
```
준비
100M 달리기
결승선
```

**포인트**
- `ready()`, `finish()`가 `private` → 하위 클래스에서 직접 호출 불가. 흐름을 외부에서 끊을 방법이 없음
- `run()`이 `protected abstract` → 하위 클래스에서만 구현 가능, 외부에서는 `race()`를 통해서만 실행됨
- `Run200`, `Hurdle` 등을 추가해도 `race()`의 흐름은 항상 동일하게 보장됨

---

### Step 2. 인터페이스 — Message (다형성)

**패키지**: `oop.interface_ex`

```java
// Message.java — 인터페이스
public interface Message {
    void getMessage();  // public abstract 자동 적용
}
```

```java
// MessageKo.java
public class MessageKo implements Message {
    @Override
    public void getMessage() { System.out.println("안녕 하세요."); }
}

// MessageEn.java
public class MessageEn implements Message {
    @Override
    public void getMessage() { System.out.println("Hello!!"); }
}
```

```java
// MessageApp.java — 실행
public class MessageApp {
    public static void main(String[] args) {
        MessageKo msg = new MessageKo();
        msg.getMessage();           // 안녕 하세요.

        MessageEn msg1 = new MessageEn();
        msg1.getMessage();          // Hello!!

        Message msg2 = new MessageEn();  // 인터페이스 타입으로 업캐스팅
        msg2.getMessage();          // Hello!! — 실제 인스턴스 메서드 호출
    }
}
```

**출력 결과**
```
안녕 하세요.
Hello!!
Hello!!
```

**포인트**
- `msg2`의 타입은 `Message`지만 실행은 `MessageEn.getMessage()` → 가상 메서드 원리
- Spring DI로 연결되면: `Message msg = new MessageKo()` 대신 설정 파일에서 `MessageKo` 또는 `MessageEn`을 주입 → 소스 코드 수정 없이 언어 전환 가능

---

## 결과 / 산출물

- `oop.abstract_ex`: `RunningRace.java`, `Run100.java`, `RunApp.java`
- `oop.interface_ex`: `Message.java`, `MessageKo.java`, `MessageEn.java`, `MessageApp.java`

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 템플릿 메서드 패턴 흐름을 코드로 확인 | 인터페이스를 어떤 상황에서 직접 설계해야 할지 아직 감이 없음 |
| `private` / `protected abstract` / `final` 접근 제어 조합의 의도 파악 | |
| 인터페이스 업캐스팅 후 가상 메서드 동작 확인 | |

---

## 다음에 이어서 할 것
- Spring DI 실습 코드 정리 (인터페이스 + 의존성 주입 연결)
- 인터페이스를 직접 설계해보는 연습

---
*작성일: 2026-05-29 | 인공지능 사관학교 AI 보안 과정*
