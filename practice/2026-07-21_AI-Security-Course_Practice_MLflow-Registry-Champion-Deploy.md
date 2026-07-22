# 2026-07-21 실습 — MLflow 실험 관리부터 champion 배포·롤백까지 (ch03-2 + ch03-3)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**:
>
> - `../lectures/2026-07-20_AI-Security-Course_Lecture_MLOps-MLflow.md` (ch03.2 — Tracking·Registry·Alias)
> - `../lectures/2026-07-21_AI-Security-Course_Lecture_Deployment-Verification-Model-Versioning.md` (ch03-3)

---

## 느낀 것 / 배운 것

### 📅 2026-07-21

- **`train.py` 한 파일 교체로 Level 1이 Level 2가 됐다.** 어제까지는 결과를 파일 하나에 덮어썼는데, Tracking 버전으로 바꾸고 파라미터를 달리해 세 번 돌리니 Run 세 개가 각각 남았다. **성숙도 단계 사이의 간격이 생각보다 좁았다.**
- **champion 게이트가 게이트가 아니라는 걸 직접 확인했다.** Step 5에서 로컬 조회 값을 입력창에 손으로 타이핑했다. `confirm-champion` job은 그 값이 비어 있는지만 확인하고 로그에 찍는다 — 다른 숫자를 넣어도 통과했을 것이다. **검증처럼 생겼지만 실제로는 기록이었다.**
- **왜 검증이 안 되는지가 구조로 설명된다.** `mlflow.db`는 내 노트북에 있고 runner는 일회성 환경이라 그 파일에 닿지 못한다. 원격 Tracking 서버는 "있으면 편한 것"이 아니라 **검증을 가능하게 만드는 전제**였다.

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

- **MLflow 3.0 이상 필수** — `registry.py`의 `run.outputs.model_outputs`가 3.0에서 추가된 경로.

### Step 2. `train.py`를 MLflow Tracking 버전으로 교체

```powershell
Copy-Item ..\ch03-2\train.py src\devmlops_ml\train.py -Force
Copy-Item ..\ch03-2\registry.py src\devmlops_ml\registry.py
Get-Content src\devmlops_ml\train.py | Select-String "mlflow"
```

- 반영 확인: `mlflow.set_experiment(...)`, `with mlflow.start_run():`, `mlflow.log_param(...)`.

### Step 3~4. Run 세 개 쌓기

```powershell
python -m devmlops_ml.train
# → run_id=<32자리> accuracy=0.9778

python -c "from devmlops_ml.train import train; print(train(test_size=0.2, max_iter=200))"
python -c "from devmlops_ml.train import train; print(train(test_size=0.3, max_iter=500))"
```

- 각 실행이 서로를 덮어쓰지 않고 독립된 Run으로 남는다.

### Step 5. MLflow UI로 Run 비교

```powershell
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

`http://127.0.0.1:5000` → **Model Training** → Experiments → `iris-classification` → Runs 체크 → **Compare**

- 확인: 실험 목록, Run 3개 이상, 아티팩트(`metrics.json`·Logged Model), 실행별 accuracy·파라미터.
- 백엔드 스토어는 반드시 `sqlite:///mlflow.db`. (`./mlruns`로 주면 유지보수 모드 오류)

### Step 6~7. Registry 등록과 champion 승격

```powershell
python -m devmlops_ml.registry --run-id <복사한_run_id> --alias challenger
# → iris-classifier version 1 @ challenger
python -m devmlops_ml.registry --run-id <같은_run_id> --alias champion
# → iris-classifier version 2 @ champion
```

- **의도적 단순화 지점**: 같은 run_id로 재등록해 버전 2를 만들었고 버전 1과 내용이 같다. 실무에서는 새 버전을 만들지 않고 **검증 끝난 버전의 별칭만 옮긴다.** 두 별칭이 다른 버전을 가리키는 상태(`{'challenger': 1, 'champion': 2}`)를 눈으로 보려는 구성.

### Step 8. UI에서 Registry 최종 상태 확인

- Model Registry → `iris-classifier` → 버전 1(`challenger`), 2(`champion`), 각 버전에서 원본 Run으로 이동 가능.

---

## 진행 과정 — Part 2. champion 기반 배포와 롤백 (ch03-3)

### Step 1. `resolve_champion.py` 로컬 실행

```powershell
cd ch03-1
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
# → model=iris-classifier alias=champion version=2 run_id=<32자리>
```

- 이 값이 이후 배포 입력값이 된다.

