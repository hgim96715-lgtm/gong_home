---
aliases:
  - awk
  - sed
  - 텍스트가공
  - 데이터 전처리
  - 스트림에디터
tags:
  - Linux
  - Shell
related:
  - "[[Data_Statistics]]"
  - "[[Filtering_Text]]"
  - "[[00_Linux_HomePage]]"
  - "[[File_Management]]"
  - "[[Text_Transformation_tr]]"
---
## 개념 한 줄 요약

**"터미널 세상의 '엑셀(Excel)'과 '워드(Word)'다."**
* **`awk` (엑셀):** 데이터를 표(행과 열)로 인식해서, 특정 **열(Column)** 을 뽑거나 계산할 때 쓴다.
* **`sed` (워드):** 문서 전체에서 특정 단어를 **찾아 바꾸기(Find & Replace)** 하거나 삭제할 때 쓴다.

---
## 왜 필요한가? (Why)

**문제점:**
- 로그 파일에서 "IP 주소만 싹 긁어오고 싶은데" 텍스트 덩어리라서 복사가 안 된다. (`awk` 필요)
- 설정 파일(`config.yaml`)에 있는 `localhost`를 `192.168.0.1`로 바꿔야 하는데, 파일이 100개다. 하나씩 열어서 수정할 텐가? (`sed` 필요)

**해결책:**
- **`awk`** 로 공백이나 쉼표로 구분된 데이터의 특정 칸(Column)만 쏙쏙 뽑아낸다.
- **`sed`** 로 파일 열지 않고 명령어 한 줄로 100개 파일의 내용을 순식간에 교체한다.

---
##  실무 적용 사례 (Practical Context)

1.  **프로세스 킬러 (`awk`):**
    - "지금 떠 있는 파이썬 프로그램의 **PID(프로세스 ID)** 만 뽑아줘."
    - `ps aux | grep python | awk '{print $2}'`

2.  **데이터 클리닝 (`sed`):**
    - "로그 파일에서 개인정보(전화번호)는 전부 `*** `로 마스킹 처리해."
    - `sed 's/[0-9]\{11\}/***/g' user.log`

---
##  Code Core Points: ① `awk` (데이터 분석가)

`awk`는 데이터를 **공백(스페이스)** 기준으로 잘라서 엑셀처럼 다룹니다.

###  기본 조회 (Select)

가장 많이 쓰는 기능입니다. 원하는 "칸"만 뽑아옵니다.

```bash
# 문법: $1(1열), $2(2열)... $0(전체 줄) 
# "첫 번째 칸이랑 두 번째 칸만 보여줘"
awk '{print $1, $2}' file.txt

# [꿀팁] 깔끔하게 포맷팅 (C언어 printf 문법)
awk '{printf "이름: %s, 나이: %d\n", $1, $2}' file.txt
```

### 구분자 다루기 (Delimiter) ⭐️ [중요]

기본적으로 `awk`는 공백을 기준으로 자르지만, 실무 데이터는 쉼표나 탭인 경우가 많습니다.

- **`FS` (Field Separator):** 입력 파일을 읽을 때 자르는 기준 (옵션: `-F` 또는 **`BEGIN {FS="\t"}`**)
- **`OFS` (Output Field Separator):** 결과를 출력할 때 사이에 끼워줄 문자 (변수: `OFS`)

```bash
# 1. 들어올 때 (Input): 쉼표(,)로 구분된 CSV 파일 읽기
awk -F, '{print $1}' data.csv

# 2. 들어올 때 (Input): 탭(Tab)으로 구분된 파일 읽기
awk -F'\t' '{print $1}' data.tsv

# 콜론(:)으로 구분된 파일 읽기 (예: /etc/passwd)
awk -F: '{print $1}' /etc/passwd

# 2. 내부 변수 (FS) 변경
# BEGIN 블록은 "파일 읽기 전"에 실행되므로 여기서 설정을 잡습니다.
awk 'BEGIN {FS="\t"} {print $1}' file.tsv

# 3. 나갈 때 (Output): 콤마를 찍어서 내보내기 (CSV 변환)
# "들어올 땐 공백이지만, 나갈 땐 쉼표로 이어줘"
# 주의: print $1, $2 처럼 쉼표(,)를 찍어야 OFS가 작동함
awk -v OFS="," '{print $1, $2}' file.txt
```

