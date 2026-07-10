# 2026-07-10 강의 정리 — K8s ReplicaSet & Deployment

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: ReplicaSet(개념·자가치유·한계), Deployment(롤링 업데이트·pod-template-hash·롤백·스케일·장애 트러블슈팅)
> **관련 실습 파일**: `../practice/2026-07-10_AI-Security-Course_Practice_Color-API-Deployment.md`

---

## 핵심 인사이트 모음

### 📅 2026-07-10

- ReplicaSet의 유일한 목적이 "라벨 셀렉터와 일치하는 팟의 개수 유지"라는 게 핵심이었다. 그래서 팟이 죽으면 template으로 새로 찍어 개수를 채우는 자가치유가 되는데, 반대로 개수만 맞으면 명세(이미지 등)가 바뀌어도 기존 팟은 그대로 둔다. 내 메모의 "이미지가 바뀌어도 기존 팟엔 반영 안 되고 새로 만들어지는 팟에만 적용"되는 이유가 이거였다.
- 그래서 순수 ReplicaSet만으로는 무중단 배포(롤링 업데이트)가 불가능하다. 이미지를 바꿔도 기존 팟을 일일이 수동 삭제해야 새 template으로 교체되니까. 이 수동 작업의 공백을 메우려고 상위 추상화인 Deployment가 존재한다는 흐름이 자연스럽게 이어졌다.
- Deployment가 무중단 배포를 하는 비결이 **pod-template-hash**라는 게 인상적이었다. 팟 template 전체를 해시로 변환해 라벨에 박아두고, template이 한 글자라도 바뀌면 완전히 새 해시가 나온다. Deployment는 이 해시 변화를 감지해서 "새 해시의 ReplicaSet을 만들고 구 해시 ReplicaSet의 팟을 줄이는" 교차 제어로 롤링 업데이트를 완성한다. 결국 Deployment는 두 개의 ReplicaSet(구/신)을 교차 통제하는 지휘자였다.
- 라벨 셀렉터의 "많으면 죽이고, 이미 있으면 편입한다"는 단순 규칙이 흥미로웠다. 목표보다 팟이 많으면(4개인데 목표 3개) 초과분을 즉시 Terminate하고, 셀렉터와 일치하는 독립 팟이 미리 있으면 새로 안 만들고 그걸 입양한다. 지능적 판단이 아니라 "개수 + 라벨"이라는 명료한 규칙만으로 움직인다는 점에서, 라벨 네이밍을 고유하게 설계하지 않으면 의도치 않은 충돌·편입이 생길 수 있겠다는 보안·운영 관점의 경각심이 들었다.
- 롤링 업데이트가 장애를 방어하는 방식이 똑똑했다. 새 팟이 Healthy/Ready가 되는 걸 확인한 후에야 구 팟을 제거하기 때문에, 잘못된 이미지 태그(ImagePullBackOff)로 새 팟이 안 뜨면 배포를 그 자리에서 동결시키고 구 팟을 그대로 유지한다. 그래서 사용자는 구버전으로 계속 서비스를 받는다. "안전하게 실패한다(fail-safe)"는 설계가 여기서 구체적으로 보였다.

---

## 1. ReplicaSet — 개념과 자가 치유

### 핵심 개념
- 지정한 개수(replicas)만큼 동일 팟을 상시 유지·보장하는 오브젝트. 고가용성(HA)과 결함 허용의 기반

### 상세 내용
- **명세 핵심 필드** (`apiVersion: apps/v1`, `kind: ReplicaSet`):
  - `replicas`: 유지할 팟 목표 개수(Source of Truth)
  - `selector.matchLabels`: 관리 대상 팟을 찾는 식별 기준
  - `template`: 부족할 때 새 팟을 찍어낼 붕어빵 틀. template 하위 metadata·spec은 생성될 팟의 설정
