# 2026-05-29 강의 정리 — Java 추상 클래스 & 인터페이스

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 추상 클래스 (9장) · 인터페이스 (10장)
> **관련 실습 파일**: `../practice/2026-05-29_AI-Security-Course_Practice_Java-Abstract-Interface.md`

---

## 핵심 인사이트 모음

### 📅 2026-05-29

- **Java가 AI 시대에도 살아있는 이유**: AI 다루기엔 상대적으로 부적합한 언어가 맞다. 하지만 강력한 객체지향 덕분에 생태계가 무너지지 않았다. 설계 능력이 핵심인 언어이고, 그 설계 능력을 오늘 배우고 있다
- **AI와 설계의 관계**: AI 이전엔 구현 시간에 쫓겨 클래스 설계에 할애할 시간이 부족했다. AI 덕분에 구현 속도가 빨라지면서 설계에 더 집중할 수 있게 됐다. 과거 프로젝트에서 설계 문제를 발견해도 전체 영향도를 고려하면 고치기 어려웠는데, AI 도움으로 더 애자일하게 리팩토링할 수 있겠다는 생각이 들었다
- **추상 클래스의 핵심**: 하위 클래스에 `run()`은 반드시 구현하게 강제하면서, `ready()`와 `finish()`의 실행 흐름은 상위 클래스가 독점한다. `RunningRace` 예제에서 명확히 체감됐다
- **인터페이스는 아직 감이 안 잡힘**: 개념은 이해했지만 실제로 어떤 상황에서 써야 할지는 아직 불명확하다. Spring DI와 연결되면서 조금씩 보이기 시작했다 — `Message msg = new MessageKo()` 대신 설정 파일에서 주입받는 구조가 인터페이스가 없으면 불가능한 패턴

---

## 1. 추상 클래스 (9장)

### 핵심 개념
- **추상 클래스**: `abstract` 키워드. 추상 메서드를 포함하며 `new`로 인스턴스 생성 불가
- **하위 클래스 규칙**: 추상 클래스를 상속받은 클래스는 반드시 추상 메서드를 `@Override` 구현해야 함. 단, 하위 클래스도 추상 클래스라면 구현 강제 없음
- **템플릿 메서드 패턴**: 실행 흐름은 `final` 메서드로 고정, 세부 동작은 하위 클래스가 구현

### 상세 내용

**추상 클래스 규칙**

```java
public abstract class Computer {
    public abstract void display();  // 구현 없음 → abstract 필수
    public abstract void typing();
}

// 하위 클래스가 추상 클래스면 구현 강제 없음
public abstract class PartialComputer extends Computer {
    @Override
    public void display() { ... }  // display만 구현해도 OK
    // typing()은 미구현 → 여전히 abstract 클래스여야 함
}
```

**RunningRace 예제 — 템플릿 메서드 패턴**

```java
public abstract class RunningRace {
    private void ready() { System.out.println("준비"); }

    protected abstract void run();  // 하위 클래스가 각자 구현

    private void finish() { System.out.println("결승선"); }

    // 흐름 고정 — 재정의 불가
    public final void race() {
        ready();
        run();
        finish();
    }
}
```

- `ready()`와 `finish()`는 `private` → 하위 클래스에서 직접 호출도 불가
- `run()`만 `protected abstract` → 하위 클래스는 달리기 방식만 정의
- `race()`는 `final` → 어떤 종목이든 준비 → 달리기 → 결승선 순서 보장

**final 예약어**

| 대상 | 의미 | 예시 |
|---|---|---|
| final 변수 | 값 변경 불가 (상수) | `public static final double PI = 3.14` |
| final 메서드 | 오버라이딩 불가 | `public final void race()` |
| final 클래스 | 상속 불가 | Java의 `String` 클래스 |

---

## 2. 인터페이스 (10장)

### 핵심 개념
- **인터페이스**: 모든 메서드가 추상 메서드인 설계 도구. `interface` / `implements` 키워드
- **다중 구현 가능**: 클래스는 `extends` 하나만, 인터페이스는 여러 개 `implements` 가능
- **클라이언트-서비스 약속**: 사용하는 쪽은 구현 방식을 모르고 인터페이스만 믿고 호출

### 상세 내용

**인터페이스 선언**

```java
public interface Calc {
    double PI = 3.14;           // 자동으로 public static final (상수)
    int ERROR = -999999999;

    int add(int num1, int num2);       // 자동으로 public abstract
    int substract(int num1, int num2);
    int times(int num1, int num2);
    int divide(int num1, int num2);
}
```

**단계적 구현 패턴**

```
Calc (인터페이스)
  └─ Calculator (추상 클래스, add/substract만 구현)
       └─ CompleteCalc (구체 클래스, times/divide 추가 구현)
```

- 인터페이스의 모든 메서드를 구현하지 않으면 해당 클래스는 자동으로 추상 클래스가 됨

**인터페이스 타입으로 형 변환**

```java
Calc calc = new CompleteCalc();   // 인터페이스 타입으로 업캐스팅
calc.add(10, 5);                  // Calc에 선언된 메서드만 호출 가능
```

**인터페이스 요소 정리**

| 요소 | 설명 |
|---|---|
| 상수 | 모든 변수 → `public static final` 자동 변환 |
| 추상 메서드 | 모든 메서드 → `public abstract` 자동 변환 |
| 디폴트 메서드 | `default` 키워드로 구현 코드 포함 가능 (Java 8+). 재정의 가능 |
| static 메서드 | 인터페이스 이름으로 직접 호출. `Calc.total(arr)` |
| private 메서드 | 인터페이스 내부 헬퍼. 구현 클래스에서 접근 불가 |

**두 인터페이스 디폴트 메서드 충돌 시**

```java
// Buy, Sell 모두 default void order() 보유 → 충돌
public class Customer implements Buy, Sell {
    @Override
    public void order() {           // 반드시 재정의로 충돌 해소
        System.out.println("고객 판매 주문");
    }
}
```

**왜 인터페이스를 쓰는가 — DAO 예제**

```
UserInfoWeb  →  IUserInfoDao  ←  OracleDao / MysqlDao / MssqlDao
```

- `UserInfoWeb`은 `IUserInfoDao`의 메서드 명세만 보고 호출
- DB를 교체해도 `UserInfoWeb` 코드 수정 불필요 → 구현체만 교체
- Spring DI와 결합하면 소스 코드 수정 없이 설정 파일만으로 구현체 교체 가능

**추상 클래스 vs 인터페이스**

| 구분 | 추상 클래스 | 인터페이스 |
|---|---|---|
| 키워드 | `abstract class` / `extends` | `interface` / `implements` |
| 인스턴스 생성 | 불가 | 불가 |
| 구현 메서드 | 가능 | 디폴트/static/private만 가능 |
| 다중 상속/구현 | 단일 상속 | 다중 구현 가능 |
| 변수 | 일반 변수 선언 가능 | 모두 상수(`public static final`) |
| 주 용도 | 공통 구현 + 일부 강제 구현 | 기능 계약 (구현 방식 무관) |

---

*작성일: 2026-05-29 | 인공지능 사관학교 AI 보안 과정*
