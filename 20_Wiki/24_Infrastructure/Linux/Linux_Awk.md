---
aliases:
  - awk
  - 필드 처리
  - 텍스트 집계
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Grep]]"
  - "[[Linux_File_Commands]]"
---

# Linux_Awk — awk 텍스트 처리

## 한 줄 요약

```
라인을 필드(열)로 쪼개서 조건 처리·집계하는 도구
CSV / 로그 / 공백 구분 데이터에서 특정 열만 뽑거나 계산할 때

⭐️ awk 는 stdin 에서 읽는다
   read 와 함께 쓰면 read 가 먼저 한 줄 소비
   → awk 는 남은 줄(실제 데이터)만 처리
   → 첫 줄(N) 은 awk 에 안 들어옴
```

---

---

# ① 기본 구조

```bash
awk '패턴 { 액션 }' 파일명
awk '{ 액션 }' 파일명          # 패턴 없으면 모든 줄에 적용
awk -F "구분자" '{ 액션 }' 파일명  # 구분자 지정
```

```bash
# 예시 파일 (data.txt)
# Alice 30 Seoul
# Bob   25 Busan
# Carol 28 Seoul

awk '{ print $1 }' data.txt
# Alice
# Bob
# Carol
```

---

---

# ② $숫자 — 필드 접근 ⭐️

```
awk 에서 $ 는 "변수"가 아니라 "필드(열) 번호" 를 의미
쉘 변수와 완전히 다른 개념!

$1   첫 번째 필드
$2   두 번째 필드
$3   세 번째 필드
$NF  마지막 필드 (NF = 현재 줄의 필드 개수)
$0   줄 전체

$1, $2 는 줄 번호가 아니라 열 번호:

  데이터:
    1 2
    3 4
    5 6

  $1 → 1, 3, 5   ← 각 줄의 첫 번째 칸
  $2 → 2, 4, 6   ← 각 줄의 두 번째 칸

  awk 는 줄을 하나씩 읽으면서
  그 줄에서 $1, $2 를 추출
  줄이 바뀌면 $1, $2 도 새 줄의 값으로 바뀜

⚠️ $2 는 고정된 값이 아님
   1번째 줄: Alice 30 Seoul  → $2 = 30
   2번째 줄: Bob   25 Busan  → $2 = 25
   3번째 줄: Carol 28 Seoul  → $2 = 28
```

```
⚠️ $num ≠ 변수 num

쉘 스크립트에서:
  num=3
  echo $num   → "3"  (변수 num 의 값)

awk 에서:
  $3          → 세 번째 필드 값
  num=3
  $num        → 세 번째 필드 값  ($3 과 동일!)
  print num   → 3 (변수 num 의 값)
  print $num  → 세 번째 필드 값

즉, awk 에서 $num 은
  "변수 num 에 담긴 숫자번째 필드를 가져와라"
```

```bash
# Alice 30 Seoul

awk '{ print $1 }' data.txt    # Alice  (첫 번째 필드)
awk '{ print $2 }' data.txt    # 30     (두 번째 필드)
awk '{ print $NF }' data.txt   # Seoul  (마지막 필드)
awk '{ print $0 }' data.txt    # Alice 30 Seoul (줄 전체)

# 변수로 필드 번호 지정
awk '{ n=2; print $n }' data.txt   # 30  ($2 와 동일)
```

---

---

# ③ 내장 변수

```
NF   현재 줄의 필드 개수 (Number of Fields)
NR   현재까지 읽은 줄 번호 (Number of Records)
FS   입력 구분자 (Field Separator, 기본 공백)
OFS  출력 필드 구분자 (Output Field Separator, 기본 공백)
ORS  출력 레코드 구분자 (Output Record Separator, 기본 \n)
RS   레코드 구분자 (기본 줄바꿈)
```

## OFS vs ORS — 헷갈리는 포인트 ⭐

>"OFS는 열(컬럼), ORS는 행(라인)을 바꾼다"

```
OFS  같은 줄 안에서 필드(열)와 필드 사이 구분자
ORS  줄과 줄 사이 구분자 (줄바꿈을 다른 문자로 교체)

print $1, $2, $3
→  $1 [OFS] $2 [OFS] $3 [ORS]
       ↑                  ↑
   열 구분 (공백)       줄 끝 (\n)

OFS 바꾸면: 열과 열 사이가 바뀜
ORS 바꾸면: 줄바꿈 자체가 바뀜
```

```bash
# OFS 예시 — 열 구분자 변경
awk 'BEGIN { OFS="|" } { print $1, $2, $3 }' data.txt
# Alice|30|Seoul
# Bob|25|Busan

# ORS 예시 — 줄바꿈을 탭으로 교체
awk 'BEGIN { ORS="\t" } { print $1 }' data.txt
# Alice	Bob	Carol	  (줄바꿈 없이 탭으로 이어짐)

# ORS="\t" 는 세로 데이터를 가로로 만들 때 유용
# paste -s -d '\t' 와 동일한 효과
awk '{ print $1 }' data.txt | awk 'BEGIN { ORS="\t" } { print }'
# Alice	Bob	Carol
```

