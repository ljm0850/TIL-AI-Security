# 2026-07-15 강의 정리 — CI/CD와 GitHub Actions 기반 CI

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: CI/CD 개념과 GitHub Actions 기반 CI 파이프라인 (ch02.1)
> **관련 실습 파일**: `../practice/2026-07-15_AI-Security-Course_Practice_CI-GitHub-Actions.md`

---

## 핵심 인사이트 모음

### 📅 2026-07-15

- CI/CD의 핵심은 결국 **"개인 환경에 의존한 검증"을 "저장소 기준 자동 검증"으로 바꾸는 것**이다. 개발자마다 OS·패키지 버전이 달라 같은 코드도 결과가 다르고, 수동 점검은 빠뜨리기 쉽다. 그래서 팀 전체에 일관된 품질 기준을 맞추려면 검증을 사람 손이 아니라 파이프라인에 맡겨야 한다.
- CI는 병합 **전** 검증(테스트+린트), CD는 검증된 결과물의 빌드·배포. 같은 파이프라인 위에서 순서대로 돌지만 역할이 갈린다. CD는 CI 위에 도커 빌드/클라우드 배포 스텝을 얹어 확장하는 구조.
- (보안 렌즈) CI/CD 파이프라인 = **트리거에 반응해 코드를 자동 실행하는 표면**. `secrets`(AWS 키·API 토큰)가 러너 env로 주입되는 구조라, 악의적 PR이 워크플로우를 통해 secrets를 탈취하는 게 하나의 공격류로 성립한다. 자동 실행 표면을 늘리는 건 blast radius를 늘리는 것 — 펑션 콜링 때 느낀 것과 같은 결이다.
- 그래서 이 표면을 어떻게 좁힐지 고민해봤다. 정리하면 세 갈래:
  - (1) **외부/포크(fork) PR에는 secrets를 주입하지 않는다.** 남의 fork에서 온 PR이 내 credential을 env로 받는 순간이 제일 위험한 지점.
  - (2) **신뢰받지 않은 PR의 워크플로우는 승인(approval) 후에만 실행한다.** 아무 PR이나 자동 실행되게 두지 않고, 사람이 한 번 게이트를 건다.
  - (3) **secrets 주입 범위를 배포 단계로만 좁힌다.** 테스트·빌드 단계엔 아예 노출하지 않는다 — 검증만 하는 단계가 credential을 들고 있을 이유가 없으니 최소권한 원리로 범위를 자른다.
- 테스트와 린트는 검사 영역이 안 겹친다(실행 결과 vs 코드 텍스트). 하나만 쓰면 빈틈이 생기니 둘을 붙여야 커버된다.
- `pyproject.toml`은 Python판 통합 설정 파일이다. Java에서 JDK+Gradle로 빌드 환경을 잡던 것처럼, 흩어져 있던 프로젝트 정보·의존성·도구 설정을 한 곳에 모은다.
- Jenkins는 규모(개발 인원)가 크거나 보안이 중요할 때 많이 고려된다고 함. CI/CD를 어필하고 싶으면 한 번 해보는 것도 괜찮을 듯. (self-host라 관리 부담 vs 통제력 tradeoff로 보임)
- CI는 사람 코드리뷰를 **대체하지 않는다.** 기계적 규칙과 상태 코드만 확인하고, 설계 합리성·비즈니스 로직 보안성은 사람 몫. 자동화가 늘수록 리뷰가 새 병목이 된다는 얘기랑 이어진다.

---

## 1. CI/CD 개요

### 핵심 개념
- CI(Continuous Integration, 지속적 통합): 코드가 저장소에 병합되기 전 자동으로 테스트+코드 스타일 검사를 실행.
- CD(Continuous Delivery/Deployment, 지속적 전달/배포): 검증된 결과물을 빌드해 실제 서비스 환경에 준비/적용.

### 상세 내용
- **CI가 필요한 이유**: 개발자마다 OS·패키지 버전이 달라 같은 코드도 실행 결과가 다르게 나온다. PR마다 수동으로 테스트를 돌려 공유하는 과정에서 빠뜨리기 쉽고, 개인 환경 의존 검증으론 팀 일관 품질을 못 맞춘다.
- CI는 코드 변경이 들어올 때 미리 정한 검사 절차를 자동 실행하고, 통과 여부를 PR 페이지에 바로 표시한다. 도커 빌드·클라우드 배포 단계도 이 CI 파이프라인 위에 스텝을 추가하는 방식으로 확장.
- **CD 두 갈래**: Delivery(배포 가능한 상태로 준비) / Deployment(실제 환경에 배포까지 자동). 강의는 둘 다 포함하는 개념으로 다룸.
- 예시 — CI: pytest 기능 테스트, ruff 스타일 검사. CD: 도커 이미지 빌드 + ECR(Elastic Container Registry) Push, ECS(Elastic Container Service) 태스크 업데이트.

---

## 2. 검사 도구·프로젝트 설정