### 조건 필터링 (Where)

SQL의 `WHERE` 절과 똑같습니다.

```bash
# 1. 숫자 조건: "2번째 칸이 20보다 큰 줄만 보여줘"
awk '$2 > 20' file.txt

# 2. 문자 검색: "Admin이라는 글자가 있는 줄만 보여줘" (grep 대용)
awk '/Admin/ {print $0}' file.txt
```

### 통계 및 계산 (Aggregation) 

`awk`의 꽃입니다. 합계와 평균을 즉석에서 계산합니다.

> **💡 핵심 원리:**
> 1. **`{...}` (반복):** 매 줄마다 계산기를 두드립니다. (누적)
> 2. **`END {...}` (결과):** 파일을 다 읽은 후 **마지막에 딱 한 번** 결과를 출력합니다.

```bash
# 2번째 컬럼의 합계(Sum) 구하기
awk '{sum += $2} END {print sum}' data.txt

# 평균(Avg) 구하기 (합계 / 개수)
awk '{total += $2; count++} END {print "평균:", total/count}' data.txt
```

###  쉘 변수 전달하기 (`-v` 옵션) 

터미널의 변수(Shell Variable)는 `awk` 내부로 바로 못 들어갑니다.
**`-v` (Variable)** 옵션으로 통행증을 발급해줘야 합니다.

```bash
limit=50
# "바깥의 $limit 값을 안쪽의 lim 변수로 전달해라!"
awk -v lim="$limit" '$2 > lim {print $0}' data.txt
```

###  고급 데이터 가공 (Advanced)

로그 분석할 때 필수로 쓰는 **빈도 분석**과 **시간 변환**입니다.

```bash
# 1. [Group By] 빈도수 세기 (Word Cloud 원리)
# "1번째 칸(이름)이 각각 몇 번 나왔는지 세어줘."
# names라는 사전(Map)에 횟수를 저장했다가, 마지막(END)에 한꺼번에 출력함.
awk '{names[$1]++} END {for (name in names) print name, names[name]}' file1.txt

# 2. [Time] 유닉스 타임스탬프 변환
# 1672531200 같은 외계어 숫자를 -> "2023-01-01 09:00:00"으로 변환
# 로그 파일에 찍힌 시간을 사람이 읽을 수 있게 바꿀 때 필수!
awk '{print strftime("%Y-%m-%d %H:%M:%S", $1)}' timestamps.txt
```

### 문자열 가공 (String Manipulation) - `substr` 

데이터가 공백으로 예쁘게 안 잘려있고, **"글자 위치"로 잘라야 할 때** 사용합니다. (엑셀의 `MID` 함수와 동일)

> **⚠️ 주의 (함정 카드):** 파이썬이나 C언어는 숫자를 0부터 세지만, **`awk`는 사람처럼 1부터 셉니다.**
> - Python: `text[0]` (첫 번째 글자)
> - awk: `substr($0, 1, 1)` (첫 번째 글자)

- **문법**: `{scss}substr(대상, 시작위치, 가져올_개수)`

**상황:** 아래와 같은 주문 번호가 있다n. `ORD-2024-A01` (앞의 3글자는 주문 유형, 뒤의 4글자는 연도)

```bash
# 예제 데이터 생성
echo "ORD-2024-A01" > order.txt

# 1. 발견하신 기능: 3번째 글자부터 1개만 (결과: D)
awk '{print substr($0, 3, 1)}' order.txt

# 2. [응용] 연도만 추출하기: 5번째 글자부터 4개 (결과: 2024)
awk '{print substr($0, 5, 4)}' order.txt

# 3. [꿀팁] 개수 생략하면 끝까지: 5번째부터 끝까지 (결과: 2024-A01)
awk '{print substr($0, 5)}' order.txt

# 4. [고급] 조각 모음 (Concatenation) ⭐️
# "떨어져 있는 글자를 오려서 풀로 붙이기"
# awk는 함수 사이에 아무것도 안 넣으면(공백도 없이) 문자열을 합칩니다.

# 예제: 2번째 글자('R')와 7번째 글자('2')를 합쳐라 -> 결과: R2
awk '{print substr($0,2,1) substr($0,7,1)}' order.txt

# 비교: 콤마(,)를 넣으면 한 칸 띄어서 출력됩니다 -> 결과: R 2
awk '{print substr($0,2,1), substr($0,7,1)}' order.txt
```

