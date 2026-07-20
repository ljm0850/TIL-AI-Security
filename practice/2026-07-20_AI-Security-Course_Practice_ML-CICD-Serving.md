# 2026-07-20 실습 — ML 서빙 컨테이너와 검증 파이프라인 (ch03-1)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-20_AI-Security-Course_Lecture_Model-Serving-ML-CICD.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-20

- **배포 차단을 직접 재현해본 게 오늘의 핵심이었다.** 임계값을 1.0으로 올려 push했더니 `test-and-validate`가 빨간불, `docker-build`와 `deploy`는 아예 **Skipped**로 남았다. 실패가 "에러 났다"가 아니라 **뒤 단계가 시작조차 안 됐다*
- **테스트가 매번 재학습한다는 걸 코드 보고 알았다.** `test_model_quality.py`가 `train.train()`을 호출한다. 그러면 검증하는 모델은 테스트 시점의 학습 결과고, 실제 배포되는 모델은 Docker 빌드 중 다시 학습된 별개 파일이다
- **PR 본문에 성능 수치를 적게 한 게 의미 있었다.** 오늘 아침에 "ML 개발에서 PR을 생각해본 적이 없다"고 메모했는데, PR이 없는 게 아니라 리뷰 대상이 코드 diff만이 아니었던 거다. diff만 보면 "숫자 하나 0.9로 되돌림"인데, 실제로는 배포 가능 여부를 가르는 값이다. 맥락 없이는 리뷰가 불가능한 변경이라 본문에 수치와 이력을 적는 것.

---

## 실습 개요

- **목표**: 학습된 iris 분류 모델을 FastAPI로 서빙하고, 코드 테스트 + 모델 성능 검증 + 승인 게이트를 갖춘 ML CI/CD 파이프라인을 구성해 ECS까지 배포한다.
- **방법**: ch02-3의 ECR·ECS 배포 환경과 OIDC 역할을 그대로 재사용하고, 배포 대상만 ML 서빙 컨테이너로 교체한다. 검증 실패 → 배포 차단 → 원복 → 승인 배포 순서로 진행.

---

## 진행 과정

### Step 1~2. 환경 준비와 모델 학습

- 저장소 루트의 `.venv`를 재사용한다(ch03-1 안에 새 가상환경을 만들지 않음). `pip install -e ".[dev]"`.
- `python -m devmlops_ml.train` 실행 → `model/model.joblib`, `model/metrics.json` 생성.

```
model saved. accuracy=0.9778
```

- 여기서 나온 `0.9778`이 이후 모든 단계의 기준점이 된다. 임계값 0.9는 통과, 1.0은 실패.

### Step 3. 서빙 API 로컬 실행과 예측 요청

```powershell
python -m uvicorn devmlops_ml.main:app --host 127.0.0.1 --port 8000
```

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
Invoke-RestMethod -Method Post http://127.0.0.1:8000/predict -ContentType "application/json" `
  -Body '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

- `/health`가 `ok` → 서버뿐 아니라 **모델 로딩까지** 성공했다는 뜻. 모델 파일 없이 띄우면 `model_not_loaded`가 나온다.
- `/predict` 응답: `setosa`, 확률과 `model_version` 포함.
- 응답에 모델 버전이 같이 나오는 게 눈에 띄었다. 나중에 "이 예측은 어느 모델이 낸 건가"를 되짚을 수 있게 하는 장치.

### Step 4. 전체 테스트 (코드 + 모델 검증)

```powershell
python -m pytest -v
```

```
tests/test_api.py::test_health_returns_ok_when_model_loaded   PASSED
tests/test_api.py::test_predict_returns_known_class           PASSED
tests/test_api.py::test_predict_rejects_negative_input        PASSED
tests/test_model_quality.py::test_model_accuracy_meets_threshold PASSED
4 passed
```

- 코드 검증 3개(`test_api.py`)와 모델 검증 1개(`test_model_quality.py`)가 **하나의 스위트**로 돌아간다. pytest 입장에선 둘 다 그냥 테스트인데, 실패했을 때 조치 방향은 완전히 다르다.
- `test_predict_rejects_negative_input` — 음수 입력이 422로 거부되는지 확인. Pydantic 검증이 모델 호출 전 방어선으로 작동한다는 걸 테스트로 못 박아둔 것.

### Step 5. 검증 실패 재현 — 배포 차단 상황 만들기

```powershell
$env:MODEL_ACCURACY_THRESHOLD = "1.0"
python -m pytest tests/test_model_quality.py -v
```

```
FAILED tests/test_model_quality.py::test_model_accuracy_meets_threshold
AssertionError: 모델 정확도 0.9778가 임계값 1.0보다 낮다. 배포를 진행하면 안 되는 상태이다.
```

