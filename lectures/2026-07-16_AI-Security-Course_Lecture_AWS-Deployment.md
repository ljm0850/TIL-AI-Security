# 2026-07-16 강의 정리 — AWS 배포 (ECR · ECS · ALB · OIDC)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: 컨테이너 이미지를 AWS로 배포 — 배포 아키텍처, IAM·OIDC 인증, ECR/ECS 자동화, 배포 검증·롤백 (ch02.3)
> **관련 강의 파일**: `2026-07-15_AI-Security-Course_Lecture_Docker-Intro.md` (Docker 파트)
> **관련 실습 파일**: `../practice/2026-07-16_AI-Security-Course_Practice_AWS-ECS-Deployment.md`

---

## 핵심 인사이트 모음

### 📅 2026-07-16

- AWS 배포는 결국 **"저장(ECR) / 실행(ECS) / 트래픽(ALB)"의 관심사를 나눈 것**. 나눠두면 한 영역 설정을 바꿔도 전체 배포 구조를 한꺼번에 안 건드린다. 로컬 `docker run` 한 줄이 운영에선 이렇게 세 갈래로 쪼개진다는 감각.
- **ECS/Fargate**: 로컬과 운영의 차이 = 비정상 종료 시 재실행 + 필요한 개수 유지 + 무중단 배포. Fargate는 서버 관리 대신 태스크 정의·서비스 설정에 집중하게 해줌. 다만 **GPU 세팅 같은 하드웨어 구성·섬세한 인스턴스 제어·비용 최적화가 필요하면 EC2 방식**. (Fargate는 GPU 미지원이라 ML/GPU 워크로드는 EC2 런치 타입이 필요 — AI 쪽 작업이면 바로 걸리는 지점)
- (보안 렌즈) **OIDC vs Access Key가 오늘의 핵심**. 어제 적은 "secrets 표면 좁히기" 고민의 실물 해답이다. 장기 키를 GitHub Secrets에 저장하지 않고, 워크플로우 실행마다 STS로 짧게 사는 임시 자격증명을 발급받고, Trust Policy로 특정 repo/branch만 role을 맡게 제한. 유출돼도 (1) 애초에 저장된 장기 키가 없고 (2) 임시 증명은 금방 만료되니 blast radius가 확 준다.
- (보안 렌즈) **역할 3분리** — 학습자 CLI 사용자 / GitHub Actions OIDC 배포 role / ECS Task Execution Role이 용도별로 권한 범위가 다르다. 최소권한·격리 원리
- **선언적 배포** — 새 배포 = 실행 중 컨테이너를 직접 수정하는 게 아니라, 새 이미지 반영한 태스크 정의 리비전을 등록하고 서비스가 그 리비전을 쓰도록 갱신. 태스크 정의=설계문서, 서비스=그 문서대로 실제 상태를 유지하는 관리자. (K8s에서 본 "원하는 상태 선언 → 시스템이 수렴"이랑 같은 결)
- **롤백의 한계** — 이전 리비전이 안 지워지고 남아있어서 서비스 업데이트 한 번으로 복귀 가능. 단 앱 코드·컨테이너 실행 기준만 되돌리고, **DB 스키마 변경이나 외부 상태는 자동 복원 안 됨**. 롤백=만능 undo가 아니라는 것.
- **배포 Workflow 성공 ≠ 서비스 정상**. Workflow 성공은 "자동화 명령이 오류 없이 끝났다"에 가깝다. 실제 사용자 접근 서비스가 정상 응답하는지는 Console/CLI로 태스크 수·리비전을 대조해야 알 수 있음.

---

## 1. AWS 배포 아키텍처

### 핵심 개념
- Docker 이미지를 실제 서비스로 운영하려면 세 구성: **이미지를 저장할 위치(ECR) / 실행할 환경(ECS) / 사용자 요청을 전달할 경로(ALB)**.

### 상세 내용
- **관심사 분리 이유**: 저장·실행·트래픽 처리는 서로 다른 관심사. ECR은 버전 관리·접근 권한이, ECS는 장애 복구·실행 개수 유지가, ALB(외부 요청 계층)는 트래픽 분산·헬스체크가 중요.
- 역할을 나눠두면 특정 영역 설정을 바꿔도 전체 배포 구조를 한꺼번에 바꾸지 않고 관리 가능.
- **진행 흐름**: 배포 아키텍처 → IAM·OIDC 인증 → ECR 자동화 → ECS 자동화 → 배포 검증·롤백. Console(리소스 관계를 눈으로) → CLI(설정값을 명령으로 검증) → GitHub Actions(둘을 배포 자동화로 연결).

---

