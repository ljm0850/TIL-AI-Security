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

### 📅 2026-05-22 (이어서)

- **JOIN은 "옆으로 붙이는 것"이다.** EMP 테이블에 DEPTNO가 있어도 부서명을 볼 수 없다. JOIN은 그 부서명을 DEPT 테이블에서 찾아 옆에 붙여주는 연산이다. 이 감각을 잡으면 어떤 테이블을 어떤 키로 연결할지가 자연스럽게 보인다.
- **INNER JOIN은 교집합이고, OUTER JOIN은 기준 테이블의 전체다.** LEFT OUTER JOIN은 왼쪽 테이블의 모든 행을 보여주고 오른쪽에 매칭이 없으면 NULL로 채운다. "어느 테이블이 기준인가"가 JOIN 선택의 핵심 판단 기준이다.
- **다중 열 서브쿼리는 "동시에 만족"을 표현한다.** `WHERE sal IN (SELECT MAX(sal) ...)` 는 다른 부서의 최고 급여와 우연히 같은 사원까지 잡아낸다. `WHERE (deptno, sal) IN (...)`처럼 복합 조건으로 묶어야 진짜 "그 부서의 최고 급여 사원"을 정확히 뽑을 수 있다.
- **트랜잭션은 DB 신뢰성의 최소 단위다.** COMMIT 전까지는 나만 보이고, 남은 못 본다. 이 고립성이 Lock으로 구현되고, Lock이 길어지면 데드락이 된다. ACID를 "개념"이 아니라 "잠금의 작동 원리"로 이해하면 실무에서 왜 트랜잭션을 짧게 유지해야 하는지가 납득된다.
- **VARCHAR2는 가변이라 효율적으로 보이지만, 값이 자주 바뀌면 단편화가 생긴다.** 데이터가 늘어날 때 원래 자리에 못 들어가면 흩어져 저장되고 탐색이 느려진다. CHAR는 고정 크기라 항상 같은 자리를 쓴다. 잘 변하지 않는 코드성 데이터에는 CHAR가 더 적합한 이유가 여기에 있다.
- **VIEW는 보안 도구다.** 복잡한 JOIN을 숨기는 편의성도 있지만, SQL 노출 시 원본 테이블 구조와 민감 컬럼을 숨길 수 있는 보안적 가치가 크다. AI 보안 과정에서 DB 보안을 생각할 때 VIEW는 컬럼 수준 접근 제어의 수단이 된다.
- **VIEW·SP·FN은 모두 "재사용"이지만 재사용 대상이 다르다.** VIEW는 조회 구조, SP는 작업 절차, FN은 계산 로직을 재사용한다. 최근 실무는 DB 중심 → 애플리케이션 중심으로 이동 중이라 SP·FN 사용이 줄고 있지만, VIEW는 여전히 핵심이다.

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

---

## 📅 2026-05-22 이어서

---

## 11. JOIN — 테이블 결합

### 핵심 개념

- JOIN: 물리적으로 분리된 테이블들을 논리적 연관성에 따라 결합해 실행 시점에 메모리(PGA) 상에 임시 결과 집합을 생성하는 연산
- **테이블 N개를 조인하려면 조인 조건이 최소 N-1개 필요** — 조건이 부족하면 카테시안 곱(Cartesian Product) 발생
- ANSI JOIN(`JOIN ... ON`) 사용 권장 — DB 종속성 없이 어디서나 동작

### 상세 내용

**JOIN 유형 요약**

| 유형                | 설명                                       | 특징                                           |
| ------------------- | ------------------------------------------ | ---------------------------------------------- |
| INNER JOIN (등가)   | 양쪽에 모두 존재하는 행만 반환             | 가장 많이 사용                                 |
| INNER JOIN (비등가) | `BETWEEN` 등 범위 조건으로 결합            | 급여 등급 산정 등                              |
| LEFT OUTER JOIN     | 왼쪽 테이블 전체 + 오른쪽 매칭 없으면 NULL | 기준 테이블이 핵심                             |
| RIGHT OUTER JOIN    | 오른쪽 테이블 전체 + 왼쪽 매칭 없으면 NULL |                                                |
| FULL OUTER JOIN     | 양쪽 모두 전체 출력                        | 오라클 `(+)` 문법으로는 불가, ANSI 표준만 가능 |
| SELF JOIN           | 같은 테이블을 두 번 불러 계층 구조 처리    | 반드시 별칭 사용                               |
| CROSS JOIN          | 조인 조건 없이 모든 행 조합 (카테시안 곱)  | 테스트 데이터 생성용                           |

