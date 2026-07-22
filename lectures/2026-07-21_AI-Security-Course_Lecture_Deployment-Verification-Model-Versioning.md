# 2026-07-21 강의 정리 — 배포 검증과 모델 버전 운영 (ch03-3)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 모델 버전과 배포 이미지의 연동, champion 별칭 기반 배포 조건, ECS 배포 재검증과 롤백
> **관련 강의 파일**: `2026-07-20_AI-Security-Course_Lecture_MLOps-MLflow.md` (ch03.2 — Model Registry·Alias)
> **관련 실습 파일**: `../practice/2026-07-21_AI-Security-Course_Practice_MLflow-Registry-Champion-Deploy.md`

---

## 핵심 인사이트 모음

### 📅 2026-07-21

- **어제 짚었던 문제** "검증한 모델과 배포한 모델이 같다는 보장이 없다"고 적었는데, 오늘 강의는 champion 별칭을 버전 3으로 옮겨도 ECS는 버전 2 학습 결과가 든 이미지를 계속 실행할 수 있다고 명시한다. 그래서 **MLflow UI의 성능 기록과 운영 API의 실제 동작이 달라질 수 있다**. 더 나아가 "모델 버전과 이미지 태그를 완전히 하나로 묶는 구성은 만들지 않는다"고 못 박는다. 문제를 해결하는 대신 **문제가 있다는 사실을 기록으로 남기는 쪽을 선택한 것**
- **champion 게이트는 게이트가 아니다.** `confirm-champion` job이 실제로 하는 일은 입력값이 비어 있는지(`-z`) 확인하는 것뿐이다. GitHub Actions runner는 로컬 `mlflow.db`를 읽을 수 없으니 사람이 손으로 version과 run_id를 입력창에 넣고, 워크플로는 그 값을 로그에 찍는다. **아무 숫자나 넣어도 통과한다.** 검증(verification)이 아니라 기록(attestation)
- **왜 검증이 안 되는지가 구조적으로 명확하다.** 신뢰의 근거가 되는 데이터(로컬 SQLite `mlflow.db`)가 검증을 수행해야 하는 주체(원격 runner)의 사정권 밖에 있다. runner는 실행마다 새로 만들어지는 일회성 환경이라 로컬 상태를 알 수 없다. **검증하려면 검증 주체가 근거에 직접 닿을 수 있어야 한다** — 원격 Tracking 서버가 필요한 이유

---

## 1. 모델 버전과 배포 이미지 연동

### 핵심 개념

MLflow는 **모델 버전과 학습 실행**을 기록하고, ECR·ECS는 **컨테이너 이미지와 태스크 정의**를 기록한다. 두 기록은 서로 다른 시스템에 있으므로, 배포할 모델과 실행 중인 컨테이너의 관계는 **대조해야만** 확인할 수 있다.

### 상세 내용

| 시스템    | 남는 것                         |
| --------- | ------------------------------- |
| ECR / ECS | 이미지 태그, 태스크 정의 리비전 |
| MLflow    | 모델 버전, run_id, 별칭         |

**두 기록이 어긋나는 경우**: champion 별칭을 버전 3으로 옮겨도 ECS는 버전 2가 든 이미지를 실행할 수 있다. → **MLflow UI 기록과 운영 API 동작이 달라진다.**

**태그 접두사로 이미지 구분**: ECR 하나를 두 Workflow가 함께 쓴다. `ecr-push`(ch02-3)는 기본 API 이미지를 커밋 해시로, `ml-cicd`(ch03-1)는 ML 서빙 이미지를 `ml-` 접두사 + 커밋 해시로 push한다. `ml-deploy`는 `ml-` 태그만 조회해 기본 API 이미지가 잘못 선택되는 걸 피한다.

- 과정에서는 모델 버전과 이미지 태그를 하나로 묶지 않고, **배포 전 champion의 버전·run_id를 확인해 기록에 남기는 방식**을 쓴다.

---

## 2. Model Registry 기반 배포 조건

### 핵심 개념

**배포 조건(Deployment Condition)**: 배포 job이 시작되기 전에 만족해야 하는 검증 기준.

- ch03-1은 **모델 성능 임계값**을 조건으로 썼다.
- ch03-3은 여기에 **champion 별칭의 존재**를 추가한다. 별칭이 없다는 건 운영에 올릴 모델을 아직 고르지 않았다는 뜻이므로 배포를 시작하지 않는다.

