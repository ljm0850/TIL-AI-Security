# 2026-05-20 강의 정리 — 오라클 SQL 입문

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 데이터베이스 개념 / 오라클 SQL 입문 (데이터 조회 및 정렬)
> **관련 실습 파일**: Oracle SQL Developer — sys 계정 및 scott 계정 생성 후 실습

---

## 핵심 인사이트 모음

### 📅 2026-05-20

- **기획은 뜬구름을 물로 만드는 과정이다.** 지난 3일 프로젝트를 돌아보면, 막연하게 "AI로 뭔가 만들자"에서 시작해 페르소나 → HMW → MVP → PRD 순으로 좁혀가는 과정이 결국 추상적인 문제를 구체적인 기능으로 변환하는 추상화 작업이었다. 강의에서 나온 "문제 → 추상화 → 정의" 흐름이 그 과정과 정확히 겹쳤다.
- **데이터와 정보는 다르다.** 데이터는 세상의 모든 지칭 가능한 것이고, 정보는 그 중에서 유용하다고 판단해 필터링한 집합이다. SQL은 결국 데이터에서 정보를 뽑아내는 도구다.
- **개발자는 세상을 명사와 동사로 추상화한다.** 명사는 상태를 가지고, 동사는 그 상태를 변화시킨다. 불변 데이터는 파일/DB에, 가변 데이터는 메모리에 저장하는 것도 이 관점의 연장이다.
- **DBMS의 핵심은 공유다.** 파일이 기본적으로 배타적(한 번에 하나만 접근)인 반면, DBMS는 여러 클라이언트가 동시에 데이터를 다룰 수 있게 해주는 서버 역할을 한다. 공유가 가능해지면 보안이 그만큼 중요해진다.
- **대형 회사들이 비싼 DB를 쓰는 이유는 리스크 관리다.** 오픈소스 DB는 무료지만 DB가 깨졌을 때 대응할 공식 지원이 없다. 금융권·게임사 등 데이터 무결성이 생명인 곳에서는 그 리스크 비용이 라이선스 비용보다 크다.

### 📅 2026-05-21 (이어서)

- **SQL 실행 순서를 알면 쿼리 오류가 보인다.** FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY 순서를 이해하면 "왜 WHERE에서 집계 함수를 쓸 수 없는가", "왜 SELECT 별칭을 HAVING에서 못 쓰는가"가 자연스럽게 설명된다.
- **LIKE와 NOT은 편리하지만 성능 비용이 있다.** `%검색어%` 패턴은 인덱스를 타지 못하고 풀스캔을 유발한다. 편의를 위해 쓰기 전에 데이터 규모와 인덱스 구조를 먼저 고민해야 한다.
- **LPAD/RPAD의 마스킹 처리는 보안과 연결된다.** 단순한 문자열 채우기 함수처럼 보이지만, 개인정보(카드번호, 전화번호 등)를 화면에 표시할 때 일부를 `*`로 치환하는 마스킹의 기본 도구다. DB 레벨에서 마스킹을 처리하면 애플리케이션 코드까지 올라오기 전에 민감 정보를 보호할 수 있다.
- **PIVOT/UNPIVOT은 데이터 구조 변환이다.** 행↔열 변환을 자유롭게 할 수 있다는 건 단순히 보고서를 예쁘게 만드는 게 아니라, 데이터를 분석 목적에 맞게 재구성하는 능력이다.
- **GROUP BY + HAVING + ROLLUP의 조합이 실무 보고서의 핵심이다.** 단순 집계에서 소계·총계까지 한 번의 쿼리로 뽑아낼 수 있다는 것은, 애플리케이션 레이어에서 루프를 돌릴 필요 없이 DB가 직접 요약 정보를 만들어준다는 의미다.

---

## 1. 3일 프로젝트 리뷰 (강사님 피드백)

### 핵심 개념

