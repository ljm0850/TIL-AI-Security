# 2026-07-02 심화 정리 — AWS 로드밸런서 & 쿠버네티스 트래픽 분산

> **과정**: 인공지능 사관학교 AI 보안 과정
> **출처 강의 파일**: `../lectures/2026-07-02_AI-Security-Course_Lecture_AWS-Intro.md`
> **작성 계기**: AWS VPC/EC2 강의 중 "MSA에서 컨테이너 간 네트워크를 어떻게 구성하는가" 의문에서 출발. Claude와의 대화로 정리.

---

## 왜 찾아봤는가
AWS 강의에서 VPC와 네트워킹을 배우면서, 과거 프로젝트의 MSA 구성이 실제로 어떻게 연결되는지 구조적으로 이해하고 싶었다. 특히 "ALB가 EC2 내부 프로세스를 어떻게 조절하는가"와 "쿠버네티스를 쓰면 Nginx가 사라지는가"가 불명확했다.

---

## 핵심 내용

### ALB의 역할 — 내부 프로세스 조절이 아니라 네트워크 레벨 분산

ALB는 EC2 인스턴스 내부를 건드리지 않는다. "헬스체크를 통과한 인스턴스에게 요청을 어떻게 나눌지"만 결정하는 네트워크 레벨 라우터다.

```
사용자 요청 → ALB (헬스체크 통과 인스턴스만 대상으로 분산)
           → EC2 (Ubuntu)
              └ Docker: Nginx (리버스 프록시) → BE 컨테이너
```

- 트래픽에 따라 EC2를 자동으로 늘리는 건 ALB가 아니라 **Auto Scaling Group(ASG)**
- ALB는 늘어난 인스턴스에 요청을 분산만 함

### 서버 1대에서 ALB가 의미 있는 경우
분산 대상이 1대뿐이면 ALB의 핵심 가치(부하 분산+이중화)는 발휘되지 않는다. 단, 예외:
- SSL/TLS 인증서 관리(ACM + 자동 갱신)
- 배포 중 헬스체크 기반 순간 차단
- 미래 확장 대비 구조 확보

### 쿠버네티스에서 Nginx의 위치 변화

K8s를 쓴다고 Nginx가 사라지는 게 아니라 **역할과 위치가 바뀐다**.

| 구분 | 흐름 |
|---|---|
| EC2 + Docker (K8s 없음) | `ALB → EC2 → Nginx → 컨테이너들` |
| 쿠버네티스 (EKS) | `ALB → Ingress Controller(nginx 기반) → Service(kube-proxy) → Pod들` |

- 클러스터 내부 Pod 간 분산은 **Service(ClusterIP) + kube-proxy**가 담당
- 외부 진입점은 **Ingress Controller** — 가장 널리 쓰이는 구현체가 nginx-ingress-controller
- 경로 기반 라우팅, rate limiting, 카나리 배포 등 세밀한 제어는 여전히 Ingress/Nginx 레벨

### 무중단 배포(Rolling Update) 원리

단위는 "서버"가 아니라 "파드(컨테이너)". EC2 1대여도 파드를 여러 개 띄우면 순차 교체 가능.

```
Deployment 업데이트 (v1 → v2)
  → 새 파드(v2) 생성
  → Readiness Probe 통과할 때까지 Service에서 제외
  → 통과하면 엔드포인트 추가 (트래픽 수신 시작)
  → 기존 파드(v1) 정상 종료 (그레이스풀 셧다운)
  → 반복
```

| 구분 | 순수 Docker | 쿠버네티스 |
|---|---|---|
| 헬스체크 | 직접 스크립트 또는 Nginx `max_fails` | Readiness Probe 자동 |
| 트래픽 전환 | 사람이 Nginx 설정 변경 | Deployment 컨트롤러 자동 |
| 자동화 | CI/CD 파이프라인 직접 스크립팅 | `kubectl apply`만으로 전체 사이클 |

---

## 이해한 것 / 아직 모르는 것

| 이해한 것 | 아직 모르는 것 |
|---|---|
| ALB는 네트워크 레벨 분산만, 내부 프로세스는 Nginx가 처리 | ALB Ingress Controller와 nginx-ingress-controller 중 언제 뭘 선택하는지 |
| K8s에서 Nginx는 사라지는 게 아니라 Ingress Controller로 위치 이동 |  |
| 무중단 배포의 단위는 서버가 아니라 파드 | |

---

## 출처
- Claude와의 대화 (2026-07-02)

---
*작성일: 2026-07-02 | 인공지능 사관학교 AI 보안 과정*
