# 2026-07-07 실습 — Docker & 컨테이너 기초

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-07_AI-Security-Course_Lecture_Docker-Basics.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-07

- 컨테이너 라이프사이클을 CLI로 직접 만져보니, (`run`/`start`/`stop`/`kill`/`rm`)개념으로 배울 때보다 훨씬 명확하게 들어왔다. `docker ps`(실행 중)와 `docker ps -a`(전체, 정지 포함)를 계속 번갈아 찍어보면서 "컨테이너는 삭제 전까지 정지 상태로도 남아있다"는 걸 체감.
- NGINX 커스터마이징 하면서 컨테이너 내부에 직접 들어가서(`docker exec -it ... sh`) `vim`으로 파일을 고쳐본 건 재밌었는데, 이 방식의 한계도 바로 느껴졌다. 명령을 하나하나 다 손으로 쳐야 하고, 컨테이너가 죽으면 안에서 고친 내용도 같이 날아간다는 점. 그래서 왜 Dockerfile로 이 과정을 코드화해야 하는지가 실습을 통해 자연스럽게 납득이 됨.
- 태그(`nginx:1.30.0`, `v0.0.1` 등)를 이용한 이미지 버전 관리를 직접 해보니, Dockerfile을 캐시된 레이어로 재빌드하는 과정도 눈에 들어옴 — 변경 없는 레이어는 재사용하고 바뀐 부분부터 다시 빌드되는 걸 build 로그에서 확인.

---

## 실습 개요
- **목표**: Docker CLI로 컨테이너 라이프사이클 전체(pull → run → stop → rm)를 다뤄보고, Dockerfile을 작성해 NGINX 이미지를 커스터마이징한 뒤 DockerHub에 push까지 해본다.
- **방법**: `docker run`/`ps`/`stop`/`rm`/`images` 등 기본 명령 실습 → NGINX 컨테이너에 직접 접속해 index.html 수정 → Dockerfile로 동일 작업을 코드화 → 로컬 파일 COPY → DockerHub 계정 생성 및 이미지 push

---

## 진행 과정

### Step 1. 첫 컨테이너 실습 (Running Containers)
- `docker pull nginx`, `docker run -d nginx`로 컨테이너 실행
- `docker ps` / `docker ps -a`로 실행 중·전체 컨테이너 상태 확인
- full container ID, short ID, 컨테이너 이름 세 가지 방식으로 각각 `docker kill` 실행해봄
- `docker run -d -p 8080:80 --name web_server nginx`로 포트 매핑 + 이름 지정 후 `curl http://localhost:8080`으로 응답 확인
- `docker stop` → `docker rm` → `docker container prune`(정지된 컨테이너 일괄 삭제) → `docker rmi`(이미지 삭제) 순서로 정리

### Step 2. Container Lifecycle 실습
- `docker start <cId/cName>`으로 정지된 컨테이너 재시작
- 필터링: `docker ps --filter name=...`, `docker ps --filter id=...`
- 일괄 처리: `docker stop $(docker ps -q)`(실행 중인 것 전체 정지), `docker rm $(docker ps -aq)`(전체 삭제)

### Step 3. 컨테이너 내부 접근 및 로그 확인
- `docker logs -f web_server`로 실시간 로그 스트리밍하면서 다른 창에서 `curl` 반복 실행해 로그 찍히는 것 확인
- `docker exec -it web_server /bin/bash`로 컨테이너 내부 셸 접속, `index.html` 직접 확인

### Step 4. 최초 Dockerfile 작성
```dockerfile
FROM ubuntu:latest
CMD ["echo", "Hello from my first Docker image!"]
```
- `docker build .`로 Dockerfile 없이도 빌드 가능함을 확인 (현재 디렉터리의 Dockerfile을 기본으로 인식)
- 빌드된 이미지로 `docker run` / `docker start -a`(연결된 상태로 시작, 출력 바로 확인) 실습

### Step 5. NGINX 커스터마이징 프로젝트
- `docker pull nginx:1.30.0`으로 특정 태그 지정 pull
- `docker run -d -p 80:80 --name web_server nginx:1.30.0` 후 `curl http://localhost`로 기본 응답 확인
- `docker exec -it web_server sh`로 접속 → `apt update` / `apt install vim` → `vim /usr/share/nginx/html/index.html` 직접 수정
- 수정 후 `curl http://localhost`로 변경된 응답 확인
- **프로젝트 한계 체감**: 명령을 다 손으로 입력해야 해서 확장성이 낮고, 컨테이너가 사라지면 수정 내용도 같이 사라짐(컨테이너 저장소는 임시), 명령 자체가 어디에도 기록되지 않음

### Step 6. Dockerfile로 코드화 (dockerfile-nginx)
```dockerfile
FROM nginx:1.30.0

RUN apt update
RUN apt install vim
```
- `docker build -t web_server .`로 빌드 후 컨테이너 실행, `docker exec -it <cId> sh`로 vim 설치 확인
- 이미지 이름을 바꿔 재빌드(`web_server_image`)하며 캐시된 레이어가 재사용되는 것을 build 로그에서 확인

### Step 7. 로컬 파일 COPY 실습
- 같은 디렉터리에 `index.html` 생성 후 Dockerfile에 추가:
```dockerfile
COPY index.html /usr/share/nginx/html/index.html
```
- 재빌드 후 `curl http://localhost`로 커스텀 페이지 노출 확인
- 퍼미션 문제 발생 시 `RUN chown nginx:nginx /usr/share/nginx/html/index.html` 추가로 해결

### Step 8. DockerHub 계정 및 Push
- DockerHub 계정 생성 → Settings > Personal access tokens에서 액세스 토큰 발급
- `docker login -u iceman4u`로 CLI 로그인
- `docker search ubuntu`, `docker pull --all-tags hello-world`로 이미지 검색·전체 태그 pull 실습
- `docker build -t simple_hello_world .` → `docker tag simple_hello_world:latest iceman4u/simple_hello_world:v0.0.1` → `docker push iceman4u/simple_hello_world:v0.0.1`로 개인 저장소에 업로드

---

## 결과 / 산출물
- 컨테이너 라이프사이클 전체 명령(run/ps/stop/start/rm/prune) 실습 완료
- Dockerfile 기반 NGINX 커스텀 이미지 빌드 (vim 설치 + index.html COPY) 완료
- DockerHub에 개인 이미지(`iceman4u/simple_hello_world:v0.0.1`) push 완료

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 컨테이너 라이프사이클 각 명령의 상태 전이를 직접 확인하며 체득함 | 실습 중 저장을 안 하고 재시작을 눌러서 작업 내용이 날아감 — 앞으로 저장 습관에 더 주의하기로 함 |
| Dockerfile로 수동 작업을 코드화하는 과정을 COPY/RUN 단계별로 직접 경험함 | |

---
*작성일: 2026-07-07 | 인공지능 사관학교 AI 보안 과정*