- 발주·수주 방식으로 프로젝트를 진행한 이유: 소통 능력과 상대방을 설득하는 경험을 쌓기 위함
- 기획의 본질: 뜬구름(막연한 문제)을 물(구체적인 정의)로 만드는 과정

### 상세 내용

기획에서 추상화 → 정의의 흐름은 단순히 아이디어를 구체화하는 것이 아니라, 본질적인 문제가 무엇인지를 발굴하는 과정이다. 발주/수주 형태는 "내가 원하는 것을 말로 전달하고", "상대방이 원하는 것을 듣고 파악하는" 양방향 소통 훈련이다.

---

## 2. 데이터베이스란

### 핵심 개념

- 데이터베이스는 단순한 저장소가 아니라 "특정 조직의 업무 수행에 필요한 상호 관련된 데이터들의 집합"
- `데이터 ≠ 정보`: 데이터는 지칭 가능한 모든 것, 정보는 그 중 유용하다고 판단해 필터링한 집합

### 상세 내용

**DIKW 모델 — 데이터가 지혜로 변환되는 계층**

| 계층        | 설명                                                      |
| ----------- | --------------------------------------------------------- |
| Data        | 순수한 사실의 나열                                        |
| Information | 데이터 간 관계를 설정해 의미를 부여한 것 (JOIN 등)        |
| Knowledge   | 정보를 구조화하여 패턴과 원리를 발견한 것                 |
| Wisdom      | 지식을 바탕으로 미래를 예측하거나 올바른 판단을 내리는 것 |

SQL과 DBMS는 Data → Information 변환을 담당한다. INPUT(테이블의 Row들) → PROCESS(SELECT, WHERE, JOIN, GROUP BY) → OUTPUT(Result Set).

---

## 3. 추상화와 데이터 모델링

### 핵심 개념

- 문제를 정의하려면 추상화가 먼저다
- 추상화: 객체의 핵심 특징만 추려내는 것
- 개발자는 세상을 **명사(상태를 가진 것)**와 **동사(상태를 변화시키는 것)**로 추상화한다

### 상세 내용

- 화가는 시각적 정보를 추상화하고, 개발자는 명사와 동사로 추상화한다
- 명사의 상태는 두 종류: **불변성 데이터** (파일·DB 저장에 적합) / **가변성 데이터** (기본적으로 메모리에 저장)
- 파일은 기본적으로 **배타적** — 여러 프로그램이 동시에 다루는 것을 막는 구조
- DBMS: 데이터를 **공유**(서버 역할)하고 동시에 다루기 위한 소프트웨어
- 공유 → **보안**이 매우 중요해짐

---

## 4. 관계형 데이터베이스(RDBMS)

### 핵심 개념

- 데이터를 행(Row)과 열(Column)로 구성된 2차원 표(Table) 형태로 관리
- 현재 기업 데이터 관리의 표준, 특히 **데이터 무결성**이 핵심 가치

### 상세 내용

**DB 종류 선택의 현실**

- MySQL, MariaDB, MongoDB 등 오픈소스 DB가 많지만 금융권·게임회사 등 대형 회사는 잘 안 쓴다
- 이유: DB가 깨지거나 보안 문제 발생 시 공식 지원을 받을 수 없는 리스크 → 비싼 라이선스(Oracle 등)를 쓰는 것이 오히려 리스크 비용이 낮다

**관계형 모델의 3대 구성 요소**

- 구조(Structure): 데이터를 2차원 테이블로 표현
- 연산(Manipulation): SQL로 조회·삽입·수정·삭제
- 제약 조건(Integrity Constraints): 데이터 정확성을 지키는 규칙
  - 기본키(PK): 행을 유일하게 식별 (중복 불가, NULL 불가)
  - 외래키(FK): 타 테이블과 관계를 맺는 값 (참조 무결성 유지)

**데이터 기본 구조**

