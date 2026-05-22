# 2026-05-21 심화 정리 — SQL LIKE vs SUBSTR 인덱스 탐색 차이

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-05-20_AI-Security-Course_Lecture_Oracle-SQL-Intro.md`
> **작성 계기**: 강의에서 LIKE와 NOT은 성능에 주의하라고 했는데, 정확히 왜 느린지 / SUBSTR과 어떻게 다른지 궁금해서 찾아봄

---

## 왜 찾아봤는가

강의에서 `LIKE '%검색어%'` 패턴이 느리다고 했고, 대안으로 문자열 함수(SUBSTR, INSTR 등) 활용을 언급했다. 그런데 SUBSTR도 함수인데 왜 LIKE보다 나은 경우가 있는지, 반대로 왜 더 나쁠 수 있는지 기준이 없었다. 성능 차이의 근본 원인을 이해하고 싶었다.

---

## 핵심 내용

### 성능 차이의 핵심: 인덱스를 탈 수 있는가

B-Tree 인덱스는 컬럼 원본값을 **앞에서부터 정렬**해서 저장한다. 따라서 **앞부분이 고정된 검색**은 인덱스를 활용할 수 있고, **앞부분이 가변적인 검색**은 인덱스를 활용하기 어렵다.

---

### LIKE 패턴별 인덱스 사용 여부

| 조건 | 인덱스 사용 | 이유 |
|---|---|---|
| `LIKE 'kim%'` | 가능 (Index Range Scan) | 앞부분 고정 → 정렬 구간 탐색 가능 |
| `LIKE '%kim'` | 어려움 (Full Scan) | 앞부분 가변 → 정렬 기준 활용 불가 |
| `LIKE '%kim%'` | 어려움 (Full Scan) | 중간 위치 불명 → 전체 검사 필요 |

**`LIKE 'kim%'`의 내부 동작**
```sql
-- 인덱스 상에서 아래 범위 검색처럼 처리됨
name >= 'kim' AND name < 'kin'
```
정렬된 인덱스에서 구간을 특정할 수 있으므로 빠르다.

---

### SUBSTR은 왜 일반 인덱스를 못 타는가

```sql
WHERE SUBSTR(name, 1, 3) = 'kim'
```

인덱스는 `name` 원본값 기준으로 정렬되어 있다. `SUBSTR(name, 1, 3)`은 원본이 아니라 **함수 적용 결과**이므로, DB는 매 Row마다 아래 과정을 반복해야 한다.

```
1. name 원본값 읽기
2. SUBSTR 계산
3. 결과 비교
```

인덱스가 `SUBSTR 결과` 기준이 아니기 때문에 일반 인덱스 활용이 불가하다.

---

### LIKE vs SUBSTR — 어느 쪽이 낫나

| 상황 | 권장 방식 | 이유 |
|---|---|---|
| 접두사 검색 (앞이 고정) | `LIKE 'kim%'` | 인덱스 Range Scan 가능 |
| 접미사 검색 (끝이 고정) | 둘 다 Full Scan, LIKE가 옵티마이저 친화적 | SUBSTR은 함수 계산 비용 추가 |
| 중간 포함 검색 | 둘 다 Full Scan | 데이터 규모에 따라 전문 검색 엔진 고려 |

---

### 함수 인덱스 (Function-Based Index)

SUBSTR을 자주 쓰는 패턴이라면 **함수 인덱스**를 별도로 생성할 수 있다.

```sql
-- 함수 인덱스 생성
CREATE INDEX idx_name_prefix ON users(SUBSTR(name, 1, 3));

-- 이후 이 쿼리도 인덱스 사용 가능
WHERE SUBSTR(name, 1, 3) = 'kim'
```

---

### 끝 글자 검색을 빠르게 하는 방법

`LIKE '%kim'` 같은 접미사 검색은 인덱스를 타기 어렵다. 대안 두 가지:

**방법 1. Reverse 컬럼 활용**
```sql
-- 데이터를 뒤집어서 저장: 'kimtest' → 'tsetmik'
CREATE INDEX idx_rev_name ON users(reverse_name);

-- 접두사 검색으로 변환
WHERE reverse_name LIKE 'mik%'
```

**방법 2. 함수 인덱스**
```sql
CREATE INDEX idx_suffix ON users(SUBSTR(name, -3, 3));
WHERE SUBSTR(name, -3, 3) = 'kim'
```



## 출처

- 강의 내용 (LIKE 성능 주의 언급)
- 자체 학습 정리

---
*작성일: 2026-05-21 | 인공지능 사관학교 AI 보안 과정*
