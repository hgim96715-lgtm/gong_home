---
aliases:
  - 리다이렉션
  - tee
  - stdout
  - stdin
  - stderr
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Commands]]"
  - "[[Linux_Grep]]"
  - "[[Linux_Awk]]"
  - "[[Linux_Basic_Commands]]"
---

# Linux_Redirection — 표준 스트림 & 리다이렉션

## 한 줄 요약

```
리다이렉션 = 명령어의 입력/출력 방향을 바꾸는 것
기본: 키보드 입력 → 명령어 → 화면 출력
리다이렉션: 파일 입력 → 명령어 → 파일 출력
```

---

---

# ① 표준 스트림 3가지 ⭐️

```
모든 명령어는 3개의 스트림을 가짐

stdin  (표준 입력)  파일 디스크립터 0  → 키보드 (기본)
stdout (표준 출력)  파일 디스크립터 1  → 화면 (기본)
stderr (표준 에러)  파일 디스크립터 2  → 화면 (기본)

정상 출력 → stdout
에러 메시지 → stderr  ← 별도 스트림! > 만으로는 저장 안 됨
```

---

---

# ② 출력 리다이렉션

## > — 덮어쓰기 (stdout)

```bash
echo "Hello from LabEx" > hello.txt   # 파일 생성 + 쓰기
ls -l > file_list.txt                 # ls 결과를 파일로

# ⚠️ 기존 내용 경고 없이 전부 삭제
echo "새 내용" > file_list.txt        # 기존 내용 사라짐!
```

```
> 는 1> 의 줄임 표현 (stdout = 파일 디스크립터 1)
파일 없으면 새로 생성
파일 있으면 무조건 덮어씀 → 주의!
```

## >> — 추가 (append)

```bash
echo "첫 번째 줄" > greetings.txt
echo "두 번째 줄" >> greetings.txt    # 기존 내용 유지 + 뒤에 추가

date > activity.log
date >> activity.log                  # 타임스탬프 계속 추가
cat activity.log
# Sat Apr 5 08:35:34 AM CST 2026
# Sat Apr 5 08:35:42 AM CST 2026
```

```
로그 파일 만들 때 필수 패턴
>> 는 파일 없으면 생성 / 있으면 뒤에 추가
```

---

---

# ③ 에러 리다이렉션

## 2> — stderr 덮어쓰기

```bash
ls non_existent_file
# ls: cannot access 'non_existent_file': No such file or directory
# → 이 에러는 stderr 로 출력됨

# > 만 쓰면 에러는 저장 안 됨 (화면에만 출력)
ls non_existent_file > output.txt    # 에러는 여전히 화면에!

# 에러를 저장하려면 2>
ls non_existent_file 2> error.log
cat error.log
# ls: cannot access 'non_existent_file': No such file or directory
```

## 2>> — stderr 추가

```bash
ls fake_file1 2> error.log
ls fake_file2 2>> error.log          # 에러 누적
cat error.log
# ls: cannot access 'fake_file1': No such file or directory
# ls: cannot access 'fake_file2': No such file or directory
```

---

---

# ④ stdout + stderr 동시 저장 ⭐️

## &> — 둘 다 한 번에 (bash 전용)

```bash
ls /etc/passwd non_existent_file &> combined.log
cat combined.log
# ls: cannot access 'non_existent_file': No such file or directory
# -rw-r--r-- 1 root root 1916 /etc/passwd
```

## > file 2>&1 — 전통적인 방법 (POSIX 호환)

```bash
ls /etc/passwd non_existent_file > combined.log 2>&1
```

```
2>&1 의 의미:
  2>   stderr 를 리다이렉션
  &1   파일 디스크립터 1 (stdout) 과 같은 곳으로

  → stderr 를 stdout 이 가는 곳으로 합침
  → 둘 다 combined.log 에 저장됨

순서 중요:
  command > file 2>&1   ✅  stdout → file / stderr → stdout 과 같은 곳(file)
  command 2>&1 > file   ❌  stderr → 화면(현재 stdout) / stdout → file (의도와 다름!)
```

