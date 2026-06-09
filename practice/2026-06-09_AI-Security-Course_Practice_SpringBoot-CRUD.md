# 2026-06-09 실습 — Spring Boot + Oracle CRUD

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-06-09_AI-Security-Course_Lecture_SpringBoot-MVC.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-09

- `@RestController`에서 `Map<String, Object>`로 응답을 직접 구성하는 방식이 Django의 `JsonResponse`랑 비슷한 느낌이었다. `ResponseEntity`를 쓰면 HTTP 상태코드까지 제어할 수 있다는 것도 오늘 주석에서 확인했다 — 아직 직접 써보지는 않았다.
- `RedirectAttributes.addFlashAttribute`로 리다이렉트 후에도 메시지를 전달하는 방식이 새로웠다. POST → redirect → GET 패턴(PRG 패턴)을 Spring에서 이렇게 구현하는구나 싶었다.

---

## 실습 개요

- **목표**: Spring Boot + Mybatis + Oracle DEPT 테이블 CRUD 전체 계층 직접 작성
- **방법**: 강사 코드를 따라 치며 각 계층의 역할과 연결 방식 확인
- **프로젝트**: `springBtWeb` (기존 프로젝트에 `dept` 패키지 추가)

---

## 진행 과정

### Step 1. Dept.java — 모델(Entity) 작성 (강사 코드)

```java
// app.labs.dept.model.Dept
@Getter @Setter
@ToString
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Dept {
    private int deptNo;   // DB: DEPTNO (NUMBER)
    private String dName; // DB: DNAME (VARCHAR2)
    private String loc;   // DB: LOC (VARCHAR2)
}
```

**포인트**
- Lombok 어노테이션으로 getter/setter/toString/생성자를 자동 생성
- `@Builder`: 빌더 패턴으로 객체 생성 가능 (`Dept.builder().deptNo(10).build()`)
- `application.properties`의 `mybatis.configuration.map-underscore-to-camel-case=true` 덕분에 DB의 `DEPT_NO` → Java의 `deptNo` 자동 변환

---

### Step 2. DeptRepository.java — DAO 인터페이스 (강사 코드)

```java
// app.labs.dept.dao.DeptRepository
@Mapper  // Mybatis mapper XML과 Java 메서드를 매핑
public interface DeptRepository {
    int getDeptCount();
    int getDeptCount(@Param("deptno") int deptno);  // 동일 이름 메서드 → @Param 필수
    List<Dept> getDeptList();
    Dept getDeptInfo(int deptno);
    void updateDept(Dept dept);
    void insertDept(Dept dept);
    int deleteDept(@Param("deptno") int deptno);
}
```

**포인트**
- `@Mapper`: Mybatis가 이 인터페이스의 구현체를 자동 생성 (프록시 패턴)
- 메서드 이름 = `mapper XML`의 `id` 속성값과 반드시 일치해야 함
- `@Param("deptno")`: 매개변수가 1개라도 오버로딩된 메서드거나 XML에서 이름으로 참조할 때 필요

---

### Step 3. DeptService.java + BasicDeptService.java — 서비스 계층 (강사 코드)

```java
// DeptService.java — 인터페이스 (계약 정의)
public interface DeptService {
    int getDeptCount();
    int getDeptCount(int deptno);
    List<Dept> getDeptList();
    Dept getDeptInfo(int deptno);
    void updateDept(Dept dept);
    void insertDept(Dept dept);
    int deleteDept(int deptno);
}

// BasicDeptService.java — 구현체
@Service  // DAO에서 데이터를 가져와 가공한 뒤 Controller에 전달
public class BaiscDeptService implements DeptService {

    @Autowired
    DeptRepository deptRepository;

    @Override
    public List<Dept> getDeptList() {
        return deptRepository.getDeptList();
    }

    @Override
    public int deleteDept(int deptno) {
        return deptRepository.deleteDept(deptno);  // 삭제된 행 수 반환
    }
    // ... 나머지 메서드도 동일 패턴
}
```

**포인트**
- 인터페이스와 구현체 분리 → 나중에 구현체를 바꿔도 Controller 코드는 수정 불필요
- `deleteDept`가 `void`가 아닌 `int`를 반환하는 이유: 실제 삭제 여부를 확인하기 위해 (삭제된 행 수)

---

### Step 4. DeptController.java — SSR 컨트롤러 (강사 코드)

