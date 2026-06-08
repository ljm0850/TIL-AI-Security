# 2026-06-08 강의 정리 — Spring Boot & Thymeleaf

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Spring Boot 프로젝트 구조, 설정, Thymeleaf 뷰 템플릿
> **관련 실습 파일**: `../practice/2026-06-08_AI-Security-Course_Practice_SpringBoot.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-08

- Spring Boot는 "설정을 자동화한 Spring Framework"다. Django와 비교하면 큰 그림은 비슷하다 — 설정 파일, URL 매핑, 뷰 렌더링 구조가 닮았다. Django를 배운 경험이 있어서 흐름 이해는 어렵지 않았다.
- Spring Legacy는 XML 설정이 많고 진입장벽이 높다. Spring Boot는 그걸 자동 설정으로 해결한다. 뒤집어 말하면 Spring Boot는 내부 동작을 숨기므로 설정을 직접 써본 경험이 없으면 블랙박스가 되기 쉽다.
- Thymeleaf는 HTML 파일 안에 `th:text`, `th:each`, `th:if` 같은 속성을 넣어서 서버 데이터를 바인딩하는 방식이다. JSP는 `<% %>` 같은 Java 코드가 HTML에 섞이는데, Thymeleaf는 HTML을 HTML처럼 유지하면서 서버 로직을 넣는다.
- 전자정부 프레임워크 등에서 Spring 내에서 HTML 뷰를 직접 만드는 방식을 여전히 사용한다는 이야기를 듣고 당황스러웠다. 현재는 REST API + 프론트엔드 분리 구조만 쓸 줄 알았는데, 실무에서는 SSR(Server-Side Rendering)도 여전히 현역이다.
- `application.properties`가 Django의 `settings.py`와 역할이 같다. 포트, DB 연결, 뷰 리졸버, Mybatis 매핑 위치 등을 모두 여기서 관리한다.

---

## 1. Spring Boot 개요

### 핵심 개념

- Spring Framework는 진입장벽이 높고 초기 설정이 복잡함
- Spring Boot는 이를 해결 — 자동 설정(Auto Configuration), 내장 서버(Tomcat), 스타터 의존성으로 초기 설정 최소화

### 상세 내용

**Spring Framework vs Spring Boot 비교**

| 항목 | Spring Framework (Legacy) | Spring Boot |
|---|---|---|
| 초기설정 | XML 또는 Java 설정 직접 작성 | 자동 설정 제공 |
| 학습 난이도 | 높음 | 낮음 (초보자 친화) |
| 개발 속도 | 초기 느림 | 초기 빠름 |
| 뷰 템플릿 | JSP 주로 사용 | Thymeleaf 권장 |
| 애플리케이션 유형 | WAS 기반 모놀리식 | 마이크로서비스, REST API, 클라우드 |
| 진입장벽 | 높음 | 낮음 |

---

## 2. 프로젝트 구조

### 핵심 개념

- Spring Boot 프로젝트의 핵심 폴더 구조를 이해하면 파일이 어디 있어야 하는지 파악된다

### 상세 내용

```
src/main/java/           → 자바 소스코드 (Controller, Service 등)
src/main/resources/
  ├── static/            → CSS, JS, 이미지 파일
  ├── templates/         → Thymeleaf HTML 파일
  └── application.properties  → 스프링 설정 파일
src/main/webapp/WEB-INF/ → JSP 파일 등 웹 관련 파일 (JSP 사용 시)
pom.xml                  → Maven 의존성 설정
```

---

## 3. 스프링 설정 파일 (application.properties)

### 핵심 개념

- Django의 `settings.py`와 동일한 역할
- 포트, 뷰 리졸버, DB 연결, Mybatis 위치 등 모든 환경 설정을 담당

### 상세 내용

```properties
server.port=8080

# Thymeleaf 설정
spring.thymeleaf.prefix=classpath:templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.cache=false  # 개발 시 false, 운영 시 true

