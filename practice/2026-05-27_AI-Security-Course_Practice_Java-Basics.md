# 2026-05-27 실습 — Java 기초 (제어흐름 · 배열 · 클래스와 객체)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강의 따라가며 직접 코딩 + 추가 테스트)
> **관련 강의 파일**: `../lectures/2026-05-27_AI-Security-Course_Lecture_Java-Basics.md`

---

## 느낀 것 / 배운 것

### 📅 2026-05-27

- **switch 표현식에서 `yield`를 처음 써봤다.** 블록 안에서 값을 반환하는 개념이 Python의 `yield`와 이름은 같지만 완전히 다른 맥락이라 혼동하지 않도록 주의해야겠다. Java의 `yield`는 제어흐름 안에서 값을 돌려주는 것, Python의 `yield`는 제너레이터 개념.
- **오버로딩을 직접 테스트하면서 Java의 메서드 구별 기준을 체감했다.** `Student(int)`와 `Student(String)` 두 가지를 만들어보니 컴파일러가 타입으로 구별한다는 게 명확하게 이해됐다.
- **`switch (medal) ->` 문법이 JS arrow function처럼 보이지만 함수가 아니라 분기 케이스라는 걸 코드로 직접 비교하면서 정리됐다.**

---

## 실습 개요

- **목표**: 강의 예제를 직접 코딩하며 제어흐름, 배열, 클래스 기초 개념 체화 + 궁금한 부분 직접 테스트
- **방법**: IntelliJ / Eclipse에서 패키지 구조(`basic`, `oop.basic`) 구분하여 작성

---

## 진행 과정

### Step 1. 제어흐름 실습 — `controlEx.java`

**if-else / switch 비교 실습**
```java
String medal = "Gold";

// if-else: equals()로 직접 비교
if (medal.equals("Gold")) { ... }

// switch Java 7: String case 사용
switch (medal) {
    case "Gold" : System.out.println("금메달 입니다."); break;
}

// switch Java 14: arrow 문법, break 불필요
switch (medal) {
    case "Gold" -> System.out.println("금메달 입니다.");
}

// switch 표현식: 값을 변수에 담기
String Message = switch (medal) {
    case "Gold" -> "금메달 입니다.";
    default -> "메달이 없습니다.";
};
```

**월별 마지막 날짜 출력 — yield 사용**
```java
int day = switch(month) {
    case 1,3,5,7,8,10,12 -> 31;
    case 4,6,9,11 -> 30;
    case 2 -> 28;
    default -> {
        if (month < 1 || month > 12) System.out.println("없는 달입니다.");
        else System.out.println("알 수 없는 오류입니다.");
        yield 0;  // 블록에서 값 반환
    }
};
```

**반복문 실습**
```java
// while: 횟수 불명확 시
while (num <= 10) { sum += num; ++num; }

// do-while: 1회 보장
do { sum += num; } while(++num <= 10);

// for: 횟수 고정
for (int i = 1; i <= 10; i++) { sum += i; }

// continue: 짝수 건너뛰고 홀수만 합산
for (int i = 1; i <= 100; i++) {
    if (i % 2 == 0) continue;
    sum += i;
}

// break: 합이 50 이상이 되면 중단
for (int i = 1; i <= 100; i++) {
    if (i >= 50) break;
    sum += i;
}
```

---

### Step 2. 배열 실습 — `ArrayEx.java`

```java
int[] arrays = new int[3];
System.out.println(arrays[1]);  // 선언만 해도 0으로 초기화됨 → 출력: 0

arrays[0] = 1; arrays[1] = 2; arrays[2] = 3;
System.out.println(arrays[1]);    // 2
System.out.println(arrays.length); // 3

// 인덱스 기반 순회 (cnt 변수로 length 미리 저장 — 매 루프 호출 비용 절감)
for (int i = 0, cnt = arrays.length; i < cnt; i++) {
    System.out.println(i + "번 index의 값은 " + arrays[i]);
}

// 값 기반 순회 (enhanced for)
for (int item : arrays) {
    System.out.println();  // 강의 예제 코드 그대로 — 추후 item 출력으로 수정 필요
}
```

---

### Step 3. 클래스와 객체 실습 — `Student.java` / `StudentApp.java`

**Student 클래스 설계**
```java
public class Student {
    int studentId;
    String studentName;
    int grade;
    String address;

    // 디폴트 생성자 (명시적으로 작성)
    public Student() {}

    // 생성자 오버로딩
    public Student(int _studentId) { this.studentId = _studentId; }
    public Student(String studentName) { this.studentName = studentName; }
    public Student(int studentId, String studentName) {
        this.studentId = studentId;
        this.studentName = studentName;
    }

    public void showStudentInfo() {
        System.out.println(studentName + "," + address + "," + this.studentId);
    }
}
```

**인스턴스 생성 및 테스트**
```java
Student student = new Student(1);  // heap에 인스턴스 생성
student.studentName = "홍길동";
student.showStudentInfo();          // 홍길동,주소,1

// 오버로딩 테스트: int + String 조합 생성자
Student studentOverloadingTest = new Student(2, "김길동");
studentOverloadingTest.showStudentInfo();  // 김길동,null,2
```

**메서드를 클래스 외부(StudentApp)에서 정의하고 참조 전달**
```java
static void printInfo(Student student) {
    System.out.println("학생명 : " + student.studentName);
}
```

---

## 결과 / 산출물

- `controlEx.java`: 제어흐름 예제 코드 (if-else, switch 3가지 버전, 반복문 전체)
- `ArrayEx.java`: 배열 선언·초기화·순회 예제
- `Student.java` / `StudentApp.java`: 클래스 설계, 생성자 오버로딩, 인스턴스 생성 테스트

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| switch 3가지 버전을 한 파일에서 직접 비교하며 차이를 체감 | `address`를 초기화하지 않으면 `null`이 출력되는데, null 방어 처리를 해보지 않음 |
| 생성자 오버로딩을 int/String 2가지 타입으로 직접 만들어 테스트 |  |
| `yield` 개념을 월별 날짜 예제로 자연스럽게 적용 | |

---

## 다음에 이어서 할 것

- 5장 나머지 강의 내용 수강 후 TIL 추가
- `address`가 null일 때 방어 처리 (`Optional` 또는 기본값 설정)
- ArrayList 사용 실습 코드 작성

---
*작성일: 2026-05-27 | 인공지능 사관학교 AI 보안 과정*
