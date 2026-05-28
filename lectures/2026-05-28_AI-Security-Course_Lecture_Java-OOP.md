# 2026-05-28 강의 정리 — Java OOP (객체지향 심화)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 클래스와 객체 2 (6장) · 상속과 다형성 (8장) · 추상 클래스 (9장)
> **관련 실습 파일**: `../practice/2026-05-27_AI-Security-Course_Practice_Java-Basics.md`

---

## 핵심 인사이트 모음

### 📅 2026-05-28

- **private + getter/setter의 진짜 의미**: 변수를 직접 못 건드리게 막고, setter에 검증 로직을 끼워 넣는 것이 유지보수의 핵심이다. 2월 31일 같은 불가능한 값을 클래스 개발자가 막을 수 있다는 게 핵심
- **static은 클래스에 붙는 것**: 메모리 어디에 올라가는지(데이터 영역)를 알면, 왜 static 메서드에서 인스턴스 변수를 못 쓰는지도 자연스럽게 이해된다
- **추상 클래스**: "전체 흐름은 내가 정의할게, 세부 구현은 너가 해" → 템플릿 메서드 패턴으로 이어진다
- **Vue 선택 경험과 연결**: 과거 프로젝트에서 프론트 경험 없는 팀원들을 위해 Vue를 선택했던 이유가 오늘 강의의 private/캡슐화와 정확히 겹친다. 제약이 품질을 보장한다는 것 — 프레임워크의 제약 = 클래스의 접근 제어자

---

## 1. 6장 — 클래스와 객체 2

### 핵심 개념
- **this**: 생성된 인스턴스 자신을 가리키는 예약어. JS의 this와 달리 항상 인스턴스를 가리킴
- **static**: 클래스 변수. 인스턴스가 아닌 클래스 자체에 묶여 메모리 데이터 영역에 올라감
- **Singleton 패턴**: 시스템 전체에서 단 하나의 인스턴스만 존재하도록 강제하는 패턴

### 상세 내용

**정보 은닉 — private + getter/setter**

```java
// public으로 열어두면 → 검증 불가
public int day;
date.day = 31;  // 2월 31일도 허용됨. 버그

// private + setter로 검증 로직 삽입
private int day;
public void setDay(int day) {
    if (month == 2) {
        if (day < 1 || day > 28) {
            System.out.println("오류입니다");
        } else {
            this.day = day;  // this 없으면 매개변수 day에 대입하는 것으로 인식
        }
    }
}
```

**캡슐화**: 정보와 기능을 하나의 클래스에 모으고 외부에는 필요한 메서드만 오픈. `customer.buyPhone(store)` 한 줄로 요금제 등록, 프로모션 확인, 데이터 이전이 모두 처리됨

**this의 3가지 역할**

| 역할 | 설명 | 예시 |
|---|---|---|
| 자신의 메모리 참조 | 멤버 변수와 매개변수 이름이 같을 때 구분 | `this.name = name` |
| 생성자에서 다른 생성자 호출 | 생성자 최상단에서만 가능 | `this("이름없음", 1)` |
| 자신의 주소 반환 | 빌더 패턴 메서드 체이닝에 활용 | `return this` |

**static 변수 — 메모리 구조로 이해하기**

```
Stack    → 지역 변수 (함수 호출 시 생성, 종료 시 소멸)
Heap     → 인스턴스 변수 (new로 생성, GC가 수거)
데이터 영역 → static 변수 (프로그램 시작 시 생성, 종료 시 소멸)
```

- static 변수는 인스턴스 생성 전부터 존재 → `Student.serialNum = 100;` (클래스로 직접 접근)
- static 메서드에서 인스턴스 변수 사용 불가 → 인스턴스가 아직 없을 수 있기 때문

**변수 유효 범위**

| 변수 유형 | 선언 위치 | 메모리 | 생성/소멸 |
|---|---|---|---|
| 지역 변수 | 함수 내부 | 스택 | 함수 호출 / 종료 |
| 인스턴스 변수 | 클래스 멤버 | 힙 | new / GC 수거 |
| static 변수 | 클래스 내부 (static) | 데이터 영역 | 프로그램 시작 / 종료 |

**Singleton 패턴**

```java
public class Singleton {
    // 1. static으로 유일한 인스턴스를 클래스 변수로 선언
    private static Singleton singleton = new Singleton();

    // 2. 생성자를 private → 외부에서 new 불가
    private Singleton() {}

    // 3. public static 메서드로만 인스턴스 반환
    public static Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    }
}
```

- DB 커넥션, 설정 객체 등 시스템 전체에서 하나만 있어야 하는 것에 사용

---

## 2. 8장 — 상속과 다형성

### 핵심 개념
- **상속(inheritance)**: 기존 클래스를 `extends`로 물려받아 속성과 기능을 확장
- **다형성(polymorphism)**: 상위 클래스 타입 하나로 여러 하위 클래스 인스턴스를 다룰 수 있는 특성
- **업캐스팅/다운캐스팅**: 하위→상위는 묵시적(자동), 상위→하위는 명시적(강제 캐스팅 + instanceof 확인)

### 상세 내용

**상속 기본 문법**

```java
class VIPCustomer extends Customer {
    double saleRatio;

    public VIPCustomer() {
        super();  // 컴파일러가 자동으로 상위 클래스 기본 생성자 호출
        customerGrade = "VIP";
        bonusRatio = 0.05;
        saleRatio = 0.1;
    }
}
```