```bash
# 실전: 여러 줄 데이터를 한 줄로 합치기
# 방법 1: ORS 변경
awk 'BEGIN { ORS="," } { print $1 }' data.txt
# Alice,Bob,Carol,   (마지막에 쉼표 남음)

# 방법 2: paste -s 와 비교 # [[Linux_File_Commands#⑧ paste — 파일/줄 붙이기]] 참고 
awk '{ print $1 }' data.txt | paste -s -d ','
# Alice,Bob,Carol    (깔끔)

# ORS 방식은 마지막 구분자가 남는 점 주의
```


```bash
# NR — 줄 번호 출력
awk '{ print NR, $1 }' data.txt
# 1 Alice
# 2 Bob
# 3 Carol

# NF — 필드 개수 / 마지막 필드
awk '{ print NF, $NF }' data.txt
# 3 Seoul
# 3 Busan
# 3 Seoul
```

---

---

---

# ④ 구분자 지정 (-F)

```bash
# CSV 파일
# Alice,30,Seoul
# Bob,25,Busan

awk -F "," '{ print $1, $3 }' data.csv
# Alice Seoul
# Bob Busan

# 콜론 구분 (/etc/passwd)
# /etc/passwd 구조:
# root:x:0:0:root:/root:/bin/bash 
# $1  $2  $3  $4    $5   $6   $7
# 사용자명:패스워드:UID:GID:설명:홈디렉토리:쉘
awk -F ":" '{ print $1, $3 }' /etc/passwd
# root 0
# ...
```

## -F 말고 BEGIN { FS } 로 지정하는 이유

```
-F ","                    → 명령줄에서 구분자 지정 (간단)
BEGIN { FS="," }          → awk 코드 안에서 구분자 지정

차이:
  -F 는 외부 옵션 → 스크립트 파일로 저장할 때 함께 안 따라감
  BEGIN { FS } 는 코드 안에 포함 → 스크립트 파일로 저장해도 유지

스크립트로 재사용하거나
구분자를 조건에 따라 동적으로 바꿀 때 BEGIN { FS } 사용
```


```bash
# -F 방식 (간단)
awk -F "," '{ print $1 }' data.csv

# BEGIN { FS } 방식 (스크립트 파일로 저장할 때)
awk 'BEGIN { FS="," } { print $1 }' data.csv

# 위 둘은 결과 동일
# 차이: BEGIN 방식은 FS 외에 초기값 세팅도 같이 할 수 있음
awk 'BEGIN { FS=","; OFS="|" } { print $1, $3 }' data.csv
# Alice|Seoul
# Bob|Busan
# OFS: 출력할 때 , 대신 | 로 구분
```

```
BEGIN 안에서 할 수 있는 것들:
  FS  입력 구분자 설정
  OFS 출력 구분자 설정
  변수 초기화   BEGIN { sum=0; count=0 }
  헤더 출력     BEGIN { print "=== 결과 ===" }
```


---

---

# ⑤ 조건 패턴

```bash
# 특정 패턴 매칭
awk '/Seoul/ { print $1 }' data.txt
# Alice
# Carol

# 조건식
awk '$2 > 27 { print $1, $2 }' data.txt
# Alice 30
# Carol 28

# 특정 줄만 (NR)
awk 'NR==2 { print }' data.txt      # 2번째 줄만
awk 'NR>=2 && NR<=4' data.txt       # 2~4번째 줄
```

---

---

# ⑥ BEGIN / END

```
BEGIN  파일 읽기 전에 실행 (초기화 / 헤더 출력)
{ }    모든 줄마다 실행 (본문)
END    파일 다 읽은 후 실행 (집계 결과 출력)
```

## 집계 계산 상세

```bash
# data.txt
# Alice 30 Seoul
# Bob   25 Busan
# Carol 28 Seoul
```

```bash
# 합계 계산
awk '{ sum += $2 } END { print "합계:", sum }' data.txt
# 합계: 83

# $2 는 고정값이 아님
# awk 는 파일을 한 줄씩 읽으면서 { } 안을 실행
# 줄이 바뀔 때마다 $2 도 그 줄의 두 번째 칸으로 바뀜

# 1번째 줄: Alice 30 Seoul  → $2=30  → sum = 0+30 = 30
# 2번째 줄: Bob   25 Busan  → $2=25  → sum = 30+25 = 55
# 3번째 줄: Carol 28 Seoul  → $2=28  → sum = 55+28 = 83
# END 에서 출력 → 합계: 83
```

