---
aliases:
  - jq
  - json파싱
  - json
  - pretty-print
  - json포맷팅
tags:
  - Linux
  - API
related:
  - "[[Filtering_Text]]"
  - "[[File_Transfer]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[Python_JSON]]"
  - "[[00_Linux_HomePage]]"
  - "[[Serialization_JSON_XML]]"
---

## 개념 한 줄 요약
**"터미널에서 JSON 데이터를 읽기 좋게 성형(Pretty-print)하고, SQL처럼 원하는 값만 쏙 뽑아내는 도구."**

* **`jq`:** 리눅스 환경에서 JSON 데이터를 파싱(Parsing), 검색(Query), 변환(Transform)할 수 있는 가볍고 강력한 명령줄 도구입니다[
- 복잡한 JSON 구조에서 특정 필드만 필터링하거나, 데이터를 재구조화할 때 필수적입니다

---
## 왜 필요한가? (Why)

**문제점:**
- "`curl`로 API 데이터를 받았는데, 한 줄로 뭉쳐져 있어서 눈이 아파요." (가독성 제로)
- "JSON 파일 안에 있는 'user_id' 값만 뽑아서 변수에 저장하고 싶은데, `grep`이나 `awk`로는 괄호(`{}`) 처리가 너무 힘들어요."

**해결책:**
- **`jq .`** 명령어 하나면 뭉친 JSON을 **들여쓰기(Indentation)**가 적용된 예쁜 형태로 보여줍니다
- **`jq '.user.id'`** 처럼 경로만 찍어주면 정확하게 그 값만 추출할 수 있습니다

---
## Code Core Points: 필수 필터 문법 (Filters)

`jq` 뒤에 오는 **작은따옴표(`''`) 안의 기호**들이 핵심입니다.

### ① 전체 보기 & 예쁘게 보기 (Pretty Print)

```bash
# 점(.)은 "현재 JSON 데이터 전체"를 의미합니다.
# 뭉쳐진 JSON을 보기 좋게 정렬해줍니다.
echo '{"name":"John", "age":30}' | jq '.'
```

### ② 특정 값만 뽑기 (Select Field)

```bash
# .key이름 : 해당 키의 값을 가져옵니다.
echo '{"name": "John", "age": 30}' | jq '.name'
# 결과: "John"
```

### ③ 배열(Array) 다루기

```bash
# [] : 배열 안에 있는 모든 요소를 꺼냅니다.
# .[].name : 배열을 순회하며 각 객체의 'name'만 뽑습니다.
echo '[{"name": "John"}, {"name": "Jane"}]' | jq '.[].name'
# 결과:
# "John"
# "Jane"
```

---
## Code Core Points: 자주 쓰는 옵션 (Options)

이 두 가지 옵션은 실무에서 매일 씁니다

| **옵션**                | **설명**                                                                  | **사용 예시**               |
| --------------------- | ----------------------------------------------------------------------- | ----------------------- |
| **`-r`** (Raw)        | **따옴표 제거.** 쉘 스크립트 변수로 넘길 때 필수입니다. 문자열의 따옴표(`""`)를 벗겨서 순수 텍스트로 출력합니다.   | `jq -r '.name'`         |
| **`-c`** (Compact)    | **압축 모드.** 불필요한 공백과 줄바꿈을 제거하여 한 줄로 만듭니다. (로그 파일 저장용)                    | `jq -c .`               |
| **`-s`** (Slurp)      | **통합 모드 (Slurp).** 여러 개의 JSON 입력(파일)을 **하나의 거대한 배열(`[]`)**로 묶어서 읽어들입니다. | `jq -s . a.json b.json` |
| **`-M`** (Monochrome) | **컬러 끄기.** 출력 결과에서 알록달록한 색상 코드를 제거합니다. (파일로 리다이렉션 `>` 하거나 파이프           | 넘길 때 깨짐 방지)             |

---
## 실전 활용 예제 (Real-world Examples)

### 특정 필드 값만 추출 (Extract a Specific Field)

```bash
echo '{"name": "John", "age": 30}' | jq '.name'
```

### 배열 데이터 필터링 (Filter JSON Data)

```bash
echo '[{"name": "John", "age": 30}, {"name": "Jane", "age": 25}]' | jq '.[].name' 
```

### 데이터 값 수정하기 (Modify JSON Data)

```bash
echo '{"name": "John", "age": 30}' | jq '.age = 31' 
```

### 데이터 구조 재조립 (Restructure JSON Data)

```bash
# 배열 안의 객체에서 name만 뽑아 새로운 객체 배열로 만듦 
echo '[{"name": "John", "age": 30}, {"name": "Jane", "age": 25}]' | jq '[{name: .[].name}]'
```

### 보기 좋게 출력 (Pretty-Print JSON Data)

```bash
echo '{"name": "John", "age": 30}' | jq . 
```

### 따옴표 없이 순수 텍스트 출력 (Raw Output, -r)

```bash
echo '{"name": "Alice", "age": 25}' | jq -r '.name'
```

### 여러 파일을 하나의 배열로 합치기 (Slurp Mode, -s)

```bash
echo '{"name": "Alice"}' > file1.json
echo '{"name": "Bob"}' > file2.json 
jq -s '.' file1.json file2.json
```

### 한 줄로 압축해서 출력 (Compact Output, -c)

```bash
echo '{"name": "Alice", "age": 25}' | jq -c '.'
```

### 컬러 출력 끄기 (Disable Color, -M)

```bash
# (로그 파일로 저장하거나 파이프로 넘길 때 유용)
echo '{"name": "Alice", "age": 25}' | jq -M '.'
```



---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "결과에 쌍따옴표(`"`)가 붙어서 나와요!"

- `jq '.name'`을 하면 결과가 `"John"` 처럼 JSON 문자열 규격대로 나옵니다.
- 순수한 텍스트 `John`만 얻고 싶다면 **`-r` (raw output)** 옵션을 반드시 붙여야 합니다.

### ② "파이프(`|`)를 어디에 써야 하죠?"

- 리눅스 파이프(`|`)와 `jq` 내부 파이프(`|`)를 혼동하기 쉽습니다.
- **리눅스 파이프:** 명령어와 명령어 사이 (`cat file.json | jq .`)
- **jq 내부 파이프:** 필터와 필터 사이 (`jq '.users | .[].name'`)

