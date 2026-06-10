# 2026-06-10 실습 — 바이브 코딩 (SpringBtWeb4)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (AI 활용 바이브 코딩)
> **소요 시간**: 오전 세션 대부분

---

## 느낀 것 / 배운 것

### 📅 2026-06-10

- 어제(SpringBtWeb3)는 빈 폴더에서 시작했다가 폴더 구조와 빌드 설정부터 문제가 터졌다. 오늘은 STS에서 Spring 프로젝트를 먼저 생성한 뒤 시작했고 훨씬 안정적이었다. AI에게 줄 시작점을 잘 만들어주는 것도 실력이다.
- 에러의 원인이 코드가 아니라 **문서**였다. Skill 문서에 Oracle 12c+ 기준 페이징 SQL(`OFFSET ... FETCH NEXT`)을 썼는데 실제 환경은 Oracle XE 11g였다. 문서가 틀리면 AI가 만드는 코드도 틀린다. 환경 정보를 정확히 기입하는 것이 Skill 문서 작성의 핵심이다.
- `CLAUDE.md`(공통 환경·원칙)와 `skill.md`(프로젝트별 스펙)를 분리하는 구조를 오늘 처음 제대로 써봤다. 공통 부분을 한 번만 쓰고 재사용할 수 있어서 깔끔했다. 팀으로 바이브 코딩을 한다면 이 두 문서를 맞추는 것 자체가 설계 회의가 되겠다는 생각이 들었다.
- `HANDOFF.md`가 실제로 작동하는 걸 봤다. 빌드 성공 후 다음 단계가 명확히 적혀 있으니 다음 세션에서 어디서부터 이어야 할지 고민할 필요가 없다. 내 프로젝트에도 이 방식을 적용해봐야겠다.

---

## 실습 개요

- **목표**: SpringBtWeb4 프로젝트를 바이브 코딩으로 생성, 오류 해결, 빌드 성공
- **방법**: STS에서 Spring 프로젝트 생성 → `CLAUDE.md` + `skill.md` 작성 → AI 코드 생성 → 오류 해결 → `HANDOFF.md` 갱신

---

## 진행 과정

### Step 1. 어제 실패 원인 분석 및 접근 방식 변경

**어제 문제 (SpringBtWeb3)**
- 빈 폴더에서 시작 → 폴더 구조, `build.gradle`, 패키지 설정부터 AI가 생성 → 빌드 환경 불일치로 오류 다수 발생

**오늘 변경 사항**
- STS에서 Spring Starter Project로 프로젝트 먼저 생성 (기본 뼈대 확보)
- 생성된 프로젝트를 AI에게 전달 → 기존 환경을 분석하게 한 뒤 코드 추가

---

### Step 2. CLAUDE.md 작성 (공통 환경 · 작업 원칙)

강사 제공 양식 기반으로 작성. 프로젝트 전반에 적용되는 공통 규칙을 담았다.

**핵심 내용**

```markdown
# CLAUDE.md
## 작업 원칙
- 작업 시작 시 반드시 HANDOFF.md를 먼저 읽고 이어서 진행
- 실행 테스트는 하지 않는다. 빌드/단위 테스트만 진행

## 환경
| Oracle | XE 11g (jdbc:oracle:thin:@localhost:1521:XE, SID 방식) |
| Java   | 17 (Eclipse Adoptium) |
| 빌드   | Gradle 8 (Wrapper), -parameters 플래그 필수 |
| IDE    | STS 5 (Eclipse 기반)                                   |
```

**`-parameters` 플래그 주의사항**
- STS는 기본적으로 `-parameters` 없이 컴파일
- `@RequestParam`, `@PathVariable`에 반드시 이름을 명시해야 함
  - `@RequestParam(name="page")` — 이름 미지정 시 런타임 오류

**MyBatis ↔ JPA 전환 구조**
```java
// RepositoryConfig.java — @Primary 한 줄만 옮기면 전환
@Primary  // ← MyBatis 사용. JPA로 바꾸려면 아래 메서드로 이동
@Bean
DeptRepository mybatisRepo(DeptMapper mapper) { return new DeptMyBatisRepository(mapper); }

@Bean
DeptRepository jpaRepo(DeptJpaDao dao) { return new DeptJpaRepository(dao); }
```