```java
@Slf4j
@Controller
@RequestMapping("dept")
public class DeptController {

    @Autowired
    DeptService basicDeptService;

    // GET /dept/list → 목록 조회
    @GetMapping("list")
    public String getAllDepts(Model model) {
        model.addAttribute("deptList", basicDeptService.getDeptList());
        model.addAttribute("dept", new Dept());  // 빈 객체 (입력 폼용)
        return "dept/list";
    }

    // GET /dept/list/{deptno} → 단건 조회 (선택)
    @GetMapping("/list/{deptno}")
    public String getDeptInfo(@PathVariable("deptno") int deptno, Model model) {
        model.addAttribute("deptList", basicDeptService.getDeptList());
        model.addAttribute("dept", basicDeptService.getDeptInfo(deptno));
        return "dept/list";
    }

    // POST /dept/insert → 등록 후 리다이렉트
    @PostMapping("insert")
    public String insertDept(Dept dept, RedirectAttributes redirectAttributes) {
        try {
            basicDeptService.insertDept(dept);
            redirectAttributes.addFlashAttribute("message", dept.getDeptNo() + "번 부서가 신규 등록되었습니다.");
        } catch (RuntimeException e) {
            redirectAttributes.addFlashAttribute("message", e.getMessage());
        }
        return "redirect:/dept/list";  // PRG 패턴
    }

    // POST /dept/delete → 삭제
    @PostMapping("delete")
    public String deleteDept(Dept dept, Model model, RedirectAttributes redirectAttrs) {
        int deletedRow = basicDeptService.deleteDept(dept.getDeptNo());
        if (deletedRow > 0) {
            redirectAttrs.addFlashAttribute("message", dept.getDeptNo() + "번 부서가 삭제되었습니다.");
            return "redirect:/dept/list";
        } else {
            model.addAttribute("message", "부서번호가 다릅니다.");
            return "dept/list";  // 리다이렉트 없이 뷰 반환
        }
    }
}
```

**포인트**
- `RedirectAttributes.addFlashAttribute`: 리다이렉트 후 1회성으로 메시지 전달 (세션에 임시 저장)
- PRG 패턴: POST 처리 후 `redirect:` 접두사로 GET으로 이동 → 새로고침 시 중복 제출 방지
- `@RequestParam(required=false, defaultValue="0")`: 쿼리 파라미터 선택적 수신

---

### Step 5. DeptAPIController.java — REST 컨트롤러 (강사 코드)

```java
@Slf4j
@RestController         // @Controller + @ResponseBody → 모든 메서드가 JSON 응답
@RequestMapping("dept/api")
public class DeptAPIController {

    private DeptService deptService;

    // 생성자 주입 방식 (Spring 공식 권장)
    public DeptAPIController(@Autowired DeptService deptService) {
        this.deptService = deptService;
    }

    // GET /dept/api/ → 목록 조회 (JSON)
    @GetMapping("/")
    public Map<String, Object> getAllDepts(Model model) {
        Map<String, Object> resultMap = new HashMap<>();
        resultMap.put("deptList", deptService.getDeptList());
        return resultMap;
        // ResponseEntity를 쓰면 HTTP 상태코드, 헤더까지 세밀하게 조정 가능
    }

    // GET /dept/api/{deptno} → 단건 조회
    @GetMapping("/{deptno}")
    public Dept getDeptInfo(@PathVariable("deptno") int deptno, Model model) {
        return deptService.getDeptInfo(deptno);
    }

    // POST /dept/api/ → 등록
    @PostMapping("/")
    public Map<String, Object> insertDept(Dept dept) {
        Map<String, Object> resultMap = new HashMap<>();
        try {
            deptService.insertDept(dept);
            resultMap.put("message", dept.getDeptNo() + "번 부서가 신규 등록되었습니다.");
        } catch (RuntimeException e) {
            resultMap.put("message", e.getMessage());
        }
        return resultMap;
    }

    // PUT /dept/api/ → 수정
    @PutMapping("/")
    public Map<String, Object> updateDept(Dept dept) { ... }

    // DELETE /dept/api/{deptno} → 삭제
    // GET처럼 payload 불가 → @PathVariable로 전달
    @DeleteMapping("/{deptno}")
    public Map<String, Object> deleteDept(@PathVariable("deptno") int deptno) { ... }
}
```

**포인트**
- `@RestController` = `@Controller` + `@ResponseBody` — 반환값이 자동으로 JSON 직렬화
- `DELETE`는 `GET`처럼 Request Body(payload)가 불가 → `@PathVariable` 또는 Query Parameter 사용
- `@GetMapping({"index", "view"})`: 하나의 메서드에 여러 URL 매핑 가능

---

## 결과 / 산출물

- `dept` 패키지 전체 계층 완성
  - SSR: `GET /dept/list`, `POST /dept/insert`, `POST /dept/update`, `POST /dept/delete`
  - REST API: `GET /dept/api/`, `GET /dept/api/{deptno}`, `POST`, `PUT`, `DELETE`

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 전체 계층 흐름을 순서대로 직접 작성하며 구조 파악 |  |
| `@Controller`와 `@RestController` 차이 코드로 확인 | `ResponseEntity` 직접 써보지 못함 |
| PRG 패턴 + `addFlashAttribute` 동작 방식 이해 | — |

---
*작성일: 2026-06-09 | 인공지능 사관학교 AI 보안 과정*