**INNER JOIN 문법 비교**

```sql
-- ANSI 표준 (권장)
SELECT e.ename, d.dname
FROM emp e
INNER JOIN dept d ON e.deptno = d.deptno;

-- 오라클 구문
SELECT e.ename, d.dname
FROM emp e, dept d
WHERE e.deptno = d.deptno;
```

**비등가 조인 — 급여 등급 산정**

```sql
SELECT e.ename, e.sal, s.grade
FROM emp e
JOIN salgrade s ON e.sal BETWEEN s.losal AND s.hisal;
```

**OUTER JOIN — 오라클 `(+)` 주의사항**

```sql
-- LEFT OUTER: 없는 쪽(오른쪽)에 (+) 붙임
WHERE e.deptno = d.deptno(+)

-- FULL OUTER JOIN은 (+) 양쪽 사용 불가 → ANSI 표준 문법만 가능
SELECT e.ename, d.dname
FROM emp e
FULL OUTER JOIN dept d ON e.deptno = d.deptno;
```

**SELF JOIN — 상사 이름 찾기**

```sql
-- INNER JOIN: 상사가 없는 KING 제외
SELECT e.empno, e.ename, m.ename AS manager
FROM emp e
JOIN emp m ON e.mgr = m.empno;

-- LEFT OUTER JOIN: 상사 없는 사원(KING)까지 포함
SELECT e.empno, e.ename, m.ename AS manager
FROM emp e
LEFT OUTER JOIN emp m ON e.mgr = m.empno;
```

**다중 조인 — 순차 처리 주의**

```sql
-- 모든 부서·사원·급여등급·상사명 출력
-- DBMS는 (A와 B) 먼저 조인 → 그 결과와 C 조인 순서로 처리
-- 첫 번째 조인에서 누락된 데이터는 이후 OUTER로 살릴 수 없음
SELECT d.dname, e.ename, s.grade, e2.ename AS manager_name
FROM dept d
LEFT OUTER JOIN emp e ON d.deptno = e.deptno
LEFT OUTER JOIN salgrade s ON e.sal BETWEEN s.losal AND s.hisal
LEFT OUTER JOIN emp e2 ON e.mgr = e2.empno;
```

**JOIN + GROUP BY + ROLLUP 결합**

```sql
-- 부서명(DNAME)·직무(JOB) 기준 급여 합계 + 부서 소계 + 전체 총계
SELECT d.dname, e.job, SUM(e.sal) AS salary_sum
FROM emp e
JOIN dept d ON e.deptno = d.deptno
GROUP BY ROLLUP(d.dname, e.job);
-- 핵심: GROUP BY 먼저 해야 조인 결과를 바로 집계할 수 있어 연산이 줄어듦
```

---

## 12. 서브쿼리 (Subquery)

### 핵심 개념

- 서브쿼리는 괄호`()`로 감싸며, 메인 쿼리보다 먼저 실행되어 결과를 전달하는 **선행 단계**
- 단일행 서브쿼리: 결과 1행 → `=`, `>` 등 단일 비교 연산자 사용
- 다중행 서브쿼리: 결과 여러 행 → `IN`, `ANY`, `ALL` 연산자 사용

### 상세 내용

**다중행 서브쿼리 — IN / ANY / ALL**

```sql
-- IN: 목록 중 하나라도 일치
WHERE sal IN (SELECT MAX(sal) FROM emp GROUP BY deptno)
-- ⚠️ 문제: 다른 부서의 최고 급여와 우연히 같은 사원까지 포함됨

-- ANY: 최소 하나보다 크면 참 (= 최솟값보다 큰 것)
WHERE sal > ANY (SELECT sal FROM emp WHERE deptno = 30)

-- ALL: 모두보다 커야 참 (= 최댓값보다 큰 것)
WHERE sal > ALL (SELECT sal FROM emp WHERE deptno = 30)
-- ⚠️ ALL은 서브쿼리 결과에 NULL이 있으면 결과 없음
```

**다중 열 서브쿼리 — "동시에 만족" 표현**

```sql
-- 부서별 최고 급여 사원 정확하게 조회
-- IN만 쓰면: 다른 부서에서 동일 금액 받는 사원도 포함됨
-- (deptno, sal) 쌍으로 묶어야 "그 부서의 최고 급여 사원"을 정확히 특정

SELECT ename, deptno, sal
FROM emp
WHERE (deptno, sal) IN (
    SELECT deptno, MAX(sal) FROM emp GROUP BY deptno
);

-- 또는 인라인 뷰 방식 (더 명시적)
SELECT e.deptno, e.ename, e.sal
FROM emp e
JOIN (
    SELECT deptno, MAX(sal) AS max_sal
    FROM emp
    GROUP BY deptno
) m ON e.deptno = m.deptno AND e.sal = m.max_sal;
```

