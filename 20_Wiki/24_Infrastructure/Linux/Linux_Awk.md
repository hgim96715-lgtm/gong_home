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
# Linux_Awk — 필드 단위 텍스트 처리

## 한 줄 요약

```
줄을 필드(열)로 쪼개서
조건 처리 / 집계 / 계산하는 도구

sed   줄(line) 단위 처리  → 치환 / 삭제
awk   필드(열) 단위 처리  → 열 추출 / 조건 / 계산 / 집계
```

---

---

# ① 기본 구조

```bash
awk '패턴 { 액션 }' 파일
awk '{ 액션 }' 파일          # 패턴 없으면 모든 줄에 적용
awk -F "구분자" '{ 액션 }' 파일
명령어 | awk '{ 액션 }'      # 파이프로 stdin 입력
```

```
실행 흐름:
  BEGIN  { }   → 파일 읽기 전 1번 실행 (초기화)
  { }          → 모든 줄마다 실행 (본문)
  END    { }   → 파일 다 읽은 후 1번 실행 (결과 출력)
```

---

---

# ② 필드 접근 — $숫자 ⭐️

```
awk 에서 $ 는 "변수"가 아니라 "필드(열) 번호"
쉘 변수 $var 와 완전히 다른 개념

$1   첫 번째 필드
$2   두 번째 필드
$NF  마지막 필드 (NF = 현재 줄의 필드 개수)
$0   줄 전체
```

```bash
# 예시 데이터 (data.txt)
# Alice 30 Seoul
# Bob   25 Busan
# Carol 28 Seoul

awk '{ print $1 }' data.txt    # Alice / Bob / Carol
awk '{ print $2 }' data.txt    # 30 / 25 / 28
awk '{ print $NF }' data.txt   # Seoul / Busan / Seoul  (마지막 열)
awk '{ print $0 }' data.txt    # 줄 전체
```

```
⚠️ $2 는 줄이 바뀌면 값도 바뀜
  1번째 줄: Alice 30 Seoul → $2 = 30
  2번째 줄: Bob   25 Busan → $2 = 25
  3번째 줄: Carol 28 Seoul → $2 = 28
```

## print 에서 콤마 = 공백(OFS) 으로 구분

```bash
awk '{ print $1, $2, $3 }' data.txt
# Alice 30 Seoul    ← 콤마가 공백으로 출력됨

# print $1 $2       ← 붙여서 출력 (AliceBusan)
# print $1, $2      ← OFS(기본 공백)로 구분 (Alice 30)
# print $1 "," $2   ← 쉼표 직접 지정 (Alice,30)
```

```
콤마(,) 의 역할:
  print 에서 , 는 OFS(출력 구분자) 를 끼워 넣음
  OFS 기본값 = 공백
  → print $1, $2 = "$1 $2" 처럼 보임

  OFS 바꾸면:
  BEGIN { OFS="|" }
  print $1, $2    → "Alice|30"
```

---

---

# ③ 내장 변수

|변수|의미|기본값|
|---|---|---|
|`NF`|현재 줄의 필드 개수|(줄마다 다름)|
|`NR`|현재까지 읽은 줄 번호|1부터|
|`FS`|입력 구분자|공백|
|`OFS`|출력 필드 구분자|공백|
|`ORS`|출력 레코드 구분자|`\n`|

```bash
awk '{ print NR, $1 }' data.txt
# 1 Alice
# 2 Bob
# 3 Carol

awk '{ print NF, $NF }' data.txt
# 3 Seoul    ← 필드 3개, 마지막 = Seoul
```

## OFS vs ORS

```
OFS  같은 줄 안에서 필드(열) 사이 구분자
ORS  줄과 줄 사이 구분자 (줄바꿈 대체)

print $1, $2, $3
→ $1 [OFS] $2 [OFS] $3 [ORS]
       ↑                  ↑
   열 구분 (공백)       줄 끝 (\n)
```

```bash
awk 'BEGIN { OFS="|" } { print $1, $2 }' data.txt
# Alice|30
# Bob|25

awk 'BEGIN { ORS="\t" } { print $1 }' data.txt
# Alice  Bob  Carol    ← 줄바꿈 → 탭으로 (가로로 이어짐)
```

---

---

# ④ 구분자 지정 (-F / BEGIN FS)

```bash
# CSV
awk -F "," '{ print $1, $3 }' data.csv

# 콜론 구분 (/etc/passwd)
awk -F ":" '{ print $1, $3 }' /etc/passwd

# BEGIN { FS } 방식 — 스크립트 파일로 저장할 때
awk 'BEGIN { FS=","; OFS="|" } { print $1, $3 }' data.csv
```

```
-F ","                 → 명령줄 옵션으로 지정 (간단)
BEGIN { FS="," }       → 코드 안에 포함 (스크립트 파일에 유리)
→ 결과 동일 / 상황에 따라 선택
```

---

---

# ⑤ 조건문 — if / else if / else ⭐️

```
awk 안에서 if / else if / else 전부 사용 가능
중괄호 { } 안에서 일반 프로그래밍 언어처럼 작성
```

## 기본 if

```bash
awk '{ if ($2 >= 80) print $1, "합격" }' data.txt
```

## if / else if / else — 성적 계산 패턴

```bash
awk '{
    avg = ($2 + $3 + $4) / 3
    if      (avg >= 80) grade = "A"
    else if (avg >= 60) grade = "B"
    else if (avg >= 50) grade = "C"
    else                grade = "FAIL"
    print $0, ":", grade
}' data.txt
```

