---
aliases:
  - JSON 처리
  - JSON 모듈
  - 직렬화
  - 역직렬화
  - json.dumps
  - json.dump
  - json.loads
  - json.load
tags:
  - Python
  - JSON
related:
  - "[[Python_Dictionaries]]"
  - "[[Python_File_IO]]"
  - "[[Airflow_Hooks]]"
  - "[[Airflow_XComs]]"
  - "[[Serialization_JSON_XML]]"
---
## 개념 한 줄 요약

**JSON(JavaScript Object Notation)** 은 데이터를 교환할 때 쓰는 가장 표준적인 텍스트 포맷입니다. 
파이썬의 **Dictionary(`{}`)** 와 생김새가 거의 똑같아서, 파이썬에서는 `json` 모듈을 이용해 자유자재로 변환할 수 있습니다.

---
## 왜 필요한가 (Why)

**문제점:**
- 파이썬 프로그램 안에서 `{"name": "Luke"}`는 **딕셔너리(객체)** 이지만, 이걸 파일로 저장하거나 네트워크로 전송하려면 **문자열(String)** 로 바꿔야 합니다.
- 반대로, API에서 받아온 결과는 그냥 긴 **텍스트(String)** 일 뿐이라, 이걸 파이썬이 이해할 수 있는 **딕셔너리**로 바꿔야 데이터를 꺼내 쓸 수 있습니다.

**해결책:**
- **Serialization (직렬화):** 파이썬 객체(Dict) -> JSON 문자열 (전송/저장용)
- **Deserialization (역직렬화):** JSON 문자열 -> 파이썬 객체(Dict) (코드 사용용)

---
## Practical Context (실무 활용)

데이터 엔지니어는 숨 쉬듯이 JSON을 다룹니다.
1.  **Airflow XCom:** Task끼리 데이터를 주고받을 때 JSON 형태로 직렬화해서 DB에 저장합니다.
2.  **API 호출:** `HttpOperator`로 받아온 결과는 텍스트이므로 `json.loads()`로 풀어서 씁니다.
3.  **설정 파일:** DB 접속 정보 등을 `.json` 파일에 저장해두고 읽어서 씁니다.

---
## Code Core Points (암기 공식 4가지)

가장 헷갈리는 's'의 유무만 기억하면 됩니다.
- **`s`가 붙으면:** **String(문자열)** 을 다룬다. (메모리 상에서 변환)
- **`s`가 없으면:** **File(파일)** 을 다룬다. (하드디스크에 저장/읽기)

| 구분 | 파이썬 객체(Dict) → JSON | JSON → 파이썬 객체(Dict) |
| :--- | :--- | :--- |
| **문자열 (String)** | `json.dumps(data)` | `json.loads(text)` |
| **파일 (File)** | `json.dump(data, f)` | `json.load(f)` |

---
## Detailed Analysis

```python
import json

# 1. 파이썬 데이터 (Dictionary)
data = {
    "name": "Airflow",
    "is_active": True,
    "version": 2.8
}

# --- [Case A] 메모리에서 변환 (String) ---

# Dict -> JSON 문자열 (직렬화)
json_str = json.dumps(data)
print(type(json_str))  # <class 'str'> (이제 그냥 글자임)

# JSON 문자열 -> Dict (역직렬화)
restored_data = json.loads(json_str)
print(restored_data['name'])  # 'Airflow' (딕셔너리니까 키로 접근 가능!)


# --- [Case B] 파일로 저장/읽기 (File) ---

# Dict -> 파일로 저장
with open("config.json", "w") as f:
    json.dump(data, f)  # dumps 아님! dump 임!

# 파일 -> Dict로 읽기
with open("config.json", "r") as f:
    config = json.load(f) # loads 아님! load 임!
```

---
## 초보자가 자주 착각하는 포인트

1. **"JSON이랑 딕셔너리는 같은 거 아닌가요?"**
    - **아닙니다!** JSON은 그냥 **'텍스트(문자열)'** 이고, 딕셔너리는 파이썬의 **'메모리 구조'** 입니다.
    - `json_str['name']` 하면 에러 납니다. (문자열에 인덱싱한다고 혼남) 
    - 반드시 `json.loads()`를 거쳐야 합니다.

- **"따옴표 문제 (Single vs Double Quote)"**
    - 파이썬 딕셔너리는 `'key'`(작은따옴표)를 써도 되지만, **JSON 표준은 무조건 `"key"`(큰따옴표)** 여야 합니다.
    - 손으로 JSON 파일을 만들 때 실수로 작은따옴표를 쓰면 `json.load()` 할 때 에러가 납니다.

- **"한글이 깨져서 나와요!"**
    - `json.dumps(data)`를 하면 한글이 `\uXXXX` 같은 유니코드로 변환됩니다.
    - 한글을 그대로 보고 싶다면 `json.dumps(data, ensure_ascii=False)` 옵션을 켜세요.
