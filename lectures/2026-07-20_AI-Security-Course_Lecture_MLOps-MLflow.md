# 2026-07-20 강의 정리 — MLOps와 MLflow (ch03.2)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: DevOps와 MLOps의 차이, 성숙도 모델, MLflow Tracking과 Model Registry
> **관련 강의 파일**: `2026-07-20_AI-Security-Course_Lecture_Model-Serving-ML-CICD.md` (ch03.1)

---

## 핵심 인사이트 모음

### 📅 2026-07-20

- **Git이 코드의 변화를 추적한다면, MLOps는 모델의 변화를 추적한다.** 오늘 메모에 이렇게 적었는데 정리하면서 더 정확해졌다. 문제는 모델의 변화가 커밋 이력으로 설명이 안 된다는 점이다. 같은 코드에 데이터만 달라져도 다른 모델이 나오니까, 커밋 해시만으로는 "왜 이 모델이 이 성능인가"에 답할 수 없다. 그래서 기록 대상이 **커밋 이력 + 실험 이력 + 모델 버전** 세 축으로 늘어난다.

- **성숙도 모델로 내 위치를 확인했다.** ch03-1까지 만든 파이프라인은 Level 1이다 — 코드 테스트·모델 검증·배포는 자동화됐지만 실험 기록과 모델 버전은 수동이다. `train.py`가 결과를 파일 하나에 저장하니 다음 실행이 앞 결과를 덮어쓴다.

- **ch03-1에서 걸렸던 문제의 답이 여기 있었다.** "검증한 모델과 배포한 모델이 같은 물건이라는 보장이 없다"고 정리했는데, MLflow가 실행 단위로 파라미터·메트릭·아티팩트를 묶어서 기록하고 Model Registry에 버전으로 등록하는 게 정확히 그 문제를 해결한다. **평가한 그 실행의 결과물을 지목해서 배포하는 것**과, 배포 직전에 다시 학습하는 것은 완전히 다른 얘기다.

- **실험 기록은 늘어나기만 하고, 배포 후보는 좁혀져야 한다.** 실험(Experiment)/실행(Run)과 Model Registry가 왜 따로 있는지 여기서 이해했다. Run은 시도의 기록이라 계속 쌓이는 게 정상이고, Registry는 그중 "이건 나갈 수 있다"고 판단한 것만 이름과 버전으로 승격시키는 곳이다. 기록의 공간과 승인의 공간을 분리한 구조다.

- **실습 구성이 로컬 SQLite라는 점은 기억해둘 필요가 있다.** 지금은 `mlflow.db` 파일 하나에 실행 기록과 Registry 정보가 다 들어간다. 팀이 실험 기록을 같이 쓰려면 원격 서버가 필요한데, 그 순간 "누가 어떤 모델을 승격했는가"가 권한 문제가 된다. 팀 프로젝트에서 MLOps 요소를 쓰게 되면 이 지점을 먼저 정해야 할 것 같다.

### 📅 2026-07-21 (이어서)

- **코드 버전 관리는 git, 모델 버전 관리는 MLflow** — 어제 정리한 대비를 오늘 Registry까지 보고 나서 다시 쓰면 이렇게 된다. 다만 완전한 대응은 아니다. git은 커밋 해시 하나로 "그때 그 상태"가 확정되는데, MLflow는 Run(어떻게 만들었나)과 Registry 버전(무엇을 쓸 건가)이 **두 층으로 나뉜다**. 만드는 행위와 고르는 행위를 분리해야 했기 때문이라고 이해했다.
- **재현성 문제는 ML 모델만의 것이 아니라는 생각이 들었다.** LLM이 생성한 코드도 같은 프롬프트로 같은 결과가 나온다는 보장이 없고, "이 코드가 어떤 요청과 어떤 맥락에서 나왔는가"가 남아 있지 않으면 나중에 되짚을 수 없다. MLflow가 파라미터·메트릭·아티팩트를 실행 단위로 묶는 것과 같은 문제를, 생성형 도구를 쓰는 개발 과정도 안고 있다. 도구 이름만 다를 뿐 **"결과물과 그 결과물을 만든 조건을 함께 남기는가"**라는 질문은 동일하다.

---

## 1. MLOps 개요

### 핵심 개념

