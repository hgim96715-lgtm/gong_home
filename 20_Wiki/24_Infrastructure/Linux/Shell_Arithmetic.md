---
aliases:
  - Shell Math
  - Arithmetic Expansion
  - 산술 연산
  - Bash 계산기
tags:
  - Linux
related:
  - "[[Linux_Shell_Script]]"
  - "[[Shell_Input_Output]]"
---
# Shell_Arithmetic — 쉘 산술 연산

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
$(( 수식 ))
  Bash 내장 산술 확장
  정수만 가능, 소수점 불가
  가장 빠르고 권장되는 방법
```

```bash
result=$(( 10 + 20 ))
echo $result   # 30

# 변수도 사용 가능 ($ 없이 써도 됨)
a=10
b=3
echo $(( a + b ))   # 13
echo $(( a * b ))   # 30
echo $(( a / b ))   # 3  ← 소수점 버림
echo $(( a % b ))   # 1  ← 나머지
echo $(( a ** b ))  # 1000 ← 거듭제곱 (a의 b제곱)
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

(( count++ ))    # count = 6  (후위 증가)
(( count-- ))    # count = 5  (후위 감소)
(( count += 3 )) # count = 8
(( count -= 2 )) # count = 6
(( count *= 2 )) # count = 12

# if 조건에서도 사용
if (( count > 10 )); then
    echo "10 초과"
fi
```

```
⚠️ Bash 는 소수점 계산 불가
   $(( 10 / 3 )) = 3  (3.333... 아님)
   소수점이 필요하면 bc 사용 (아래 참고)
```

---

---

# ② bc — 소수점 계산 (정밀)

```
bc (Basic Calculator)
  Bash 에서 소수점 계산하는 외부 도구
  echo "수식" | bc 패턴으로 사용
  scale 로 소수점 자리수 지정
```

## 기본 사용

```bash
# 파이프로 연결
echo "10 / 3" | bc        # 3  ← scale 없으면 정수만
echo "scale=2; 10 / 3" | bc   # 3.33
echo "scale=5; 10 / 3" | bc   # 3.33333
echo "scale=10; 10 / 3" | bc  # 3.3333333333
```

## scale — 소수점 자리수

```
scale=N
  결과의 소수점 아래 자리수를 N 자리까지 표시

scale=0   → 정수만 (기본값)
scale=2   → 소수점 둘째 자리
scale=5   → 소수점 다섯째 자리
scale=10  → 소수점 열째 자리 (정밀 계산)
```

```bash
echo "scale=0;  22 / 7" | bc   # 3
echo "scale=2;  22 / 7" | bc   # 3.14
echo "scale=5;  22 / 7" | bc   # 3.14285
echo "scale=10; 22 / 7" | bc   # 3.1428571428
```

## bc -l — 수학 라이브러리 로드

```
bc -l
  -l 옵션: 수학 함수 라이브러리 로드
  sin / cos / log / sqrt 등 고급 수학 함수 사용 가능
  -l 을 붙이면 scale 이 자동으로 20 으로 설정됨
```

```bash
# sqrt: 제곱근
echo "sqrt(2)" | bc -l        # 1.41421356237309504880

# s(x): sin (라디안)
echo "s(1)" | bc -l           # 0.84147098480789650665

# c(x): cos (라디안)
echo "c(0)" | bc -l           # 1.00000000000000000000

# l(x): 자연로그
echo "l(1)" | bc -l           # 0.00000000000000000000

# e(x): e의 x승
echo "e(1)" | bc -l           # 2.71828182845904523536
```

```
bc -l 수학 함수 목록:
  sqrt(x)  제곱근
  s(x)     sin
  c(x)     cos
  a(x)     arctan
  l(x)     자연로그 (ln)
  e(x)     e^x
```

## bc 변수 활용

```bash
# 여러 줄 계산
result=$(echo "scale=2; a=10; b=3; a/b" | bc)
echo $result   # 3.33

# 퍼센트 계산
total=150
done=45
echo "scale=1; $done / $total * 100" | bc   # 30.0
```


## bc 반올림 — printf 조합

```
bc scale=N 은 반올림 없이 그냥 자름 (truncate)
  scale=4; 22/7 → 3.1428  (실제 3.14285... 인데 5 버림)

반올림이 필요하면:
  scale 을 원하는 자리 + 1 로 계산
  printf %.Nf 로 반올림 출력
```

```bash
# scale 만 쓰면 그냥 자름
echo "scale=4; 22/7" | bc   # 3.1428  ← 반올림 없음

# printf 로 반올림
result=$(echo "scale=5; 22/7" | bc)   # 한 자리 더 계산
printf "%.4f\n" $result                # 3.1429  ← 5번째에서 반올림
```

```bash
# read 와 같이 쓸 때
read air
result=$(echo "scale=5; $air" | bc)
printf "%.4f\n" $result
```

```
printf "%.Nf" 의미:
  %f    실수 형식으로 출력
  .N    소수점 N번째 자리까지 (N+1번째에서 반올림)

  printf "%.2f" 3.14567  →  3.15  (3번째 자리에서 반올림)
  printf "%.4f" 3.14285  →  3.1429
```


---

---

# ③ expr — 구식 방법 (참고용)

```
expr 은 오래된 방식
$(( )) 이 도입된 이후로는 잘 안 씀
레거시 스크립트에서 볼 수 있으므로 읽을 줄만 알면 됨
```

```bash
expr 10 + 3     # 13
expr 10 \* 3    # 30  ← * 는 반드시 \* 로 이스케이프
expr 10 / 3     # 3

# 변수에 저장
result=$(expr 10 + 3)
echo $result    # 13
```

```
expr 단점:
  * 앞에 \ 붙여야 하는 불편함
  공백 필수 (expr 10+3 → 에러)
  → $(( )) 쓰는 게 훨씬 편함
```

---

---

# 실전 패턴 모음

```bash
# 파일 용량 계산 (KB → MB)
size_kb=2048
size_mb=$(( size_kb / 1024 ))
echo "${size_mb}MB"   # 2MB

# 진행률 계산 (소수점 필요)
done=45
total=150
pct=$(echo "scale=1; $done / $total * 100" | bc)
echo "진행률: ${pct}%"   # 진행률: 30.0%

# 홀짝 판별
num=7
if (( num % 2 == 0 )); then
    echo "짝수"
else
    echo "홀수"
fi

# 카운터 활용
count=0
for file in *.csv; do
    (( count++ ))
done
echo "CSV 파일 수: $count"
```

---

---

# 정리

|방법|소수점|속도|용도|
|---|---|---|---|
|`$(( ))`|❌|빠름|정수 계산, 카운터, 조건|
|`echo "..." \| bc`|✅|보통|소수점 계산|
|`echo "..." \| bc -l`|✅|보통|삼각함수, 로그, 제곱근|
|`expr`|❌|느림|레거시 스크립트 읽기용|

```
결론:
  정수 → $(( ))
  소수점 → echo "scale=N; 수식" | bc
  수학함수 → echo "함수(x)" | bc -l
```