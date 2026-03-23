---
aliases:
  - sed
  - 텍스트가공
  - 데이터 전처리
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Commands]]"
  - "[[Linux_Awk]]"
  - "[[Linux_Grep]]"
---
# Linux_Sed — 스트림 편집기

## 한 줄 요약

```
파일이나 stdin 을 한 줄씩 읽어서 변환하는 스트림 편집기
파일 직접 열지 않고 텍스트를 찾아서 바꾸거나 필터링
```

---

---

# ① 기본 문법

```bash
sed '명령어' 파일
명령어 | sed '명령어'   # stdin 입력
```

```bash
# 파일에서 실행
sed 's/old/new/' file.txt

# 파이프로 입력
cat file.txt | sed 's/old/new/'
echo "hello world" | sed 's/world/korea/'
```

---

---

# ② 치환 — s/찾기/바꾸기/플래그

## 기본 구조

```
s / 찾을패턴 /   바꿀문자 /    플래그
↑              ↑           ↑
substitute    replacement  options
```

```bash
# 첫 번째 매칭만 치환 (기본)
echo "aaa bbb aaa" | sed 's/aaa/XXX/'
# XXX bbb aaa  ← 첫 번째만 바뀜

# g 플래그: 줄 전체 모두 치환
echo "aaa bbb aaa" | sed 's/aaa/XXX/g'
# XXX bbb XXX  ← 전부 바뀜

# i 플래그: 대소문자 무시
echo "Hello HELLO hello" | sed 's/hello/hi/gi'
# hi hi hi
```

## 구분자 변경 — / 대신 다른 문자

```bash
# 경로에 / 가 있으면 \/ 로 이스케이프 해야 해서 지저분
sed 's/\/usr\/local/\/opt/'

# 구분자를 | 또는 # 으로 바꾸면 깔끔
sed 's|/usr/local|/opt|'
sed 's#/usr/local#/opt#'
```

---

---

# ③ s/\t/;/g — 어떻게 만드는가

```
s   → substitute (치환)
\t  → 탭 문자 (특수문자는 \ 붙임)
;   → 바꿀 문자
g   → global (줄 전체 모두 치환)

→ "탭을 전부 ; 로 바꿔라"
```

```bash
# paste 결과의 탭을 ; 로 치환
echo -e "1\n2\n3\n4\n5\n6" | paste - - - | sed 's/\t/;/g'
# 1;2;3
# 4;5;6

# CSV 에서 쉼표를 탭으로
sed 's/,/\t/g' data.csv

# 여러 공백을 탭으로
sed 's/  */\t/g' data.txt
```

## 특수문자 패턴 정리

|패턴|의미|
|---|---|
|`\t`|탭|
|`\n`|줄바꿈|
|`\s`|공백 (GNU sed)|
|`\.`|점 (. 은 정규식에서 아무 문자)|
|`\\`|역슬래시|
|`\*`|별표|
|`[A-Z]`|대문자 A~Z 중 하나|
|`[a-z]`|소문자 a~z 중 하나|
|`[0-9]`|숫자 0~9 중 하나|
|`[A-Za-z]`|대소문자 전체|

## 문자 클래스 [A-Z] 치환 패턴 ⭐️

```
sed 's/[A-Z]/./'
  s      → substitute (치환)
  [A-Z]  → 대문자 A~Z 중 하나 (범위 지정)
  .      → 치환할 문자 (점으로 바꾸기)
  기본    → 첫 번째 매칭만 치환 (g 없으면 줄당 1개)
```

```bash
# 첫 번째 대문자만 치환 (g 없음)
echo "Hello World" | sed 's/[A-Z]/./'
# .ello World  ← H 만 바뀜 (W 는 그대로)

# 모든 대문자 치환 (g 플래그)
echo "Hello World" | sed 's/[A-Z]/./g'
# .ello .orld  ← H, W 모두 바뀜

# 소문자 → X 로 치환
echo "hello world" | sed 's/[a-z]/X/g'
# XXXXX XXXXX

# 숫자만 제거
echo "abc123def456" | sed 's/[0-9]//g'
# abcdef
```

```
[A-Z] vs [A-Z]* 차이:
  [A-Z]   → 대문자 1개 (문자 하나씩 매칭)
  [A-Z]*  → 대문자 0개 이상 (연속된 대문자 덩어리 전체)

echo "ABCdef" | sed 's/[A-Z]/X/g'    # XXXdef  (각각 X로)
echo "ABCdef" | sed 's/[A-Z]*/X/'    # Xdef    (연속 덩어리 → 1개 X)
```

---

---

# ④ 정규식 활용

```bash
# .  아무 문자 1개
echo "cat bat rat" | sed 's/.at/dog/g'
# dog dog dog

# *  앞 문자 0개 이상
echo "aaabbbccc" | sed 's/b*/X/g'

# ^  줄 시작
echo "hello" | sed 's/^/>>> /'
# >>> hello

# $  줄 끝
echo "hello" | sed 's/$/ <<</'
# hello <<<

# [abc]  문자 클래스
echo "cat bat rat" | sed 's/[cbr]at/dog/g'
# dog dog dog
```

