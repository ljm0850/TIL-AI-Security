# 2026-07-21 실습 — MLflow 실험 관리부터 champion 배포·롤백까지 (ch03-2 + ch03-3)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**:
> - `../lectures/2026-07-20_AI-Security-Course_Lecture_MLOps-MLflow.md` (ch03.2 — Tracking·Registry·Alias)
> - `../lectures/2026-07-21_AI-Security-Course_Lecture_Deployment-Verification-Model-Versioning.md` (ch03-3)
>

---

## 느낀 것 / 배운 것

### 📅 2026-07-21

- **`train.py` 한 파일 교체로 Level 1이 Level 2가 되는 게 실감났다.** 어제까지 `train.py`는 결과를 파일 하나에 덮어썼다. MLflow Tracking을 넣은 버전으로 바꾸고 파라미터를 달리해 세 번 돌리니 Run 세 개가 각각 남았다. **성숙도 단계 사이의 간격이 생각보다 좁다**
- **champion 게이트가 게이트가 아니라는 걸 실습에서 직접 확인했다.** Step 5에서 `ml-deploy`를 실행할 때, 나는 로컬에서 조회한 값을 **입력창에 손으로 타이핑했다.** `confirm-champion` job이 하는 일은 그 값이 비어 있는지 확인하고 로그에 찍는 것뿐이다. 다른 숫자를 넣었어도 통과했을 것이다. **검증처럼 생겼지만 실제로는 기록**
- **왜 검증이 불가능한지가 구조로 설명된다.** `mlflow.db`는 내 노트북에 있고, runner는 실행마다 새로 생성되는 일회성 환경이다. 검증하려면 검증하는 쪽이 근거 데이터에 닿을 수 있어야 하는데 그 연결이 아예 없다. 원격 Tracking 서버가 "있으면 편한 것"이 아니라 **검증을 가능하게 만드는 전제**였다.

---

## 실습 개요

- **목표**
  - (ch03-2) 학습 코드에 MLflow Tracking을 붙여 실행별 파라미터·메트릭·아티팩트를 분리 기록하고, 가장 나은 Run을 Model Registry에 등록해 `challenger` → `champion`으로 승격한다.
  - (ch03-3) champion 별칭을 배포 입력값으로 삼아 `ml-deploy` Workflow로 배포하고, 재학습·별칭 이동·재배포·롤백까지 한 사이클을 돌린다.
- **방법**: ch03-1 폴더와 저장소 루트 공통 `.venv`를 그대로 사용. MLflow는 로컬 SQLite(`mlflow.db`)에 기록.

---

## 진행 과정 — Part 1. MLflow Tracking과 Model Registry (ch03-2)

### Step 1. 환경 준비

```powershell
cd ch03-1
..\.venv\Scripts\Activate.ps1
python -m pip install "mlflow>=3.0.0"
```

- **MLflow 3.0 이상이 필수**다. `registry.py`가 쓰는 `run.outputs.model_outputs`가 3.0에서 추가된 경로라 그 미만에서는 동작하지 않는다.
- `pyproject.toml`의 dependencies에 이미 `mlflow>=3.0.0`이 포함되어 있어 `pip install -e ".[dev]"`로도 같은 결과.

### Step 2. `train.py`를 MLflow Tracking 버전으로 교체

```powershell
Copy-Item ..\ch03-2\train.py src\devmlops_ml\train.py -Force
Copy-Item ..\ch03-2\registry.py src\devmlops_ml\registry.py
Get-Content src\devmlops_ml\train.py | Select-String "mlflow"
```

- 반영 확인: `mlflow.set_experiment(EXPERIMENT_NAME)`, `with mlflow.start_run():`, `mlflow.log_param(...)`.
- 어제 짚었던 "다음 실행이 앞 결과를 덮어쓴다" 문제를 여기서 닫는다.

### Step 3~4. Run 세 개 쌓기

```powershell
python -m devmlops_ml.train
```

```
run_id=<32자리 16진수> accuracy=0.9778
```

파라미터를 바꿔 두 번 더 실행:

```powershell
python -c "from devmlops_ml.train import train; print(train(test_size=0.2, max_iter=200))"
python -c "from devmlops_ml.train import train; print(train(test_size=0.3, max_iter=500))"
```

