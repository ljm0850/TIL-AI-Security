# 2026-07-21 실습 — 운영 모니터링과 리소스 정리 (ch04-1)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-21_AI-Security-Course_Lecture_Operational-Monitoring-Drift.md`
>
> ※ 이 실습도 ch03-1 폴더를 그대로 사용한다. 저장소 루트는 `labs_devmlops/devmlops/`.

---

## 느낀 것 / 배운 것

### 📅 2026-07-21

- **정상 요청과 오류 요청을 나란히 보내본 게 좋았다.** 음수 입력으로 일부러 422 에러를 만들고, 정상 요청 로그와 비교했다.  **운영 로그에서는 어떤 모양으로 남는지**를 이번에 봤다.
- **Step 5가 어제 짚은 우회 경로를 그대로 탄다.** 로깅 코드 변경을 main에 넣으니 `ml-cicd`가 돌면서 champion 별칭 확인 없이 같은 ECS 서비스에 배포됐다.통제를 하나 추가하는 것과 통제되지 않는 경로를 닫는 것은 완전히 다른 일
- **추론 로깅을 넣는 순간 로그의 성격이 바뀐다.** 그전까지 로그는 "서비스가 살아있는지"를 보는 것이었는데, `predict request={...} prediction=... probability=...`를 남기기 시작하면 로그가 **모델 관찰 데이터**가 된다. 동시에 입력값이 통째로 CloudWatch에 쌓인다는 뜻이기도 하다. iris 꽃 치수라 지금은 무해하지만, 도메인이 사람 데이터였다면 이 한 줄이 그대로 개인정보 저장소가 된다.
- **Step 8의 GitHub Issue 작성이 인상 깊었다.** 기존에도 Github Issue에는 어떤 것을 작성해야 하는가에 대한 예시를 보았기에 인상 깊었다. 점검 결과를 협업 도구에 남겨 "발견한 문제가 묻히지 않게" 하라는 것인데, 어제 PR 본문에 성능 수치를 적은 것과 같은 계열이다. **관찰한 것을 기록으로 옮기지 않으면 관찰하지 않은 것과 같다**

---

## 실습 개요

- **목표**: 배포가 끝난 서비스의 로그를 실제로 조회·해석하고, 추론 로깅을 추가해 모델 관찰에 필요한 정보를 남긴 뒤, 운영 점검 체크리스트와 Issue로 결과를 기록한다. 마지막으로 실습 인프라를 정리해 비용을 정리한다.
- **방법**: Console과 CLI를 병행해 같은 정보를 두 경로로 확인. 코드 변경은 로컬 → 배포 → 운영 로그 순으로 3단계 검증.

---

## 진행 과정

### Step 1~2. 로그 그룹·스트림 확인 (Console → CLI)

**Console**: CloudWatch → 로그 > 로그 관리 → `/ecs/devmlops-fastapi` → 최근 스트림 → 시간대는 현지 시간 선택.

**CLI**:

```powershell
aws logs describe-log-groups --log-group-name-prefix /ecs/devmlops-fastapi
aws logs describe-log-streams --log-group-name /ecs/devmlops-fastapi `
  --order-by LastEventTime --descending --max-items 5
```

```
logGroupName:  /ecs/devmlops-fastapi
logStreamName: ecs/devmlops-fastapi/<태스크ID>
```

- `logStreamName`은 전체 이름 그대로 이후 조회에 쓴다.
- 로그 그룹이 안 보이면 **리전 확인**이 먼저. ECS 서비스와 같은 리전이어야 한다.

### Step 3. 응답 시간과 에러 로그 확인

먼저 태스크가 실행 중인지 확인하고, `runningCount`가 0이면 올린다.

```powershell
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service `
  --query "services[0].{desiredCount:desiredCount,runningCount:runningCount,status:status}"

aws ecs update-service --cluster devmlops-cluster --service devmlops-service --desired-count 1
aws ecs wait services-stable --cluster devmlops-cluster --services devmlops-service
```

- ALB Target Group이 healthy가 되기까지 몇 분 걸릴 수 있다.

정상 요청과 오류 요청을 각각 보낸다.

```powershell
Invoke-RestMethod http://<ALB-DNS-이름>/health

Invoke-RestMethod http://<ALB-DNS-이름>/predict -Method Post -ContentType "application/json" `
  -Body '{"sepal_length":5.1,"sepal_width":3.5,"petal_length":1.4,"petal_width":0.2}'

# 의도적으로 422를 만든다
Invoke-RestMethod http://<ALB-DNS-이름>/predict -Method Post -ContentType "application/json" `
  -Body '{"sepal_length":-1,"sepal_width":3.5,"petal_length":1.4,"petal_width":0.2}'
```

```powershell
$logStreamName = aws logs describe-log-streams --log-group-name /ecs/devmlops-fastapi `
  --order-by LastEventTime --descending --max-items 1 `
  --query "logStreams[0].logStreamName" --output text

