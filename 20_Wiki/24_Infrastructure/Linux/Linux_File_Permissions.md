---
aliases:
  - 파일 권한
  - chmod
  - chown
  - chgrp
  - permission denied
  - "755"
  - "777"
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_User_Group]]"
  - "[[Linux_Filesystem]]"
  - "[[Linux_Basic_Commands]]"
---
# Linux_File_Permissions — 파일 권한

## 한 줄 요약

```
모든 파일 / 디렉토리에 "누가 무엇을 할 수 있는가" 가 적혀 있음
Permission denied = 권한 없음
```

---

---

# ① 권한 구조 — ls -l 해석 ⭐️

```bash
ls -l
# -rw-r--r--  1  root  root   681  Mar 23  xattr.conf
# drwxr-xr-x  4  root  root  4096  Jan 25  ssl/
```

## 7개 구역 해석

```
-  rw-  r--  r--   1   root   root   681   Mar 23   xattr.conf
①   ②    ③    ④   ⑤    ⑥      ⑦     ⑧      ⑨        ⑩

① 파일 타입:  -=파일 / d=디렉토리 / l=심볼릭링크
② 소유자(u) 권한: rw-  (읽기+쓰기)
③ 그룹(g) 권한:   r--  (읽기만)
④ 기타(o) 권한:   r--  (읽기만)
⑤ 링크 수
⑥ 소유자 이름
⑦ 소유 그룹 이름
⑧ 파일 크기 (byte)
⑨ 마지막 수정 날짜
⑩ 파일명
```

## 권한 기호 & 숫자

|기호|의미|숫자|
|---|---|:-:|
|`r`|Read (읽기)|4|
|`w`|Write (쓰기)|2|
|`x`|eXecute (실행)|1|
|`-`|권한 없음|0|

```
암산 연습:
  rwx = 4+2+1 = 7  (모든 권한)
  rw- = 4+2+0 = 6  (읽기+쓰기)
  r-x = 4+0+1 = 5  (읽기+실행)
  r-- = 4+0+0 = 4  (읽기만)
  --- = 0           (권한 없음)
```

## 실전 해석

```bash
# 파일: 소유자=읽쓰 / 그룹=읽기 / 기타=읽기
-rw-r--r--  root  root  subgid
→ 644

# 디렉토리: 소유자=모두 / 그룹=읽기+실행 / 기타=읽기+실행
drwxr-xr-x  root  root  ssl/
→ 755

# 디렉토리: 소유자=모두 / 그룹=모두 / 기타=읽기+실행
drwxrwxr-x  labex  research  RandD/
→ 775
```

---

---

# ② chmod — 권한 변경 ⭐️

## 숫자 모드 (실무 선호)

```bash
chmod 755 script.sh   # 주인=7(rwx) / 그룹=5(r-x) / 기타=5(r-x)
chmod 644 file.txt    # 주인=6(rw-) / 그룹=4(r--) / 기타=4(r--)
chmod 600 secret.key  # 주인=6(rw-) / 그룹=0(---) / 기타=0(---)
chmod -R 755 mydir/   # 디렉토리 전체 재귀 적용
```

## 기호 모드 (직관적)

```bash
chmod +x run.sh          # 모두에게 실행 권한 추가
chmod g+w file.txt       # 그룹에 쓰기 권한 추가
chmod o-r file.txt       # 기타에서 읽기 권한 제거
chmod u+x,g-w file.txt   # 소유자 실행 추가 + 그룹 쓰기 제거

# u=소유자 / g=그룹 / o=기타 / a=전체
# +=추가 / -=제거 / ==정확히 설정
```

## 자주 쓰는 권한 값

```
644  일반 파일 (소유자 읽쓰 / 나머지 읽기)
755  실행 파일 / 디렉토리 (소유자 전체 / 나머지 읽기+실행)
600  민감한 파일 (소유자만 읽쓰)
700  민감한 디렉토리 (소유자만 접근)
775  팀 공용 디렉토리 (소유자+그룹 전체 / 기타 읽기+실행)
777  ❌ 절대 금지 (모든 사람이 모든 것 가능)
```

