# 2026-07-16 실습 — AWS ECS 배포 (ch02-3)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (ch02-3)
> **관련 강의 파일**: `../lectures/2026-07-16_AI-Security-Course_Lecture_AWS-Deployment.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-16

- (보안) **OIDC 역할 구성이 이 실습의 핵심 손맛**이었다. Trust Policy에서 `sub` 조건을 `repo:<org>/<repo>:*`로 직접 제한해 다른 저장소가 이 역할을 못 맡게 막는 걸 손으로 해봄. 어제~오늘 계속 나온 "secrets 표면 좁히기"가, 여기선 "누가 이 역할을 맡을 수 있나"를 문자열 조건으로 자르는 형태로 나타났다.
- (보안) **역할이 용도별로 갈린다는 걸 실제로 만들며 체감** — 학습자 CLI용 IAM User / GitHub Actions OIDC 배포 role / ECS Task Execution Role. 각각 붙이는 정책이 다르고, 실습 편의로 AdministratorAccess를 쓰되 "운영에선 최소 7개 정책으로 축소"하라는 안내가 있었음. `iam:PassRole`을 `ecs-tasks`로만 조건 건 인라인 정책도 직접 붙임.
- **ECR=저장 / ECS=실행 / ALB=트래픽** 이라는 분담이, Console에서 리소스를 하나씩 만들며 눈에 들어왔다. 특히 보안그룹을 ALB(80, `0.0.0.0/0`)와 태스크(8000, ALB SG에서만)로 나눠 여는 게 곧 외부 노출 최소화라는 감각.
- (보안 ↔ 운영 tension) Step 8에서 헬스체크 타임아웃으로 한참 막혔는데, 원인이 **ALB에 ALB용 SG가 아니라 Task용 SG가 잘못 붙어있던 것**이었다. 애초에 기본 SG를 예전에 손댄 게 있어서 임시로 새 SG를 만들어 쓰던 상태라, 이름 비슷한 SG가 여러 개 떠서 생성 화면에서 잘못 골랐다. → **격리를 세밀하게 나눌수록(ALB SG / Task SG 분리) 보안은 좋아지지만, "어느 리소스에 어느 SG"를 정확히 매핑하는 게 그만큼 실수 표면이 된다**는 걸 몸으로 배움. 최소권한·격리의 실전 비용.
- (디버깅 사고 도구) **"Request timed out"은 404/500 같은 앱 오류와 달리 거의 항상 네트워크 경로 문제**다. SYN에 대한 응답 자체가 없다는 뜻이라, 코드보다 SG·라우팅·NACL을 먼저 의심하는 게 효율적. 네트워크 계층과 앱 계층을 나눠 생각하는 게 핵심이었다.
- **로그는 "기동됐다"까지만 알려주지 "외부에서 접근 가능한지"는 모른다.** ECS Exec으로 컨테이너 안에 직접 들어가 `curl 127.0.0.1:8000/health`를 때려 앱이 완전히 정상임을 먼저 확정한 게 결정적이었음 — 그때부터 문제를 ALB↔Task 네트워크 경로로 좁힐 수 있었다. 소거법(SG→TG→NACL→라우팅→포트매핑→내부 curl→ALB SG)이 실제로 통했다. 참고로 desired 1인데 태스크가 3~4개 뜨는 스래싱 자체가 "뭔가 반복 실패 중"이라는 신호였다.
- (다음 프로젝트용 메모) **스트레스 테스트**, 팀 프로젝트 과정에서 간단하게라도 해보고 기록해두면 괜찮을 듯. (ALB·desired-count를 만지다 보니 실제 부하를 걸어보고 싶어짐)
- 로컬 `docker run`에서 시작한 게 결국 퍼블릭 ALB DNS로 `/health` 응답을 받는 데까지 이어졌다. push → 빌드 → ECR → ECS → ALB가 한 줄로 연결되는 걸 막힘 구간까지 뚫고 끝까지 통과해본 게 이번 실습의 가장 큰 수확.

---

## 실습 개요
- **목표**: ch02-2 Docker 이미지를 AWS로 올려 인터넷에서 접근 가능한 서비스로 만들기. Console→CLI→GitHub Actions 순으로 리소스를 만들고 대조하고 자동화.
- **방법**: AWS CLI 자격증명 → OIDC IAM 역할 → ECR 리포 → GitHub Secrets → ECR Push 워크플로우 → ECS 클러스터/태스크정의/서비스 → 보안그룹·ALB → ECS 배포 워크플로우 → 검증·롤백.
- **전제**: ch02-2가 main에 병합돼 있어야 함 (빌드 대상이 ch02-2 폴더).

---

## 진행 과정

> 세션 설정·자격증명 입력값은 축약. AWS 콘솔 클릭 경로는 핵심만.

### Step 1~2. AWS CLI 자격증명 (학습자용)
- 루트 키 아님 — 실습용 IAM User Access Key 발급 → `aws configure`(서울 리전 `ap-northeast-2`) → `aws sts get-caller-identity`로 인증 주체 확인. 이 키는 Console/CLI 확인용이고, 배포에 쓰는 OIDC 역할과는 **별개**.

### Step 3. OIDC IAM 역할 구성 ★
- OIDC Provider 등록: `token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
- Web identity 역할 생성 + `AmazonEC2ContainerRegistryPowerUser`, `AmazonECS_FullAccess`.
- **Trust Policy 제한**: `sub`를 `repo:<org>/<repo>:*` (StringLike) 로 수정 → 다른 저장소가 역할 못 맡게 차단.
- **PassRole 인라인 정책**: `iam:PassRole`을 `iam:PassedToService: ecs-tasks.amazonaws.com` 조건으로만 허용 (ECS Task Execution Role 전달용).

