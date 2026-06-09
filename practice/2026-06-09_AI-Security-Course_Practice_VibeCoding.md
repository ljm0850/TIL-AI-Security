# 2026-06-09 실습 — 바이브 코딩 (SpringBtWeb3 Skill 문서 작성)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (AI 활용 바이브 코딩)
> **소요 시간**: 약 1시간

---

## 느낀 것 / 배운 것

### 📅 2026-06-09

- 강사님이 `요청사항.txt`와 `skill_temp.md`를 제공해주셨다. `요청사항.txt`를 보고 `skill_temp.md` 양식에 맞춰 `SpringBtWeb3.md`를 작성한 뒤, 이를 기반으로 바이브 코딩을 진행했다.
- **예전 내가 바이브 코딩으로 프로젝트를 할 때 작성했던 문서들이 바로 생각났다.** 그때는 할 일 목록 수준으로 쪼개는 게 전부였다. 디자인 패턴, 아키텍처 결정, DTO 분리 기준, 예외 처리 전략 같은 것들이 없었다. 그래서 진행하면서 문제가 계속 터졌는데, 오늘 `SpringBtWeb3.md`를 읽으면서 내가 뭘 빠뜨렸는지 바로 보였다.
- Skill 문서에서 특히 인상 깊었던 것들: `HANDOFF.md` 규칙(작업 시작 시 반드시 읽고 이어 진행), 소스 파일 헤더 규칙(작성자/생성일/설명), MyBatis/JPA 구현체를 `application.yml` 설정값으로 선택하는 구조. 이런 것들이 없으면 AI가 맥락 없이 코드를 생성하고, 나도 어디까지 됐는지 잃어버린다.
- "제약이 설계다"라는 말이 다시 떠올랐다. Skill 문서가 길고 제약이 많을수록 AI가 더 일관된 코드를 만들어낸다. 자유도가 높으면 매 요청마다 다른 스타일이 나온다.

---

## 실습 개요

- **목표**: 요청사항을 기반으로 바이브 코딩용 Skill 문서 작성, AI 코드 생성 실습
- **방법**: `요청사항.txt` → `skill_temp.md` 양식으로 `SpringBtWeb3.md` 작성 → AI에 Skill 문서 전달하여 코드 생성

---

## 진행 과정

### Step 1. 요청사항 분석

강사 제공 `요청사항.txt` 핵심 내용:

- **프로젝트명**: SpringBtWeb3
- **목적**: Spring Boot + DB를 이용한 MVP 수준 SPA
- **기능**: DEPT 테이블 CRUD, 부서명 검색, 페이징(최대 20개), 버튼 활성화 제어
- **기술**: Java 17, Spring Boot, Gradle, MyBatis + JPA(선택 구조), Lombok, Vue.js 3 CDN, Bootstrap CDN
- **요구조건**: MVC + SRP, RESTful, DTO 분리, HANDOFF.md, 소스 헤더, 보안 테스트

### Step 2. Skill 문서 작성 (`SpringBtWeb3.md`)

`skill_temp.md` 양식에 맞춰 아래 섹션 작성:

**아키텍처 결정 사항**
- MyBatis/JPA 구현체 선택 구조: `DeptRepository` 인터페이스 → `DeptMyBatisRepository` / `DeptJpaRepository` 각각 구현 → `application.yml`의 `repository.type` 설정으로 선택
- Controller는 Entity를 직접 반환하지 않음 → Request DTO / Response DTO 분리
- `@ConditionalOnProperty`로 구현체 자동 주입

**REST API 명세 (확정)**

| Method | URI | 설명 |
|---|---|---|
| GET | `/api/depts` | 목록 조회 (페이징) |
| GET | `/api/depts/{deptno}` | 단건 조회 |
| POST | `/api/depts` | 신규 등록 |
| PUT | `/api/depts/{deptno}` | 수정 |
| DELETE | `/api/depts/{deptno}` | 삭제 |
| GET | `/api/depts/search?dname=` | 부서명 검색 |

**예외 처리 전략**
- `@RestControllerAdvice`의 `GlobalExceptionHandler` 사용
- 중복 deptno → 409 CONFLICT
- 존재하지 않는 deptno 조회 → 404 NOT_FOUND
- 입력값 검증 실패 → 400 BAD_REQUEST

**프론트엔드 버튼 활성화 규칙**

| 상태 | 추가 | 검색 | 수정 | 삭제 |
|---|---|---|---|---|
| 데이터 미선택 | 활성화 | 활성화 | 비활성화 | 비활성화 |
| 데이터 선택 | 비활성화 | 비활성화 | 활성화 | 활성화 |

- 조건문(`v-if`)보다 CSS 속성 선택자(`attribute selector`)로 제어 — 상태를 HTML 속성에 반영하고 CSS가 그것을 읽는 방식

**HANDOFF.md 규칙** (작업 인수인계 문서)
```markdown
## 작업 요약
## 현재 상태
## 시도한 작업
## 성공한 작업
## 실패한 작업
## 이슈 및 원인
## 다음 작업
## 주의사항
```

**소스 파일 헤더 규칙** (모든 Java, JS, CSS 파일 상단)
```
작성자:
생성일:
설명:
```

### Step 3. AI 바이브 코딩 진행

작성한 `SpringBtWeb3.md`를 AI에 전달하여 프로젝트 코드 생성 실습.

---

## 결과 / 산출물

- `SpringBtWeb3.md` — 바이브 코딩용 Skill 문서 완성
- AI가 생성한 프로젝트 뼈대 코드 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| Skill 문서의 제약 조건이 많을수록 AI 코드 일관성이 높아진다는 것 체감 |
| 과거 바이브 코딩의 문제점(디자인 패턴 미기입)이 무엇이었는지 명확히 파악 |
| HANDOFF.md 방식으로 AI 작업 맥락을 유지하는 방법 학습 |

---
*작성일: 2026-06-09 | 인공지능 사관학교 AI 보안 과정*
