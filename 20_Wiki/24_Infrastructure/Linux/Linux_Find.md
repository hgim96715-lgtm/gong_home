---
aliases:
  - find
  - locate
  - whereis
  - which
  - type
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Basic_Commands]]"
  - "[[Linux_Grep]]"
  - "[[Linux_Environment_Variables]]"
---

# Linux_Find — 파일 & 명령어 검색

## 한 줄 요약

```
find    → 실시간 파일 검색 (정확 / 느림)
locate  → DB 기반 파일 검색 (빠름 / 최신 파일 못 찾음)
whereis → 명령어 바이너리 / 매뉴얼 위치
which   → 실행될 명령어 경로
type    → 명령어 정체 (alias / builtin / 외부 프로그램)
```

---

---

# ① find — 실시간 파일 검색 ⭐️

## 기본 구조

```bash
find [경로] [조건]

find .          # 현재 디렉토리부터 전체 검색
find /etc       # /etc 부터 검색
find /          # 루트부터 전체 (느림 / 부하 큼)
```

## 이름으로 검색

```bash
find . -name "report.txt"       # 정확한 파일명
find . -name "*.txt"            # 확장자로 검색
find . -iname "*.txt"           # 대소문자 무시 (-iname)
find . -name "*.log"            # 로그 파일 전부
```

```
⚠️ 따옴표 필수:
  find . -name *.txt     ← 따옴표 없으면 쉘이 먼저 확장
                           현재 디렉토리 파일로 치환됨 → 엉뚱한 결과
  find . -name "*.txt"   ← 따옴표로 감싸야 find 가 제대로 처리
```

## 타입으로 검색

```bash
find . -type f              # 일반 파일만
find . -type d              # 디렉토리만
find . -type l              # 심볼릭 링크만
find . -type f -name "*.log"  # 일반 파일 중 .log
```

## 크기 / 시간 조건

```bash
find . -size +1M            # 1MB 초과 파일
find . -size -100k          # 100KB 미만 파일
find . -mtime -7            # 7일 이내 수정된 파일
find . -mtime +30           # 30일 이상 수정 안 된 파일
find . -newer file.txt      # file.txt 보다 최신 파일
```

## 권한 / 소유자 조건

```bash
find . -perm 755            # 권한이 755 인 파일
find . -user labex          # 소유자가 labex
find . -group docker        # 그룹이 docker
```

---

---

# ② -exec — 검색 결과에 명령어 실행 ⭐️

```bash
# 구조: find ... -exec 명령어 {} \;
#                          ↑  ↑
#                    찾은 파일  명령 끝

# 찾은 .txt 파일 상세 목록 출력
find . -name "*.txt" -exec ls -l {} \;

# 찾은 .log 파일 내용 출력
find . -name "*.log" -exec cat {} \;

# 찾은 파일 삭제 (확인 없이)
find . -name "*.tmp" -exec rm {} \;
```

## -ok — 실행 전 확인 (삭제 등 위험 작업)

```bash
# 각 파일마다 실행 전 y/n 물어봄
find . -name "*.log" -ok rm {} \;
# < rm ... ./file2.log > ? y
# < rm ... ./subdir/another.log > ? y
```

```
-exec vs -ok:
  -exec  → 확인 없이 바로 실행
  -ok    → 파일마다 y/n 확인 후 실행 (삭제 시 권장)
```

---

---

# ③ xargs — 파이프로 인자 전달 ⭐️

```
find | xargs 명령어
  find 결과를 명령어의 인자로 전달
  -exec 보다 빠름 (파일 한꺼번에 처리)
```

```bash
# find 결과를 ls -l 에 전달
find . -name "*.log" | xargs ls -l

# find 결과 파일 삭제
find . -name "*.log" | xargs rm

# 확인하며 삭제 (-p 옵션)
find . -name "*.log" | xargs -p rm
# rm ./file2.log ./subdir/another.log ?...y
```

## 공백 포함 파일명 — print0 + -0

```bash
# 파일명에 공백 있으면 xargs 가 잘못 분리
# → find -print0 | xargs -0 조합 사용

find . -name "*.txt" -print0 | xargs -0 ls -l
#                    ↑ NULL 문자로 구분    ↑ NULL 구분자로 읽기
```

## -exec vs xargs 성능 비교

```
-exec:
  찾은 파일마다 새 프로세스 생성
  파일 100개 → 프로세스 100번

xargs:
  여러 파일을 한 번에 명령어로 전달
  파일 100개 → 프로세스 1~2번
  대량 파일 처리 시 훨씬 빠름
```

