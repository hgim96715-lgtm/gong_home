---
aliases:
  - git
  - git commit
  - git push
  - git reset
  - git amend
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
---
# Git_Commands — 자주 쓰는 Git 명령어

## 한 줄 요약

```
버전 관리 + 협업 도구
커밋 수정 / 되돌리기 / 강제 푸시가 자주 필요함
```

---

---

# ① 기본 흐름

```bash
git status                    # 변경 파일 확인
git add .                     # 전체 스테이징
git add 파일명                 # 특정 파일만
git commit -m "메시지"         # 커밋
git push origin main           # 원격 푸시
```

---

---

# ① -1 로컬 폴더 → GitHub 처음 올리기 ⭐️

```
새 프로젝트 폴더를 GitHub 에 처음 연결할 때
GitHub 에서 새 저장소 만든 뒤 아래 명령어 실행
```

```bash
# 로컬 폴더에서 Git 초기화
git init
git add .
git commit -m "first commit"
git branch -M main

# GitHub 원격 저장소 연결
git remote add origin https://github.com/사용자명/저장소명.git

# 처음 push (-u 로 upstream 설정)
git push -u origin main
```

```
-u (--set-upstream) 의미:
  처음 한 번만 붙이면
  이후부터는 git push 만 해도 됨 (origin main 생략 가능)

이미 git init 된 폴더라면:
  git init 건너뛰고
  git remote add origin ... 부터 시작
```

## 이미 로컬에 저장소가 있을 때

```bash
# GitHub 에서 새 저장소 만들고
git remote add origin https://github.com/사용자명/저장소명.git
git branch -M main
git push -u origin main
```

## 원격 저장소 확인 / 변경

```bash
git remote -v                                     # 현재 연결된 원격 확인
git remote set-url origin https://새_주소.git     # 원격 주소 변경
git remote remove origin                          # 원격 연결 제거
```

---

---

# ② commit --amend — 마지막 커밋 수정

```
방금 한 커밋을 수정하고 싶을 때
메시지 수정 / 파일 추가 / 코드 수정
```

```bash
# 커밋 메시지만 수정
git commit --amend -m "새 메시지"

# 파일 추가 후 커밋에 포함
git add 빠트린_파일.py
git commit --amend --no-edit    # 메시지 유지

# 원격에 반영 (강제 푸시 필요)
git push --force
git push --force-with-lease     # 더 안전한 방법 ← 권장
```

```
⚠️ push --force 주의:
  혼자 작업하는 브랜치에서만 사용
  팀 공유 브랜치에서 force push 하면 다른 사람 커밋 날아갈 수 있음
  --force-with-lease: 원격에 내가 모르는 커밋 있으면 거부 → 더 안전
```

---

---

# ③ reset — 커밋 되돌리기

```
reset 종류:
  --soft   커밋만 취소, 변경사항 staged 상태 유지  ← 자주 씀
  --mixed  커밋 취소 + unstaged 상태 (기본값)
  --hard   커밋 취소 + 변경사항 전부 삭제 ⚠️
```

```bash
# 마지막 커밋 취소 (변경사항 유지)
git reset --soft HEAD~1

# 여러 개 취소
git reset --soft HEAD~3        # 최근 3개 커밋 취소

# 특정 커밋으로 되돌리기
git reset --soft 해시값         # git log 에서 해시 확인
```

```bash
# 원격에 반영
git push origin main --force
```

```
HEAD~1 vs 해시:
  HEAD~1    현재에서 1개 전 커밋
  HEAD~3    현재에서 3개 전 커밋
  abc1234   git log 에서 복사한 특정 커밋 해시

--soft 후 상태:
  취소된 커밋의 변경사항이 staged 로 남아있음
  → 다시 수정 후 커밋 가능
```

---

---

# ④ log — 커밋 이력 확인

```bash
git log                        # 전체 이력
git log --oneline              # 한 줄로 간결하게 ← 자주 씀
git log --oneline -5           # 최근 5개만
```

```
예시 출력:
  abc1234 (HEAD -> main) 세 번째 커밋
  def5678 두 번째 커밋
  ghi9012 첫 번째 커밋

→ reset --soft def5678 하면 "세 번째 커밋" 만 취소됨
```

---

---

# ⑤ 자주 쓰는 패턴

```bash
# 커밋 메시지 오타 수정
git commit --amend -m "올바른 메시지"
git push --force

# 파일 빠트리고 커밋했을 때
git add 빠트린_파일.py
git commit --amend --no-edit
git push --force

# 커밋 여러 개를 하나로 합치기 (squash)
git reset --soft HEAD~3        # 3개 취소
git commit -m "합친 커밋 메시지"
git push origin main --force

# 실수로 .env 커밋했을 때
git reset --soft HEAD~1        # 마지막 커밋 취소
# .gitignore 에 .env 추가
git add .gitignore
git commit -m "fix: remove .env"
git push origin main --force
```

---

---

# ⑥ 기타 유용한 명령어

```bash
# 원격 저장소 확인
git remote -v

# 브랜치 목록
git branch
git branch -a                  # 원격 포함

# 변경사항 임시 저장 (stash)
git stash                      # 임시 저장
git stash pop                  # 복원

# 특정 파일 변경사항 되돌리기
git checkout -- 파일명

# 스테이징 취소
git restore --staged 파일명
```

---

---

# 한눈에 정리

|상황|명령어|
|---|---|
|새 폴더 GitHub 에 처음 올리기|`git init` → `git remote add origin` → `git push -u origin main`|
|이미 있는 폴더 GitHub 연결|`git remote add origin` → `git push -u origin main`|
|마지막 커밋 메시지 수정|`git commit --amend -m "새 메시지"`|
|파일 빠트리고 커밋|`git add 파일` + `git commit --amend --no-edit`|
|마지막 커밋 취소 (변경사항 유지)|`git reset --soft HEAD~1`|
|특정 커밋으로 되돌리기|`git reset --soft 해시값`|
|강제 푸시|`git push --force-with-lease`|
|커밋 이력 확인|`git log --oneline`|