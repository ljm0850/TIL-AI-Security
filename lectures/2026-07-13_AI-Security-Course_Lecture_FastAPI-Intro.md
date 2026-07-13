# 2026-07-13 강의 정리 — Python 웹앱 & FastAPI 개론

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 웹 서비스/프레임워크 개념, Python 웹 프레임워크 비교, WSGI/ASGI, FastAPI 구조, SQLAlchemy 개요
> **관련 실습 파일**: 없음 (앞으로 DevOps/MLOps 실습에서 FastAPI를 쓸 예정이라 오늘은 개념만 가볍게 훑음)

---

## 핵심 인사이트 모음

### 📅 2026-07-13
- "무거운 프레임워크에 익숙해진 습관으로 모든 걸 내장 기능에만 의존하려 하면 안 된다"는 주의사항이 와닿았다 — Django 풀스택 경험 때문에 FastAPI도 다 갖춰줄 거라 기대하면 안 되고, 필요한 것만 선택적으로 붙이는 감각이 필요해 보인다.
- WSGI vs ASGI 차이는 결국 "동기(요청마다 프로세스/스레드 점유) vs 비동기(I/O 대기 중 다른 요청 처리)" 처리 모델의 차이로 정리됐다.
- 오늘은 DevOps/MLOps 실습에서 쓸 걸 대비한 사전 훑기라 깊이보다는 흐름 위주로 들었다.

---

## 1. 웹 서비스와 프레임워크

### 핵심 개념
- 웹 서비스: HTTP/HTTPS 같은 표준 통신 규약으로 서로 다른 기기·애플리케이션이 데이터/기능을 공유하는 소프트웨어 시스템
- 웹 프레임워크: 웹 앱 구축에 필요한 구조·기능을 표준화해 제공하는 개발용 뼈대 — 개발자가 비즈니스 로직에만 집중하게 해줌

### 상세 내용
- 프레임워크의 역할: HTTP 파싱·미들웨어 같은 저수준 작업 자동화, MVC/MTV 같은 구조 강제, ORM·템플릿 엔진 등 내장 컴포넌트로 생산성 확보
- 주의사항
  - 설정보다 관습(Convention over Configuration) — 프레임워크가 권장하는 구조를 따라야 자동화 혜택을 누림
  - 경량 프레임워크(FastAPI 등)는 필요한 기능만 선택적으로 추가하는 능력이 중요
  - 추상화의 함정: 프레임워크가 내부에서 무슨 일을 하는지(예: ORM이 어떤 SQL을 생성하는지) 모니터링하지 않으면 트래픽이 커졌을 때 병목을 못 잡음

---

## 2. Python 웹 프레임워크 비교

### 핵심 개념
| 프레임워크 | 성격 | 특징 |
|---|---|---|
| Django | 풀스택 | 인증·관리자페이지·ORM 등 내장, 대규모 프로젝트에 적합하지만 초기 설정 부담 큼 |
| Flask | 마이크로 | 최소 기능만 제공, 아키텍처 자유도 높고 학습곡선 완만, DB/보안은 직접 설정 필요 |
| FastAPI | 고성능 | async/await 완전 지원(비동기), Pydantic 기반 타입 검증, Swagger UI 자동 생성 |

### 상세 내용
- 선택 기준: 테이블 50개 이상·복잡한 권한 체계 → Django / 실시간·대용량 I/O(채팅, AI 추론 결과 전달) → FastAPI / 팀 숙련도 낮거나 마이크로서비스 하나만 → Flask

---

## 3. WSGI vs ASGI

### 핵심 개념
- WSGI: 요청 하나당 프로세스/스레드가 응답까지 점유하는 동기(Blocking) 방식 — Django(이전 버전)·Flask 기반
- ASGI: async/await로 I/O 대기 중 다른 요청 처리가 가능한 비동기(Non-blocking) 방식 — 웹소켓·HTTP/2 등 지원, FastAPI가 채택

---

## 4. FastAPI 핵심 구성요소

### 핵심 개념
- ASGI 기반, Starlette(요청/응답 처리) + Pydantic(데이터 검증) 조합
- Python 타입 힌트 자체가 API 명세가 됨 → Swagger UI / ReDoc 자동 생성

### 상세 내용
- 주요 구성요소: `FastAPI()`(진입점), `@app.get/post/put/patch/delete`(라우팅), Path/Query Parameter(자동 타입 변환·검증), `BaseModel`(요청/응답 구조), `Depends()`(의존성 주입), `response_model`(응답 필드 필터링), `HTTPException`(에러 처리), `Middleware`(횡단 관심사)
- 권장 프로젝트 구조: `main.py`(앱 생성만, 비즈니스 로직 X) / `api/` / `schemas/`(Pydantic, ORM과 분리) / `crud/`(DB 로직) / `db/models/`(ORM) / `core/config.py`(환경변수) / `dependencies/`(공통 Depends) / `tests/`
- 적용 가이드: 타입 힌트 생략 금지, dict 대신 Pydantic 모델 사용, 동기 블로킹 함수를 async 안에서 직접 호출하면 서버 전체가 멈출 수 있음(주의)

---

## 5. SQLAlchemy ORM 개요

### 핵심 개념
- Python 클래스/메서드로 DB를 다루게 해주는 ORM, FastAPI의 사실상 표준 DB 연동 도구 (2.0 스타일 문법 기준)

### 상세 내용
- 구조: `Core`(Engine/Connection/SQL Expression Language, 저수준) + `ORM`(DeclarativeBase/Session/Relationship, 고수준)
- 흐름: Engine 생성 → `DeclarativeBase` 상속 모델 정의 → `Base.metadata.create_all()`로 테이블 생성 → `Session`으로 트랜잭션 단위 작업(`add`/`commit`/`rollback`/`close`)
- CRUD는 `select()` + `db.scalars()`/`db.execute()` 조합, 비동기 라우터를 쓴다면 `AsyncSession` 필수
- (오늘은 구조만 훑음 — 실제 CRUD/관계 정의는 나중에 실습에서 직접 다룰 때 보강 예정)

## 
*작성일: 2026-07-13 | 인공지능 사관학교 AI 보안 과정*