---

---

# ④ locate — DB 기반 빠른 검색

```
미리 만들어진 파일 경로 DB 에서 검색
find 보다 훨씬 빠름
단, DB 업데이트 이후 생성된 파일은 못 찾음
```

```bash
# 설치 (없으면)
sudo apt-get install mlocate

# 검색
locate report.txt
locate "*.conf"

# DB 수동 업데이트
sudo updatedb

# 업데이트 전 새 파일 → 못 찾음
touch special_report.pdf
locate special_report.pdf   # 결과 없음

# updatedb 후 → 찾음
sudo updatedb
locate special_report.pdf   # 결과 나옴
```

```
find vs locate 선택:
  방금 만든 파일 / 정확한 검색  → find
  빠른 시스템 전체 검색         → locate
  locate DB = 보통 cron 으로 매일 자동 갱신
```

---

---

# ⑤ whereis — 명령어 위치 찾기

```
명령어의 바이너리 / 소스 / 매뉴얼 위치 찾기
일반 파일 검색 용도 아님 (시스템 디렉토리만 검색)
```

```bash
whereis passwd
# passwd: /usr/bin/passwd /etc/passwd /usr/share/man/man1/passwd.1.gz

whereis -b passwd    # 바이너리만
# passwd: /usr/bin/passwd

whereis -m passwd    # 매뉴얼만
# passwd: /usr/share/man/man1/passwd.1.gz

whereis report.txt   # 일반 파일 → 결과 없음 (표준 시스템 디렉토리에 없음)
```

```
whereis 활용:
  명령어 실행 파일 위치 확인
  설정 파일 위치 빠르게 파악
  매뉴얼 페이지 위치 확인
```

---

---

# ⑥ which / type — 명령어 정체 확인 ⭐️

## which — 실행될 명령어 경로

```bash
which python3    # /usr/bin/python3
which ls         # /usr/bin/ls
which pwd        # pwd: aliased to date  (alias 있으면 표시)
```

## type — 명령어 정체 완전 분석

```bash
type pwd
# pwd is a shell builtin       ← 쉘 내장 명령어

type ls
# ls is /usr/bin/ls            ← 외부 프로그램

# alias 설정 후
alias pwd='date'
type pwd
# pwd is an alias for date     ← alias

# -a 옵션: 모든 위치 + 우선순위 출력
type -a pwd
# pwd is an alias for date     ← 1순위 (alias)
# pwd is a shell builtin       ← 2순위 (내장)
# pwd is /usr/bin/pwd          ← 3순위 (외부)
# pwd is /bin/pwd
```

```
명령어 실행 우선순위:
  1. alias (별칭)
  2. shell builtin (내장 명령어)
  3. $PATH 의 외부 프로그램

which vs type:
  which  → 외부 프로그램 경로 (alias / builtin 구별 못할 때 있음)
  type -a → 정체 + 우선순위 전부 보여줌 (더 정확)
```

## alias 무시하고 원본 실행

```bash
alias rm='rm -i'   # rm 에 -i 붙인 alias

rm file.txt        # alias 실행 → 확인 물어봄
\rm file.txt       # alias 무시 → 원본 rm 실행 (확인 없이)
```

---

---

# ⑦ 검색 도구 선택 기준

|상황|도구|
|---|---|
|방금 만든 파일 찾기|`find`|
|조건 복잡한 검색|`find`|
|빠른 시스템 전체 검색|`locate`|
|명령어 실행 파일 위치|`whereis -b` / `which`|
|명령어 매뉴얼 위치|`whereis -m`|
|명령어 정체 정확히 확인|`type -a`|

---

---

# 자주 하는 실수

| 실수                          | 원인               | 해결                         |
| --------------------------- | ---------------- | -------------------------- |
| `find . -name *.txt` 이상한 결과 | 따옴표 없어서 쉘이 먼저 확장 | `"*.txt"` 따옴표 필수           |
| `find /` 너무 느림              | 루트부터 전체 탐색       | 경로 좁혀서 `find /etc` 처럼      |
| `locate` 로 새 파일 못 찾음        | DB 업데이트 전        | `sudo updatedb` 후 재검색      |
| 공백 파일명에서 `xargs` 오작동        | 공백을 구분자로 인식      | `find -print0 \| xargs -0` |
| `-exec` 대량 파일에서 느림          | 파일마다 프로세스 생성     | `xargs` 로 교체               |
| `which` 로 builtin 못 찾음      | which 는 외부 프로그램만 | `type -a` 사용               |