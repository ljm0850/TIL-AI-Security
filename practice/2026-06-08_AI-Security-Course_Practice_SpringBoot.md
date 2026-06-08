# 2026-06-08 실습 — Spring Boot & Thymeleaf

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-08_AI-Security-Course_Lecture_SpringBoot.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-08

- 큰 그림은 Django를 배운 경험이 있어서 이해는 됐다. URL 매핑 → 컨트롤러 → 서비스 → 뷰로 이어지는 흐름이 Django의 urls.py → views.py → template 구조와 거의 같다.
- 그런데 Spring 내에서 HTML 뷰를 직접 만들어 쓰는 방식을 실무에서 여전히 사용한다는 이야기를 듣고 조금 당황스러웠다. REST API + 프론트엔드 분리 구조만 쓰는 줄 알았는데, 전자정부 프레임워크 등에서는 현역이다.
- `application.properties`를 보면서 Django `settings.py`랑 역할이 같다는 걸 바로 느꼈다. DB 연결, 포트, 로깅, 뷰 설정 다 여기 있다.
- Thymeleaf는 JSP와 다르게 HTML을 HTML답게 유지한다는 게 실제로 편했다. `th:text`, `th:each`, `th:if` 정도만 알아도 기본적인 뷰는 다 만들 수 있겠다는 감이 왔다.

---

## 실습 개요

- **목표**: Spring Boot 프로젝트 구조 파악, Controller-Service 연결, Thymeleaf 뷰 작성
- **방법**: 강의 예제 코드 직접 작성 (`springBtWeb` 프로젝트)

---

## 진행 과정

### Step 1. 프로젝트 설정 확인 (application.properties)

```properties
spring.application.name=springBtWeb
server.port=8080

# Thymeleaf 템플릿 설정
spring.thymeleaf.prefix=classpath:templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.cache=false

# Oracle DB 연결
spring.datasource.driver-class-name=oracle.jdbc.driver.OracleDriver
spring.datasource.url=jdbc:oracle:thin:@localhost:1521:XE
spring.datasource.username=SCOTT
spring.datasource.password=TIGER

# Mybatis 매퍼 위치
mybatis.mapper-locations=classpath:/mappers/*.xml
mybatis.configuration.map-underscore-to-camel-case=true
```

### Step 2. 엔트리포인트 & ServletInitializer (강사 코드)

```java
// SpringBtWebApplication.java — 내장 Tomcat으로 실행
@SpringBootApplication
public class SpringBtWebApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBtWebApplication.class, args);
    }
}

// ServletInitializer.java — 외부 Tomcat 배포 시 사용
public class ServletInitializer extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(SpringBtWebApplication.class);
    }
}
```

- `SpringBtWebApplication`: 내장 Tomcat으로 바로 실행 → `http://localhost:8080/`
- `ServletInitializer`: 외부 Tomcat에 War 배포 시 사용 → URL에 컨텍스트 이름 붙음

### Step 3. Service 인터페이스 & 구현체 (강사 코드)

```java
// HomeService.java — 인터페이스 정의
public interface HomeService {
    List<String> getList();
    Map<String, String> getMap();
}

// DefaultHomeService.java — 구현체
@Service
public class DefaultHomeService implements HomeService {
    @Override
    public List<String> getList() {
        List<String> list = new ArrayList<>();
        list.add("A"); list.add("B"); list.add("C"); list.add("D"); list.add("E");
        return list;
    }

    @Override
    public Map<String, String> getMap() {
        Map<String, String> map = new HashMap<>();
        map.put("A", "가"); map.put("B", "나"); map.put("C", "다");
        map.put("D", "라"); map.put("E", "마");
        return map;
    }
}
```

### Step 4. Controller 작성 (강사 코드)