**인라인 뷰(Inline View)와 WITH 절(CTE)**

```sql
-- 인라인 뷰: FROM 절 안의 일회용 서브쿼리 (쿼리 실행 동안만 존재)
-- 조인 전 미리 데이터를 줄여 성능 최적화에 유용
SELECT e.ename, d.dname, e.sal
FROM (SELECT * FROM emp WHERE sal >= 2000) e
JOIN dept d ON e.deptno = d.deptno;

-- WITH 절(CTE): 서브쿼리에 이름 부여 → 반복 사용 + 가독성 향상
-- 오라클은 반복 참조 시 임시 저장(Materialize)으로 성능 향상
-- 재귀 쿼리(조직도, 트리 구조)에도 활용
WITH dept_avg AS (
    SELECT deptno, AVG(sal) AS avg_sal
    FROM emp
    GROUP BY deptno
)
SELECT e.ename, e.sal, a.avg_sal
FROM emp e
JOIN dept_avg a ON e.deptno = a.deptno
WHERE e.sal > a.avg_sal;
```

---

## 13. DML(데이터 조작어)과 트랜잭션

### 핵심 개념

- DML: 테이블 데이터를 생성(INSERT)·수정(UPDATE)·삭제(DELETE)하는 명령어
- **UPDATE/DELETE는 WHERE 없으면 전체 데이터 변경/삭제** — 실행 전 반드시 SELECT로 대상 확인
- 트랜잭션: DB 신뢰성을 보장하는 최소 작업 단위. COMMIT 전까지는 나만 볼 수 있고, 롤백 가능

### 상세 내용

**INSERT / UPDATE / DELETE 기본**

```sql
-- INSERT
INSERT INTO t_user (user_id, user_name, join_date)
VALUES ('USER01', '홍길동', SYSDATE);

-- 대량 삽입 (Subquery INSERT) — 데이터 이관·백업에 활용
INSERT INTO t_user_backup
SELECT * FROM t_user WHERE join_date < '2025-01-01';

-- UPDATE
UPDATE t_user
SET user_grade = 'VIP', point = point + 1000
WHERE user_id = 'USER01';

-- Subquery UPDATE — 다른 테이블 값을 참조해 반영
UPDATE t_user u
SET u.status = 'ACTIVE'
WHERE EXISTS (SELECT 1 FROM t_order o WHERE o.user_id = u.user_id);

-- DELETE
DELETE FROM t_user
WHERE last_login < ADD_MONTHS(SYSDATE, -12); -- 1년 이상 미접속자

-- TRUNCATE: 구조는 유지, 데이터만 전체 삭제. DELETE보다 빠르지만 ROLLBACK 불가
TRUNCATE TABLE t_user;
```

**NULL 주의 — 문자열 'null'과 실제 NULL은 다르다**

```sql
INSERT INTO dept_temp (deptno, dname, loc) VALUES (80, 'null', null);
-- dname = 'null' (문자열), loc = NULL (진짜 NULL)
SELECT * FROM dept_temp WHERE dname IS NULL; -- 조회 안 됨: dname은 문자열 'null'
```

**트랜잭션 ACID와 TCL**

| ACID                | 설명                                                      |
| ------------------- | --------------------------------------------------------- |
| 원자성(Atomicity)   | All or Nothing — 하나라도 실패하면 전체 취소              |
| 일관성(Consistency) | 완료 후에도 제약 조건 유지                                |
| 고립성(Isolation)   | 진행 중인 트랜잭션의 중간 상태는 다른 세션에서 볼 수 없음 |
| 지속성(Durability)  | 커밋된 결과는 장애 후에도 영구 보존                       |

```sql
-- TCL (트랜잭션 제어어)
COMMIT;                 -- 영구 반영 (한 번 커밋하면 ROLLBACK 불가)
ROLLBACK;               -- 마지막 COMMIT 이후 전체 취소 + Lock 해제
SAVEPOINT sp_01;        -- 중간 지점 설정
ROLLBACK TO sp_01;      -- 해당 지점까지만 부분 취소

-- 예시: SAVEPOINT 활용
INSERT INTO orders VALUES (101, '상품A', 2000);
SAVEPOINT sp_01;
UPDATE users SET point = 0 WHERE user_id = 'KIM'; -- 실수
ROLLBACK TO sp_01;      -- sp_01 이후만 취소 (INSERT는 유지)
COMMIT;
```

