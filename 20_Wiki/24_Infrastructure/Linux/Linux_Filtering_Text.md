---
aliases:
  - grep
  - 그렙
  - 문자열검색
  - 정규표현식
  - 파이프
tags:
  - Linux
related:
  - "[[File_Permissions]]"
  - "[[SSH_Connection]]"
  - "[[00_Linux_HomePage]]"
  - "[[00_Linux_HomePage]]"
  - "[[SQL_Regular_Expression]]"
---


# Linux_Filtering_Text — grep

## 개념 한 줄 요약

```
산더미처럼 쌓인 텍스트 속에서 내가 원하는 패턴이 들어있는 줄만 쏙쏙 뽑아주는 거름망
이름 유래: Global Regular Expression Print
```

---

---

# ① 왜 필요한가

```
서버 로그는 수십만 줄 → "Error" 발생 줄을 눈으로 찾는 건 불가능
ls -al 파일 1,000개 → 내가 찾는 파일이 어디 있는지 스크롤 하다 놓침

grep 을 사용하면:
  수만 줄 중 내가 궁금한 단어가 포함된 줄만 출력
  불필요한 정보는 버리고 핵심 정보만 남기는 데이터 엔지니어 필수 도구
```

---

---

# ② 기본 문법

```bash
# grep [옵션] "찾을패턴" [파일]

# 파일에서 찾기
grep "ERROR" server.log

# 다른 명령어 결과에서 건져내기 (Pipe)
ls -al | grep "txt"
```

## 파이프(`|`) 란

```
앞 명령어(ls)의 실행 결과를 파일로 저장하지 않고
즉시 뒤 명령어(grep)의 입력으로 넘겨주는 연결 통로

장점: 임시 파일 불필요 → 빠르고 깔끔
```

## 자주 쓰는 조합 TOP 2

```bash
# 실행 중인 프로그램 찾기
ps aux | grep python

# 에러 로그만 보기
cat server.log | grep "ERROR"
```

---

---

# ③ 옵션 총정리

|옵션|이름|설명|예시|
|---|---|---|---|
|`-i`|Insensitive|대소문자 구분 없이|`grep -i "error"` → Error, ERROR, error 전부|
|`-v`|Invert|패턴 제외하고 출력|`grep -v "INFO"` → INFO 빼고 나머지|
|`-r`|Recursive|하위 폴더까지 재귀 탐색|`grep -r "def" .`|
|`-n`|Number|줄 번호 함께 출력|`grep -n "main" app.py`|
|`-c`|Count|매칭된 줄 수만 출력|`grep -c "FAIL" test.log`|
|`-w`|Word|**단어 단위 매칭**|`grep -w "the" text.txt`|
|`-E`|Extended|확장 정규표현식 사용|`grep -E "Error\|Fail"`|
|`-e`|Expression|여러 패턴을 동시에 검색|`grep -e "Error" -e "Fail"`|
|`-l`|List|매칭된 파일명만 출력|`grep -rl "TODO" .`|
|`-A n`|After|매칭 줄 + 아래 n줄|`grep -A 3 "ERROR" log`|
|`-B n`|Before|매칭 줄 + 위 n줄|`grep -B 3 "ERROR" log`|
|`-C n`|Context|매칭 줄 + 위아래 n줄|`grep -C 3 "ERROR" log`|

---

---

# ④ `-w` — 단어 단위 매칭 (중요)

```bash
# -w 없이 → "the" 를 포함하는 모든 문자열 매칭
grep "the" text.txt
# "the", "there", "another", "atherosclerosis" 전부 매칭 ❌

# -w 있으면 → 정확히 단어 "the" 만 매칭
grep -w "the" text.txt
# "the" 만 ✅  "there", "another" 는 제외
```

```
-w 의 원리:
  단어 경계(word boundary) 기준으로 매칭
  영문자·숫자·언더스코어(_) 가 아닌 문자(공백, 마침표, 쉼표 등)로 둘러싸인 경우만 매칭

  grep -w "the"  는 사실  grep '\bthe\b' 과 같음
```

---

---

# ⑤ `-E` — 확장 정규표현식 (OR / 패턴 조합)

```bash
# OR 조건 — "Error" 또는 "Fail" 이 들어간 줄
grep -E "Error|Fail" server.log

# 여러 단어 OR + 대소문자 무시 조합
grep -iE "error|fail|warn" server.log

# 단어 단위 OR (가장 정확한 방법)
grep -iwE "the|that|then|those" text.txt
# ↑ the / that / then / those 각각 단어로만 매칭
# "there", "another" 같은 것은 제외
```

## `-iE` vs `-iwE` 차이

```bash
echo "there is another thing" | grep -iE "the|that"
# "there", "another" 도 매칭됨 ❌ (문자열 일부 포함이면 다 걸림)

echo "there is another thing" | grep -iwE "the|that"
# 매칭 없음 ✅ (정확히 단어로 "the" / "that" 이 없으므로)

echo "the cat and that dog" | grep -iwE "the|that"
# "the", "that" 만 매칭 ✅
```

```
-E "the|that|then|those"        문자열 일부 포함이면 매칭 ❌
-wE "the|that|then|those"       단어 단위로만 매칭 ✅
-iwE "the|that|then|those"      대소문자 무시 + 단어 단위 ✅ (가장 정확)
```

---

---

# ⑥ `-e` — 여러 패턴 동시 검색 + 역참조

## 기본 사용

```bash
# -e 로 패턴 여러 개 동시 지정
grep -e "Error" -e "Fail" server.log
# "Error" 또는 "Fail" 이 있는 줄 전부 출력
```

