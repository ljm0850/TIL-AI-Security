# 2026-05-21 심화 정리 — 글로벌 서비스의 시간 처리 (UTC & Timezone)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-05-20_AI-Security-Course_Lecture_Oracle-SQL-Intro.md`
> **작성 계기**: 강의에서 날짜 함수와 세션 NLS_DATE_FORMAT을 다루면서, 글로벌 서비스에서 시간을 나라별로 어떻게 처리하는지 궁금해서 찾아봄

---

## 왜 찾아봤는가

 글로벌 서비스라면 나라마다 시간대가 다를 텐데, DB에 시간을 어떤 기준으로 저장하고 어떻게 변환해서 보여주는지 실무 구조가 궁금해졌다.

---

## 핵심 내용

### 핵심 원칙: 저장은 UTC 하나로, 출력만 변환

글로벌 서비스의 시간 처리는 "나라별로 따로 저장"이 아니라 **"UTC 하나로 저장 → 출력 시 사용자 timezone으로 변환"** 구조다.

```
저장: 2026-05-21 03:00:00 UTC (하나의 기준)

출력:
  한국 사용자  → 2026-05-21 12:00 (KST, UTC+9)
  뉴욕 사용자  → 2026-05-20 23:00 (EDT, UTC-4)
  파리 사용자  → 2026-05-21 05:00 (CEST, UTC+2)
```

같은 순간인데 보여주는 시각만 다르다.

---

### 왜 UTC로 저장해야 하는가

한국 시간(`2026-05-21 12:00`)으로 저장하면 이 값이 한국 기준인지, 일본 기준인지, 미국 기준인지 알 수 없다. 특히 **썸머타임(DST, Daylight Saving Time)** 적용 국가에서는 같은 timezone이라도 계절에 따라 offset이 바뀌어서 더 위험하다.

UTC는 썸머타임이 없는 절대 기준이기 때문에 혼란이 없다.

---

### 실무 구조

**1. DB 저장 — UTC**
```sql
-- 오라클: UTC 기준으로 현재 시각 저장
SYSTIMESTAMP AT TIME ZONE 'UTC'
```

**2. 사용자 timezone 저장**
사용자 프로필에 timezone 정보를 별도로 저장한다.
```
Asia/Seoul
America/New_York
Europe/Paris
```

**3. 조회 시 변환 — 백엔드 또는 프론트엔드에서 처리**

오라클에서 직접 변환:
```sql
SELECT created_at AT TIME ZONE 'Asia/Seoul'
FROM orders;
```

프론트엔드에서 처리 (요즘 더 많이 쓰는 방식):
```javascript
// 백엔드는 UTC 그대로 반환
// 브라우저가 사용자 OS timezone으로 자동 렌더링
new Date(utcDate)
```

---

### 용도별 기준 정리

| 용도 | 기준 |
|---|---|
| DB 저장 | UTC |
| 로그 | UTC |
| 서버 간 통신 | UTC |
| 배치 서버 | UTC 권장 |
| 사용자 화면 표시 | 사용자 timezone |

---

### 오라클에서 글로벌 서비스용 타입

글로벌 서비스라면 일반 `DATE`나 `TIMESTAMP` 대신 아래 타입을 사용하는 것이 권장된다.

- `TIMESTAMP WITH TIME ZONE`: timezone 정보를 값과 함께 저장
- `TIMESTAMP WITH LOCAL TIME ZONE`: DB 서버의 로컬 timezone으로 자동 변환하여 저장

---

## 출처

- 강의 내용 (오라클 날짜 함수 및 세션 NLS 설정)
- 자체 학습 정리

---
*작성일: 2026-05-21 | 인공지능 사관학교 AI 보안 과정*