### Step 4~7. ECR + ECR Push 워크플로우
- Console에서 프라이빗 리포 `devmlops-fastapi` 생성 → Repository URI 복사 → CLI `aws ecr describe-repositories`로 대조.
- GitHub Environment(`production`) Secrets: `AWS_ROLE_ARN` / `AWS_REGION` / `ECR_REPOSITORY`.
- `ecr-push.yml`을 `ch02-3-cd` 브랜치에 push → Configure AWS credentials / Login to ECR / Build and push 스텝 성공 확인 (빌드 대상은 ch02-2 폴더).

### Step 8~11. ECS 구성(Console) + CLI 대조
- 클러스터(Fargate) `devmlops-cluster` → ECS Task Execution Role(`ecsTaskExecutionRole`, `AmazonECSTaskExecutionRolePolicy`) → 태스크 정의(`devmlops-fastapi`, 0.25 vCPU / 0.5 GB, 포트 8000, 로그그룹 `/ecs/devmlops-fastapi`) → 서비스(`devmlops-service`, desired 1, ALB `devmlops-alb` / TG `devmlops-tg` / 헬스체크 `/health`).
- **보안그룹 인바운드**: 외부→ALB(80, `0.0.0.0/0`), ALB→태스크(8000, ALB SG). ALB/태스크가 같은 SG면 한 SG에 두 규칙.
- Target Group Targets가 `healthy`인지, CLI `describe-services`로 running=desired인지 대조.
- ⚠️ **여기서 크게 막힘** — Target이 계속 `Unhealthy (Request timed out)`, 태스크 스래싱. 원인·소거법·해결은 아래 `## 🔧 문제 해결` 섹션 참조.

### Step 12~13. ECS 배포 워크플로우 + 최종 대조
- GitHub Secrets `ECS_CLUSTER`/`ECS_SERVICE` 등록, `task-definition.json` 자리표시자 채움.
- `ecs-deploy.yml` push → render / deploy 스텝, `wait-for-service-stability` 안정 도달 확인.
- Console/CLI 대조 + `http://<ALB-DNS>/health`로 퍼블릭 응답 확인.

### Step 14~16. 롤백 / 병합 / 비용 관리
- `aws ecs list-task-definitions`로 이전 리비전 확인 후 `update-service --task-definition ...:<이전>`으로 롤백 (리비전 2개 이상일 때만).
- `ch02-3-cd` → main 병합.
- 실습 안 할 땐 `update-service --desired-count 0`으로 Fargate 실행 비용 절감 (ALB/ECR 비용은 별도로 남음).

---

## 🔧 문제 해결 — ECS/ALB 헬스체크 타임아웃 (Step 8)

> 서비스 생성 후 Target Group 대상이 계속 `Unhealthy (Request timed out)`. 태스크가 STOPPED→재생성 스래싱(desired 1인데 3~4개), ALB에 "Reachability may be impacted" 경고.

