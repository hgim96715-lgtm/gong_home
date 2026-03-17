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
  - read
tags:
  - Shell
  - Linux
related:
  - "[[Shell_Parameter_Expansion]]"
  - "[[Linux_Shell_Arrays]]"
  - "[[00_Linux_HomePage]]"
---
# Linux_Shell_Script — 쉘 스크립트

## 한 줄 요약

```
리눅스 명령어들을 파일 하나에 모아서 순서대로 실행시키는 자동화 대본

문제: 매일 아침 명령어 10개를 매번 타이핑한다
해결: bash backup.sh 한 방으로 끝
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

```
만들 땐 그냥 대입
쓸 땐 $ 붙이기
```

```bash
# 변수 선언 (등호 앞뒤 공백 절대 금지!)
name="Alice"       # ✅
# name = "Alice"   # ❌ 에러 — 공백 있으면 명령어로 인식

# 변수 사용
echo "Hello, $name!"
echo "Hello, ${name}!"   # {} : 변수 이름 경계 명확히 할 때
```

```bash
# 명령어 결과를 변수에 저장
hour=$(date +%H)
files=$(ls *.csv)
```

## 변수 확장 — 대소문자 변환 ⭐️

```
${var,,}  → 모든 문자 소문자로
${var^^}  → 모든 문자 대문자로
${var,}   → 첫 글자만 소문자로
${var^}   → 첫 글자만 대문자로
```

```bash
name="HELLO World"

echo "${name,,}"   # hello world  (전체 소문자)
echo "${name^^}"   # HELLO WORLD  (전체 대문자)
echo "${name,}"    # hELLO World  (첫 글자만 소문자)
echo "${name^}"    # HELLO World  (첫 글자만 대문자, 이미 대문자면 그대로)
```

```
활용:
  사용자 입력을 소문자로 통일 후 case 비교
  → 대소문자 구분 없이 y/Y/Yes/YES 전부 y 로 처리

  read c
  case "${c,,}" in      ← c 를 소문자로 변환 후 비교
      y) echo "YES" ;;
      n) echo "NO"  ;;
  esac
```

```bash
# 한 줄 축약 버전
read c; case "${c,,}" in y) echo YES ;; n) echo NO ;; esac

# 원리:
#   사용자가 Y, YES, Yes 뭘 입력해도
#   ${c,,} 가 소문자로 변환 → y 하나로 통일 → 패턴 매칭
```

---

---

# ③ read — 사용자 입력 받기

```
read 변수명
  표준 입력(stdin) 에서 한 줄 읽어서 변수에 저장

코딩 테스트 / 알고리즘에서 가장 많이 쓰는 패턴:
  첫 줄에 개수(n) 가 오고
  두 번째 줄에 실제 데이터가 오는 형식
```

## 기본 사용

```bash
read n
echo "입력받은 값: $n"

# 실행 후 "5" 입력하면
# 입력받은 값: 5
```

## 여러 값 한 줄에 받기

```bash
# "1 2 3" 입력 시
read a b c
echo $a   # 1
echo $b   # 2
echo $c   # 3
```

## 배열로 받기 (-a 옵션)

```bash
# "1 2 3 4 5" 입력 시
read -a arr
echo ${arr[0]}     # 1
echo ${arr[1]}     # 2
echo ${#arr[@]}    # 5  (배열 길이)
```

## 핵심 패턴 — 첫 줄 n 으로 버리기 ⭐️

```
알고리즘 문제에서 자주 나오는 형식:
  4          ← 첫 줄: 데이터 개수
  1 2 3 4    ← 두 번째 줄: 실제 데이터

"4" 를 없애는 게 아니라
"4 를 읽어서 n 에 담는다"
→ n = 4 (배열 크기로 활용 가능)
```

```bash
read n           # 첫 줄 "4" 읽기 → n=4
read -a arr      # 두 번째 줄 "1 2 3 4" 읽기 → 배열

# n 을 활용해서 평균 계산
sum=0
for x in "${arr[@]}"; do
    sum=$(( sum + x ))
done
echo $(( sum / n ))    # 2  (10/4=2, 정수 나눗셈)

# 소수점까지
echo "scale=2; $sum / $n" | bc   # 2.50
```

```bash
# n 을 안 쓰고 그냥 버리는 경우도 있음
read _           # _ 에 담아서 무시 (관행)
read -a arr
```

## -p 옵션 — 프롬프트 같이 출력

```bash
read -p "이름 입력: " name
echo "안녕하세요, $name"

read -p "포트 번호: " port
echo "http://localhost:$port"
```

---

---

# ④ 조건문 — if / elif / else

```
대괄호 [ ] 안쪽 양옆에는 무조건 공백 필요
```

```bash
#!/bin/bash

hour=$(date +%H)

if [ "$hour" -lt 12 ]; then
    echo "Good morning!"
elif [ "$hour" -lt 18 ]; then
    echo "Good afternoon!"
else
    echo "Good evening!"
fi   # if 를 거꾸로 → "여기서 끝"
```

## 비교 연산자

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
|`-a`|AND (단일 괄호)|`[ "$x" -gt 10 -a "$x" -lt 20 ]`|
|`-o`|OR (단일 괄호)|`[ "$x" -eq 1 -o "$x" -eq 10 ]`|
|`&&`|AND (이중 괄호)|`[[ "$x" -gt 10 && "$x" -lt 20 ]]`|
|`\|`|OR (이중 괄호)|`[[ "$x" -eq 1 \| "$x" -eq 10 ]]`|

---

---

# ④ 다중 선택 — case

```
if-elif-elif 가 길어질 때 가독성이 훨씬 좋다
사용자 입력(Yes/No), 메뉴 선택에 주로 사용
```

## 기본 구조

```bash
echo "진행하시겠습니까? (y/n)"
read answer

case "$answer" in
    y|Y|yes|YES)           # | 로 여러 패턴 묶기 (OR 조건)
        echo "네, 진행합니다!"
        ;;                 # ;; = break 역할 (필수!)
    n|N|no|NO)
        echo "아니요, 멈춥니다."
        ;;
    *)                     # * = 나머지 전부 (else 역할)
        echo "잘못된 입력입니다."
        exit 1
        ;;