### 핵심 개념
- 테스트(pytest) + 린트(ruff)는 검사 영역이 안 겹쳐서, 같이 써야 빈틈이 준다.
- `pyproject.toml` = 파이썬 표준 설정 파일 (TOML 포맷).

### 상세 내용
- **테스트(pytest)**: 함수 동작 결과, API 응답 상태 코드, 예외 발생 여부를 프로그램을 직접 실행해 확인. 한계 — 작성한 시나리오 범위 **밖**의 버그는 못 잡는다.
- **린트(ruff)**: 코드 실행 전에 텍스트를 분석해 코딩 스타일·미사용 변수·문법 오류를 확인. 한계 — 비즈니스 로직 오류나 요구사항 충족 여부는 판단 못 한다.
- **pyproject.toml 구성**: `[project]`(이름/버전/의존성), `[project.optional-dependencies]`(dev 그룹으로 pytest·ruff 분리), `[tool.ruff]`, `[tool.pytest.ini_options]`를 한 파일에 통합.
- **TOML**: key=value 쌍, `[...]`로 영역 구분. 파이썬 3.6부터 공식 지원되지만 3.10+ 사용 권장.

```toml
[project]
name = "devmlops-ci"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.110.0",
    "uvicorn>=0.28.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.3.0",
]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

---

## 3. GitHub Action 구성요소

### 핵심 개념
- 워크플로우 / 잡 / 스텝 / 러너 — 네 요소의 조합으로 파이프라인 전체를 표현.

### 상세 내용
- **워크플로우(Workflow)**: 파이프라인 전체 실행 명세. `.github/workflows/` 하위에 YAML로 작성.
- **잡(Job)**: 같은 실행 환경에서 처리되는 스텝들의 모음. 기본적으로 독립된 러너에서 병렬 실행되며, `needs` 키워드로 잡 간 실행 순서를 제어.
- **스텝(Step)**: 잡 내부에서 순서대로 실행되는 개별 작업 단위. `run`으로 명령을 직접 실행하거나 `uses`로 만들어진 액션을 불러옴.
- **러너(Runner)**: 잡을 실제로 수행하는 가상 머신. 실습에선 `ubuntu-latest`를 주로 사용.

---

## 4. 워크플로우 YAML 구조

### 핵심 개념
- `name` / `on` / `env` / `secrets` / `jobs` / `steps` 키워드 조합으로 파이프라인을 설정.

### 상세 내용
- **name**: Actions 탭에 표시되는 라벨.
- **on**: 실행을 시작하는 트리거 이벤트(push, pull_request)와 대상 브랜치를 지정.
- **env**: 워크플로우/잡/스텝 범위에서 쓸 환경 변수.
- **secrets**: AWS 액세스 키, API 토큰 등 외부에 노출되면 안 되는 인증 정보를 저장소 설정에서 읽어옴. → CI 보안의 핵심 표면이 바로 이 지점.
- **uses**: GitHub Marketplace에 등록된 액션을 이용해 실행 환경 설정(checkout, setup-python 등).
- **run**: 러너의 셸에서 직접 명령어를 실행.
- **YAML 주의사항**: 탭(Tab) 금지 — 공백 들여쓰기 깊이로만 상하 관계 구분(어긋나면 실행 자체가 중단). 소수점 포함 값은 반드시 따옴표로 감싸 문자열화("3.12"; 안 그러면 파서가 3.1로 오독).

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

env:
  PYTHON_VERSION: "3.12"

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          python -m pip install -e ".[dev]"
      - name: Run tests
        run: python -m pytest
      - name: Run lint
        run: python -m ruff check .
```

---

## 5. 실행 트리거·PR Checks

### 핵심 개념
- push / pull_request 트리거, PR Checks로 병합 전 차단.

### 상세 내용
- **push 이벤트**: 특정 브랜치에 직접 커밋이 반영됐을 때 실행. 주로 병합이 끝난 main의 통합 결과를 최종 검증.
- **pull_request 이벤트**: PR이 생성되거나 소스 브랜치에 새 커밋이 push될 때 실행. 가상 병합(Merge) 상태를 만들어 테스트를 미리 돌려, 불완전한 코드가 공유 영역으로 들어오는 걸 막는다.
- push + pull_request를 함께 쓰면 병합 **전후 모두** 검증.
- **PR Checks**: PR 페이지 하단에 자동 연결되는 검증 결과. 초록 체크 / 빨강 실패 로그가 뜨고, 실패 시 병합 차단 경고. "통과 못 한 PR은 main에 병합하지 않는다"를 팀 합의로 규칙화 가능.
- **수동 리뷰와 차이**: 기계적 규칙·상태 코드 확인은 CI가, 설계 요구사항 충족·API 구조 합리성·비즈니스 로직 보안성은 사람이 본다. CI는 리뷰를 대체하는 게 아니라 수동 검토 시간을 줄이는 보조.

---

## 찾아본 것
| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| | | | |
|||||

---
*작성일: 2026-07-15 | 인공지능 사관학교 AI 보안 과정*
