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

| 시스템 | 남는 것 |
|---|---|
| ECR / ECS | 이미지 태그, 태스크 정의 리비전 |
| MLflow | 모델 버전, run_id, 별칭 |

**두 기록이 어긋나는 경우**

ML CI/CD는 모델을 학습하고 결과 파일을 이미지에 포함해 ECR에 push한다. MLflow Registry는 학습 실행·등록 버전·별칭을 따로 기록한다. 둘이 동기화되지 않으면:

- champion 별칭을 버전 3으로 옮긴 뒤에도 ECS는 버전 2 학습 결과가 들어 있는 이미지를 실행할 수 있다.
- 이 때문에 **MLflow UI의 성능 기록과 운영 API의 동작이 달라진다.**

**태그 접두사로 이미지를 구분한다**

ECR 리포지터리 하나를 두 Workflow가 함께 쓴다.

| Workflow | 대상 | 태그 |
|---|---|---|
| `ecr-push` (ch02-3) | ch02-2 기본 API 이미지 | 커밋 해시 |
| `ml-cicd` (ch03-1) | ch03-1 ML 서빙 이미지 | `ml-` 접두사 + 커밋 해시 |

- 둘 다 main push마다 실행되므로, 배포 Workflow가 **"최신 이미지"만 골라 쓰면 ML 서빙 이미지 대신 기본 API 이미지가 선택될 수 있다.**
- `ml-deploy`는 `ml-` 접두사 태그만 조회해서 이 문제를 피한다.
- 지난주 커밋 해시 태그를 배운 게 여기서 한 겹 더 쓰인다. 태그가 단순 식별자가 아니라 **선택 기준**으로도 쓰인다는 걸 처음 봤다.

**완전한 일치는 만들지 않는다 (강의의 명시적 선택)**

- 모델 버전과 이미지 태그를 하나로 묶으려면 이미지 태그에 run_id나 모델 버전을 넣고, 배포 Workflow가 그 태그를 직접 선택하는 구성이 필요하다.
- 과정에서는 그 대신 **배포 전에 champion의 모델 버전과 run_id를 확인하고 배포 기록에 남기는 방식**을 쓴다.

| 용어 | 의미 |
|---|---|
| 모델 버전(Model Version) | Registry에 등록한 모델의 순번 |
| run_id | MLflow가 학습 실행(Run)에 부여하는 고유 ID. 파라미터·메트릭·아티팩트가 어느 실행에서 나왔는지 추적할 때 사용 |
| 이미지 태그(Image Tag) | ECR 이미지를 구분하는 값 |

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

- `get_model_version_by_alias()`는 별칭이 가리키는 모델 버전 정보(run_id, version)를 조회한다. **별칭이 없으면 예외가 발생하고, 스크립트는 그 예외를 배포 중단 신호로 사용한다.**

```python
from mlflow.exceptions import MlflowException
from mlflow.tracking import MlflowClient

client = MlflowClient()
try:
    version = client.get_model_version_by_alias("iris-classifier", "champion")
except MlflowException:
    raise SystemExit("champion 별칭이 없습니다. 배포를 진행할 수 없습니다.")

print(version.version, version.run_id)
```

**적용 범위가 좁다는 점을 강의가 명시한다**

- champion 확인은 **`ml-deploy` Workflow에만** 적용된다.
- `ml-cicd`는 main 브랜치 push마다 별칭 확인 없이 별도로 배포한다.
- 즉 champion 확인은 "모델 버전을 지정해 배포하는 경로를 하나 추가한 것"이지, ECS 서비스로 향하는 **모든** 배포를 막는 장치가 아니다.
- 어느 경로로 배포됐는지는 GitHub Actions 실행 기록의 Workflow 이름과 실행 시각으로 사후 구분한다.

---

## 3. MLflow 기반 배포 검증 — 로컬 Registry와 원격 runner의 간극

### 핵심 개념

실습은 원격 MLflow 서버를 만들지 않는다. 그래서 **로컬 데이터베이스와 원격 runner가 분리된 상태**를 먼저 이해해야 한다.

### 상세 내용

- MLflow 기록은 `ch03-1/mlflow.db` 파일에 저장된다. 이 파일은 실습 로컬 컴퓨터에 있다.
- **GitHub Actions runner는 Workflow 실행마다 새로 만들어지는 환경**이므로 로컬 `mlflow.db`의 Registry 상태를 알지 못한다.
- 원격 Tracking 서버 + S3 아티팩트 저장소를 구성하면 runner도 같은 Registry를 조회할 수 있지만, 과정 범위 밖이다.

**그래서 이렇게 우회한다**

```bash
py ../../ch07/code/resolve_champion.py --model-name iris-classifier --alias champion
```

```
model=iris-classifier alias=champion version=<champion_버전> run_id=<run_id>
```

로컬에서 확인한 값을 `workflow_dispatch` 입력값으로 직접 넣는다.

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

**이 구성이 무엇을 하고 무엇을 하지 않는가 — 강의의 명시**

| 한다 | 하지 않는다 |
|---|---|
| 입력값이 비어 있으면 배포를 막는다 | 입력한 값이 실제 champion인지 확인하지 않는다 |
| 어떤 버전·run_id 기준으로 배포했는지 로그에 남긴다 | 로컬 Registry를 조회하지 않는다 |

- 운영 환경이라면 `MLFLOW_TRACKING_URI`를 원격 서버로 설정해 Actions가 같은 Registry를 조회하게 하거나, 모델 버전과 이미지 태그를 하나의 배포 메타데이터로 연결해야 한다.

| 용어 | 의미 |
|---|---|
| 배포 기록(Deployment Record) | 배포에 사용한 모델 버전과 run_id를 적은 기록. PR 설명·Actions 로그·실습 결과표에 남긴다 |
| 원격 Tracking URI | 여러 실행 환경이 같은 MLflow 기록을 조회하도록 연결하는 주소 |

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

- `taskDef` 값이 현재 서비스가 실행 중인 태스크 정의 리비전이다.
- 이 값과 Actions 로그·실습 기록의 champion run_id를 **함께 남기면** 모델 승인 기록과 서비스 리비전의 연결을 확인할 수 있다.
- 다만 강의도 인정한다 — **run_id 기록만으로 이미지와 모델 파일이 완전히 같은 조합임을 증명할 수는 없다.** 그래서 조합을 배포 기록에 적는다.

**롤백 — 되돌리는 대상이 둘로 나뉜다**

| 롤백 | 대상 | 효과 시점 |
|---|---|---|
| ECS 리비전 롤백 | **실행 중인 서비스 상태** | 즉시 |
| champion 별칭 롤백 | **다음 배포에서 사용할 모델 기준** | 다음 배포 때 |

운영 API를 먼저 되돌려야 하면 ECS부터.

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service `
  --task-definition devmlops-fastapi:<직전_정상_리비전>
```

다음 배포 기준은 별칭을 이전 검증 버전으로 옮겨 복구한다.

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()
client.set_registered_model_alias(
    name="iris-classifier",
    alias="champion",
    version=<이전_champion_버전>,
)
```

- 기록에는 **복구한 태스크 정의 리비전, 이전 champion 버전, 롤백 뒤 CLI 조회 결과**를 함께 적는다.
- 지난주 ch02-3에서 배운 "리비전 복귀와 그 한계"가 여기서 한 층 더 복잡해졌다. 되돌릴 것이 하나가 아니라 두 시스템에 걸쳐 있다.

*작성일: 2026-07-21 | 인공지능 사관학교 AI 보안 과정*
