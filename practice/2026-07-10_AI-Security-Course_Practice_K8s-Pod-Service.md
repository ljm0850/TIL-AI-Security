# 2026-07-10 실습 — K8s Pod & Service 관리

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-10_AI-Security-Course_Lecture_K8s-Objects-Manifest.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-10

- 팟 IP가 클러스터 내부 전용이라는 걸 실습으로 확실히 체감했다. 호스트 PC에서 팟 IP로 curl 하면 먹통이 되는데, 클러스터 안에 Alpine 팟을 하나 띄워서 같은 IP로 찌르니 바로 nginx 응답이 왔다. "격리"라는 말이 실제로 어떤 경계인지 눈으로 본 셈.
- 서비스를 만들기 전엔 팟 이름으로 통신이 안 되다가(`curl nginx` 실패), 서비스를 붙이니 이름 기반 통신이 되는 흐름을 직접 확인했다. 어제 Docker 사용자 정의 네트워크에서 이름 통신이 되던 것과 같은 결인데, K8s에선 그 역할을 서비스가 한다는 게 연결됐다.
- 서비스의 진짜 가치는 팟을 죽였다 살렸을 때 드러났다. 서비스 IP는 그대로인데 대상 팟을 지우면 통신이 끊기고(`Connection refused`), 같은 라벨로 팟을 새로 띄우면 서비스가 알아서 다시 연결해준다. 서비스가 "이름"이 아니라 "라벨"로 팟을 찾는다는 걸 이 실습으로 납득함.
- 이미 돌아가는 팟/서비스에서 `--dry-run=client -o yaml`로 매니페스트 뼈대를 뽑아내는 게 특히 편했다. expose 할 때 대상 팟 라벨을 알아서 selector에 채워주는 걸 보고, 명령형으로 만들던 걸 선언형 파일로 옮기는 게 생각보다 부담이 없다는 걸 느낌.

---

## 실습 개요
- **목표**: kubectl로 팟을 생성·조회·통신·삭제하고, 서비스(NodePort)를 붙여 고정 엔드포인트와 라벨 셀렉터 동작을 검증한다. 매니페스트 파일 기반 생성과 dry-run 자동 생성도 실습한다.
- **방법**: 팟 describe/logs/네트워크 격리 실습 → expose로 서비스 생성 → 팟 삭제·재생성으로 서비스 안정성 검증 → YAML 매니페스트로 Pod/Service 생성 → dry-run으로 YAML 뼈대 추출

---

## 진행 과정

### Step 1. 팟 상세 조회 및 네트워크 격리 (describe)
```bash
kubectl describe pod nginx
```
- Namespace/Node/IP/Status/Image/Conditions/Events 확인
- **격리 검증**:
```bash
# 호스트 PC(클러스터 외부)에서 → 실패(hang)
curl [nginx_Pod_IP]

# 클러스터 내부 Alpine 팟 띄워서 → 성공
kubectl run alpine -it --image=alpine:3.20 -- sh
apk update && apk add curl
curl [nginx_Pod_IP]     # nginx index.html 정상 출력
curl nginx              # cannot resolve host (이름 통신 불가 → Service 필요)
```

### Step 2. 로그 확인 및 팟 삭제 (logs / delete)
```bash
kubectl logs nginx              # 단일 컨테이너
kubectl logs nginx -c nginx     # 다중 컨테이너면 -c로 지정
kubectl delete pod alpine       # 특정 팟 삭제
# (nginx 팟은 다음 Service 실습을 위해 남겨둠)
```

### Step 3. 서비스 노출 (expose) 및 안정성 검증
```bash
kubectl expose pod nginx --type=NodePort --port=80
kubectl get service    # nginx 서비스와 고정 Cluster-IP 확인
```
- **팟 삭제→재생성으로 서비스 안정성 검증**:
```bash
# Alpine 내부에서 서비스 Cluster-IP로 요청 → 성공
curl [Service_Cluster-IP]

# nginx 팟 삭제 후 → 통신 실패
kubectl delete pod nginx

# 같은 이름·이미지로 재생성 → 팟 IP는 바뀌지만(10.244.0.3→10.244.0.6)
kubectl run nginx --image=nginx:1.27.0
# 서비스 Cluster-IP로 다시 요청 → 즉시 정상 응답 (서비스가 백엔드 매핑 자동 갱신)
```
- 이후 이름(DNS) 기반 통신도 성공: Alpine에서 `curl nginx`(서비스 이름)로 응답 수신

### Step 4. 매니페스트 파일 기반 Pod 생성 (명령형 파일)
`object-management/nginx-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:1.30.0
    ports:
    - containerPort: 80
```
```bash
kubectl create -f nginx-pod.yaml
kubectl get pods
kubectl describe pod nginx-pod    # 라벨·이미지·포트가 파일대로 반영됐는지 교차 검증
```
- IDE(VS Code) K8s 확장: spec 선언 시 스키마 검증·자동완성. resources.limits/requests 미설정 시 경고(자원 독점 방지용, 후반부 학습)

### Step 5. 매니페스트 파일 기반 Service 생성 및 라벨 셀렉터 검증
`object-management/nginx-svc.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx     # 이 라벨을 가진 팟으로 트래픽 라우팅
```
```bash
kubectl create -f nginx-svc.yaml
kubectl get services
```
- **셀렉터 메커니즘 검증** (팟 소멸→재가동):
  - 서비스 켜진 상태에서 `kubectl delete -f nginx-pod.yaml` → Alpine에서 서비스 IP 호출 시 실패(백엔드 없음)
  - `kubectl create -f nginx-pod.yaml`로 재생성 → 서비스 IP·설정 안 바뀌었는데도 새 팟의 `app: nginx` 라벨을 자동 감지해 다시 정상 응답
- `kubectl run` vs `kubectl create`: run은 단일 Pod 임시 생성(테스트/디버깅), create는 다양한 오브젝트를 YAML 기반으로 생성(운영/인프라 구성)

### Step 6. dry-run으로 매니페스트 자동 생성
```bash
kubectl run color-api --image=iceman4u/color-api:1.0.0 --dry-run=client -o yaml
kubectl expose pod nginx-pod --type=NodePort --port=80 --dry-run=client -o yaml
```
- expose dry-run 시 대상 팟 라벨을 조회해 서비스 `selector`에 자동 기입 → 매칭 기준 수동 계산 불필요
- `> pod.yaml`로 저장 후 `create/apply -f`로 재사용 가능

---

## 결과 / 산출물
- 팟 네트워크 격리(내부 전용 IP)와 이름 통신 불가를 직접 검증
- NodePort 서비스로 팟 삭제·재생성 시에도 안정적 엔드포인트 유지 확인(라벨 셀렉터 동작)
- 매니페스트 파일로 Pod/Service 생성, dry-run으로 YAML 뼈대 추출 완료

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 서비스가 이름이 아니라 라벨로 팟을 찾는다는 걸 삭제·재생성 실습으로 확실히 검증함 | (특이사항 없음) |
| dry-run으로 명령형→선언형 마이그레이션 흐름을 체득함 | |

---

## 다음에 이어서 할 것
- ReplicaSet·Deployment 실습으로 확장 (별도 실습 파일 참고)

---
*작성일: 2026-07-10 | 인공지능 사관학교 AI 보안 과정*