| 구조                    | 설명                                                         |
| ----------------------- | ------------------------------------------------------------ |
| Database (= Tablespace) | 연관된 테이블들의 집합체. 오라클에선 Tablespace라고 부르기도 함 |
| Table                   | 특정 주제의 데이터가 모인 2차원 구조                         |
| Row (Tuple)             | 개별 데이터 레코드 (가로 줄)                                 |
| Column (Attribute)      | 데이터의 속성과 타입 (세로 줄)                               |

---

## 5. DBMS와 세션(SESSION)

### 핵심 개념

- DBMS: DB가 저장소라면, DBMS는 그 저장소를 효율적으로 관리·운영하는 소프트웨어
- **세션(SESSION)**: 클라이언트와 서버 사이에서 사용자의 상태(state)를 유지하기 위한 서버 측 저장 공간 또는 연결 정보

### 상세 내용

**데이터베이스의 4가지 핵심 특징**

- 실시간 접근성: 요청 즉시 결과 제공
- 계속적인 변화: 삽입·삭제·수정을 통해 항상 최신 상태 유지
- 동시 공유: 여러 사용자가 동시에 같은 데이터 이용 가능
- 내용에 의한 참조: 저장 위치가 아닌 데이터 값으로 검색

**오라클 파일 구조**

```
oraclexe/app/oracle/oradata/XE/
```

- XE는 계정의 SID 이름
- 이 안의 데이터가 실제 핵심 데이터 → 해킹 시 가장 먼저 노리는 파일

**오라클 8080 포트 비활성화**

```sql
EXEC DBMS_XDB.SETHTTPPORT(0);
```

**세션 날짜 포맷 설정**

```sql
SELECT sysdate FROM dual;

SELECT *
FROM NLS_SESSION_PARAMETERS
WHERE PARAMETER = 'NLS_DATE_FORMAT';

ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS';
```

---

## 6. 데이터 조회 및 정렬 — SQL 기본 문법

### 핵심 개념

- **셀렉션(Selection)**: WHERE절로 특정 행(Row) 선택 — 가로 방향 필터
- **프로젝션(Projection)**: SELECT 컬럼 명시로 특정 열(Column) 선택 — 세로 방향 필터
- 실무에서는 `SELECT *` 대신 필요한 컬럼을 명시하는 프로젝션이 기본

### 상세 내용

**DB 구조 탐색**

```sql
SELECT * FROM TAB;        -- 현재 계정 소유 테이블 목록 (오라클 전용)
DESC [테이블명];           -- 컬럼명, 데이터 타입, NULL 허용 여부 확인
```

**DISTINCT — 중복 제거**

```sql
SELECT DISTINCT deptno FROM emp;
```

- SELECT 바로 다음에 위치, 선택된 모든 컬럼의 조합 기준으로 중복 체크
- 대량 데이터에서는 정렬/해싱 작업으로 성능 부하 발생 가능

**산술 연산과 별칭(Alias)**

```sql
SELECT ename, sal, sal * 12 + comm AS "Annual Salary" FROM emp;
```

- NULL이 포함된 산술 연산의 결과는 무조건 NULL → `NVL(comm, 0)`으로 치환 필요
- 별칭에 공백·특수문자 포함 시 큰따옴표(" ") 필수

**정렬 (ORDER BY)**

```sql
ORDER BY DEPTNO DESC, ENAME ASC;
```

- SQL 문장의 가장 마지막에 위치
- 생략 시 기본값은 오름차순(ASC)

**조건절 (WHERE) 및 비교 연산자**

```sql
WHERE DEPTNO = 30
WHERE ENAME = 'SMITH'       -- 문자열은 홑따옴표, 대소문자 구분
WHERE HIREDATE = '81/06/09'
```

| 연산자           | 의미                     |
| ---------------- | ------------------------ |
| `=`              | 같다                     |
| `!=`, `<>`, `^=` | 다르다                   |
| `>`, `<`         | 크다, 작다               |
| `>=`, `<=`       | 크거나 같다, 작거나 같다 |

