# 2026-05-26 강의 정리 — Java 프로그래밍 기초 (1~3장)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Java 프로그래밍 시작 / 변수와 자료형 / 연산자
> **관련 실습 파일**: `../practice/2026-05-26_AI-Security-Course_Practice_Java-Basics.md`

---

## 핵심 인사이트 모음

### 📅 2026-05-26

- **Java의 구조는 package > class > method > code다.** Python의 파일 단위 실행과 달리, Java는 클래스가 코드의 최소 단위다. 모든 코드는 클래스 안에 있어야 하고, 실행은 반드시 `main` 메서드에서 시작한다. 이 구조가 처음엔 번거롭게 느껴지지만, 대형 시스템에서 코드를 체계적으로 구성하기 위한 설계다.
- **Java는 리터럴의 기본 타입이 정해져 있다.** 정수 리터럴은 기본이 `int`, 실수는 `double`이다. 이 때문에 `long`에 큰 수를 넣을 때 `L`을, `float`에 소수를 넣을 때 `f`를 명시해야 한다. Python에서는 이 개념 자체가 없어서 처음엔 낯설게 느껴진다.
- **Overflow는 에러가 아니라 조용히 틀린 답을 낸다.** `int numTest = 123456789 * 100`은 컴파일 오류가 없지만 결과는 `-539222988`이다. 두 리터럴이 모두 `int`라 연산 자체가 `int`로 처리된 후 범위를 넘어 wraparound가 발생한다. `long numTest = 123456789 * 100`도 마찬가지다. `123456789L * 100`으로 한쪽을 `long`으로 만들어야 정상 계산된다.
- **`==`는 주소를 비교하고, `.equals()`는 값을 비교한다.** `String s1 = "abc"`와 `String s2 = "abc"`는 Constant Pool의 같은 객체를 가리켜 `s1 == s2`가 `true`지만, `new String("abc")`는 Heap에 새 객체를 만들어 주소가 달라진다. 문자열 비교는 항상 `.equals()`를 써야 한다.
- **단락 회로 평가(Short-Circuit Evaluation)는 부수 효과를 가진다.** `||`는 앞이 `true`면 뒤를 실행하지 않고, `&&`는 앞이 `false`면 뒤를 실행하지 않는다. 뒤에 `++d` 같은 부수 효과가 있는 코드가 있으면 실행 여부에 따라 결과가 달라지므로 주의가 필요하다.
- **Java 메모리 구조를 알면 `==` vs `.equals()`의 이유가 보인다.** Stack에는 기본값과 참조값(주소)이, Heap에는 객체 인스턴스가, Constant Pool에는 리터럴 상수가 저장된다. `==`는 Stack에 저장된 주소를 비교하는 것이고, `.equals()`는 그 주소가 가리키는 Heap/Pool의 값을 비교하는 것이다.

---

## 1. Java 프로그래밍 시작하기

### 핵심 개념

- Java는 플랫폼 독립적이다. `.java` → 컴파일 → `.class`(바이트코드) → JVM이 각 OS에서 실행
- C/C++는 OS별로 다른 컴파일러와 실행 파일이 필요하지만, Java는 JVM만 있으면 어디서든 동일하게 동작한다

### 상세 내용

**Java 코드 기본 구조**

```java
package basic;           // 패키지 (폴더 단위 묶음)

public class HelloWorld { // 클래스 (소스코드 단위)
    public static void main(String[] args) { // 메서드 (실행 진입점)
        System.out.println("Hello World!!!"); // 코드
        String version = System.getProperty("java.version");
        System.out.println(version);
    }
}
```

- 패키지 > 클래스 > 메서드 > 코드 순의 계층 구조
- `main` 메서드: `public static void main(String[] args)` — 프로그램 실행의 시작점, 형식 고정

**Java의 장점**

- 플랫폼 독립적 (Write Once, Run Anywhere)
- 객체지향 언어 → 유지보수·확장성 우수
- 안정적, 풍부한 오픈소스 생태계
- 활용: 웹 서버, 안드로이드 앱, 게임

---

## 2. 변수와 자료형