- 출력 형태: `{'accuracy': 0.9..., 'run_id': '...'}`
- 어제까지는 이 세 번이 서로를 지웠을 텐데, 이제 각각 독립된 Run으로 남는다.

### Step 5. MLflow UI로 Run 비교

```powershell
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

`http://127.0.0.1:5000` → 왼쪽 메뉴 **Model Training** 선택 → Experiments → `iris-classification` → Runs 체크 → **Compare**

| 확인 항목 | 기준 |
|---|---|
| 실험 목록 | `iris-classification` 표시 |
| 실행 개수 | Run 3개 이상 |
| 아티팩트 | `metrics.json`, 연결된 Logged Model에 `model` 표시 |
| 메트릭 | 실행마다 accuracy 표시 |
| 파라미터 | `test_size`, `max_iter`가 실행마다 다르게 표시 |

- 백엔드 스토어를 `./mlruns`로 주면 "파일 저장소 유지보수 모드" 오류가 난다. **`sqlite:///mlflow.db`로 실행해야 한다.**
- GenAI 메뉴는 생성형 AI 애플리케이션 추적용이라 이번 실습에서는 쓰지 않는다.

### Step 6~7. Registry 등록과 champion 승격

```powershell
python -m devmlops_ml.registry --run-id <복사한_run_id> --alias challenger
```
```
iris-classifier version 1 @ challenger
```

```powershell
python -m devmlops_ml.registry --run-id <같은_run_id> --alias champion
```
```
iris-classifier version 2 @ champion
```

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; print(MlflowClient().get_registered_model('iris-classifier').aliases)"
```
```
{'challenger': 1, 'champion': 2}
```

- **여기가 의도적 단순화 지점.** 같은 run_id로 재등록해서 버전 2가 새로 만들어졌고, 버전 1과 내용이 같다. 실무에서는 새 버전을 만들지 않고 **검증 끝난 버전 1의 별칭만 옮긴다.** 두 별칭이 다른 버전을 가리키는 상태를 눈으로 보기 위한 구성.

### Step 8. UI에서 Registry 최종 상태 확인

- Model Registry → `iris-classifier` → 버전 1, 2 모두 표시, 버전 2에 `champion` / 버전 1에 `challenger`.
- 각 버전에서 원본 Run으로 이동 가능 → **배포된 모델에서 학습 파라미터까지 역추적되는 경로**가 실제로 이어져 있는 걸 확인.

---

## 진행 과정 — Part 2. champion 기반 배포와 롤백 (ch03-3)

### Step 1. `resolve_champion.py` 로컬 실행

```powershell
cd ch03-1
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
```
```
model=iris-classifier alias=champion version=2 run_id=<32자리 16진수>
```

- 이 값이 이후 배포 입력값이 된다. 기록해둘 것.

### Step 2. champion 별칭 제거 후 조회 실패 확인

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().delete_registered_model_alias('iris-classifier', 'champion')"
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
```
```
iris-classifier에 'champion' 별칭이 없습니다. 배포를 진행할 수 없습니다.
```

- 별칭이 없으면 Workflow에 넣을 `champion_version`·`champion_run_id`를 만들 수 없다. **배포 입력값 자체가 생성되지 않는 상태**를 재현한 것.
- 어제 임계값 1.0으로 배포 차단을 재현한 것과 같은 방식의 학습이다.