## 2. ECR (이미지 레지스트리)

### 핵심 개념
- ECR(Elastic Container Registry) = Docker 이미지를 저장하는 AWS의 프라이빗 레지스트리.

### 상세 내용
- 로컬 이미지를 ECR에 올리면 ECS가 pull해서 컨테이너로 실행. ECR은 단순 파일 저장소가 아니라 **배포 가능한 이미지 버전을 관리하는 기준**.
- IAM 권한으로 접근 제어 → AWS 계정 안에서 이미지 저장·배포 권한을 함께 관리.
- **리포지터리(Repository)**: ECR 안에서 이미지를 모아두는 단위. 보통 앱 하나에 리포 하나, 커밋 해시 태그로 여러 버전 관리. 같은 앱이라도 태그가 다르면 서로 다른 배포 후보.
- **Repository URI**: Push/Pull 시 쓰는 주소. 계정 ID·리전 정보 포함 → 이미지가 어느 계정·리전에 저장되는지 확인 가능. ECS 태스크 정의에서 실행할 이미지 지정에 사용.

---

## 3. ECS와 Fargate

### 핵심 개념
- ECS(Elastic Container Service) = 컨테이너를 실행·관리하는 서비스.

### 상세 내용
- 로컬 `docker run` 한 번과 달리, 운영은 비정상 종료 시 재실행 + 필요한 개수 유지 + 배포 중 서비스 중단 최소화 관리가 필요. ECS는 이걸 **서비스 단위로 처리**.
- **Fargate**: ECS에서 컨테이너를 실행하는 방식 중 하나. EC2 방식은 서버를 직접 만들고 관리, Fargate는 서버 관리보다 태스크 정의·서비스 설정에 집중. EC2가 불필요하다는 뜻은 아니고 — 섬세한 인스턴스 제어, 특정 하드웨어 구성(GPU 등), 비용 최적화가 중요하면 EC2가 더 적합.
- **클러스터(Cluster)**: 서비스·태스크를 묶어 관리하는 논리 단위.
- **태스크 정의(Task Definition)**: 어떤 이미지를 얼마의 CPU·메모리로, 어떤 포트로 실행할지 정의한 JSON. 새 이미지 배포 = 실제로는 새 리비전(Revision)의 태스크 정의를 등록하는 작업.
- **서비스(Service)**: 지정한 개수의 태스크가 계속 실행되도록 유지. 태스크가 죽으면 새 태스크를 실행해 개수를 맞춤.
- **태스크(Task)**: 태스크 정의 기준으로 실제 실행된 컨테이너 묶음. 단일 컨테이너 앱에선 로컬 컨테이너 하나와 비슷한 단위.

---

## 4. ALB와 트래픽 분산

### 핵심 개념
- ALB(Application Load Balancer) = 외부 요청을 받아 실행 중인 태스크로 전달.

### 상세 내용
- 태스크가 하나뿐이면 ALB 없이도 되지만, 운영은 배포 중 태스크 교체·개수 증가가 생김. ALB를 두면 사용자는 고정 진입점으로 요청을 보내고, 뒤쪽 태스크 변경은 ALB+ECS가 함께 처리. 새 태스크가 헬스체크를 통과하면 트래픽을 보내고, 문제 태스크는 대상에서 제외 → 배포·확장에 유리.
- **Target Group**: ALB가 트래픽을 보낼 태스크를 등록해두는 그룹. ECS가 태스크를 만들거나 종료하면 등록 상태도 함께 변경.
- **Listener**: ALB가 어떤 포트로 받아 어떤 Target Group으로 보낼지 정하는 규칙 (예: 80 HTTP → 특정 TG).
- **Security Group**: 인스턴스·태스크로 들어오고 나가는 트래픽을 포트·프로토콜 단위로 허용/차단하는 방화벽 규칙. ALB와 ECS 태스크의 접근 범위를 나눠 설정하면 외부 노출 범위를 줄일 수 있음.
- **헬스체크(Health Check)**: ALB가 주기적으로 `/health` 같은 경로를 호출해 태스크가 요청 처리 가능 상태인지 확인. 실패한 태스크는 Target Group에서 제외되어 트래픽을 안 받음.

---

## 5. IAM과 OIDC 인증

### 핵심 개념
- GitHub Actions가 AWS 리소스에 접근하려면 인증이 필요. 인증 방식은 배포 편의성뿐 아니라 **보안 사고 시 피해 범위**에도 영향을 준다.