---

## 📅 2026-05-21 이어서

---

## 7. SQL 논리와 데이터 가공

### 핵심 개념

- 논리 연산자(AND, OR, NOT)로 다중 조건을 결합해 정교한 필터링 구현
- AND가 OR보다 우선순위가 높으므로 복잡한 조건에는 반드시 괄호 사용

### 상세 내용

**논리 연산자**

```sql
-- AND: 교집합 — 모든 조건 참
WHERE DEPTNO = 30 AND JOB = 'SALESMAN'

-- OR: 합집합 — 하나라도 참
WHERE DEPTNO = 20 OR JOB = 'SALESMAN'

-- NOT: 부정 — 조건 반전
WHERE NOT DEPTNO = 10
```

**BETWEEN / IN / LIKE**

```sql
-- 범위 검색 (경계값 포함)
WHERE SAL BETWEEN 2000 AND 3000

-- 목록 검색 (OR 대체 — 체크박스 형태 검색에 활용)
WHERE JOB IN ('MANAGER', 'SALESMAN')

-- 패턴 매칭
WHERE ENAME LIKE 'S%'    -- S로 시작
WHERE ENAME LIKE '_L%'   -- 두 번째 글자가 L
```

**LIKE·NOT 성능 주의사항**

- `%검색어%` 패턴은 인덱스를 사용하지 못해 풀스캔 발생 → 데이터 규모가 클수록 성능 저하
- NOT 연산자도 인덱스 활용이 어렵기 때문에 사용 전 고민 필요
- 가능하면 `SUBSTR`, `INSTR` 등 문자열 함수 활용 또는 함수 인덱스 검토
- 자세한 내용: [`../deep-dive/2026-05-21_SQL-LIKE-vs-SUBSTR-Index.md`]

**NULL 처리**

```sql
WHERE COMM IS NULL       -- NULL 조회 (= NULL은 동작 안 함)
WHERE COMM IS NOT NULL   -- NULL이 아닌 행만 조회
```

**집합 연산자**

| 연산자    | 특징              | 비고                       |
| --------- | ----------------- | -------------------------- |
| UNION     | 합집합, 중복 제거 | 정렬 과정 포함 → 성능 소모 |
| UNION ALL | 합집합, 중복 포함 | 가장 빠름, 실무 권장       |
| MINUS     | 차집합            | 오라클 전용 (표준: EXCEPT) |
| INTERSECT | 교집합            | AND 연산과 유사한 결과     |

**DUAL 테이블**

- 오라클의 가상 1행 테이블 — FROM 절 문법을 만족시키기 위해 사용
- 함수 결과나 수식 테스트에 활용: `SELECT SYSDATE FROM dual;`

---

## 8. 단일행 함수

### 핵심 개념

- 단일행 함수: 행마다 개별 적용 → 입력 행 수 = 출력 행 수
- 다중행 함수(집계 함수): 여러 행을 모아 하나의 결과 반환

### 상세 내용

**문자열 함수**

```sql
-- 대소문자 변환 (검색 시 대소문자 통일에 필수)
WHERE UPPER(ename) = UPPER('검색어')

-- 길이 확인
SELECT LENGTH('한글'), LENGTHB('한글') FROM DUAL;
-- 결과: 2 (글자 수), 6 (AL32UTF8 기준 바이트 수)

-- 부분 문자열 추출 (인덱스 1부터 시작)
SELECT SUBSTR('20260427', 1, 4) FROM DUAL;  -- 2026
SELECT SUBSTR('ORACLE', -3) FROM DUAL;       -- CLE (뒤에서 3글자)

-- 문자 위치 검색
SELECT SUBSTR('admin@oracle.com', 1, INSTR('admin@oracle.com', '@') - 1) FROM DUAL;
-- 결과: admin

-- 문자 치환 (데이터 정제)
SELECT REPLACE('010-1234-5678', '-', '') FROM DUAL;  -- 01012345678

-- 마스킹 처리 (보안 활용)
SELECT RPAD('SCOTT', 10, '*') FROM DUAL;   -- SCOTT*****
SELECT LPAD('5000', 10, '0') FROM DUAL;    -- 0000005000

-- 공백 제거
SELECT TRIM(' ORACLE ') FROM DUAL;         -- 'ORACLE'
SELECT LTRIM('000123', '0') FROM DUAL;     -- 123
```

