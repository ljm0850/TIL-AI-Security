# 2026-05-22 심화 정리 — SQL DB 객체: VIEW·SP·FN

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-05-20_AI-Security-Course_Lecture_Oracle-SQL-Intro.md`
> **작성 계기**: 강의 중 "유지보수를 잘하려면 VIEW·SP·FN을 잘 다뤄야 한다"는 언급을 듣고, 세 객체의 차이와 실무 맥락을 정리하기 위해 작성

---

## 왜 찾아봤는가

강의에서 세 가지를 묶어서 언급했는데 "재사용"이라는 공통점 외에 무엇이 다른지, 왜 SP·FN은 줄어드는 추세인지 구체적으로 정리가 필요했다.

---

## 핵심 내용

### 세 객체의 재사용 대상

| 객체 | 재사용 대상 | 한 줄 비유 |
|---|---|---|
| VIEW | 조회 구조 | 가상 테이블 / SELECT 매크로 |
| SP (Stored Procedure) | 작업 절차 | DB 내부 API / Service Method |
| FN (Stored Function) | 계산 로직 | SQL 버전 Utility Function |

---

### VIEW

```sql
CREATE OR REPLACE VIEW vw_user_orders AS
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.user_id = o.user_id;

-- 사용
SELECT * FROM vw_user_orders;
```

**유지보수 장점**
- 복잡한 JOIN을 한 곳에서 관리 → 테이블 구조 변경 시 VIEW만 수정
- 앱 개발자는 복잡한 SQL 없이 단순 SELECT만 사용
- **보안**: 원본 테이블의 민감 컬럼(급여, 주민번호) 숨기기 → SQL 노출 시에도 원본 구조 보호

**유지보수 단점**
- VIEW 안에 VIEW를 중첩하면 추적이 매우 어려워짐 (`vw_a → vw_b → vw_c`)
- 단순해 보이는 SELECT 뒤에 복잡한 JOIN·GROUP BY가 숨어 있어 성능 문제를 놓치기 쉬움
- VIEW 자체가 성능을 빠르게 해주지는 않음 → 기반 쿼리 최적화가 핵심

**실무 권장**
- 공통 조회, 읽기 전용 데이터, 보안 목적: 적극 활용
- 복잡한 비즈니스 로직 or VIEW 중첩: 지양

---

### SP (Stored Procedure)

```sql
CREATE PROCEDURE usp_create_order
AS
BEGIN
    INSERT INTO orders ...;
    UPDATE inventory ...;
    UPDATE user_points ...;
    INSERT INTO logs ...;
END;

EXEC usp_create_order;
```

**유지보수 장점**
- 여러 DML 작업을 하나의 단위로 묶어 트랜잭션 관리 용이
- 데이터 가까이에서 처리 → 대량 데이터에 유리한 경우 있음
- 중요한 처리 로직을 DB에서 통합 관리

**유지보수 단점**
- 로직이 DB에 숨어 있어 앱 코드만 봐서는 동작 파악 어려움
- Git 버전 관리가 앱 코드에 비해 불편
- MSSQL/Oracle/MySQL마다 문법이 달라 DB 종속성 증가

**실무 권장**
- 배치 작업, 정산, 대량 데이터 처리, 트랜잭션이 중요한 작업에 제한적으로 사용
- 전체 비즈니스 로직을 SP에 몰아넣는 것은 지양

---

### FN (Stored Function)

```sql
CREATE FUNCTION fn_get_age(@birth DATE)
RETURNS INT
AS
BEGIN
    RETURN DATEDIFF(YEAR, @birth, GETDATE());
END;

-- SQL 문 내에서 직접 호출
SELECT dbo.fn_get_age(birth) FROM users;
```

**유지보수 장점**
- 반복되는 계산 공식을 한 곳에서 관리
- `fn_get_age(birth)`처럼 의미가 명확해 가독성 향상

**유지보수 단점**
- **Scalar Function**: 행마다 호출 → 데이터 양이 많으면 성능이 급격히 저하됨
- 겉으로는 단순해 보이지만 내부 연산이 복잡할 수 있어 성능 함정 주의

**실무 권장**
- 단순 계산, 공통 포맷, 재사용 계산식에 한정
- 대량 데이터를 다루는 복잡한 연산에는 지양

---

### 실무 트렌드 — 왜 SP·FN은 줄어드는가

과거에는 DB 중심 개발이 일반적이었다. 성능을 위해 최대한 DB 레이어에서 처리했기 때문에 SP·FN이 핵심이었다.

현재는 애플리케이션 중심 개발로 이동하고 있다. 주요 이유는 두 가지다.

첫째, ORM(JPA, Hibernate 등)의 보급으로 SQL을 직접 작성하지 않아도 복잡한 쿼리를 코드로 표현할 수 있게 됐다. 비즈니스 로직이 자연스럽게 애플리케이션 레이어로 올라왔다.

둘째, 마이크로서비스 아키텍처에서는 DB가 특정 서비스의 내부 구현 세부 사항으로 취급된다. DB에 로직이 있으면 서비스 분리와 독립 배포가 어려워진다.

결과적으로 VIEW는 보안과 조회 편의성 때문에 여전히 활발히 쓰이지만, SP와 FN은 레거시 시스템이나 대량 배치 처리 같은 특수 목적으로 제한되는 경향이 있다.

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 | 아직 모르는 것 |
|---|---|
| 세 객체가 재사용 대상이 다름을 구분 | ORM이 SP를 어디까지 대체하는지 실제 사례 |
| VIEW의 보안 활용 가치 | Scalar Function 성능 문제의 구체적인 수치 |
| SP·FN이 줄어드는 이유 (ORM, MSA 흐름) | 오라클에서 함수 인덱스로 FN 성능 개선하는 방법 |
| 트랜잭션 관리에서 SP가 여전히 쓰이는 맥락 | |

---

## 출처

- 오라클 SQL 입문 강의 자료 (2026-05-22)
- 강의 중 직접 정리한 메모

---
*작성일: 2026-05-22 | 인공지능 사관학교 AI 보안 과정*
