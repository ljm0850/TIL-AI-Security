# 2026-07-15 실습 — GitHub Actions 기반 CI (pytest · ruff)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (ch02-1)
> **관련 강의 파일**: `../lectures/2026-07-15_AI-Security-Course_Lecture_CI-CD-Intro.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-15

- 예전 팀 프로젝트 때는 구현에 정신이 쏠려 CI를 아예 못 챙겼는데, 이번에 pytest+ruff를 워크플로우에 붙여 PR마다 자동으로 돌아가는 걸 직접 보니 좀 인상적이었다. "돌아가게 만드는 것"과 "팀이 운영 가능하게 만드는 것"이 다르다는 걸 CI가 구체적으로 보여준다. (구현 급해서 구조 부채 쌓았던 경험이랑 정확히 대비됨)
- Step 8에서 일부러 status를 `error`로 바꿔 PR Checks를 빨갛게 만들어봤다. CI가 "실패하면 병합 못 하게" 실제로 막아주는 걸 눈으로 확인 → 규칙을 문서가 아니라 파이프라인으로 강제한다는 감각을 얻음.
- (보안 렌즈) 워크플로우가 push/PR에 반응해 러너에서 코드를 자동 실행한다. `secrets`가 여기 env로 주입되는 구조라, 이 자동 실행 표면 자체가 관리 대상이라는 걸 실습하며 체감.
- 로컬 수동 검증(Step 6: venv+pytest+ruff) → 워크플로우 통합(Step 7) 순서가 자연스러웠다. 로컬에서 먼저 통과시킨 뒤 CI에 얹으니 파이프라인이 뭘 자동화한 건지 명확해짐.

---

## 실습 개요
- **목표**: PR·push마다 pytest(기능 테스트) + ruff(스타일 검사)를 자동 실행하는 CI 파이프라인을 구성하고, 실패가 실제로 병합을 막는지 검증.
- **방법**: ch02-1 실습 브랜치에서 최소 워크플로우 → 앱/테스트 코드 작성 → 로컬 수동 검증 → 워크플로우에 테스트/린트 스텝 통합 → 의도적 실패 유도로 PR Checks 검증 → 병합.

---

## 진행 과정

> 세션 설정(PowerShell 실행 정책, venv 활성화 등)과 이미 익숙한 브랜치 생성·커밋·push는 한 줄 복습으로 축약. 새로 익힌 부분 위주로 기록.

### Step 1~3. 최소 워크플로우
- `ch02-1-ci` 브랜치에서 `.github/workflows/ci.yml` 작성. GitHub Actions는 저장소 루트의 `.github/workflows/`만 인식한다.
- 최소 워크플로우: `on`(push/PR) → `welcome-job` → `echo` 메시지. push 후 Actions 탭에서 로그에 시작 문구가 뜨는지 확인.

### Step 4~5. 앱·테스트 코드
- **두 테스트 패턴을 함께 연습**: `calculator.py`(입출력이 명확한 순수 함수) + `main.py`(FastAPI `/health` 엔드포인트). CI는 이 둘을 같은 명령으로 자동 실행.
- 테스트: `test_calculator` 3개(add / add_negative / subtract) + `test_main` 1개(health_check — `TestClient`로 상태 코드·JSON 응답 검증).

### Step 6. 로컬 수동 검증
- venv 생성·활성화 → `pip install -e ".[dev]"` → `pytest`(4 passed) → `ruff check .`. 자동화에 얹기 전에 로컬 통과를 먼저 확인.

### Step 7. 워크플로우에 테스트·린트 통합
- `ci.yml`을 통합 버전으로 덮어씀: `defaults.run.working-directory: ch02-1`, checkout → setup-python → install → pytest → ruff.
- (메모) 실습 자료는 `actions/checkout@v7`·`setup-python@v6`를 쓰는데, 강의 예시는 `@v4`·`@v5`였다 → major 버전 표기가 서로 다름. 실제로 돌린 건 v7/v6.

### Step 8. 의도적 실패 → PR Checks 검증
- `main.py`의 상태값을 `error`로 바꿔 push → PR 생성 → `ci / test`가 빨갛게 실패, `Run tests` 스텝에서 assertion 오류 확인.
- 상태값을 `ok`로 복구 → Checks 전부 초록.

### Step 9. 병합
- Checks 통과 후 `ch02-1-ci` → main 병합, 브랜치 정리. 이후 ch02-2(Docker)가 main 기준으로 새 브랜치를 분기할 수 있는 상태가 됨.

---

## 결과 / 산출물
- push·PR마다 pytest+ruff가 자동 실행되는 CI 워크플로우(`ci.yml`).
- 순수 함수 테스트 3개 + API 테스트 1개, ruff 통과.
- 의도적 실패가 PR 병합을 실제로 차단함을 확인.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 로컬 수동 검증 → CI 통합 순서로 파이프라인 역할이 명확했음 |
| 의도적 실패로 PR Checks 차단을 눈으로 검증 |

---

## 다음에 이어서 할 것
- ch02-2 Docker 실습(main 기준 새 브랜치)에서 이 CI 위에 컨테이너 빌드 스텝을 얹는 흐름으로 이어짐.

---
*작성일: 2026-07-15 | 인공지능 사관학교 AI 보안 과정*