esac   # case 를 거꾸로 → "여기서 끝"
```

## 패턴 매칭 활용

```bash
# [yY]* → y 또는 Y 로 시작하는 모든 문자열
case "$answer" in
    [yY]*)   echo "yes 계열";;   # y, Y, yes, Year, yep 전부 잡음
    [nN]*)   echo "no 계열";;
    [0-9]*)  echo "숫자 입력";;
    *)       echo "기타";;
esac
```

## ⭐️ ${var,,} 로 소문자 통일 후 case — 패턴 최소화

```
기존 방식: y|Y|yes|YES 전부 나열
개선 방식: ${var,,} 로 소문자 변환 후 y 하나만 비교

→ 패턴이 단순해지고 누락 위험 없음
```

```bash
read answer

case "${answer,,}" in   # 입력값을 소문자로 변환 후 비교
    y|yes)
        echo "네, 진행합니다!"
        ;;
    n|no)
        echo "아니요, 멈춥니다."
        ;;
    *)
        echo "잘못된 입력입니다."
        ;;
esac
```

```bash
# 한 줄 축약 (코딩테스트 / 간단 스크립트)
read c; case "${c,,}" in y) echo YES ;; n) echo NO ;; esac

# 구조 분해:
#   read c          → 입력 받기
#   ${c,,}          → 소문자 변환
#   case ... in     → 패턴 비교
#   y) echo YES ;;  → y 면 YES 출력
#   n) echo NO  ;;  → n 면 NO 출력
#   esac            → 끝
```

```
세 가지 방식 비교:

1. 전부 나열         y|Y|yes|YES|Yes  → 패턴 많음, 누락 위험
2. 정규식 패턴       [yY]*            → 의도치 않은 문자도 잡힘 (Year, yep 등)
3. ${var,,} 소문자화 y|yes            → 가장 명확하고 안전
```

---

---

# ⑤ 반복문 — for / while

## A. for — 리스트 반복

```bash
for i in 1 2 3 4 5; do
    echo "Number: $i"
done
```

## B. 중괄호 확장 {시작..끝} ⭐️

```bash
# 숫자 범위
for i in {1..5}; do echo "$i"; done
# 1 2 3 4 5

# 증가값 지정
for i in {0..10..2}; do echo "$i"; done
# 0 2 4 6 8 10

# 내림차순
for i in {5..1}; do echo "$i"; done
# 5 4 3 2 1

# 알파벳도 가능
for c in {a..e}; do echo "$c"; done
# a b c d e
```

## C. C스타일 for

```bash
for ((i=1; i<=50; i++)); do
    echo "Count: $i"
done
```

## D. while

```bash
count=0
while [ "$count" -lt 5 ]; do
    echo "Count: $count"
    count=$((count + 1))
done
```

## E. 배열 반복 — readarray ⭐️

```bash
# 파일 → 배열
readarray -t countries < countries.txt

# 반복문 없이 전체 출력
echo "${countries[@]}"         # Korea Japan China

# 하나씩 처리
for country in "${countries[@]}"; do
    echo "Processing: $country"
done
```

---

---

# ⑥ 문법 구조 요약

```
if   ~ fi       (finish)
for  ~ do ~ done
while ~ do ~ done
case ~ esac     (case 거꾸로)
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`command not found`|`name = "Alice"` 공백|`name="Alice"` 붙여쓰기|
|`missing ]`|`if ["$a" -eq 1]` 공백 없음|`if [ "$a" -eq 1 ]` 안쪽 공백 필수|
|변수값 오작동|`$name` 따옴표 없음|`"$name"` 쌍따옴표로 감싸기|
|`{1..5}` 가 문자 그대로 출력|`sh` 로 실행|`bash` 로 실행 (`sh` 는 중괄호 확장 미지원)|
|`readarray` 요소에 `\n` 포함|`-t` 옵션 누락|`readarray -t arr < file`|
|case 에서 Y 입력이 안 잡힘|패턴 누락|`${var,,}` 로 소문자 통일 후 `y` 하나만 비교|

## ./script.sh vs source script.sh

```
./script.sh       자식 프로세스(새 창)에서 실행 → 변수가 현재 쉘에 안 남음
source script.sh  현재 쉘에서 바로 실행         → 환경 변수 적용할 때 사용
```