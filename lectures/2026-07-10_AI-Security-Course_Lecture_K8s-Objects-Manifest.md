# 2026-07-10 강의 정리 — K8s 오브젝트 관리 & 매니페스트

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Pod 관리(describe/logs/네트워크), 매니페스트 4대 필드, 오브젝트 관리 3방식, kubectl apply(3-way merge), dry-run YAML 생성, Service 개념/4타입
> **관련 실습 파일**: `../practice/2026-07-10_AI-Security-Course_Practice_K8s-Pod-Service.md`

---

## 핵심 인사이트 모음

### 📅 2026-07-10

- 서비스(Service)가 팟 앞단에 고정 엔드포인트를 제공한다는 게 오늘 K8s 이해의 핵심 열쇠였다. 팟은 일시적(Ephemeral)이라 죽고 새로 뜨면 IP가 완전히 바뀌는데, 서비스는 그 위에 안정적인 고정 IP(ClusterIP)와 DNS 이름을 얹어준다. 어제 배운 "팟은 이름으로 통신이 안 된다(기본 브리지처럼 DNS 안 됨)"의 해답이 바로 이 서비스였다.
- 내 메모에서 "`kubectl expose pod nginx`가 nginx라는 '이름'의 pod에 묶이는 건가?"라고 적었는데, 정확히는 **이름이 아니라 라벨(label)로 묶인다.** expose 순간 K8s가 대상 팟의 라벨을 조회해서 서비스의 `selector`에 자동으로 채워 넣고, 그 이후부터는 이름과 무관하게 `app: nginx` 라벨을 가진 팟이면 뭐든 트래픽을 받는다. 그래서 팟을 지웠다 같은 라벨로 새로 띄우면 서비스가 알아서 다시 연결해준다. "이름 바인딩"이 아니라 "라벨 셀렉터 바인딩"이라는 게 핵심.
- 이미 돌아가는 팟/서비스로 매니페스트 파일을 만드는 게 꽤 편해 보인다 `kubectl ... --dry-run=client -o yaml`. 실제 API 서버로 요청을 보내지 않고 로컬 kubectl 단에서 YAML 뼈대만 뽑아주는 것. 명령형으로 빠르게 만들던 걸 선언형 파일로 마이그레이션하는 디딤돌로 쓸 수 있어서, "명령형→선언형" 전환이 생각보다 부드럽게 이어진다는 걸 느꼈다.
- 오브젝트 관리가 명령형 CLI → 명령형 파일 → 선언형 파일 순으로 갈수록 학습 곡선은 가팔라지지만 실무 표준에 가까워진다는 구조가 인상적이었다. 특히 `kubectl replace`(명령형)가 시스템 자동 주입 필드(nodeName, serviceAccount 등) 때문에 이미지 하나 바꾸려다 요청 전체가 거부당하는 걸 보고, 왜 `apply`의 3-way merge가 필요한지 납득됐다. 로컬 파일엔 없는 라이브 시스템 필드를 건드리지 않고 내가 바꾼 부분만 골라 패치하는 게 선언형의 핵심 가치.
- 매니페스트가 apiVersion/kind/metadata/spec 4대 필드로 구성되고, spec의 하위 구조는 kind에 따라 완전히 달라진다는 게 깔끔했다. 다 외울 필요 없이 dry-run이나 IDE 스키마 검증으로 뼈대를 잡으면 된다는 접근이 실용적.

---

## 1. Pod 관리 (describe / logs / 네트워크)

### 핵심 개념
- `get`은 요약 리스트, `describe`는 세부 명세·이벤트, `logs`는 컨테이너 표준출력

### 상세 내용
- **`kubectl describe pod [name]`**: Namespace, Node(배치된 워커노드), IP(사설), Status, Image, Conditions, 그리고 최하단 **Events**(이미지 다운로드·컨테이너 생성 실패 등 트러블슈팅의 핵심)
- **팟 네트워크 격리**: 팟 IP는 클러스터 내부 전용 사설 IP → 호스트 PC(클러스터 외부)에서 `curl [팟IP]` 하면 실패(hang). 클러스터 내부에 Alpine 팟을 띄워서 같은 IP로 요청하면 정상 응답
- **이름 기반 통신 불가**: 클러스터 내부에서도 `curl nginx`(팟 이름)는 `cannot resolve host` → K8s 기본 상태에선 팟 이름 DNS 해석 불가. 이름 통신을 하려면 별도의 **Service** 객체가 필요 (→ 6번)
- **`kubectl logs [name]`**: 컨테이너 stdout/stderr. 다중 컨테이너 팟은 `-c [container]`로 지정 필수
- **`kubectl delete pod [name]`**: 특정 팟 삭제. `--force`는 유예기간 없이 즉시 삭제(실습 환경에서만, 운영에선 데이터 유실 위험으로 지양)