### 상세 내용
- **IAM 주요 용어**:
  - **User(사용자)**: 사람/앱을 나타내는 개별 자격 증명. 장기 Access Key 발급 가능하나, 자동화 환경에 고정 키를 저장하면 유출 위험↑.
  - **Role(역할)**: 특정 권한 집합을 임시로 위임받아 사용. 필요할 때만 발급받고 일정 시간 지나면 만료.
  - **Policy(정책)**: 어떤 리소스에 어떤 동작을 허용할지 정의한 JSON (예: ECR Push, ECS 서비스 업데이트 권한).
  - **Trust Policy(신뢰 정책)**: 역할을 누가 맡아 쓸 수 있는지 정의. GitHub Actions가 역할을 쓰려면 신뢰 정책에 GitHub OIDC 공급자 + 허용할 리포지터리 조건이 포함돼야 함.
- **Access Key 방식의 한계**: GitHub Secrets에 AWS Access Key/Secret Key를 저장해 쓰는 흔한 방식. 설정은 단순하지만, 키 값이 유출되면 만료 전까지 누구나 같은 권한으로 접근 가능. 키 교체 주기·보관 위치를 별도 관리해야 하고, 여러 리포/환경을 운영하면 부담↑. → 장기 키를 줄이는 이유는 편의성보다 유출 시 피해 범위를 줄이는 게 더 중요하기 때문.
- **OIDC(OpenID Connect) 방식**: GitHub Actions가 실행될 때마다 AWS에 임시 자격 증명을 요청. 장기 비밀 키를 저장하지 않으므로 유출 위험↓, 발급된 증명은 짧은 시간만 유효(노출돼도 사용 가능 시간 제한). 대신 신뢰 정책 설정이 한 단계 더 필요 — AWS가 OIDC 토큰을 확인해 어떤 리포/브랜치에서 실행된 Workflow인지 검증. 설정은 복잡해지지만 배포 주체·권한 범위를 더 좁게 제한.
- **OIDC 인증 흐름**:
  1. Workflow 실행 시 자신의 신원을 증명하는 OIDC 토큰을 발급받음.
  2. 그 토큰으로 AWS STS(Security Token Service)에 IAM 역할 사용을 요청.
  3. AWS가 역할의 신뢰 정책을 확인해 GitHub Actions의 리포/브랜치 조건이 맞는지 검증.
  4. 검증 성공 시 짧게 유효한 임시 자격 증명 발급.
  5. Workflow는 그 임시 증명으로 ECR Push, ECS 배포를 실행.

| 인증 방식 | 장점 | 한계 |
|---|---|---|
| Access Key | 설정 단순, 초기 학습 쉬움 | 키 유출 시 만료 전까지 위험 지속, 주기적 교체 관리 필요 |
| OIDC | 장기 비밀 키 줄임, 자격 증명이 짧게 유지 | 신뢰 정책·권한 조건을 정확히 설정해야 함 |

---

## 6. ECR 자동화 (GitHub Actions)

### 핵심 개념
- ECR 자동화 = 로컬에서 만든 이미지를 AWS 레지스트리에 올리는 과정. Console에서 리포 생성 → CLI로 조회 대조 → Actions에서 빌드·Push 자동화.

### 상세 내용
- **Console→CLI 확인 흐름**: Console에서 ECR 리포 생성 시 Repository URI를 화면에서 확인 (이 값은 이미지 태그·ECS 태스크 정의에서 계속 쓰이므로 처음부터 정확히). CLI로 같은 값을 조회해 Console에서 만든 리소스가 실제 계정·리전에 존재하는지 대조.
- **ECR Push Workflow**: 도커 빌드 검증 단계에 AWS 인증 + 이미지 Push를 더한 구성. OIDC로 역할을 맡고 → ECR 로그인 → 빌드한 이미지를 Repository URI + 커밋 해시 태그로 Push.
- `permissions: id-token: write`: Workflow가 OIDC 토큰을 발급받도록 허용하는 설정. 없으면 역할을 맡기 위한 토큰을 못 받음.
- `aws-actions/configure-aws-credentials`: OIDC 토큰으로 지정한 IAM 역할의 임시 자격 증명을 env로 설정하는 공식 액션.
- `aws ecr get-login-password` / `amazon-ecr-login`: 임시 자격 증명으로 ECR 로그인.

```bash
aws ecr describe-repositories --repository-names devmlops-fastapi
```

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: ${{ secrets.AWS_REGION }}

  - name: Login to Amazon ECR
    id: ecr-login
    uses: aws-actions/amazon-ecr-login@v2

  - name: Build and push image
    env:
      REGISTRY: ${{ steps.ecr-login.outputs.registry }}
      REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
      IMAGE_TAG: ${{ github.sha }}
    run: |
      docker build -t "$REGISTRY/$REPOSITORY:$IMAGE_TAG" ch02-2
      docker push "$REGISTRY/$REPOSITORY:$IMAGE_TAG"