### 반복문 (Loops) - `for`

`awk`는 기본적으로 줄(Row)을 자동으로 넘겨주지만, **한 줄 안에서 여러 칸(Column)을 훑어야 할 때** `for`문을 씁니다. 

- **`{bash}for (i=1; i<=NF; i++)`**: **"가로 본능"**. 한 줄에 있는 모든 데이터를 싹 훑을 때 사용
- 핵심 변수: `NF` (Number of Fields) : 현재 줄에 칸이 총 몇 개인지 알려줍니다. (예: `A B C` 면 `NF`는 3)

#### 컬럼 훑기 (C-Style)

가장 많이 쓰는 패턴입니다. `$1`, `$2`... 이렇게 하나씩 적기 힘들 때 사용합니다.

```bash
# 예제 데이터: 칸 개수가 들쑥날쑥한 파일
echo "사과 배 포도" > fruits.txt
echo "수박 딸기" >> fruits.txt

# 상황: "모든 칸을 하나씩 검사해서 출력하고 싶어!"
# 해석: i는 1부터 시작해서, 마지막 칸(NF)까지 1씩 증가하며 돕니다.
awk '{ 
    for (i = 1; i <= NF; i++) {
        printf "현재 칸($%d): %s\n", i, $i
    }
    print "---한 줄 끝---"
}' fruits.txt
```

#### 조건부 반복 (if와 결합)

"이번 줄에 있는 숫자 중에서 50점 넘는 것만 찾고 싶어" 같은 상황입니다.

```bash
echo "철수 40 60 30" > scores.txt
echo "영희 80 90 95" >> scores.txt

# 이름($1)을 제외하고, 점수($2부터)만 검사해서 50점 넘으면 출력
awk '{
    printf "%s의 합격 점수: ", $1
    for (i = 2; i <= NF; i++) {
        if ($i > 50) {
            printf "%s ", $i
        }
    }
    printf "\n"
}' scores.txt
```


#### 역순 출력 (Reverse)

데이터를 거꾸로 뒤집을 때도 `for`문을 씁니다.

```bash
echo "1 2 3 4 5" | awk '{
    for (i = NF; i >= 1; i--) {
        printf "%s ", $i
    }
    printf "\n"
}'
# 결과: 5 4 3 2 1
```
----
## Code Core Points: ② `sed` (텍스트 수정가)

- **문법:** `{bash}sed 's/찾을거/바꿀거/g' 파일명`

### A. 교체하기 (Substitution)

가장 기본이자 핵심 기능인 `s` (Substitute) 명령어다.

```bash
# 1. 첫 번째만 바꾸기
# 한 줄에 "hello"가 여러 개 있어도 맨 앞의 하나만 "world"로 바꿈
sed 's/hello/world/' file.txt

# 2. 싹 다 바꾸기 (Global) ⭐️
# 한 줄에 있는 모든 "hello"를 "world"로 바꿈 (가장 많이 씀)
sed 's/hello/world/g' file.txt

# 3. 대소문자 무시하고 바꾸기 (Ignore case)
# Apple, apple, APPLE 상관없이 전부 orange로 변경
sed 's/apple/orange/gI' file.txt
```

### B. 삭제하기 (Deletion)

원하는 조건의 줄을 아예 없애버린다. (`d` flag)

```bash
# 1. 특정 단어가 포함된 줄 삭제
# "delete me"라는 글자가 있는 줄은 통째로 날려버림
sed '/delete me/d' file.txt

# 2. 범위 삭제 (Line Range)
# 1번째 줄부터 3번째 줄까지 삭제
sed '1,3d' file.txt

# 3. 나머지 삭제 (Head 처럼 쓰기)
# 3번째 줄부터 끝($)까지 삭제 -> 즉, 1~2번째 줄만 남김
sed '3,$ d' file.txt
```