---

## 2. 매니페스트(Manifest) 4대 탑레벨 필드

### 핵심 개념
- 매니페스트 = 오브젝트의 원하는 상태를 정의한 YAML 파일. 대부분 apiVersion/kind/metadata/spec 4개 필드 필수

### 상세 내용
- **apiVersion**: API 그룹과 버전. `v1`(코어 그룹), `apps/v1`(확장 그룹, Deployment·ReplicaSet 등)
- **kind**: 리소스 종류(Pod, Service, Deployment 등). apiVersion이 지원하는 종류여야 함
- **metadata**: 식별 정보. `name`, `namespace`, `labels`(식별·검색용 키-값), `annotations`(비식별·시스템 지시용). 생성 시 K8s가 시스템 관리 데이터를 자동 주입
- **spec**: 실제 세부 명세. 하위 구조는 kind에 따라 완전히 달라짐 (Pod는 `containers` 필수)
- 실무 팁: spec 하위를 다 외울 필요 없음 → dry-run이나 IDE의 K8s 확장(스키마 검증·자동완성) 활용

---

## 3. 오브젝트 관리 3방식

### 핵심 개념
- 명령형 CLI → 명령형 설정파일 → 선언형 설정파일. 뒤로 갈수록 학습곡선↑, 실무 표준에 가까움. 상태 조회(`get`/`describe`/`logs`)는 어느 방식이든 공통

### 상세 내용
- **① 명령형 CLI (Imperative CLI)**: `kubectl run/expose/scale/create/delete`. 파일 없이 한 줄로 처리. 빠르고 직관적이지만 템플릿 저장·코드리뷰·감사추적(Audit) 불가, 이력이 안 남음
- **② 명령형 설정파일 (Imperative Config)**: `kubectl create -f`, `replace -f`, `delete -f`. YAML 자산을 Git 관리·리뷰·감사 가능, 재생성 쉬움. 단 리소스를 통째로 교체하는 방식이라 스키마 완전 숙지 필요, `replace` 시 라이브 데이터(외부 주입 annotation 등) 유실 위험
- **③ 선언형 설정파일 (Declarative Config) ★실무 표준**: `kubectl apply -f`, `diff -f`. 원하는 최종 상태만 선언하면 시스템이 알아서 동기화. 3-way merge로 라이브 상태 보존, 디렉터리 통째 적용(`apply -f ./`) 가능. 학습곡선 높고 부분 패치 실패 시 디버깅 복잡. **삭제는 선언형 세계에서도 명령형 `delete -f`를 공식 권장**

---

## 4. kubectl apply와 3-way merge

### 핵심 개념
- 선언형이 부분 패치를 똑똑하게 하는 비결은 `last-applied-configuration` 애노테이션과 3자 비교

### 상세 내용
- **명령형 replace의 한계**: 로컬 YAML엔 시스템 자동 주입 필드(volumeMounts, serviceAccountName, nodeName 등)가 없음. `kubectl replace`는 "라이브를 로컬 파일로 완전 덮어쓰기"라, API 서버가 이 시스템 필드들을 null로 지우려는 걸로 오해 → `Forbidden` 에러(구동 중 팟의 nodeName·serviceAccount 변경은 금지)
  - 명령형 우회책: 팟을 delete 후 create(재생성) → 하지만 매번 다운타임 발생, 운영에선 불가
- **선언형 apply의 3-way merge**: 세 가지를 비교해 시스템 필드는 건드리지 않고 의도한 필드만 패치
  1. `last-applied-configuration` (과거 내가 제출한 상태)
  2. 로컬 YAML 현재 상태 (지금 요청하는 상태)
  3. etcd의 라이브 설정 (시스템 필드가 추가된 실제 상태)