aws logs get-log-events --log-group-name /ecs/devmlops-fastapi --log-stream-name $logStreamName
```

- uvicorn 접근 로그에서 **응답 시간**을, 422 기록에서 **검증 오류**를 확인.
- 어제 테스트로 검증했던 Pydantic 방어선이 운영 로그에서 어떤 모양인지 확인하는 단계.

### Step 4. `main.py`에 추론 로깅 추가 (로컬 검증)

```powershell
cd ch03-1
Copy-Item ..\ch04-1\main.py .\src\devmlops_ml\main.py -Force
python -m devmlops_ml.train
python -m uvicorn devmlops_ml.main:app --host 127.0.0.1 --port 8000 --reload
```

다른 터미널에서 요청:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/predict -Method Post -ContentType "application/json" `
  -Body '{"sepal_length":5.1,"sepal_width":3.5,"petal_length":1.4,"petal_width":0.2}'
```

```
INFO:devmlops_ml.inference:predict request={'sepal_length': 5.1, ...} prediction=setosa probability=0.97..
```

- 로그가 안 나오면 `logging.basicConfig(level=logging.INFO)`의 **위치**를 본다. 그 이전에 다른 핸들러가 설정되면 먹지 않는다.
- 여기서 입력값이 통째로 로그에 찍히는 걸 눈으로 확인했다. 관측성과 개인정보의 충돌이 코드 한 줄에서 발생하는 지점.

### Step 5. 변경 사항 배포

```powershell
py -m pytest -v      # main.py 변경 후 기존 테스트가 여전히 통과하는지
cd ..
git add ch03-1/src/devmlops_ml/main.py
git commit -m "Add inference logging"
$branch = git branch --show-current
git push origin $branch
```

PR → Checks(`test-and-validate`, `docker-build`) 통과 → main 병합 → Actions에서 `ml-cicd` 실행 확인 → `deploy` job이 승인 대기면 **Review deployments** → production 승인.

- **이 배포는 champion 별칭 조건을 거치지 않는다.** `ml-cicd`는 코드 변경을 자동 배포하는 별도 경로이고, 오전에 만든 `ml-deploy`의 게이트와 무관하게 같은 ECS 서비스를 갱신한다.
- PR에서는 `deploy` job이 실행되지 않는 게 정상. main push에서만 실행된다.

### Step 6. 배포된 서비스에서 추론 로깅 확인

```powershell
Invoke-RestMethod http://<ALB-DNS-이름>/predict -Method Post -ContentType "application/json" `
  -Body '{"sepal_length":5.1,"sepal_width":3.5,"petal_length":1.4,"petal_width":0.2}'

$logStreamName = aws logs describe-log-streams --log-group-name /ecs/devmlops-fastapi `
  --order-by LastEventTime --descending --max-items 1 `
  --query "logStreams[0].logStreamName" --output text

aws logs get-log-events --log-group-name /ecs/devmlops-fastapi --log-stream-name $logStreamName
```

- `predict request=... prediction=... probability=...` 형식이 CloudWatch에 보이면 정상.
- 배포로 태스크가 교체되면 새 스트림이 생기므로 **최신 스트림을 다시 조회**해야 한다.

### Step 7. 운영 점검 체크리스트 작성

| 점검 항목 | 확인 방법 | 결과 |
|---|---|---|
| 헬스체크 상태 | `/health` 응답 | ✅ ok |
| 응답 시간 | CloudWatch의 uvicorn 접근 로그 | ✅ 확인 |
| 에러율 | 422·503 응답 발생 여부 | ✅ 422 재현 확인 |
| 배포된 모델 버전 | ch03-3에서 기록한 champion run_id·버전 | ✅ 기록 대조 |
| 추론 로깅 동작 | `/predict` 로그에 입력·예측 결과 포함 여부 | ✅ 확인 |

- 로그가 있어도 **점검 기준이 없으면 정상 여부를 판단할 수 없다**는 게 이 단계의 요지.

### Step 8. GitHub Issue로 개선 사항 정리

```
제목: [ch04-1] 운영 점검 결과 및 개선 사항

- 점검 일시:
- 확인한 항목: (Step 7 체크리스트 요약)
- 발견한 문제: (응답 시간이 느려지는 시점, 에러 발생 패턴 등)
- 제안하는 개선: (원격 MLflow 서버 도입, 알림 설정, 드리프트 탐지 도구 도입)
```

- 웹 UI 또는 `gh issue create --title "..." --body-file issue-body.md`.
- 점검 결과를 협업 도구에 남겨 **발견한 문제가 묻히지 않게** 하는 단계.

### Step 9~12. 리소스 정리

**① ECS 서비스**

```powershell
aws ecs update-service --cluster devmlops-cluster --service devmlops-service --desired-count 0
aws ecs delete-service --cluster devmlops-cluster --service devmlops-service --force
```

**② ALB · Target Group · Security Group** (Console)

- EC2 콘솔 → 로드 밸런서 → `devmlops-alb` 삭제
- 대상 그룹 → `devmlops-tg` 삭제
- 보안 그룹 → **삭제 가능한 실습용 SG만** 선택 삭제 (필요 시 상세에서 규칙 먼저 제거)

**③ ECR 이미지**

```powershell
aws ecr list-images --repository-name devmlops-fastapi --query "imageIds[*]" --output json