### 상세 내용

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()
version = client.get_model_version_by_alias("iris-classifier", "champion")
print(version.run_id, version.version)
```

- `get_model_version_by_alias()`는 별칭이 가리키는 버전 정보를 조회한다. **별칭이 없으면 예외 발생 → 배포 중단 신호로 사용.**

**적용 범위**: champion 확인은 `ml-deploy`에만 적용된다. `ml-cicd`는 main push마다 별칭 확인 없이 배포하므로, 이 확인이 **모든** 배포를 막지는 않는다. 어느 경로로 배포됐는지는 Actions 실행 기록으로 사후 구분한다.

---

## 3. MLflow 기반 배포 검증 — 로컬 Registry와 원격 runner의 간극

### 핵심 개념

실습은 원격 MLflow 서버를 만들지 않는다. 그래서 **로컬 데이터베이스와 원격 runner가 분리된 상태**를 먼저 이해해야 한다.

### 상세 내용

- MLflow 기록은 로컬 `mlflow.db`에 있고, **runner는 실행마다 새로 만들어지는 환경**이라 이 파일을 읽지 못한다.
- 원격 Tracking 서버 + S3 저장소를 구성하면 해결되지만 과정 범위 밖.

**그래서 우회한다** — 로컬에서 champion 값을 확인해 `workflow_dispatch` 입력값으로 직접 넣는다.

```bash
py ../../ch07/code/resolve_champion.py --model-name iris-classifier --alias champion
# → model=iris-classifier alias=champion version=<champion_버전> run_id=<run_id>
```

```yaml
on:
  workflow_dispatch:
    inputs:
      champion_version:
        description: "로컬에서 resolve_champion.py로 확인한 champion 모델 버전"
        required: true
      champion_run_id:
        description: "로컬에서 resolve_champion.py로 확인한 champion run_id"
        required: true

jobs:
  confirm-champion:
    runs-on: ubuntu-latest
    steps:
      - name: Require champion inputs
        run: |
          if [ -z "${{ inputs.champion_version }}" ] || [ -z "${{ inputs.champion_run_id }}" ]; then
            echo "champion_version과 champion_run_id를 입력해야 배포를 진행할 수 있습니다." >&2
            exit 1
          fi
          echo "champion version=${{ inputs.champion_version }} run_id=${{ inputs.champion_run_id }}"

  deploy:
    needs: confirm-champion
    runs-on: ubuntu-latest
    environment: production
```

**이 구성이 하는 것 / 안 하는 것**

| 한다                               | 하지 않는다                                |
| ---------------------------------- | ------------------------------------------ |
| 입력값이 비어 있으면 배포를 막는다 | 입력값이 실제 champion인지 확인하지 않는다 |
| 배포한 버전·run_id를 로그에 남긴다 | 로컬 Registry를 조회하지 않는다            |

---

## 4. ECS 배포 재검증과 롤백

### 핵심 개념

배포 뒤에는 Console과 CLI에서 ECS 서비스·태스크·배포 상태를 확인한다. 문제가 생기면 **ECS 태스크 정의 리비전과 MLflow champion 별칭을 각각** 되돌린다.

### 상세 내용

**배포 상태 재검증**

```powershell
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].{running:runningCount, desired:desiredCount, taskDef:taskDefinition}"
```

- `taskDef`(실행 중인 태스크 정의 리비전)와 Actions 로그의 champion run_id를 **함께 남겨** 모델 승인 기록과 서비스 리비전을 연결한다.

**롤백 — 되돌리는 대상이 둘**

| 롤백               | 대상                  | 효과 시점    |
| ------------------ | --------------------- | ------------ |
| ECS 리비전 롤백    | 실행 중인 서비스 상태 | 즉시         |
| champion 별칭 롤백 | 다음 배포의 모델 기준 | 다음 배포 때 |

운영 API가 급하면 ECS부터 되돌린다.

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service `
  --task-definition devmlops-fastapi:<직전_정상_리비전>
```

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()
client.set_registered_model_alias("iris-classifier", "champion", <이전_champion_버전>)
```

---

*작성일: 2026-07-21 | 인공지능 사관학교 AI 보안 과정*