### Step 3. 별칭 재등록

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().set_registered_model_alias('iris-classifier', 'champion', <기존_champion_버전>)"
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
```

- Step 1과 같은 값이 나오면 복구 완료.

### Step 4. Workflow 파일 push와 Secrets 확인

```bash
git add .github/workflows/ml-deploy.yml ch03-3/resolve_champion.py
git commit -m "Add ML deploy workflow"
git push origin main
```

Settings → Secrets and variables → Actions에서 확인:

| Secret | 용도 |
|---|---|
| `AWS_ROLE_ARN` | OIDC로 위임받을 IAM 역할 |
| `AWS_REGION` | AWS 리전 |
| `ECR_REPOSITORY` | 이미지가 저장된 ECR 리포지터리 |
| `ECS_SERVICE`, `ECS_CLUSTER` | 배포 대상 ECS 서비스·클러스터 |

- 값 자체는 가려져서 보이지 않는다. **이름만 확인 가능한 게 정상**이고, 어제 배운 "Secrets는 마스킹된다"의 UI 측면.

### Step 5. `ml-deploy` Workflow 실행 (정상 배포)

Actions → `ml-deploy` → **Run workflow** → 입력값 채우기:

| 입력 필드 | 값 |
|---|---|
| `champion_version` | Step 3에서 조회한 version |
| `champion_run_id` | Step 3에서 조회한 run_id |

| Job | 확인 기준 |
|---|---|
| `confirm-champion` | 성공, 로그에 입력한 version·run_id 출력 |
| `deploy` | 성공, `wait-for-service-stability`까지 통과 |

- **여기서 손으로 값을 입력했다는 사실이 핵심이다.** Actions는 로컬 `mlflow.db`를 조회하지 않고, 입력값이 비어 있지 않은지만 확인한 뒤 배포 로그에 남긴다.

### Step 6. Console·CLI로 배포 상태 대조

```powershell
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].{running:runningCount, desired:desiredCount, taskDef:taskDefinition}"
```

- CLI의 `taskDef` 리비전과 Console의 리비전 번호가 같은지 확인.
- `running`이 1 이상이고 `desired`와 같아야 실제 태스크 실행 중.
- **Actions 로그의 champion run_id와 이 리비전 번호를 함께 기록** — 강의에서 강조한 세 값(모델 버전 / run_id / 태스크 정의 리비전)의 연결 고리.

### Step 7. 재학습 후 champion 별칭 이동

```powershell
python -c "from devmlops_ml.train import train; print(train(test_size=0.25, max_iter=300))"
python -m devmlops_ml.registry --run-id <새_run_id> --alias champion
```
```
iris-classifier version 3 @ champion
```

```powershell
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
```

- 재학습 → 등록 → 별칭 이동. **학습 → 평가 → 등록 → 배포**라는 MLOps 반복 단위를 한 사이클 돌린 셈.

### Step 8. 재배포

- Step 5와 동일하게 `ml-deploy`를 다시 실행하되 입력값은 Step 7에서 확인한 새 값으로.
- `confirm-champion` 로그에 새 값이 그대로 출력되고, 배포 후 Step 6의 CLI 조회로 태스크 정의 리비전도 다시 기록.

### Step 9. 롤백 ① — 태스크 정의 리비전

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service `
  --task-definition devmlops-fastapi:<직전_정상_리비전>

aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].taskDefinition"
```

- `taskDef` 값이 지정한 리비전으로 바뀌면 정상. **실행 중인 서비스 상태**가 즉시 되돌아간다.

### Step 10. 롤백 ② — champion 별칭

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().set_registered_model_alias('iris-classifier', 'champion', <이전_champion_버전>)"

python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; print(MlflowClient().get_registered_model('iris-classifier').aliases)"
```
```
{'challenger': 1, 'champion': 2}
```

- 이건 **다음 배포에서 사용할 모델 기준**을 되돌리는 작업이다. Step 9와 대상이 다르다.

---

## 결과 / 산출물

- `iris-classification` 실험 아래 파라미터가 서로 다른 Run 4개 이상 (기본 + 2회 + 재학습)
- Model Registry `iris-classifier` — 버전 1(`challenger`), 2, 3, 별칭 이동 이력
- `ml-deploy` Workflow (`workflow_dispatch` 입력 기반 `confirm-champion` → `deploy`)
- champion 값을 바꿔가며 실행한 배포 2회 + ECS 리비전 롤백 + 별칭 롤백 기록
- 배포 기록 3종 세트: 모델 버전 / run_id / 태스크 정의 리비전

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| Tracking 도입 → Run 비교 → 등록 → 승격 → 배포 → 재학습 → 재배포 → 롤백까지 끊김 없이 한 사이클 완주 |
| 별칭을 일부러 지워 배포 입력값을 만들 수 없는 상태를 재현 |
| 롤백을 두 시스템에서 각각 수행하고 대상 차이를 명확히 구분 |
| 배포 기록 3종 세트를 남겨 모델 승인과 서비스 리비전을 연결 |

---
*작성일: 2026-07-21 | 인공지능 사관학교 AI 보안 과정*
