---
aliases:
  - cat
  - less
  - head
  - tail
  - 텍스트 명령어
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Redirect]]"
  - "[[Linux_Log]]"
---

# Linux_Text_Commands — 파일 내용 보기

## 한 줄 요약

```
cat   = 파일 전체 출력 (짧은 파일)
less  = 페이지 단위로 보기 (긴 파일)
head  = 앞 N줄
tail  = 뒤 N줄 / tail -f 실시간 모니터링
```

---

---

# ① cat — 파일 전체 출력 ⭐️

```bash
# 기본 — 파일 내용 출력
cat file.txt
cat config.json
cat /etc/hosts

# 여러 파일 연결해서 출력
cat file1.txt file2.txt

# 줄 번호 포함
cat -n file.txt

# 빈 줄 제거해서 출력
cat -s file.txt     # squeeze: 연속 빈 줄 → 하나로

# 파일 내용 보면서 새 파일 만들기 (> 덮어쓰기)
cat file.txt > copy.txt

# 파일 내용 이어붙이기 (>>)
cat extra.txt >> main.txt
```

## cat 실전 패턴

```bash
# 보고서 내용 확인
cat system_report.txt
cat ~/project/error_report.txt
cat ~/project/config_diff.txt

# 설정 파일 내용 확인
cat /etc/hosts
cat /etc/passwd | head -5

# 로그 파일 (짧은 경우)
cat app.log
```

```
cat 을 쓸 때:
  파일이 짧을 때 → cat (전부 한 번에)
  파일이 길 때   → less (페이지 단위로)
  로그 끝만 볼 때 → tail
  에러 찾을 때   → cat + grep
```

---

---

# ② less — 페이지 단위로 보기 ⭐️

```bash
# 페이지 단위로 보기 (긴 파일에 필수)
less 파일명
less /var/log/syslog
less /etc/nginx/nginx.conf

# 파이프와 함께
ps aux | less
dmesg | less
```

## less 단축키

```
방향키 ↑↓  줄 이동
Space       한 페이지 아래
b           한 페이지 위 (back)
/검색어      앞으로 검색
?검색어      뒤로 검색
n           다음 검색 결과
N           이전 검색 결과
g           맨 처음으로
G           맨 끝으로
q           종료 ← 가장 중요!
```

```
⚠️ less 에서 나오려면 q 키
  모르면 갇힌 것처럼 느껴짐
```

---

---

# ③ head — 앞 N줄 보기

```bash
# 기본: 앞 10줄 (기본값)
head file.txt

# N줄 지정
head -n 20 file.txt    # 앞 20줄
head -5 file.txt       # 앞 5줄 (단축)
head -n 1 file.txt     # 첫 번째 줄만

# 실전 활용
head -5 /etc/passwd    # 계정 정보 상위 5개
head -10 error.log     # 로그 첫 10줄
cat big_file.csv | head -3   # CSV 헤더 확인
```

---

---

# ④ tail — 뒤 N줄 보기 ⭐️

```bash
# 기본: 뒤 10줄
tail file.txt

# N줄 지정
tail -n 20 file.txt
tail -20 file.txt      # 단축형
tail -n 1 file.txt     # 마지막 줄만

# 실전 활용
tail -20 /var/log/syslog     # 최근 로그 20줄
tail -50 app.log             # 최근 50줄
```

## tail -f — 실시간 모니터링 ⭐️

```bash
# -f : follow (파일에 내용 추가될 때마다 자동 출력)
tail -f /var/log/syslog
tail -f /opt/airflow/logs/scheduler/*.log
tail -f /var/log/nginx/access.log

# 실시간 + grep 조합
tail -f app.log | grep "ERROR"      # 에러만 실시간
tail -f app.log | grep -i "warn"    # 경고만
```

```
tail -f 는 Ctrl + C 로 종료
서버 로그 실시간 모니터링의 핵심 패턴
Airflow DAG 실행 로그 추적 시 매우 유용
```

---

---

# ⑤ wc — 줄/단어/바이트 수 세기

```bash
wc -l file.txt          # 줄 수만
wc -w file.txt          # 단어 수
wc -c file.txt          # 바이트 수

# 파이프와 조합
grep "ERROR" app.log | wc -l    # 에러 줄 수
cat /etc/passwd | wc -l         # 계정 수
ls /var/log/ | wc -l            # 로그 파일 수
```

---

---

# ⑥ 실전 패턴 — 로그 분석

```bash
# 로그 파일 확인 흐름
# 1. 파일 크기 확인
ls -lh /var/log/app.log

# 2. 최근 로그 확인
tail -50 /var/log/app.log

# 3. 에러 추출
grep "ERROR" /var/log/app.log

# 4. 실시간 모니터링
tail -f /var/log/app.log | grep "ERROR"

# 5. 에러 수 확인
grep "ERROR" /var/log/app.log | wc -l
```

## Airflow 로그 모니터링

```bash
# DAG 실행 로그 실시간 보기
tail -f ~/airflow/logs/dag_name/task_name/날짜/*.log

# 에러만 필터
tail -f ~/airflow/logs/scheduler/*.log | grep -iE "error|fail"
```

---

---

# 명령어 한눈에

|명령어|역할|언제|
|---|---|---|
|`cat 파일`|전체 출력|파일이 짧을 때|
|`less 파일`|페이지 보기|파일이 길 때|
|`head -n N 파일`|앞 N줄|파일 시작 확인|
|`tail -n N 파일`|뒤 N줄|최근 로그 확인|
|`tail -f 파일`|실시간 추적|로그 모니터링|
|`wc -l 파일`|줄 수|로그 에러 개수|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`less` 에서 못 나옴|q 키 모름|`q` 로 종료|
|`cat` 으로 큰 파일|화면 가득 쏟아짐|`less` 사용|
|`tail -f` 안 꺼짐|종료 방법 모름|`Ctrl + C`|
|head/tail 줄 수 헷갈림|기본값이 10줄|`-n N` 으로 명시|