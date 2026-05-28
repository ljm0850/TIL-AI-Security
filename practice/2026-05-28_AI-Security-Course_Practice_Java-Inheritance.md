# 2026-05-28 실습 — Java 상속 (Customer / VIPCustomer)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-05-28_AI-Security-Course_Lecture_Java-OOP.md`

---

## 느낀 것 / 배운 것

### 📅 2026-05-28

- 생성자 체이닝(`this()` 연쇄 호출)을 직접 써보니 왜 중복을 없애는지 체감됐다. 초기화 로직이 한 곳에만 있으면 나중에 수정할 때 한 곳만 건드리면 된다
- 실습 코드에서 VIP 기본값 설정 코드가 생성자마다 반복되는 걸 보고, 메서드로 빼는 게 자연스럽다고 느꼈다. 단일 책임 원칙을 적용 해봄
- `super()`는 단순히 "부모 생성자 호출"이 아니라, 상위 클래스의 초기화 흐름 전체를 위임하는 것. 생성자 체이닝이 Customer 안에서도 걸려 있어서 `super(id, name)` 한 줄로 3단 연쇄가 실행됨
- `@Override` 어노테이션을 직접 써봤고, 시험에서도 관련 문제가 나왔다. 강제가 아니라 안전망이라는 걸 오늘 확인함

---

## 실습 개요

- **목표**: Customer 클래스를 상속받은 VIPCustomer 구현. 생성자 체이닝, 메서드 오버라이딩, super 활용
- **방법**: 강사 실습 코드 기반으로 직접 작성 후 일부 스스로 리팩토링
- **패키지**: `oop.inheritance`

---

## 진행 과정

### Step 1. Customer 클래스 구현

생성자 3개를 체이닝 구조로 작성. `this()` 호출로 초기화 로직 중복 제거.

```java
public Customer() {
    this.customerGrade = "SILVER";
    this.bonusPoint = 0;
    this.bonusRatio = 0.1;
}

public Customer(int customerId, String customerName) {
    this();  // 기본 생성자 호출 → SILVER, bonusRatio 0.1 초기화
    this.customerId = customerId;
    this.customerName = customerName;
}

public Customer(int customerId, String customerName, int bonusPoint) {
    this(customerId, customerName);  // 2번 생성자 호출
    this.bonusPoint = bonusPoint;
}
```

- 초기화 기본값(SILVER, bonusRatio 0.1)은 기본 생성자 한 곳에서만 관리됨
- `calcPrice()`에서 bonusPoint를 누적 후 price 그대로 반환 (일반 고객은 할인 없음)

### Step 2. VIPCustomer 클래스 구현

`Customer extends`로 상속받고, VIP 전용 속성(`saleRatio`) 추가.

**강사 코드 원형 (각 생성자마다 반복)**

```java
public VIPCustomer() {
    this.customerGrade = "VIP";
    this.bonusRatio = 0.15;
    this.saleRatio = 0.1;
}

public VIPCustomer(int customerId, String customerName) {
    super(customerId, customerName);
    this.customerGrade = "VIP";
    this.bonusRatio = 0.15;
    this.saleRatio = 0.1;
}
```

**직접 리팩토링한 버전** — 반복되는 초기화 코드를 private 메서드로 추출

```java
private void setVIPCustomerDefaultProperty() {
    this.customerGrade = "VIP";
    this.bonusRatio = 0.15;
    this.saleRatio = 0.1;
}

public VIPCustomer() {
    setVIPCustomerDefaultProperty();
}

public VIPCustomer(int customerId, String customerName) {
    super(customerId, customerName);
    setVIPCustomerDefaultProperty();
}
```

- super()와 this()를 둘 다 쓰는 것은 불가능 하여 코드 중복을 줄일 방법에 대해 고민하며 든 생각
- 강사 실습 코드에는 없던 구조. 동일한 코드가 생성자마다 반복되는 걸 보고 스스로 메서드로 분리함
- VIP 기본값이 바뀌면 `setVIPCustomerDefaultProperty()` 한 곳만 수정하면 됨

### Step 3. calcPrice() 오버라이딩

```java
@Override
public int calcPrice(int price) {
    this.bonusPoint += price * this.bonusRatio;  // 15% 적립
    return price - (int)(price * this.saleRatio);  // 10% 할인 적용
}
```

- `price * this.saleRatio`는 double → 명시적으로 `(int)` 캐스팅
- Customer의 `calcPrice()`는 price를 그대로 반환, VIPCustomer는 할인가 반환 → 같은 메서드명으로 다른 동작

### Step 4. CustomerApp에서 테스트

```java
Customer cust2 = new Customer(10, "홍길동");
cust2.calcPrice(5000);  // bonusPoint += 500

VIPCustomer vip1 = new VIPCustomer(100, "둘리");
System.out.println(vip1.calcPrice(100000));  // 90000 출력 (10% 할인)
```

- `super(customerId, customerName)` 호출 시 Customer의 생성자 체이닝이 연쇄 실행됨
  - `Customer(id, name)` → `this()` → 기본 생성자 순으로 올라감
  - 그 후 `setVIPCustomerDefaultProperty()`로 VIP 값으로 덮어씀

---

## 결과 / 산출물

- `Customer.java`, `VIPCustomer.java`, `CustomerApp.java` (`oop.inheritance` 패키지)
- 출력 확인: VIPCustomer 100,000원 구매 시 90,000원 반환, bonusPoint 15,000 누적

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 생성자 체이닝 구조를 직접 설계하고 동작 확인 |          |
| VIP 초기화 코드를 메서드로 추출해 중복 제거 | |
| super() 체이닝이 어떻게 연쇄 실행되는지 흐름 파악 | |
| @Override 직접 작성 후 동작 확인 | |

---

## 다음에 이어서 할 것
- 인터페이스(interface) 실습

---
*작성일: 2026-05-28 | 인공지능 사관학교 AI 보안 과정*