---

---

# ③ chown — 소유자 / 그룹 변경 ⭐️

```
chown = Change Owner
파일·디렉토리의 소유자와 그룹을 변경
sudo 권한 필요
```

## 기본 문법

```bash
chown [소유자]:[그룹] 대상
```

## 패턴별 사용법

```bash
# 소유자만 변경
chown john file.txt

# 소유자 + 그룹 동시 변경 (가장 많이 씀)
chown labex:research RandD

# 그룹만 변경 (소유자 생략)
chown :research RandD
chown :developers /project

# 재귀 변경 (-R) — 디렉토리 안 전부
sudo chown -R labex:research RandD/
sudo chown -R john:developers /var/www/html
```

## 디렉토리 소유권 변경 실전 ⭐️

```bash
# 1. 디렉토리 생성
mkdir RandD
ls -l
# drwxrwxr-x  labex  labex  RandD    ← 생성자 = 소유자 / 주 그룹

# 2. 그룹을 research 로 변경
sudo chown labex:research RandD
ls -l
# drwxrwxr-x  labex  research  RandD  ← 그룹 변경됨!
```

```
새 디렉토리 기본 소유권:
  소유자 = 만든 사람
  그룹   = 만든 사람의 주 그룹 (Primary Group)

팀 협업 설정:
  chown 소유자:팀그룹 공용디렉토리
  chmod 775 공용디렉토리
  → 팀원들이 자유롭게 파일 공유 가능
```

---

---

# ④ chgrp — 그룹만 변경

```
chown :그룹 으로 대체 가능
실무에서는 chown 사용 권장
```

```bash
# chgrp 방식
chgrp -R developers /project

# chown 방식 (추천 ⭐️)
chown -R :developers /project
# ↑ 소유자 생략 = 소유자는 그대로 / 그룹만 변경
```

---

---

# ⑤ 소유자 / 그룹 / 권한 흐름 ⭐️

```
그룹 생성
  sudo groupadd research

사용자를 그룹에 추가
  sudo usermod -aG research labex

디렉토리 생성
  mkdir RandD

소유권 변경
  sudo chown labex:research RandD

권한 설정
  chmod 775 RandD

결과:
  drwxrwxr-x  labex  research  RandD
  → labex 소유 / research 그룹 전체 읽쓰실 / 기타 읽기+실행
  → research 그룹에 속한 팀원 모두 사용 가능
```

```bash
# 전체 흐름 한 번에
sudo groupadd research
sudo usermod -aG research labex
mkdir RandD
sudo chown labex:research RandD
chmod 775 RandD
ls -l
# drwxrwxr-x  labex  research  RandD
```

---

---

# ⑥ 실무 에러 패턴

```
Permission denied 원인 3가지:

1. 실행 권한 없음
   ./run.sh → Permission denied
   해결: chmod +x run.sh

2. 소유자 / 그룹 달라서 쓰기 불가
   설정 파일 저장 안 됨
   해결: sudo chown 나:내그룹 파일
         또는 sudo vim 으로 편집

3. 웹서버 이미지 안 뜸
   Others 에 읽기(r) 권한 없음
   해결: chmod o+r 이미지파일
         또는 chmod 644 이미지파일
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`chmod 777`|"귀찮아서"|`755` 또는 `644` 사용|
|`chmod -R 777 /`|실수로 루트 전체|경로 세 번 확인 / `-R` 신중하게|
|`usermod` 후 즉시 적용 안 됨|세션 미반영|재로그인 또는 `newgrp 그룹명`|
|그룹만 바꾸려고 `chgrp` 찾음|명령어 모름|`chown :그룹명 파일` 로 대체|
|`-R` 없이 디렉토리 변경|하위 파일 미반영|`chown -R` 사용|