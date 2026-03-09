---
aliases:
  - Encoding
  - Decoding
  - ASCII
  - UTF-8
  - CP949
  - EUC-KR
  - Unicode
  - 인코딩
  - 한글깨짐
tags:
  - CS
  - Encoding
related:
  - "[[Pandas_Read_Write]]"
  - "[[Data_Formats_CSV]]"
  - "[[Python_URL_Parsing]]"
---
# Encoding_Decoding_Concept

## 개념 한 줄 요약

> **"컴퓨터(0과 1)와 인간(글자) 사이의 통역 번역기(Dictionary)."**

|방향|이름|변환|비유|
|---|---|---|---|
|글자 → 숫자|**인코딩 (Encoding)**|`'가'` → `0101...`|사람의 데이터를 기계어로 포장|
|숫자 → 글자|**디코딩 (Decoding)**|`0101...` → `'가'`|기계어를 다시 사람의 데이터로 해석|

```python
# Python
"안녕".encode("utf-8")    # str -> bytes (저장/전송용)
b'\xec\x95\x88\xeb\x85\x95'.decode("utf-8")  # bytes -> str (출력/분석용)
```

```
헷갈림 방지 팁:
Encoding: Enter  (컴퓨터 속으로 들어감)
Decoding: Detect (사람이 다시 식별함)
```

> **깨짐 현상:** 저장할 때 쓴 번역기(A) 와 읽을 때 쓴 번역기(B) 가 달라서 발생하는 대참사.

---

---

# ① 인코딩 역사 3단계

## ASCII — "미국 표준"

```
1바이트(7비트) 사용 → 128개 문자만 표현
영문 대소문자, 숫자, 특수기호

한글: 전혀 표현 못 함
한글을 ASCII 로 읽으면 ??? 또는 에러
```

## CP949 (EUC-KR) — "한국 윈도우 표준"

```
ASCII 로는 한글을 못 쓰니까 한국이 독자적으로 2바이트로 확장한 규격

EUC-KR : 완성형 한글 표준 (2,350자만 표현 가능)
          '똠', '펲' 같은 일부 글자는 표현 못 함
CP949  : MS 가 EUC-KR 을 더 확장한 것 (모든 한글 가능, 일명 MS949)

문제점: 윈도우/엑셀 에서만 표준
        맥 / 리눅스 / 서버 환경과 호환이 안 됨
```

## UTF-8 (유니코드) — "전 세계 표준"

```
"전 세계 모든 문자를 하나의 규격으로!"

가변 길이:
  영어/숫자: 1바이트 (ASCII 와 호환, 용량 절약)
  한글:      3바이트 (CP949 보다 1.5배 더 큼)

현황: 웹, 리눅스, Spark, Python 3 의 기본값(Default)
```

## 한글 1글자 용량 비교

|구분|ASCII|CP949|UTF-8|
|---|:-:|:-:|:-:|
|**저장 크기**|1 Byte|2 Bytes|3 Bytes|
|**한글 지원**|X|O (일부)|O (전부)|
|**호환**|영어만|윈도우|전 세계|
|**비고**|—|'똠방각하' 깨질 수 있음|용량이 1.5배 더 큼|

---

---

# ② 한글이 깨지는 이유 — Mojibake (깩뛟쀍?!)

> 가장 흔한 시나리오: **윈도우 엑셀(CP949) 로 만든 CSV 를 Python / Spark (UTF-8) 로 열었을 때**

## 발생 원리

```
① 저장 (CP949): '안' 이라는 글자를 2바이트 (B0 A1) 로 저장

② 읽기 (UTF-8): Python 은 기본적으로 3바이트를 읽어서 한 글자를 만들려 함

③ 결과: B0 A1 뒤에 있는 엉뚱한 바이트까지 끌어다 해석
         → 외계어 (깩뛟쀍) 가 나오거나 UnicodeDecodeError 발생
```

---

---

# ③ 해결 방법

> **무조건 "파일을 저장할 때 썼던 인코딩 방식" 을 옵션으로 알려줘야 한다.**

## Pandas

```python
# 기본 UTF-8 로 읽으려다 에러 발생
# df = pd.read_csv("korean_data.csv")  <- UnicodeDecodeError

# 해결: 인코딩 지정
df = pd.read_csv("korean_data.csv", encoding="cp949")
# 또는
df = pd.read_csv("korean_data.csv", encoding="euc-kr")
```

## Spark

```python
# Spark 도 기본값이 UTF-8
df = spark.read \
    .option("encoding", "cp949") \
    .csv("korean_data.csv")
```

## Python 파일 읽기

```python
# 기본 UTF-8
with open("korean_data.txt", "r", encoding="utf-8") as f:
    data = f.read()

# CP949 파일 읽기
with open("korean_data.txt", "r", encoding="cp949") as f:
    data = f.read()

# 인코딩을 모를 때: errors="ignore" 로 일단 무시하고 열기
with open("korean_data.txt", "r", encoding="utf-8", errors="ignore") as f:
    data = f.read()
```

---

---

# 실무 팁

```
신입들이 제일 많이 당황하는 게 "엑셀 파일 받아서 열었는데 깨져요!" 야.
범인은 99% 확률로 CP949.

개발자끼리는 UTF-8 이 국룰이지만,
경영지원팀이나 클라이언트가 보내주는 파일은
대부분 엑셀에서 저장된 거라 CP949 일 확률이 높다는 걸 항상 염두에 둬.
```

|출처|인코딩 추정|
|---|---|
|개발자가 만든 파일|UTF-8|
|윈도우 엑셀에서 저장한 CSV|CP949|
|공공데이터포털 CSV 다운로드|CP949 (주의!)|
|리눅스 / 맥 / 웹|UTF-8|

> 공공데이터포털에서 받은 CSV 도 CP949 인 경우가 많다. `pd.read_csv("파일.csv", encoding="cp949")` 먼저 시도해보기.