- 실패 메시지에 **실제값과 기준값이 둘 다** 찍힌다. 이게 코드 오류(traceback)와 모델 검증 실패를 로그 형태로 구분하게 해주는 지점.
- 환경 변수 원복(`$null`) 후 재실행 → `1 passed`.
- 환경 변수가 셸에 남아 있으면 이후 단계가 계속 실패하니, 원복 확인이 필수였다.

### Step 6. Docker 이미지 빌드와 컨테이너 확인

```powershell
docker build -t devmlops-ml:local .
docker run --rm -p 8000:8000 devmlops-ml:local
```

- 빌드 로그에 `RUN python -m devmlops_ml.train`이 실행되는 것을 확인. **이미지 빌드 중에 학습이 일어난다** → 코드와 모델이 같은 태그에 묶인다.
- 컨테이너에 다른 값으로 예측 요청 → `virginica`.
- 여기가 앞서 짚은 "검증한 모델 ≠ 배포한 모델" 지점이다. Step 4의 테스트가 학습한 모델과 이 이미지 안의 모델은 서로 다른 실행 결과다.

### Step 7. 승인 게이트 설정

- Settings → Environments → `production`(ch02-3에서 생성) → **Required reviewers** 체크 → 승인자 추가 → Save.
- Free·Pro·Team 플랜의 **Private 저장소에서는 승인 게이트가 제한**될 수 있다. 이 경우 Public 전환이 대안인데, 전환 전에 커밋 이력에 비밀 값·개인정보가 없는지 먼저 확인해야 한다.
  - Public 전환 조건이 "비밀이 커밋되지 않았을 것"이라는 게 Step 12의 민감정보 점검과 그대로 연결된다. 점검을 안 해두면 선택지 자체가 줄어든다.

### Step 8. Workflow 실행 — 검증 실패부터

워크플로의 임계값을 `"1.0"`으로 바꾸고 push.

```powershell
(Get-Content -Encoding UTF8 .github\workflows\ml-cicd.yml) `
  -replace 'MODEL_ACCURACY_THRESHOLD: "0.9"', 'MODEL_ACCURACY_THRESHOLD: "1.0"' `
  | Set-Content -Encoding UTF8 .github\workflows\ml-cicd.yml
```

```bash
git checkout -b ch03-1-ml-pipeline
git add .
git commit -m "feat: ML 서빙 컨테이너와 검증 파이프라인 추가 (검증 실패 재현)"
git push origin ch03-1-ml-pipeline
```

Actions 탭에서 확인한 결과:

| job | 상태 |
|---|---|
| `test-and-validate` | 실패 (빨간색) |
| `docker-build` | **Skipped** |
| `deploy` | **Skipped** |

- 실패한 job → 실패한 step → 로그 순으로 열어 assert 메시지 확인.
- Skipped가 두 개 나란히 있는 화면이 "배포 차단"의 실체였다.

### Step 9. ECS 서비스 재개 확인

```powershell
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].{running:runningCount, desired:desiredCount}"
```