# 데이터소스 (Oracle 예시)
spring.datasource.driver-class-name=oracle.jdbc.driver.OracleDriver
spring.datasource.url=jdbc:oracle:thin:@localhost:1521:XE
spring.datasource.username=SCOTT
spring.datasource.password=TIGER

# Mybatis
mybatis.mapper-locations=classpath:/mappers/*.xml
mybatis.configuration.map-underscore-to-camel-case=true
```

---

## 4. 컨트롤러 & 실행 구조

### 핵심 개념

- `@Controller` + `@RequestMapping`으로 URL과 메서드를 매핑
- `Model`에 데이터를 담아 뷰(HTML)로 전달
- `@ResponseBody`를 붙이면 뷰를 찾지 않고 데이터를 바로 응답 (JSON 등)

### 상세 내용

**주요 어노테이션**

| 어노테이션 | 역할 |
|---|---|
| `@Controller` | 클래스를 컨트롤러로 등록 |
| `@RequestMapping` | 클래스 단위 URL 매핑 |
| `@GetMapping` | GET 요청 처리 메서드 매핑 |
| `@PostMapping` | POST 요청 처리 메서드 매핑 |
| `@ResponseBody` | 뷰 대신 데이터를 직접 HTTP 응답으로 반환 |
| `@Autowired` | Spring이 빈(Bean)을 자동 주입 |
| `@Slf4j` | Lombok 로그 어노테이션 (`log.info()` 사용 가능) |

**DemoApplication vs ServletInitializer**
- `DemoApplication`: 내장 Tomcat으로 실행 (Run As > Java Application), `http://localhost:8080/`
- `ServletInitializer`: 외부 Tomcat에 배포할 때 사용 (Run As > Run on Server), URL에 컨텍스트 이름 붙음

---

## 5. Thymeleaf 뷰 템플릿

### 핵심 개념

- HTML과 유사한 문법의 서버 사이드 템플릿 엔진
- HTML 파일 안에 `th:` 속성을 추가해 서버 데이터를 바인딩
- JSP(`<% %>`)와 달리 HTML을 HTML답게 유지

### 상세 내용

**기본 설정**
```html
<html xmlns:th="http://www.thymeleaf.org">
```
- 파일 위치: `src/main/resources/templates/`
- 확장자: `.html` 권장

**5가지 표준 표현식**

| 표현식 | 역할 |
|---|---|
| `${...}` | 변수 값 출력 (request 데이터) |
| `*{...}` | 선택한 객체의 필드 값 출력 (th:object와 함께) |
| `#{...}` | 국제화 메시지 출력 |
| `@{...}` | URL 설정 (링크, 이미지 src, form action) |
| `~{...}` | Fragment(외부 조각 파일) 포함 |

**주요 속성**

| 속성 | 역할 |
|---|---|
| `th:text` | 태그 텍스트 설정 (HTML 태그 이스케이프) |
| `th:utext` | 태그 텍스트 설정 (HTML 태그 허용) |
| `th:each` | 반복 처리 |
| `th:if / th:unless` | 조건 처리 |
| `th:switch / th:case` | switch-case 처리 |
| `th:insert / th:replace` | Fragment 삽입 |
| `th:with` | 로컬 변수 선언 |

**인라인 출력**
```html
[[ ${msg} ]]   <!-- 태그 허용 X (이스케이프) -->
[( ${msg} )]   <!-- 태그 허용 O -->
```

**Fragment (재사용 가능한 HTML 조각)**
```html
<!-- fragment.html 정의 -->
<div th:fragment="frag01">...</div>
<div th:fragment="frag02(param01, param02)">...</div>

<!-- 사용 -->
<div th:insert="~{/home/fragment :: frag01}"></div>
<div th:replace="~{/home/fragment :: frag02('가', '나')}"></div>
```
- `th:insert`: 태그를 유지하고 안에 삽입
- `th:replace`: 태그 자체를 Fragment로 교체

*작성일: 2026-06-08 | 인공지능 사관학교 AI 보안 과정*
