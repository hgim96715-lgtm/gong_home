---
aliases:
  - 쉘스크립트
  - bash
  - shell_script
  - if문
  - for문
  - while문
  - 쉘변수
  - case문
tags:
  - Shell
  - Linux
related:
  - "[[Shell_Parameter_Expansion]]"
  - "[[Linux_Shell_Arrays]]"
---
# Linux_Shell_Script

## 개념 한 줄 요약

> **"리눅스 명령어들을 파일 하나에 모아서 순서대로 실행시키는 자동화 대본."**

```
문제: 매일 아침 명령어 10개를 매번 타이핑한다
해결: sh backup.sh 한 방으로 끝
     새벽 3시에 자고 있어도 Crontab 으로 자동 실행
```

> **핵심 규칙: Bash 는 공백이 문법의 생명이다.** 붙여야 할 때와 띄어야 할 때를 엄격히 구분한다.

---

---

# ① 기본 구조와 실행

## Shebang — 첫 줄에 반드시

```bash
#!/bin/bash
# 첫 줄: "나는 /bin/bash 로 실행하는 스크립트" 라는 선언
# # 으로 시작하면 주석

echo "Hello, Linux World!"
```

## 실행 방법

```bash
# 1. 실행 권한 부여 (처음 한 번만)
chmod +x my_script.sh

# 2. 실행
./my_script.sh

# 또는 bash 로 직접 실행 (권한 없어도 가능)
bash my_script.sh
```

---

---

# ② 변수

> **만들 땐 그냥, 쓸 땐 `$`**

```bash
# 변수 선언 (등호 앞뒤 공백 절대 금지!)
name="Alice"       # O
# name = "Alice"   # X  에러 발생!

# 변수 사용
echo "Hello, $name!"
echo "Hello, ${name}!"   # 중괄호: 변수 이름 경계 명확히 할 때
```

```bash
# 명령어 결과를 변수에 저장
hour=$(date +%H)
files=$(ls *.csv)
```

---

---

# ③ 조건문 — if / elif / else

> 대괄호 `[ ]` 안쪽 양옆에는 **무조건 공백** 필요.

```bash
#!/bin/bash

hour=$(date +%H)

if [ "$hour" -lt 12 ]; then
    echo "Good morning!"
elif [ "$hour" -lt 18 ]; then
    echo "Good afternoon!"
else
    echo "Good evening!"
fi   # if 를 거꾸로 쓴 것 = "여기서 끝"
```

## 비교 연산자 치트시트

### 숫자 비교

|기호|의미|예시|
|---|---|---|
|`-eq`|같다|`[ "$x" -eq "$y" ]`|
|`-ne`|다르다|`[ "$x" -ne "$y" ]`|
|`-gt`|크다|`[ "$x" -gt "$y" ]`|
|`-lt`|작다|`[ "$x" -lt "$y" ]`|
|`-ge`|크거나 같다|`[ "$x" -ge "$y" ]`|
|`-le`|작거나 같다|`[ "$x" -le "$y" ]`|

### 문자열 비교

|기호|의미|예시|
|---|---|---|
|`=`|같다|`[ "$str" = "yes" ]`|
|`!=`|다르다|`[ "$str" != "no" ]`|
|`-z`|비어있다|`[ -z "$var" ]`|

### 파일 조건

|기호|의미|예시|
|---|---|---|
|`-f`|파일이 존재|`[ -f "a.txt" ]`|
|`-d`|폴더가 존재|`[ -d "bin" ]`|

### 복합 조건

|기호|의미|예시|
|---|---|---|
|`-a`|AND|`[ "$x" -gt 10 -a "$x" -lt 20 ]`|
|`-o`|OR|`[ "$x" -eq 1 -o "$x" -eq 10 ]`|
|`&&`|AND (이중 괄호)|`[[ "$x" -gt 10 && "$x" -lt 20 ]]`|
|`\|`|OR (이중 괄호)|`[[ "$x" -eq 1 \| "$x" -eq 10 ]]`|

---

---

# ④ 다중 선택 — case

> `if-elif-elif` 가 길어질 때 가독성이 훨씬 좋다. 사용자 입력(Yes/No), 메뉴 선택에 주로 사용.

