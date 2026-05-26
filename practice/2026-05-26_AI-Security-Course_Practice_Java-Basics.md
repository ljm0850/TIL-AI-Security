# 2026-05-26 실습 — Java 기초 (자료형·연산자·형변환)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-05-26_AI-Security-Course_Lecture_Java-Basics.md`

---

## 느낀 것 / 배운 것

### 📅 2026-05-26

- **직접 코드를 돌려보니 Overflow가 얼마나 조용히 잘못되는지 실감했다.** `int numTest = 123456789 * 100`은 컴파일도 되고 실행도 되는데 결과가 `-539222988`이다. Python이었다면 자동으로 큰 정수로 처리했을 텐데, Java는 리터럴 타입이 고정돼 있어서 직접 `L`을 붙여야 한다는 걸 코드로 확인했다.
- **`==`로 String을 비교했을 때 `true`가 나오는 경우와 `false`가 나오는 경우를 나란히 보니 Constant Pool의 존재가 실감됐다.** `"abc" == "abc"`가 `true`인 이유, `new String("abc") == "abc"`가 `false`인 이유가 메모리 구조로 연결되는 순간이었다.
- **`Integer.parseInt()`와 `(int)` 캐스팅의 차이를 확인했다.** `(int)String`은 컴파일 오류고, `Integer.parseInt(String)`이 참조형 → 기본형 변환의 올바른 방법임을 직접 오류를 내보고 알게 됐다.

---

## 실습 개요

- **목표**: Java 기초 문법 실습 — 자료형 선언 및 범위, 형변환, 연산자 동작 확인

---

## 진행 과정

### Step 1. HelloWorld — 구조 파악

```java
package basic;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World!!!");
        String version = System.getProperty("java.version");
        System.out.println(version);
    }
}
```

- `package > class > method > code` 계층 구조 확인
- `System.getProperty("java.version")`으로 현재 JDK 버전 출력

---

### Step 2. TypeEx — 자료형 범위와 Overflow

```java
// Overflow 실험
System.out.println(Long.MAX_VALUE);     // 9223372036854775807
System.out.println(Long.MAX_VALUE + 1); // -9223372036854775808 (Overflow)

// 실수 오차
double d = 0.1 + 0.1 + 0.1;
System.out.println(d); // 0.30000000000000004 (오차 발생)

BigDecimal e = new BigDecimal("0.1");
System.out.println(e.add(e).add(e)); // 0.3 (정확)

// 문자 ↔ 숫자 변환
int cha = 65;
System.out.println((char)cha); // A
char chb = 'a';
System.out.println((int)chb);  // 97

// var 추론
var g = 10; // int로 추론
```

**직접 실험 — Overflow 재현**

```java
// 두 리터럴이 모두 int → int 연산 → 범위 초과 → -539222988
int numTest = 123456789 * 100;
long numTest1 = 123456789 * 100; // long 변수여도 연산은 int로 처리됨 → 같은 결과

// 한쪽을 long으로 만들어야 정상
long numTest2 = 123456789L * 100; // 12345678900
```

---

### Step 3. TypeConversion — 형변환

```java
// double → int: 소수점 버림
double dNum = 3.14;
int num = (int)dNum;
System.out.println(num); // 3

// String → int: (int) 캐스팅 불가, parseInt 필요
String a = "1", b = "2";
System.out.println(a + b);                        // "12" (문자열 연결)
System.out.println(Integer.parseInt(a) + Integer.parseInt(b)); // 3

// int → String 3가지 방법
String s1 = Integer.toString(a1);
String s2 = String.valueOf(b1);
String s3 = a1 + ""; // 간편하지만 암묵적 변환
```

---

### Step 4. OperatorEx — 연산자 동작 확인

```java
// 증감 연산자 위치 차이
int i = 5;
System.out.println(i++); // 5 (출력 후 증가)
System.out.println(i);   // 6

i = 5;
System.out.println(++i); // 6 (증가 후 출력)

// String == 와 .equals()
String s1 = "abc", s2 = "abc";
String s3 = new String("abc");
System.out.println(s1 == s2);     // true (Constant Pool 동일 주소)
System.out.println(s1 == s3);     // false (Heap의 다른 주소)
System.out.println(s1.equals(s3)); // true (값 비교)

// 단락 회로 평가
int c = 5, d = 0;
System.out.println(c != 0 || ++d != 0); // true (++d 실행 안 됨)
System.out.println(c + "," + d);         // 5,0

System.out.println(c != 0 && ++d != 0); // true (++d 실행됨)
System.out.println(c + "," + d);         // 5,1
```

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| Overflow 원인을 코드로 직접 재현하고 원인 파악 | `(int)String` 캐스팅이 안 된다는 것을 오류 내보고 알게 됨 |
| Constant Pool vs Heap의 차이를 `==` 결과로 실감 |  |
| `BigDecimal`로 실수 오차 해결법 확인 | |

---

## 다음에 이어서 할 것

- 조건 연산자(`? :`) 실습
- 비트 연산자 (`&`, `|`, `^`, `~`, `<<`, `>>`) 실습

---

## 아직 찾아보지 않은 것

- `String.valueOf()` vs `Integer.toString()` 내부 차이
- `var` 키워드 사용 제약 조건 (지역 변수만 가능한 이유)

---

*작성일: 2026-05-26 | 인공지능 사관학교 AI 보안 과정*
