# 2026-07-10 실습 — Color API 배포 & Deployment/ClusterIP

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-10_AI-Security-Course_Lecture_K8s-ReplicaSet-Deployment.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-10

- 로컬에서 만든 Express 앱(Color API)을 이미지로 빌드→Docker Hub에 푸시→K8s가 pull 받아 팟으로 띄우는 전체 파이프라인을 처음부터 끝까지 돌려봤다. 어제 배운 "로컬 build → 레지스트리 push → 서버 pull" 흐름이 K8s 클러스터를 대상으로 그대로 재현되는 걸 보니, Docker 지식이 K8s로 자연스럽게 이어진다는 게 실감났다.
- 가변 IP의 한계를 실습으로 직접 겪은 게 오늘의 핵심이었다. 트래픽 제너레이터가 특정 팟 IP를 하드코딩하고 있을 때 그 팟을 삭제하니, Deployment가 새 팟을 자동 생성(자가치유)했음에도 트래픽이 끊겼다. 새 팟은 완전히 다른 IP를 받았고, 제너레이터는 옛 IP만 바라보고 있으니 새 팟을 찾을 방법이 없었다. "자가치유가 되는데도 통신이 끊긴다"는 역설이 ClusterIP 서비스의 필요성을 몸으로 이해시켜줬다.
- ClusterIP를 붙이고 트래픽 제너레이터의 타겟을 팟 IP 대신 서비스 이름(`http://color-api-clusterip/api`)으로 바꾸니, 팟을 삭제해도 로그 스트리밍이 끊김 없이 이어졌다. 서비스가 죽은 팟을 빼고 살아있는 팟으로만 트래픽을 우회시키기 때문. 게다가 hostname 값이 계속 바뀌는 걸 보고 5개 팟에 라운드 로빈으로 분산되는 로드 밸런싱도 눈으로 확인함.
- 로그에 찍히는 hostname이 컨테이너 고유 해시값 앞부분과 일치한다는 걸 알게 됐다. os.hostname()이 컨테이너 ID를 반환하는 거라, 어떤 팟이 응답했는지 로그만 보고 추적할 수 있다는 게 디버깅 관점에서 유용해 보였다.

---

## 실습 개요
- **목표**: Color API 앱을 만들어 이미지로 빌드·푸시하고 K8s 팟으로 배포한다. Traffic Generator로 트래픽을 흘려보며, 팟 가변 IP의 한계를 실증하고 ClusterIP 서비스로 해결한다.
- **방법**: Express 앱 작성 → Docker 빌드/Hub 푸시 → 팟 배포·통신 검증 → Deployment(replicas 5) 배포 → 팟 IP 하드코딩의 한계 실습 → ClusterIP + 서비스 이름 통신으로 해결

---

## 진행 과정

### Step 1. Color API 앱 작성 & 로컬 Docker 검증
- `npm init -y` → `npm install express@4.9.2 --save-exact`
- `src/index.js`: 루트(`/`)에서 파란 글씨 HTML 반환, port 80 리스닝
- 이후 랩에서 `os.hostname()` 추가 + `?format=json` 분기(`/api` 라우트)로 확장, 버전 1.1.0으로 범핑
- `.dockerignore`(node_modules) + Dockerfile(`node:22-alpine`, `npm ci`, `CMD ["npm","start"]`)
```bash
docker build -t color-api .
docker run -p 3000:80 --name color-api color-api   # localhost:3000 확인
```

### Step 2. 이미지 빌드 → Docker Hub 푸시 → K8s 팟 배포
```bash
docker build -t [DOCKER_ID]/color-api:1.0.0 .
docker login -u [DOCKER_ID]        # 비밀번호 대신 PAT 입력 (토큰은 기록하지 않음)
docker push [DOCKER_ID]/color-api:1.0.0

kubectl run color-api --image=[DOCKER_ID]/color-api:1.0.0
kubectl get pods
kubectl logs color-api             # "color-API listening on port 80" 확인
```
- 클러스터 내부 통신 검증: Alpine 팟에서 `curl [color-api_팟_IP]` → 파란 HTML 응답
- 정리: `kubectl delete pod alpine --force` / `kubectl delete pod color-api --force` (실습 환경에서만 force 사용, 운영에선 지양)

### Step 3. Traffic Generator 제작 (Alpine + sh)
- `traffic-gen.sh`: 인자 2개(대상 URL, 인터벌) 유효성 검사 → 무한 루프로 curl 요청 + 타임스탬프 로그
  - **셔뱅 주의**: Alpine은 bash가 없으므로 `#!/bin/sh` (bash로 쓰면 실행 안 됨)
  - **조건문 주의**: `if [ $# -lt 2 ]`처럼 대괄호 안·연산자 전후 공백 필수
