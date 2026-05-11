---
aliases:
  - git 되돌리기
  - git restore
  - git reset
  - git revert
tags:
  - Git
related:
  - "[[00_Git_HomePage]]"
  - "[[Git_Commands]]"
  - "[[Git_Concept]]"
---

# Git_Undo — 되돌리기 & 실수 복구

## 한 줄 요약

```
git restore --staged  스테이징 취소 (add 취소)
git restore           워킹 디렉토리 변경 취소
git reset             커밋 취소
git revert            커밋 내용을 되돌리는 새 커밋
```

---

---

# ① git restore --staged — 스테이징 취소 ⭐️

```
git add 했다가 마음이 바뀔 때
스테이징 영역에서 꺼내기 (파일 내용은 그대로)
```

```bash
# 특정 파일 스테이징 취소
git restore --staged hello.py

# 전체 스테이징 취소
git restore --staged .

# 확인
git status
# Changes not staged for commit:
#   modified: hello.py  ← Staged 에서 내려옴
```

```
git restore --staged 동작:
  파일을 Staged → Unstaged 로 이동
  파일 내용 자체는 변경 안 됨
  커밋 준비 목록에서만 빠짐

여행 가방 비유:
  가방(Staging Area) 에 담았다가
  다시 꺼내서 방(워킹 디렉토리) 에 두기
  물건(파일 내용) 자체는 그대로
```

---

---

# ② git restore — 워킹 디렉토리 변경 취소

```
파일 수정했다가 취소하고 싶을 때
마지막 커밋 상태로 되돌림
→ ⚠️ 수정 내용이 사라짐 (복구 불가)
```

```bash
# 특정 파일 수정 취소 (마지막 커밋 상태로)
git restore hello.py

# 전체 취소
git restore .
```

```
⚠️ 주의:
  git restore (--staged 없음) 는 파일 내용 자체를 되돌림
  수정한 내용이 영구 삭제됨
  실행 전 신중하게 확인
```

## restore 두 가지 비교

|명령어|대상|파일 내용|
|---|---|---|
|`git restore --staged 파일`|Staged → Unstaged|변경 없음 ✅|
|`git restore 파일`|수정 내용 취소|마지막 커밋으로 ⚠️|

---

---

# ③ git reset — 커밋 취소

## reset 3가지 모드

```bash
# --soft: 커밋만 취소 / 스테이징 유지
git reset --soft HEAD~1
# → 이전 커밋으로 / 변경 파일은 Staged 상태로

# --mixed (기본값): 커밋 + 스테이징 취소 / 파일 내용 유지
git reset HEAD~1
git reset --mixed HEAD~1
# → 이전 커밋으로 / 변경 파일은 Unstaged 상태로

# --hard: 커밋 + 스테이징 + 파일 내용 전부 취소
git reset --hard HEAD~1
# ⚠️ 파일 내용까지 사라짐 / 복구 어려움
```

|모드|커밋 취소|스테이징 취소|파일 내용|
|---|:-:|:-:|:-:|
|`--soft`|✅|❌|유지|
|`--mixed`|✅|✅|유지|
|`--hard`|✅|✅|⚠️ 삭제|

```
실무 사용:
  --soft   커밋 메시지 잘못 썼을 때 (내용은 유지하고 재커밋)
  --mixed  커밋 내용 일부 수정 후 재커밋
  --hard   완전히 없애고 싶을 때 (되도록 사용 금지)
```

---

---

# ④ git revert — 안전한 되돌리기

```
reset 은 커밋 이력 자체를 지움
revert 는 되돌리는 내용을 새 커밋으로 만듦
→ 공유 브랜치(main) 에서 안전하게 사용 가능
```


```bash
# 마지막 커밋 되돌리기
git revert HEAD

# 특정 커밋 되돌리기
git revert abc1234
git log --oneline   # revert 커밋이 새로 추가됨

# 커밋 없이 변경만 (직접 커밋할 때)
git revert --no-commit HEAD
```

```
git revert 실행 시 에디터가 열림:
  메시지 수정 없이 그대로 저장하고 닫기
  Vim:  Esc → :wq → Enter
  nano: Ctrl+X → Y → Enter
```

## reset vs revert ⭐️

| |reset|revert|
|---|---|---|
|방식|커밋 이력 삭제|새 커밋 추가|
|히스토리|재작성됨|보존됨|
|push|force push 필요|일반 push 가능|
|사용 상황|혼자 쓰는 브랜치|공유 브랜치 ✅|

```
공유 브랜치(main)에서 실수 → revert 사용
  이유 ①: 히스토리 보존 (팀원이 변경 이력 볼 수 있음)
  이유 ②: force push 불필요 (일반 push 로 공유)
  이유 ③: 감사 추적 (언제 왜 취소됐는지 기록 남음)
```

---

---

# ⑤ git stash — 임시 저장

```
작업 중인데 다른 브랜치로 급히 이동해야 할 때
현재 변경 사항을 임시 보관
```

```bash
# 현재 변경 임시 저장
git stash

# stash 목록 확인
git stash list
# stash@{0}: WIP on main: abc1234 메시지

# 가장 최근 stash 꺼내기
git stash pop      # stash 에서 삭제하며 꺼내기
git stash apply    # stash 는 유지하며 꺼내기

# 특정 stash 꺼내기
git stash pop stash@{1}

# stash 삭제
git stash drop stash@{0}
git stash clear    # 전체 삭제
```

---

---

# 상황별 선택 가이드

```
git add 했다가 취소    → git restore --staged
파일 수정 취소         → git restore (⚠️ 내용 삭제)
커밋 취소 (혼자)       → git reset --soft / --mixed
커밋 취소 (공유)       → git revert
작업 임시 저장         → git stash
```

|상황|명령어|안전성|
|---|---|---|
|add 취소|`git restore --staged`|✅ 안전|
|수정 취소|`git restore`|⚠️ 내용 삭제|
|커밋 취소 (soft)|`git reset --soft`|✅ 안전|
|커밋 취소 (hard)|`git reset --hard`|❌ 위험|
|공유 브랜치 되돌리기|`git revert`|✅ 안전|
|작업 임시 저장|`git stash`|✅ 안전|