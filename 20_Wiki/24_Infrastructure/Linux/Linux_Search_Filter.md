---
aliases:
  - grep
  - 검색
  - 필터
  - 로그 분석
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Redirect]]"
  - "[[Linux_Text_Commands]]"
  - "[[Linux_Diff]]"
  - "[[Linux_Log]]"
  - "[[00_Linux_HomePage]]"
---

# Linux_Search_Filter — grep & 로그 검색

## 한 줄 요약

```
grep = 파일이나 출력에서 특정 패턴이 포함된 줄만 걸러내기
로그 분석 / 에러 추출 / 설정 파일 검색에 매일 사용
```

---

---

# ① grep 기본 사용법 ⭐️

```bash
# 기본: grep "패턴" 파일
grep "ERROR" app.log

# 여러 파일
grep "ERROR" *.log

# 결과를 파일로 저장
grep "ERROR" app.log > error_report.txt

# 파이프로 연결
cat app.log | grep "ERROR"
dmesg | grep "error"
```

---

---

# ② grep 주요 옵션 ⭐️

## -i — 대소문자 무시

```bash
grep -i "error" app.log
# Error / ERROR / error / eRrOr 전부 검색
```

## -r — 디렉토리 재귀 검색

```bash
# 하위 디렉토리까지 전부 검색
grep -r "ERROR" /var/log/
grep -r "worker_processes" /etc/nginx/
```

## -n — 줄 번호 표시

```bash
grep -n "ERROR" app.log
# 15:ERROR: connection failed
# 42:ERROR: timeout
# ↑ 줄 번호가 앞에 표시됨
```

## -v — 패턴 제외 (반전)

```bash
# ERROR 가 없는 줄만
grep -v "ERROR" app.log

# grep 자체 프로세스 제외
ps aux | grep python | grep -v grep
```

## -c — 매칭 줄 수만

```bash
grep -c "ERROR" app.log
# 15  ← 에러가 15줄 있음
```

## -l — 파일명만 출력

```bash
# 패턴이 있는 파일 이름만
grep -rl "ERROR" /var/log/
# /var/log/app.log
# /var/log/nginx/error.log
```

## -E — 정규표현식 (여러 패턴 한 번에) ⭐️

```bash
# fail 또는 error 를 포함한 줄
grep -E 'fail|error' app.log

# 대소문자 무시 + 정규식
grep -iE 'fail|error|warn' app.log

# 실전: dmesg 에서 문제 징후 검색
sudo dmesg | grep -E 'fail|error' > boot_issue.txt
sudo dmesg | grep -iE 'fail|error|warn|critical'
```

## -A / -B / -C — 전후 줄 함께 보기

```bash
# 매칭 줄 + 이후 3줄
grep -A 3 "ERROR" app.log

# 매칭 줄 + 이전 3줄
grep -B 3 "ERROR" app.log

# 매칭 줄 + 전후 3줄
grep -C 3 "ERROR" app.log
```

```
에러 원인 파악 시 유용:
  에러 메시지 단독으로는 원인 불명
  -A 3 으로 이후 줄까지 보면 상세 원인 확인
```

---

---

# ③ 파이프 | — 명령어 연결 ⭐️

```
앞 명령어 출력 → 뒤 명령어 입력으로 전달
여러 명령어를 조합해서 강력한 처리
```

```bash
# 기본 조합 패턴
명령어1 | grep "패턴"
명령어1 | grep "패턴" | grep -v "제외"
명령어1 | grep "패턴" > 결과파일.txt

# 실전 예시
ps aux | grep python
ps aux | grep python | grep -v grep
dmesg | grep -iE 'error|fail' | tail -20
cat /etc/passwd | grep "labex"
ss -tulnp | grep ":8080"
```

---

---

## 에러 추출 + 보고서 생성 ⭐️

```bash
# 에러 추출 → 보고서 파일 생성
grep "ERROR" ~/project/logs/app.log > ~/project/error_report.txt

# Nginx 설정 항목 이어붙이기 (>> 로 기존 내용 유지)
grep "worker_processes" ~/project/config/nginx.conf >> ~/project/error_report.txt

# 전체 보고서 확인
cat ~/project/error_report.txt
```