- Dockerfile: `alpine:3.20` + `apk add --no-cache curl` + 스크립트 복사·`chmod +x` + `ENTRYPOINT ["./traffic-gen.sh"]`
```bash
docker build -t [DOCKER_ID]/traffic-generator:1.0.0 .
docker run --rm [DOCKER_ID]/traffic-generator:1.0.0            # 인자 없으면 Usage 출력
docker run --rm [DOCKER_ID]/traffic-generator:1.0.0 google.com 0.5
docker push [DOCKER_ID]/traffic-generator:1.0.0
```
- ENTRYPOINT로 실행부를 고정하고 `docker run` 뒤 인자가 스크립트 뒤에 append되도록 설계 (어제 배운 CMD vs ENTRYPOINT 활용)

### Step 4. Deployment 배포 & 가변 IP 한계 실습
`color-api-deployment.yaml` (replicas: 5, `app: color-api`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: color-api-deployment
  labels:
    app: color-api
spec:
  replicas: 5
  selector:
    matchLabels:
      app: color-api
  template:
    metadata:
      labels:
        app: color-api
    spec:
      containers:
        - name: color-api
          image: [DOCKER_ID]/color-api:1.1.0
          ports:
            - containerPort: 80
```
- 트래픽 제너레이터 팟이 특정 팟 IP를 하드코딩(`http://<TARGET_POD_IP>/api`)한 상태로 배포
```bash
kubectl apply -f color-api-deployment.yaml
kubectl get pods / deployment / replicaset
kubectl describe pod color-api-deployment-xxxxx   # 타겟 팟 IP 확인·기입
kubectl apply -f traffic-generator.yaml
kubectl logs -f traffic-generator                 # 로그 스트리밍
# 다른 터미널에서 타겟 팟 강제 삭제
kubectl delete pod color-api-deployment-xxxxx
```
- **결과**: 팟 삭제 즉시 트래픽 로그 끊김. Deployment가 새 팟을 자동 생성했지만 **완전히 다른 IP**를 받아서, 옛 IP를 하드코딩한 제너레이터는 새 팟을 추적 불가 → 가변 IP의 한계 실증

### Step 5. ClusterIP 서비스로 해결
`color-api-clusterip.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: color-api-clusterip
  labels:
    app: color-api
spec:
  type: ClusterIP
  selector:
    app: color-api
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```
- 트래픽 제너레이터 타겟을 팟 IP → **서비스 이름**으로 변경: `http://color-api-clusterip/api`
```bash
kubectl apply -f color-api-clusterip.yaml
kubectl get svc
kubectl apply -f traffic-generator.yaml
kubectl logs -f traffic-generator
```
- **로드 밸런싱 확인**: 로그의 hostname이 계속 교차 변경 → 5개 백엔드 팟에 라운드 로빈 분산
- **무중단 확인**: 트래픽 중 백엔드 팟 하나를 삭제해도 로그 끊김 없음(정상 팟으로만 우회)
- **엔드포인트 자동 갱신**: `kubectl describe svc color-api-clusterip`의 Endpoints가 Deployment 재배포 시 신규 팟 IP들로 자동 갱신됨
- **핵심 결론**: ClusterIP 자체도 서비스를 지웠다 만들면 IP가 바뀌므로, 통신 명세엔 IP가 아니라 **서비스 이름(DNS)**을 써야 안전. ClusterIP는 클러스터 내부 전용(외부 접근 불가)

---

## 결과 / 산출물
- Color API / Traffic Generator 이미지 빌드·Hub 푸시 완료
- 전체 파이프라인(로컬 개발 → 이미지 → 레지스트리 → K8s 배포) 완주
- 팟 가변 IP 한계 실증 → ClusterIP + 서비스 이름 통신으로 해결, 로드 밸런싱·무중단 확인

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| "자가치유되는데도 통신이 끊기는" 역설을 실습으로 겪으며 서비스의 필요성을 체득함 |
| ENTRYPOINT 인자 append, sh 셔뱅 등 어제 배운 Docker 지식을 실습에 그대로 적용함 |

---

## 다음에 이어서 할 것
- LoadBalancer/Ingress 등 외부 노출 서비스 (클라우드 연동 강의에서 예정)

---
*작성일: 2026-07-10 | 인공지능 사관학교 AI 보안 과정*
