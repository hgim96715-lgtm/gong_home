---
aliases:
  - grep
  - 텍스트 검색
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Sed]]"
  - "[[Linux_Awk]]"
  - "[[Linux_File_Commands]]"
---
# Linux_Grep — 텍스트 검색

## 한 줄 요약

```
파일이나 출력에서 패턴(문자열/정규식)에 맞는 줄을 찾아내는 도구
grep = Global Regular Expression Print
```

---

---

# ① 기본 구조

```bash
grep '패턴' 파일
grep '패턴' 파일1 파일2      # 여러 파일 동시 검색 → 파일명도 출력됨
명령어 | grep '패턴'          # 파이프로 stdin 입력
```

```bash
# 단일 파일
grep labex /etc/passwd

# 여러 파일 동시 검색 — 파일명이 앞에 붙음
sudo grep labex /etc/passwd /etc/shadow /etc/group
# /etc/passwd:labex:x:1000:1000:...
# /etc/shadow:labex:$6$...
# /etc/group:labex:x:1000:
# ↑ 파일명:내용 형식으로 출력
```

```
권한 주의:
  /etc/shadow 같은 파일은 sudo 없으면 Permission denied
  반드시 sudo grep 으로 실행
```

---

---

# ② 자주 쓰는 옵션 ⭐️

|옵션|의미|예시|
|---|---|---|
|`-i`|대소문자 무시|`grep -i 'error' app.log`|
|`-n`|줄 번호 표시|`grep -n 'ERROR' app.log`|
|`-v`|반전 (매칭 안 되는 줄)|`grep -v '#' config.txt`|
|`-c`|매칭된 줄 수만 출력|`grep -c 'ERROR' app.log`|
|`-r`|디렉토리 재귀 탐색|`grep -r 'TODO' ./src`|
|`-l`|파일명만 출력|`grep -rl 'ERROR' ./logs`|
|`-w`|단어 단위 매칭|`grep -w 'log' app.log`|
|`-A n`|매칭 줄 아래 n줄 포함|`grep -A 3 'ERROR' app.log`|
|`-B n`|매칭 줄 위 n줄 포함|`grep -B 3 'ERROR' app.log`|
|`-C n`|위아래 n줄 포함|`grep -C 3 'ERROR' app.log`|
|`-E`|확장 정규식 사용|`grep -E 'err|
|`-o`|매칭된 부분만 출력|`grep -o '[0-9]\+' data.txt`|

---

---

# ③ 실전 패턴

## 기본 검색

```bash
# 특정 문자열 찾기
grep 'ERROR' app.log

# 대소문자 무시
grep -i 'error' app.log
```

## -n — 줄 번호 표시 ⭐️

```bash
grep -n 'ERROR' app.log
# 42:ERROR: connection failed
# ↑줄번호:내용

# 여러 파일에서 줄 번호 포함
sudo grep -n labex /etc/passwd /etc/shadow /etc/group
# /etc/passwd:32:labex:x:1000:...
# ↑파일명      :줄번호:내용
```

```
결과 해석:
  /etc/passwd:32:labex...
  → /etc/passwd 파일의 32번째 줄에 labex 존재

  활용:
    vim +32 /etc/passwd  ← 32번째 줄로 바로 이동해서 열기
    디버깅 / 설정 수정 시 정확한 위치 바로 파악 가능
```

## ^ $ 앵커 — 줄 위치 매칭 ⭐️

```bash
# 어디에 있든 root 검색
grep root /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# myroot:x:1001:...          ← 의도치 않은 결과 포함

# 줄이 root 로 시작하는 것만
grep ^root /etc/passwd
# root:x:0:0:root:/root:/bin/bash  ← root 계정만

# 줄이 bash 로 끝나는 것만
grep bash$ /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# labex:x:1000:1000::/home/labex:/bin/bash

# 빈 줄만 찾기 (^ 와 $ 사이 아무것도 없음)
grep ^$ file.txt

# 빈 줄 제외
grep -v ^$ file.txt
```

```
앵커 사용 효과:
  grep root       → root, myroot, root123 전부 매칭
  grep ^root      → root 로 시작하는 줄만
  grep bash$      → bash 로 끝나는 줄만
  → 검색 정확도 비약적 향상
```

## -w — 단어 단위 매칭 ⭐️

```bash
# 예시 파일
# there is the other log
# the log file
# logger.info

grep 'the' data.txt
# there is the other log   ← 'there', 'other' 안의 'the' 도 매칭
# the log file

grep -w 'the' data.txt
# the log file             ← 정확히 'the' 단어만
# 'there', 'other' 제외!
```

```
-w 의 기준:
  단어 경계 = 공백 / 특수문자 / 줄 시작·끝
  앞뒤가 알파벳·숫자·_ 이 아닐 때만 매칭

  'the'    ✅  단독 단어
  'there'  ❌  the + re → 단어 경계 아님
  'other'  ❌  ot + her → 단어 경계 아님
  'the_'   ❌  _ 도 단어 문자로 취급
```

```bash
# 실전: 정확히 'ERROR' 만 찾기 (ERRORS, error_msg 제외)
grep -w 'ERROR' app.log