```bash
# 평균 계산
awk '{ sum += $2; count++ } END { print "평균:", sum/count }' data.txt
# 평균: 27.6667

# sum += $2   → 합계 누적 (위와 동일)
# count++     → count = count + 1  (줄 수 카운트)
#               1번째 줄 후: count=1
#               2번째 줄 후: count=2
#               3번째 줄 후: count=3
# END: 83 / 3 = 27.6667
```

```bash
# BEGIN 으로 초기화 명시 (더 명확한 코드)
awk '
BEGIN { sum=0; count=0 }
{ sum += $2; count++ }
END { print "합계:", sum; print "평균:", sum/count }
' data.txt
# 합계: 83
# 평균: 27.6667

# awk 는 숫자 변수를 자동으로 0 초기화하지만
# BEGIN 에서 명시하면 코드 의도가 명확해짐
```

```
연산자 정리:
  sum += $2    sum = sum + $2  (누적 덧셈)
  count++      count = count + 1 (1씩 증가)
  sum -= $2    sum = sum - $2  (누적 뺄셈)
  n *= 2       n = n * 2

  awk 변수는 선언 없이 바로 사용 가능
  숫자 변수 초기값 = 0
  문자 변수 초기값 = ""
```

---

---

# ⑦ 출력 포맷 — printf

```bash
# printf 로 정렬된 출력
awk '{ printf "%-10s %3d %s\n", $1, $2, $3 }' data.txt
# Alice       30 Seoul
# Bob         25 Busan
# Carol       28 Seoul

# %-10s  왼쪽 정렬 10칸
# %3d    오른쪽 정렬 3칸 정수
# %s     문자열
```

---

---

# ⑧ 실전 패턴 모음

```bash
# ① 특정 열만 출력
awk '{ print $1, $3 }' data.txt

# ② CSV 에서 2열이 특정 값인 행 필터
awk -F "," '$2 == "Seoul" { print $1 }' data.csv

# ③ 로그에서 에러 줄 + 줄번호
awk '/ERROR/ { print NR": "$0 }' app.log

# ④ 파일에서 빈 줄 제외
awk 'NF > 0' data.txt

# ⑤ 줄 수 세기 (wc -l 대용)
awk 'END { print NR }' data.txt

# ⑥ 특정 열 합계
awk '{ sum += $2 } END { print sum }' data.txt

# ⑦ 중복 없이 특정 열 값 모으기
awk '!seen[$3]++ { print $3 }' data.txt

# ⑧ 마지막 열 출력
awk '{ print $NF }' data.txt
```

## read + awk 조합 패턴 ⭐️

```
알고리즘 / 코딩 테스트 입력 형식:
  4          ← 첫 줄: 데이터 개수 N
  1 2 3 4    ← 두 번째 줄: 실제 숫자 데이터

read num   → 첫 줄(4) 을 소비해서 num 에 저장
             이후 awk 는 남은 줄부터 읽음
awk        → 실제 숫자 데이터만 처리
$1         → 각 줄의 첫 번째 숫자
```


```bash
# 입력:
# 4
# 1 2 3 4

read num                                    # "4" 소비 → num=4
awk '{ for(i=1; i<=NF; i++) sum += $i }
     END { print sum/NF }' | read result    # 평균 계산

# 더 일반적인 패턴
read num
awk -v n="$num" '{ sum=0; for(i=1; i<=NF; i++) sum+=$i; print sum/n }'
```


```bash
# 한 줄에 숫자 여러 개 → 합계
# 입력:
# 4
# 10 20 30 40

read num
awk '{ for(i=1; i<=NF; i++) sum += $i }
     END { print "합계:", sum; print "평균:", sum/NF }'

# 합계: 100
# 평균: 25
```


```bash
# 여러 줄에 숫자 하나씩 → 합계
# 입력:
# 4
# 10
# 20
# 30
# 40

read num
awk '{ sum += $1 } END { print sum/NR }'
# $1  → 각 줄의 숫자 (한 줄에 하나)
# NR  → 총 줄 수 (num 과 동일)
# 평균: 25
```

```
read num 이 "첫 줄을 소비" 한다는 의미:
  stdin(표준 입력) 에서 한 줄을 읽어서 num 에 저장
  다음 명령어(awk) 는 남은 입력부터 읽음
  → 첫 줄(4) 은 awk 에 안 들어옴
  → awk 는 실제 데이터 줄만 처리
```

---

---

# ⑨ sed vs awk

```
sed   줄(line) 단위 처리   → 치환 / 삭제 / 삽입
awk   필드(field) 단위 처리 → 열 추출 / 집계 / 조건 필터

특정 단어 치환     → sed
특정 열만 출력     → awk
합계/평균 계산     → awk
패턴 있는 줄 삭제  → sed
조건에 맞는 행 출력 → awk (또는 grep)
```