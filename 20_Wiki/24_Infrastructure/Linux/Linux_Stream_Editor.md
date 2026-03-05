---
aliases:
  - awk
  - sed
  - 텍스트가공
  - 데이터 전처리
  - 스트림에디터
  - 텍스트 처리
tags:
  - Linux
related:
  - "[[Linux_Data_Statistics]]"
  - "[[Linux_Filtering_Text]]"
  - "[[00_Linux_HomePage]]"
  - "[[File_Management]]"
  - "[[Linux_Text_Transformation_tr]]"
  - "[[File_Content_Viewing]]"
---

# Linux Stream Editor — `awk` · `sed`

## 개념 한 줄 요약

> **"터미널 세상의 엑셀(awk)과 워드(sed)다."**

|도구|별명|핵심 역할|
|---|---|---|
|`awk`|터미널 엑셀|데이터를 **행/열**로 인식 → 특정 **열(Column)** 추출·계산|
|`sed`|터미널 워드|문서 전체에서 **찾아 바꾸기** · 삭제|

---

## 왜 필요한가?

```bash
# awk: 로그에서 IP 주소 열만 추출
ps aux | grep python | awk '{print $2}'

# sed: 설정 파일 100개의 localhost → 192.168.0.1 일괄 교체
sed -i 's/localhost/192.168.0.1/g' *.conf
```

---

---

# ① awk — 데이터 분석가

> awk 는 데이터를 **공백** 기준으로 잘라서 엑셀처럼 다룬다.

```
입력 줄:  Alice  30  Seoul
           ↓      ↓    ↓
          $1     $2    $3       $0 = 줄 전체
```

## 내장 변수

|변수|Full Name|설명|
|---|---|---|
|`NR`|Number of Records|현재 **행 번호** (1, 2, 3...)|
|`NF`|Number of Fields|현재 줄의 **전체 칸 개수** (마지막 칸 = `$NF`)|
|`FS`|Field Separator|**입력** 구분자 (기본값: 공백)|
|`OFS`|Output Field Separator|**출력** 구분자|

---

## A. 기본 조회

```bash
# 원하는 열만 출력
awk '{print $1, $2}' file.txt

# C 언어 스타일 포맷팅
awk '{printf "이름: %s, 나이: %d\n", $1, $2}' file.txt
```

---

## B. 구분자 다루기 ⭐️

```bash
# 쉼표(,) — CSV
awk -F, '{print $1}' data.csv

# 탭(Tab) — TSV
awk -F'\t' '{print $1}' data.tsv

# 콜론(:) — /etc/passwd
awk -F: '{print $1}' /etc/passwd

# BEGIN 블록으로 구분자 설정 (파일 읽기 전 실행)
awk 'BEGIN {FS="\t"} {print $1}' file.tsv

# 출력 구분자 지정 (CSV 로 변환)
# ⚠️ print $1, $2 처럼 쉼표(,) 를 찍어야 OFS 가 작동함
awk -v OFS="," '{print $1, $2}' file.txt
```

---

## C. 공백 제거 트릭 — `$1=$1` ⭐️

`uniq -c` 처럼 숫자를 오른쪽 정렬로 출력하는 명령어는 앞에 공백이 붙는다.

```bash
echo -e "apple\napple\nbanana" | sort | uniq -c
#       2 apple    ← 앞에 공백!
#       1 banana
```

```bash
# 해결: $1=$1 로 레코드 재조합
sort data.txt | uniq -c | awk '{$1=$1; print}'
#       2 apple  →  2 apple  ✅
```

**원리:**

```
$1=$1 을 실행하면 awk 가 필드를 재조합(rebuild) 한다
→ $1 + OFS + $2 + OFS + $3 ... 형태로 다시 이어붙임
→ OFS 기본값은 공백 1개
→ 이 과정에서 앞뒤 불필요한 공백이 제거됨
```

|단계|값|
|---|---|
|`uniq -c` 출력|`" 2 apple"`|
|`$1=$1` 후|`"2"` + `" "` + `"apple"`|
|최종 출력|`2 apple` ✅|

---

## D. 조건 필터링

```bash
# 2번째 칸이 20 보다 큰 줄만
awk '$2 > 20' file.txt

# "Admin" 글자가 있는 줄만 (grep 대용)
awk '/Admin/ {print $0}' file.txt
```

---

## E. 통계 및 계산

> `{...}` — 매 줄마다 반복 실행 (누적) `END {...}` — 파일을 다 읽은 후 딱 한 번 실행 (결과 출력)

```bash
# 2번째 컬럼 합계
awk '{sum += $2} END {print sum}' data.txt

# 2번째 컬럼 평균
awk '{total += $2; count++} END {print "평균:", total/count}' data.txt
```

---

## F. 쉘 변수 전달 (`-v` 옵션)

작은따옴표(`'`) 안에서는 쉘 변수 `$` 가 인식되지 않는다.

```bash
limit=50

# ❌ 틀린 방법
awk '$2 > $limit {print}' data.txt

# ✅ 올바른 방법: -v 로 변수 전달
awk -v lim="$limit" '$2 > lim {print $0}' data.txt
```

