# 2026-07-14 실습 — Git Branch & Pull Request 워크플로우

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습
> **관련 강의 파일**: `../lectures/2026-07-14_AI-Security-Course_Lecture_Git-Branch-Strategy.md`

---

## 느낀 것 / 배운 것

### 📅 2026-07-14
- 오늘 실습은 대부분 이미 익숙한 흐름이라 새로울 게 많지 않았다.
- 가장 새로웠던 건, PR에 리뷰 코멘트를 남긴 뒤 로컬에서 작업해서 commit → push 하면 그 PR이 그대로 이어진다는 점이었다. 예전에는 이걸 모르고 PR을 close한 다음 다시 올리곤 했는데, 이제 PR 갱신 흐름을 정확히 알게 됐다.
- 코드리뷰 스텝(Step 15)을 직접 해보면서 "사람이 코드리뷰를 못 따라가는 문제"가 다시 떠올랐고, 이건 실제로 파볼 생각이다 (lecture 파일 "다음에 파고들 만한 질문" 참고).

---

## 실습 개요
- **목표**: `devmlops` 저장소에서 feature 브랜치 생성 → 작업 → PR → 리뷰 반영 → merge까지 Git 협업 흐름 전체를 손으로 익히기
- **방법**: GitHub 웹 UI + 로컬 git 명령어(PowerShell/bash) 병행

---

## 진행 과정

### Step 0. 환경설정 및 기본 명령 복습 (Step 1~10)
- GitHub 계정 인증(PAT/SSH), `devmlops` 저장소 clone, `git status`/`add`/`commit`/`push`, push 거절 시 `pull --no-rebase` 대응, `git restore --staged`로 스테이징 취소까지 — 이미 익숙한 내용이라 복습 수준으로 진행

### Step 1. feature 브랜치 생성 및 작업
- `git checkout -b feature/profile-api`로 새 브랜치 생성
- `profile.md` 파일 작성 후 `git add` → `git commit`

### Step 2. 원격 push 및 PR 생성
- `git push origin feature/profile-api`로 브랜치를 원격에 새로 push
- GitHub에서 "Compare & pull request"로 PR 생성 (제목 `feat: 사용자 프로필 파일 추가`, 본문에 변경 내용 / 변경 이유 / 확인 방법 작성)

### Step 3. 리뷰 코멘트 및 반영
- PR의 Files changed 탭에서 코멘트 남기고 Submit review
- 로컬에서 `git pull --no-rebase origin feature/profile-api`로 원격 최신 내용 반영
- 리뷰 코멘트대로 `profile.md`에 `role`, `github` 항목 추가 후 커밋 → 다시 `git push origin feature/profile-api`
- **여기서 새로 알게 된 점**: 같은 브랜치에 커밋을 추가로 push하면 기존 PR이 자동으로 갱신됨(새 PR을 만들거나 재오픈할 필요 없음)

### Step 4. merge 및 브랜치 정리
- PR 승인(Approve) → Merge pull request → Confirm merge → Delete branch
- 로컬에서도 `git checkout main` → `git pull origin main` → `git branch -d feature/profile-api` → `git fetch --prune`으로 원격 삭제 정보 반영

---

## 결과 / 산출물
- `devmlops` 저장소 main에 병합된 `profile.md`
- 로컬/원격 모두 정리된 브랜치 상태 (`feature/profile-api` 삭제 확인)

---

## 잘 된 것 / 안 된 것

| 잘 된 것 |
|---|
| 브랜치 생성 → PR → 리뷰 반영 → merge → 정리까지 전체 흐름을 막힘없이 완주 |

---

## 다음에 이어서 할 것
- Step 17 이후(같은 파일을 두 브랜치에서 다르게 수정해 충돌 상황 재현 및 해결)는 다음 실습에서 이어서 진행 예정

---
*작성일: 2026-07-14 | 인공지능 사관학교 AI 보안 과정*