### C. 끼워넣기 (Append & Insert) 

설정 파일 중간에 새로운 설정을 추가할 때 필수다.

```bash
# 1. 뒤에 추가 (Append) - a
# "insert_here"가 있는 줄 바로 '다음 줄'에 "New Line"을 추가함
sed '/insert_here/a New Line' file.txt

# 2. 앞에 추가 (Insert) - i
# "insert_here"가 있는 줄 바로 '앞 줄'에 "New Line before"를 넣음
sed '/insert_here/i New Line before' file.txt
```

### D. 고급 옵션 (Advanced Flags)

```bash
# 1. 진짜 파일 수정하기 (-i) 🚨 주의!
# 결과를 화면에만 보여주는 게 아니라, 원본 파일을 덮어쓴다.
sed -i 's/apple/orange/g' file.txt

# 2. 바뀐 내용만 확인하기 (-n, p)
# -n (조용히 해) + p (바뀐 것만 출력해)
# 수백만 줄 로그에서 내가 건드린 부분만 확인할 때 유용
sed -n 's/apple/orange/p' file.txt

# 3. 여러 명령 한 번에 실행하기 (-e) ⭐️
# "apple을 orange로 바꾸고" + "delete me 있는 줄은 지워라"
sed -e 's/apple/orange/g' -e '/delete me/d' file.txt
```


---
## 심화: 내장 변수와 옵션 (Advanced)

`awk`가 미리 정의해둔 편리한 변수들입니다.

|**변수**|**의미 (Full Name)**|**설명**|
|---|---|---|
|**NR**|Number of Records|현재 **행 번호** (1, 2, 3...)|
|**NF**|Number of Fields|현재 줄의 **전체 칸 개수** (맨 마지막 칸은 `$NF`)|
|**FS**|Field Separator|**입력** 구분자 (기본값: 공백)|
|**OFS**|Output Field Separator|**출력** 구분자 (출력할 때 뭘로 나눌지)|

### 실무 꿀팁 예제

```bash
# 1. 줄 번호(NR) 같이 찍기
echo -e "데이터1\n데이터2" | awk '{print NR, $0}'
# 결과: 1 데이터1 ...

# 2. CSV 파일 처리하기 (-F 옵션) ️
# "이 파일은 쉼표(,)로 나뉘어 있어!"라고 알려줌
awk -F, '{print $1}' data.csv

# 3. 특수문자 출력 (echo -e)
# \n(엔터), \t(탭)을 문자가 아닌 기능으로 인식시킴
echo -e "첫줄\n둘째줄"
```

- echo -e에 대해 알고 싶다면 , [[File_Management#① `echo` 화면 출력 & 파일 생성의 마술사|echo -e ]] 참조 

---
## 실무 종합 예제 (Log Analysis)

**미션:** "서버 로그에서 사용자 이름만 뽑아내라."

```bash
# 로그 예시: [INFO] 2026-01-26 User:Gong Action:Login

# 단계 1. awk로 3번째 칸(User:Gong)을 잡는다.
cat server.log | awk '{print $3}'
# 결과: User:Gong

# 단계 2. sed로 "User:" 라는 글자를 삭제(빈카으로 교체)한다.
cat server.log | awk '{print $3}' | sed 's/User://g'
# 결과: Gong (깔끔!)
```

---
## 초보자가 자주 하는 실수 (Mistakes)

1. **"sed 썼는데 파일이 그대로예요!"**
    - `sed`는 기본적으로 "미리보기"만 제공합니다.
    - 진짜 저장하려면 **`-i` (In-place)** 옵션을 써야 합니다.
        
2. **"CSV 파일이 이상하게 잘려요!"**
    - `awk`는 공백 기준입니다. 
    - 쉼표(`,`)로 된 파일은 반드시 **`-F,`** 옵션을 붙이세요.
        
3. **"awk 변수가 안 먹혀요!"**
    - `awk '$2 > $limit'` (X) -> 작은따옴표 안에서는 `$` 변수 인식이 안 됩니다.
    - `awk -v lim="$limit" ...` (O) -> **`-v`** 옵션으로 넘겨주세요.



