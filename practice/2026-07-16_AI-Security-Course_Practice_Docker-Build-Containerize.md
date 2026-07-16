# 2026-07-16 실습 — Docker 이미지 빌드·컨테이너화 (ch02-2)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (ch02-2)
> **관련 강의 파일**: `../lectures/2026-07-15_AI-Security-Course_Lecture_Docker-Intro.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-16

- (지난 시간 리뷰 — 테스트) 면접에서 테스트 기법을 가끔 물어봤던 기억이 있는데, 그땐 이게 뭘 의미하는지 몰라서 "문제 생기면 `console.log()`로 함수 흐름 따라가며 디버깅했다"고 답했었다. 이제 보면 그건 테스트가 아니라 사후 추적이었음. 앞으로 프로젝트에선 TDD까지는 아니어도 **개발 계획에 테스트 결과를 넣어두면, 개발을 잘 모르는 팀원이 리뷰할 때 드는 시간이 많이 단축될 것** 같다. (CI가 이걸 강제해준다는 걸 어제 실습으로 봤으니 더 그렇게 느낌)
- 커밋 해시 태그를 `manual-001`로 직접 빌드해보고, 실제 파이프라인은 `github.sha` 앞 7자리를 쓴다는 걸 확인. 이미지↔코드를 왜 묶는지(추적·롤백) 실습으로 체감.

---

## 실습 개요
- **목표**: ch02-1 CI에서 검증한 FastAPI 코드를 Docker 이미지로 빌드·컨테이너 실행하고, docker-build 워크플로우로 빌드 검증을 자동화.
- **방법**: ch02-2 브랜치 → 소스 복사·정리 → 로컬 점검 → 이미지 빌드/컨테이너 실행/헬스체크 → 커밋 해시 태그 → 워크플로우 작성·push·병합.

---

## 진행 과정

> 세션 설정과 이미 익숙한 git ops(main 최신화, 브랜치 생성, add/commit/push/PR 병합)는 한 줄 복습으로 축약. 새로 익힌 부분 위주.

### Step 2~4. 소스 준비 + 로컬 점검
- ch02-1의 `main.py`·`test_main.py`를 `devmlops_docker` 패키지로 복사 → ch02-2 구조에서 `pytest` 통과(2 passed) → `uvicorn`으로 로컬 실행해 `/health` 응답(status ok, version 0.1.0) 확인. **이미지 빌드 전에 앱 문제부터 거르는 순서**.

### Step 6. Docker 이미지 빌드
- 멀티스테이지 Dockerfile: builder(venv 생성 + 패키지 설치) / runtime(`/opt/venv`만 복사). 강의 예시보다 붙은 것: `syntax` 디렉티브, `PYTHONDONTWRITEBYTECODE`/`PYTHONUNBUFFERED` env, 그리고 `HEALTHCHECK`(컨테이너가 `/health`를 주기적으로 자가 점검).
- `docker build -t devmlops-fastapi:local .`

### Step 8. 컨테이너 실행
- `docker run --rm -p 8000:8000 devmlops-fastapi:local` → 별도 터미널에서 `/health` 확인. **포트 바인딩(`-p`)이 없으면 외부 접근이 안 된다**는 걸 실행으로 확인 (`EXPOSE`는 기록일 뿐).

### Step 9. 커밋 해시 태그
- `docker build -t devmlops-fastapi:manual-001 .`로 임의 태그 빌드 연습. 실제 파이프라인은 `github.sha` 앞 7자리를 태그로 사용해 이미지↔코드 이력을 연결.

### Step 10~13. docker-build 워크플로우
- `.github/workflows/docker-build.yml`: 트리거 push(main, `ch02-2-*`)/PR(main), `working-directory: ch02-2`, `${GITHUB_SHA::7}`로 짧은 커밋 해시 태그, `docker build` + `docker image inspect`로 이미지 생성 검증. push → Actions에서 빌드·검사 통과 확인 → PR로 main 병합 (ch02-3가 이 폴더를 빌드 대상으로 이어받음).
- (메모) 실습은 `actions/checkout@v7`, 강의 예시는 `@v4` — 버전 표기가 계속 다름.

---

## 결과 / 산출물
- 멀티스테이지 Dockerfile(HEALTHCHECK 포함), 로컬 이미지·컨테이너 정상 동작 확인.
- 커밋 해시 태그로 빌드·검사하는 `docker-build` 워크플로우, main 병합.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 로컬 pytest→실행→이미지→컨테이너 순으로 단계마다 검증하며 진행 |
| Dockerfile `HEALTHCHECK`로 컨테이너 자체 헬스체크를 처음 실습 |

---

## 다음에 이어서 할 것
- ch02-3 AWS 배포 실습에서 이 이미지를 ECR에 올리고 ECS로 실행하는 흐름으로 이어짐. (docker-build가 main에 병합돼 있어야 `ecr-push`가 ch02-2 폴더를 빌드 대상으로 찾음)

---
*작성일: 2026-07-16 | 인공지능 사관학교 AI 보안 과정*
