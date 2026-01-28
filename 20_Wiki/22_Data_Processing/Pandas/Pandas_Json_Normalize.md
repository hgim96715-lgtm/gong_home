---
aliases:
  - json_normalize
  - 제이슨정규화
  - JSON펼치기
tags:
  - Pandas
  - Python
  - 전처리
related:
  - "[[Airflow_Hooks]]"
  - "[[Python_JSON]]"
  - "[[Python_Requests_Response]]"
  - "[[Serialization_JSON_XML]]"
---
## 개념 한 줄 요약

**`json_normalize`** 는 Pandas가 제공하는 특수 함수로, 
딕셔너리나 리스트가 중첩된(Nested) **복잡한 JSON 데이터를 평평한 테이블(DataFrame) 형태로 '다림질'해주는 도구**입니다.

---
## 왜 필요한가 (Why)

**문제점:**
- API에서 받아온 데이터는 보통 이렇게 생겼습니다: `{"이름": "루크", "고향": {"행성": "타투인", "인구": 200000}}`
- 이걸 그냥 `pd.DataFrame()`에 넣으면, `고향` 컬럼 안에 딕셔너리가 통째로 들어가 버려서 나중에 데이터를 꺼내 쓰기가 매우 불편합니다.

**해결책:**
- `json_normalize`를 쓰면 **`고향.행성`**, **`고향.인구`** 처럼 하위 키들을 자동으로 컬럼으로 꺼내서 납작하게 펴줍니다.

---
## Practical Context (실무 활용)

데이터 엔지니어링 실무에서 **API 호출(`HttpOperator`) 결과**를 DB에 적재하기 전 가공할 때 필수적으로 쓰입니다.
- **Star Wars API 예시:** 캐릭터 정보 안에 '영화 목록(films)'이나 '탈것(vehicles)' 같은 리스트가 들어있을 때, 이걸 깔끔한 CSV 행으로 만들 때 씁니다.

---
## Code Core Points

```python
from pandas import json_normalize

# 1. 중첩된 JSON 데이터 (API 응답 예시)
data = [
    {
        "id": 1,
        "name": "Luke Skywalker",
        "location": {               # <-- 딕셔너리 안에 또 딕셔너리 (Nested)
            "planet": "Tatooine",
            "galaxy": "Outer Rim"
        }
    }
]

# 2. 그냥 DataFrame으로 만들었을 때 (안 예쁨 ❌)
# df = pd.DataFrame(data)
# 결과: location 컬럼 안에 {'planet': 'Tatooine'...} 이 그대로 들어감.

# 3. json_normalize 사용 (예쁨 ✅)
df = json_normalize(data)
```

결과 (`df`):

|**id**|**name**|**location.planet**|**location.galaxy**|
|---|---|---|---|
|1|Luke Skywalker|Tatooine|Outer Rim|

>**점(.)** 으로 연결된 새로운 컬럼 이름이 자동으로 생성됩니다!

---
## Advanced Usage (리스트가 섞여 있을 때)

데이터가 더 복잡해서 "어떤 키는 그대로 두고, 리스트 안에 있는 것만 펴고 싶을 때"는 파라미터를 씁니다.

- **`record_path`**: "이 경로에 있는 리스트를 메인 데이터로 써라."
- **`meta`**: "리스트 밖(상위 레벨)에 있는 이 정보도 옆에 붙여줘라."

```python
data = {
    "school": "Jedi Academy",
    "students": [  # <-- 이걸 펴고 싶음 (record_path)
        {"name": "Anakin", "age": 9},
        {"name": "Obi-Wan", "age": 25}
    ]
}

# 학생들 정보를 펴면서, 학교 이름(school)도 옆에 붙이기
df = json_normalize(
    data, 
    record_path=['students'], 
    meta=['school']
)
```

**결과:**

|**name**|**age**|**school**|
|---|---|---|
|Anakin|9|Jedi Academy|
|Obi-Wan|25|Jedi Academy|

---
## Practical Context (실무 활용: API 데이터 변환)

데이터 엔지니어링에서는 **"Requests로 가져오고 -> Pandas로 편다"** 가 국룰 패턴입니다.

```python
import requests 
from pandas import json_normalize 

# 1. 파이썬(requests)으로 데이터 가져오기 
# Star Wars API에서 루크 스카이워커 정보 호출 
response = requests.get("https://swapi.dev/api/people/1/").json() 
# response 결과: {'name': 'Luke...', 'height': '172', ...} (그냥 딕셔너리) 

# 2. 판다스(json_normalize)로 테이블 만들기 
# 필요한 정보만 쏙쏙 뽑아서 깔끔한 표로 변환 
df = json_normalize({ 
	"name": response["name"], 
	"height": response["height"], 
	"mass": response["mass"] 
})

# 3. 결과 확인 (df) 

# | name | height | mass | 
# |----------------|--------|------| 
# | Luke Skywalker | 172 | 77 |
```


---
## Common Beginner Misconceptions

1. **"그냥 `pd.DataFrame(json)` 쓰면 안 되나요?"**
    - JSON이 **완전히 평평한(Flat)** 구조라면 괜찮습니다. 
    - 하지만 실무 데이터의 99%는 중첩 구조라 `json_normalize`가 답입니다.

2. **"리스트(`[]`)랑 딕셔너리(`{}`)가 섞여 있으면 에러 나요."**
    - `json_normalize`는 기본적으로 **리스트 안에 담긴 딕셔너리** 구조를 좋아합니다. 
    - 최상위가 딕셔너리 하나라면 대괄호 `[]`로 감싸서 리스트로 만들어주는 센스가 필요할 때도 있습니다.

3. **"Requests는 뭔가요?"**: 
	- `requests`는 파이썬 내장 함수가 아니라 **외부 라이브러리**입니다. 
	- Airflow에는 기본으로 깔려 있지만, 내 컴퓨터에서 쓰려면 `pip install requests`를 해야 합니다.

4. **"왜 `.json()`을 붙이나요?"**: 
	- `requests.get()`의 결과는 '응답 상자(Response Object)'입니다. 
	- 그 상자를 까서 **내용물(딕셔너리)** 을 꺼내려면 `.json()`을 꼭 붙여야 합니다.