### 핵심 개념

- 변수: 메모리 주소 대신 부르는 이름. 사용 전 반드시 선언 필요
- Java는 자료형마다 메모리 크기와 표현 범위가 다르다 → 타입을 명시해야 하는 이유
- 리터럴은 기본적으로 `int`(정수), `double`(실수)로 처리된다

### 상세 내용

**기본 자료형 (Primitive Type)**

| 바이트 | 정수형 | 문자형 | 실수형 | 논리형 |
|---|---|---|---|---|
| 1 | `byte` | — | — | `boolean` |
| 2 | `short` | `char` | — | — |
| 4 | `int` | — | `float` | — |
| 8 | `long` | — | `double` | — |

**정수형 주의사항**

```java
// int가 기본 타입이므로 long 범위를 넘는 리터럴엔 L 필수
long num1 = 12345678900L;    // OK
// long num1 = 12345678900;  // 오류: 리터럴 자체가 int로 처리되어 범위 초과

// Overflow: 컴파일 오류 없이 조용히 틀린 값을 냄
int numTest = 123456789 * 100;    // -539222988 (두 리터럴 모두 int → int 연산)
long numTest2 = 123456789L * 100; // 12345678900 (한쪽을 long으로 만들어야 정상)
```

**실수형 주의사항**

```java
double dnum = 3.14;   // 기본 double
float fnum = 3.14f;   // f 명시 필수

// 부동소수점 오차: 0을 표현할 수 없어 약간의 오차 발생
double d = 0.1 + 0.1 + 0.1;    // 0.30000000000000004
BigDecimal e = new BigDecimal("0.1");
BigDecimal f = e.add(e).add(e); // 0.3 (정밀 계산 필요 시 BigDecimal 사용)
```

**문자형 (char)**

```java
char chb = 'a';                   // 문자 리터럴 (홑따옴표)
char ch2 = 66;                    // 정수값으로도 대입 가능 (유니코드)
System.out.println((char)65);     // A (인코딩: 숫자 → 문자)
System.out.println((int)'a');     // 97 (디코딩: 문자 → 숫자)
```

- `char`는 2바이트. 아스키는 1바이트로 영문·숫자·특수문자 표현

**큰 수 처리**

```java
// long도 부족한 경우: BigInteger, BigDecimal 활용
BigInteger a = new BigInteger("1000000000000000000000");
BigInteger b = new BigInteger("1");
BigInteger c = a.add(b); // 사칙연산은 메서드로
```

**상수와 리터럴**

```java
final double PI = 3.14;   // final: 변경 불가 상수
// PI = 3.15;             // 오류

// 리터럴: 코드에 직접 쓰인 값 (10, 3.14, 'A', true)
// Constant Pool에 저장 → 동일한 값은 하나만 존재
```

**자료형 추론 (Java 10+)**

```java
var g = 10;            // 컴파일러가 int로 추론
var dNum = 10.0;       // double로 추론
var str = "hello";     // String으로 추론
// 로컬 변수에만 사용 가능 (지역 변수 자료형 추론)
```

**형 변환 (Type Conversion)**

```java
// 묵시적 형변환: 작은 → 큰 (자동)
// byte → short → int → long → float → double
long num = 3;          // int → long 자동 변환

// 명시적 형변환: 큰 → 작은 (데이터 손실 가능, 직접 명시)
double dNum = 3.14;
int num = (int)dNum;   // 3 (소수점 버림)

// String ↔ 기본형 변환
int a1 = Integer.parseInt("1");    // String → int
String s1 = Integer.toString(a1);  // int → String
String s2 = String.valueOf(a1);    // int → String
String s3 = a1 + "";               // int → String (간편하지만 권장X)
```

---

## 3. 연산자

### 핵심 개념

- 항의 개수: 단항(++num), 이항(a + b), 삼항(조건식 ? 결과1 : 결과2)
- 대입 연산자(`=`)는 우선순위가 가장 낮고, 오른쪽에서 왼쪽으로 연산

### 상세 내용

**증감 연산자 — 위치에 따라 순서가 다르다**