**근본 원인** — ECS 서비스 생성 화면에서 로드밸런서를 구성할 때 **ALB에 붙일 SG로 `devmlops-alb-sg`가 아니라 `devmlops-task-sg`가 잘못 선택**돼 있었다.
- (배경) 기본 SG를 예전에 손댄 게 있어 임시로 새 SG들을 만들어 쓰던 상태 → 이름 비슷한 SG가 여러 개 떠서 오선택.
- `devmlops-alb-sg`(의도한 ALB용): 인바운드 `80/tcp ← 0.0.0.0/0`
- `devmlops-task-sg`(실제로 ALB에 붙어있던 것): 인바운드 `8000/tcp ← alb-sg만 허용`
- → ALB가 외부 80 요청을 받는 규칙도, Task 8000으로 내보내는 데 필요한 규칙도 없는 상태 → ALB→Task 8000 헬스체크 요청이 전달되지 못하고 타임아웃.

**소거법 점검** (범위 좁히기)

| # | 점검 항목 | 결과 |
|---|---|---|
| 1 | Task SG 인바운드 (8000 ← ALB SG) | 정상 |
| 2 | CloudWatch 로그 (`/ecs/devmlops-fastapi`) | Uvicorn 기동·`startup complete` 정상 |
| 3 | Target Group 설정 (IP, HTTP:8000) | 정상 |
| 4 | 헬스체크 (`/health`, timeout 5s, interval 30s) | 정상 |
| 5 | Task ENI에 붙은 SG | `task-sg` 정상 |
| 6 | ALB SG 인바운드(80)/아웃바운드 | 정상 |
| 7 | 서브넷 라우팅 (`0.0.0.0/0 → igw`) | 정상 |
| 8 | NACL (기본, ALL ALLOW) | 정상 |
| 9 | 태스크 정의 포트 매핑 (`8000:8000 tcp`) | 정상 |
| 10 | **ECS Exec 컨테이너 내부 curl** | `127.0.0.1:8000/health`·프라이빗IP:8000/health 둘 다 `{"status":"ok"}` → **앱은 완전 정상** |
| 11 | **ALB에 실제 붙은 SG 확인** | ❌ `alb-sg`가 아니라 `task-sg`가 붙어있었음 (원인 특정) |

10번에서 컨테이너 자체가 정상임을 확정 → 문제를 ALB↔Task 네트워크 경로로 좁힘 → 11번에서 원인 특정.

**해결** — EC2 콘솔 → Load Balancers → `devmlops-alb` → 보안(Security) 탭 → 편집 → SG를 `devmlops-task-sg` → `devmlops-alb-sg`로 교체 → 1~2분 후 Target 상태 `healthy` 전환.

**사용한 도구/기법**
- **CloudWatch Logs**: 앱 기동 여부 1차 확인.
- **ECS Exec** (`aws ecs execute-command`): 컨테이너 내부에 직접 접속해 응답 검증. 사전 준비 — 태스크 정의에 `taskRoleArn`(`ssmmessages:*` 권한), 서비스 `--enable-execute-command`로 재배포, 로컬에 Session Manager Plugin 설치.
- **소거법**: SG → TG → NACL → 라우팅 → 포트 매핑 → 내부 curl → ALB SG 순으로 하나씩 배제하며 범위 축소.

---

## 결과 / 산출물
- OIDC 배포 역할 + 스코핑된 Trust Policy / PassRole 인라인 정책.
- ECR 리포 + ECR Push 워크플로우.
- ECS 클러스터 / 태스크 정의 / 서비스 + ALB / Target Group.
- ECS 배포 워크플로우(render→deploy, wait-for-service-stability) + 퍼블릭 엔드포인트 `/health` 응답 확인.
- 롤백 절차 확인, `ch02-3-cd` → main 병합.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| Console→CLI 대조로 리소스가 실제 계정/리전에 있는지 확인하며 진행 | Step 8 헬스체크 타임아웃 — ALB에 Task용 SG가 잘못 붙어 스래싱 (임시 SG 여러 개 중 오선택) |
| Trust Policy·PassRole을 조건으로 좁히는 걸 손으로 해봄 | 로그만으론 원인 못 좁혀서, ECS Exec 내부 curl까지 가서야 앱 정상을 확정 |
| 소거법 + ECS Exec으로 ALB↔Task 네트워크 경로 문제로 원인 특정·해결 | |

---

## 다음에 이어서 할 것
- ch03 이후 실습이 이 배포 기반(main의 ch02-2/ch02-3)을 이어받는다.
- 실습 안 하는 기간에는 `desired-count 0`으로 내려둘 것 (ALB·ECR 비용은 별도로 남음).

---
*작성일: 2026-07-16 | 인공지능 사관학교 AI 보안 과정*