## stdout vs stderr 분리 저장

```bash
ls /etc/passwd non_existent_file > output.txt 2> error.txt

cat output.txt   # 정상 출력만
cat error.txt    # 에러만
```

---

---

# ⑤ /dev/null — 버리기

```bash
# 에러 무시 (버리기)
ls non_existent_file 2> /dev/null

# 출력 + 에러 전부 버리기
command &> /dev/null

# stdout 만 버리고 에러는 화면에
command > /dev/null
```

```
/dev/null = 블랙홀
  여기로 보낸 데이터는 전부 사라짐
  크론탭 / 백그라운드 작업에서 출력 억제할 때 사용
```

---

---

# ⑥ 입력 리다이렉션 — <

```bash
# 기본: 키보드에서 입력
sort

# 파일에서 입력
echo "banana" > items.txt
echo "apple" >> items.txt
echo "cherry" >> items.txt

sort < items.txt    # 파일 내용을 입력으로
# apple
# banana
# cherry

cat < items.txt     # cat 에 파일 입력 (cat items.txt 와 동일)
```

```
< 는 입력 방향을 파일로 변경
대부분 명령어가 파일명을 인자로 받으니 실제로 많이 쓰진 않음
heredoc (<<) 에서 활용 많음
```

---

---

# ⑦ tee — 출력 분할 (화면 + 파일 동시에)

```bash
ls /etc/ | tee etc_listing.txt
# 화면에도 출력 + etc_listing.txt 에도 저장

# -a 옵션: 추가 (>> 처럼)
command | tee -a logfile.txt
```

```
| > 는 파일에만 저장 (화면 안 보임)
| tee 는 화면 출력 + 파일 저장 동시에

용도:
  명령어 결과를 보면서 동시에 파일로 저장
  파이프라인 중간에서 중간 결과 저장
```

```bash
# 파이프라인 중간 저장
cat /etc/passwd | tee raw.txt | grep root | tee root.txt
#                   ↑                          ↑
#              전체 저장                    grep 결과 저장
```

---

---

# ⑧ 리다이렉션 전체 정리

| 연산자                 | 방향                    | 의미          |
| ------------------- | --------------------- | ----------- |
| `>`                 | stdout → 파일           | 덮어쓰기        |
| `>>`                | stdout → 파일           | 추가          |
| `2>`                | stderr → 파일           | 에러 덮어쓰기     |
| `2>>`               | stderr → 파일           | 에러 추가       |
| `&>`                | stdout+stderr → 파일    | 둘 다 덮어쓰기    |
| `> file 2>&1`       | stdout+stderr → 파일    | 둘 다 (POSIX) |
| `<`                 | 파일 → stdin            | 입력 리다이렉션    |
| <code>\|</code>     | stdout → 다음 명령어 stdin | 파이프         |
| <code>\| tee</code> | stdout → 화면 + 파일      | 출력 분할       |
| `> /dev/null`       | stdout → 버림           | 출력 무시       |

---

---

# ⑨ 실전 패턴

```bash
# 로그 파일 만들기
echo "$(date): 작업 시작" >> pipeline.log
command >> pipeline.log 2>&1
echo "$(date): 작업 완료" >> pipeline.log

# 에러만 따로 모으기
for f in *.csv; do
    python3 process.py "$f" 2>> errors.log
done

# 출력 보면서 파일 저장
dbt run | tee dbt_run.log

# 에러 무시하고 실행
command 2> /dev/null

# stdout / stderr 분리
command > output.log 2> error.log
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|에러가 파일에 저장 안 됨|`>` 만 사용 (stdout만)|`2>` 또는 `&>` 사용|
|기존 파일 내용 날아감|`>` 덮어쓰기|`>>` 추가 사용|
|`2>&1` 순서 실수|`2>&1 > file` 순서 잘못|`> file 2>&1` 순서 지키기|
|파이프 결과가 화면에 안 보임|`\|` 로 다음 명령어로 넘어감|`\| tee` 로 화면+파일 동시|
|문자열만 쓰면 에러|명령어로 인식|`echo "문자열" >` 으로 사용|