**숫자 처리 함수**

```sql
SELECT ROUND(12.56), ROUND(12.567, 2) FROM DUAL;          -- 13, 12.57
SELECT TRUNC(12345.67), TRUNC(12345.678, 2) FROM DUAL;    -- 12345, 12345.67
SELECT CEIL(3.1), FLOOR(3.9) FROM DUAL;                   -- 4, 3
SELECT MOD(10, 3) FROM DUAL;                              -- 1
```

**날짜 처리 함수**

```sql
SELECT SYSDATE FROM DUAL;
SELECT SYSDATE + 7 AS "일주일 후" FROM DUAL;
SELECT ADD_MONTHS(SYSDATE, 3) FROM DUAL;               -- 3개월 후
SELECT MONTHS_BETWEEN('2026-12-31', SYSDATE) FROM DUAL;
SELECT LAST_DAY('2026-02-01') FROM DUAL;               -- 2026-02-28
```

- 날짜에 숫자를 더하면 **일(Day)** 단위 계산
- 개월 단위 계산은 반드시 `ADD_MONTHS` 사용 (달마다 일수가 다르기 때문)
- 글로벌 서비스의 시간 처리(UTC): [`../deep-dive/2026-05-21_Global-Service-Timezone.md`]

**형변환 함수**

```sql
-- 날짜/숫자 → 문자 (출력 포맷 지정)
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM DUAL;
SELECT TO_CHAR(3000000, 'L999,999,999') FROM DUAL;    -- ₩3,000,000
SELECT TO_CHAR(SYSDATE, 'YYYY"년 "MM"월"') FROM DUAL; -- 2026년 05월

-- 문자 → 숫자
SELECT TO_NUMBER('123.45') + 10 FROM DUAL;            -- 133.45

-- 문자 → 날짜 (날짜 비교 시 필수)
SELECT * FROM emp WHERE hiredate > TO_DATE('1981/06/01', 'YYYY/MM/DD');
```

- 암시적 형변환은 성능 저하·오류 원인 → **명시적 형변환** 습관화

---

## 9. SQL 조건 제어 및 그룹 분석

### 핵심 개념

- `NVL` / `NVL2`: NULL을 기본값으로 치환
- `DECODE` / `CASE`: SQL 내에서 if-else 로직 구현
- `GROUP BY` + `HAVING`: 그룹화 및 그룹 결과 필터링
- SQL 실행 순서: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

### 상세 내용

**NULL 처리 함수**

```sql
-- NVL: NULL이면 대체값으로 치환
SELECT ename, sal + NVL(comm, 0) AS total_sal FROM emp;

-- NVL2: NULL 여부에 따라 각기 다른 값 반환 (IF-THEN-ELSE)
SELECT ename, NVL2(comm, '수당대상자(O)', '일반대상자(X)') AS status FROM emp;
```

**조건문 — DECODE vs CASE**

```sql
-- DECODE: 오라클 전용, 등가 비교(=)만 가능
SELECT ename, DECODE(deptno, 10, 'ACCOUNTING', 20, 'RESEARCH', 'OPERATIONS') AS dname
FROM emp;

-- CASE: ANSI 표준, 범위 비교·복합 조건 모두 가능 → 실무 권장
SELECT ename,
    CASE WHEN sal >= 3000 THEN 'VIP'
         WHEN sal >= 2000 THEN 'GOLD'
         ELSE 'SILVER'
    END AS grade
FROM emp;
```