- ch02-3 마지막에 `desired-count`를 0으로 내려뒀으므로 1로 복구.

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service --desired-count 1
aws ecs wait services-stable --cluster devmlops-cluster --services devmlops-service
```

- `running 1 / desired 1` 확인.
- 이 단계를 건너뛰면 deploy job은 성공하는데 `/predict`가 응답하지 않는다. **배포 성공과 서비스 가용은 별개**라는 걸 구조적으로 보여주는 단계.

### Step 10. 임계값 원복과 승인 후 배포

- 임계값을 `"0.9"`로 되돌려 커밋 → push.
- PR 생성 (base: `main`, compare: `ch03-1-ml-pipeline`).
- PR 본문에 남긴 내용:

```
모델 정확도: accuracy=0.9778
검증 기준: threshold=0.9
실패 재현: Step 8에서 test-and-validate 실패, docker-build와 deploy는 Skipped 확인
원복 조치: MODEL_ACCURACY_THRESHOLD를 "0.9"로 되돌림
배포 승인: main 병합 후 production 승인 게이트에서 승인 예정
```

- PR Checks에서 `test-and-validate`, `docker-build` 성공 확인. **`deploy`는 PR에서 실행되지 않는 것이 정상** — 실제 배포는 main 병합 후에 시작된다.
- Merge → 로컬 정리.

```bash
git checkout main
git pull origin main --ff-only
git branch -d ch03-1-ml-pipeline
```

- main에서 시작된 `ml-cicd` 실행 → deploy job이 **Waiting** → Review deployments → production 체크 → **Approve and deploy**.
- 승인 화면을 실제로 눌러본 게 처음이었다. 이 시점 전까지 그 job은 production Environment의 AWS Secrets을 읽을 수 없다는 게 오늘 강의에서 가장 인상 깊었던 부분.

### Step 11. 배포된 ML API와 CloudWatch 로그 확인

```powershell
Invoke-RestMethod http://<ALB-DNS>/health
Invoke-RestMethod -Method Post http://<ALB-DNS>/predict `
  -ContentType "application/json" `
  -Body '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

→ `prediction : setosa`

- CloudWatch → Log Management → `/ecs/devmlops-fastapi` → 최신 로그 스트림에서 애플리케이션 시작 로그와 요청 처리 로그 확인.
- 콘솔 메뉴가 기존 Logs > Log groups에서 **Log Management로 통합**되어 있었다.

```powershell
aws logs tail /ecs/devmlops-fastapi --since 10m
```

- 배포로 태스크가 교체되면 **새 로그 스트림**이 생긴다. 옛 스트림을 보고 "로그가 안 남는다"고 착각하기 쉬운 지점.
- `/predict`가 404면 이전(ch02-2) 이미지가 아직 서비스 중이라는 신호 → 태스크 정의 리비전 확인.

### Step 12. 민감 정보 점검과 체크리스트

```powershell
git ls-files | Select-String -Pattern "\.env|credential|secret"
Select-String -Path ".github/workflows/*.yml" -Pattern "AKIA|aws_secret"
```

→ 출력 없음 (정상)

작성한 운영·보안 체크리스트:

| 점검 항목 | 확인 방법 | 결과 |
|---|---|---|
| 비밀 값의 Secrets 관리 | Workflow에 하드코딩된 키 없음 | ✅ |
| `.env`·자격 증명 파일 미커밋 | `git ls-files` 확인 | ✅ |
| 로그에 비밀 값 미출력 | Actions 로그에서 마스킹 확인 | ✅ |
| 이미지에 비밀 값 미포함 | Dockerfile에 비밀 인자 없음 | ✅ |
| IAM 권한 최소화 | OIDC 역할의 정책 범위 확인 (ECR·ECS만) | ✅ |
| 모델 검증 임계값 | Workflow env 값 `"0.9"` 확인 | ✅ |

### Step 13. 실습 종료 — ECS 서비스 일시 중지

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service --desired-count 0
```

- `running`이 즉시 0이 되지 않을 수 있어 1~2분 뒤 재확인.
- 다음 배포 검증 전에 다시 1로 올려야 한다 (Step 9와 동일).

---

## 결과 / 산출물

- iris 분류 모델을 서빙하는 FastAPI 애플리케이션 (`/health`, `/predict`)
- 코드 검증 3건 + 모델 성능 검증 1건으로 구성된 테스트 스위트
- 학습 단계를 포함한 Docker 이미지 (코드 + 모델이 같은 태그)
- `test-and-validate → docker-build → deploy` 3단계 `ml-cicd` 워크플로우, deploy에 production 승인 게이트 적용
- 검증 실패 시 배포 차단(Skipped) 재현 기록 및 PR 문서화
- 운영·보안 체크리스트 6항목

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 검증 실패 → Skipped → 원복 → 승인 배포까지 한 사이클을 끊김 없이 돌림 | 검증 실패를 "기준을 올려서" 만든 것이라 실제 운영의 실패 인과와 반대. 데이터를 훼손해 성능을 떨어뜨리는 방식은 못 해봄 |
| ch02-3 인프라를 그대로 재사용해 배포 대상만 교체 — 인프라 재구축 없이 진행 | 테스트가 매번 재학습하는 구조라 "검증한 모델 = 배포한 모델" 보장이 없음. 지금은 결정적이라 안 터질 뿐 |
| PR 본문에 성능 수치·실패 재현·원복 조치를 남겨 코드 diff만으로는 안 보이는 맥락을 기록 | 임계값이 워크플로 파일 안에 있어 게이트와 우회 권한이 분리되지 않음 |
| Step 9에서 desired-count 복구를 먼저 확인해 "배포 성공인데 응답 없음" 상황을 미리 차단 | 승인 게이트가 플랜·저장소 공개 범위에 걸리는 구조라, 팀 프로젝트에서 그대로 쓸 수 있을지는 미확인 |

---

## 다음에 이어서 할 것

- ch03-2: MLflow Tracking을 학습 코드에 붙여 실행별 파라미터·메트릭·아티팩트 기록
- 실습 재개 전 `desired-count`를 1로 복구할 것

## 다음에 파고들 만한 질문

<!-- 이번 실습은 곁가지 여지가 적어 비워둔다. -->

---
*작성일: 2026-07-20 | 인공지능 사관학교 AI 보안 과정*
