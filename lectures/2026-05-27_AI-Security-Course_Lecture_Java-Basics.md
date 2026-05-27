# 2026-05-27 강의 정리 — Java 기초 (제어흐름 · 배열 · 클래스와 객체)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Java 4장 제어의 흐름 · 7장 배열과 ArrayList · 5장 클래스와 객체 1 (미완)
> **관련 실습 파일**: `../practice/2026-05-27_AI-Security-Course_Practice_Java-Basics.md`

---

## 핵심 인사이트 모음

### 📅 2026-05-27

- **제어흐름은 "예측 가능성"의 문제다.** 모든 프로그램은 결과를 예측할 수 있어야 하고, Java는 `{}` 블록으로 그 경계를 명확히 만든다. 코드를 읽을 때 블록 단위로 사고하는 습관이 중요하다.
- **switch가 Java 버전마다 진화했다는 게 흥미로웠다.** 7버전(String 지원) → 14버전(arrow 문법) → 표현식(값 반환)으로 이어지는 흐름이 마치 JS의 발전 과정과 유사하게 느껴졌다. `->` 문법은 JS의 arrow function처럼 생겼지만 함수가 아니라 분기 케이스라는 점이 다르다.
- **OOP의 핵심은 "객체들 사이의 관계"다.** 절차형은 시간 흐름, 객체지향은 객체 간 협력 관계 중심. 클래스는 그 관계를 코드로 표현한 설계도다.
- **인스턴스는 heap, 참조변수는 stack.** 클래스(설계도)는 static 영역에 있고, `new`로 찍어낸 인스턴스는 heap에 생성되며, 그 주소를 담은 참조변수는 stack에 산다. 메모리 구조를 이미지로 이해하는 게 중요하다.
- **생성자 오버로딩은 "진입점을 여러 개 만드는 것"이다.** 같은 클래스에서 매개변수 조합이 다른 생성자를 여러 개 두면, 외부에서 다양한 방식으로 객체를 초기화할 수 있다. Java는 메서드명 + 매개변수 타입 조합으로 메서드를 구별한다.

---

## 1. 제어의 흐름 (4장)

### 핵심 개념
- 제어문: 데이터 흐름을 `if`, `for` 등으로 분기·반복시키는 것
- Java는 `{}` 블록으로 실행 단위를 명확히 구분 → 예측 가능한 코드 작성

### 조건문

**if / else if / else**
```java
if (조건식) { ... }
else if (조건식) { ... }
else { ... }
```

**조건 연산자 (삼항)**
```java
max = (a > b) ? a : b;
```

**switch-case**

| 버전 | 특징 |
|---|---|
| Java 7 이전 | int, char, enum만 가능 |
| Java 7 | String을 case 값으로 사용 가능 |
| Java 14 | `->` 문법으로 break 없이 분기 |
| Java 14+ | switch를 표현식으로 사용, 값 반환 가능 |

```java
// Java 7 이상
switch (medal) {
    case "Gold" : System.out.println("금메달"); break;
    default : System.out.println("메달 없음"); break;
}

// Java 14 arrow 문법
switch (medal) {
    case "Gold" -> System.out.println("금메달");
    default -> System.out.println("메달 없음");
}

// 표현식 (값 반환)
String message = switch (medal) {
    case "Gold" -> "금메달입니다.";
    default -> "메달이 없습니다.";
};

// yield: 블록 내부에서 값을 반환할 때
int day = switch(month) {
    default -> {
        System.out.println("오류");
        yield 0;
    }
};
```

> switch의 String 비교 원리(hashCode + equals 2단계)는 별도 심화 파일 참고  
> `../deep-dive/2026-05-27_Java-Switch-String-Comparison.md`

### 반복문

| 구분 | 문법 | 사용 시점 |
|---|---|---|
| while | `while(조건) { }` | 조건 기반, 횟수 불명확 |
| do-while | `do { } while(조건);` | 최소 1회 실행 보장 |
| for | `for(초기화; 조건; 증감) { }` | 횟수 고정 |
| enhanced for | `for(타입 변수 : 배열) { }` | 배열 전체 순회 |

---

## 2. 배열과 ArrayList (7장)

### 핵심 개념
- 같은 자료형 변수를 순차적으로 묶어 관리
- 배열은 선언 시 크기가 고정 → 동적 크기가 필요하면 ArrayList 사용

### 배열 선언과 초기화

```java
int[] arr = new int[10];               // 선언, 정수 0으로 초기화
int[] arr = new int[]{1, 2, 3};        // 선언 + 초기화 (개수 명시 X)
int[] arr = {1, 2, 3};                 // 단축 표현
```

- 미초기화 시 기본값: 정수 `0`, 실수 `0.0`, 객체 `null`
- 인덱스는 0부터 시작, 범위: `0 ~ length-1`
- `arr.length`: 배열 전체 크기 (유효 데이터 수가 아님)

---

## 3. 클래스와 객체 1 (5장, 미완)

### 핵심 개념
- 절차 프로그래밍: 시간 흐름 중심
- 객체지향 프로그래밍(OOP): 객체 간 협력 관계 중심
- OOP 3대 특성: 상속성, 다형성, 은폐성(캡슐화)

### 클래스 정의

```java
public class Student {
    // 멤버 변수 (속성, property, attribute)
    int studentId;
    String studentName;

    // 메서드 (기능)
    public String getStudentName() {
        return studentName;
    }
}
```

- 클래스명은 대문자 시작
- 패키지명, 변수명, 메서드명은 소문자 시작 (camelCase)
- 하나의 `.java` 파일에 `public` 클래스는 하나, 파일명과 동일해야 함

### 인스턴스 생성과 메모리

```java
Student student = new Student();
student.studentName = "홍길동";
```

| 용어 | 설명 |
|---|---|
| 클래스 | 객체 설계도 (static 코드 영역) |
| 인스턴스 | 클래스가 메모리(heap)에 생성된 상태 |
| 참조 변수 | 인스턴스를 가리키는 변수 (stack) |
| 참조 값 | 인스턴스가 생성된 heap 메모리 주소 |

### 함수와 스택 메모리

- 함수 호출 시 stack에 메모리 공간 생성
- 함수 수행 완료 시 stack 메모리 자동 반환
- `main()` 호출 → `add()` 호출 → `add()` 종료 후 반환 → `main()` 계속 실행

### 생성자

```java
public Student() {}                                          // 디폴트 생성자
public Student(int studentId) { this.studentId = studentId; }  // 매개변수 생성자
public Student(int studentId, String studentName) {          // 오버로딩
    this.studentId = studentId;
    this.studentName = studentName;
}
```

- 클래스명과 동일, 반환 타입 없음
- 생성자 미작성 시 컴파일러가 디폴트 생성자 자동 생성
- 매개변수 생성자 하나라도 추가하면 디폴트 생성자 자동 생성 안 됨 → 명시적으로 작성 필요
- **오버로딩**: Java는 `메서드명 + 매개변수 타입`으로 메서드를 구별 → 같은 이름, 다른 시그니처 허용

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| switch String 비교 방식 (hashCode + equals) | 2026-05-27 | `../deep-dive/2026-05-27_Java-Switch-String-Comparison.md` 참고 | 직접 정리 |

---
*작성일: 2026-05-27 | 인공지능 사관학교 AI 보안 과정*