---

## G. 고급 데이터 가공

```bash
# 빈도 분석 (Group By)
awk '{names[$1]++} END {for (name in names) print name, names[name]}' file.txt

# 유닉스 타임스탬프 → 사람이 읽는 시간으로 변환
awk '{print strftime("%Y-%m-%d %H:%M:%S", $1)}' timestamps.txt
```

---

## H. 반복문 — `for`

```bash
# 한 줄의 모든 칸 출력
echo "사과 배 포도" | awk '{
    for (i = 1; i <= NF; i++) {
        printf "칸[%d]: %s\n", i, $i
    }
}'

# 50점 넘는 점수만 출력
echo "철수 40 60 30" | awk '{
    printf "%s의 합격 점수: ", $1
    for (i = 2; i <= NF; i++) {
        if ($i > 50) printf "%s ", $i
    }
    printf "\n"
}'

# 역순 출력
echo "1 2 3 4 5" | awk '{
    for (i = NF; i >= 1; i--) printf "%s ", $i
    printf "\n"
}'
# 결과: 5 4 3 2 1
```

---

---

# ② awk 내장 문자열 함수

## tolower() / toupper() — 대소문자 변환

```bash
echo "Hello WORLD" | awk '{print tolower($0)}'   # hello world
echo "hello world" | awk '{print toupper($0)}'   # HELLO WORLD

# 특정 컬럼에만 적용
echo "Alice,Seoul,30" | awk -F',' '{print tolower($1), $2}'
# alice Seoul
```

**실무 — 대소문자 섞인 로그 통일 후 집계:**

```bash
cat access.log | awk '{print tolower($0)}' | sort | uniq -c | sort -nr
```

|방식|동작|
|---|---|
|`sort -f`|정렬 순서만 무시, 원본 데이터는 그대로 출력|
|`tolower()`|실제로 소문자로 변환해서 출력|

---

## length() — 문자열 길이

```bash
# 문법: length(문자열)  /  인자 없으면 $0 (줄 전체) 길이

echo "hello" | awk '{print length($0)}'   # 5

# 특정 컬럼 길이
echo "Alice,Seoul,30" | awk -F',' '{print $1, length($1)}'
# Alice 5

# 실무: 100글자 넘는 줄만 필터링 (비정상 로그 탐지)
awk 'length($0) > 100' access.log

# 인자 생략 시 줄 전체 길이
awk '{print length}' file.txt
```

---

## substr() — 부분 문자열 추출

> **⚠️ awk 인덱스는 1부터! (Python 의 0 과 다름)**

```bash
# 문법: substr(문자열, 시작위치, 길이)

echo "Hello World" | awk '{print substr($0, 1, 5)}'   # Hello
echo "Hello World" | awk '{print substr($0, 7)}'      # World (길이 생략 시 끝까지)
```

```
인덱스 시각화:
H  e  l  l  o     W  o  r  l  d
1  2  3  4  5  6  7  8  9  10 11

substr($0, 1, 5) → "Hello"
substr($0, 7)    → "World"
```

```bash
# 날짜 컬럼에서 연도만 추출 ("2026-03-05")
echo "2026-03-05,1000" | awk -F',' '{print substr($1, 1, 4), $2}'
# 2026 1000

# 두 조각 붙이기
echo "ORD-2024-A01" | awk '{print substr($0,2,1) substr($0,7,1)}'   # R2  (공백 없음)
echo "ORD-2024-A01" | awk '{print substr($0,2,1), substr($0,7,1)}'  # R 2 (공백 1칸)
```

---

## index() — 문자열 위치 찾기

```bash
# 문법: index(문자열, 찾을문자열)
# 처음 등장하는 위치(정수) 반환 / 없으면 0

echo "Hello World" | awk '{print index($0, "World")}'   # 7
echo "Hello World" | awk '{print index($0, "xyz")}'     # 0

# 특정 문자열 포함 행만 필터링 (grep 대용)
awk 'index($0, "ERROR") > 0' access.log

# index + substr 조합: 이메일에서 아이디 추출
echo "user@email.com" | awk '{at=index($0,"@"); print substr($0,1,at-1)}'
# user
```

---

## 문자열 함수 한눈에 보기

|함수|문법|반환값|예시 결과|
|---|---|---|---|
|`tolower(s)`|`tolower($0)`|소문자 문자열|`"HELLO"` → `"hello"`|
|`toupper(s)`|`toupper($0)`|대문자 문자열|`"hello"` → `"HELLO"`|
|`length(s)`|`length($0)`|정수 (글자 수)|`"hello"` → `5`|
|`substr(s,i,n)`|`substr($0,1,5)`|부분 문자열|`"Hello World"` → `"Hello"`|
|`index(s,t)`|`index($0,"abc")`|정수 (위치, 없으면 0)|`"Hello"` → `0`|

> ⚠️ **awk 인덱스는 1부터 시작.** Python · JavaScript 의 0 과 다르다.

---

---

# ③ awk 변수 — 선언 없이 쓴다 ⭐️

> **awk 는 변수 선언이 없다. 처음 값이 할당되는 순간 변수가 자동 생성된다.**