$imageIds = aws ecr list-images --repository-name devmlops-fastapi --query "imageIds" --output json
if ($imageIds.Count -gt 0) {
    $imageIdsJson = $imageIds | ConvertTo-Json -Compress
    aws ecr batch-delete-image --repository-name devmlops-fastapi --image-ids $imageIdsJson
}
```

- 리포지터리 자체 삭제는 선택(`aws ecr delete-repository --force`, 되돌릴 수 없음).

**④ IAM 역할**

- `github-actions-ecs-deploy` 삭제. 붙어 있던 정책:
  - `AmazonEC2ContainerRegistryPowerUser`
  - `AmazonECS_FullAccess`
  - `ecs-pass-execution-role` (인라인)

```powershell
aws iam list-attached-role-policies --role-name github-actions-ecs-deploy
aws iam detach-role-policy --role-name github-actions-ecs-deploy --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
aws iam detach-role-policy --role-name github-actions-ecs-deploy --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess
aws iam delete-role-policy --role-name github-actions-ecs-deploy --policy-name ecs-pass-execution-role
aws iam delete-role --role-name github-actions-ecs-deploy
```

- **관리형 정책은 분리(detach), 인라인 정책은 삭제(delete)** 후에야 역할을 지울 수 있다. 순서가 정해져 있는 이유가 참조 관계 때문이라는 게 명확했다.
- `ecsTaskExecutionRole`과 OIDC Provider(`token.actions.githubusercontent.com`)는 **재사용 가능성 때문에 남긴다.** 정리도 일괄 삭제가 아니라 판단이 필요한 작업.

### Step 13. 잔여 리소스 확인

```powershell
aws ecs list-services --cluster devmlops-cluster
aws elbv2 describe-load-balancers --query "LoadBalancers[?LoadBalancerName=='<ALB-이름>']"
aws ecr describe-images --repository-name devmlops-fastapi
aws iam get-role --role-name github-actions-ecs-deploy
```

- ECS·ALB는 빈 목록, ECR은 빈 목록 또는 `RepositoryNotFoundException`, IAM은 `NoSuchEntity`가 나오면 정리 완료.
- **"지웠다고 생각하는 것"과 "실제로 지워진 것"을 대조하는 단계.** 배포 검증과 같은 구조다.

---

## 결과 / 산출물

- Console·CLI 양쪽에서 조회한 CloudWatch 로그 그룹·스트림 확인 기록
- 정상 요청 / 422 오류 요청의 로그 형태 비교
- 추론 로깅이 추가된 `main.py` — 로컬 검증 → `ml-cicd` 배포 → 운영 로그 확인 3단계 완료
- 운영 점검 체크리스트 5항목
- GitHub Issue `[ch04-1] 운영 점검 결과 및 개선 사항`
- ECS·ALB·TG·SG·ECR·IAM 정리 및 잔여 리소스 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| Console → CLI 순으로 같은 정보를 두 경로로 확인해 자동화 가능한 형태를 확보 | 드리프트를 실제로 재현해보지는 못함. 개념으로만 이해한 상태 |
| 로컬 → 배포 → 운영 로그 3단계로 로깅 코드 검증 | 추론 로깅에서 입력값 마스킹은 적용하지 않음. 원칙만 알고 구현은 안 해봄 |
| 정상·오류 요청을 나란히 보내 로그 형태를 비교 | `ml-cicd`가 champion 게이트를 우회한다는 걸 확인만 하고, 우회를 막는 구성은 시도하지 않음 |
| 정리 순서(의존성 역순)와 남길 리소스 판단을 구분해 수행 | 응답 시간·에러율의 정상 범위 기준을 숫자로 정하지 못함. 체크리스트는 만들었지만 임계값은 없음 |

---

## 다음에 이어서 할 것

- 인프라를 정리했으므로 이후 실습에서 배포 검증이 필요하면 ch02-3 구성부터 다시 생성해야 함
- 운영 점검 체크리스트의 항목별 정상 범위(응답 시간·에러율 임계값)를 숫자로 정해보기

## 다음에 파고들 만한 질문

- 개념 드리프트는 정답을 알아야 감지되는데, 운영 중 정답을 바로 알 수 없는 도메인에서는 실제로 어떤 대리 지표(proxy)를 쓰나?

---
*작성일: 2026-07-21 | 인공지능 사관학교 AI 보안 과정*