```bash
echo "진행하시겠습니까? (y/n)"
read answer

case "$answer" in
    y|Y|yes|YES)           # | 로 여러 패턴 묶기 (OR 조건)
        echo "네, 진행합니다!"
        ;;                 # ;; 이 break 역할 (필수!)
    n|N|no|NO)
        echo "아니요, 멈춥니다."
        ;;
    *)                     # * = 나머지 전부 (else 역할)
        echo "잘못된 입력입니다."
        exit 1
        ;;
esac   # case 를 거꾸로 쓴 것 = "여기서 끝"
```

```bash
# 정규식 패턴 활용
case "$answer" in
    [yY]*)   echo "yes 계열";;   # y, Y, yes, Year, yep 전부 잡음
    [nN]*)   echo "no 계열";;
    *)       echo "기타";;
esac
```

---

---

# ⑤ 반복문 — for / while

## A. for — 리스트 반복

```bash
# 직접 목록 나열
for i in 1 2 3 4 5; do
    echo "Number: $i"
done
```

## B. 중괄호 확장 {시작..끝} ⭐️

> `{1..5}` 는 `1 2 3 4 5` 를 자동으로 생성한다. 매번 숫자를 직접 나열할 필요가 없다.

```bash
# 숫자 범위
for i in {1..5}; do
    echo "Number: $i"
done
# 1 2 3 4 5

# 증가값 지정: {시작..끝..증가값}
for i in {0..10..2}; do
    echo "$i"
done
# 0 2 4 6 8 10

# 내림차순
for i in {5..1}; do
    echo "$i"
done
# 5 4 3 2 1

# 알파벳도 가능
for c in {a..e}; do
    echo "$c"
done
# a b c d e
```

## C. C스타일 for — 숫자 카운팅

```bash
for ((i=1; i<=50; i++)); do
    echo "Count: $i"
done

# 감소
for ((i=10; i>=1; i--)); do
    echo "$i"
done
```

## D. while — 조건 반복

```bash
count=0
while [ "$count" -lt 5 ]; do
    echo "Count: $count"
    count=$((count + 1))
done
```

## E. 배열 반복 — readarray 활용 ⭐️

> `readarray -t` 로 배열을 채운 뒤 `echo "${arr[@]}"` 로 **반복문 없이** 전체를 한 번에 출력할 수 있다.

```bash
# 파일에서 읽어서 배열로
readarray -t countries < countries.txt

# 반복문 없이 전체 출력
echo "${countries[@]}"
# Korea Japan China  (한 줄로 전부 출력)

# 같은 배열을 여러 번 참조
echo "${countries[@]}" "${countries[@]}" "${countries[@]}"
# Korea Japan China Korea Japan China Korea Japan China

# 반복문으로 하나씩 처리할 때
for country in "${countries[@]}"; do
    echo "Processing: $country"
done
```

```bash
# echo "${arr[@]}" vs 반복문 비교
countries=("Korea" "Japan" "China")

# 그냥 출력만 할 때 -> echo 한 줄이면 충분
echo "${countries[@]}"

# 각 요소를 가공해야 할 때 -> 반복문 사용
for c in "${countries[@]}"; do
    echo "Country: $c"
done
```

---

---

# ⑥ 문법 구조 정리

```
if   ~ fi       (finish)
for  ~ do ~ done
while ~ do ~ done
case ~ esac     (case 거꾸로)
```

---

---

# 자주 하는 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`command not found` 에러|`name = "Alice"` 공백|`name="Alice"` 붙여쓰기|
|`missing ]` 에러|`if ["$a" -eq 1]` 공백 없음|`if [ "$a" -eq 1 ]` 안쪽 공백 필수|
|변수값 오작동|`$name` 따옴표 없음|`"$name"` 쌍따옴표로 감싸기|
|`{1..5}` 가 문자 그대로 출력|`sh` 로 실행|`bash` 로 실행 (`sh` 는 중괄호 확장 미지원)|
|`readarray` 요소에 `\n` 포함|`-t` 옵션 누락|`readarray -t arr < file`|

## ./script.sh vs source script.sh

```
./script.sh     자식 프로세스(새 창)에서 실행 -> 변수가 현재 쉘에 안 남음
source script.sh  현재 쉘에서 바로 실행     -> 환경 변수 적용할 때 사용
```