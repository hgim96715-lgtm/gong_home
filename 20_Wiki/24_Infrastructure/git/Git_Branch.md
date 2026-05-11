---
aliases:
  - git branch
  - 브랜치
  - git merge
  - git switch
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Concept]]"
  - "[[Git_Commands]]"
  - "[[Git_Undo]]"
---

# Git_Branch — 브랜치 & 병합

## 한 줄 요약

```
브랜치 = 메인 코드 건드리지 않고 새 기능 개발하는 별도 시간선
main 에서 브랜치 → 작업 → 다시 main 에 병합
```

---

---

# ① 브랜치 기본 명령어

## 브랜치 목록 확인

```bash
git branch          # 로컬 브랜치 목록
git branch -a       # 원격 포함 전체 목록
git branch --merged # 현재 브랜치에 병합된 브랜치만

# * = 현재 내가 있는 브랜치
# * master
#   feature-dimension
```

## 브랜치 생성

```bash
git branch feature-dimension        # 브랜치 생성 (이동 안 함)
git branch hotfix/login-bug         # 슬래시로 계층 구조 가능
```

## 브랜치 이동

```bash
# 구버전 방식
git checkout feature-dimension

# 신버전 방식 (Git 2.23 이상 권장) ⭐️
git switch feature-dimension
```

## 생성 + 이동 한 번에 ⭐️

```bash
# 구버전
git checkout -b feature-dimension

# 신버전 (권장)
git switch -c feature-dimension
#          ↑ -c = create
```

```
checkout vs switch:
  git checkout  브랜치 이동 + 파일 복구 등 여러 기능
  git switch    브랜치 이동 전용 (더 명확 / 권장)

  둘 다 결과 동일 / switch 가 의도를 더 명확히 전달
```

---

---

# ② 브랜치 작업 흐름 ⭐️

```bash
# 1. main 에서 새 브랜치 생성 + 이동
git switch -c feature/새기능

# 2. 작업 후 커밋
git add .
git commit -m "feat: 새 기능 추가"

# 3. main 으로 복귀
git switch main

# 4. 브랜치 병합
git merge feature/새기능

# 5. 브랜치 삭제
git branch -d feature/새기능
```

---

---

# ③ git merge — 병합

```bash
# 현재 브랜치에 다른 브랜치를 병합
git switch main
git merge feature-dimension
# → feature-dimension 의 변경사항이 main 에 합쳐짐
```

## merge 출력 해석

```
Fast-forward:
  main 에 새 커밋이 없어서 그냥 앞으로 이동
  가장 깔끔한 병합

Merge commit:
  양쪽 모두 커밋이 있어서 합치는 커밋 생성
  "Merge branch 'feature-...' into main"
```

## 브랜치 간 차이 확인

```bash
# 현재 브랜치와 main 의 차이
git diff main

# 두 브랜치 비교
git diff main..feature-dimension
```

---

---

# ④ 브랜치 삭제

```bash
# 안전한 삭제 (-d) — 병합된 브랜치만 삭제
git branch -d feature-dimension
# 병합 안 됐으면 에러 → 실수 방지 안전장치

# 강제 삭제 (-D) — 병합 여부 무관
git branch -D feature-dimension
# ⚠️ 병합 안 된 커밋도 사라짐 / 신중하게 사용
```

```
-d vs -D:
  -d  병합된 것만 삭제 (안전) ← 주로 이걸 쓸 것
  -D  무조건 삭제 (위험)
  
병합 완료 확인:
  git branch --merged  → 목록에 있으면 -d 가능
```

---

---

# ⑤ 브랜치 네이밍 컨벤션

```
feature/기능명    새 기능 개발
fix/버그명        버그 수정
hotfix/긴급수정   긴급 패치
refactor/내용     코드 개선
chore/작업        설정 변경 / 패키지 업데이트
release/버전      배포 준비

예시:
  feature/user-login
  fix/payment-null-error
  hotfix/security-patch
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`git branch`|브랜치 목록|
|`git branch 이름`|브랜치 생성|
|`git switch 이름`|브랜치 이동|
|`git switch -c 이름`|생성 + 이동 ⭐️|
|`git checkout -b 이름`|생성 + 이동 (구버전)|
|`git merge 브랜치`|병합|
|`git branch -d 이름`|브랜치 삭제 (안전)|
|`git branch -D 이름`|브랜치 강제 삭제|
|`git branch --merged`|병합된 브랜치 목록|
|`git branch -a`|원격 포함 전체 목록|
|`git diff main`|현재 브랜치 vs main 차이|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|main 에서 작업함|브랜치 이동 안 함|작업 전 `git switch -c 브랜치명`|
|merge 후 브랜치 안 지움|깜빡|`git branch --merged` 확인 후 `-d`|
|`-D` 로 잘못 삭제|실수|`git reflog` 로 복구 가능|
|checkout 으로 파일 복구됨|checkout 의 다중 기능 때문|이동은 `switch` / 파일 복구는 `restore`|