**MLOps(Machine Learning Operations)**: 자동화·버전 관리·지속적 통합/배포 원칙을 코드뿐 아니라 **데이터와 모델의 변경 이력에도** 적용하는 운영 방법론.

### 상세 내용

머신러닝 시스템에서는 코드 외에 데이터와 모델도 바뀐다.

| 구분      | DevOps               | MLOps                             |
| --------- | -------------------- | --------------------------------- |
| 관리 대상 | 코드                 | 코드 + 데이터 + 모델              |
| 변경 원인 | 코드 수정            | 코드 수정, 데이터 변화, 재학습    |
| 반복 단위 | 빌드 → 테스트 → 배포 | 학습 → 평가 → 등록 → 배포         |
| 기록 대상 | 커밋 이력            | 커밋 이력 + 실험 이력 + 모델 버전 |

- **재현성(Reproducibility)**: 같은 코드·데이터로 같은 모델을 다시 만들 수 있는 성질. 파라미터·평가 결과가 없으면 재현 여부조차 확인할 수 없다.

### MLOps 성숙도 모델 (Maturity Level)

| 단계    | 특징                                                         | 주요 기술 |
| ------- | ------------------------------------------------------------ | --------- |
| Level 0 | 노트북에서 수동 학습·기록·배포                               | —         |
| Level 1 | 코드 테스트·모델 검증·배포는 CI/CD 자동화. **실험 기록·모델 버전은 수동** | CI/CD     |
| Level 2 | 실험 기록·모델 등록·버전 관리까지 자동화. 데이터·코드·파라미터 ↔ 모델 추적 가능 | MLOps     |

- **ch03-1까지가 Level 1**, ch03-2가 Level 2로 넘어가는 작업이다.

---

## 2. MLflow 개념

### 핵심 개념

MLflow는 학습 실행의 **파라미터, 성능 수치, 생성 파일**을 함께 기록한다. 기록된 실행 중에서 배포 후보를 골라 모델 레지스트리에 등록한다.

### 상세 내용

**실험(Experiment)과 실행(Run)**

```
실험 "iris-classification"
 ├─ 실행 1 (test_size=0.2)                → accuracy=0.95 → model.joblib
 ├─ 실행 2 (test_size=0.3)                → accuracy=0.98 → model.joblib
 └─ 실행 3 (test_size=0.3, max_iter=500)  → accuracy=0.98 → model.joblib
```

| 용어                | 의미                                                         |
| ------------------- | ------------------------------------------------------------ |
| 실험(Experiment)    | 같은 목표로 진행한 여러 실행을 모으는 상위 단위              |
| 실행(Run)           | 학습 코드 한 번의 실행 기록. 파라미터·메트릭·아티팩트가 실행 단위로 저장 |
| 파라미터(Parameter) | 학습 시 지정한 설정값. 예: `test_size=0.3`, `max_iter=200`   |
| 메트릭(Metric)      | 학습·평가 결과 수치. 예: `accuracy=0.9778`                   |
| 아티팩트(Artifact)  | 실행 결과로 저장되는 파일. 실습에서는 학습된 모델 파일       |

**모델 레지스트리(Model Registry)**

```
모델 레지스트리 "iris-classifier"
 ├─ 버전 1 (실행 2에서 등록)
 └─ 버전 2 (실행 3에서 등록)
```

- 학습 실행에서 나온 모델을 이름과 버전으로 등록해 관리하는 저장소.
- 실험 기록은 계속 늘어나지만, **배포 후보는 레지스트리에 등록한 것으로 좁혀서** 관리한다. 등록된 모델은 어느 Run에서 왔는지 연결되어 있다.

---

## 3. MLflow Tracking 적용

### 핵심 개념

`train.py`가 결과를 파일 하나에만 저장하면 다음 실행이 앞 결과를 덮어쓴다. Tracking은 실행마다 파라미터·메트릭·아티팩트를 나누어 기록한다.

### 상세 내용

```python
import mlflow

mlflow.set_experiment("iris-classification")

with mlflow.start_run():
    mlflow.log_param("test_size", 0.3)
    mlflow.log_param("max_iter", 200)

    model, accuracy = train_and_evaluate()

    mlflow.log_metric("accuracy", accuracy)
    mlflow.sklearn.log_model(model, name="model")
    mlflow.log_artifact("model/metrics.json")
```