- Java는 **단일 상속만** 지원 (`extends` 뒤에 클래스 하나만)
- 하위 클래스 생성 시 상위 클래스 인스턴스가 먼저 생성됨 → heap에 상위 + 하위 멤버 변수 모두 올라감
- `super`는 하위 클래스에서 상위 클래스를 가리키는 예약어 (`this`가 자신을 가리키듯)

**접근 제한자 가시성**

| 접근 제한자 | 외부 클래스 | 하위 클래스 | 동일 패키지 | 내부 클래스 |
|---|---|---|---|---|
| public | O | O | O | O |
| protected | X | **O** | O | O |
| default | X | X | O | O |
| private | X | X | X | O |

- 상속 관계에서 하위 클래스가 접근해야 하는 상위 클래스 변수는 `protected`로 선언

**오버라이딩(Overriding)**

```java
@Override
public int calcPrice(int price) {
    bonusPoint += price * bonusRatio;
    return price - (int)(price * saleRatio);  // VIP는 할인율 추가 적용
}
```

- 상위 클래스 메서드를 하위 클래스에서 같은 이름·매개변수로 재정의
- `@Override`는 강제가 아님. 하지만 부모 클래스에 해당 메서드가 없으면 컴파일 에러 → 오타·시그니처 불일치를 잡아주는 안전망 역할

**가상 메서드(Virtual Method)와 다형성**

```java
Customer vc = new VIPCustomer();  // 업캐스팅
vc.calcPrice(10000);
// 타입은 Customer지만 실제 인스턴스(VIPCustomer)의 calcPrice가 호출됨
```

- 메서드는 **실제 인스턴스 타입** 기준으로 실행됨 (가상 메서드 테이블)
- 변수 접근은 **선언 타입** 기준 → `vc`에서 VIPCustomer 전용 메서드는 호출 불가

**다형성 활용**

```java
public void moveAnimal(Animal animal) {
    animal.move();  // 코드는 동일, 실행은 각각 다름
}

moveAnimal(new Human());   // 사람이 두 발로 걷습니다
moveAnimal(new Tiger());   // 호랑이가 네 발로 뜁니다
moveAnimal(new Eagle());   // 독수리가 하늘을 납니다
```

**상속 vs 포함 관계**

| 관계 | 키워드 | 사용 시점 |
|---|---|---|
| IS-A | extends | "VIP 고객은 고객이다" — 일반↔구체 관계 |
| HAS-A | 멤버 변수로 선언 | "학생은 과목을 가진다" — 코드 재사용 목적 |

- 단순 코드 재사용이 목적이면 상속 대신 HAS-A(composition)

**다운캐스팅 & instanceof**

```java
Animal hAnimal = new Human();
if (hAnimal instanceof Human) {
    Human human = (Human) hAnimal;  // 명시적 다운캐스팅
}
```

---

## 3. 9장 — 추상 클래스

### 핵심 개념
- **추상 클래스(abstract class)**: 구현 코드 없이 선언만 있는 추상 메서드를 포함하는 클래스. `new`로 인스턴스 생성 불가
- **템플릿 메서드 패턴**: 전체 흐름은 상위 클래스가 `final` 메서드로 정의하고, 세부 구현은 하위 클래스가 담당

### 상세 내용

**추상 클래스 구현**

```java
public abstract class Computer {
    public abstract void display();  // 선언만, 구현 없음
    public abstract void typing();
    // 일반 메서드도 포함 가능 → 하위 클래스 공통 기능 제공
}

public class DeskTop extends Computer {
    @Override
    public void display() { System.out.println("DeskTop display()"); }
    @Override
    public void typing() { System.out.println("DeskTop typing()"); }
}
```

- 추상 메서드가 있으면 클래스도 반드시 `abstract` 선언
- 모든 메서드에 구현이 있어도 클래스를 `abstract`로 선언하면 `new` 불가

**템플릿 메서드 패턴**

```java
public abstract class PlayerLevel {
    public abstract void run();
    public abstract void jump();
    public abstract void turn();

    // 전체 흐름 고정 — 하위 클래스가 재정의 못 하도록 final
    final public void go(int count) {
        run();
        for (int i = 0; i < count; i++) {
            jump();
        }
        turn();
    }
}
```

- `BeginnerLevel`, `AdvancedLevel`, `SuperLevel`이 각각 `run()`, `jump()`, `turn()`을 다르게 구현
- `go()`의 실행 순서는 항상 동일 → 프레임워크 설계에 자주 사용

**final 예약어**

| 대상 | 의미 |
|---|---|
| final 변수 | 값 변경 불가 (상수) |
| final 메서드 | 하위 클래스에서 오버라이딩 불가 |
| final 클래스 | 상속 불가 (예: Java의 String) |



## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 | 출처 |
|---|---|---|---|
| `@Override` 어노테이션의 역할 | 2026-05-28 | 오버라이딩 자체를 강제하지 않음. 붙였는데 부모 클래스에 동일 시그니처 메서드가 없으면 컴파일 에러 → 오타·시그니처 불일치를 잡아주는 안전망. 없어도 오버라이딩은 동작함 | 시험 후 직접 확인 |

---
*작성일: 2026-05-28 | 인공지능 사관학교 AI 보안 과정*
