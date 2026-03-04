---
aliases:
  - awk
  - sed
  - 텍스트가공
  - 데이터 전처리
  - 스트림에디터
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
# Linux Stream Editor — `awk` & `sed`

## 개념 한 줄 요약

> **"터미널 세상의 엑셀(Excel)과 워드(Word)다."**

|도구|별명|핵심 역할|
|---|---|---|
|`awk`|터미널 엑셀|데이터를 **행/열**로 인식 → 특정 **열(Column)** 추출·계산|
|`sed`|터미널 워드|문서 전체에서 **찾아 바꾸기(Find & Replace)** · 삭제|

---

## 왜 필요한가?

**`awk` 가 필요한 상황**

- 로그 파일에서 IP 주소 열만 긁어오고 싶은데, 덩어리 텍스트라 복사가 안 된다.

**`sed` 가 필요한 상황**

- 설정 파일 100개에 있는 `localhost`를 `192.168.0.1`로 바꿔야 한다. 하나씩 열 텐가?

---

## 실무 적용 사례

```bash
# awk 실무: 실행 중인 파이썬 프로세스의 PID만 뽑기
ps aux | grep python | awk '{print $2}'

# sed 실무: 로그 파일의 전화번호를 *** 로 마스킹
sed 's/[0-9]\{11\}/\*\*\*/g' user.log
```

---

## 내장 변수 요약표

|변수|Full Name|설명|
|---|---|---|
|`NR`|Number of Records|현재 **행 번호** (1, 2, 3...)|
|`NF`|Number of Fields|현재 줄의 **전체 칸 개수** (마지막 칸 = `$NF`)|
|`FS`|Field Separator|**입력** 구분자 (기본값: 공백)|
|`OFS`|Output Field Separator|**출력** 구분자|


---

# ① `awk` — 데이터 분석가

`awk`는 데이터를 **공백** 기준으로 잘라서 엑셀처럼 다룬다.

```
입력 줄:  Alice  30  Seoul
          ↓      ↓   ↓
         $1     $2   $3        $0 = 줄 전체
```

---

## A. 기본 조회 (Select)

```bash
# 원하는 열만 출력
awk '{print $1, $2}' file.txt

# C언어 스타일 포맷팅
awk '{printf "이름: %s, 나이: %d\n", $1, $2}' file.txt
```

---

## B. 구분자 다루기 ⭐️

기본은 공백이지만, 실무 데이터는 쉼표·탭인 경우가 많다.

```bash
# 쉼표(,) 구분 CSV 읽기
awk -F, '{print $1}' data.csv

# 탭(Tab) 구분 파일 읽기
awk -F'\t' '{print $1}' data.tsv

# 콜론(:) 구분 파일 (예: /etc/passwd)
awk -F: '{print $1}' /etc/passwd

# BEGIN 블록으로 구분자 설정 (파일 읽기 전에 실행됨)
awk 'BEGIN {FS="\t"} {print $1}' file.tsv

# 출력할 때 쉼표로 내보내기 (CSV 변환)
# ⚠️ print $1, $2 처럼 쉼표(,)를 찍어야 OFS가 작동함
awk -v OFS="," '{print $1, $2}' file.txt
```

---

## C. 공백 제거 트릭 — `$1=$1` ⭐️

### 언제 필요한가?

