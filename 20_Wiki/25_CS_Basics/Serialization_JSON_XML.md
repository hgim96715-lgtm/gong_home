---
aliases:
  - Serialization
  - Deserialization
  - JSON
  - XML
  - 직렬화
  - 역직렬화
  - Parsing
tags:
  - CS
  - FileFormat
related:
  - "[[Data_Formats_CSV]]"
  - "[[Linux_JSON_Processing]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[Python_JSON]]"
  - "[[00_CS_HomePage]]"
  - "[[CS_HTTP_Basics]]"
  - "[[CS_REST_API_Methods]]"
---
## 개념 한 줄 요약 

**"메모리에 떠 있는 '객체(Object)'를 파일이나 네트워크로 보내기 위해 '문자열(String)'로 납작하게 펴는 것."**

* **직렬화 (Serialization):** 파이썬 딕셔너리(`dict`) ➡️ JSON 파일 (`dump`)
* **역직렬화 (Deserialization):** JSON 파일 ➡️ 파이썬 딕셔너리 (`load`)
* **비유:** 조립된 가구(객체)는 배송하기 힘드니까, 납작하게 분해(직렬화)해서 박스에 넣고, 도착하면 다시 조립(역직렬화)하는 과정.

---
## JSON (JavaScript Object Notation) 

현재 웹과 데이터 엔지니어링의 **사실상 표준(Standard)** 입니다.

### ① 특징

* **Key-Value 구조:** 파이썬의 **Dictionary**와 거의 똑같이 생겼습니다.
* **가벼움:** 괄호`{}`와 콤마`,`만 사용해서 군더더기가 없습니다.
* **타입 지원:** 문자열, 숫자, 불린(True/False), 리스트(배열), Null 지원.

### ② 생김새

```json
{
  "name": "Spark",
  "version": 3.5,
  "features": ["Fast", "Distributed"],
  "is_active": true
}
```

### ③ 언제 쓰나요?

- REST API 응답 데이터 (웹 서버 ↔️ 클라이언트)
- NoSQL 데이터베이스 (MongoDB, Elasticsearch)
- 로그 파일 저장

---
## XML (eXtensible Markup Language) 

JSON 이전에 세상을 지배했던, 문서 중심의 포맷입니다. 
HTML의 조상님입니다.

### ① 특징

- **태그(Tag) 구조:** `<tag>값</tag>` 형태로 엽고 닫습니다.
- **계층적(Tree):** 부모-자식 관계가 명확합니다.
- **무거움:** 닫는 태그(`</name>`) 때문에 글자 수가 많아 용량이 큽니다.
- **메타데이터:** 속성(Attribute)을 넣을 수 있어 복잡한 표현이 가능합니다.

### ② 생김새

```XML
<project>
  <name>Spark</name>
  <version type="stable">3.5</version>
  <features>
    <item>Fast</item>
    <item>Distributed</item>
  </features>
</project>
```

### ③ 언제 쓰나요?

- **설정 파일:** 하둡(Hadoop), 스파크(Spark)의 설정 파일(`*-site.xml`, `pom.xml`)은 여전히 XML을 씁니다.
- 레거시(Legacy) 시스템 간 통신 (SOAP API).

---
## JSON vs XML 비교

|**구분**|**JSON**|**XML**|
|---|---|---|
|**가독성**|**좋음** (깔끔함)|보통 (태그가 많아 복잡해 보임)|
|**용량**|**가벼움**|무거움 (닫는 태그 때문)|
|**파싱 속도**|**빠름**|상대적으로 느림|
|**주석(Comment)**|**공식적으로 불가** ❌|가능 `` ✅|
|**스키마 검증**|가능하긴 함 (JSON Schema)|**매우 강력함** (DTD, XSD)|
|**주 용도**|**데이터 전송 (API), 로그**|**설정 파일 (Config), 문서 저장**|

---
## Python & Spark 활용 코드

### ① Python (기본 내장 라이브러리)

```python
import json

data = {"name": "Test", "score": 100}

# 1. 직렬화 (Dict -> Str)
json_str = json.dumps(data) 

# 2. 역직렬화 (Str -> Dict)
data_dict = json.loads(json_str)
```

### ② Spark (데이터 읽기)

스파크는 JSON을 아주 잘 읽지만, XML은 외부 라이브러리가 필요합니다.

```python
# JSON 읽기 (스키마 자동 추론)
df_json = spark.read.json("data.json")

# XML 읽기 (spark-xml 라이브러리 필요)
df_xml = spark.read \
    .format("com.databricks.spark.xml") \
    .option("rowTag", "project") \
    .load("data.xml")
```

----
>"신규 프로젝트라면 무조건 **JSON**을 써. 고민할 필요도 없어.
> 하지만 데이터 엔지니어라면 **XML**을 피할 수는 없을 거야.
>  왜냐하면 **하둡(Hadoop) 생태계의 모든 설정 파일이 XML**로 되어 있거든. 
>  그래서 'XML은 구식이야!'라고 무시하지 말고, **'태그 안에 값이 있다'** 는 구조 정도는 눈에 익혀둬야 해!"