- **팟은 파괴되지 않는다**: apply로 이미지를 바꿔도 팟 오브젝트 자체(생성시간·UID)는 유지되고, 내부의 구버전 컨테이너만 Terminated 후 새 컨테이너로 교체 (`Last State: Terminated`로 검증)
- **명령형→선언형 마이그레이션**: `create`로 만든 리소스도 삭제 없이 첫 `apply` 한 번으로 전환 가능(누락된 `last-applied-configuration`를 자동 패치, 경고만 뜨고 에러 아님). 미리 `create -f ... --save-config` 하면 경고도 방지

---

## 5. dry-run으로 매니페스트 자동 생성

### 핵심 개념
- 명령형 명령어로 정확한 YAML 뼈대를 순식간에 뽑는 기능. 명령형→선언형 전환의 디딤돌

### 상세 내용
- **`--dry-run=client`**: 실제 API 서버로 보내지 않고 로컬 kubectl 단에서만 가상 검증
- **`-o yaml`**: 결과 명세를 YAML로 출력
- 예: `kubectl run color-api --image=... --dry-run=client -o yaml`, `kubectl expose pod nginx-pod --type=NodePort --port=80 --dry-run=client -o yaml`
- **핵심 자동화**: expose 시 대상 팟의 라벨을 조회해 서비스의 `selector`에 자동으로 채워줌 → 매칭 기준을 수동 계산할 필요 없음
- `> pod.yaml`로 저장 후 `kubectl create/apply -f`로 재사용. 단 운영 반영 전 누락 필드는 문서 교차검증 권장

---

## 6. Service — 개념과 4가지 타입

### 핵심 개념
- 팟의 네트워크 정보를 추상화해 안정적 엔드포인트(고정 IP + 내부 DNS 이름)를 제공. `selector` 라벨로 대상 팟을 동적 추적

### 상세 내용
- **왜 필요한가**: 팟은 Ephemeral → 재기동 시 IP가 바뀜. 프론트가 백엔드 팟 IP를 하드코딩하면 재기동 순간 통신 단절. 서비스가 상위 추상화 레이어로 고정 엔드포인트를 제공하고, 정상 팟으로 트래픽 라우팅
- **서비스 명세 핵심**: `metadata.name`(내부 DNS 이름), `spec.selector`(대상 팟 라벨), `spec.ports`(port=서비스 리스닝 / targetPort=팟 컨테이너 포트 / nodePort=노드 외부 노출 포트)

**4가지 타입**

| 타입 | 특징 | 접근 범위 | 용도 |
|---|---|---|---|
| ClusterIP (기본) | 내부 전용 고정 가상 IP | 클러스터 내부만 | 내부 마이크로서비스 간 통신(백엔드↔DB) |
| NodePort | 모든 워커노드 IP에 정적 포트(30000~32767) 개방 | 외부에서 `노드IP:NodePort` | 개발·테스트용 빠른 외부 연동(운영엔 거의 안 씀) |
| LoadBalancer | 클라우드 공급자 LB와 연동, 단일 공인 IP | 외부 | 부하분산·고가용성 필요한 운영 환경 |
| ExternalName | 내부 팟이 외부 시스템을 고정 DNS로 찾도록 CNAME 매핑 | (역방향) | 외부 API·DB URL 하드코딩 없이 접근 |

- 클라우드 연동 전까지는 실습 용이성을 위해 ClusterIP/NodePort/ExternalName 위주로 다룰 예정

---

## 다음에 파고들 만한 질문
> 강의가 다시 다루지 않을 곁가지 호기심. 진짜 팔 의향이 있는 것만.

- 여러 대의 컴퓨터(AWS EC2 인스턴스 여러 개)에서 K8s를 사용하는 구조는 실제로 어떻게 설계되는가? (마스터/워커 노드를 EC2에 어떻게 배치하고 연결하는지, 직접 구조를 그려봐야 전체가 이해될 것 같음 — 주말에 설계해볼 예정)

---

## 찾아본 것

*(오늘 강의 범위에서 별도로 찾아본 곁가지 없음)*

---
*작성일: 2026-07-10 | 인공지능 사관학교 AI 보안 과정*