`uniq -c` 처럼 숫자를 **오른쪽 정렬**로 출력하는 명령어는 앞에 공백이 붙는다. ([[Linux_Data_Statistics#`uniq` 중복 제거기|uniq]] 참조)

```bash
echo -e "apple\napple\nbanana" | sort | uniq -c
#       2 apple    ← 앞에 공백이 있다!
#       1 banana
```

이 공백이 문제가 되는 상황:

- 채점 문제에서 `"2 apple"` 형식을 기대하는데 `" 2 apple"` 이 들어오면 **오답**
- 파이프라인에서 다음 명령어가 포맷 불일치로 오동작

### 해결법

```bash
sort data.txt | uniq -c | awk '{$1=$1; print}'
#       2 apple    →    2 apple   ✅ 공백 제거!
#       1 banana   →    1 banana
```

### 원리

```
$1=$1 을 실행하면 awk가 필드를 재조합(rebuild) 한다
  → 내부적으로: $1 + OFS + $2 + OFS + $3 ... 형태로 다시 이어붙임
  → OFS 기본값은 공백 1개
  → 이 과정에서 앞뒤 불필요한 공백이 자동으로 제거됨
```

|단계|값|
|---|---|
|`uniq -c` 출력|`" 2 apple"`|
|`$1=$1` 후|`"2"` + `" "` + `"apple"` = `"2 apple"`|
|최종 출력|`2 apple` ✅|

> **💡 핵심 개념:** `$1=$1` 자체는 값을 바꾸는 게 아니라, awk 가 **레코드를 재조합하도록 강제**하는 트릭이다. 
> 재조합 과정에서 OFS 기준으로 깔끔하게 다시 이어붙여지면서 공백이 사라진다.

---

## D. 조건 필터링 (WHERE)

SQL의 `WHERE` 절과 똑같다.

```bash
# 2번째 칸이 20보다 큰 줄만
awk '$2 > 20' file.txt

# "Admin" 글자가 있는 줄만 (grep 대용)
awk '/Admin/ {print $0}' file.txt
```

---

## E. 통계 및 계산 (Aggregation)

> **💡 핵심 원리**
> 
> - `{...}` — 매 줄마다 반복 실행 (누적)
> - `END {...}` — 파일을 다 읽은 후 **딱 한 번** 실행 (결과 출력)

```bash
# 2번째 컬럼 합계
awk '{sum += $2} END {print sum}' data.txt

# 2번째 컬럼 평균
awk '{total += $2; count++} END {print "평균:", total/count}' data.txt
```

---

## F. 쉘 변수 전달 (`-v` 옵션)

작은따옴표(`'`) 안에서는 쉘 변수 `$`가 인식되지 않는다. `-v` 로 통행증을 발급해야 한다.

```bash
limit=50

# ❌ 틀린 방법: 작은따옴표 안에서 $limit 인식 안 됨
awk '$2 > $limit {print}' data.txt

# ✅ 올바른 방법: -v 로 변수 전달
awk -v lim="$limit" '$2 > lim {print $0}' data.txt
```

---

## G. 고급 데이터 가공

```bash
# 빈도 분석 (Group By)
# 1번째 칸이 각각 몇 번 나왔는지
awk '{names[$1]++} END {for (name in names) print name, names[name]}' file.txt

# 유닉스 타임스탬프 → 사람이 읽는 시간으로 변환
awk '{print strftime("%Y-%m-%d %H:%M:%S", $1)}' timestamps.txt
```

---

## H. 문자열 자르기 — `substr`

데이터가 공백으로 안 잘리고 **글자 위치**로 잘라야 할 때 사용. (엑셀의 `MID` 함수)

> **⚠️ awk는 1부터 센다!** Python은 `text[0]`, awk는 `substr($0, 1, 1)`

**문법:** `substr(대상, 시작위치, 가져올_개수)`

```bash
# 예제 데이터: ORD-2024-A01
echo "ORD-2024-A01" > order.txt

# 5번째 글자부터 4개 → 결과: 2024
awk '{print substr($0, 5, 4)}' order.txt

# 5번째 글자부터 끝까지 → 결과: 2024-A01
awk '{print substr($0, 5)}' order.txt

# 두 조각 붙이기 (사이에 아무것도 없으면 바로 합쳐짐)
# 2번째('R') + 7번째('2') → 결과: R2
awk '{print substr($0,2,1) substr($0,7,1)}' order.txt

# 콤마(,)를 넣으면 공백 한 칸 추가 → 결과: R 2
awk '{print substr($0,2,1), substr($0,7,1)}' order.txt
```

---

## I. 반복문 — `for`

awk는 자동으로 줄(Row)을 넘겨주지만, **한 줄 안에서 여러 칸을 훑어야 할 때** `for` 를 쓴다.

```bash
# 한 줄의 모든 칸 출력 (가로 탐색)
echo "사과 배 포도" | awk '{
    for (i = 1; i <= NF; i++) {
        printf "칸[%d]: %s\n", i, $i
    }
}'

# 조건부: 50점 넘는 점수만 출력
echo "철수 40 60 30" > scores.txt
awk '{
    printf "%s의 합격 점수: ", $1
    for (i = 2; i <= NF; i++) {
        if ($i > 50) printf "%s ", $i
    }
    printf "\n"
}' scores.txt

# 역순 출력
echo "1 2 3 4 5" | awk '{
    for (i = NF; i >= 1; i--) printf "%s ", $i
    printf "\n"
}'
# 결과: 5 4 3 2 1
```

---

# ② `sed` — 텍스트 수정가

**기본 문법:** `sed 's/찾을거/바꿀거/g' 파일명`

---

## A. 교체 (Substitution)

```bash
# 첫 번째만 교체 (줄당 1개)
sed 's/hello/world/' file.txt

# 전체 교체 ⭐️ (줄에 있는 모든 hello)
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

# 3번째 줄~끝까지 삭제 (= 앞 2줄만 남김)
sed '3,$d' file.txt
```

---

## C. 끼워넣기 (Append & Insert)

```bash
# 특정 줄 다음에 추가 (Append)
sed '/insert_here/a New Line' file.txt

# 특정 줄 앞에 추가 (Insert)
sed '/insert_here/i New Line before' file.txt
```

---

## D. 특정 줄만 뽑기 (Printing)

`head`·`tail` 조합보다 타이핑도 짧고 빠르다.

**문법:** `sed -n '시작줄,끝줄p' 파일명`

```bash
# 12~22번째 줄만 출력
sed -n '12,22p' file.txt

# 정확히 100번째 줄만
sed -n '100p' file.txt

# 마지막 줄 (= tail -n 1)
sed -n '$p' file.txt

# 12번째부터 10줄 더 (= 12~22번째, 총 11줄)
sed -n '12,+10p' file.txt
```

> **`-n` + `p` 조합 원리**
> 
> - `-n` : "일단 아무것도 출력하지 마" (은신 모드)
> - `p` : "이 범위를 만나면 그때만 출력해"
> - 결합: "다 조용히 하고, 내가 시킨 구간만 말해" ✅

---

## E. 고급 옵션

```bash
# 원본 파일 직접 수정 (-i) 🚨 되돌릴 수 없음!
sed -i 's/apple/orange/g' file.txt

# 바뀐 내용만 확인 (-n + p)
sed -n 's/apple/orange/p' file.txt

# 여러 명령 한 번에 (-e) ⭐️
sed -e 's/apple/orange/g' -e '/delete me/d' file.txt
```

---

# 실무 종합 예제 — 로그 분석

**미션:** 서버 로그에서 사용자 이름만 뽑아라.

```bash
# 로그 형태: [INFO] 2026-01-26 User:Gong Action:Login

# 1단계: awk 로 3번째 칸(User:Gong) 추출
cat server.log | awk '{print $3}'
# → User:Gong

# 2단계: sed 로 "User:" 접두어 제거
cat server.log | awk '{print $3}' | sed 's/User://g'
# → Gong ✅
```

---

## 빈도 분석 파이프라인 (sort + uniq + awk)

```bash
# 각 항목이 몇 번 나왔는지 깔끔하게 출력
sort data.txt | uniq -c | awk '{$1=$1; print}'

# 빈도 내림차순 정렬까지
sort data.txt | uniq -c | sort -rn | awk '{$1=$1; print}'
```

---

# 초보자가 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`sed` 썼는데 파일이 그대로|기본은 미리보기만|`-i` 옵션 추가|
|CSV 파일이 이상하게 잘림|기본 구분자가 공백|`-F,` 옵션 추가|
|`awk` 변수가 안 먹힘|작은따옴표 안에서 `$` 인식 안 됨|`-v lim="$limit"` 로 전달|
|`uniq -c` 결과에 공백|숫자 오른쪽 정렬 포맷|`awk '{$1=$1; print}'` 로 제거|

---