- **★라벨 매칭 주의**: `template.metadata.labels`는 반드시 `selector.matchLabels`와 일치해야 함. 다르면 만든 팟을 자기 셀렉터로 못 알아봐서 무한 생성 루프에 빠짐
- **지속적 감시 루프**: 컨트롤러가 셀렉터 라벨을 가진 팟 개수를 실시간 카운트 → 목표보다 적으면(장애로 1개 삭제 등) template으로 새 팟 생성 → 새 팟은 같은 라벨을 달고 나오므로 셀렉터 범위에 자동 편입 → 목표 개수 복구
- **자가치유 검증**: 3개 중 1개를 `kubectl delete pod`로 지우면, 나머지 2개는 유지되고 새 무작위 해시 팟이 즉시 생성됨. `describe rs`의 Events에서 `SuccessfulCreate` 이력 확인
- 팟 이름 규칙: `[RS이름]-[랜덤문자열]` (팟 이름 중복 불가라 접미사 자동 생성)

---

## 2. ReplicaSet의 한계

### 핵심 개념
- ReplicaSet은 "개수 유지"만 할 뿐, 롤아웃 업데이트·롤백을 자체 지원하지 못함

### 상세 내용
- **템플릿 변경이 기존 팟에 반영 안 됨**: `spec.template`의 이미지를 바꿔 apply해도, 이미 목표 개수만큼 팟이 돌고 있으면 컨트롤러는 아무 액션도 안 함(`configured` 메시지만 뜨고 팟 변화 없음). RS 템플릿은 새 이미지인데 실제 팟은 구 이미지로 도는 불일치 발생
- **수동 롤아웃 강제**: 구버전 팟을 사람이 직접 삭제해야 RS가 새 template으로 다시 찍어냄 → 전체 최신화하려면 팟을 하나하나 다 지워야 하는 번거로움
- **라벨 셀렉터 상호작용 (단순 규칙)**:
  - 케이스1 — 목표(3) 달성 상태에서 같은 라벨 독립 팟을 추가하면, CURRENT가 4로 올랐다가 초과분을 즉시 Terminate (많으면 죽임)
  - 케이스2 — 같은 라벨 독립 팟이 먼저 있고 나중에 RS 배포하면, 부족분 2개만 새로 만들고 기존 팟을 입양(이미 있으면 안 만들고 편입)
  - 교훈: 라벨 네이밍을 고유하게 설계하지 않으면 다른 서비스·독립 팟과 충돌
- **결론**: 무중단 배포·자동 롤백·버전 관리 부재 → 상위 추상화인 Deployment 필요

---

## 3. Deployment — 개념과 롤링 업데이트

### 핵심 개념
- ReplicaSet을 감싸 통제하는 상위 추상화. 무중단 롤링 업데이트·자동 롤백·개정 이력 관리 제공

### 상세 내용
- **제공 기능**: 롤링 업데이트(다운타임 없이 점진 교체), 자동 롤백(버그 시 이전 버전 복구), 개정 이력(Revision, K8s 자체 레벨에서 배포 히스토리 추적), 안전 통제(배포 중 허용 가능한 불능 팟 수 제한)
- **명세 추가 필드** (`kind: Deployment`, `apps/v1`): `strategy.type: RollingUpdate`, `rollingUpdate.maxUnavailable`(동시 다운 허용 최대), `maxSurge`(목표 초과 일시 허용). strategy 생략 시 기본값 RollingUpdate (25% maxUnavailable, 25% maxSurge)
- **롤링 업데이트 동작**: template 변경(이미지 업그레이드 등) 감지 → 기존 RS(v1)는 두고 새 RS(v2)를 0개로 생성 → v2가 새 팟 1개 생성·Ready 확인 → v1이 구 팟 1개 제거 → 교차 반복 → v2가 목표치 채우고 v1이 비면 배포 완료
- **3대 오브젝트 관계**: Deployment(배포 전략·롤백·이력) → ReplicaSet(개수 보장) → Pod(실제 컨테이너 실행)

---

## 4. pod-template-hash 메커니즘

### 핵심 개념
- Deployment가 무중단 롤아웃을 구현하는 핵심 식별 장치

### 상세 내용
- Deployment 배포 시 하부에 `[Deployment이름]-[해시]` 형태의 RS가 자동 생성됨(사용자가 RS를 직접 안 만들어도)
- 팟의 `metadata.labels`에 `pod-template-hash` 라벨이 강제 주입됨. 이 해시는 상위 RS 이름의 해시와 100% 동일
- **동작 원리**: 팟 template(`spec.template`) 문자열 전체를 해시로 변환 → template을 한 글자라도 수정하면 완전히 새 해시 생성 → Deployment가 이 해시 변화를 감지해 "구 해시 RS 팟을 줄이고 새 해시 RS를 생성·교체"하는 롤아웃 수행
- 즉 이 라벨 시스템이 선언적 무중단 자동 롤아웃을 가능하게 하는 고유 식별 링크