```

---

## 7. ECS 자동화 (GitHub Actions)

### 핵심 개념
- 이미지가 ECR에 올라간 뒤 그 이미지를 실행할 ECS 서비스를 구성. 핵심은 실행 중 컨테이너를 직접 수정하는 게 아니라, **새 이미지를 반영한 태스크 정의 리비전을 만들고 서비스가 그 리비전을 쓰도록 갱신**하는 것.

### 상세 내용
- 태스크 정의 = 컨테이너 실행 방법을 담은 설계 문서, 서비스 = 그 문서 기준으로 실제 태스크를 유지하는 관리자.
- 컨테이너를 직접 중지·재시작하면 순간 서비스가 끊길 수 있음. ECS 서비스 업데이트는 **새 태스크를 먼저 실행하고 헬스체크 통과 후 이전 태스크를 정리** → 운영 중 서비스 중단 축소.
- `amazon-ecs-render-task-definition`: 태스크 정의 템플릿의 이미지 자리표시자를 실제 이미지 URI로 바꿔 새 태스크 정의 파일 생성.
- `amazon-ecs-deploy-task-definition`: 렌더된 태스크 정의를 새 리비전으로 등록하고, 지정 서비스가 그 리비전을 쓰도록 업데이트.
- `wait-for-service-stability: true`: 배포 명령이 즉시 끝나는 걸 막고 서비스가 안정 상태(원하는 태스크 수만큼 정상 실행)에 도달할 때까지 대기 → 배포 직후 실패를 빨리 확인. 단 헬스체크 설정이 잘못돼 있으면 오래 대기하거나 실패할 수 있어, 헬스체크 경로와 앱 실제 응답이 맞는지 함께 확인해야 함.

```yaml
- name: Render new task definition
  id: render
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: ch02-3/task-definition.json
    container-name: devmlops-fastapi
    image: ${{ steps.build.outputs.image }}

- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v2
  with:
    task-definition: ${{ steps.render.outputs.task-definition }}
    service: ${{ secrets.ECS_SERVICE }}
    cluster: ${{ secrets.ECS_CLUSTER }}
    wait-for-service-stability: true
```

---

## 8. 배포 검증과 롤백

### 핵심 개념
- 배포 Workflow가 성공했다고 서비스가 정상 동작한다고 바로 판단할 수 없음. 검증은 배포 실패가 여러 위치에서 발생할 수 있기 때문에 필요.

### 상세 내용
- Workflow 성공 = 자동화 명령이 오류 없이 끝났다는 의미에 가깝고, 사용자가 실제 접근하는 서비스의 정상 응답까지 보장하진 않음. 배포 후 GitHub Actions 로그 + AWS Console 상태 + CLI 조회를 함께 확인.
- **실패가 날 수 있는 위치**: 이미지는 ECR에 정상 Push됐지만 ECS 서비스 업데이트 실패 / 태스크는 실행됐지만 ALB 헬스체크 실패 / 앱은 실행 중이어도 env·포트가 안 맞아 외부 요청 처리 실패.
- **Console/CLI 대조**: 실행 중 태스크 수 = desired 수인지, 태스크 정의 리비전이 예상값인지. 다르면 서비스가 아직 안정 상태가 아니거나, Workflow가 다른 리소스를 바라보고 있거나, env 설정이 잘못됐을 수 있음.
- **롤백**: ECS 태스크 정의는 등록마다 리비전 번호가 증가하고 이전 리비전은 삭제되지 않고 남음. 롤백은 이 특성을 이용해 서비스가 쓸 정의를 이전 리비전으로 되돌리는 것 → 재빌드나 코드 되돌리기 없이 서비스 업데이트 한 번으로 실행 기준 복귀. 단 앱 코드·컨테이너 실행 기준만 되돌리므로 **DB 스키마 변경·외부 상태는 자동 복원 안 됨**. 롤백 대상 리비전은 임의로 고르지 말고, 배포 로그·이전 정상 배포 기록을 확인해 선택. 배포 전후 이미지 태그·커밋 해시·리비전을 함께 기록해두면 판단이 쉬워짐.

```bash
# Console/CLI 대조
aws ecs describe-services --cluster devmlops-cluster --services devmlops-service \
  --query "services[0].{running:runningCount, desired:desiredCount, taskDef:taskDefinition}"

# 이전 리비전으로 롤백
aws ecs update-service --cluster devmlops-cluster --service devmlops-service \
  --task-definition devmlops-fastapi:12
```

*작성일: 2026-07-16 | 인공지능 사관학교 AI 보안 과정*