```java
@Slf4j
@Controller
@RequestMapping("home")
public class homeController {

    // GET /home/msg → templates/home/home.html 반환
    @GetMapping("/msg")
    public String MsgView(Model model) {
        return "home/home";
    }

    // GET /home/today → Model에 날짜 담아 뷰로 전달
    @GetMapping("/today")
    public String index(Model model) {
        LocalDateTime now = LocalDateTime.now();
        String today = now.format(DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss"));
        model.addAttribute("today", today);
        return "home/hello-world";
    }

    // POST /home/msg → JSON 응답 (@ResponseBody)
    @PostMapping("/msg")
    @ResponseBody
    public Map<String, String> JsonMsg(Model model) {
        return Map.of("text", "안녕하세요");
    }

    // GET /home/basic → Service에서 List, Map 가져와 뷰로 전달
    @Autowired
    HomeService defaultHomeService;

    @GetMapping("/basic")
    public String basic(Model model) {
        List<String> list = defaultHomeService.getList();
        Map<String, String> map = defaultHomeService.getMap();
        model.addAttribute("str", "안녕하세요.");
        model.addAttribute("msg", "<b>EL Test</b>");
        model.addAttribute("num", 3.14);
        model.addAttribute("list", list);
        model.addAttribute("map", map);
        return "home/basic";
    }
}
```

**포인트**
- `return "home/hello-world"` → `templates/home/hello-world.html`을 찾아 렌더링
- `@ResponseBody` 붙이면 뷰 대신 반환값을 HTTP Body에 직접 담아 응답 (JSON)
- `model.addAttribute("key", value)` → Thymeleaf에서 `${key}`로 접근

### Step 5. Thymeleaf 뷰 작성

**hello-world.html** — Model 데이터 바인딩
```html
<html xmlns:th="http://www.thymeleaf.org">
<body>
    <h1>현재시간</h1>
    <h3 th:text="${today}"></h3>
</body>
```

**home.html** — jQuery Ajax로 POST 요청 후 JSON 응답 렌더링
```html
<script src="https://cdn.jsdelivr.net/npm/jquery@4.0.0/dist/jquery.min.js"></script>
<script>
    $.post('/home/msg', {}, function () {})
        .done(function (data) {
            $('#msg').text(data.text);
        });
</script>
```

**basic.html** — Thymeleaf 주요 기능 실습
```html
<!-- 변수 출력 -->
<li th:text="${str}"></li>
<li th:text="${msg}"></li>         <!-- 태그 이스케이프 O -->
<li>[[ ${ msg } ]]</li>            <!-- 태그 이스케이프 O (인라인) -->
<li>[( ${ msg } )]</li>            <!-- 태그 허용 O (인라인) -->

<!-- 스프링 빈 직접 호출 -->
<li>[[ ${ @defaultHomeService.getList()[0] } ]]</li>

<!-- 조건 처리 -->
<span th:if="${num > 5}">크다</span>
<span th:if="${num <= 5}">작다</span>

<!-- switch-case -->
<div th:with="str='F'" th:switch="${str}">
    <span th:case="A">90~</span>
    <span th:case="*">~69</span>   <!-- default -->
</div>

<!-- 반복 처리 (stat 객체: index, count, size, current) -->
<div th:each="item, stat:${list}">
    <div th:text="| ${stat.index}, ${stat.count}, ${stat.size}, ${stat.current} : ${item} |"></div>
</div>
```

**fragment.html** — 재사용 가능한 HTML 조각 정의
```html
<div th:fragment="frag01">
    <h4>fragment</h4>
</div>
<div th:fragment="frag02(param01, param02)">
    <div th:text="${param01}"></div>
    <div th:text="${param02}"></div>
</div>
```

Fragment 사용 (basic.html에서):
```html
<div th:insert="~{/home/fragment :: frag01}"></div>
<div th:replace="~{/home/fragment :: frag01}"></div>
<div th:replace="~{/home/fragment :: frag02('가', '나')}"></div>
```

---

## 결과 / 산출물

- `springBtWeb` Spring Boot 프로젝트
  - `GET /home/today` → 현재 시간 출력
  - `POST /home/msg` → JSON 응답 → jQuery Ajax로 화면 렌더링
  - `GET /home/basic` → Thymeleaf 조건/반복/Fragment 전체 실습

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| Controller → Service → View 전체 흐름 한 번에 파악 |
| `@ResponseBody`로 JSON 응답 vs 뷰 반환 차이 이해 |

---

## 다음에 이어서 할 것

- Spring Boot + Mybatis + Oracle DB 연동 실습
- REST API 전용 `@RestController` 사용

---
*작성일: 2026-06-08 | 인공지능 사관학교 AI 보안 과정*