---

## 5. Rollout — 롤백과 이력 관리

### 핵심 개념
- `kubectl rollout`으로 배포 상태 추적·롤백·일시정지 제어

### 상세 내용
- **서브커맨드**: `history`(버전 이력), `undo`(직전 배포 취소·롤백), `status`(진행 상태), `pause`/`resume`, `restart`
- **롤백(`undo`) 메커니즘**: 새 팟을 발주하지 않고 이전 세대 RS의 해시를 추적해 구버전 팟을 복구
- **Revision 번호 변화**: `undo` 시 과거 Revision이 삭제되는 게 아니라 최신 번호로 복사·이관됨 (예: Rev1→Rev3으로 이관, Rev1은 리스트에서 사라지고 Rev2·Rev3 남음)
- **CHANGE-CAUSE 기록**:
  - 선언형: YAML `metadata.annotations`에 `kubernetes.io/change-cause` 추가
  - 명령형(CI/CD 친화): `kubectl annotate deployment/... kubernetes.io/change-cause="..." --overwrite` — 파일 수정 없이 Git 커밋 메시지·빌드 번호 주입에 유용

---

## 6. Scale — 수량 조절과 선언형 충돌

### 핵심 개념
- `kubectl scale`로 즉시 수량 조절(명령형)하지만 선언형과 충돌 위험

### 상세 내용
- `kubectl scale deployment/... --replicas=3`: YAML 수정 없이 즉시 팟 수 조절
- **선언형과 충돌**: scale은 일시적. 로컬 YAML의 replicas(예: 5)는 그대로라, 다음 `apply` 시 라이브 수량(3)이 파일 기준(5)으로 강제 원복
  - 실무 리스크: 트래픽 폭증으로 임시로 10개 늘려놨는데 다른 개발자·CI/CD가 기존 YAML(5)로 apply하면 순식간에 반토막 → 대형 장애
  - 권장: 장기적 수량 변경은 반드시 YAML의 replicas를 수정 후 apply(선언형 고수)
- **예외 유용 케이스 — 팟 강제 재시작**: `--replicas=0`으로 전부 종료 후 원래 수량으로 원복 → 오동작 팟을 깨끗이 재기동. 단 template 스펙 변경이 아니므로 rollout history엔 새 Revision이 안 쌓임

---

## 7. 장애 배포 트러블슈팅 & fail-safe

### 핵심 개념
- 잘못된 이미지 태그 배포 시 롤링 업데이트가 안전하게 실패하도록 방어

### 상세 내용
- **장애 재현**: 존재하지 않는 태그(`nginx:127.1-alpine`, 도트 누락) 배포 → Desired 5인데 Up-to-date 3에 멈추는 불일치
- **디버깅 절차**:
  1. `describe deployment`: 원하는 수량·이미지명 정도만, 구체적 실패 원인은 안 나옴
  2. `get pods`: 일부 팟 STATUS가 `ImagePullBackOff`/`ErrImagePull`
  3. `describe pod [실패팟]`: Events에서 `manifest for ... not found` → 태그 오타·비공개 레포 인증 의심
- **fail-safe 메커니즘**: 롤링 업데이트는 새 팟이 Healthy/Ready가 되어야만 구 팟을 제거. 새 팟이 안 뜨면 배포를 그 자리에서 동결(Stuck)하고 구 RS 유지 → 사용자는 구버전으로 중단 없이 서비스 이용. `rollout status`가 `Waiting for ... 3 out of 5...`에서 멈춤
- **복구 2가지**:
  - 임시(`rollout undo`): 이전 정상 버전 해시로 즉시 복구. 단 로컬 YAML엔 버그가 남아 다음 CI/CD 때 재발
  - 영구(Roll-forward): YAML 태그를 바로잡아 `apply` + `annotate`로 사유 기록. 수정 후 template 해시가 과거 정상 버전과 일치하면 K8s가 이력을 최신 번호로 병합 이관
- 정리: Deployment 삭제 시 하위 RS와 모든 팟이 종속 삭제됨

## 
*작성일: 2026-07-10 | 인공지능 사관학교 AI 보안 과정*
