---
aliases:
  - cat
  - head
  - tail
  - wc
  - sort
  - uniq
  - cut
  - paste
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Redirection]]"
  - "[[Linux_Grep]]"
  - "[[Linux_Awk]]"
  - "[[Linux_Sed]]"
---

# Linux_File_Commands — 파일 내용 보기 & 처리

## 한 줄 요약

```
파일을 읽고 / 자르고 / 합치고 / 정렬하는 리눅스 기본 도구
로그 분석 / 데이터 전처리에 매일 사용
```

---

---

# ① cat — 파일 내용 출력

```bash
cat file.txt              # 전체 출력
cat -n file.txt           # 줄 번호 포함
cat file1.txt file2.txt   # 두 파일 이어서 출력
cat file1.txt file2.txt > merged.txt  # 합쳐서 저장
```

---

---

# ② head / tail — 앞뒤 N줄

```bash
head file.txt         # 앞 10줄 (기본값)
head -n 5 file.txt    # 앞 5줄
head -n 20 file.txt   # 앞 20줄

tail file.txt         # 뒤 10줄 (기본값)
tail -n 5 file.txt    # 뒤 5줄
tail -f file.txt      # 실시간 추가되는 줄 계속 출력 ← 로그 모니터링
```

```bash
# 실전: 로그 실시간 확인
tail -f /var/log/syslog
tail -f airflow.log

# 앞 10줄 제외하고 나머지 출력
tail -n +11 file.txt
```

---

---

# ③ less / more — 페이지 단위로 보기

```bash
less file.txt    # 위아래 스크롤 가능 ← 권장
more file.txt    # 아래로만 스크롤

# less 안에서:
# j / k      아래 / 위
# G          맨 끝으로
# g          맨 처음으로
# /패턴      검색
# q          종료
```

---

---

# ④ wc — 줄 수 / 단어 수 / 바이트 수

```bash
wc file.txt          # 줄 수 / 단어 수 / 바이트 수 전부
wc -l file.txt       # 줄 수만 ← 가장 자주 씀
wc -w file.txt       # 단어 수만
wc -c file.txt       # 바이트 수만
```

```bash
# 실전: 전체 행 수 확인
wc -l data.csv

# 파이프와 조합
cat data.csv | wc -l
grep "ERROR" app.log | wc -l   # 에러 줄 수
```

---

---

# ⑤ sort — 정렬

```bash
sort file.txt          # 알파벳순 정렬
sort -r file.txt       # 역순
sort -n file.txt       # 숫자 순서로 정렬
sort -k 2 file.txt     # 2번째 컬럼 기준 정렬
sort -k 2 -n file.txt  # 2번째 컬럼 숫자 정렬
sort -t ',' -k 2 file.csv   # 구분자 쉼표, 2번째 컬럼 기준
sort -u file.txt       # 중복 제거 후 정렬 (uniq 효과)
```

```bash
# 실전: 가장 많이 등장하는 값 찾기
sort file.txt | uniq -c | sort -rn | head -10
```

---

---

# ⑥ uniq — 중복 처리

```
반드시 sort 후에 사용
인접한 중복만 제거하기 때문
```

```bash
sort file.txt | uniq          # 중복 제거
sort file.txt | uniq -c       # 중복 횟수 카운트 ← 자주 씀
sort file.txt | uniq -d       # 중복된 것만 출력
sort file.txt | uniq -u       # 중복 없는 것만 출력
```

```bash
# 실전: IP별 접속 횟수
awk '{print $1}' access.log | sort | uniq -c | sort -rn
```

---

---

# ⑦ cut — 특정 컬럼 추출

```bash
# -f : 필드 번호 / -d : 구분자

cut -f 1 file.txt           # 탭 구분, 1번째 컬럼
cut -f 1,3 file.txt         # 1, 3번째 컬럼
cut -d ',' -f 2 file.csv    # 쉼표 구분, 2번째 컬럼
cut -d ':' -f 1 /etc/passwd # 콜론 구분, 1번째 (사용자명)

# 문자 위치로 자르기
cut -c 1-5 file.txt         # 1~5번째 문자
cut -c 3-   file.txt        # 3번째 이후 전부
```

```bash
# 실전: /etc/passwd 에서 사용자명만 추출
cut -d ':' -f 1 /etc/passwd

# CSV 에서 특정 컬럼만
cut -d ',' -f 1,3 data.csv
```

---

---

# ⑧ paste — 파일/줄 붙이기

```
cut 의 반대 개념
여러 파일을 열로 붙이거나
세로 데이터를 가로로 이어붙이기
```

## 여러 파일 열로 붙이기

```bash
# file1.txt   file2.txt
# A           1
# B           2
# C           3

paste file1.txt file2.txt
# A    1
# B    2
# C    3
# (탭으로 연결)

paste -d ',' file1.txt file2.txt
# A,1
# B,2
# C,3
```

## -s : 세로 → 가로 (직렬화)

```bash
# numbers.txt
# 1
# 2
# 3
# 4
# 5

paste -s numbers.txt
# 1    2    3    4    5   (탭으로 한 줄)

paste -s -d ',' numbers.txt
# 1,2,3,4,5

paste -s -d '\t' numbers.txt
# 1	2	3	4	5   (탭 구분)
```

```
-s 동작 원리:
  원래 paste = 여러 파일을 옆으로 붙이기
  -s (serial) = 한 파일의 줄들을 가로로 이어붙이기
  세로로 늘어진 데이터를 한 줄로 만들 때 사용

-d '\t' :
  \t = 탭 문자
  구분자를 탭으로 지정
  기본값도 탭이라 -s 만 써도 같은 결과
```

```bash
# 실전: 여러 줄 데이터를 한 줄로 만들기
echo -e "A\nB\nC\nD" | paste -s -d ','
# A,B,C,D

# awk 결과를 한 줄로
awk '{print $1}' data.txt | paste -s -d '\t'
```

---

---

# 명령어 조합 패턴

```bash
# 에러 로그에서 IP 주소 추출 → 정렬 → 중복 횟수 카운트
grep "ERROR" app.log | awk '{print $3}' | sort | uniq -c | sort -rn

# CSV 2번째 컬럼만 추출 → 중복 제거
cut -d ',' -f 2 data.csv | sort | uniq

# 파일 줄 수 확인 후 앞 10줄만 보기
wc -l data.txt && head -10 data.txt

# 여러 컬럼 파일을 탭으로 한 줄 합치기
cat ids.txt | paste -s -d '\t'
```

---

---

# 한눈에 정리

|명령어|역할|주요 옵션|
|---|---|---|
|`cat`|파일 출력 / 합치기|`-n` 줄번호|
|`head`|앞 N줄|`-n` 줄 수|
|`tail`|뒤 N줄 / 실시간|`-n` `-f`|
|`less`|페이지 스크롤|`/` 검색 `q` 종료|
|`wc`|줄/단어/바이트 수|`-l` `-w` `-c`|
|`sort`|정렬|`-r` `-n` `-k` `-t`|
|`uniq`|중복 처리|`-c` 횟수 `-d` 중복만|
|`cut`|컬럼 추출|`-d` 구분자 `-f` 필드|
|`paste`|파일/줄 붙이기|`-d` 구분자 `-s` 직렬화|