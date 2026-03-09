---
aliases:
  - 배열
  - shell array
  - bash array
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Shell_Script]]"
  - "[[Shell_Parameter_Expansion]]"
---

# Shell_Arrays

## 개념 한 줄 요약

> **"여러 값을 하나의 변수에 순서대로 담는 쉘의 자료구조."** 반복 처리할 파일 목록, 서버 목록, 파이프라인 태스크 목록 등에 필수.

---

---

# ① 배열 선언

```bash
# 방법 1: 괄호로 한 번에 선언
fruits=("apple" "banana" "grape")

# 방법 2: 인덱스를 직접 지정
fruits[0]="apple"
fruits[1]="banana"
fruits[2]="grape"

# 방법 3: 명령어 결과를 배열로 받기
files=($(ls *.csv))
```

---

---

# ② 인덱스 접근

```bash
fruits=("apple" "banana" "grape")

echo ${fruits[0]}    # apple   <- 첫 번째
echo ${fruits[1]}    # banana  <- 두 번째
echo ${fruits[-1]}   # grape   <- 뒤에서 첫 번째 (bash 4.0+)
```

> 변수 이름만 쓰면 첫 번째 요소만 나온다. `echo $fruits` 는 `echo ${fruits[0]}` 과 동일.

---

---
# ② - 1 배열에 데이터 채워넣기 — readarray / while read

>파일이나 명령어 결과를 배열로 받을 때 쓰는 두 가지 방법.

## readarray — 한 방에 읽기 

```bash
# 파일의 각 줄을 배열 요소로 읽기
readarray countries < countries.txt

echo ${countries[@]}   # 전체 출력
echo ${countries[0]}   # 첫 번째 줄
```

### -t 옵션 — 줄바꿈(\n) 제거 ⭐️

>`readarray` 는 각 줄을 읽을 때 **줄 끝의 `\n` 까지 요소에 포함**시킨다.
> `-t` 를 붙이면 `\n` 을 제거해준다. **거의 항상 붙인다.**

```bash
# -t 없이 읽기
readarray countries < countries.txt
echo "${countries[0]}"   # "Korea\n"  <- \n 이 붙어있음

# -t 붙이고 읽기
readarray -t countries < countries.txt
echo "${countries[0]}"   # "Korea"    <- 깔끔
```

```text
정리:
readarray countries       <- 각 요소에 \n 포함  (비교/출력 시 문제 발생)
readarray -t countries    <- \n 제거            (항상 이걸 써라)
```

## while read — 읽으면서 가공할 때

```bash
#!/bin/bash
countries=()
while read country; do
    countries+=("$country")
done
echo ${countries[@]}
```

```bash
# 파일에서 읽기
countries=()
while read country; do
    countries+=("$country")
done < countries.txt

echo ${countries[@]}
```

```bash
# 명령어 결과에서 읽기
countries=()
while read country; do
    countries+=("$country")
done < <(cat countries.txt | sort)

echo ${countries[@]}
```

## 두 방법 비교

|구분|`readarray`|`while read`|
|---|---|---|
|**코드 길이**|짧음|길어짐|
|**읽으면서 가공**|불가|가능 (`if`, `grep` 등)|
|**추천 상황**|그냥 파일/결과를 배열로 담을 때|읽으면서 조건 처리할 때|


```bash
# 읽으면서 조건 처리 예시
servers=()
while read line; do
    if [[ "$line" == web* ]]; then       # web 으로 시작하는 것만
        servers+=("$line")
    fi
done < servers.txt

echo ${servers[@]}   # web-01 web-02
```


---
---
# ③ 전체 요소 — @ vs *

> 배열 전체를 다룰 때 가장 자주 쓰는 문법.

```bash
fruits=("apple" "banana" "grape")

echo ${fruits[@]}    # apple banana grape  <- 각 요소를 별개로
echo ${fruits[*]}    # apple banana grape  <- 전체를 하나의 문자열로
```

## @ 와 * 의 차이

```bash
fruits=("apple" "banana grape")  # 두 번째 요소에 공백 포함

# @ : 각 요소를 따옴표로 보호 (공백 있어도 안전)
for f in "${fruits[@]}"; do
    echo "$f"
done
# apple
# banana grape  <- 공백 포함된 요소가 하나로 유지됨 ✅

# * : 전체를 하나의 문자열로 합침
for f in "${fruits[*]}"; do
    echo "$f"
done
# apple banana grape  <- 공백이 구분자가 되어 뭉개짐 ❌
```

```
규칙:
배열 순회할 때는 항상 "${fruits[@]}" 를 써야 안전하다.
"${fruits[*]}" 는 전체를 하나의 문자열로 다룰 때만 사용.
```

---

---

# ④ 배열 길이 — ${#array[@]}

```bash
fruits=("apple" "banana" "grape")

echo ${#fruits[@]}   # 3  <- 요소 개수
echo ${#fruits[0]}   # 5  <- 0번 요소의 문자 길이 ("apple" = 5글자)
```