```
> vs >> 차이:
  >  = 덮어쓰기 (기존 에러 로그 날아감 ⚠️)
  >> = 이어쓰기 (기존 내용 유지하며 추가)
  보고서 작성 시 항상 >> 사용
```

## 부팅 메시지 분석 — dmesg + grep ⭐️

```bash
# dmesg = 커널 링 버퍼 메시지 (부팅 + 하드웨어 메시지)
sudo dmesg | grep -E 'fail|error' > ~/project/boot_issue.txt

# -E 로 여러 패턴 동시 검색
sudo dmesg | grep -iE 'fail|error|warn'
# -i = 대소문자 무시 / -E = OR 패턴

# 부팅 메시지에서 특정 장치 관련 에러
sudo dmesg | grep -i "usb"
sudo dmesg | grep -i "disk"
```

## 에러 추출 → 파일 저장

```bash
# 앱 로그에서 ERROR 줄만 추출
grep "ERROR" ~/project/logs/app.log > ~/project/error_report.txt

# 에러 + 경고 함께
grep -E "ERROR|WARN" app.log > issues.txt

# 날짜 범위 필터 (날짜 문자열 포함)
grep "2026-04-20" app.log | grep "ERROR"
```

## 설정 파일 검색

```bash
# nginx 설정에서 worker_processes 찾기
grep "worker_processes" /etc/nginx/nginx.conf

# 보고서 파일에 추가
grep "worker_processes" ~/project/config/nginx.conf >> ~/project/error_report.txt
```

## 여러 결과 하나의 보고서로

```bash
# 보고서 초기화
> system_report.txt

# 각 분석 결과 추가 (>> 로 이어쓰기)
grep "ERROR" app.log >> system_report.txt
grep "worker_processes" nginx.conf >> system_report.txt
sudo dmesg | grep -E 'fail|error' >> system_report.txt

# 최종 확인
cat system_report.txt
```

## dmesg 부팅 메시지 분석

```bash
# 부팅 메시지에서 에러 찾기
sudo dmesg | grep -iE 'fail|error'

# 파일로 저장
sudo dmesg | grep -E 'fail|error' > boot_issue.txt

# 권한 없으면 sudo 필요
# "Operation not permitted" → sudo dmesg 로 실행
```

---

---

# ⑤ 자주 쓰는 조합

```bash
# 실행 중인 프로세스에서 python 찾기 (grep 자체 제외)
ps aux | grep python | grep -v grep

# 로그에서 에러만 뽑아서 줄 수 세기
grep "ERROR" app.log | wc -l

# 에러 종류별 집계
grep "ERROR" app.log | sort | uniq -c | sort -rn

# 오늘 날짜 에러만
grep "$(date +%Y-%m-%d)" app.log | grep "ERROR"

# 설정 파일에서 주석 제외하고 보기
grep -v "^#" /etc/nginx/nginx.conf | grep -v "^$"
```

---

---

# 옵션 한눈에

|옵션|의미|예시|
|---|---|---|
|`-i`|대소문자 무시|`grep -i "error"`|
|`-r`|재귀 검색|`grep -r "error" /var/log/`|
|`-n`|줄 번호 표시|`grep -n "error" file`|
|`-v`|패턴 제외|`grep -v "DEBUG"`|
|`-c`|매칭 줄 수|`grep -c "ERROR"`|
|`-l`|파일명만|`grep -rl "error" /dir/`|
|`-E`|정규식 (여러 패턴)|`grep -E 'fail\|error'`|
|`-A N`|이후 N줄|`grep -A 3 "ERROR"`|
|`-B N`|이전 N줄|`grep -B 3 "ERROR"`|
|`-C N`|전후 N줄|`grep -C 3 "ERROR"`|

---

---

# 자주 하는 실수

| 실수                  | 원인          | 해결                   |
| ------------------- | ----------- | -------------------- |
| 대소문자 안 잡힘           | 기본은 대소문자 구분 | `-i` 옵션 추가           |
| grep 결과에 grep 자체 포함 | grep 도 프로세스 | `\| grep -v grep` 추가 |
| `\|` 로 여러 패턴이 안 됨   | 기본 grep     | `-E` 옵션 필요           |
| dmesg 권한 에러         | root 필요     | `sudo dmesg`         |
| `>` 로 기존 보고서 날림     | 덮어쓰기        | `>>` 로 이어쓰기          |