# 변수명 정확히 찾기 (log 가 포함된 logger, logging 제외)
grep -w 'log' *.py
```

## 반전 (-v) — 없는 줄 찾기

```bash
# 주석(#) 줄 제외
grep -v '^#' config.txt

# 빈 줄 제외
grep -v '^$' data.txt

# 주석 + 빈 줄 동시 제외
grep -v '^#' config.txt | grep -v '^$'
```

## 여러 패턴 동시 검색

```bash
# -E 로 | 사용
grep -E 'ERROR|WARN' app.log

# -e 로 여러 패턴
grep -e 'ERROR' -e 'WARN' app.log

# 실전: 로그에서 여러 레벨 동시
grep -E 'ERROR|WARN|FAIL' app.log
```

## 주변 줄 포함 (-A -B -C)

```bash
# 에러 발생 전후 3줄 확인 (컨텍스트 파악)
grep -C 3 'ERROR' app.log
# --
# 정상 로그
# 정상 로그
# 정상 로그
# ERROR: something failed   ← 매칭
# 이후 로그
# 이후 로그
# 이후 로그
# --
```

## 재귀 검색 (-r)

```bash
# 현재 디렉토리 전체에서 찾기
grep -r 'TODO' ./

# 파일명만 출력 (-l)
grep -rl 'import pandas' ./

# 특정 확장자만
grep -r 'ERROR' --include='*.log' ./
grep -r 'def ' --include='*.py' ./
```

---

---

# ④ 정규식 패턴

## 기본 정규식 (BRE) ⭐️

```bash
# . 임의의 문자 1개
grep 'ro.' /etc/passwd        # ro 뒤에 문자 1개 (roa, rob, ro1 ...)

# * 앞 문자 0회 이상
grep 'roo*' /etc/passwd       # ro 뒤에 o 가 없거나(ro) / 있거나(roo, rooo)
```

```
주의: 정규식의 * 는 파일 검색의 * 와 다름!
  find *.txt       → 모든 txt 파일 (쉘 와일드카드)
  grep 'ab*' file  → ab, a, abb, abbb (앞 문자 0회 이상)
  정규식에서 "모든 문자" = .* (점 + 별)
```

```bash
# ^ 줄 시작
grep '^ERROR' app.log         # ERROR 로 시작하는 줄

# $ 줄 끝
grep 'failed$' app.log        # failed 로 끝나는 줄

# ^$ 빈 줄
grep '^$' file.txt            # 빈 줄만

# [] 문자 클래스
grep '[0-9]' data.txt         # 숫자 포함 줄
grep '[A-Z]' data.txt         # 대문자 포함 줄
grep '[aeiou]' data.txt       # 모음 포함 줄

# [^] 부정 문자 클래스
grep '[^0-9]' data.txt        # 숫자 아닌 문자 포함 줄
```

## 확장 정규식 (-E) ⭐️

```bash
# + 앞 문자 1회 이상
grep -E '[0-9]+' data.txt     # 숫자 1개 이상

# ? 앞 문자 0 또는 1회
grep -E 'colou?r' data.txt    # color 또는 colour

# {n} 정확히 n회
grep -E 'o{2}' /etc/passwd    # o 가 정확히 2번 (oo)

# {n,m} n~m회
grep -E '[0-9]{2,4}' data.txt # 숫자 2~4개

# | OR
grep -E 'root|Root' /etc/passwd   # root 또는 Root
grep -E 'ERROR|WARN|FAIL' app.log # 여러 레벨 동시

# () 그룹
grep -E '(ERROR|WARN): ' app.log
```

```
-E 없이 {} | + ? 쓰면:
  그냥 일반 문자로 취급 → 원하는 결과 안 나옴
  확장 정규식 쓸 땐 반드시 -E 붙이기
```

---

---

# ⑤ 파이프 조합 패턴

```bash
# 로그에서 에러만 뽑아서 개수 세기
grep 'ERROR' app.log | wc -l

# 특정 프로세스 확인
ps aux | grep 'python'

# 에러 줄에서 IP 주소만 추출
grep 'ERROR' app.log | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'

# 중복 제거 후 정렬
grep -o 'region=[a-z]*' data.txt | sort | uniq -c

# 로그에서 오늘 날짜 에러만
grep "$(date '+%Y-%m-%d')" app.log | grep 'ERROR'
```

---

---

# ⑥ grep vs egrep vs fgrep

```
grep    기본 정규식 (BRE)
egrep   확장 정규식 (ERE) — grep -E 와 동일
fgrep   정규식 없이 문자열 그대로 — grep -F 와 동일
        특수문자(. * [ 등)를 그대로 검색할 때

예시:
  grep -F '1.2.3.4' log.txt   ← . 을 임의 문자가 아닌 점으로 검색
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`grep -r` 결과가 너무 많음|모든 파일 탐색|`--include='*.py'` 로 확장자 제한|
|정규식 특수문자가 안 먹힘|BRE vs ERE 차이|`-E` 옵션 추가|
|`[0-9]+` 가 안 됨|기본 grep 은 `+` 인식 못함|`grep -E` 또는 `egrep` 사용|
|검색 결과에 바이너리 파일 포함|바이너리 자동 검색|`--include='*.txt'` 로 제한|
|패턴에 공백 있는데 매칭 안 됨|따옴표 없이 사용|`grep 'ERROR message'` 따옴표 필수|