| 함수                         | 역할                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| `mlflow.start_run()`         | 실행의 시작·종료 범위 지정. `with` 블록을 벗어나면 자동 종료 |
| `mlflow.log_param()`         | 실행에 사용한 설정값 기록                                    |
| `mlflow.log_metric()`        | 실행 결과 수치 기록                                          |
| `mlflow.sklearn.log_model()` | 학습된 모델을 MLflow Model 형식으로 기록. **프레임워크 정보를 함께 저장**해서 이후 모델 등록에 사용 가능 |
| `mlflow.log_artifact()`      | 파일을 실행 기록에 첨부. `metrics.json`처럼 모델 등록 대상이 아닌 파일용 |

- `log_model()`은 **배포 후보가 될 수 있는 것**, `log_artifact()`는 **참고 기록**으로 나뉜다.
- 실습은 로컬 SQLite `mlflow.db`에 기록을 저장한다. 팀 공유용 원격 Tracking 서버는 과정 범위 밖.

---

## 4. MLflow UI로 결과 비교하기

### 핵심 개념

파라미터를 바꿔가며 학습을 반복하면 Run이 쌓인다. 파라미터와 성능 지표를 나란히 놓고 비교해야 더 나은 실행을 찾을 수 있다.

### 상세 내용

```bash
mlflow ui --backend-store-uri ./mlruns
```

- 기본 접속 주소는 `http://127.0.0.1:5000`.
- **Run 비교(Run Comparison)**: 여러 실행의 파라미터·메트릭을 나란히 놓고 차이를 확인.
- 검색 기능에 `metrics.accuracy > 0.97` 같은 조건을 쓰면 기준을 만족하는 실행만 조회할 수 있다.
- UI에 **GenAI**와 **Model Training** 메뉴가 함께 뜬다. iris 학습 실행은 Model Training > Experiments에서 확인. GenAI 화면은 생성형 AI 애플리케이션 기록 추적용.

---

## 📅 2026-07-21 이어서

## 5. Model Registry와 모델 버전 관리

### 핵심 개념

Model Registry는 **배포 후보 모델**을 이름과 버전으로 관리한다. 버전에 별칭을 붙이면 배포 과정은 버전 번호를 직접 적지 않고 **역할 이름**으로 모델을 조회할 수 있다.

### 상세 내용

**모델 등록(Model Registration)** — 특정 Run의 모델을 실험 기록과 분리해 레지스트리에 저장한다.

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient()
run_id = "abc123..."   # mlflow ui에서 확인한 실행 ID

# 실행이 기록한 모델의 URI(models:/<model_id>)를 찾아 그 URI로 등록한다.
model_id = client.get_run(run_id).outputs.model_outputs[0].model_id
mlflow.register_model(
    model_uri=f"models:/{model_id}",
    name="iris-classifier",
)
```

| 용어                          | 의미                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| 모델 등록(Model Registration) | 특정 Run의 모델을 레지스트리에 이름과 함께 저장하는 작업     |
| 버전(Version)                 | 같은 이름으로 등록한 모델의 순번. 새로 등록할 때마다 1씩 증가하고 **이전 버전은 보존** |

- `run.outputs.model_outputs`는 MLflow 3.0에서 추가된 경로다. 3.0 미만에서는 동작하지 않는다.

### 모델 별칭(Alias) 관리

등록된 버전이 늘어나면 번호만으로 현재 운영 모델을 알기 어렵다.

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()
client.set_registered_model_alias(
    name="iris-classifier",
    alias="champion",
    version=2,
)
```

- **별칭(Alias)**: 모델 버전에 붙이는 **역할 이름**. 버전 2에 `champion`을 붙이면 배포는 `models:/iris-classifier@champion`으로 조회한다. 새 모델을 올릴 때는 별칭만 옮기면 되고 배포 코드는 그대로다.

| 별칭         | 의미                       |
| ------------ | -------------------------- |
| `challenger` | 검증 중인 후보 모델        |
| `champion`   | 현재 운영에 사용 중인 모델 |

**Stage → Alias 전환**: 구버전 Stage는 `Staging`/`Production`/`Archived` 정해진 이름만, 버전 하나에 상태 하나만 지정했다. MLflow 2.9부터 deprecated. 새 코드는 Alias를 쓴다(개념은 기존 코드에서 볼 수 있으니 알아둘 것).

---

*작성일: 2026-07-20 | 인공지능 사관학교 AI 보안 과정*