```
-e vs -E 차이:
  -E "Error|Fail"    확장 정규식으로 OR 표현 (한 패턴 안에서)
  -e "Error" -e "Fail"  패턴을 여러 개 따로 지정 (각각의 패턴)

  결과는 같지만 패턴이 복잡할 때 -e 가 가독성 좋음
```

## 역참조(Backreference) 와 함께 — `\1`

```
\( \)  → 그룹 캡처 (BRE — 기본 정규표현식에서는 괄호에 \ 붙임)
\1     → 첫 번째 그룹에서 잡힌 내용을 다시 참조

→ "같은 패턴이 반복되는 것" 을 찾을 때 사용
```

```bash
# 같은 숫자가 연속으로 두 번 (11, 22, 33 ...)
grep '\([0-9]\)\1' file.txt
#     ↑캡처  ↑\1=앞과 같은 숫자

# 공백을 사이에 두고 같은 숫자 반복 (1 1, 9 9 ...)
grep '\([0-9]\) \1' file.txt
#     ↑캡처   ↑공백 ↑\1=앞과 같은 숫자

# 두 패턴 동시에 검색 — -e 로 OR 연결
grep -e '\([0-9]\)\1' -e '\([0-9]\) \1' file.txt
```

```
패턴 분석:
  \([0-9]\)   → 숫자 1개를 그룹으로 캡처
  \1          → 바로 앞에서 캡처한 숫자와 동일한 값

  \([0-9]\)\1   → "11", "22", "99" 처럼 붙어있는 반복
  \([0-9]\) \1  → "1 1", "9 9" 처럼 공백 사이 반복
```

```
⚠️ 주의: 기본 grep (BRE) 에서는 \( \) 로 그룹 작성
         grep -E (ERE) 에서는 ( ) 로 작성 (\ 없음)

grep    '\([0-9]\)\1'    BRE ✅
grep -E '([0-9])\1'     ERE ✅ (같은 결과)
```

---

---

# ⑦ 정규표현식 기초

```bash
# ^ : 줄의 시작
grep "^ERROR" log.txt           # ERROR 로 시작하는 줄

# $ : 줄의 끝
grep "done$" log.txt            # done 으로 끝나는 줄

# . : 임의의 문자 1개
grep "c.t" words.txt            # cat, cut, cot 등

# [0-9] : 숫자 하나
grep "^[0-9]" data.txt          # 숫자로 시작하는 줄

# * : 앞 문자 0번 이상 반복 (쉘 와일드카드 * 와 다름!)
grep "lo*g" text.txt            # lg, log, loog, looog 등

# -E 로 확장 정규표현식 사용 가능
grep -E "^[0-9]{4}-[0-9]{2}" log.txt   # 날짜 형식으로 시작하는 줄
```

```
⚠️ 쉘 와일드카드 * vs 정규표현식 *

  쉘:    *.txt   → 확장자가 .txt 인 모든 파일 (파일명 매칭)
  grep:  lo*g    → l 다음에 o가 0번 이상 반복된 후 g (문자 패턴 매칭)

  완전히 다른 문법 — 혼동 금지
```

---

---

# ⑧ 실무 활용 패턴

```bash
# 로그에서 에러만 추출 + 줄번호 표시
grep -n "ERROR" server.log

# 에러 발생 전후 3줄씩 함께 보기 (맥락 파악용)
grep -C 3 "ERROR" server.log

# 에러 발생 건수만 세기
grep -c "ERROR" server.log

# INFO 로그 제외하고 나머지만 보기
grep -v "INFO" server.log

# 프로젝트 전체에서 TODO 주석 찾기 (파일명 + 줄번호)
grep -rn "TODO" .

# 특정 확장자 파일에서만 찾기
grep -rn "import" . --include="*.py"

# 프로세스 찾기 (grep 자신은 제외)
ps aux | grep python | grep -v grep

# 여러 로그 파일 중 에러 있는 파일명만 출력
grep -rl "ERROR" ./logs/

# 카프카 토픽 목록에서 특정 토픽 확인
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh --list \
  --bootstrap-server localhost:9092 | grep "train"
```

---

---

# ⑨ 자주 하는 실수

## "파일 내용이 지워지는 건 아닌가요?"

```bash
grep -v "INFO" server.log
# → 화면에서만 INFO 가 안 보일 뿐
# → 원본 파일은 절대 변경되지 않음 (Read Only)
```

## grep 자기 자신이 결과에 포함될 때

```bash
ps aux | grep python
# "grep python" 프로세스 자체도 결과에 포함됨

# 해결책
ps aux | grep python | grep -v grep
# 또는
ps aux | grep "[p]ython"   # 정규표현식으로 grep 자신 제외
```

## `-E` 없이 `|` 쓰면 안 됨

```bash
grep "Error|Fail" log.txt      # ❌ | 를 문자 그대로 검색함
grep -E "Error|Fail" log.txt   # ✅ OR 조건으로 동작
```

---

---

# ⑩ 옵션 조합 치트시트

```bash
grep -i    "error" log          # 대소문자 무시
grep -in   "error" log          # 대소문자 무시 + 줄번호
grep -v    "INFO"  log          # INFO 제외
grep -c    "ERROR" log          # 건수만
grep -w    "the"   text         # 단어 단위
grep -E    "A|B"   log          # OR 조건
grep -iE   "A|B"   log          # OR + 대소문자 무시
grep -iwE  "A|B"   log          # OR + 대소문자 무시 + 단어 단위
grep -rn   "str"   .            # 재귀 + 줄번호
grep -rl   "str"   .            # 재귀 + 파일명만
grep -C 3  "ERR"   log          # 위아래 3줄 맥락
grep -A 5  "ERR"   log          # 아래 5줄
grep -B 2  "ERR"   log          # 위 2줄
```