```java
int i = 5;
System.out.println(i++); // 5 출력 후 i=6 (현재 값 먼저 사용, 후증가)
System.out.println(i);   // 6

i = 5;
System.out.println(++i); // i=6 먼저, 6 출력 (먼저 증가, 후사용)
System.out.println(i);   // 6
```

**`==` vs `.equals()` — String 비교의 핵심**

```java
String s1 = "abc";
String s2 = "abc";
String s3 = new String("abc"); // Heap에 새 객체 생성

s1 == s2     // true  (Constant Pool의 동일 주소)
s1 == s3     // false (Heap의 다른 주소)
s1.equals(s2) // true  (값 비교)
s1.equals(s3) // true  (값 비교)
// → 문자열 비교는 항상 .equals() 사용
```

**단락 회로 평가 (Short-Circuit Evaluation)**

```java
int c = 5, d = 0;

// || : 앞이 true면 뒤 실행 안 함
System.out.println(c != 0 || ++d != 0); // true (++d 실행 안 됨)
System.out.println(c + "," + d);         // 5,0

// && : 앞이 true면 뒤 실행
System.out.println(c != 0 && ++d != 0); // true (++d 실행됨)
System.out.println(c + "," + d);         // 5,1
// → 뒤에 부수 효과(side effect)가 있는 코드는 주의
```

**조건 연산자 (삼항)** -- 다음 강의에서 이어서

```java
int num = (5 > 3) ? 10 : 20; // 조건 참이면 10, 거짓이면 20
```

**비트 연산자 **

- `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` — 다음 강의에서 이어서

**연산자 우선순위 핵심**

- 높음: 일차식 `()[]` → 단항 `! ++ --` → 산술 `* / %` → 산술 `+ -` → 비트이동 → 관계 → 등가 → 비트논리 → 논리 → 조건 → 대입
- 낮음: `= += -= ...`
- 헷갈리면 괄호로 명시하는 것이 최선

---

## 4. JVM 메모리 구조

### 핵심 개념

Java 프로그램이 실행되면 JVM이 메모리를 4개 영역으로 나눠 관리한다. 이 구조를 알아야 `==` vs `.equals()`, Overflow, 참조 타입의 동작 방식이 이해된다.

### 상세 내용

| 영역 | 저장 대상 | 공유 범위 | 수명 |
|---|---|---|---|
| Stack | 기본값, 참조값(주소), 리턴 주소 | 스레드별 독립 | 메서드 실행 동안 |
| Heap | 객체 인스턴스, 배열 | 모든 스레드 | GC가 수거할 때까지 |
| Method Area | 클래스 정보, `static` 변수 | 모든 스레드 | JVM 종료 시까지 |
| Constant Pool | 리터럴 상수 (String, 숫자) | 모든 스레드 | JVM 종료 시까지 |

- `String s1 = "abc"` → Constant Pool의 "abc" 참조 (주소 Stack에)
- `String s3 = new String("abc")` → Heap에 새 객체 생성 (다른 주소)
- `static` 변수는 Method Area에 → 여러 스레드가 공유하므로 동시성 주의
- 자세한 내용: [`../deep-dive/2026-05-26_Java-Memory-Areas.md`]

---

## 아직 찾아보지 않은 것

- `var` 키워드의 한계 — 메서드 파라미터나 필드에 못 쓰는 이유
- `BigDecimal` vs `double` — 금융 계산에서 어떻게 선택하는가
- GC(Garbage Collector) 동작 방식 — 어떤 기준으로 Heap 객체를 수거하는가
- 비트 연산자 실무 활용 — 마스킹, 플래그 처리가 실제로 어디에 쓰이는가
- `static` 변수의 동시성 문제 — synchronized와의 관계

---

## 찾아본 것

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| JVM 메모리 구조 (Stack / Heap / Method Area / Constant Pool) | 2026-05-26 | 별도 deep-dive 파일 참고 | [`../deep-dive/2026-05-26_Java-Memory-Areas.md`] |

---

*작성일: 2026-05-26 | 인공지능 사관학교 AI 보안 과정*