**세션·Lock·데드락**

- DB는 **공유** 자원이라 A 세션이 특정 행을 수정 중이면 B 세션은 그 행에 접근 불가 (행 레벨 락)
- Lock은 COMMIT 또는 ROLLBACK 시 해제 → 트랜잭션이 짧을수록 Lock 점유 시간 감소
- **데드락**: A가 1번 행을 잠근 채 2번 행을 원하고, B가 2번 행을 잠근 채 1번 행을 원할 때 발생 → 오라클이 자동 감지 후 한쪽을 강제 취소(`ORA-00060`)
- 예방: 일관된 작업 순서 유지 + 트랜잭션 최소화 + 사용자 입력 대기 중 Lock 금지

---

## 14. DDL(데이터 정의어)과 DB 객체

### 핵심 개념

- DDL은 실행 즉시 **Auto Commit** → ROLLBACK 불가. 실행 전 설계도 완벽 검토 필수
- DDL 실행 시 이전의 모든 DML 작업도 함께 자동 커밋됨

### 상세 내용

**주요 DDL 명령어**

```sql
-- CREATE
CREATE TABLE t_member (
    mem_no    NUMBER(10)    PRIMARY KEY,
    mem_id    VARCHAR2(20)  NOT NULL UNIQUE,
    mem_age   NUMBER(3)     CHECK(mem_age > 0),
    join_date DATE          DEFAULT SYSDATE,
    dept_id   NUMBER(10)    REFERENCES t_dept(dept_id)
);

-- ALTER: 구조 변경
ALTER TABLE t_student ADD    (email VARCHAR2(100));       -- 컬럼 추가
ALTER TABLE t_student MODIFY (std_name VARCHAR2(100));    -- 타입/길이 변경
ALTER TABLE t_student RENAME COLUMN birth TO birthday;    -- 컬럼명 변경
ALTER TABLE t_student DROP COLUMN email;                  -- 컬럼 삭제

-- DROP / TRUNCATE
DROP TABLE 테이블명;      -- 테이블 자체 삭제
TRUNCATE TABLE 테이블명;  -- 구조 유지, 데이터만 전체 삭제 (ROLLBACK 불가)
```

**VARCHAR2 vs CHAR — 단편화 주의**

- `VARCHAR2(10)`: 실제 데이터 길이만큼 저장 → 공간 효율적
- `CHAR(10)`: 항상 10자리 고정 저장 → 공간 낭비지만 갱신 시 단편화 없음
- **값이 자주 바뀌는 VARCHAR2 컬럼**: 데이터가 늘어나면 원래 자리에 못 들어가 흩어져 저장됨 → 탐색 시 연속되지 않은 블록을 읽어야 해서 성능 저하
- 결론: 길이가 고정이고 잘 변하지 않는 코드성 데이터(`Y/N`, `01/02` 등)에는 CHAR가 적합

**주요 데이터 타입**

| 타입           | 설명                                           | 용도                 |
| -------------- | ---------------------------------------------- | -------------------- |
| `NUMBER(p, s)` | 정수·실수 통합. p=전체 자릿수, s=소수점 자릿수 | 금액, 수량           |
| `VARCHAR2(n)`  | 가변 길이 문자열                               | 이름, 주소 등 대부분 |
| `CHAR(n)`      | 고정 길이 문자열                               | 코드, 플래그         |
| `DATE`         | 연월일시분초                                   | 일반 날짜            |
| `TIMESTAMP`    | 밀리초 + 시간대 포함                           | 금융 로그, 정밀 시각 |
| `CLOB`         | 4GB 텍스트                                     | 게시판 본문, 약관    |
| `BLOB`         | 이진 데이터                                    | 이미지, 동영상       |

**인덱스 (Index)**

- 풀스캔을 피하기 위해 특정 컬럼을 정렬해 별도 관리하는 B-Tree 구조 객체
- PK·UNIQUE 제약 조건 설정 시 자동 생성
- 검색 성능 향상 ↔ INSERT/UPDATE/DELETE 시 인덱스도 갱신되므로 수정 성능 저하

```sql
CREATE INDEX idx_emp_sal ON emp(sal);
```

| 스캔 방식         | 조건                              | 성능                          |
| ----------------- | --------------------------------- | ----------------------------- |
| Index Unique Scan | `WHERE pk = 값`                   | 가장 빠름                     |
| Index Range Scan  | `WHERE sal BETWEEN 3000 AND 5000` | 일반적, 효율적                |
| Index Full Scan   | 인덱스 전체를 읽어야 할 때        | 테이블 풀스캔보다 낫지만 느림 |