---

### Step 3. skill.md 작성 (프로젝트별 스펙)

`요청사항.txt` → `skill_temp.md` 양식 → `skill.md` 순서로 작성.

**어제(SpringBtWeb3)와 달라진 점**

| 항목 | SpringBtWeb3 | SpringBtWeb4 |
|---|---|---|
| Oracle 버전 | 미기재 (12c+ 기준 SQL 작성) | **11g 명시** → ROWNUM 이중 중첩 페이징 |
| IDE 특성 | 미기재 | **`-parameters` 미적용** 명시 |
| DTO/Entity 분리 | 없음 | **Request DTO / Response DTO 분리** |
| 예외 처리 | try-catch 개별 처리 | **`@RestControllerAdvice` 전역 처리** |
| API 명세 관리 | 없음 | `docs/api/openapi.yaml` + `changelog.md` |

**Oracle 11g 페이징 SQL (핵심 수정)**
```sql
-- Oracle 12c+: OFFSET ... ROWS FETCH NEXT ... ROWS ONLY (사용 불가)
-- Oracle 11g: ROWNUM 이중 중첩 방식 사용
SELECT DEPTNO, DNAME, LOC FROM (
    SELECT a.*, ROWNUM RN FROM (
        SELECT DEPTNO, DNAME, LOC FROM DEPT
        WHERE UPPER(DNAME) LIKE UPPER('%' || #{keyword} || '%')
        ORDER BY DEPTNO
    ) a WHERE ROWNUM <= :offset + :size
) WHERE RN > :offset
```

**CSS 속성 선택자 버튼 제어 (어제와 동일 방식 유지)**
```css
/* 미선택: 수정/삭제 비활성 */
#dept-toolbar[data-selected=""] [data-role="edit"],
#dept-toolbar[data-selected=""] [data-role="delete"] {
    pointer-events: none;
    opacity: .5;
}
/* 선택됨: 추가/검색 비활성 */
#dept-toolbar:not([data-selected=""]) [data-role="add"],
#dept-toolbar:not([data-selected=""]) [data-role="search"] {
    pointer-events: none;
    opacity: .5;
}
```

---

### Step 4. API 명세 관리 규칙 추가 (api_skill.md)

강사 제공 `api_skill.md`를 `CLAUDE.md`에 통합.

```markdown
## API 명세 관리
엔드포인트 추가/수정/삭제 시 반드시:
1. docs/api/openapi.yaml 읽기
2. 코드 변경 수행
3. openapi.yaml 업데이트
4. docs/api/changelog.md에 변경 내용과 날짜 기록
```

---

### Step 5. 빌드 성공 및 HANDOFF.md 갱신

**빌드 결과**: `./gradlew clean build` — BUILD SUCCESSFUL, 테스트 16/16 통과

**HANDOFF.md 최종 상태**

```markdown
## 성공한 것
- gradlew clean build 통과 (BUILD SUCCESSFUL)
- 16개 테스트 중 16개 모두 통과

## 다음 단계
1. Oracle DB 연결 후 실행 테스트 (application.yml 접속정보 수정 후 bootRun)
2. Eclipse에서 Import > Existing Gradle Project로 import 확인
3. H2 MODE=Oracle 미지원 SQL 구문 있을 경우 실제 Oracle 실행 시 수정 필요
```

---

## 결과 / 산출물

- `CLAUDE.md` — 공통 환경·원칙 문서
- `skill.md` — SpringBtWeb4 프로젝트 스펙 문서
- `HANDOFF.md` — 작업 이력 및 다음 단계
- SpringBtWeb4 프로젝트 빌드 성공

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| STS로 먼저 프로젝트 생성 후 시작 → 어제보다 훨씬 안정적 | CLAUDE.md/skill.md 작성 시간이 생각보다 많이 걸림 |
| Oracle 11g 환경을 문서에 정확히 기입 → 페이징 SQL 오류 사전 방지 |  |
| HANDOFF.md로 다음 세션 이어갈 지점 명확히 정리 |  |

---
*작성일: 2026-06-10 | 인공지능 사관학교 AI 보안 과정*