---

---

# ⑤ 요소 추가 / 삭제

```bash
fruits=("apple" "banana")

# 추가: += 사용
fruits+=("grape")
fruits+=("kiwi" "mango")  # 여러 개 한 번에
echo ${fruits[@]}  # apple banana grape kiwi mango

# 삭제: unset
unset fruits[1]             # 1번 인덱스 삭제
echo ${fruits[@]}           # apple grape kiwi mango
# 인덱스 번호는 재정렬 안 됨 (1번 자리가 빔)

# 전체 삭제
unset fruits
```

---

---

# ⑥ 슬라이싱 — ${array[@]:시작:길이}

>Python 의 `[3:7]` (시작:끝) 과 다르다. 
>쉘은 `[3:5]` (시작:길이) 다.

```bash
nums=(10 20 30 40 50 60 70 80)
#      0   1   2   3   4   5   6   7
```

```bash
# 인덱스 3부터 인덱스 7까지 (3,4,5,6,7 총 5개) 가져오기
echo ${nums[@]:3:5}    # 40 50 60 70 80

# 헷갈리는 포인트:
# Python:  nums[3:7]   <- 시작:끝   (3,4,5,6 → 끝 인덱스 미포함)
# Shell:   ${nums[@]:3:5} <- 시작:길이 (3부터 5개 → 3,4,5,6,7)
```

```bash
공식:
  길이 = 끝 인덱스 - 시작 인덱스 + 1
  3 부터 7 까지 (둘 다 포함) = 7 - 3 + 1 = 5  →  :3:5
```

```bash
echo ${nums[@]:1:3}    # 20 30 40  <- 1번부터 3개
echo ${nums[@]:2}      # 30 40 50 60 70 80  <- 2번부터 끝까지
echo ${nums[@]: -2}    # 70 80     <- 뒤에서 2개 (공백 주의!)
```

> `${nums[@]: -2}` 에서 `-2` 앞에 **공백이 반드시 있어야 한다.** 
> `${nums[@]:-2}` 는 기본값 문법으로 해석되어 오작동함.

---

---

# ⑦ for 루프와 함께 — 가장 많이 쓰는 패턴

```bash
servers=("web-01" "web-02" "db-01")

# 기본 순회
for server in "${servers[@]}"; do
    echo "Connecting to $server..."
done

# 인덱스와 값 동시에
for i in "${!servers[@]}"; do
    echo "$i: ${servers[$i]}"
done
# 0: web-01
# 1: web-02
# 2: db-01
```

---

---

# ⑧ 연관 배열 (Associative Array) — bash 4.0+

> 인덱스 대신 **문자열 키**를 사용하는 배열. Python 의 dict 와 같다.

```bash
# 반드시 declare -A 로 선언
declare -A config

config["host"]="localhost"
config["port"]="5432"
config["db"]="train_db"

echo ${config["host"]}   # localhost
echo ${config["port"]}   # 5432

# 전체 키 목록
echo ${!config[@]}       # host port db

# 전체 값 목록
echo ${config[@]}        # localhost 5432 train_db

# 순회
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done
# host = localhost
# port = 5432
# db = train_db
```

---

---

# ⑨ 실전 패턴 모음

```bash
# CSV 파일 목록 처리
files=(data_2024.csv data_2025.csv result.csv)
for f in "${files[@]}"; do
    echo "Processing $f..."
done

# 배열에서 특정 값 찾기
target="banana"
fruits=("apple" "banana" "grape")
for f in "${fruits[@]}"; do
    if [[ "$f" == "$target" ]]; then
        echo "Found: $f"
    fi
done

# 배열 길이로 반복 횟수 제어
tasks=("extract" "transform" "load")
echo "총 ${#tasks[@]}개 태스크 실행"
for task in "${tasks[@]}"; do
    echo "Running: $task"
done
```

---

---

# 전체 치트시트

|문법|설명|
|---|---|
|`arr=("a" "b" "c")`|배열 선언|
|`${arr[0]}`|인덱스 접근|
|`${arr[-1]}`|마지막 요소|
|`${arr[@]}`|전체 요소 (각각 별개)|
|`${arr[*]}`|전체 요소 (하나의 문자열)|
|`${#arr[@]}`|배열 길이 (요소 개수)|
|`${#arr[0]}`|특정 요소의 문자 길이|
|`${arr[@]:1:3}`|슬라이싱 (1번부터 3개)|
|`arr+=("d")`|요소 추가|
|`unset arr[1]`|요소 삭제|
|`unset arr`|배열 전체 삭제|
|`${!arr[@]}`|인덱스 목록 (연관 배열 키)|
|`declare -A arr`|연관 배열 선언|

```
핵심 규칙:
순회할 때는 항상 "${arr[@]}"  <- 따옴표 + @ 세트
```