**집계 함수 (다중행 함수)**

```sql
-- COUNT(*) = 전체 행 수, COUNT(컬럼) = NULL 제외 행 수
SELECT COUNT(*), COUNT(comm) FROM emp;

-- AVG(comm): 수당 있는 사람만의 평균
-- AVG(NVL(comm, 0)): 전체 사원 대비 평균 → 분모가 달라지므로 요건에 맞게 선택
SELECT AVG(comm), AVG(NVL(comm, 0)) FROM emp;
```

**GROUP BY & HAVING**

```sql
-- 부서별 평균 급여
SELECT deptno, AVG(sal) FROM emp GROUP BY deptno;

-- 평균 급여 2000 이상인 부서만
SELECT deptno, AVG(sal) FROM emp GROUP BY deptno HAVING AVG(sal) >= 2000;
```

**SQL 내부 실행 순서**

```
① FROM      — 테이블 로드
② WHERE     — 행 필터링 (집계 함수 사용 불가)
③ GROUP BY  — 그룹화
④ HAVING    — 그룹 결과 필터링
⑤ SELECT    — 출력 컬럼 결정
⑥ ORDER BY  — 정렬
```

---

## 10. 고급 그룹화 및 PIVOT

### 핵심 개념

- `ROLLUP`: 계층적 소계·총계 자동 산출
- `CUBE`: 모든 조합의 다차원 집계
- `GROUPING SETS`: 원하는 집계 조합만 선택
- `PIVOT`: 행 → 열 변환 / `UNPIVOT`: 열 → 행 변환

### 상세 내용

**ROLLUP / CUBE / GROUPING SETS**

```sql
-- ROLLUP: 오른쪽→왼쪽으로 그룹을 줄여가며 소계·총계
GROUP BY ROLLUP(deptno, job)

-- CUBE: 모든 조합의 집계
GROUP BY CUBE(deptno, job)

-- GROUPING SETS: 원하는 집계만 골라서 출력
GROUP BY GROUPING SETS(deptno, job)
```

**PIVOT / UNPIVOT**

```sql
-- PIVOT: 세로(행) → 가로(열) 변환
SELECT *
FROM (SELECT job, deptno, sal FROM emp)
PIVOT (
    SUM(sal)
    FOR deptno IN (10, 20, 30)
);

-- UNPIVOT: 가로(열) → 세로(행) 변환
SELECT job, deptno, sal
FROM dept_sal_report
UNPIVOT (sal FOR deptno IN (D10, D20, D30));
```

**OVER 절 — 행 수 유지하며 집계값 추가**

```sql
-- GROUP BY와 달리 원본 행을 유지하면서 집계 결과를 컬럼으로 추가
SELECT deptno, job, sal,
    SUM(sal) OVER(PARTITION BY deptno) AS 부서급여합계
FROM emp;
```

---

## 찾아본 것

| 주제                                   | 찾아본 날짜 | 내용 요약 또는 링크                                          | 출처                                                    |
| -------------------------------------- | ----------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| DIKW 모델                              | 2026-05-20  | Data → Information → Knowledge → Wisdom 계층 구조. SQL은 Data→Information 담당 | 강의 자료                                               |
| SQL LIKE vs SUBSTR 인덱스 탐색         | 2026-05-21  | 별도 deep-dive 파일 참고                                     | [`../deep-dive/2026-05-21_SQL-LIKE-vs-SUBSTR-Index.md`] |
| 글로벌 서비스 시간 처리 (UTC/Timezone) | 2026-05-21  | 별도 deep-dive 파일 참고                                     | [`../deep-dive/2026-05-21_Global-Service-Timezone.md`]  |

---

*작성일: 2026-05-20 ~ 2026-05-21 | 인공지능 사관학교 AI 보안 과정*
