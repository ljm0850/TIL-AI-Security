# 2026-06-09 강의 정리 — Spring Boot MVC & Oracle 연동 구조

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: MVC/MVVM 패턴 리뷰, DAO 계층, Spring + Oracle 전체 흐름
> **관련 실습 파일**: `../practice/2026-06-09_AI-Security-Course_Practice_SpringBoot-CRUD.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-09

- React의 핵심은 Component 단위 재활용이다. SPA가 무조건 정답이 아니다 — 사용자 단은 SPA가 유리하지만, 관리자 페이지는 굳이 SPA로 만들 필요가 없다. 트레이드오프를 고민하자.
- MVC 패턴의 흐름을 코드로 직접 치면서 Django와의 대응 관계가 더 선명해졌다. `Controller` = `views.py`, `Service` = 비즈니스 로직 분리 계층, `Repository` = ORM/쿼리 계층.
- DAO 패턴의 핵심은 "DB 접근 로직을 별도 계층으로 분리"하는 것이다. Spring에서는 `@Mapper` 인터페이스가 그 역할을 한다. Mybatis가 XML과 Java 메서드를 연결해준다.
- 오늘 실습에서 `Dept.java → dept.xml → DeptRepository → DeptService → BasicDeptService → DeptController` 순서로 쌓아 올라가는 구조를 직접 손으로 쳤다. 계층이 명확히 분리되어 있어서 어디서 뭘 고쳐야 하는지가 한눈에 보인다.
- `@Controller`는 뷰를 반환하고, `@RestController`는 데이터(JSON)를 반환한다. 같은 CRUD 기능인데 컨트롤러가 두 개(`DeptController`, `DeptAPIController`)로 나뉘는 이유가 여기 있다.

---

## 1. 지난 시간 리뷰 — React & Spring Boot

### 핵심 개념

- **React**: Component 단위 재활용이 핵심
- **SPA 트레이드오프**: 사용자 화면은 SPA가 유리, 관리자 페이지는 SSR도 충분

### 상세 내용

- React는 JSX로 React 요소 객체를 만들어 재사용하는 구조
- Spring Boot는 Spring의 복잡한 설정을 자동화 — 딥하게 알지 못해도 사용 가능하지만, 내부 동작 학습 기회가 적다는 단점이 있음

---

## 2. MVC vs MVVM 디자인 패턴

### 핵심 개념

| 패턴 | 구성 | 주요 사용처 |
|---|---|---|
| MVC | Client ↔ Controller ↔ (Service) ↔ Model | Spring, Django, 서버 사이드 |
| MVVM | Model ↔ ViewModel ↔ View | Vue.js, React, 클라이언트 사이드 |

### 상세 내용

**MVC (Model-View-Controller)**
```
Client
  ↕ HTTP 요청/응답
Controller   ← URL 매핑, 요청 처리
  ↕
Service      ← 비즈니스 로직
  ↕
Repository   ← DB 접근 (DAO)
  ↕
DB
```

**MVVM (Model-View-ViewModel)**
```
View (HTML/JSX)
  ↕ 데이터 바인딩
ViewModel (Vue data/methods, React state)
  ↕
Model (API 응답 데이터)
```

---

## 3. DAO (Data Access Object) 패턴

### 핵심 개념

- **데이터 접근 객체**: DB나 외부 저장소에 대한 접근 로직을 별도 계층으로 분리하는 디자인 패턴
- Spring + Mybatis에서는 `@Mapper` 인터페이스가 DAO 역할을 담당

### 상세 내용

**JDBC 타입 변환**
- SQL `NUMBER` → Java `int`
- SQL `VARCHAR2` → Java `String`
- JDBC가 DB 타입과 Java 타입 사이의 변환을 자동으로 처리

**Mybatis에서의 DAO**
```java
@Mapper  // Mybatis mapper XML의 id와 Java 메서드를 매핑
public interface DeptRepository {
    List<Dept> getDeptList();       // XML의 <select id="getDeptList"> 와 연결
    Dept getDeptInfo(int deptno);
    void insertDept(Dept dept);
    // ...
}
```
- 메서드 이름 = XML의 `id` 속성
- 매개변수와 반환형도 XML과 동일하게 맞춰야 함

---

## 4. Spring + Oracle 전체 계층 구조

### 핵심 개념

오늘 실습에서 아래 순서로 계층을 쌓아 올렸다:

```
Dept.java (Model/Entity)
  → dept.xml (Mybatis SQL 매핑)
  → DeptRepository.java (DAO 인터페이스, @Mapper)
  → DeptService.java (Service 인터페이스)
  → BasicDeptService.java (Service 구현체)
  → DeptController.java (SSR 컨트롤러, @Controller)
  → DeptAPIController.java (REST 컨트롤러, @RestController)
  → main.html / list.html (Thymeleaf 뷰)
```

### 상세 내용

**각 계층의 책임**

| 계층 | 클래스 | 역할 |
|---|---|---|
| Model | `Dept.java` | DB 테이블 대표 객체 (Lombok으로 getter/setter 자동 생성) |
| DAO | `DeptRepository.java` | Mybatis XML과 Java 메서드 매핑 (`@Mapper`) |
| Service 인터페이스 | `DeptService.java` | 비즈니스 로직 계약 정의 |
| Service 구현체 | `BasicDeptService.java` | DAO에서 데이터 가져와 가공 후 Controller에 전달 |
| Controller (SSR) | `DeptController.java` | 뷰 렌더링 (`@Controller`, Thymeleaf) |
| Controller (API) | `DeptAPIController.java` | JSON 응답 (`@RestController`, REST API) |

**`@Controller` vs `@RestController`**

| | `@Controller` | `@RestController` |
|---|---|---|
| 반환 | 뷰 이름 (Thymeleaf 렌더링) | 데이터 (JSON 직렬화) |
| 리다이렉트 | `RedirectAttributes`로 Flash 메시지 전달 | — |
| 주 용도 | SSR (서버 사이드 렌더링) | REST API |

**생성자 주입 vs `@Autowired` 필드 주입**
- `DeptAPIController`는 생성자 주입 방식 사용 → Spring 공식 권장 방식
- `DeptController`는 `@Autowired` 필드 주입 방식 사용
- 두 방식의 차이: 생성자 주입은 테스트 용이, 의존성 불변성 보장



*작성일: 2026-06-09 | 인공지능 사관학교 AI 보안 과정*
