# 2026-07-09 실습 — Docker 스토리지 & Kubernetes Pod 생성

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-09_AI-Security-Course_Lecture_Docker-Volume-Compose.md`, `../lectures/2026-07-09_AI-Security-Course_Lecture_Kubernetes-Intro.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-09

- PostgreSQL 컨테이너로 "데이터가 사라진다/유지된다"를 직접 재현. stop→start는 데이터가 남고, rm 후 새 컨테이너를 띄우면 이전에 만든 DB/테이블/데이터가 전부 사라진다. 그 다음 볼륨을 붙여서 같은 시나리오를 돌리니 이번엔 컨테이너를 지우고 새로 만들어도(`db-postgres` → `db-postgres2`) 볼륨에 붙은 유저(`\du`)가 그대로 남아있었다. 
- Bind Mount는 컨테이너 안에서 만든 파일이 호스트 폴더에 즉시 나타나는 걸 두 개의 터미널로 동시에 확인하니 "공유 폴더" 감각이 바로 왔다. 컨테이너 `/work`에 디렉터리를 만들면 호스트 `~/dockers/mydata`에 그대로 생기는 구조.
- minikube로 로컬에 단일 노드 클러스터를 띄우고 `kubectl run`으로 첫 nginx 팟을 생성해봤다. Docker에서 `docker run`으로 컨테이너를 띄우던 것과 명령 형태가 닮아서 진입 장벽이 낮게 느껴졌지만, 이게 실제로는 API 서버로 요청이 번역되어 들어가고 스케줄러가 노드에 배치하는 과정을 거친다는 걸 강의로 배운 뒤라 겉보기보다 훨씬 많은 일이 뒤에서 돌아간다는 걸 인식하며 실행함.

---

## 실습 개요
- **목표**: (1) Docker 볼륨/바인드 마운트로 데이터 영속성을 직접 검증하고, (2) minikube+kubectl 로컬 클러스터를 구축해 첫 번째 팟을 생성한다.
- **방법**: PostgreSQL 컨테이너로 데이터 유실/보존 시나리오 재현 → Named Volume 연동 → Bind Mount 실습 → kubectl/minikube 설치 → 컨텍스트 확인 후 nginx 팟 생성·검사

---

## 진행 과정

### Step 1. 데이터 유실/보존 시나리오 (볼륨 없이)
```bash
docker run --name webserver -d nginx:1.30.0
docker exec -it webserver sh
cd /tmp && echo "Hello World" > hello.txt
```
- **stop & start**: 중지 후 재시작해도 `/tmp/hello.txt` 유지됨
- **stop & rm 후 새 컨테이너**: 삭제 후 같은 이미지로 새로 만들면 `cat /tmp/hello.txt` → `No such file or directory` (파일시스템 레이어가 컨테이너와 함께 삭제됨)

### Step 2. PostgreSQL로 데이터 영속성 확인 (볼륨 없이)
```bash
docker run --name db-postgres -e POSTGRES_PASSWORD=qwer1234 -d postgres
docker exec -it db-postgres /bin/bash
psql -U postgres
```
```sql
CREATE USER user01 PASSWORD '1234' SUPERUSER;
CREATE DATABASE test01 OWNER user01;
\c test01 user01
CREATE TABLE table01(id INTEGER PRIMARY KEY, name VARCHAR(20));
INSERT INTO table01 (id, name) VALUES (1, 'IceMan4U');
```
- stop→start 후에는 `table01` 데이터 유지됨
- 하지만 rm 후 새 컨테이너를 만들면 `test01`/`table01`이 전부 사라짐 → 볼륨 필요성 확인

> 📝 **곁가지 메모 — Oracle vs 기타 RDBMS(MySQL/PostgreSQL 등)**: PostgreSQL 실습 중 정리한 내용. Oracle은 DB를 "우주"에 비유할 만큼 계정이 DB에 종속되어 있고(계정이 DB 안에 속함), 테이블·뷰 같은 객체가 계정(유저)에 생성됨. 반면 MySQL/PostgreSQL 등은 계정이 상대적으로 독립적이고 DB는 하나의 "저장소" 개념에 가깝다. (오늘 Docker/K8s 주제와는 별개인 곁가지 기록)

### Step 3. Named Volume 생성 및 연동
```bash
docker volume ls
docker volume create myvolume01

docker run -e POSTGRES_PASSWORD=qwer1234 \
  --mount type=volume,source=myvolume01,target=/var/lib/postgresql \
  -d --name db-postgres postgres

docker exec -it db-postgres /bin/bash
psql -U postgres
```
```sql
CREATE USER user01 PASSWORD '1234' SUPERUSER;
\du
```
- 컨테이너 삭제 후 **같은 볼륨**을 새 컨테이너에 연결:
```bash
docker stop db-postgres && docker rm db-postgres
docker run -e POSTGRES_PASSWORD=qwer1234 \
  --volume myvolume01:/var/lib/postgresql \
  -d --name db-postgres2 postgres
docker exec -it db-postgres2 /bin/bash
psql -U postgres
\du   # user01이 그대로 남아있음 → 볼륨 데이터 영속성 확인
```
- `docker volume inspect myvolume01`로 `Mountpoint` 확인

### Step 4. Bind Mount 실습
```bash
mkdir -p ~/dockers/mydata && cd ~/dockers/mydata
ls / > ls.txt

docker run -e POSTGRES_PASSWORD=qwer1234 \
  --mount type=bind,source=/home/zynzza/dockers/mydata,target=/work \
  -d --name db-postgres postgres
```
- 컨테이너 내부에서 `/work/ls.txt` 확인 가능
- 컨테이너에서 변경 → 호스트에 즉시 반영:
```bash
# 컨테이너 내부
cd /work && mkdir test_dir
echo "Hello from Container" > ./test_dir/greeting
# 다른 터미널(호스트)
cd ~/dockers/mydata/test_dir && cat greeting   # "Hello from Container"
```

### Step 5. K8s 실습 환경 구축 (minikube + kubectl)
- kubectl 설치 (공식 문서 참고)
- minikube 설치: 로컬에서 단일 노드 클러스터(마스터+워커 통합)를 띄우는 도구. Docker 드라이버 권장
```bash
minikube start --driver=docker
minikube status
```
- 최소 사양: CPU 2코어, RAM 2GB 이상 (권장 4GB+). minikube는 클러스터만 띄우므로 kubectl 별도 필수

### Step 6. 컨텍스트 확인 후 첫 nginx 팟 생성
```bash
kubectl version
kubectl config current-context     # minikube인지 확인 (엉뚱한 원격 배포 방지)
# 필요 시: kubectl config use-context minikube

kubectl run nginx --image=nginx:1.27.0    # latest 대신 명확한 버전 태그
kubectl get pods
```
- 출력 지표 해석: NAME(nginx), READY(1/1), STATUS(Running이면 성공), RESTARTS(0이 정상), AGE

---

## 결과 / 산출물
- 볼륨 없이 → 컨테이너 삭제 시 DB 데이터 유실, 볼륨 연동 → 컨테이너 교체 후에도 데이터 보존을 직접 검증
- Bind Mount로 컨테이너↔호스트 실시간 파일 공유 확인
- minikube 로컬 클러스터 구축 및 첫 nginx 팟 생성·상태 확인 완료

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 볼륨 유무에 따른 데이터 영속성 차이를 직접 재현해 개념을 확실히 검증함 | (특이사항 없음) |
| 팟 생성 전 `current-context` 확인 습관을 실습으로 익힘 | |

---
*작성일: 2026-07-09 | 인공지능 사관학교 AI 보안 과정*
