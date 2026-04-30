---
aliases:
  - 리다이렉션
  - 표준 입출력
  - stderr
  - 파이프
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_System_Info]]"
  - "[[Linux_Text_Commands]]"
  - "[[Linux_Search_Filter]]"
  - "[[Linux_Diff]]"
---

# Linux_Redirect — 입출력 리다이렉션

## 한 줄 요약

```
>   = 덮어쓰기 (기존 내용 삭제)
>>  = 이어쓰기 (기존 내용 뒤에 추가)
<   = 파일을 입력으로
|   = 앞 명령어 출력을 뒤 명령어 입력으로
2>  = 에러만 저장
```

---

---

# ① 표준 입출력 개념

```
리눅스의 3가지 스트림:
  stdin  (0) = 표준 입력  (키보드 → 프로그램)
  stdout (1) = 표준 출력  (프로그램 → 화면)
  stderr (2) = 표준 에러  (에러 메시지 → 화면)

리다이렉션 = 이 스트림의 방향을 바꾸는 것
  화면 대신 파일로 / 키보드 대신 파일에서
```

---

---

# ② > — 덮어쓰기 (Overwrite)

```bash
# 명령어 결과를 파일에 저장 (기존 내용 삭제!)
echo "hello" > output.txt
ls -l > file_list.txt
date > timestamp.txt

# 파일 내용 확인
cat output.txt   # hello
```

```
⚠️ > 는 파일을 먼저 비우고 씀
  이미 내용 있는 파일에 > 사용 → 기존 내용 전부 삭제
  주의해서 사용
```

---

---

# ③ >> — 이어쓰기 (Append) ⭐️

```bash
# 기존 파일 내용 뒤에 추가
echo "line 1" > log.txt       # 파일 생성
echo "line 2" >> log.txt      # 이어붙임
echo "line 3" >> log.txt      # 이어붙임

cat log.txt
# line 1
# line 2
# line 3
```

## 보고서 자동 생성 패턴 ⭐️

```bash
# 시스템 상태 보고서 생성
touch system_report.txt            # 빈 파일 생성

whoami   >> system_report.txt      # 사용자 추가
uname -a >> system_report.txt      # 커널 정보 추가
uptime   >> system_report.txt      # 가동 시간 추가
date     >> system_report.txt      # 현재 시각 추가
df -h    >> system_report.txt      # 디스크 사용량 추가

cat system_report.txt              # 전체 확인
```

```
> vs >> 핵심 차이:
  >  한 번만 사용 → 파일 생성 / 덮어쓰기
  >> 계속 추가    → 로그 누적 / 보고서 작성

  보고서 / 로그 파일: >> 사용
  매번 새로 쓰는 상태 파일: > 사용
```

---

---

# ④ 2> — 에러 리다이렉션

```bash
# 에러 메시지만 파일에 저장
ls /없는경로 2> error.log
# 화면에는 아무것도 안 나옴 / error.log 에 에러 저장

# 에러 파일 확인
cat error.log
# ls: cannot access '/없는경로': No such file or directory

# 에러 추가 모드
ls /없는경로 2>> error.log
```

---

---

# ⑤ 2>&1 — 표준출력 + 에러 동시에 ⭐️

```bash
# stdout + stderr 모두 같은 파일에 저장
command >> all.log 2>&1

# 예시: 크론탭 로그에서 자주 씀
python3 script.py >> /var/log/myapp.log 2>&1
#                    ↑ 일반 출력          ↑ 에러도 같이

# /dev/null 로 버리기 (출력 무시)
noisy_command > /dev/null 2>&1   # 모든 출력 버림
noisy_command 2> /dev/null       # 에러만 버림
```

```
2>&1 읽는 법:
  2  = stderr
  >  = 리다이렉션
  &1 = stdout 과 같은 곳으로

  순서 주의:
  >> log.txt 2>&1   ✅ stdout → 파일, stderr → stdout과 같은 파일
  2>&1 >> log.txt   ❌ 의도와 다름
```

---

---

# ⑥ < — 입력 리다이렉션

```bash
# 파일을 명령어의 입력으로
sort < unsorted.txt        # 파일 내용을 sort 에 전달
wc -l < data.txt          # 줄 수 세기
mysql -u root < dump.sql  # SQL 파일 실행
```

---

---

# ⑦ | — 파이프 (Pipe) ⭐️

```
앞 명령어의 출력 → 뒤 명령어의 입력
명령어를 조합해서 강력한 처리
```

```bash
# 기본 파이프
ls -l | grep ".py"           # .py 파일만 필터
ps aux | grep python         # python 프로세스만
cat log.txt | sort | uniq    # 정렬 후 중복 제거

# 여러 개 연결
cat /etc/passwd | cut -d: -f1 | sort   # 사용자 목록 정렬

# 파이프 + 파일 저장
ps aux | grep python > python_procs.txt

# less 로 페이지 단위 보기
ls -la | less
dmesg | less
```

## tee — 화면 + 파일 동시

```bash
# 화면에 출력하면서 파일에도 저장
ls -l | tee output.txt

# 이어쓰기 모드
ls -l | tee -a output.txt

# sudo 명령 결과를 시스템 파일에 저장
echo "설정값" | sudo tee /etc/설정파일
echo "module" | sudo tee -a /etc/modules-load.d/module.conf
```

---

---

# ⑧ 자주 쓰는 조합 패턴

```bash
# 특정 에러만 로그에 추가
command 2>> error.log

# 크론탭 로그 패턴
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1

# 실시간 로그 보기
tail -f /var/log/myapp.log

# 파이프로 필터링
cat /var/log/syslog | grep ERROR | tail -20

# wc 로 줄 수 세기
cat error.log | wc -l
grep ERROR /var/log/app.log | wc -l
```

---

---

# 리다이렉션 한눈에

|기호|의미|예시|
|---|---|---|
|`>`|덮어쓰기|`echo hi > file.txt`|
|`>>`|이어쓰기|`date >> log.txt`|
|`<`|파일을 입력으로|`sort < data.txt`|
|`2>`|에러만 저장|`cmd 2> err.log`|
|`2>&1`|에러를 stdout 으로|`cmd >> log.txt 2>&1`|
|`\|`|파이프|`ps aux \| grep python`|
|`tee`|화면 + 파일 동시|`cmd \| tee output.txt`|
|`/dev/null`|출력 버리기|`cmd > /dev/null 2>&1`|

---

---

# 자주 하는 실수

| 실수                  | 원인      | 해결                  |
| ------------------- | ------- | ------------------- |
| `>` 로 기존 파일 날림      | 덮어쓰기    | 로그는 항상 `>>`         |
| `2>&1 >> log` 순서 틀림 | 순서가 중요함 | `>> log 2>&1` 순서    |
| 파이프 없이 긴 출력         | 화면 넘침   | `\| less` 로 페이지 보기  |
| sudo 결과 파일 저장 안 됨   | 권한 문제   | `\| sudo tee -a 파일` |