```bash
# 다른 언어라면:
# int count = 0;  (C)
# count = 0       (Python)

# awk 는 선언 없이 바로 사용
awk '{count++; print count}' file.txt
#     ↑ 자동으로 0 으로 초기화됨
```

## 자동 초기화 규칙

```
숫자 연산에 처음 쓰이면   → 0  으로 자동 초기화
문자열 연산에 처음 쓰이면 → "" 으로 자동 초기화
```

```bash
echo "hello" | awk '{x++; print x}'          # 1  (0에서 시작)
echo "hello" | awk '{s = s "world"; print s}' # world  ("" 에서 시작)
```

## 실무 활용

```bash
# 줄 수 세기 (wc -l 과 동일)
awk '{count++} END{print count}' file.txt

# 특정 컬럼 합산
awk -F',' '{sum += $2} END{print sum}' data.csv

# 조건부 카운트
awk '$3 > 100 {count++} END{print count}' data.txt
```

## BEGIN / END 블록

```bash
# BEGIN: 파일 읽기 전 실행
# END:   파일 다 읽고 나서 실행

awk 'BEGIN{sum=0} {sum += $1} END{print "합계:", sum}' data.txt
#    ↑ 명시적 초기화 (안 해도 되지만 의도를 명확히 할 때)
```

> **왜 선언이 없는가?** 
> awk 는 텍스트 처리 전용 스크립팅 언어라서 타입 시스템이 느슨하게 설계되어 있다. 변수는 숫자로도 문자열로도 동시에 동작하며 컨텍스트에 따라 자동 변환된다.

---

---

# ④ sed — 텍스트 수정가

**기본 문법:** `sed 's/찾을거/바꿀거/g' 파일명`

---

## A. 교체 (Substitution)

```bash
# 줄당 첫 번째만 교체
sed 's/hello/world/' file.txt

# 전체 교체 ⭐️
sed 's/hello/world/g' file.txt

# 대소문자 무시하고 교체 (Apple, APPLE, apple 모두)
sed 's/apple/orange/gI' file.txt
```

---

## B. 삭제 (Deletion)

```bash
# 특정 단어 포함된 줄 통째로 삭제
sed '/delete me/d' file.txt

# 1~3번째 줄 삭제
sed '1,3d' file.txt

# 3번째 줄~끝까지 삭제
sed '3,$d' file.txt
```

---

## C. 끼워넣기 (Append / Insert)

```bash
# 특정 줄 다음에 추가
sed '/insert_here/a New Line' file.txt

# 특정 줄 앞에 추가
sed '/insert_here/i New Line before' file.txt
```

---

## D. 특정 줄만 뽑기

**문법:** `sed -n '시작줄,끝줄p' 파일명`

```bash
sed -n '12,22p' file.txt    # 12~22번째 줄
sed -n '100p' file.txt      # 정확히 100번째 줄
sed -n '$p' file.txt        # 마지막 줄 (= tail -n 1)
sed -n '12,+10p' file.txt   # 12번째부터 10줄 더 (총 11줄)
```

> `-n` = "아무것도 출력하지 마" (은신 모드) `p` = "이 범위만 출력해"

---

## E. 고급 옵션

```bash
# 원본 파일 직접 수정 (-i) 🚨 되돌릴 수 없음!
sed -i 's/apple/orange/g' file.txt

# 바뀐 내용만 확인
sed -n 's/apple/orange/p' file.txt

# 여러 명령 한 번에 (-e) ⭐️
sed -e 's/apple/orange/g' -e '/delete me/d' file.txt
```

---

---

# ⑤ 실무 종합 예제 — 로그 분석

```bash
# 로그 형태: [INFO] 2026-01-26 User:Gong Action:Login

# 1단계: awk 로 3번째 칸 추출
cat server.log | awk '{print $3}'
# → User:Gong

# 2단계: sed 로 "User:" 접두어 제거
cat server.log | awk '{print $3}' | sed 's/User://g'
# → Gong ✅
```

## 최강 콤보 파이프라인

```bash
# 가장 많이 등장한 항목 Top 3 (공백 제거까지)
cat access.log | sort | uniq -c | sort -nr | head -3 | awk '{$1=$1; print}'
```

```
cat    → 파일 읽기
sort   → 같은 항목끼리 인접하게 정렬 (uniq 준비)
uniq -c → 중복 묶고 횟수 표시
sort -nr → 횟수 내림차순 재정렬
head -3  → 상위 3개만
awk      → 앞 공백 제거
```

---

---

# 초보자가 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`sed` 썼는데 파일이 그대로|기본은 미리보기만|`-i` 옵션 추가|
|CSV 파일이 이상하게 잘림|기본 구분자가 공백|`-F,` 옵션 추가|
|`awk` 변수가 안 먹힘|작은따옴표 안에서 `$` 인식 안 됨|`-v lim="$limit"` 로 전달|
|`uniq -c` 결과에 공백|숫자 오른쪽 정렬 포맷|`awk '{$1=$1; print}'` 추가|
|`substr` 인덱스가 이상함|Python 은 0부터, awk 는 1부터|시작 위치에 1 사용|

---
