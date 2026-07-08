# 2026-07-08 실습 — Express App 컨테이너화

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-08_AI-Security-Course_Lecture_Docker-Image-Network.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-08

- 어제까지는 남이 만든 nginx 이미지를 가져다 커스터마이징하는 수준이었는데, 오늘은 직접 짠 Express 앱을 처음부터 이미지로 만들어봤다. "코드 작성 → Dockerfile → build → run"의 전체 라이프사이클을 한 바퀴 직접 돌려보니, Docker가 결국 "내 앱 + 실행 환경"을 통째로 하나로 묶는 도구라는 게 몸으로 이해됨.
- `docker stop`이 컨테이너를 종료하는 것과 달리, `docker pause`는 프로세스를 "얼리는" 것(SIGSTOP 기반)이라 unpause하면 멈췄던 지점에서 그대로 재개된다는 차이를 알게 됨.

---

## 실습 개요
- **목표**: Node.js/Express로 작성한 백엔드 API를 Dockerfile로 이미지화하고, 컨테이너로 실행해 개발→빌드→실행 라이프사이클 전체를 경험한다.
- **방법**: npm 프로젝트 초기화 → Express 앱 작성(라우팅 + 인메모리 사용자 등록) → Dockerfile 작성 → build/run → Postman으로 API 테스트 → pause/unpause로 컨테이너 상태 제어

---

## 진행 과정

### Step 1. Express 앱 작성
- `npm init -y`로 프로젝트 초기화, `npm install express@4.19.2 body-parser@1.20.2 --save-exact`로 버전 고정 설치
- `index.js`에 기본 라우팅(`GET /` → "Hello Docker World!") 작성
- `package.json`의 `scripts`에서 `test` 제거하고 `"start": "node index.js"` 추가
- 처음엔 라우팅만 작성하고 `app.listen()`을 빼먹어서 서버가 안 떴음 → `app.listen(port, ...)` 추가 후 정상 기동

### Step 2. 인메모리 사용자 기능 추가
- `const users = []` 배열을 메모리에 두고 두 개의 라우터 추가:
  - `GET /users`: 등록된 사용자 목록 반환 (`res.json({ users })`)
  - `POST /users`: `userId`를 받아 등록. 누락 시 400(Missing userId), 중복 시 400(already exists), 성공 시 201
- Postman을 설치해 GET/POST 요청을 직접 날려 동작 확인

### Step 3. 애플리케이션 도커화
- `src` 폴더를 만들어 `index.js`를 이동, `package.json`의 경로도 그에 맞게 수정
- Dockerfile 작성:
```dockerfile
FROM node:22

WORKDIR /app
# Copy the package.json and package-lock.json
COPY package*.json .
# Install the dependencies
RUN npm ci
# Copy the source code of our app
COPY src/index.js index.js
# Expose the application port
EXPOSE 3000
# Start application
CMD ["node", "index.js"]
```
- `COPY package*.json .`으로 package.json/package-lock.json을 먼저 복사한 뒤 `npm ci`로 설치 → 의존성 레이어를 코드 레이어와 분리해 캐시 효율을 노림

### Step 4. 빌드 및 실행
```bash
docker build -t express_app:v0.0.1 .
docker images
docker run -p 3000:3000 --name express_app express_app:v0.0.1
```
- 빌드한 커스텀 이미지로 컨테이너를 띄우고 API가 정상 응답하는지 확인

### Step 5. 컨테이너 상태 제어 (pause / unpause)
```bash
docker pause express_app
docker unpause express_app
```
- 컨테이너를 종료하면 인메모리 데이터가 사라짐을 확인
- `pause`는 컨테이너를 종료하지 않고 프로세스를 일시 정지시키는 것 (unpause 시 정지 시점부터 재개)

---

## 결과 / 산출물
- Express 기반 백엔드 API(사용자 등록/조회) 작성 완료
- `express_app:v0.0.1` 커스텀 이미지 빌드 및 컨테이너 실행 완료
- Postman으로 GET/POST API 동작 검증 완료

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 직접 작성한 앱을 처음부터 이미지화해 라이프사이클 전체를 완주함 |
| Dockerfile의 COPY/RUN 순서를 레이어 캐시 관점으로 이해하게 됨 |

---
*작성일: 2026-07-08 | 인공지능 사관학교 AI 보안 과정*