---

---

# ⑤ 자주 쓰는 패턴

## 공백 / 탭 처리

```bash
# 앞뒤 공백 제거
echo "  hello  " | sed 's/^ *//;s/ *$//'
# hello

# 여러 공백 → 하나로
echo "a   b    c" | sed 's/  */ /g'
# a b c

# 탭 → 공백
sed 's/\t/ /g' file.txt

# 공백 → 탭
sed 's/ /\t/g' file.txt
```

## 줄 삭제

```bash
# 특정 패턴 포함된 줄 삭제
sed '/error/d' log.txt

# 빈 줄 삭제
sed '/^$/d' file.txt

# 주석 줄 삭제 (# 으로 시작)
sed '/^#/d' config.txt
```

```
/^$/d 가 빈 줄을 삭제하는 이유:

  / / 사이 = 패턴
  ^  = 줄의 시작
  $  = 줄의 끝
  d  = delete (해당 줄 삭제)

  → 줄 시작(^) 바로 뒤에 줄 끝($) 이 오는 줄
  → 시작과 끝 사이에 아무것도 없음 = 빈 줄

비교:
  /^abc/   줄이 abc 로 시작
  /abc$/   줄이 abc 로 끝
  /^abc$/  줄이 정확히 abc 인 줄
  /^$/     줄이 정확히 비어있는 줄 ← 빈 줄
```

```bash
# 확인
echo -e "apple\n\nbanana\n\norange" | sed '/^$/d'
# apple
# banana
# orange
```

## 줄 출력 / 필터링

```
-n (--quiet / --silent):
  sed 는 기본적으로 모든 줄을 출력함
  -n 을 붙이면 자동 출력을 끔
  → p 명령어로 명시적으로 출력한 줄만 나옴

-n 없이 p:  모든 줄 출력 + 매칭 줄 한 번 더 출력 (중복)
-n + p:     매칭 줄만 출력 (원하는 것만)
```

```bash
# -n 없이 → 모든 줄 + 매칭 줄 중복 출력
echo -e "apple\nerror\nbanana" | sed '/error/p'
# apple
# error   ← 매칭
# error   ← p 로 한 번 더
# banana

# -n + p → 매칭 줄만 출력
echo -e "apple\nerror\nbanana" | sed -n '/error/p'
# error   ← 이것만

# 특정 패턴 포함된 줄만 출력
sed -n '/error/p' log.txt

# 특정 줄 번호만 출력
sed -n '3p' file.txt      # 3번째 줄
sed -n '1,5p' file.txt    # 1~5번째 줄
```

```
grep 과 비교:
  grep 'error' log.txt         ← 간단한 패턴 필터링
  sed -n '/error/p' log.txt    ← 같은 결과

  sed -n 는 다른 sed 명령어와 조합할 때 유용
  (치환 + 필터링 동시에)
```

## 파일 직접 수정 (-i)

```bash
# 파일 직접 수정 (주의: 원본 덮어씀)
sed -i 's/old/new/g' file.txt

# 백업 만들고 수정 (macOS 는 -i '' 필요)
sed -i.bak 's/old/new/g' file.txt   # Linux
sed -i '' 's/old/new/g' file.txt    # macOS
```

---

---

# ⑥ 여러 명령어 동시 실행

```bash
# -e 로 여러 명령어
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# ; 로 구분
sed 's/foo/bar/g; s/baz/qux/g' file.txt

# 실전: 앞뒤 공백 제거
sed 's/^ *//;s/ *$//' file.txt
```

---

---

# ⑦ sed vs awk 선택 기준

```
sed:
  텍스트 치환 / 삭제 / 필터링
  s/찾기/바꾸기/g 같은 간단한 변환
  파이프라인 중간에 한 줄 삽입

awk:
  컬럼(필드) 단위 처리
  조건 분기 / 계산 / 집계
  여러 컬럼을 각각 다르게 처리
```

```bash
# 탭 → ; 로 치환 → sed
paste - - - | sed 's/\t/;/g'

# 2번째 컬럼만 출력 → awk
cat file.txt | awk '{print $2}'
```

---

---

# 패턴 만드는 법 — s/X/Y/g 읽는 법

```
s / \t / ; / g
↑    ↑    ↑   ↑
s    찾기  바꾸기 g(전체)

단계별로 읽기:
  1. s  → 치환 명령어
  2. 찾을 것: \t (탭)
  3. 바꿀 것: ; (세미콜론)
  4. g  → 줄 전체에 적용

새 패턴 만들 때:
  s / [찾을것] / [바꿀것] / [플래그]
  플래그: g(전체) / i(대소문자무시) / 숫자(N번째만)
```

```bash
# 예시: 두 번째 쉼표만 | 로 치환
echo "a,b,c,d" | sed 's/,/|/2'
# a,b|c,d   ← 2번째 , 만 바뀜

# 예시: 숫자 앞에 # 붙이기
echo "abc 123 def" | sed 's/[0-9][0-9]*/&#/g'
# abc 123# def
# & = 매칭된 전체 문자열
```