### Step 2. champion 별칭 제거 후 조회 실패 확인

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().delete_registered_model_alias('iris-classifier', 'champion')"
python ..\ch03-3\resolve_champion.py --model-name iris-classifier --alias champion
# → champion 별칭이 없습니다. 배포를 진행할 수 없습니다.
```

- 별칭이 없으면 배포 입력값 자체가 생성되지 않는 상태를 재현.

### Step 3. 별칭 재등록

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().set_registered_model_alias('iris-classifier', 'champion', <기존_champion_버전>)"
```

### Step 4. Workflow 파일 push와 Secrets 확인

```bash
git add .github/workflows/ml-deploy.yml ch03-3/resolve_champion.py
git commit -m "Add ML deploy workflow"
git push origin main
```

- Settings → Secrets and variables → Actions에 `AWS_ROLE_ARN`, `AWS_REGION`, `ECR_REPOSITORY`, `ECS_SERVICE`, `ECS_CLUSTER` 확인 (값은 마스킹되어 이름만 보이는 게 정상).

### Step 5. `ml-deploy` Workflow 실행 (정상 배포)

Actions → `ml-deploy` → **Run workflow** → `champion_version`·`champion_run_id`에 Step 3 값 입력.

| Job                | 확인 기준                               |
| ------------------ | --------------------------------------- |
| `confirm-champion` | 성공, 로그에 입력값 출력                |
| `deploy`           | 성공, `wait-for-service-stability` 통과 |

- **값을 손으로 입력한다는 게 핵심.** Actions는 로컬 `mlflow.db`를 조회하지 않고 입력값이 비었는지만 확인한다.

### Step 6. Console·CLI로 배포 상태 대조

```powershell
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].{running:runningCount, desired:desiredCount, taskDef:taskDefinition}"
```

- CLI `taskDef` 리비전과 Console 번호가 같은지, `running == desired == 1`인지 확인.
- **Actions 로그의 champion run_id와 이 리비전 번호를 함께 기록** (모델 버전 / run_id / 태스크 정의 리비전).

### Step 7. 재학습 후 champion 별칭 이동

```powershell
python -c "from devmlops_ml.train import train; print(train(test_size=0.25, max_iter=300))"
python -m devmlops_ml.registry --run-id <새_run_id> --alias champion
# → iris-classifier version 3 @ champion
```

### Step 8. 재배포

- Step 5와 동일, 입력값만 Step 7 값으로. → 배포 대상이 바뀌는 흐름 확인.

### Step 9. 롤백 ① — 태스크 정의 리비전 (실행 중인 서비스, 즉시)

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service `
  --task-definition devmlops-fastapi:<직전_정상_리비전>
```

### Step 10. 롤백 ② — champion 별칭 (다음 배포의 기준)

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); from mlflow.tracking import MlflowClient; MlflowClient().set_registered_model_alias('iris-classifier', 'champion', <이전_champion_버전>)"
```

- Step 9와 되돌리는 대상이 다르다.

---

## 결과 / 산출물

- `iris-classification` 실험 아래 파라미터가 서로 다른 Run 4개 이상 (기본 + 2회 + 재학습)
- Model Registry `iris-classifier` — 버전 1(`challenger`), 2, 3, 별칭 이동 이력
- `ml-deploy` Workflow (`workflow_dispatch` 입력 기반 `confirm-champion` → `deploy`)
- champion 값을 바꿔가며 실행한 배포 2회 + ECS 리비전 롤백 + 별칭 롤백 기록
- 배포 기록 3종 세트: 모델 버전 / run_id / 태스크 정의 리비전

---

## 잘 된 것 / 안 된 것

| 잘 된 것                                                     | 안 된 것                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Tracking → 비교 → 등록 → 승격 → 배포 → 재학습 → 재배포 → 롤백까지 한 사이클 완주 | `confirm-champion`이 실제 검증이 아님을 확인했지만, 원격 Tracking 서버 구성(해결된 형태)은 과정 범위 밖 |
| 별칭을 지워 배포 입력값을 만들 수 없는 상태를 재현           | champion을 옮겨도 ECS 이미지는 그대로일 수 있다는 문제는 강의 설명으로만 확인 |
| 롤백을 두 시스템에서 각각 수행하고 대상 차이를 구분          | Registry 버전 1·2가 내용상 중복 (자료가 밝힌 단순화)         |

---

*작성일: 2026-07-21 | 인공지능 사관학교 AI 보안 과정*
