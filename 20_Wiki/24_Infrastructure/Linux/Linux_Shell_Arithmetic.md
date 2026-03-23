---
aliases:
  - Shell Math
  - Arithmetic Expansion
  - 산술 연산
  - Bash 계산기
  - printf
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Shell_Script]]"
  - "[[Linux_Scheduling_Crontab]]"
---
# Linux_Shell_Arithmetic — 쉘 산술 연산

## 한 줄 요약

```
쉘은 모든 것을 문자열로 취급
1+1 입력하면 숫자 2 가 아니라 글자 "1+1" 이 나옴
→ 계산하려면 특수 문법 안에 가둬야 함
```

---

---

# ① $(( )) — 정수 계산 (기본)

```
Bash 내장 산술 확장
정수만 가능 / 소수점 불가
가장 빠르고 권장되는 방법
```

```bash
result=$(( 10 + 20 ))
echo $result   # 30

# 변수 사용 ($ 없이 써도 됨)
a=10
b=3
echo $(( a + b ))   # 13
echo $(( a * b ))   # 30
echo $(( a / b ))   # 3   ← 소수점 버림
echo $(( a % b ))   # 1   ← 나머지
echo $(( a ** b ))  # 1000 ← 거듭제곱
```

## 사칙연산 기호

|기호|의미|예시|결과|
|---|---|---|---|
|`+`|더하기|`$(( 10 + 3 ))`|13|
|`-`|빼기|`$(( 10 - 3 ))`|7|
|`*`|곱하기|`$(( 10 * 3 ))`|30|
|`/`|나누기 (정수 몫)|`$(( 10 / 3 ))`|3|
|`%`|나머지|`$(( 10 % 3 ))`|1|
|`**`|거듭제곱|`$(( 2 ** 8 ))`|256|

## 증감 연산자

```bash
count=5
(( count++ ))    # 6  (후위 증가)
(( count-- ))    # 5  (후위 감소)
(( count += 3 )) # 8
(( count -= 2 )) # 6
(( count *= 2 )) # 12

# if 조건에서 사용
if (( count > 10 )); then
    echo "10 초과"
fi
```

---

---

# ② bc — 소수점 계산

```
bc (Basic Calculator)
  Bash 에서 소수점 계산하는 외부 도구
  echo "수식" | bc 패턴
  scale 로 소수점 자리수 지정
```

## 기본 사용

```bash
echo "10 / 3" | bc              # 3  ← scale 없으면 정수
echo "scale=2; 10 / 3" | bc    # 3.33
echo "scale=5; 10 / 3" | bc    # 3.33333
echo "scale=10; 10 / 3" | bc   # 3.3333333333
```

## scale — 소수점 자리수

```
scale=N  → 소수점 아래 N 자리까지 표시 (기본값 0)
주의: 반올림 없이 그냥 자름 (truncate)
```

```bash
echo "scale=4; 22/7" | bc   # 3.1428  ← 실제 3.14285... 인데 5 버림
```

## bc -l — 수학 라이브러리

```
-l 옵션: sin / cos / log / sqrt 등 수학 함수 사용 가능
-l 붙이면 scale 자동 20 설정
```

```bash
echo "sqrt(2)" | bc -l    # 1.41421356237309504880
echo "s(1)"    | bc -l    # 0.84147098480789650665  sin
echo "c(0)"    | bc -l    # 1.00000000000000000000  cos
echo "l(1)"    | bc -l    # 0.00000000000000000000  자연로그
echo "e(1)"    | bc -l    # 2.71828182845904523536  e^x
```

|함수|의미|
|---|---|
|`sqrt(x)`|제곱근|
|`s(x)`|sin (라디안)|
|`c(x)`|cos (라디안)|
|`a(x)`|arctan|
|`l(x)`|자연로그 (ln)|
|`e(x)`|e^x|

---

---

# ③ printf — 출력 형식 지정 ⭐️

```
printf "서식" 값
  C 언어 printf 와 동일한 문법
  echo 보다 정밀한 출력 제어 가능
```

## 서식 지정자