---

## 15. VIEW·SP·FN — 재사용성과 유지보수

### 핵심 개념

- 세 객체는 모두 "재사용"을 위한 DB 객체지만 재사용 대상이 다르다
- **VIEW = 조회 구조 재사용 / SP = 작업 절차 재사용 / FN = 계산 로직 재사용**
- 최근 실무 트렌드: DB 중심 → 애플리케이션 중심으로 이동 중. VIEW는 여전히 활발, SP·FN은 줄어드는 추세

### 상세 내용

**VIEW (뷰)**

```sql
-- 생성·수정 (OR REPLACE: 동일 이름 존재 시 삭제 후 재생성)
CREATE OR REPLACE VIEW v_emp_details AS
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    d.department_name,
    e.salary * 12 AS annual_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- 사용 — 복잡한 JOIN 없이 단 한 줄로
SELECT * FROM v_emp_details WHERE department_name = 'IT';

-- 삭제 (원본 테이블 데이터는 삭제되지 않음)
DROP VIEW v_emp_details;
```

- **보안**: 원본 테이블의 민감 컬럼(급여, 주민번호)을 숨기고 허용된 컬럼만 노출 → **SQL 노출 시에도 원본 구조 보호**
- **편의성**: 복잡한 JOIN을 캡슐화해 앱 개발자는 단순 SELECT만 사용
- **논리적 독립성**: 원본 테이블 구조 변경 시 VIEW만 수정하면 앱 코드는 그대로
- **주의**: VIEW 자체가 성능을 빠르게 해주지는 않음 → 기반 쿼리 최적화 필수. VIEW 안에 VIEW 중첩은 추적 어려워짐

**SP (Stored Procedure)**

- 여러 DML 작업(주문 생성 → 재고 감소 → 포인트 적립)을 하나의 단위로 묶어 DB에 저장
- 트랜잭션 관리와 대량 데이터 처리에 강점
- 단점: 로직이 DB에 숨어 코드 파악 어려움 + Git 관리 불편 + DB 종속성 증가

**FN (Stored Function)**

- 단일 값을 반환하는 재사용 계산 로직. SQL 문 내에서 직접 호출 가능
- 단점: Scalar Function은 행마다 호출 → 대량 데이터에서 성능 급감 주의

**실무 선택 기준 요약**

| 객체 | 재사용 대상 | 적합한 상황                    | 주의                                      |
| ---- | ----------- | ------------------------------ | ----------------------------------------- |
| VIEW | 조회 구조   | 공통 조회, 보안 컬럼 숨기기    | VIEW 중첩, 성능은 기반 쿼리에 달림        |
| SP   | 작업 절차   | 배치, 정산, 트랜잭션 중요 작업 | 로직 분산, 버전 관리 어려움               |
| FN   | 계산 로직   | 단순 반복 계산, 공통 포맷      | 대량 데이터에서 Scalar Function 성능 주의 |

---

## 아직 찾아보지 않은 것

- B-Tree 인덱스 vs Bitmap 인덱스 — 오라클에서 언제 Bitmap을 선택하는가
- 실행 계획(Execution Plan) 직접 읽는 법 — `EXPLAIN PLAN FOR` 사용법
- SP가 줄어드는 구체적인 이유 — ORM(JPA 등)과의 관계

---

## 찾아본 것

| 주제                                   | 찾아본 날짜 | 내용 요약 또는 링크                                          | 출처                                                    |
| -------------------------------------- | ----------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| DIKW 모델                              | 2026-05-20  | Data → Information → Knowledge → Wisdom 계층 구조. SQL은 Data→Information 담당 | 강의 자료                                               |
| SQL LIKE vs SUBSTR 인덱스 탐색         | 2026-05-21  | 별도 deep-dive 파일 참고                                     | [`../deep-dive/2026-05-21_SQL-LIKE-vs-SUBSTR-Index.md`] |
| 글로벌 서비스 시간 처리 (UTC/Timezone) | 2026-05-21  | 별도 deep-dive 파일 참고                                     | [`../deep-dive/2026-05-21_Global-Service-Timezone.md`]  |
| VIEW·SP·FN 유지보수 관점 정리          | 2026-05-22  | 별도 deep-dive 파일 참고                                     | [`../deep-dive/2026-05-22_SQL-VIEW-SP-FN.md`]           |

---

*작성일: 2026-05-20 ~ 2026-05-22 | 인공지능 사관학교 AI 보안 과정*