```
avg = ($2 + $3 + $4) / 3:
  awk 에서 사칙연산 바로 가능
  변수 선언 없이 바로 사용 (자동으로 0 초기화)

print $0, ":", grade:
  $0   → 줄 전체
  ","  → OFS(공백) 삽입
  ":"  → 리터럴 문자열
  grade → 변수
  → "Alice 90 85 78 : A" 처럼 출력
```

```bash
# 한 줄로 쓰기 (세미콜론으로 구분)
awk '{ avg=($2+$3+$4)/3; if(avg>=80) g="A"; else if(avg>=60) g="B"; else g="FAIL"; print $1, avg, g }' data.txt
```

## 조건 패턴 (패턴으로 필터링)

```bash
# 정규식 매칭
awk '/Seoul/ { print $1 }' data.txt

# 숫자 조건
awk '$2 > 27 { print $1, $2 }' data.txt

# 특정 줄 번호
awk 'NR==2 { print }' data.txt
awk 'NR>=2 && NR<=4' data.txt

# 필드 값 일치
awk '$3 == "Seoul" { print $1 }' data.txt
```

---

---

# ⑥ 반복문 — for

```bash
# 한 줄의 모든 필드 합산
awk '{
    sum = 0
    for (i = 1; i <= NF; i++)
        sum += $i
    print $0, "합계:", sum
}' data.txt
```

```
for (i=1; i<=NF; i++):
  i = 1 부터 NF(마지막 필드 번호) 까지
  $i = i 번째 필드 (변수로 필드 번호 지정)
```

```bash
# 실전: 여러 점수 평균 계산 (열 수 몰라도 됨)
awk '{
    sum = 0
    for (i = 2; i <= NF; i++)    # 1열은 이름이라 2열부터
        sum += $i
    avg = sum / (NF - 1)
    printf "%s 평균: %.1f\n", $1, avg
}' data.txt
```

---

---

# ⑦ BEGIN / END — 집계 계산

```bash
# 합계 + 평균
awk '
BEGIN { sum=0; count=0 }
{
    sum += $2
    count++
}
END {
    print "합계:", sum
    print "평균:", sum/count
}
' data.txt
```

```
실행 순서:
  BEGIN: sum=0, count=0 초기화
  1줄: $2=30 → sum=30, count=1
  2줄: $2=25 → sum=55, count=2
  3줄: $2=28 → sum=83, count=3
  END: 합계 83 / 평균 27.6667 출력
```

```bash
# 최댓값 찾기
awk '
BEGIN { max=0 }
{ if ($2 > max) max = $2 }
END { print "최대:", max }
' data.txt

# 그룹별 카운트
awk '{ count[$3]++ }
END { for (city in count) print city, count[city] }' data.txt
# Seoul 2
# Busan 1
```

---

---

# ⑧ printf — 포맷 출력

```bash
awk '{ printf "%-10s %3d %s\n", $1, $2, $3 }' data.txt
# Alice       30 Seoul
# Bob         25 Busan
# Carol       28 Seoul
```

|서식|의미|예시|
|---|---|---|
|`%s`|문자열|`%-10s` 왼쪽 정렬 10칸|
|`%d`|정수|`%3d` 오른쪽 정렬 3칸|
|`%f`|실수|`%.2f` 소수점 2자리|
|`\n`|줄바꿈||
|`\t`|탭||

```bash
awk '{
    avg = ($2 + $3 + $4) / 3
    printf "%-10s 평균: %6.2f\n", $1, avg
}' data.txt
# Alice      평균:  84.33
# Bob        평균:  70.67
```

---

---

# ⑨ 실전 패턴 모음

```bash
# 특정 열만 출력
awk '{ print $1, $3 }' data.txt

# 빈 줄 제외
awk 'NF > 0' data.txt

# 줄 수 세기 (wc -l 대용)
awk 'END { print NR }' data.txt

# 마지막 열
awk '{ print $NF }' data.txt

# 로그에서 에러 줄 + 줄번호
awk '/ERROR/ { print NR": "$0 }' app.log

# 중복 없이 특정 열 값 모으기
awk '!seen[$3]++ { print $3 }' data.txt

# CSV 특정 열 합계
awk -F "," '{ sum += $2 } END { print sum }' data.csv
```

## read + awk 조합 — 알고리즘 입력 처리 ⭐️

```
알고리즘 입력 형식:
  4          ← 첫 줄: 데이터 개수 N
  1 2 3 4    ← 두 번째 줄: 실제 데이터

read num   → 첫 줄(4) 소비 → num=4
awk        → 남은 줄(실제 데이터)만 처리
```

```bash
# 입력: 4 / 10 20 30 40

read num
awk -v n="$num" '{
    sum = 0
    for (i=1; i<=NF; i++) sum += $i
    printf "합계: %d\n평균: %.1f\n", sum, sum/n
}'

# -v n="$num"  → 쉘 변수 num 을 awk 변수 n 으로 전달
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`print $1$2` 붙어서 출력|콤마 빠짐|`print $1, $2` 콤마로 구분|
|`$2` 가 항상 같은 값인 줄 앎|줄이 바뀌면 값도 바뀜|awk 는 줄마다 $1 $2 재계산|
|쉘 변수 `$var` 랑 혼동|awk 의 $ 는 필드 번호|awk 에서 변수는 `$` 없이 `var`|
|`if` 조건 후 `{}` 없어서 에러|여러 문장은 `{}` 필수|단일 문장이면 생략 가능|
|`sum/NR` 이 0|BEGIN 에서 초기화 안 함|`BEGIN { sum=0 }`|