|서식|의미|예시|결과|
|---|---|---|---|
|`%s`|문자열|`printf "%s" "hello"`|`hello`|
|`%s`|문자열 + 공백|`printf "%s " "hello"`|`hello`|
|`%d`|정수|`printf "%d\n" 42`|`42`|
|`%f`|실수|`printf "%f\n" 3.14`|`3.140000`|
|`%.2f`|소수점 2자리|`printf "%.2f\n" 3.14567`|`3.15`|
|`%05d`|0으로 패딩|`printf "%05d\n" 42`|`00042`|
|`\n`|줄바꿈|||

```bash
printf "%s " "hello"      # hello  (공백 포함)
printf "%s\n" "world"     # world  (줄바꿈 포함)
printf "%d\n" 42          # 42
printf "%.2f\n" 3.14567   # 3.15   ← 반올림
printf "%05d\n" 42        # 00042  ← 0 패딩
```

## %s 는 문자열 + 공백 패턴

```bash
# 배열 원소를 공백으로 이어서 출력
arr=("apple" "banana" "cherry")
printf "%s " "${arr[@]}"   # apple banana cherry
echo                        # 마지막 줄바꿈

# 숫자도 %s 로 출력 가능 (문자열로 취급)
printf "%s " 1 2 3 4 5     # 1 2 3 4 5
```

## printf 반올림 — bc 조합 ⭐️

```
bc scale=N 은 반올림 없이 자름
반올림이 필요하면:
  scale 을 원하는 자리 + 1 로 넉넉히 계산
  printf %.Nf 로 반올림 출력
```

```bash
# scale 만 쓰면 그냥 자름
echo "scale=4; 22/7" | bc    # 3.1428  ← 반올림 없음

# printf 조합으로 반올림
result=$(echo "scale=5; 22/7" | bc)   # 한 자리 더 계산
printf "%.4f\n" $result                # 3.1429  ← 5번째에서 반올림

# read + bc + printf 패턴
read num
result=$(echo "scale=5; $num / 3" | bc)
printf "%.4f\n" $result
```

```
printf "%.Nf" 동작:
  .N    소수점 N번째 자리까지 표시
        N+1 번째 자리에서 반올림

  printf "%.2f" 3.14567  →  3.15
  printf "%.4f" 3.14285  →  3.1429
  printf "%.0f" 3.7      →  4    ← 정수 반올림
```

---

---

# ④ expr — 구식 방법 (참고용)

```
$(( )) 이전 방식
레거시 스크립트에서 볼 수 있으므로 읽을 줄만 알면 됨
```

```bash
expr 10 + 3     # 13
expr 10 \* 3    # 30  ← * 는 반드시 \* 로 이스케이프
expr 10 / 3     # 3
result=$(expr 10 + 3)
```

```
단점:
  * 앞에 \ 붙여야 함
  공백 필수 (expr 10+3 → 에러)
  → $(( )) 이 훨씬 편함
```

---

---

# 실전 패턴 모음

```bash
# 파일 용량 (KB → MB)
size_kb=2048
echo "${size_kb} / 1024 = $(( size_kb / 1024 ))MB"

# 진행률 (소수점)
done=45; total=150
pct=$(echo "scale=1; $done / $total * 100" | bc)
echo "진행률: ${pct}%"   # 30.0%

# 홀짝 판별
num=7
if (( num % 2 == 0 )); then echo "짝수"; else echo "홀수"; fi

# 카운터
count=0
for file in *.csv; do (( count++ )); done
echo "CSV 파일 수: $count"

# 배열 원소 공백으로 출력
arr=(1 2 3 4 5)
printf "%s " "${arr[@]}"; echo
```

---

---

# 정리

|방법|소수점|속도|용도|
|---|---|---|---|
|`$(( ))`|❌|빠름|정수 계산 / 카운터 / 조건|
|`echo "..." \| bc`|✅|보통|소수점 계산|
|`echo "..." \| bc -l`|✅|보통|삼각함수 / 로그 / 제곱근|
|`printf "%.Nf"`|✅|빠름|반올림 출력 / 형식 지정|
|`expr`|❌|느림|레거시 스크립트 읽기용|

```
정수 계산    → $(( ))
소수점 계산  → echo "scale=N; 수식" | bc
반올림 출력  → printf "%.Nf" $result
수학 함수    → echo "함수(x)" | bc -l
배열 출력    → printf "%s " "${arr[@]}"
```